---
name: tricipta-web-pr-local-review
description: "When reviewing a tricipta-edafy-asset-web PR read-only in a worktree (local summary only) or checking port parity."
---

# Tricipta Web PR — Read-Only Local Review

Use for `AFED-digital/tricipta-edafy-asset-web` (Nuxt 4 / Vue 3 / TS) when the user says **local only / don't push comments**. For the Go repo (`tricipta-edafy-asset-api`) use `tricipta-pr-local-review` instead — its `go build`/`go test`/testcontainers steps do not apply here.

## 0. Standing constraint

Deliver findings in chat. NEVER `gh pr comment`, `gh pr review`, `gh api` mutations, or `git push`. Read-only `gh`/`git` inspection is fine. Lead with a verdict, then `file:line` evidence per finding.

### Orchestration split (when dispatching a `reviewer` subagent)

`reviewer` agents are read-only and will refuse to create worktrees or run vitest/build. The **parent owns** §2 setup, §3 verification, and §8 cleanup. Spawn the reviewer, create the worktree, then `write agent://<id>` with the worktree path + head SHA and say "describe any probe test; parent runs it". Relay vitest/build results to the reviewer as they finish. Reviewer scouts sometimes read the dirty main checkout instead of the worktree / `origin/main` — tell it to cite only `git show <ref>:<path>` or worktree paths.

## 1. Recon before worktrees

```bash
gh pr view <N> --json number,title,state,isDraft,baseRefOid,headRefOid,headRefName,author,body,files
gh pr checks <N>                              # note the head SHA it ran on
gh run view <run-id> --json headSha           # confirm the green run is the CURRENT head
```

Read every issue the PR claims to close (`gh issue view <N>`), especially the acceptance criteria — they turn parity gaps into scoped findings.

## 2. Worktree setup (four traps)

```bash
cd <main-repo> && git status --porcelain       # record starting state
git fetch origin pull/<N>/head:pr-<N> --force
git worktree add /tmp/pr<N>-wt  pr-<N>      --detach
git worktree add /tmp/pr<N>-base <baseSha>  --detach   # for base-vs-head attribution
# node_modules: a post-checkout hook APFS-CLONES a real node_modules dir into new worktrees.
# Only symlink if absent — otherwise `ln -sfn` creates <wt>/node_modules/node_modules → duplicate Vue.
[ -e /tmp/pr<N>-wt/node_modules ] || ln -s <main-repo>/node_modules /tmp/pr<N>-wt/node_modules
cd /tmp/pr<N>-wt && npx nuxt prepare          # generates .nuxt/ — REQUIRED
```

- **Duplicate-Vue signature:** ~50+ unrelated tests fail with `TypeError: Cannot read properties of null (reading 'ce')` in `renderSlot` (reka-ui Dialog), stack shows two different `@vue/runtime-core` paths. Fix: `rm /tmp/pr<N>-wt/node_modules/node_modules`, rerun. Don't attribute these to the PR.
- Skipping `nuxt prepare` → every vitest file fails `TSConfckParseError: ... .nuxt/tsconfig.app.json ENOENT`.
- Never `git checkout` in the main worktree if it has uncommitted changes. Read base content with `git show <baseSha>:<path>`.

## 3. Verification (run yourself)

```bash
npx vitest run                                        # full suite, PR and base
NUXT_IGNORE_LOCK=1 NODE_OPTIONS=--max-old-space-size=8192 npx nuxt build
```

- Attribute every failure: run the same files on the base worktree, and re-run failing files in isolation. `test/pages/ConnectivityPage.integration.test.ts` is a known full-suite flake (fails on base full run, passes alone).
- The build OOMs at default heap (CI sets 8192). Duplicated auto-import `SchemaError` warning (connectivityPage.ts vs sensitivityPage.ts) is pre-existing.
- `npm run lint` is broken repo-wide (ESLint 9 vs ESM `.eslintrc.json`), CI `continue-on-error` — pre-existing, never a regression.

## 4. Runtime probes

`console.log` from a vitest test body is swallowed. `writeFileSync` render output from the test and `cat` it:

```ts
// test/components/zz-probe.test.ts  (delete before finishing)
const w = mount(Component, { props: { /* producer-shaped payload */ } });
writeFileSync("/tmp/probe.txt", w.html().replace(/\s+/g, " "));
```

Use it to prove which value wins when a component has a fallback/hardcode path. Delete the probe and confirm `git status --porcelain` clean.

## 5. Parity checks vs the uiux prototype (the authority)

Prototype: `~/Sites/afed/tricipta-uiux-asset/tricipta-platform/src/`. GitHub `user-attachments` URLs return 404 to curl — ground parity on prototype source. If a PR says "ported from X" but the component structure differs materially (e.g. prototype `DateRangeFilter.jsx` = preset buttons + native date inputs; port = dropdown + two-month calendar), flag as **parity divergence needing design sign-off**, not a defect.

### Inert CSS class

```bash
grep -rn "<custom-class>" app/ ; grep -rn "<keyframes>" app/assets/css/
npx nuxt build && grep -ro "\.<custom-class>" .output/public/_nuxt/*.css   # empty = inert
grep -rlo "shadow-2xs" .output/public/_nuxt/*.css                          # positive control
```

Tailwind v4 `@theme` here declares only `--radius-*`/`--color-*`; non-stock animation/shadow tokens are suspect.

### Numeric/threshold divergence & removed behaviour

Diff magic numbers vs prototype. Grep the PR body for vocabulary of any dropped affordance; silence plus an absence-asserting test = flag.

## 6. Cross-repo contract grounding

```bash
git -C <api-repo> show origin/main:<path>          # never checkout sibling repos
git -C <api-repo> merge-base --is-ancestor <ref> origin/main && echo MERGED || echo NOT_MERGED
```

Unmerged producer + consumer reading new fields = deploy-order dependency; state it. Compare producer's emitted ID/label set against consumer literals. Check producer **denominators** (e.g. VPS health history: asset-api `HealthHistoryBuckets=48`, bucket % = non-imputed rows / rows present via `width_bucket`, so missing 10-min rows are never counted and null-observed rows with `is_imputed=false` count as up). A web change that switches to consuming such history inherits that semantics.

### Date-range boundary traps (recurring in VFM/VPS filters)

- `Date.setMonth(+3)` rolls Jan 31 → May 1; month limits need clamping.
- `computeVfmCustomTimeRange` adds +1 s to `to`; an exclusive `D2+1 00:00` end becomes inclusive, and shifts API `width_bucket` fractional boundaries vs whole-second segment labels.
- Preset-matching by calendar date alone can swallow a custom range into a preset window.

## 7. Test-quality check

For each new test: would it still pass if the fix were reverted? Check PR-body claims ("reset on failure is tested") against actual test names. Flag tests whose name encodes the defect, and composable tests that only mutate refs without calling the function under test.

## 8. Cleanup — then VERIFY it

```bash
rm -f /tmp/pr<N>-base/node_modules                     # symlink only; cloned dirs are removed with --force
cd <main-repo> && git worktree remove /tmp/pr<N>-wt --force
git worktree remove /tmp/pr<N>-base --force
git worktree prune && git branch -D pr-<N>
git branch --list 'pr*'; git worktree list; git status --porcelain   # compare to starting state
```

Leave other pre-existing `/tmp/*` worktrees alone and mention them.

## 9. Report shape

Verdict → verification table (ran / passed / pre-existing failures with attribution) → defects (VERIFIED / LIKELY / RISK / NIT, `file:line`, evidence, impact) ordered by live impact (always-on path outranks dev-only/flagged path) → test gaps → verified-sound list → design sign-off items.
