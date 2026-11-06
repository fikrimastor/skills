---
name: git-split-commit-to-new-branch-safely
description: "Move an already-made commit off the wrong branch onto a new branch, when the working tree also has an unrelated uncommitted change that must survive a git reset --hard. Use when a fix landed on a branch that's a separate PR-in-waiting (e.g. two different issues got entangled) and you need to split them without losing in-flight uncommitted work."
---

## Problem

You committed a fix onto branch A, but it belongs on its own branch (A is a separate, already-scoped PR-in-waiting, or the commit is unrelated to A's purpose). The working tree also has an unrelated uncommitted change (e.g. a local-dev config tweak) that predates this commit and must not be lost — `git reset --hard` on A will discard it.

## Why the obvious approach is dangerous

`git reset --hard <old-tip>` resets both HEAD *and the working tree* to match that commit. Any uncommitted change to a tracked file is discarded irrecoverably. If you reset A back to its pre-fix tip while an unrelated uncommitted diff sits in the working tree, that diff is gone — not recoverable via reflog (reflog only covers commits, not uncommitted working-tree state).

## Safe sequence

```bash
# 0. Confirm what's uncommitted and what the fix commit's hash is.
git status --short
git log --oneline -3          # note <fix-commit> (current tip) and <old-tip> (before it)

# 1. Stash ONLY the unrelated uncommitted file(s) — not the fix, which is already committed.
git stash push -m "wip <file> (unrelated, pre-existing)" -- <unrelated-file>

# 2. Create the new branch AT THE CURRENT TIP, before touching branch A.
#    This is the step that matters: the fix commit is now reachable from two
#    branches, so it can never become orphaned/unreachable, regardless of what
#    you do to branch A next.
git branch <new-branch> <fix-commit>

# 3. Now it's safe to rewind branch A — reset --hard only destroys the working
#    tree, but the unrelated diff is already stashed (nothing to destroy) and
#    the fix commit is safely held by <new-branch>.
git reset --hard <old-tip>

# 4. Switch to the new branch to keep working on the fix there.
git checkout <new-branch>

# 5. Restore the unrelated uncommitted change.
git stash pop

# 6. Verify — don't assume, check:
git status --short                        # unrelated file shows modified again
git diff <unrelated-file>                 # byte-compare against what you expect
git log --oneline -2 <old-branch>         # confirms rewound to <old-tip>
git log --oneline -2 <new-branch>         # confirms fix commit is there, on top
git diff origin/main --stat <old-branch>  # confirms A's diff matches its pre-entanglement scope
```

## Why this order specifically

- **Branch-before-reset**: creating `<new-branch>` at the fix commit *before* running `reset --hard` on A means the commit is never in a state where only A points at it. If something interrupts the sequence between steps, the fix commit is still safe.
- **Stash-before-reset, not `--keep`**: `git reset --keep` also protects a clean working tree by refusing to run if reset would touch uncommitted local changes on a *conflicting* path — it's a reasonable alternative when the file differs between old and new tip, but stash is more explicit and lets you `pop` deliberately after switching branches, which is clearer when the reset and the branch switch are separated by other steps.
- **Verify by diffing content, not just checking git status**: confirms the stash/pop cycle didn't silently corrupt or partially apply the unrelated change, especially when a branch switch happens between push and pop.

## When NOT to use this

If either branch is already pushed/shared, this rewrites history other people may have based work on — use a revert or cherry-pick onto a new branch instead of rewinding the shared branch.
