---
name: typecheck-regression-attribution
description: "Decide whether a PR introduced type errors or fixed them, in a repo carrying a large pre-existing typecheck-error baseline (tricipta-edafy-asset-web and similar Nuxt/vue-tsc repos). Use when a reviewer claims \"this PR causes TS errors\", when asked to verify a code-review claim about types, or before claiming a PR fixes/regresses typecheck."
---

# Typecheck regression attribution

Answering "did THIS PR add those TS errors?" in a repo with hundreds of pre-existing
errors. Raw counts mislead; you need a base-vs-head *signature* diff under an
identical toolchain.

## Procedure

1. **Pin the revisions.** `git rev-parse HEAD` and `git merge-base origin/main <head-branch>`.
   Confirm the local tree actually equals the PR tip (`git rev-parse origin/<branch>`) —
   if not, work in a worktree at the real tip.

2. **Run head typecheck.** Capture to a file, keep the exit code:
   ```bash
   npx vue-tsc --build --force > /tmp/tsc-head.txt 2>&1; echo "EXIT:$?"; grep -c "error TS" /tmp/tsc-head.txt
   ```
   `--build` is REQUIRED in Nuxt repos: the root `tsconfig.json` is project-references
   only (`"files": []`), so `-p tsconfig.json` checks nothing and falsely reports 0 errors.

3. **Baseline in a worktree.** Fresh worktrees lack generated types and deps:
   ```bash
   git worktree add /tmp/base <merge-base>
   ln -sfn "$PWD/node_modules" /tmp/base/node_modules   # reuse, don't reinstall
   cp -R .nuxt /tmp/base/.nuxt                          # generated tsconfigs
   ```
   **Verify the baseline resolved the PINNED toolchain** — `node -e "console.log(require('./node_modules/vue-tsc/package.json').version)"`
   in both trees must match. A `npx`-fetched or differently-resolved vue-tsc/TS silently
   invalidates the whole delta. Also confirm `.nuxt/tsconfig.app.json` exists in the
   baseline; without it the run is meaningless.

4. **Diff error SIGNATURES, not counts.** Strip line:col so moved code still matches:
   ```bash
   sed 's/^\([^:]*\)([0-9]*,[0-9]*): /\1: /' tsc-head.txt | grep -E "^(app|test)/" | sort > head.sig
   sed 's/^\([^:]*\)([0-9]*,[0-9]*): /\1: /' tsc-base.txt | grep -E "^(app|test)/" | sort > base.sig
   comm -23 head.sig base.sig   # NEW errors → the PR's regressions
   comm -13 head.sig base.sig   # FIXED errors
   ```
   Report all three numbers: total at base, total at head, and the new/fixed counts.
   A net *decrease* in total can still hide real new regressions.

5. **Prove the proposed fix.** In a worktree at the head commit, apply ONLY the suggested
   change, re-run, and check the error count drops by exactly the expected amount. This
   converts "the reviewer's fix sounds right" into measured fact. `git worktree remove <path> --force`
   + `git worktree prune` afterwards.

## Traps

- **Runtime vs type severity.** TS is erased. If the suite is green, a deleted interface
  field may have **zero runtime impact** — a type-contract break, not a broken feature.
  Check whether a `typecheck` script exists and whether CI gates on it (`continue-on-error:
  true` means it currently does not). Say which it is; don't inflate.
- **Name-collision types.** Grepping a field name misattributes which interface broke.
  Resolve each read to its enclosing type declaration. (This repo has two `InsightSlot`
  types; see the paired memory.)
- **Stale expected-shape tests.** A `toEqual({...})` literal on a changed interface is a
  type error but is not a construction site; count it separately from real literals.
- **Don't double-run heavy checks.** Full `vue-tsc --build` takes ~13 s; run head, base,
  and fix once each, in parallel where independent.
