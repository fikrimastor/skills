---
name: reviewing-ci-deploy-gate-prs
description: "When reviewing a PR that gates CD on CI via workflow_run, or adds a duplicate-run skip-guard."
---

# Reviewing CI/GitHub-Actions deploy-gate PRs

Use when asked to review a PR that (a) adds a `push` trigger to CI, (b) converts CD from `on: push` to `on: workflow_run` on CI, or (c) adds a job that skips duplicate CI runs when a PR is open. Goal: replace plausible-sounding YAML reasoning with observed evidence.

The construction procedure under review lives in `github-actions-gate-deploy-on-ci-success`; read it first so a
review does not re-derive the traps.

## 0. Read the PR bodies first

These PRs usually carry an unusually honest "Notes for review" section. It names its own soft spots (which steps are `continue-on-error`, that the trigger needs the file on the default branch, that the workflow name is the gate key). Mine it, then verify each claim — do not restate it as evidence.

## 1. Prove the skip actually fires (the highest-value check)

The whole feature is "skip rather than cancel". Skipped jobs still produce a workflow-level conclusion, and a `conclusion == 'success'` gate depends on it.

```bash
gh pr checks <N> --repo <owner>/<repo>          # look for: build-and-test  skipping
gh run view <pushRunId> --repo <owner>/<repo> --json conclusion,event,headBranch -q .
gh run view <pushRunId> --repo <owner>/<repo> --log | grep 'guard' | grep -i 'skip:\\|open PR count'
```

Required evidence: the guard log shows the concrete count (e.g. `skip: 1 open PR(s) target a deploy branch`), the dependent job reads `skipping`, and the run conclusion is `success`. If the guard's fallback branch fired instead, the log shows `open PR count=''` — that means the lookup failed, and the skip never happened.

## 2. Prove the deploy gate cannot misfire

Inspect four things in the head file, not the diff:

- The gate `if:` must include `github.event.workflow_run.event == 'push'`. Without it, a PR-triggered CI completion deploys an unmerged commit.
- Branch/SHA derivation must read `github.event.workflow_run.head_branch` / `.head_sha` — `workflow_run` runs the file from the **default branch**, so plain `github.ref_name`/`github.sha` silently target the wrong environment (a `staging` push deploys `dev`). Fallbacks for `workflow_dispatch` are expected and correct.
- The guard must exempt deploy branches (`main|staging|uat`) **before** any lookup, or a promotion PR can stall a deploy-branch run.
- `on.workflow_run.workflows: ["<name>"]` matches the upstream `name:` field, not the filename. Confirm the name exists in the repo (`grep -m1 '^name:' ci.yml`); a rename silently kills CD.

## 3. Score the gate's actual strictness

A green run is only as strong as its strictest blocking step.

```bash
grep -c 'continue-on-error: true' ci.yml
grep -n 'continue-on-error\|run:\|name:' ci.yml
```

If Tests or Lint is `continue-on-error: true`, the deploy gate is "CI finished", not "tests passed" — flag it as the one substantive decision, with the exact `file:line`. Expect this to differ per repo in the same PR family; check each independently.

## 4. Confirm `branches: ["**"]` covers the bare default branch

Docs cheat sheet: `'**'` = "Matches all branch and tag names". Quote it rather than arguing from intuition. `branches` on `workflow_run` filters on the branch the *triggering* workflow ran on (workflow-syntax doc), not on where CD lives.

## 5. State the unverifiable honestly

A `workflow_run` CD trigger cannot fire before the new file is on the default branch. Check `gh run list --workflow="CD Pipeline" --json event,conclusion` — if every run is `event=push`, the trigger is unexercised: correct by docs, empirically untested. Say so, and name the post-merge check.

## 6. Traps that produce wrong verdicts

- **Reading `main` instead of the PR head** for `permissions:`. A head file that adds `pull-requests: read` alongside `contents: read` has no auth gap — `gh pr list` needs that scope because a top-level permissions block zeroes unlisted scopes. Always fetch `?ref=refs/pull/<N>/head`.
- **Per-commit archaeology without chronology.** A commit that looks reverted may be a deliberate revert by the next commit. Check parent order: an intervening commit superseding it means the net diff is intended, not a defect.
- **CRLF claims.** Verify by counting bytes: `gh api ... | base64 -d | tr -dc '\\r' | wc -c` on both base and head.
- **Branch-keyed concurrency groups.** `group: ${{ github.workflow }}-${{ github.ref }}` is correct when a skip-guard exists; keying on the branch instead puts the push and PR runs in one group, so `cancel-in-progress` cancels the PR check — the exact failure the guard exists to avoid. Do not recommend "restoring" branch-keyed grouping.
- **Env-var interpolation in `run:`.** `${{ github.* }}` interpolated into a shell script is a script-injection vector on a branch name. Correct form passes values through `env:`; verify the guard does this.
- **Branch protection.** `gh api repos/<o>/<r>/branches/main/protection` 403s on non-Pro private repos. Absence of protection means the "cancelled check reads as failure" rationale is moot — note it as non-blocking, don't report an error as a finding.

## 7. Report shape

Verdict first, then: verified-working table (claim → live evidence) → the one substantive fix with `file:line` → explicitly-unverified list → nits. Never post GitHub comments unless asked; local summary only for Tricipta/AFED repos.
