---
name: tricipta-bubble-family-prs
description: "Create the cross-repo PR trio for a Tricipta bubble-map AI-insight family issue (#745/#746-style) across asset-web, plot-api, asset-api. Use when asked to \"create PR for branch/issue NNN\" involving feature/NNN-family-web/-plot/-api branches."
---

# Tricipta bubble-map AI-insight family PR trio

Use when creating PRs for a Tricipta bubble-map family branch set (`feature/NNN-<family>-web` + `-plot` + `-api`, e.g. #744 Location & Marker, #745 Injection Rate, #746 Cumulative). Recurring: one trio per issue, all repos under org `AFED-digital`.

## Discovery (parallel bash, per repo)

1. Branch set: `git -C ~/Sites/afed/<repo> branch -a | grep -iE '<NNN>'` across `tricipta-edafy-asset-web`, `tricipta-diagnostic-plot-api`, `tricipta-edafy-asset-api` (uiux-asset is NOT in the trio).
2. Existing PRs: `gh pr list --repo AFED-digital/<repo> --head feature/NNN-... --state all`.
3. Issue scope/title: `gh issue view NNN --repo AFED-digital/tricipta-edafy-asset-web`.
4. Stacking: `git -C <repo> log --oneline origin/main..origin/feature/NNN-...` — if a later branch contains the earlier family's commits as ancestors, the later PR body must claim the **delta** diffstat (`git diff --stat origin/feature/<prev>-... origin/feature/<next>-...`), NOT the main-scope one, or an advisory/`gh pr edit` fix is needed.

## Convention (matches merged #744 trio: PRs 797/19/149)

- Base: `main` (HEAD branch of every repo). Title identical across repos: `feat: AI insight for <Family> bubble map family #NNN`.
- Body template: `# AI Insight for Bubble Map <Family> Family — <Slice>` (Asset Web | Plot API | Asset API); `Closes #NNN (<slice> slice). Continues #683.`; Summary w/ diffstat + commit count; `## What changed` table; `## Commits` (short SHA + subject); `## Merge order` (only if stacked: merge earlier-family PRs first, GitHub dedupes ancestors); `## Testing`; `## Caveats` incl. companion branch names (`feature/NNN-<family>-web/-plot/-api`).
- Grounded file/commit claims from `git diff --stat --name-only origin/main...origin/<branch>` and commit subjects; never invent test counts.
- Watch for out-of-scope commits riding a branch (e.g. Ratio & Performance enablement tagged #746 with no own issue) — surface in Caveats, don't relabel.

## Verification before creating (no CI exists on feature branches)

Run targeted suites at each branch tip; checkouts are often dirty (`M` pyc/docker-compose) → use `git worktree add /tmp/wNNN-<repo> feature/NNN-...` for branches not checked out, remove with `--force` after (pytest cache).

- web: `npx vitest run test/pages/bubble-maps-dashboard.test.ts test/utils/bubbleMap*.test.ts` (branch tip ~86–153 passed)
- api: `go test ./internal/application/bubblemapinsightssvc/... ./internal/application/monthlyproductiondatasvc/...` (postgres package is integration; skip)
- plot-api: `venv/bin/python -m pytest tests/test_analysis_profiles.py tests/test_spatial_*.py -q` (venv at repo root; ~48–66 passed)

## Creation

`gh pr create --repo AFED-digital/<repo> --base main --head feature/NNN-... --title ... --body-file /tmp/pr-bodies/<slice>-NNN.md` — all 3 (or 6, stacked) in parallel bash calls. `gh pr edit <num>` re-pushes revised bodies.

Restore: leave each repo on its original checked-out branch; `git worktree prune` after removal.
