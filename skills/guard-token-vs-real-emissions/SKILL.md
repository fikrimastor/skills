---
name: guard-token-vs-real-emissions
description: "Use when adding or reviewing any guard that matches a literal token (diagnostic code, enum value, error code, status, event name, flag) against a payload produced elsewhere — to detect the false-green where the token is never actually emitted."
---

# Guard token vs. real emissions

A guard matches a specific literal: a diagnostic code, enum value, error code, status string, event name, metric key, feature flag. The token came from your head or from a fixture you wrote. The producer emits a different, closed set. The guard compiles, the test passes, the branch never fires in production.

This class is invisible to code review of the guard itself, because the guard reads correctly.

## Detection

1. **Enumerate the real set.** Grep the producer for every literal it can emit:
   ```bash
   grep -rho 'code="[a-z_]*"' <producer_dir> | sort -u
   ```
   Do this for enums, error codes, status values, event names — whatever the guard keys on.
2. **Diff against the guard's token.** Not present in the set → the guard is dead.
3. **Check for an invented token.** If the token appears only in the guard and in tests, nobody emits it.

## Repair

Choose one, and state which:

- **Match reality** — rewrite the guard against a token that is genuinely emitted.
- **Emit the token** — have the producer emit it. Prefer this when the state is real and simply unexpressed.

If you emit, two further traps:

- **Cover every path that can create the state**, not just the branch you were looking at. Enumerate the producer's shapes/callers first. A requirement satisfied for 1 of N shapes is still broken for N-1.
- **Emit at the single decision point**, not per-branch. If the state is derived from a shared upstream choice, the code that makes that choice owns the emission — otherwise every future shape silently misses it.

Avoid substring matching on code/message text. Match the literal. Substring rules pass against a fabricated fixture and never fire against the real vocabulary.

## Verification

4. **Verify the fixture through the real path before asserting on it.** Print the actual computed value first. Reasoning about which fixture reaches a branch is unreliable — pairwise medians, MAD==0 guards, and minimum-length gates routinely invalidate a fixture that looks correct.
5. **Assert non-vacuity.** Disable the emission, watch the test fail with the expected error, restore, watch it pass. A test that has never failed proves nothing.
6. **Assert the count.** `len(matches) == 1` catches a double-emission introduced by leaving an older emission site in place.

## Signals you are in this trap

- The fixture constructs the token by hand.
- The guard matches a substring of a code or message.
- The token was added to a test but not to the producer.
- One guard requirement exists for N shapes but only 1 was wired.
- Every test exercises the same input as the original bug report.

That last one is the strongest tell. The detector is not re-reading the guard — it is feeding a different input: another capability, another basis, another shape, another caller.
