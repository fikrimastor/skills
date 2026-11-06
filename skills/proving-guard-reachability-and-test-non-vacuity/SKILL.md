---
name: proving-guard-reachability-and-test-non-vacuity
description: "Verify whether a defensive guard/catch/fallback is reachable or genuinely dead code, and prove a regression test actually fails without the fix — use when anyone (human, subagent, review comment) claims a branch is unreachable, or when adding a regression test for an error path."
---

# Proving guard reachability & test non-vacuity

Use when: someone claims a defensive branch is dead/unreachable; you are about to delete a guard; or you are writing a regression test for an error/catch path.

## 1. Do not reason from the "primary" failure path alone

The classic error is analysing only the obvious trigger.

> "The fetch rejects, so the catch runs with state X."

That is incomplete. A catch is reachable if **any** statement in the `try` can throw — including statements that throw *after* state has already been mutated.

Recipe:
1. Read the whole `try` block, top to bottom, including lines that look like cleanup or persistence.
2. For each statement ask: can this throw? (`localStorage.setItem`, `JSON.parse`, DOM/window APIs, property access on possibly-undefined, quota/private-mode storage, serialisation, I/O)
3. For each throwing candidate, ask: **what state was already mutated before it?** If a mutation precedes the throw, the catch runs with that mutation applied — that is the interesting state.
4. Classify: reachable-with-stale-state (a real bug the guard fixes) vs genuinely-null-on-entry (a genuine no-op).

Real example: `applyMeResponse(res)` set `user`; the next line `localStorage.setItem("auth-user", …)` threw on quota. The catch ran with `user` populated but `token` already nulled → `isAuthenticated` (`!!user.value`) true with no credentials. The guard was load-bearing; a fetch-only analysis called it a no-op.

## 2. Never overturn your own verified reasoning on an unverified claim

If you derived conclusion A from source, and an agent/reviewer asserts not-A, do not silently switch. Verify not-A to the same evidentiary standard, ideally by executing it. Confident phrasing is not evidence. (This was the actual failure in the case above: correct reasoning was reversed on a plausible-sounding agent report.)

## 3. Prove the test is non-vacuous by mutation

A regression test for a guard must FAIL without the guard. Demonstrate it — don't assume it:

```bash
# 1. GREEN with the guard
<test cmd>                    # expect pass

# 2. MUTATE: remove/neuter the guard
cp <file> /tmp/keep; <edit out the guard>
<test cmd>                    # expect RED, with the assertion naming the stale value

# 3. RESTORE and confirm GREEN
cp /tmp/keep <file>; <test cmd>
```

Record both outputs as evidence. If the test passes at step 2 as well, it is vacuous — rewrite it to target the reachable path (usually: make the *success* path succeed, then make the side-effecting line throw). Do not "fix" it by re-pinning the assertion text.

## 4. Red flags that a claimed-dead guard is real

- The assertion would hold both before and after the change.
- The test mocks use keys the production code never reads (a mocked success that never exercises the real branch).
- The test sits outside the `describe`/`beforeEach` scope that registers the mocks.
- The justification only names one failure trigger for a `try` with several throw sites.

## 5. Apply the same standard to coverage detectors

When asserting "every X is classified", a guessed allow-list/prefix list will produce false positives against reality. Prefer **behavioural probing** (drive the real thing and classify by observed outcome) over inferred membership. Prove the detector non-vacuous by injecting a known-bad case and confirming it goes RED.
