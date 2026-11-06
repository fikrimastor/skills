---
name: tricipta-companion-pr-trio-with-visual
description: "Create the Tricipta #NNN companion PR trio (asset-web + asset-api + plot-api) with an embedded UI screenshot, avoiding the false-green traps: the APFS-worktree .env clone that fakes testcontainer failures, public-gist screenshot leakage on private repos, and reporting a whole suite as green when only some packages ran."
---

# Tricipta companion PR trio with an embedded visual

Use when asked to "create the PR for #NNN" for a Tricipta cross-repo feature and the web PR must carry a screenshot.

## 1. Order of operations

1. Web PR first (it is the one with the visual), then the two companions, then cross-link all three bodies (edit the first PR to name the sibling PR numbers).
2. Verify each branch before pushing: `git status --short --branch`, `git ls-remote --heads origin <branch>`, `gh pr list --head <branch> --state all`.
3. All three base `main` and use `Refs AFED-digital/tricipta-edafy-asset-web#NNN` — never `Closes` in a repo that does not own the issue number.
4. Push with `git push -u origin <branch>`. Pre-existing dirty/untracked files in the checkout are not pushed; never stage them.
5. PR titles follow repo convention: web `feat: <thing> #NNN`, api `feat(<area>): <thing> #NNN`, plot-api `feat(analysis): <thing> #NNN`.

## 2. The APFS-worktree `.env` trap (costs a false regression hunt)

`git worktree add --detach /tmp/<repo>-<issue>-pr HEAD` on macOS APFS uses `clonefile`, which copies the **whole working directory — including gitignored files such as the repo-root `.env`**. That `.env` in asset-api points at the staging database and sets `SKIP_AUTO_MIGRATE=true` and `DB_SSL_MODE=require`, so `internal/database` and `internal/router` fail inside the worktree with:

- `relation "tenants" does not exist (SQLSTATE 42P01)` (auto-migrate skipped)
- `tls error: server refused TLS connection` (sslmode=require against a plain `postgres:16-alpine` testcontainer)

Diagnose by proving it is environmental, not a branch regression:

```bash
cd /tmp/<repo>-<issue>-pr && mv .env .env.bak && go test ./internal/database ./internal/router -count=1; mv .env.bak .env
```

Both pass without `.env`. Also create the baseline worktree from `origin/main` and run the same packages — if the branch and main behave identically once `.env` is out of the way, it is not a regression. Report it as an environment artifact in the PR body, never as a green suite.

Running the full `go test ./internal/...` in a worktree **without** removing `.env` produces exactly those two failures and nothing else; do not let the PR body imply the whole suite passed.

## 3. Embedding a screenshot in a private-repo PR

- Commit the PNG into the branch (e.g. `.github/assets/<issue>-<slug>.png`) and reference the **same-origin** form:
  `https://github.com/<owner>/<repo>/blob/<sha>/<path>?raw=true` — renders for signed-in org members with the viewer's cookies.
- **Never** use a public gist. `gh gist create` rejects binaries anyway; the git-push workaround publishes brand-owned production screenshots (well names, rates) outside the private repo boundary.
- **Never** use `raw.githubusercontent.com` in the markdown: camo fetches server-side without a viewer cookie, so it 404s for private repos.
- Verify what is actually provable: `gh api repos/<owner>/<repo>/contents/<path>?ref=<sha> --jq '{name,size,sha}'` and a token-authenticated `curl` of the raw URL whose sha256 matches the source PNG.
- `img.naturalWidth > 0` is **not** assertable from the harness when the managed browser has no GitHub session (a private PR loads `Page not found`). State that gap explicitly instead of claiming the embed renders.

## 4. Honest verification gates

| Repo | Gates |
|---|---|
| asset-web | `npx vitest run`; `NODE_OPTIONS=--max-old-space-size=8192 npm run build`; `gh pr checks` |
| asset-api | `go build ./...`; `go test ./internal/... -count=1` (mind §2); swagger `platform_code` assertion on `docs/swagger.json` |
| plot-api | `PYTHONDONTWRITEBYTECODE=1 ./venv/bin/python -m pytest -q`; `compileall`; import smoke; `! grep -rn 'import scipy' plotting_core`; allow-list greps |

`npm run build` in asset-web OOMs without `NODE_OPTIONS=--max-old-space-size=8192` — always pass it. `npm run lint` exits 2 on ESLint 9 with no `eslint.config.*`; CI marks lint non-blocking, so report it as a pre-existing config gap.

## 5. Do not silently skip the staging smoke

The api container points `DB_HOST` at the staging database and `docker-compose.yml` boots `AutoMigrate`, so restarting it **writes to staging**. If the runbook's final task (e.g. A10) is a staging smoke, either get an explicit decision or report it as not run, and substitute the strongest non-writing evidence available — e.g. `docker inspect <container> --format '{{.Created}}'` compared with the branch head commit time (image newer than the commit ⇒ the running container contains the branch), plus `docker logs <container> --since 3h | grep -c <new_param>` to prove the live endpoint served the new contract.

## 6. Cleanup

`git worktree remove /tmp/<repo>-<issue>-pr --force && git worktree prune` in each repo once verification is done.
