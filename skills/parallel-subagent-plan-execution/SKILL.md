---
name: parallel-subagent-plan-execution
description: "Execute a house-format implementation plan (docs/superpowers/plans/*.md) by fanning tasks to parallel subagents in one repo working tree: wave planning (≤2 concurrent), self-contained briefs with frozen contracts, path-limited commits, independent verification of each claimed commit, and mid-flight decision propagation to spec/plan/diagram"
---

# Executing a phased plan with parallel subagents (one repo, shared working tree)

## When
A house-format plan (`~/Sites/afed/tricipta/docs/superpowers/plans/*.md`) with file-disjoint tasks that can run in waves — especially Tricipta design-port plans (prototype → `tricipta-edafy-asset-web`) and multi-repo feature plans (asset-api A*, plot-api P*, asset-web W*).

## Wave planning
- Cap: **2 concurrent** subagents. Recompute waves from the plan's dependency graph; a task that imports a sibling's not-yet-created file waits for that sibling.
- **One implementer per task.** Never hand one agent a chain (A1→A4): it accumulates context and skips per-task review. If the user picked "subagent-driven, one task at a time", batching reverses their choice. Parallelize only across disjoint repos/files.
- Fill a freed slot immediately with the next runnable task in any repo whose prerequisites are committed (e.g. web utils that need only the frozen contract).
- Each dispatch is ONE `task` call with one `tasks[]` item per free slot; never serialize runnable work.
- **Final verification runs alone on a frozen HEAD.** Never run a code-changing agent (review fixes, late tasks) concurrently with the final verification/benchmark/smoke agent: its numbers get measured on a pre-fix tree and become stale PR evidence. Order: whole-branch review → fixes → final verification (suites + benchmark + live load test + smoke). If fixes land after verification, re-run all of it before quoting numbers.

## Per-task briefs (extract, don't paste)
- Build each brief file from the plan: plan header (Goal + Global Constraints + waves) + the repo Part's preamble (repo constraints, implementer notes) + that `### Task Xn:` section. Split on headings outside ``` fences. Write to `/tmp/sdd-<issue>/task-<ID>-brief.md`; also dump Global Constraints to `global-constraints.md` for reviewers.
- **Rebuild all briefs after every plan edit** — briefs are snapshots.
- Dispatch text: repo/branch/HEAD, brief path ("read first, it is your requirements"), report path, interfaces from earlier tasks the brief can't know, and any correction you made. Return only STATUS/SHA/test summary/concerns.

## Brief content checklist
1. Repo path, branch, HEAD SHA; plan file + task id; spec file.
2. Exact files to create/modify; committed building blocks with symbol names and interfaces.
3. Frozen contracts copied in (labels, route strings, export names, wire field names, units).
4. Source-of-truth `file:line`; never edit reference repos; never invent numbers.
5. Git rules: literal path-limited commit; forbidden verbs (add -A, stash, reset, checkout of others' files, clean, rebase, push, PR, amend of others' commits — and amend of own commit after the lead has verified its SHA).
6. Verify: exact test command, acceptance invariants, report format (SHA, fail-first evidence, deviations).
7. Do-not-touch list incl. pre-existing dirty files and the user's running containers (no `docker compose up/down` when containers have fixed names; use uniquely suffixed throwaway containers).

## Git hygiene (hard-won)
- Path-limited `git add <paths> && git commit -m "<msg>" -- <paths>`; a sibling's `git checkout <paths>` once silently reverted another agent's uncommitted edits.
- After each task: `git show --stat <sha>` and **re-run the task's tests yourself** — "tests pass" is a claim. Keep a ledger in `<repo>/.superpowers/sdd/progress.md` (git-ignored) with one line per reviewed task.
- Check `git reflog` before final report: agents sometimes amend after reporting a SHA (reported SHA then no longer on the branch).
- Unattributed working-tree edits: keep them, re-test, commit honestly; offer revert.

## Review loop
- After each implementer: `review-package BASE HEAD` (superpowers subagent-driven-development scripts), dispatch a `reviewer` agent with brief + report + diff + global constraints, naming concrete risks to check. Never pre-judge severity.
- Critical/Important → dispatch a small fix subagent (TDD, fail-first proof, new commit) → confirm → ledger. Minor → ledger for final triage.
- Reviewers that mutation-test or run the test against the parent commit catch non-discriminating tests; ask fixers to prove fail-on-parent via `git archive <parent> | tar -x -C /tmp/x` + `PYTHONPATH`/copy, then delete the temp dir.

## Plan corrections mid-flight (common — expect several per plan)
- Typical plan defects: stale expected test counts; build expectations that can't hold between interface and impl tasks; fixtures contradicting the plan's own implementation snippet or the spec; SQL that Postgres rejects; missing timeouts; **unit mismatches at mapper boundaries**; verify commands that break on deleted files (e.g. `gofmt -l $(git diff --name-only ...)` needs `--diff-filter=d`).
- When an executor or advisory proves the plan wrong: fix plan text (code AND expected values AND verify text) with a `Corrected <date>` note explaining why, so nobody "fixes" the code back; also fix every downstream task that recomputes the same quantity (grep the plan for the formula); commit plan; rebuild briefs.
- Before dispatching a mapper/ratio task, check units yourself at every boundary: endpoint wire units → consumer's unit map → display label. "Unit parity verified" in a drafted plan is a claim.
- Decision flips propagate to spec, plan steps + verify text, and the archify workflow JSON → validate (9/9) → deliver → visual-check.

## Staging / secrets
- Credential recipes must self-locate (grep candidate .env paths for the host) and STOP with BLOCKED if absent — never fall back to an empty local DB or drop the acceptance check. Enforce read-only server-side: `PGOPTIONS='-c default_transaction_read_only=on'` and assert `current_setting('transaction_read_only')='on'`. Never print the password; confirm the env file is git-ignored. Keep credentials out of PR bodies.

## Verification duty (every task)
- Re-run tests; confirm commit scope; spot-check acceptance strings. Final task = full suite + build + browser/live check + whole-branch review, on the frozen post-fix HEAD; no push/PR without explicit user instruction. When reporting evidence, state which commit each number was measured on.
