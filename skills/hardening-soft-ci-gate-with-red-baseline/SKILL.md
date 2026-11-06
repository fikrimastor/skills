---
name: hardening-soft-ci-gate-with-red-baseline
description: "When a continue-on-error CI step must actually block a deploy gate, with a red baseline."
---

# Hardening a soft CI gate that feeds a deploy pipeline

Use when a deploy is gated on CI but the test job is soft (`continue-on-error: true`,
`|| true`, allowed failure), so "green CI" does not mean "tests pass".

The gate this job feeds is built per `github-actions-gate-deploy-on-ci-success`.

## The trap that costs the most

**Never flip the flag without first checking whether the suite is already green.**
A soft gate can stay green for months while the suite is red. Flip it and every run
turns red → the deploy gate never passes → you have silently frozen all deploys.
That is strictly worse than the soft gate you replaced.

## Procedure

1. **Establish the real baseline before touching anything.**
   `continue-on-error` masks the step's true conclusion, so a green check proves
   nothing. Read the *step log*, not the check badge:
   ```bash
   gh run view <run-id> --repo <owner>/<repo> --log | grep -E 'FAIL|exit code'
   ```
   Confirm the failure is pre-existing by finding an older run on the default branch
   that fails identically. Never attribute it to the PR you are reviewing.

2. **Repair every pre-existing failure first**, in a throwaway worktree so the user's
   dirty working tree survives:
   ```bash
   git fetch origin <branch> && git worktree add /tmp/wt origin/<branch> --detach
   ```
   Run the suite there with the runner's exact flags (e.g. `go test -race ./...`).

3. **Classify each failure before fixing it.** Two recurring kinds:
   - *Stale assertions pinning a producer's literal tokens* (rule labels, enum strings,
     diagnostic codes that were renamed). Re-anchor on **stable** text that survives
     renumbering, or assert observable behavior. Do NOT edit production code to
     re-add the old token, and do NOT delete or `t.Skip` the test — that destroys the
     coverage you are trying to gate on.
   - *Tests requiring infra nothing supplies* (live DB, secrets). Fix them the way the
     repo already fixes that class of test — look for an existing `TestMain` /
     `testcontainers` fixture in a sibling package and mirror it. Add a real fixture;
     do not weaken the test into a skip.

4. **Flip only the step you were asked about.** Removing `continue-on-error` from a
   lint step with a large existing finding backlog is a separate project; it would
   red every push for unrelated reasons. State explicitly which step you left soft.

5. **Re-check side effects of the step waking up.** A step that was soft may contain
   follow-on gates (coverage thresholds, artifact uploads) that have *never executed*,
   because a failing earlier command aborted the script first. Verify each now passes.

6. **Prove the gate bites** — inject a deliberate failure, confirm nonzero exit, revert:
   ```bash
   cp <file> /tmp/bak && printf '\nfunc TestDeliberateFailure(t *testing.T){ t.Fatal("gate proof") }\n' >> <file>
   go test ./pkg/ >/dev/null 2>&1; echo "exit=$?"   # expect 1
   cp /tmp/bak <file>
   ```

7. **Verify end-to-end in CI, not just locally.** Push, then confirm the *step*
   conclusion (now authoritative) is `success`, and that the log shows the tests really
   ran — look for per-package `ok` lines and coverage output, so you know the gate is
   non-vacuous.

## Reporting

State: which step changed, which stayed soft and why, the baseline failures you
repaired, local result before/after, and the CI step conclusion on the new commit.

## Pitfalls

- A skipped dependent job inside a `workflow_run`-triggered run can still leave the
  **run conclusion `success`** — confirm your deploy gate's actual predicate
  (`conclusion == 'success'`, branch filter, event filter) rather than assuming
  skip ⇒ blocked.
- Editing files from two agents in one shared worktree: give each an exclusive file,
  forbid repo-wide `go test ./...`, and verify each claimed edit yourself.
- Clean up the worktree and then *re-verify* the cleanup (`git worktree list`,
  `git branch --list`); a cleanup command can fail silently.
