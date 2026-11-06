---
name: verify-rebase-before-force-push
description: Prove a rebase of a published git branch was content-neutral before force-pushing. Use when git push is rejected non-fast-forward after a rebase/pull --rebase on a branch that already exists on the remote.
---

# Verify Rebase Before Force Push

## When

`git push` rejected `non-fast-forward` on a branch that already exists on the
remote, after the rebase already succeeded (reflog shows
`pull --commit --rebase` / `rebase (finish)`). The push fails because the rebase
rewrote every commit SHA — not because the remote has content you lack.

## Procedure

### 1. Confirm the failure is rewritten history, not remote-ahead content

```bash
git status -sb                      # expect: ahead N, behind M
git ls-remote origin refs/heads/<branch>
git rev-parse origin/<branch>       # must MATCH the ls-remote SHA
git merge-base --is-ancestor origin/main HEAD && echo REBASED
```

`ls-remote` == local tracking ref means **no third-party commits** landed on the
branch — this is what makes `--force-with-lease` safe. Capture `old_tip` (the
remote SHA) from this step; the local reflog has it too.

### 2. Per-commit proof

```bash
old_base=$(git merge-base origin/main "$old_tip")
git range-diff --no-color "$old_base".."$old_tip" origin/main..HEAD
```

`=` = byte-identical patch. `!` = patch differs — **investigate, do not assume loss.**

### 3. Resolve each `!` (usually context-only)

When upstream main reformatted lines that a branch commit touches, hunk
boundaries shift and range-diff prints `!` even though the committed content is
identical. Prove it at the file level:

```bash
F=path/to/file
 git rev-parse <old_commit>:$F
 git rev-parse <new_commit>:$F          # equal => context-only
 diff <(git show <old_commit>:$F) <(git show <new_commit>:$F)   # empty => identical
```

A string vanishing from the printed patch (e.g. a renamed label) is **not**
evidence of loss when the enclosing element's formatting changed upstream.

### 4. Definitive net check

```bash
for f in $(git diff --name-only origin/main..HEAD); do
  [ "$(git rev-parse $old_tip:$f)" = "$(git rev-parse HEAD:$f)" ] || echo "DIFFERS: $f"
done
```

All branch-owned files must match. This is stronger than `range-diff` and is the
invariant to quote as the evidence.

### 5. Push

```bash
git push --force-with-lease
```

## Traps

- **`git diff A..B | git hash-object` across a rebase is meaningless.** Different
  bases legitimately change hunks; a mismatch is not a red flag. Do not use this
  as the proof.
- **`git patch-id` also changes when context lines change.** Do not treat a
  patch-id mismatch as content loss; blob equality is the stronger invariant.
- **Tree-hash inequality between old tip and HEAD is expected** — HEAD's tree
  includes upstream's intervening commits. Not a content delta.
- **Never bare `--force`.** `--force-with-lease` aborts if someone else pushed;
  bare `--force` silently destroys their work.
- Force-pushing a branch with an open PR rewrites review anchors; previously
  submitted review comments may render as outdated. Tell the user.
