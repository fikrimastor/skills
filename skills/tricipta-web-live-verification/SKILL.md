---
name: tricipta-web-live-verification
description: "When verifying a Nuxt/Vue (asset-web) UI change live or via SSR — preview build, SSO, hidden-tab WebGL, DOM evidence."
---

# Live-verifying a tricipta-edafy-asset-web (Nuxt) UI change

Use when a UI change must be confirmed against the running app rather than unit tests: map/chart
rendering, layout, interactions, anything auth-gated or drawn on WebGL/ArcGIS. Includes the
fallbacks for when the dev server or the browser can't cooperate.

Repo: `~/Sites/afed/tricipta-edafy-asset-web` (Nuxt 4 / Vue 3 / TS).

## 0. Services — check before starting anything

- `lsof -nP -iTCP:3000 -sTCP:LISTEN` — a `nuxt dev` server is usually already running and binds IPv6
  `::1` only, so `nc -z 127.0.0.1 3000` **falsely reports down**. Use `curl http://localhost:3000`.
  Do not start a second web dev server on :3000.
- asset-api `:8085` and Keycloak `:8180` must be up — the login redirect needs both.
- plot-api `:8000`. Login `testuser` / `testpass` via the app's `SIGN IN WITH SSO` button (Keycloak
  round-trips silently if a session already exists). Sessions drop when a dev server restarts —
  re-login rather than debugging "the app is broken".

## 1. Prefer a built preview over the dev server when it thrashes

The user's :3000 dev server often sits in an auth-driven reload loop (`/auth/refresh` → 401 →
session-expired recovery → reload, every ~20-25 s), which destroys browser execution contexts
mid-interaction and makes local state (`aiInsightOpen`-style) unobservable; `docker logs edafy-api
--since 2m | grep auth/refresh` confirms it. HMR also leaves a stale graph after a rebase.

```bash
NODE_OPTIONS="--max-old-space-size=8192" npm run build      # default heap OOMs (ArcGIS/three.js/Chart.js)
NODE_ENV=development PORT=3001 \
  NUXT_PUBLIC_API_BASE=http://localhost:8085/api \
  NUXT_DIAGNOSTIC_PLOT_API_BASE=http://<LAN-IP>:8000 \
  NUXT_DIAGNOSTIC_PLOT_API_KEY=plot-twist-api-key-2026 \
  node .output/server/index.mjs
```
Expect `Client built` + `Nitro server built` and a `.output/` dir. Stop any process already holding
3001 first.

Hard constraints:
- **CORS**: asset-api allows `http://localhost:3000` and `:3001` only (`CORS_ORIGINS` in its
  `.env`) — serve the preview on **3001**, never 3002+ (login bounces).
- **Plot-API guard**: `server/utils/diagnosticPlotApiClient.ts` rejects a localhost plot base when
  `NODE_ENV=production` (the built server's default). Either point
  `NUXT_DIAGNOSTIC_PLOT_API_BASE` at the LAN IP (`ipconfig getifaddr en0`) or set
  `NODE_ENV=development` to defeat the guard. Without it the Diagnostic Plots page shows "Unable to
  load plot capabilities" and its insight trigger stays `inert` forever.
- `hub start` readiness probes 127.0.0.1 while Nuxt binds `[::1]`, so readiness times out even when
  the server is up — verify with `curl http://localhost:3001/`.

## 2. Authenticate: go straight to the API login

Clicking the page's "SIGN IN WITH SSO" does nothing under automation. Cookies are host-scoped, so a
session established on `localhost` works on any port there.

```js
await tab.goto("http://localhost:8085/api/v1/auth/login?redirect=" +
  encodeURIComponent("http://localhost:3000/common-platform/bubble-maps-dashboard"));
// Keycloak form: fill #username / #password (testuser / testpass) — set .value + dispatch 'input',
// or type; submit input#kc-login, button[type=submit]
// lands on :8085/... (API origin) — expected; now navigate to the app:
await tab.goto("http://localhost:3001/common-platform/<page>");
```
Success = the app path loads with a real user in the header ("Test User / Surveillance Supervisor").
If it bounces to `/login`, re-run the flow (the callback sets fresh `edafy_token`/`edafy_refresh`)
and check CORS. HttpOnly cookies mean `document.cookie` proves nothing.

## 3. Make the tab VISIBLE before judging any canvas (ArcGIS/WebGL)

Harness tabs open **hidden**; Chromium pauses `requestAnimationFrame`, so the ArcGIS/WebGL view never
renders: canvas stays at the default 300×150, zero `…/MapServer/tile/…` requests fire, screenshots
look blank or "flat colour".

```js
const client = await page.createCDPSession();
await client.send("Emulation.setFocusEmulationEnabled", { enabled: true });
await client.send("Page.setWebLifecycleState", { state: "active" });   // visibilityState -> "visible"
await page.reload({ waitUntil: "domcontentloaded" });
await client.send("Page.setWebLifecycleState", { state: "active" });   // re-apply after every reload
```
After activation the canvas resizes to the container and tiles load. If the harness **lacks**
`createCDPSession`/`bringToFront` (some shims do — check once, don't hunt), fall back to §6/§7.

Diagnostics that separate "my change is broken" from "the environment is broken":
- `.esri-view canvas` at 300×150 = the view never got a size (hidden tab / not yet rendered); a
  correctly sized canvas means the SDK is alive.
- Count requests on the **tile host** (`arcgisonline` / `url.includes("/tile/")`), not localhost.
  Metadata `…/MapServer?f=json` + `static.arcgis.com/attribution/…` loading while zero tiles fire =
  the render loop isn't running (§3), not a basemap misconfiguration.
- Identify failures by **response status**, not `requestfailed` (HTTP errors don't emit it): log
  `response.status() >= 400` with URLs. The recurring 400s here are Azure telemetry
  (`dc.services.visualstudio.com/v2/track`) — ignore.
- A tight offshore/over-water zoom legitimately reads as flat dark navy. Zoom out
  (`.esri-zoom .esri-widget--button` titled "Zoom out") before concluding tiles are missing —
  landmasses + place labels are the proof.

## 4. Evidence gathering under browser-tool limits

- **Only `tab.evaluate` with a self-contained async IIFE is reliable.** `tab.run` + `page.*`,
  `tab.observe`, `tab.id(ref)` and `tab.click` fail differently here (stale refs, CDP node-handle
  errors, `Execution context was destroyed`). Keep click → `await sleep(...)` → read in ONE call.
  Do not split click and read across cells.
- Browser code execution has a **hard ~30 s cap** regardless of the eval timeout — keep cells short,
  split a long sequence across cells, and pass `timeout: 0` to `eval` for long loops.
- **`tab.screenshot()` times out** (~20 s, canvases) and `page.screenshot({clip})` can exceed the
  cap. Don't gate on pixels; if you must, take a full-viewport shot and crop later.
- `innerText` omits clipped content — use `textContent`.
- **DOM evidence that proves things**: `getBoundingClientRect()` for dock geometry
  (`w=380 left=985` = docked right column), `document.querySelectorAll('button')` text for tab
  toggles, `aside.textContent` slices for section presence, `.bubble-playhead` `value`/`max`/`disabled`
  and its bar's `innerText` for date labels.
- **Computed style proves a utility resolved**: read `backdrop-filter`, `box-shadow`,
  `border-radius` — a class that doesn't exist in v4 silently draws nothing.
- **No-network proof**: diff `performance.getEntriesByType('resource').map(r => r.name)` before/after
  and report the **names** of new entries. A count is polluted by insight re-fetches or telemetry
  beacons.
- Inside `tab.run` the callback runs in the harness runtime: `document` is not defined — use
  `page.evaluate(() => ...)`.
- `read <png>?q=<question>` is how you "see" a screenshot; vision on small/ambiguous regions is
  unreliable — get numeric proof whenever a claim must be exact. Use vision on a cropped shot when
  judging a rendered map.

## 5. Interaction gotchas

- A fresh tab per verification run — old tabs accumulate HMR/reload state.
- HMR from concurrent file edits destroys the execution context and resets SPA state (capability back
  to default, slider 0). Finish all edits before interacting; confirm quiescence by sampling
  `performance.timeOrigin` (a change means the page reloaded).
- Vue `element.click()` via `tab.evaluate` does reach handlers on a hydrated page. The usual failure
  is clicking "the first button", which breaks once panel chrome (tab toggles) precedes content —
  select by visible text, and don't match a nav pill when you mean a card link of the same name.
- Dropdown = click trigger, then click the item (exclude the trigger's own `min-w-[260px]` class);
  slider = focus + keyboard (`End`/arrows) or the Play button. Assert DOM state after a settle wait.
- Aria refs go stale on animating pages (login carousel, maps); vue app buttons accept
  `element.click()`.

## 6. When the canvas won't paint: read the resolved prop instead

If the tab reports `document.visibilityState === "hidden"` and the shim has no focus/CDP API, do not
burn turns fighting it.

First rule out a real regression (cheap):
```bash
curl -s "http://localhost:3000/<route>" -o /tmp/ssr.html -w "http=%{http_code}\n"
grep -o "<expected text marker>\|Internal Server Error" /tmp/ssr.html | sort | uniq -c
```
plus a `tab.evaluate` presence check + `document.body.innerText.length`. If both are healthy, the
blank pixels are a capture artifact — say so and move on.

Entry animations also freeze at `opacity: 0` (a `slideUpFade`-style class), which blanks even DOM
content. Unfreeze (reveals DOM, not tiles):
```js
const s = document.createElement('style');
s.textContent = '*{animation:none!important;opacity:1!important}';
document.head.appendChild(s);
```

Then read the value off the mounted Vue instance — proves route→computed→prop wiring with no paint:
```js
const el = document.querySelector('.bubble-geo-map');
let n = el.__vueParentComponent, d = 0, out = null;
while (n && d < 8) { const p = n.props || {}; if ('basemap' in p) out = p.basemap; n = n.parent; d++; }
JSON.stringify({ search: location.search, basemap: out });
```
Works in the Vue dev build. Run it across input cases by navigating with different query strings.
Harness shim availability observed: `tab.evaluate`/`tab.run`, `tab.screenshot`, `tab.goto`,
`tab.ariaSnapshot`, `browser.open/close` exist; `page.bringToFront`, `page.createCDPSession`,
`page.addInitScript`/`evaluateOnNewDocument`, `page.setRequestInterception` may not.

## 7. When the browser is unavailable entirely: verify compiled SSR HTML

Generic Nuxt recipe — works for any repo, not just Tricipta.

1. Grep the route's middleware (`app/middleware/*.global.js` or per-page
   `definePageMeta({ middleware })`) for an early return guarding on server context:
   ```js
   export default defineNuxtRouteMiddleware((to) => {
     if (process.server) return; // <-- the tell: auth is client-only
     const auth = useAuthStore();
     if (!auth.isAuthenticated) return navigateTo('/login');
   });
   ```
   If present, SSR renders the full page (empty/default auth state) even though the client would
   redirect after hydration — fetch the real route directly, no credentials or mocks needed.
2. `read <url>:raw` (literal HTML, not reader-mode markdown) against the real route. That is the
   compiled DOM: real class strings, real computed inline `style="…"`, real `v-if`/`v-else`
   placeholder comments (`<!---->`) showing which branch rendered.
3. Grep the output for the specific thing you changed — a new CSS property, a prop's effect, a
   conditional's placeholder — with exact computed numbers (e.g. `aspect-ratio:700 / 560`).
4. If the reader wraps an embedded `<svg>` as a base64 data-URI `<img>`, decode it
   (`Buffer.from(b64,'base64').toString('utf8')`) and check the geometry by hand against the
   expected formulas — that catches silent logic errors "it rendered" misses.
5. Auth enforced server-side too, or you need controlled data → create a temporary page outside
   gated prefixes (`app/pages/dev-verify-<topic>.vue`) importing the real component with
   realistic/edge-case props (the exact shape that triggers the bug), fetch it the same way, then
   delete the harness file as cleanup (scaffolding, not a test).
6. Don't trust delegated self-reports: re-read the files and re-run the fetch yourself.

Limits: proves compilation + structural/computed-CSS correctness, not pixels, scrolling,
interaction, or cross-browser rendering. Say so, and ask the user to spot-check aesthetics.

## 8. Typecheck and CI truth (avoid a false green)

```bash
# FALSE GREEN — always 0 errors. Do NOT use, do NOT report its output.
npx vue-tsc --noEmit -p tsconfig.json

# TRUTH — follows project references.
npx vue-tsc --build --force
```
The root `tsconfig.json` is Nuxt's references pattern (`{"files": [], "references": [...]}`);
`-p` on it resolves a project with zero files, so it trivially passes. The repo-wide baseline is
large (253 errors as of 2026-09-15) — never claim a total, claim **your delta**:
```bash
git stash push -- <the files you touched>
npx vue-tsc --build --force 2>&1 | grep -cE '^app/pages/your-file\.vue'
git stash pop        # back up to /tmp first and confirm with `diff -q` — a pop is not self-proving
```
Compare by **count and error code**, not line number (your edits shift lines).

`.github/workflows/ci.yml` gates on `npm run test` (vitest) and `npm run build` only. `npm run lint`
is `continue-on-error: true` **and** broken (ESLint 9 needs `eslint.config.js`; the repo has the
legacy `.eslintrc` that contains `export default`), and `nuxt build` does not fail on SFC type
errors. There is no `typecheck` step. **Green CI proves nothing about types or lint here.**

Suite gate: `npx vitest run` in the repo root (53 files / 650 tests as of 2026-09; 63/696 as of
#808 — expect drift, quote what you actually saw).

## 9. Known data preconditions

- **Bubble Maps**: `All` / `Last N Years` presets both generate insight; the panel needs a resolved
  field (check `— field`; "No strings found for the current scope" means no scope yet).
- **Diagnostic Plots**: the insight trigger is a no-op while the slot is `inert`
  (`diagnostic-plots-dashboard.vue` `toggleInsightPanel`); the slot only leaves `inert` once a well
  **and** capability resolve. The default date window frequently yields
  `Missing inputs: minimum_points` — click the **All** range preset to get a ready render.
- Bubble Map insight ready takes ~4-5 s (LLM); the backend surfaces 502 with
  `status must be one of "snapshot", "watch", "limited_data"` when the model emits confidence
  vocabulary in the status field — a retry usually renders.
- Insight panels: assert `aside` width/left (docked column ≈380 px) and the header text, then check
  `docker logs edafy-api --since 5m | grep insight` for `/api/v1/insight/<module>`.
- Probe real data routes with `curl -s -o /dev/null -w '%{http_code}'` only for existence —
  unmatched `/api/v1/*` returns 401, not 404.
- DDM sensitivity/forecast data exists locally only for field **Bokor**.

## 10. Bubble maps: what to assert

- Plotted bubble identity = `formatLocationWellLabel(...)` (well + `-LS`/`-SS`/`-TS`); nav hierarchy
  uses `well.code` bare (`ANSI-A012`). Match with
  `label === code || label.startsWith(code + "-")`, case-insensitive — `matchesWellSelection` in
  `app/utils/bubbleMapDashboardData.ts`.
- Design is the source of truth for map card chrome: header (title + counts), legend overlay card
  (`top-3 right-3`), bottom playback overlay bar (skip-back / play / slider with `MMM yyyy` labels /
  1x-2x), loading overlay, `rounded-lg` map wrapper. The uiux app renders it with Leaflet; the web
  keeps ArcGIS — match tiles (`World_Imagery` base + `World_Boundaries_and_Places` reference), not
  the SDK.
- Real user flow worth exercising: play/pause + skip-back move `.bubble-playhead`'s `value`;
  selecting a row/well adds a halo + short label on the map.
- `eslint` cannot run in this repo (`.eslintrc.json` contains `export default`; no flat config).
  Verify with `npx vitest run` + the dev server compiling the page + a browser pass with no console
  errors.

## 11. Reporting

State exactly what was observed — geometry numbers, text slices, request URLs from `performance` +
API container logs, `exit=` codes, test totals — and mark anything covered only by unit tests as
such. Use an explicit **"could not verify"** list (e.g. no 1440 screenshot possible in a 743 px pane,
no painted frame captured). Never claim a pixel or visual check you didn't get.
