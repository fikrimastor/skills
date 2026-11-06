---
name: auditing-consumer-guard-against-producer-payload
description: "Decide whether a consumer-side guard, status rule, or spec clause can ever fire against real producer output — and stop tests that pass against fabricated inputs. Use when a spec or implementation adds a branch keyed on a producer field, diagnostic code, or severity; when a test asserts a fixture's supposed properties; or before claiming a defensive branch is covered."
---

# Auditing a consumer guard against the producer's real payload

A consumer branch (status rule, downgrade, fallback, spec clause) can be dead
in production while its test passes, because the test fabricates the input the
producer never emits. This has produced repeated false-greens; treat it as a
defect class, not bad luck.

## The three shapes it takes

1. **Unemittable field.** The clause compares against a payload field the
   producer never sends (a coverage `minimum_points` when the producer emits
   only `points`/`entities`). Unimplementable at the consumer; remove the clause
   or make the producer emit it.
2. **Unmatchable predicate.** The guard matches a substring or value that no
   real emission satisfies — e.g. matching `"anomal"` in diagnostic codes when
   none of the emitted codes contain it. The test fabricates a matching code.
3. **Wrong fixture property.** The fixture is asserted to have a property it
   does not have, so the branch under test is never actually reached (a series
   asserted to classify as `flat` that classifies as `rising`).

## Procedure

1. **Enumerate the producer's real emissions.** Do not reason about what it
   "should" emit. For diagnostic/error codes:
   `grep -rho 'code="[a-z_]*"' <producer>/ | sort -u` — then check each guard
   predicate against that literal set. If nothing matches, the branch is dead.
2. **Trace the early-return ordering.** A guard may be unreachable because the
   producer returns earlier (an `insufficient_data` path that exits before the
   field would be read). Reachability is about the whole path, not the field.
3. **Run the fixture through the real function before asserting on it.** Print
   the actual output, then write the assertion. Never assert a fixture's
   supposed property from reading the code.
4. **Prove non-vacuity by disabling the producer side**, not the consumer: comment
   out the emission/detection line, run the test, capture the failing assertion,
   restore. A test that still passes is testing nothing.
5. **Match literal codes, not substrings.** A substring rule silently never
   fires; a literal match fails loudly when the producer renames. Comment the
   constant with why.

## Engine traps worth re-checking (Tricipta plot API, and analogues elsewhere)

- **Robust slope is a median of pairwise slopes.** Theil–Sen (`_theil_sen_slope`)
  means one mid-series excursion injects large slopes of both signs and pushes
  the median off zero → classifies as `rising`, not `flat`. A `flat` verdict
  needs a *majority* of near-zero pairs (e.g. ten flat months then a two-month
  doubling). Verify with a probe before writing the test.
- **Anomaly detection needs MAD > 0.** `_detect_anomalies` returns `[]` when the
  median absolute deviation is zero, so constant or cleanly-monotone series can
  never flag, however extreme the outlier. To get both `flat` and an anomaly, use
  an alternating-jitter flat series with one spike:
  `[101,99,101,99,101,99,101,99,5000,99,101,99]` → `flat`, 1 anomaly.
- **Two bases in one feature block.** `latest` may come from the raw observed
  series while `change`/`trend` come from an anomaly-filtered series. They are
  allowed to disagree; do not "reconcile" them, and do not assert
  `latest == change` endpoint.

## Reporting

State the producer's enumerated emission set, the guard predicate, and whether
any element satisfies it. A branch with no satisfying emission is a finding,
whether or not its test is green.
