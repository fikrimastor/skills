---
name: tricipta-common-platform-login
description: "Log in to the Tricipta Common Platform (tricipta-edafy-asset-web) via Keycloak SSO with the dev testuser/testpass account. Covers the dev-server check, SSO and Keycloak form driving, cmux browser-backend quirks, and verifying the authenticated session. Use before live-verifying any asset-web UI change."
---

# Tricipta Common Platform Login

Use this skill when:
- the user asks to log in to the Tricipta Common Platform ("login to tricipta", "access common platform", "sign in as testuser")
- a login/auth flow needs driving end-to-end, e.g. before live-verifying an asset-web UI change.

## Stack (ports)

- **asset-web** Nuxt SPA: `http://localhost:3000` (repo `~/Sites/afed/tricipta-edafy-asset-web`)
- **asset-api** Go Echo: `:8085` (OIDC client `edafy-api`, callback `/api/v1/auth/callback`)
- **Keycloak** `edafy` realm: `:8180`

Dev credentials: `testuser` / `testpass` → user **Test User**, role **Surveillance Supervisor**.

## Procedure

1. **Check services** (bash): `nc -z 127.0.0.1 3000 && nc -z 127.0.0.1 8085 && nc -z 127.0.0.1 8180`. If the API and Keycloak are up, proceed.

2. **Check port 3000 before starting anything**: run `lsof -nP -iTCP:3000`. A dev server may already be running from a prior session, often with Brave connected. If one is present, do NOT start another. A second server fails port readiness and drifts to `:3001`, and the auth-callback redirects assume :3000. Use the existing server.
   Only if port 3000 is free: hub `start` `asset-web` → `npm run dev` (cwd `~/Sites/afed/tricipta-edafy-asset-web`), ready `{"port": 3000, "timeout": 90}`. If readiness times out, re-check `lsof`; the port may have been taken mid-start.

3. **Drive the browser** (`browser` tool):
   - Open `http://localhost:3000`. Depending on the branch, you land on `/login?redirect=/common-platform/dashboard` or on the login card rendered at `/` itself. The `/` case was seen on 2026-09-23 with title `Sign In | Tricipta - Surveillance Platform`. Either way the card has one "SIGN IN WITH SSO" button.
   - ARIA refs renumber on every snapshot. A ref taken in a previous eval cell came back `Element reference not found`. Snapshot and click by ref in the same cell, or click by CSS selector. On the login card, `tab.click("button")` (the first button) is the SSO button.

4. **SSO flow**:
   - The SSO click goes to Keycloak `localhost:8180/realms/edafy/protocol/openid-connect/auth?client_id=edafy-api...` (redirect_uri `http://localhost:8085/api/v1/auth/callback`).
   - The reported tab URL can lag. On the cmux backend, `tab.url()` kept returning `http://localhost:3000/` while the Keycloak form was on screen. Trust the title `Sign in to edafy` or the snapshot heading "Sign in to your account", not the URL. `tab.waitForUrl(u => u.includes("8180"))` can time out for the same reason.
   - Run `tab.fill("input#username", "testuser")` and `tab.fill("input#password", "testpass")`, then `tab.click("button[type=submit]")`. This realm's theme has no `input#kc-login`.
   - Keycloak then redirects to the API callback, which returns you to the app. If Keycloak already has an SSO session, submit redirects straight back.

5. **Verify authentication** (do not trust the URL alone):
   - Depending on branch, the landing page is `/common-platform/dashboard` or `/` (a landing page showing "Good afternoon, Test User"). Both are auth-gated.
   - The snapshot must show the header user chip: "Test User" + "Surveillance Supervisor".
   - `document.cookie` will NOT show `edafy_token`/`edafy_refresh`, because they are HttpOnly. Their absence is NOT a failure; the rendered identity is the proof.

## After login: common routes

- Diagnostic Plots dashboard (AI Insight drawer): `/common-platform/diagnostic-plots-dashboard`. `/common-platform/diagnostic-plots` does not render the dashboard.
- The capability picker is the button with class `min-w-[200px]`.
- The drawer's Evidence section and AI Analysis Summary footer render only in the report view. Click "View full analysis" to open it.

## Browser backend quirks (cmux)

- `tab.ariaSnapshot(selector?)`: the first argument is a selector string. Passing an options object throws `'[object Object]' is not a valid selector`.
- `tab.waitFor(...)` and `tab.text(...)` take selectors only. `tab.waitFor(2500)` and a bare `tab.text()` both throw. Sleep with `await new Promise(r => setTimeout(r, ms))` and read the page with `tab.evaluate("document.body.innerText")`.
- `tab.run(async ({ page }) => …)` runs outside the page, so a bare `document` throws ReferenceError. Use `page.evaluate(() => …)` inside it. `page.evaluateHandle` is unavailable.
- `tab.frames()` is unsupported.
- `tab.screenshot({ selector })` scrolls the element into view but returns the full viewport; it does not clip to the element.
- The Nuxt dev server reloads the page after a source edit (HMR). After an edit the page came back on the default capability in card view. Re-select the page state before measuring.

## Gotchas

- `tab.waitForUrl` on the redirect chain can time out even when the login succeeded. After a short delay, re-check the title and identity chip before concluding failure.
- The browser profile keeps cookies between sessions, so a previous successful login may land straight on the app. If so, skip to verification.
- If the app shows but the title says "Sign In", the session did NOT restore. Redo from step 3.
- The login page's slide-show buttons ("Go to slide N") are decoration; ignore them.
