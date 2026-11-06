---
name: split-commit-onto-new-branch
description: "Safely move a commit that landed on the wrong git branch onto its own new branch, preserving any unrelated pre-existing uncommitted changes in the working tree. Use when a fix for a different issue got committed onto a branch carrying other unpushed work, or any time a commit needs isolating via reset --hard on a branch that also has uncommitted edits worth keeping."
---

## When to use

A commit landed on branch `B` (tip `C2`) that should really be its own branch — e.g. you were working on issue A's branch and, mid-session, also fixed unrelated issue C, committing it onto the same branch. `B` also has pre-existing, unrelated **uncommitted** changes in the working tree (e.g. someone's in-flight config tweak) that must not be lost.

Only safe when `B` is not pushed anywhere (or you're prepared to force-push after confirming nothing depends on its current tip) — this procedure rewrites local history with `reset --hard`.

## Procedure

Given: branch `B` currently at tip `C2` (the commit to move off), with `C1` as `B`'s prior tip (the state to restore `B` to).

```bash
# 1. Stash the unrelated uncommitted diff FIRST.
# `git reset --hard` silently discards uncommitted changes to any tracked
# file whose committed content differs between C1 and C2 — do this
# unconditionally, don't try to reason about whether the file happens to
# be identical in both commits.
git stash push -m "wip <file> (unrelated, pre-existing)" -- <file>

# 2. Create the new branch at the CURRENT tip (C2) BEFORE touching B.
# Order matters: if you reset B first, C2 becomes reachable only via
# reflog instead of a normal ref, and you'd be recovering it under time
# pressure instead of just branching from HEAD.
git branch <new-branch> C2

# 3. Move B back to its pre-C2 tip. Safe now — <new-branch> already
# preserves C2 independently of what happens to B next.
git reset --hard C1

# 4. Switch to the new branch to continue work there.
git checkout <new-branch>

# 5. Restore the unrelated diff.
git stash pop
```

## Verify — don't just trust the sequence, check the result

- `git diff` on the restored file must show exactly the original unrelated
  diff, nothing more, nothing less (byte-identical round trip through
  stash/pop across a branch switch).
- `git log --oneline -3 <old-branch>` — tip is `C1`; the commit you moved
  off is gone from this branch's history.
- `git log --oneline -3 <new-branch>` — tip is `C2`, sitting on top of `C1`.
- `git diff origin/main --stat <old-branch>` — matches whatever file list
  was previously documented/reviewed for that branch, proving nothing
  extra bled in during the surgery.

## Why this exact ordering

- **Stash before reset**: the failure mode being avoided is silent, total
  loss of uncommitted work with no confirmation prompt.
- **Branch before reset**: costs nothing, and removes any window where the
  commit being preserved exists only in the reflog rather than a real ref.
