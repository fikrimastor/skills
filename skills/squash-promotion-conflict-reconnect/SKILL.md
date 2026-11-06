---
name: squash-promotion-conflict-reconnect
description: "Diagnose and fix conflicts on a long-lived branch promotion PR (main→staging/uat/prod) caused by an earlier squash-merged promotion. Use when a main→staging PR suddenly conflicts (often add/add) even though staging only ever receives main's content."
---

# Squash-promotion conflict: diagnose and reconnect

## Symptom
A main→staging PR conflicts, often with **add/add** conflicts on files created recently. Nobody commits directly to staging.

## Diagnose (read-only)
```bash
git fetch origin
git log --oneline origin/main..origin/staging          # promotion commits on staging
git log -1 --format='%h parents=%p %s' origin/staging  # 1 parent + 'Main >>> Staging (#N)' => squash
git log -1 --format='%h %ad %s' $(git merge-base origin/main origin/staging)  # stale base = smoking gun
git merge-tree --write-tree --name-only origin/staging origin/main  # conflicting files, no checkout touched
```
Root cause: the promotion PR was squash-merged. Staging got main's content but not its ancestry, so git still uses the old merge-base. Every file added on main after that base shows up as add/add.

## Prove staging has nothing unique (required before `-s ours`)
```bash
T=$(git rev-parse origin/staging^{tree})
for c in $(git rev-list --since=<squash-date -3d> --until=<squash-date +1d> origin/main); do
  [ "$(git rev-parse $c^{tree})" = "$T" ] && git log -1 --format='MATCH %h %s' $c; done
```
- MATCH found: it is safe to resolve every conflict to main's version.
- No match: staging has hotfixes of its own. Do **not** use `-s ours`. Do a real merge and review the staging-only diff.

## Fix (no force-push), done in a temp worktree so the user's checkout is untouched
```bash
W=$(mktemp -d)/wt && git worktree add -b chore/sync-main-to-staging "$W" origin/main && cd "$W"
git merge -s ours origin/staging -m "chore: reconnect staging history after squash promotion (#N)"
[ "$(git rev-parse HEAD^{tree})" = "$(git rev-parse origin/main^{tree})" ] && echo tree==main
git merge-tree --write-tree --name-only origin/staging HEAD   # expect no conflicts
git push -u origin chore/sync-main-to-staging
gh pr create --base staging --head chore/sync-main-to-staging --title ... --body '...merge with Create a merge commit, NOT squash'
git worktree remove "$W"
```
Verify with `gh pr view N --json mergeable,changedFiles`. Expect `MERGEABLE`, and `changedFiles` should equal `git diff --stat origin/staging origin/main`. `UNSTABLE` only means CI is pending.

Alternative: `git push --force-with-lease origin origin/main:staging`. It is also lossless when the trees match, but it rewrites a shared branch and needs a protection bypass.

## Prevent
- Merge the reconnect PR with **a merge commit**. Squashing it recreates the trap.
- Add a ruleset on promotion branches that requires merge commits. Feature→main PRs can stay squash.
- Don't hand-resolve the conflicts in the GitHub UI and squash again.
