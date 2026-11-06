---
name: registry-derived-input-contract-doc-generator
description: "Generate and maintain a committed markdown reference doc that tabulates, per capability, what a registry-driven API requires to RENDER vs to ANALYZE, driven by a committed generator script with a staleness gate. Use when asked to produce reference tables for every capability/endpoint/plugin in a large registry, to keep such docs from going stale, or when a doc must enumerate where two independent registries disagree."
---

# Registry-derived input-contract doc generator

Produce a **committed doc whose every cell is generated from the running registries**, plus the
generator script and a `--check` staleness gate. Hand-typed tables rot; a generator makes
staleness a CI failure instead of a subtle lie.

## When this applies

Two independent registries describe the same entities with different vocabularies — typically
"how do I draw it" vs "how do I analyze it". The caller wants a per-entity reference table, and
the failure mode they care about is "the first contract is satisfied but the second silently
isn't."

## Procedure

### 1. Enumerate from the live registries, never from YAML by hand

```python
from <pkg>.registries.capabilities import list_capabilities   # authoritative count
from <pkg>.catalog import describe_capability                  # per-capability contract
from <pkg>.analysis.profiles import list_analysis_profiles     # the second contract
```

Join on the shared entity name. Print the three headline numbers first (registry size, second-
registry size, difference) and put them in the doc header.

### 2. Model reachability as a CLOSURE, not a set intersection

A naive `required_columns & registry_columns` test over-reports gaps. Three expansions needed:

1. **Accepted alternatives** — a stated requirement is not what actually renders.
2. **Derived-metric inputs** — if the contract derives X from A+B, then A and B are also
   reachable inputs.
3. **Option-supplied columns** — `x`/`y` options supply coordinate columns; `entity`/`category`
   supply identity columns.

Then iterate a **formula closure to fixed point**: any metric whose own inputs are all reached
becomes reachable. Without this, formula-backed metrics look unreachable and you publish false
gaps. (In practice this collapsed a 17-row false-positive set to the 3 genuine ones.)

**Critical asymmetry:** the render path may bridge two spellings of the same quantity; the
analysis path usually does NOT. So compute reachability from the RAW contract, never from an
alias-expanded set — feeding the expanded set makes alias targets look reachable and silently
empties the very gap class the doc exists to report.

### 3. Classify gaps by severity, and probe-verify each class

Two classes worth distinguishing:

- **ALIAS** — same quantity, different spelling; a render-path bridge exists but analysis does
  not use it. Actionable: send the analysis spelling explicitly.
- **CONTRACT** — genuinely different metrics; nothing bridges them. Needs a producer-side change.

Verify BOTH directions with a real call, not by reading config:

```
render-shaped frame            -> expect insufficient_data  (proves the gap)
frame + the missing metric     -> expect ready             (proves it is the only blocker)
```

### 4. Put a non-vacuity self-check in the generator

An always-False predicate silently publishes a doc with an empty gap column. Assert:

- each severity class is **non-empty**;
- a hand-picked list of known cases is in the expected class;
- known-good entities are **not** flagged;
- a clause-bearing feature (OR-of-AND groups) actually renders its clause.

Raise before writing the doc. This is the single highest-value part of the script — it caught an
inverted membership test and a dropped OR clause during authoring.

### 5. Make `--check` compare CONTENT, not bytes

If the header carries `generated: <timestamp>` or commit stamps, exclude those lines from the
comparison. Otherwise every run after the generation minute reports stale and the gate is
non-functional from birth.

```python
VOLATILE_PREFIXES = ("- generated: ", "- plot-api: ", "- asset-api: ")
def strip_volatile(text):
    return "\n".join(l for l in text.splitlines() if not l.startswith(VOLATILE_PREFIXES))
```

Verify the gate both ways before claiming it works: regenerate → `--check` must exit 0; perturb
one doc cell → `--check` must exit 1.

## Traps

| Trap | Symptom | Cause |
|---|---|---|
| Set-intersection reachability | Many false gap rows | Ignores alternatives, derived-metric inputs, option-supplied columns |
| No formula closure | Formula-backed metrics flagged | Metrics derivable from present inputs look absent |
| Alias-expanded reachability | Gap column is silently empty | The analysis path does not run the render alias bridge |
| Dict-membership direction | Predicate always False | Testing insight names against a render-keyed dict (use `.values()`) |
| Reading only the primary group field | OR clauses vanish from rows | A profile may declare a separate alternatives field |
| Units read from the full requirements map | Units column over-reports | The gate iterates *resolved* columns — one per group, not every declared alternative |
| Timestamp inside `--check` diff | Gate fails 100% of runs | Volatile header compared as content |
| Synthetic frames under-report | Real metrics look unrequired | The real producer emits helper columns; a hand-built frame omits them |
| Gate order | `missing_inputs` names the wrong thing | Context/scope checks return before metric checks — fix the context and re-run before calling it a contract gap |

## Verification gates

- Registry count from the loader == anchored regex count over the registry file. A naive
  `grep -c name:` over-counts (option/metric keys also match).
- The second registry must be a subset of the first (every analyzable entity is renderable).
- Every claim in the doc's prose sections must reproduce against a live call.
- The generator must not dirty the target repo — if it tracks `.pyc`, set `PYTHONDONTWRITEBYTECODE`
  and confirm no bytecode mtimes moved.
- Have the script re-exec into the dependency venv rather than relying on the caller to pick the
  right interpreter.
