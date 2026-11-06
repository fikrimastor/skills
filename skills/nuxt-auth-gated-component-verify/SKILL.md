---
name: nuxt-auth-gated-component-verify
description: "When verifying an auth-gated Nuxt component without login credentials or a working browser."
---

## Problem

A component you just changed only ever renders inside a page gated by a global auth middleware (e.g. checks `useAuthStore().isAuthenticated` and redirects to `/login` for paths under some prefix). You have no login credentials, and either there's no browser-relay to the user's own logged-in Chrome tab, or the `browser`/CDP tool itself is failing in the sandbox (e.g. "Target closed", relay attach timeout). You still owe real, grounded verification evidence, not a fabricated "looks right" claim.

## Check the gate first

Read the global middleware file(s) (Nuxt: files ending `*.global.js`/`*.global.ts` under `app/middleware/`). Confirm exactly which path prefix is gated and on what condition (e.g. `if (!to.path.startsWith('/some-prefix')) return;`). Routes outside that prefix are not gated by that middleware — confirm there isn't a *second* global middleware that gates everything unconditionally before relying on this.

## Procedure

1. Create a temporary page under the app's pages directory, at a route path that does **not** start with the gated prefix (e.g. `app/pages/dev-<component>-verify.vue` when the real thing lives under `/common-platform/...`).
2. Import the real component (not a re-implementation) and mount it with realistic props. Prefer building those props from the same exported constants/fixtures the real page already uses (shared `utils`/config modules), so the harness exercises real data shapes instead of hand-rolled ones that might not match the component's actual contract. Cover: the normal case, the edge case you're actually fixing (e.g. a long list forced to overflow), and any adjacent branch that shares code with what you changed (regression check).
3. Start the dev server (check `hub jobs`/`hub ps` and `lsof -nP -iTCP:<port> -sTCP:LISTEN` first — a dev server may already be running under a different, unmanaged process; don't fight over the same port, and don't leave a duplicate running when you're done).
4. Fetch the harness route with the `read` tool (`read http://localhost:<port>/<harness-route>`). This works over plain HTTP and doesn't need the `browser` tool. Confirm:
   - No 500 / Vue compile error surfaced in the response.
   - Every expected data item appears in the rendered markup (e.g. every list entry present, not silently truncated — this is exactly the kind of bug an SVG-viewBox-clipping issue produces, and `read`'s extracted markup will show items that exist in the DOM even if you can't see whether they'd be visually clipped).
   - Inline `<svg>` elements often come back from `read` as `data:image/svg+xml;base64,...` — decode the base64 to get the literal rendered SVG markup and check exact computed values (viewBox, coordinates, tick labels) against what the component's logic should have produced. This is real evidence of correct computation, not just "it rendered something."
5. Delete the harness page immediately after use. It is throwaway verification scaffolding, not a permanent test or route — leaving it in the repo is undelivered cleanup.
6. If a real browser/relay is genuinely available, prefer it — actual pixel/scroll/interaction confirmation is stronger evidence than SSR markup inspection. This procedure is the fallback when that path is unavailable, not the default.
7. Be explicit with the user about the gap: state plainly that you verified rendering/computation via SSR markup, not live pixels/interaction in the real authenticated page, and ask them to do the final live check themselves if the change has any visual/interaction nuance (e.g. scrollbars, hover states, animations) that markup inspection can't confirm.
