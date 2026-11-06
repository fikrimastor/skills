---
name: typescript-baseline-error-attribution
description: "Attribute new-vs-pre-existing TypeScript/vue-tsc errors in a repo with a large existing error baseline, and prove a fix removed exactly the intended errors. Use when reviewing whether a branch/PR introduced type errors, when typecheck is not CI-gated (continue-on-error or absent), or when you must show a fix's precise delta."
---

# Attributing typecheck errors against a large baseline

For repos where the type checker is red by default (`continue-on-error`, no gate, or hundreds of pre-existing errors), a raw error count proves nothing. You must diff the *error set* against the merge-base.

## Setup: a baseline worktree

```bash
REPO=/path/to/repo
BASE=$(git -C $REPO merge-base origin/main HEAD)
git -C $REPO worktree add --detach /tmp/base $BASE
# The checker needs generated types; a fresh worktree has none:
ln -sfn $REPO/node_modules /tmp/base/node_modules
cp -R $REPO/.nuxt /tmp/base/.nuxt          # or: cd /tmp/base && npx nuxt prepare
```

Run the SAME command in both trees, capturing output:

```bash
cd $REPO      && npx vue-tsc --build --force > /tmp/head.txt 2>&1
cd /tmp/base  && npx vue-tsc --build --force > /tmp/base.txt 2>&1
grep -c 'error TS' /tmp/head.txt /tmp/base.txt
```

## Pre-flight: prove the toolchain is identical

Before trusting ANY delta, confirm both trees resolved the same checker. A silently `npx`-fetched or missing version invalidates the comparison.

```bash
for d in $REPO /tmp/base; do echo "$d"; \
  node -e "console.log('vue-tsc',require('$d/node_modules/vue-tsc/package.json').version)"; \
  node -e "console.log('tsc',require('$d/node_modules/typescript/package.json').version)"; done
ls /tmp/base/.nuxt/tsconfig.app.json   # must exist
```

If versions differ or `.nuxt` is absent, fix the setup and re-run; do not report the delta.

## Normalize to signatures, then diff

Line numbers shift with any edit, so strip them; keep file + code + message.

```bash
norm() { sed 's/^\([^:]*\)([0-9]*,[0-9]*): /\1: /' "$1" | grep -E '^(app|src|test)/' | sort; }
norm /tmp/head.txt > /tmp/head.sig; norm /tmp/base.txt > /tmp/base.sig
comm -23 /tmp/head.sig /tmp/base.sig   # NEW errors (not in baseline)  <-- the ones that matter
comm -13 /tmp/head.sig /tmp/base.sig   # errors the change FIXED
```

## Two traps that will make you report a wrong number

**Trap 1 — a still-present new error is invisible to `comm`.** After a partial fix, an unfixed new error exists in BOTH head and base sets, so `comm -23` does not show it and the count looks clean. Never conclude "all new errors fixed" from `comm` alone. Use per-file counts as the decisive check:

```bash
for f in path/to/edited1.ts path/to/edited2.vue; do echo "$f -> $(grep -c "^$f(" /tmp/head.txt)"; done
```

Zero in every touched file is the real acceptance criterion.

**Trap 2 — do the arithmetic, don't eyeball "close to baseline".** If the branch introduced N new errors and you fixed all N, the post-fix total MUST equal `baseline_total + (remaining_new)`. With no remaining, it equals the pre-change total exactly. A one-off discrepancy means exactly one thing: an unfixed new error. Chase it to a file:line+code rather than accepting a near-miss.

## Proving a fix is real (revert-sensitivity)

For each new regression test, confirm it FAILS against the pre-fix source. Copy only the test files into a worktree pinned at the original commit, leaving source reverted:

```bash
cd /tmp/pre && git checkout <old-commit> -- <source files> && rm -f <newly-added-source>
npx vitest run <the new test files>     # expect FAILURE
```

A test that passes pre-fix is a false green and proves nothing. Report the observed failure text as evidence.

## Cleanup

`git worktree remove /tmp/base --force` for every throwaway, then `git worktree list` to confirm none remain. Never leave the user's repo with stray worktrees or modified source.

## Notes

- `noUncheckedIndexedAccess` makes `arr[i]` and `str[i]` yield `T | undefined`. Prefer `str.charAt(i)` (always `string`) for bounds-guarded string access, or a real falsy/narrowing check — never `!` or `as`.
- A narrowing that does not survive `||`/`&&` (`typeof x?.f === 'string' || x?.f === null ? x.f : null`) errors on the optional chain. Hoist to a local and narrow once.
