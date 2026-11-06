---
name: nuxt-ssr-cost-and-server-gate-measurement
description: Measure what enabling SSR actually costs in a Nuxt app (SSR-only regressions vs pre-existing defects) and prove a Nitro server middleware gate fires on ssr:false routes — includes the console-capture and routeRules-flip traps that produce false counts.
---

# Deciding whether to turn SSR on (and whether a server gate needs it)

Two separable questions, each with a cheap decisive experiment. Do not answer either from static reading.

## Question 1: does a Nitro server-side gate fire on an `ssr:false` route?

If yes, you never need to enable SSR just to gate routes server-side. Test with a throwaway middleware (delete it immediately):

```ts
// server/middleware/zz-probe-gate.ts
import { defineEventHandler, getRequestURL, sendRedirect, setResponseHeader } from 'h3'
export default defineEventHandler((event) => {
  setResponseHeader(event, 'x-zz-probe', 'fired')
  const url = getRequestURL(event)
  if (url.pathname === '/your/protected/path' && !url.searchParams.has('zzpass')) {
    return sendRedirect(event, '/zz-probe-redirect', 302)
  }
})
```

```bash
curl -s -D- -o /dev/null http://localhost:3000/your/protected/path | grep -iE '^HTTP/|^location:|^x-zz-probe:'
curl -s -D- -o /dev/null http://localhost:3000/login            | grep -iE '^HTTP/|^x-zz-probe:'   # ssr:true control
```

A 302 on the `ssr:false` route proves the gate works without touching `routeRules`. **Verified in Nuxt 4.4 (tricipta-edafy-asset-web, 2026-09-21).**

**Scoping is mandatory.** Nitro middleware runs on *every* request, not just navigations. A blanket default-deny 302s the JS bundle and the app never boots, and 302s API proxies too. Guard with: skip `/_nuxt`, `/_payload`, `/api`, `/__nuxt_error`, extension-bearing paths (`.js`, `.css`, `.svg`…), and require `sec-fetch-dest === 'document'` (treat missing as document).

## Question 2: what does enabling SSR actually cost?

### Step 0 — back up before flipping

`cp nuxt.config.ts /tmp/cfg.baseline` **in a call that does not also edit the file**, and record `sed -n '<N>p'` of the routeRules line. A backup taken in the same command that flips the switch may already hold the flipped content.

### Step 1 — status sweep (necessary, NOT sufficient)

Flip `'/**'` to `{ ssr: true }`, wait ~30s for the dev-server restart, enumerate routes from `app/pages/**/*.vue` (`index.vue` → its directory path), substitute dynamic segments with literals, then curl each. Report non-200s and explain each redirect stub.

**Green status codes hide broken-but-200 pages.** A route whose SSR HTML contains `[Vue warn]` or missing render functions still returns 200. Never conclude "SSR is cheap" from a status sweep.

### Step 2 — classified browser sweep (the real signal)

Drive a real browser and capture `Runtime.consoleAPICalled`. Then classify each message by **root cause**, dedupe by normalized text, and only count what the ssr:false baseline does NOT also show.

Classification buckets: `HYDRATION_MISMATCH` (`/Hydration .*mismatch|Hydration completed but contains/`), `UNRESOLVED_COMPONENT` (`/Failed to resolve component/`), `TEMPLATE_UNDEFINED` (`/accessed during render but is not defined/`), `MISSING_RENDER_FN` (`/missing template or render function/`), `ROUTER_NO_MATCH` (`/No match found for location/`), `INVALID_PROP` (`/Invalid prop/`), and HTTP noise (`401|400`).

**Run the same sweep under the ssr:false baseline.** Only the delta is SSR cost. Pre-existing defects (unimported components, undefined template bindings, bad prop types) appear in *both*, and are often *worse* client-only — the anonymous client fires 401s, so SSR rendering fewer warnings can look like an improvement.

## Traps that produced wrong numbers (all hit for real)

| Trap | Symptom | Guard |
|---|---|---|
| **Message bleed between routes** | Route A's warnings attributed to route B; the tell is the same class count appearing on an unrelated route | Clear the console buffer **before** `Page.navigate`, then let the page settle (~7s) before reading |
| **Dead listener / stale CDP session** | All buckets empty on a page you already measured as warning-heavy — impossible | An all-empty result on a known-bad page is a **broken capture**, not zero defects. Re-attach and re-run; never publish it |
| **Redirect stub double-counted** | The root route `index.vue` reports the target page's warnings | A page whose whole body is `await navigateTo(...)` is not a separate failure — collapse it into its target |
| **Anonymous 401 noise** | Interceptor/network errors inflate the FAIL count and get misread as SSR breakage | Classify HTTP noise separately; it is an artifact of testing without a session |
| **Proximity heuristic on timer sites** | A grep for `setTimeout` "not inside onMounted" returns the same count twice and double-counts timers only reached from mounted hooks | Not a breakage measure. The decisive test is the curl sweep (count non-200s), not grep distance |

## Step 3 — restore and verify (do not skip)

Restore `routeRules` from the byte-exact backup, then verify three ways before reporting anything:

1. `sed -n '<N>p' nuxt.config.ts` shows `ssr: false`;
2. `git diff --stat nuxt.config.ts` is **empty** (matches the commit);
3. `curl -s -o /dev/null -w '%{size_download}' <protected-route>` is back to the small shell (e.g. ~1.7 KB), not the SSR-rendered megabytes, and `grep -qiE '<leaked literals>'` finds nothing.

Also confirm no probe files remain (`find server -name '*zz*' -o -name '*probe*'`) and `git status --short server/` is clean. A dirty `nuxt.config.ts` left in a working tree ships the leak you were analyzing.

## Interpreting the result

- **Server gate only** (Question 1 = yes): keeps `ssr:false`; cheapest path to defeating a client-side auth-bypass PoC; no SSR risk, no cookie-Domain dependency.
- **SSR-on**: only justified for first-paint/SEO. Quote its cost as the **SSR-only delta** from Step 2, and remember it is normally *gated by* having a server-readable session — SSR-on with a client-only auth store re-opens the very leak it was meant to fix.
- **Cross-host deployments**: any gate reading a session cookie needs the cookie to reach the web host (`Domain=.example.com` or a same-origin BFF); otherwise it fails closed on every request, which is bad but strictly better than leaking.
