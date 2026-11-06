---
name: tricipta-ci
description: "When user says /tricipta-ci, or before opening/pushing a Tricipta PR: run each touched repo's ci.yml checks locally."
---

# tricipta-ci

Runs the non-deploy steps of each touched Tricipta repo's `.github/workflows/ci.yml` locally via one script.

## Run
```bash
cd /Users/fikrimastor/Sites/afed/tricipta
./scripts/tricipta-ci.sh --list            # plan only: detected repos + steps, writes nothing
./scripts/tricipta-ci.sh                   # auto-detect touched repos (dirty, or commits not on origin/main)
./scripts/tricipta-ci.sh tricipta-edafy-asset-api tricipta-edafy-asset-web-1355   # explicit repos/worktrees
./scripts/tricipta-ci.sh --docker ...      # also run docker build steps
```
Run it as a background bash job (`async`): the web build alone is ~70s, the full vitest suite ~40s.

## Read results
- Per step: `PASS|FAIL|WARN|SKIP <repo> <step> (Ns)`. Logs: `/tmp/tricipta-ci/<repo>/<step>.log`.
- `WARN` = ci.yml `continue-on-error` step failed; does not fail the run (mirrors GitHub).
- `SKIP` = docker without `--docker`, needs secrets/services, or pinned tool missing (prints install hint).
- Exit 1 if any FAIL, 2 for unknown repo.

## Known pre-existing failures (as of 2026-11-05) — do not attribute to the branch
1. asset-api `test-coverage`: untracked `.env` sets `DB_SSL_MODE=require`, and `config.Load()` reads `.env` over TestMain's value → testcontainer TLS refusal. Confirm with `DB_SSL_MODE=disable go test ./internal/router/`.
2. asset-web `test`: `ConnectivityPage.integration.test.ts` times out (5s) under full-suite load; passes alone.
3. chat-bot: 3 failures in `tests/unit/interfaces/` (`/auth/realm` 404, concurrency cap 429).
4. uiux lint: ~483 eslint problems; WARN only.
Before blaming a branch for a FAIL, compare against `origin/main` (rerun the step on a clean main worktree) — see `skill://typecheck-regression-attribution`.

## Report
One table: repo × step × verdict, then each FAIL with cause (new vs pre-existing) and the log path.
