---
name: llm-narrative-json-shape
description: "Fix LLM structured-JSON 502s where a weak model mangles multi-paragraph text fields (paragraph text landing in JSON key position); covers the array-vs-escaped-string contract choice, punctuation-aware joining, retry-gate design, and the strictness/salvage traps."
---

# LLM narrative/paragraph JSON shape: fix 502s from mangled multi-paragraph text

Use when an LLM-backed endpoint 502s with `LLM response is not valid JSON: <Type>: unknown field "<some prose>"` — especially when a prompt asks for multi-paragraph prose inside a JSON string field.

## Diagnose the class first: valid JSON != correct JSON

A custom `UnmarshalJSON` whitelist error (e.g. `Finding: unknown field "{ "`) fires only AFTER an inner `json.Unmarshal` into a `map` succeeds. So the document was **syntactically valid JSON with a bogus member name** — not corrupted JSON.

Consequences, in the order they are usually gotten wrong:

- Salvage helpers (`stripJSONWrapping`, `salvageJSONObject`, first-`{`-to-last-`}` extraction) **cannot** fix it. The document already decodes.
- A retry gate that fires only on a *syntax* error (`if syntaxParse == nil { don't retry }`) will retry **zero** of these failures. Do not port that gate.
- Do not "fix" it by dropping `DisallowUnknownFields`: the model emits `"narrative": "p1", "p2 text": "field"`, so ignoring unknown keys silently keeps only `p1` and drops the rest — data loss disguised as success.

**Get the raw bytes before theorizing.** If the failure path logs only the error, add a truncated raw-content preview first — otherwise the shape is guesswork.

## The fix that works: legitimize the array shape

The corruption is the model trying to express N paragraphs as N JSON strings. Asking for a **JSON array of paragraph strings** removes the failure mode instead of policing it.

Measured on a weak OpenAI-compat model (llama-3.1-8b), real endpoint:

| Prompt asks for | Parse failures | >=3 paragraphs |
|---|---|---|
| one escaped `\n\n` string | 1/24 per-attempt | 3/12 |
| JSON array of paragraphs | **0/24** | **12/12** |

Keep the wire contract unchanged: the model-facing shape is an array, but the field stays `string` on the response. Accept `string` **or** `[]string` when decoding and join into the single string. This mirrors the lenient-coercion convention already used for sibling scalar fields (numbers/bools coerced to strings).

A struct-level `UnmarshalJSON` that replicates its own key whitelist still rejects unknown keys — it does not silently defeat strict decoding. Verify the actual behavior rather than assuming either way.

## The join is the subtle part: it must be punctuation-aware

A naive `strings.Join(elems, "\n\n")` **mangles output**. The model sometimes splits ONE sentence at an embedded quoted value:

```
"...bbl/d. The top-producer ", "ANGSI-A12", "contributes 62% of the top 20% concentration..."
```

Naive join prints `ANGSI-A12` as its own paragraph, mid-sentence.

Correct join:

1. Trim each element; drop empties.
2. Treat an element as a paragraph boundary **only if** it ends in terminal punctuation: `[.!?]["')\]]*$`.
3. Otherwise append it to the current paragraph with a single space (it is a continuation fragment).
4. Run the merge pass over the result a second time (a continuation arriving after a merged fragment must still collapse).
5. Join final paragraphs with `\n\n`.

Measured effect: 43/47 -> **41/42 well-formed paragraphs (98%)** while keeping 3.5 paragraphs/narrative, and the quoted-code example renders back into correct prose. Single-element arrays and plain strings pass through unchanged.

## Retry policy

Retry **once** on ANY parse error, not only syntax errors. Evidence: the same prompt yields *different* corruption classes across runs (bogus key / enum case / empty body), which makes this transient sampling noise rather than a deterministic schema violation. Keep transport retries in the infrastructure layer; do not duplicate them. Also handle the empty-completion case (zero-length body).

## Non-semantic normalizations worth doing in the same pass

- **Case-insensitive enums.** If `confidence` is compared with `strings.ToLower` but `status` with `!=`, a capitalized `"Snapshot"` 502s the whole response. Normalize trim+lower for both validation and the stored value.
- **Never render a pipe-list as a JSON value.** A schema example like `"confidence": "high|medium|low|qualified"` makes the model echo `"high|qualified"`. Show ONE valid value; keep the allowed set in the rule text.
- **Finish-reason truncation.** Check both `"max_tokens"` and `"length"` — Anthropic emits `max_tokens`, OpenAI-compat emits `length`. Checking only one silently misreports truncation as a JSON parse error.
- **Truncate log previews by rune, not byte**, so multi-byte prose does not emit invalid UTF-8.

## Prompt hygiene for paragraph counts

- Do not state contradictory structural bounds (e.g. "3-6 sentences" AND "3-5 paragraphs" with a 120-word cap — jointly unsatisfiable for short prose).
- If you quote the separator in the prompt (`"\n\n"`), the model may echo the quote characters into the JSON structure. Prefer an array shape, or describe the escape in words.
- Keep word/paragraph bounds as ONE coherent set.

## Verify end-to-end, not just by unit test

1. Measure the real model, per-attempt, across several capabilities — effective failure with retry-once is roughly p^2.
2. Exercise the actual HTTP path (real auth, real request shape). Validation errors surface in order, so expect to fix date format and a contract-version field before reaching the handler.
3. Confirm **fresh** generation, not a cache hit, when claiming the prompt works.
4. Check the consuming UI actually renders the result: a field can be correct on the wire and still be invisible if the renderer collapses newlines (no `whitespace-pre-line`, no split). A `\n\n`-based contract needs a split in the component.
5. Containerized app with no source bind-mount: rebuild the image, then confirm the new string is actually in the shipped binary.
6. A character-at-a-time typewriter effect throttles to ~1-2 chars/sec in background browser tabs — never wait for it to finish; verify rendering through the component's own test suite.
