---
name: pr-stacked-branch-disambiguation
description: "Disambiguate a terse 'create a PR for repo for current branch to branch' request before running gh pr create, when it's unclear whether branch is the head (PR to main) or the base (stacked PR from the currently checked-out branch). Use whenever asked to create a PR and the target branch isn't explicitly labeled head/base, especially in multi-repo or stacked-branch workflows."
---

## Problem

Requests like "create a PR for web for current branch to fix/908-auth-bypass" are structurally ambiguous:

- **Reading A:** head=`fix/908-auth-bypass`, base=`main` — treat the named branch as the thing to PR.
- **Reading B:** head=*currently checked-out branch*, base=`fix/908-auth-bypass` — a stacked PR onto a not-yet-merged branch.

Guessing wrong wastes a cycle: Reading A either hard-fails (`gh pr create` errors if a PR already exists for that head) or silently does nothing useful if it's a no-op re-creation.

## Procedure

1. **Check for an existing PR on the named branch** before creating anything:
   ```
   gh pr list --head <branch-name> --state all --json number,title,state,url
   ```
   If an OPEN PR already exists with that head → Reading A is almost certainly wrong (redundant/duplicate). Treat this as a strong signal to check Reading B.

2. **Check ancestry** between the currently checked-out branch and the named branch:
   ```
   git merge-base --is-ancestor <branch-name> <current-branch> && echo YES || echo NO
   git merge-base <current-branch> <branch-name>
   git log <branch-name>..<current-branch> --oneline   # commits current has beyond it
   git log <current-branch>..<branch-name> --oneline   # commits current is missing (should be empty)
   ```
   If `is-ancestor` is YES and the merge-base equals `<branch-name>`'s tip SHA (current branch fully contains it, missing-commits list is empty), the current branch is stacked directly on top — this confirms **Reading B**: head=current-branch, base=`<branch-name>`.

3. **Do this for every repo in the request independently** — each repo's current branch may or may not be stacked; don't assume the same reading applies uniformly just because the phrasing pattern repeats across repos.

4. **Before creating the PR**, check whether the current (head) branch is already pushed:
   ```
   git rev-parse origin/<current-branch>
   ```
   If it errors ("unknown revision"), push first: `git push -u origin <current-branch>`.

5. **Create the stacked PR** explicitly, never relying on gh's default base:
   ```
   gh pr create --base <branch-name> --head <current-branch> --title "..." --body-file <file>
   ```
   Write the PR body to a temp file when it contains multi-paragraph markdown, to avoid shell-escaping corruption.

6. **In the PR body**, state the merge order explicitly (base branch's own PR must land — or at least be reviewable — before this one), and link the base branch's PR number if it has one open.

7. **Verify** after creation: `gh pr view <number> --json baseRefName,headRefName` — confirm the base/head landed as intended, not gh's default-branch fallback.

## Why this matters

An advisory in one session caught a misread of exactly this phrasing after PRs had already been (correctly, separately) opened for the named branches to `main`. The user's actual intent was a second, stacked layer of PRs on top of those — a common pattern when iterating on a feature that depends on an in-flight fix branch. Silently defaulting to "named branch → main" without checking existing-PR state and ancestry would have either failed outright or missed the real ask entirely.
