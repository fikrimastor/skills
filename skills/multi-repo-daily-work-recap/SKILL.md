---
name: multi-repo-daily-work-recap
description: "Reconstruct what work happened in a time window (\"what did I do today/this week?\") across a multi-repo workspace where in-flight work is uncommitted on same-day branches — git reflog + file mtimes + memory search, with the branch-tip-date and memory-date-filter traps."
---

# Multi-repo daily work recap

Use when asked "what task did I do today / this week / lately?" in a workspace whose product code lives in **sibling git repos** (e.g. `/Users/fikrimastor/Sites/afed/` holds `tricipta` + 5 product repos). The honest answer is a time-ordered reconstruction, not a commit list.

## Why `git log` alone fails here

Work in progress is typically **uncommitted** on a branch cut the same day from `origin/main`:

- `git log origin/main..HEAD` → empty.
- The branch tip's commit date is the *merged upstream* commit, not when the branch was created. Reading it as "work started" is wrong (observed: tip said 14:23, `reflog` said 16:59–17:01).
- Branches get **renamed** mid-task (`feature/871-ui-update-plot` → `feature/748-tuning-plot`), so the name is not a stable identifier either.

## Procedure

Run from the workspace root.

1. **Enumerate repos** — workspace root plus every entry in `<workspace-roots>`. Each needs `git -C <repo>`; the root is often *also* a repo (the docs/vault home).

2. **Branch + reflog (the real signal):**
   ```bash
   for r in <repos>; do
     b=$(git -C $r branch --show-current)
     git -C $r reflog show "$b" --date=format:'%m-%d %H:%M' | tail -3
   done
   ```
   `Branch: Created from origin/main` / `renamed refs/...` lines carry the true timestamp.

3. **Dirty trees = the actual in-flight work:**
   ```bash
   for r in <repos>; do echo "=== $r ==="; git -C $r diff --stat; git -C $r status --short | head -30; done
   ```
   The diffstat is the deliverable list. Note untracked strays deliberately (plans, ADRs, `__pycache__` that got tracked).

4. **File mtimes = the edit timeline** (catches work never staged):
   ```bash
   find <root> -path ./node_modules -prune -o -newermt "<YYYY-MM-DD> 00:00" -type f -print \
     | grep -v -e '/\.git/' -e 'graphify-out/cache' -e '\.obsidian'
   ```
   `stat -f '%Sm %N' -t '%m-%d %H:%M' <file>` sorts a directory into a timeline — best used on a generated corpus (`docs/architecture/`) to see *when* the generator last ran.

5. **Everything actually merged today** (per repo, not just current branch):
   ```bash
   git -C $r log --all --since="<date> 00:00" --pretty='%ad %h %d %s' --date=format:'%m-%d %H:%M'
   ```

6. **Memory** — `recall` / `reflect` with a *topical* query (issue numbers, feature names, "extractors", "repowiki"). This surfaces what the user said and what was concluded, which git cannot.

7. **Artifacts of self-improvement** — `find ~/.omp/agent/managed-skills -maxdepth 2 -name SKILL.md -newermt "<date>"` reveals skills minted that day, i.e. what problems were hit and codified.

8. **Report the loose ends explicitly**: in-flight branches with nothing committed, failing gates (`python3 scripts/repowiki.py check`), untracked files. Do not report a gate that exits non-zero as a pass.

## Traps

- **Memory date-filter is a dead end.** `mcp__claude_mem_mcp_search_search` with only `dateStart`/`dateEnd` returns `No results found matching "undefined"`; a `query` is mandatory and a generic one ("today") matches nothing. Use `recall`/`reflect` instead.
- **`find -newermt` needs the workspace-local path.** Sibling repos are NOT children of the workspace root — `git -C tricipta-edafy-asset-web` from inside `tricipta/` fails with `cannot change to ...`; use absolute paths or loop from the parent.
- **`git -C .` on the root** picks up vault/graph churn (`.obsidian`, `graphify-out/cache`) — filter it or the timeline is noise.
- **Branch tip date ≠ branch creation** — see above; always `reflog`.

## Output shape

Group by *thread* (issue/feature), not by repo. Each thread: branch per repo + diffstat + what it changes; then a separate "shipped/merged today" list, then "loose ends". A mermaid graph helps when the day has two independent threads.
