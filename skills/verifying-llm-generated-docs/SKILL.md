---
name: verifying-llm-generated-docs
description: "Verify LLM- or vendor-generated documentation corpora (repo wikis, architecture docs, generated markdown) before trusting or shipping them — stub-endpoint testing with planted defects, per-page freshness anchors, and the traps that make validators silently inert or corrupt quoted code."
---

# Verifying LLM-generated documentation

Generated docs lie in ways that look authoritative. Every validator you add will
silently no-op at least once, so **prove each one fires with a planted defect
before you trust a clean report**.

## 1. Prove validators with a stub endpoint, not with real calls

Run a tiny local HTTP server that returns canned responses, point the generator at
it (`LLM_BASE_URL=http://127.0.0.1:PORT`), and queue responses so you can plant
defects per case:

- a **valid** item (must survive),
- a **fabricated** one (past-EOF line range, invented filename),
- an **empty** response and a `finish_reason: "length"` response,
- a response that omits a required element (e.g. no citations) to trigger retry.

Assert on the request body too (model id, token budget, auth header) — payload bugs
(missing `model`, wrong budget) never show up in the output otherwise.

## 2. Traps that burned real time

1. **Same basename, different directory.** `domain/x.yaml` and
   `plotting_core/domain/x.yaml` both existed with *different* headers. Resolve every
   claim against the file in the **declared source list**, never a path that appears
   in the generated prose. Grepping the wrong one produced a confident, false
   "hallucination" verdict.
2. **Markdown transforms must be fence-aware.** `re.sub(r"^#\s+", "## ", body, flags=re.M)`
   rewrites `# comment` lines inside ```yaml/```bash fences and corrupts quoted
   samples. Split on fences; rewrite only outside.
3. **Regex substring matches.** `\.(?:py|go|sql|sh)` matched `chan_diagnostics.gor_…`
   and `${{ github.sha }}`. Require a trailing boundary `(?![A-Za-z0-9_])`.
4. **Trailing punctuation in link capture.** `[^)\s]+` swallowed the closing backtick
   of `` `file://x#L1-L2` `` → every such citation verified as "dead". Exclude the
   punctuation the prose uses.
5. **Single-item tests hide per-item bugs.** A per-page budget bound to a variable
   that was only set for the *last* item passed a one-page test. Always test ≥2 items
   and assert their outputs **differ** appropriately.
6. **"0 findings" is ambiguous.** A counter that increments before the lookup
   reports "N verified" while verifying nothing. Increment only on actual
   verification; report `unresolved` separately; treat a suspicious 0 (no citations
   emitted) as itself a defect — retry once.
7. **Never manufacture precision.** Clamping an out-of-range citation down to the
   file's last line turns a fabrication into a plausible in-bounds cite and stops the
   drift metric from ever firing. Degrade to the file-level cite and count it.
8. **Skip partial grounding loudly.** A source-budget overflow that silently drops
   files produces a thin page that looks complete. Report the omitted files, and
   record the ratio in the artifact's own header so a later pass can flag it.
9. **Truncation must not mark content fresh.** Write the artifact and advance its
   freshness marker only on a clean finish; otherwise skip, leave the marker, exit
   non-zero.

## 3. Freshness model that survives partial runs

Store a **per-item anchor** (the commit each item's content came from) — never a
per-repo anchor. Otherwise regenerating one page marks 39 stale siblings clean.
Detect staleness as `cited files ∩ files changed since that item's anchor`, and
report the oldest anchor as the headline lag.

## 4. Make the artifact self-reporting

Write findings into each artifact's provenance header (unverified code blocks,
invented paths, partial grounding) and have the checker read those headers. That is
what catches rot in a corpus nobody is watching live, and it costs one regex.

## 5. Cost the prompt you actually send

A dry-run that assembles the real prompt and prints token counts is the only honest
estimate. Price it with the provider's published rates, label assumptions, and never
price the token *ceiling* as if it were output — that overstates by ~10×.
