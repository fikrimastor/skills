---
name: parallel-agents-shared-worktree-git
description: "Use when dispatching 2+ parallel subagents that edit and commit in one shared git worktree (no per-agent isolation): the git rules to put in each brief, how to verify commit scope, and how to recover when a sibling's git cleanup wipes in-flight edits."
---

# Parallel subagents in one shared git worktree

## When
Two or more subagents implement slices of the same plan in the same working tree (the `task` tool has no worktree isolation). Uncommitted edits are shared mutable state; one agent's "cleanup" is another's data loss.

## Rules to put in EVERY agent brief
- Commit path-limited, one commit per task:
  `git add <your files> && git commit -m "<task message>" -- <your files>`
- NEVER run repo-wide git: `git checkout`, `git restore`, `git stash`, `git reset`, `git clean`, `git add -A/-u`. Do not "clean the tree" to get a green status — untracked/staged changes may be a sibling's work in progress.
- Stage only your own paths; never `git add -A`.
- If `git commit` reports an index lock (siblings commit concurrently), sleep 2 s and retry, up to 5×.
- Do not touch files outside your list, even to "fix" them — report instead.

## Verification you owe after each agent reports
1. `git show --stat --oneline <sha>` — the changed paths must equal exactly that task's file list.
2. `git status --short` — a clean tree (plus other agents' in-flight files) is expected; anything else needs an explanation.
3. Re-run the task's scoped tests yourself (`npx vitest run <file>`) — a subagent's "X passed" is a claim, not evidence.

## Symptom → cause → recovery
- **Symptom:** an agent reports its edits to tracked files vanished between edit and test, while its newly created (untracked) files survive.
- **Cause:** a sibling ran a repo-wide `git checkout/restore/stash/reset` — untracked files are unaffected, tracked edits are destroyed.
- **Recovery:** tell the owner to reapply (its own source of truth), then re-verify; never reconstruct the lost edits from memory, and never "fix" it with another checkout. Freeze the tree for the duration if two agents must touch the same file.

## Bonus
An unattributed uncommitted diff appearing mid-run (mtime between two agents' commits) is not necessarily garbage: read it, run the affected tests, and if it is sound and non-destructive to keep, commit it separately with an honest message (e.g. `refactor(...): … #NNN`) rather than reverting someone's work or leaving a dirty tree.
