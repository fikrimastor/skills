---
name: spec-open-item-propagation-precision-pass
description: "Verify a spec/plan open-item propagation pass for over-broad claims, false staleness advisories, and unverified citations before reporting done. Use after propagating reviewer answers across derived artifacts (markdown, generated HTML, dashboards), or when checking whether a generated artifact is actually stale."
---

# Precision pass after propagating spec/plan open-item answers

Run this AFTER the base propagation sequence (edit spec, plan, regenerate derived artifacts). It catches the three failure classes that survive a normal propagation pass and get caught later by a reviewer.

## 1. Refute or confirm "artifact is stale" before regenerating

A staleness claim is cheap to check and expensive to obey wrongly.

```bash
# Check BOTH directions: new string present, old string absent
grep -c "NEW_CITATION"  path/to/artifact.html
grep -c "OLD_CITATION"  path/to/artifact.html

# Confirm the rendered file matches its IR
grep -o "SYMBOL:[0-9-]*" artifact.html | sort | uniq -c
grep -o "SYMBOL:[0-9-]*" artifact.json | sort | uniq -c
```

Ordering matters: a `deliver`/render that runs **after** the IR edit already reflects it. Regenerating on a false premise churns hashes and can clobber a parallel edit. Only re-render when the old string is actually present.

## 2. State every claim at exactly the level the evidence supports

Audit each answer you wrote for two overclaim shapes:

- **"X is clean / not an offender"** — can you actually read X's value? A `secretref:` (or any opaque indirection) resolves server-side and is **UNVERIFIED** from the workspace. Say so, and flag explicitly that a follow-up must **not** be scoped to one side on that evidence.
- **"verified on all N"** — re-count the sample from the artifact text. A "both directions" or "all routes" claim is often true of a named subset only. Name the subset and its size inline.

Grep for the patterns that signal an overclaim:

```bash
grep -rn "like the other\|all [0-9]* .*both directions\|sole offender\|truthful today\|is clean" --include=*.md --include=*.html .
```

## 3. Re-derive line-range citations against source before propagating

Citations written from memory land in every derived artifact at once.

```bash
# Pin the exact line, don't trust recall
grep -n "SIGNATURE_STRING" path/to/source.go

# Then sweep every artifact, and regenerate any generated HTML
grep -rn "old-range" . --include=*.md --include=*.json --include=*.html
sed -i '' 's|file\.go:OLD-RANGE|file.go:EXACT-LINE|g' <artifacts...>
# re-run validate + deliver for any generated HTML whose IR you touched
```

## 4. Merge-race check when a subagent shares the artifact

If a subagent owns a derived artifact and you edit the same file:

1. After both edits, verify **both** survived — grep for each side's distinctive text, do not assume.
2. Re-render the HTML yourself from the merged IR (the subagent's render may predate your edit).
3. Message the subagent to stand down so it cannot re-deliver over your merge.
4. Record the final hashes.

## 5. Close the pass with a phrase sweep + a render check

```bash
# Stale phrasing should survive only when explicitly labelled historical/superseded
grep -rn "OPEN — \|is unanswered\|one gate remains\|awaiting approval" --include=*.md --include=*.html .
```

Then open each generated artifact in a browser and assert the DOM/text: decided vs open row counts, the newly-closed items present, the corrected strings present, the old strings absent. A markdown edit is not proof that the rendered artifact changed.

## Reporting

State per artifact: what changed, the validation result (checks passed, error/warning counts), and the final hash. Explicitly separate **claims you verified** from **claims you could not verify** — and name the latter as UNVERIFIED rather than omitting them.
