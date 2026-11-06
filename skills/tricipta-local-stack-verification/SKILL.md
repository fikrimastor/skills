---
name: tricipta-local-stack-verification
description: "Verify a Tricipta change end-to-end against the local dev stack (web :3000, asset-api :8085, plot-api :8000, Keycloak :8180) — covers the api container rebuild, CORS allowlist, preview-mode plot-API localhost guard, build heap limit, auth-first 401 masking route changes, and how to reach the AI-insight panels in a headless browser."
---

# Verifying Tricipta changes against the local stack

Use when a change in `tricipta-edafy-asset-web` / `tricipta-edafy-asset-api` needs proof beyond unit tests: route changes, dashboard UI, insight panels.

## 1. Know what is actually running

```bash
for p in 3000 8085 8000 8180 6379; do printf "%s: " $p; (nc -z -G 1 127.0.0.1 $p && echo open) || echo closed; done
docker ps --format '{{.Names}} {{.Status}}' | head
docker inspect edafy-api --format '{{.Config.Image}} {{.Created}}'
```

- `edafy-api` (asset-api) and `plot-api-682` (plot API) run from **built images with no source mount**. A route or handler change is invisible until:
  ```bash
  cd ~/Sites/afed/tricipta-edafy-asset-api && docker compose build api && docker compose up -d api
  ```
  (the repo's `docker-compose.yml` may carry the user's own uncommitted env edits — never stage or revert it).
- Port 3000 binds IPv6 `[::1]` only, so a `nc 127.0.0.1` probe can report closed while the dev server is fine. Check `lsof -nP -iTCP:3000 -sTCP:LISTEN`.

## 2. Prove route existence — status codes lie

Unmatched `/api/v1/*` paths return **401**, not 404, because the JWT middleware runs before routing. `curl -o /dev/null -w '%{http_code}'` therefore cannot show that a route was deleted.

Reliable proofs, cheapest first:
1. `grep -c '"/new-path/' docs/swagger.json` plus absence of the old path (after `swag init -g cmd/server/main.go -o docs`, never with `--parseDependency`).
2. `internal/router/router.go` diff.
3. Authenticated live call (below).

## 3. Get a token without the browser

```bash
curl -s -X POST http://localhost:8085/api/v1/auth/login \
  -H 'Content-Type: application/json' \
  -d '{"username":"testuser","password":"testpass"}'
```

Backed by Keycloak Direct Access Grant. One or two attempts only: `LoginRateLimitMiddleware` is 0.167/s burst 5 and the account locks for 30 min after `AUTH_MAX_FAILED_ATTEMPTS=5` — a 429/403 is lockout, not a route bug.

## 4. Browser session

- CORS allowlist is `CORS_ORIGINS=http://localhost:3000,http://localhost:3001` (asset-api). Any other origin (e.g. a preview on :3002) silently fails credentialed calls and bounces to `/login`.
- Driving SSO directly at the API (`localhost:8085/api/v1/auth/login?redirect=…`) works but lands the tab on the **API origin**, and the refresh cookie it sets cannot be renewed by the app → repeated `/auth/refresh` 401 → redirect/reload loop. Prefer clicking `SIGN IN WITH SSO` **on the app origin** (login page carousel makes aria refs stale — click by text via `eval`, not by ref), or seed `localStorage['auth-token']` with a freshly minted bearer so `restoreSession()` skips refresh entirely.
- Headless tooling: `tab.run` + `page.*`, `tab.observe()`, `tab.id(ref)` and `tab.click()` frequently die with `Execution context was destroyed`. What works is one self-contained `tab.evaluate` string IIFE (promise-returning is awaited), e.g. click then `setTimeout(read, 1500)`. Keep each call under the 30 s browser-execution ceiling.
- `tab.screenshot()` times out (20 s+) on the map/chart dashboards (ArcGIS/three.js canvases) — verify by DOM geometry + text + network entries instead, or clip to the panel element once.

## 5. Preview (production build) caveats

```bash
cd ~/Sites/afed/tricipta-edafy-asset-web
NODE_OPTIONS=--max-old-space-size=8192 npm run build      # default heap OOMs
PORT=3001 NUXT_PUBLIC_API_BASE=http://localhost:8085/api \
  NUXT_DIAGNOSTIC_PLOT_API_BASE=http://127.0.0.1:8000 \
  NUXT_DIAGNOSTIC_PLOT_API_KEY=plot-twist-api-key-2026 \
  node .output/server/index.mjs
```

- Use :3001 (CORS-allowed, stop any dev server holding it first).
- `server/utils/diagnosticPlotApiClient.ts` **refuses** a localhost/127.0.0.1 plot-API base when `NODE_ENV=production`, so Diagnostic Plots capabilities 500 on a preview and its insight trigger stays inert — verify that dashboard on the dev server with `.env` loaded, or accept the test-covered gap.
- `bubbleMapInsightEnabled` is false when `APP_ENV=production`; a missing `AI Insight` button on a preview usually means that, not a broken trigger.

## 6. Dashboard interaction gotchas

- Bubble Maps trigger toggles unconditionally (`aiInsightOpen = !aiInsightOpen`); Diagnostic Plots `toggleInsightPanel()` no-ops while `insightSlot.status === 'inert'` (trigger styled `cursor-not-allowed`, title "select well to generate"), and it only leaves inert once well **and** capability **and** dates resolve — so check the trigger's own class/title before concluding wiring is broken.
- Chat tab in the AI Insight panel is front-end-only by design: assert `0` new network entries after clicking a chip; the reply lands after ~600 ms with a pulsing thinking row.
- Insight endpoints live at `POST /api/v1/insights/{module}` (see the #871 spec); the app reaches them through `authFetch` from the composables in `app/composables/use*Insight*.ts`.
