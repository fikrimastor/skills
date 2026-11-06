---
name: parallel-subagents-shared-git-tree
description: "Fan out multiple subagents into ONE git working tree safely: encode path-limited commit rules in the brief, forbid repo-wide git cleanup, verify each commit's scope, and preserve unattributed floating diffs. Use when dispatching 2+ parallel executors that each write and commit in the same checkout."
---

# Parallel subagents in one git working tree

## When
Dispatching 2+ executors (waves) that each create/modify files **and commit** inside the same checkout — e.g. a phased implementation plan executed wave-by-wave with a concurrency cap.

## The failure this prevents
A sibling agent running any repo-wide git cleanup reverted another agent's *uncommitted* edits to **tracked** files (untracked new files survived). Symptom reported: "my edits were overwritten between edit and test". One `git checkout <files>` destroyed a peer's work irreversibly.

## Brief template (put in EVERY parallel task)
- "Stage only YOUR files and commit **path-limited**: `git add <paths> && git commit -m \"<exact message>\" -- <paths>`."
- "NEVER run `git checkout`, `git restore`, `git stash`, `git reset`, `git clean`, or `git add -A/-u`. To read state use `git status --short` only."
- "On an index lock (shared tree), sleep 2 s and retry (≤5×)."
- List explicit paths: created files, modified files, test files. Nothing else may be touched.
- State that siblings are editing other files concurrently — name their files so neither touches the other's.

## Dispatcher checklist (verify, don't trust)
1. After each agent settles: `git log --oneline -4` + `git show --stat --oneline <sha>` → commit contains exactly that task's files.
2. `git status --short` → no half-staged leftovers; identify any floating diff.
3. Re-run the task's test command yourself; a subagent's "tests passed" is a claim, not evidence.
4. On a conflict/lost-edit report: relay the path-limited rule to BOTH agents via a direct message, then have the victim reapply — never let the dispatcher "fix" it with checkout/restore.

## Floating diffs (unattributed working-tree change)
Never revert (`git checkout/restore`) — it may be the user's in-flight edit. Instead: read the diff, verify it compiles/tests, then either commit it as its own small commit with an honest message (`refactor(...): … #<issue>`) or leave it and flag it to the user. Determine provenance before blaming the agent you just dispatched: compare file mtime against each agent's dispatch/commit timestamps.

## Wave mechanics
- Concurrency cap (e.g. 2) counts *running* agents: dispatch the next slice when one settles, don't batch over the cap.
- Waves must be disjoint by file; dependencies (shared data module, primitives) land in an earlier wave and are frozen before dependents start.
- Each wave: dispatch → verify (tests + commit scope) → next wave. Keep the plan's task → commit-message mapping intact so history stays one-commit-per-task.
