---
name: auditing-nuxt-route-protection
description: "When auditing whether a Nuxt 3/4 app's route protection holds: doc-checked plus live probes."
---

# Auditing Nuxt route protection (live, not on paper)

Two halves, both needed: **(A)** does each mechanism match a documented Nuxt idiom, and **(B)** does the running
app actually behave that way. An idiom checklist on a broken implementation is worthless; a live probe with no doc
grounding can't tell a sanctioned design from an accident.

Use when asked "is this app's route protection correct", "check our auth gating against best practice", before or
after fixing an auth-bypass issue, or before claiming "routes are protected".

## A. Fetch the docs first — training-data Nuxt knowledge is stale

Via the Nuxt MCP (`list-documentation-pages` with a search term, then `get-documentation-page`; the page accepts a
`sections` array — use it, `nuxt-config` is huge):
`/docs/4.x/directory-structure/app/middleware`, `/docs/4.x/guide/recipes/sessions-and-authentication`,
`/docs/4.x/guide/concepts/rendering`, `/docs/4.x/directory-structure/server`, `/docs/4.x/api/nuxt-config`
(`ssr`, `spaLoadingTemplate`, `routeRules`, `pages`), `/docs/4.x/api/utils/define-page-meta`.

Raw markdown (`https://nuxt.com/raw/docs/4.x/...`) is the fallback: the MCP has been observed returning the wrong
page for a requested slug — if the returned frontmatter `title` does not match the request, fetch raw.

## B. Doc-compliance axes — credit the matches, don't just list misses

1. **Is the global middleware actually global?** Only `app/middleware/*.global.{ts,js}` runs on every route (doc:
   "Global route middleware, placed in the `app/middleware/` with a `.global` suffix"). A bare `auth.ts` is
   **named** — it runs only where a page declares it. An AGENTS.md/README often claims "Global protection:
   `app/middleware/auth.ts`" for a file with no suffix, and every page relying on it by omission is then unguarded.
   Check the filename suffix AND `addRouteMiddleware(name, fn, { global: true })`.

   If it is named, build the real unprotected set: page has no `middleware` meta **AND** no global middleware
   covers its path prefix (a `.global` gate may early-return, e.g. `if (!to.path.startsWith('/common-platform'))
   return`). Do NOT `grep -L middleware` alone.

2. **`routeRules` `ssr:false` is the sanctioned SSR gate** — the rendering doc's own example is
   `'/admin/**': { ssr: false }`, with `/login: { ssr: true }` as the default-deny idiom. Two preconditions:
   - **Hybrid rendering requires a server output.** Grep the Dockerfile/CI for `NITRO_PRESET=node-server` and
     `.output/server/index.mjs`. If deploy is `nuxt generate`/static, route rules silently do nothing (doc:
     "Hybrid Rendering is not available when using `nuxt generate`"). This is the single most important check.
   - **`ssr:false` wants `~/spa-loading-template.html`** (doc: "If you do use `ssr: false`, you should also place an
     HTML file in `~/spa-loading-template.html`"). Missing = blank first paint. Verify with
     `ls app/spa-loading-template.html spa-loading-template.html` and by grepping the anonymous response for
     `__nuxt-loader`.
   - `definePageMeta({ ssr: false })` is a per-page alternative but forces `experimental.inlineRouteRules`.

3. **Ordering.** Numeric prefix on `.global` files is documented ("Ordering Global Middleware", `01.setup.global.ts`),
   and filenames sort as **strings** — `10.` before `2.`. Verify the gate you care about runs first.

4. **`import.meta.server` / `import.meta.client`, not `process.server`.** Every doc example uses the `import.meta`
   form. Grep both (`grep -rn 'process\.server' app/ | wc -l` vs `import\.meta\.server`) and report the split —
   legacy files are pre-existing debt; new code copying the legacy form is a deviation worth flagging. Don't demand
   a rewrite.

5. **`pages:extend` hook** — middleware doc, section "Setting Middleware at Build Time", has a recursive walker
   setting `page.meta.middleware`. When an implementation instead hand-maintains an allow-list + protected manifest
   + completeness test, name this as the documented alternative that makes drift structurally impossible, and
   weight it heavily. The doc's own inline warning: it **overrides** any middleware set in `definePageMeta` — so
   specify **merge, not replace** (`page.meta.middleware = [...toArray(page.meta.middleware), 'auth']`); a naive
   assign silently drops a `['auth','admin']` array. Never present it as a drop-in replacement for hand-maintained
   manifests.

6. **Client middleware is UX; the boundary is the server** — auth recipe: *"Client-side middleware is helpful for
   the user, but without server-side protection your data can still be accessed."* If the subject relies only on
   route middleware, say plainly that the API is the real boundary.

7. **"Reuse the existing endpoint" must be grepped, not inferred from an architecture diagram.** Docs describe
   *intent and direction* of traffic, not the endpoint surface:

   ```bash
   grep -rn 'plots/render\|plots/capabilities\|/v1/plots' internal/ --include='*.go'   # 0 hits = does not exist
   grep -n 'func (c \*Client)' internal/infrastructure/<client>/*.go                    # actual method surface
   ```

   A client with one method (`Analyze → POST /v1/analysis/capabilities`) does **not** cover two other proxied
   paths. "Small guard" then silently becomes "new backend endpoints" — a scope error of an order of magnitude.

For the full route/middleware inventory (finding the *other* unauthenticated endpoints) see
`unprotected-route-surface-audit`.

## B2. Claim-verification rules (each one produced a wrong published verdict)

**"No `definePageMeta({middleware})`" ≠ "ungated".** Before claiming a page leaks data: (1) grep the page
**template** for a permission-guard wrapper (`<DynectPermissionGuard>`, `<PermissionGuard>`, `<Can>`, …) —
`sed -n '1,12p' page.vue` shows it immediately, and `grep -rn 'PermissionGuard' app/pages | wc -l` sizes the surface;
(2) read the guard component and find the `v-if` on the `<slot>`; (3) trace the guard's inputs to a
**server-readable** source — `useUserSession().session.*` with `nuxt-auth-utils` as a direct dependency gates **SSR**
too (no session → empty permissions → slot not rendered), while a localStorage-backed store only gates the client.
Only after 1–3 is "ungated" supportable. Absence of route middleware often just means the page renders an "Access
Restricted" panel instead of redirecting to `/login` — a UX gap, not exposure, and that distinction is the
difference between a Critical finding and a nit.

**The credential injector is the guard's real precondition.** `const isAssetApiUrl = (url) => url.startsWith(apiBase)`
means `prepareApiFetchInit` (the Authorization injector) runs **only** for the API origin, so app code calling Nuxt
server routes (`$fetch('/api/...')`) arrives with no auth header. Enumerate the alternatives honestly: a **cookie**
(viable only if the cookie is sent to the Nuxt host — separate deployment hosts make that an *unverified*
assumption; state it as such), a **JWT verifier on the Nitro side** (`jose` / a *direct* `jsonwebtoken` dep — check
the lockfile for transitive entries such as via `@azure/msal-common`, present there but absent from `node_modules`
top-level and therefore unusable), or a **session library** (`nuxt-auth-utils` → Nitro owns a server-readable session
and a BFF proxy is the clean answer). If none hold, the honest fix is a new authenticated backend path, not a guard.

**`pages:extend` overrides, it does not merge** — a naive `page.meta.middleware = ['auth']` silently drops an array
declaration such as `['auth','admin']`, losing the admin gate.

## C. The blind spot that survives every SSR fix: Nitro server routes

Route middleware runs in the **Vue app only** (doc: "completely different from server middleware") and **never runs
for `server/api/**`**. `ssr:false` stops the *page* leaking; it does nothing for a caller hitting `GET/POST
/api/...` directly by URL.

1. `find server -type f` — does `server/middleware/` even exist?
2. Read every `server/api/**` handler. A proxy that injects a server-held API key (`grep -rn 'API_KEY\|X-API-Key'
   server/utils/`) while performing **zero** auth checks is an unauthenticated gateway — and if it accepts
   attacker-supplied payload (a `records` array, a query), it is unauthenticated **compute**, not just a read.
   State that distinction; it changes severity.

```bash
curl -s -o /tmp/o -w '%{http_code} %{size_download}\n' http://localhost:3000/api/<proxy>/capabilities --cookie 'none=1'
curl -s -o /tmp/r -w '%{http_code} %{size_download}\n' -X POST http://localhost:3000/api/<proxy>/render \
  -H 'Content-Type: application/json' -d '{"capability":"...","data_source":{"kind":"sample"},"options":{},"output_formats":[]}' \
  --cookie 'none=1'
```

A 200 with a real body proves the gap; do this rather than reasoning from source.

### Then ask: does the server have a credential to verify?

Do not assume it does.

- Is there a first-party JWT verifier? `jq '.dependencies' package.json | grep -iE 'jose|jsonwebtoken'`. A
  `jsonwebtoken` entry in `package-lock.json` is often **transitive** (via `@azure/msal-common`) — present in the
  lock, absent from `node_modules` top-level, unusable. Trace it before counting it.
- Is there a server-owned session? `nuxt-auth-utils` + `requireUserSession` / `getUserSession` → yes. A
  localStorage-Bearer design → no.
- **Does the browser even send credentials to the proxy?** The silent killer. Find the fetch wrapper and its URL
  matcher; a common shape is `isAssetApiUrl = (url) => url.startsWith(apiBase)` where `apiBase` is the *backend*
  origin, so `$fetch('/api/<local-proxy>')` gets **no** `Authorization` — a header-checking guard is then a no-op in
  production. A cookie-checking guard depends on cookie domain topology, which for separately-hosted web/API is
  frequently unverified (`SameSite=Lax` withholds cookies cross-site).

Say explicitly which credential the fix assumes. A guard that no-ops in prod is worse than no guard.

## D. Live two-directional probe (curl is enough for SSR, not for the client half)

```bash
for p in /common-platform/dashboard /landing /well-analytics /chemical-usage; do
  printf '%-32s ' "$p"
  curl -s -o /tmp/o.html -w 'HTTP %{http_code} bytes=%{size_download}' "http://localhost:3000$p" --cookie 'none=1'
  echo -n ' | leaks: '
  grep -o -i -E 'good afternoon|ANDP-|ANDR-DP-|<real leaked strings>' /tmp/o.html | sort -u | tr '\n' ','
  echo
done
```

A correct `ssr:false` result is **HTTP 200 with a small constant byte count identical across all protected paths**
(e.g. 2107 B), body `<div id="__nuxt"></div>` + `data-ssr="false"`. Then confirm the public allow-list was not
swallowed by the `/**` catch-all: `/login` must still return the real, much larger HTML (assert a literal form
string). An empty shell ≈ 2 KB vs a leaked dashboard ≈ 100 KB+ is a fast tell.

**`curl` cannot verify the client-side half.** `ssr:false` alone passes an SSR-only assertion while client-side
navigation remains reachable. That needs a browser (`browser.open`, authenticated and unauthenticated navigation,
both directions).

**The authenticated inverse is the likelier real failure** of a wrong allow-list entry: log in, hit the same
routes, confirm real content. An over-broad gate locks real users out and looks identical to a working gate from
the anonymous side.

## D0. The SSR-bypass triad — check all three, not the guard alone

When a pentest/Burp report says the app rendered an authenticated page without a valid session, reading the
middleware alone gives the wrong answer; the bypass lives in the interaction between three files.

**1. SSR guard bail.** The global route middleware returns early on the server, so SSR renders the protected route
with no session:

```js
// app/middleware/<area>.global.js
export default defineNuxtRouteMiddleware((to) => {
  if (!to.path.startsWith("/protected")) return;
  if (process.server) return;          // <-- SSR skips the gate entirely
  const authStore = useAuthStore();
  if (!authStore.isAuthenticated) return navigateTo("/login");
});
```

A comment in the auth *plugin* saying "async plugin — Nuxt awaits this before mounting, so middleware always sees
the correct auth state" is true **client-side only**. Do not accept it as covering SSR.

**2. `isAuthenticated` is local state.** `const isAuthenticated = computed(() => !!user.value)` over an in-memory
ref (or a token in localStorage) is not a security boundary — the server is. Check whether anything re-validates
against the server before render.

**3. Auth endpoints excluded from the 401 interceptor.**

```js
const shouldAttemptRecovery = (url) =>
  !/\/auth\/(login|register)(?:\?|$)/.test(url) &&
  !/\/auth\/me(?:\?|$)/.test(url) &&
  !/\/auth\/refresh(?:\?|$)/.test(url);
```

There is a legitimate reason (the original fix stopped 401s kicking users mid-recovery, e.g. #497-style premature
logouts). The side effect: a `/auth/me` **401 triggers no recovery and no redirect**, so stale in-memory `user`
survives failed re-validation and the synchronous guard keeps rendering. Also check `clearBearerToken()` vs
`clearSession()` — partial clears that null the token but keep `user` are exactly how stale state persists.

**Do not re-pin the guard to the symptom.** Fixing `isAuthenticated` to check a token still leaves SSR ungated;
fixing the SSR bail still leaves stale local state. Address all three, then re-run the anonymous `curl` and assert
the protected content strings are gone (or the response is a redirect). A unit test on the middleware alone will
not catch the SSR bail — the guard returns before its own assertion.

**Severity triage: shell leak vs data leak.** After proving anonymous render, do not call it a data breach yet.
(1) Does any protected page fetch server-side? `grep -rn "useAsyncData|useFetch(" app/pages/<area>/` — hits mean real
data is in the SSR HTML (serious); no hits (all fetching in `onMounted`) means SSR emitted only what the component
renders synchronously. (2) Grep each leaked value back to frontend source:

```bash
for v in "15,752" "ANDR-DP-A" "120.9"; do
  grep -rn --exclude-dir=node_modules --exclude-dir=.git --exclude=.env -- "$v" app/ server/ | head -3
done
```

Values tracing to a mock module (`app/utils/*Data.ts`, often stamped "ported 1:1 from <prototype>") mean the
exposure is **UI-shell + internal identifiers + fake numbers** — information disclosure, not customer data.
Greeting fallbacks like `useDisplayName(fallback = "User")` are another tell that the render is optimistic.

## D0b. Check the token-minting surface on the backend

A client-side gate is only half. Probe non-destructively and read the handler:

```bash
# Non-destructive route existence probe: {} -> 400 "field is required" proves the route exists and writes NO state.
# A bogus path -> 401/404 is the baseline.
curl -s -X POST "https://<api-host>/api/v1/auth/register" -H 'Content-Type: application/json' -d '{}'
```

Red flags in the handler source:

- A registration handler **without** the `if keycloak.Enabled()` gate its sibling login handler has (asymmetry is
  the bug).
- It creates a tenant + user + membership with `Role: "admin"`, `IsActive: true`, `EmailVerified: false`, and
  returns a signed JWT.
- No rate limiter attached (check whether only `/login` gets the limiter) and no global limiter.
- **The JWT middleware accepts a legacy symmetric fallback** (`HS256`) alongside the IdP's asymmetric algorithm:

  ```go
  if config.OIDCValidator != nil {
      if claims, err := config.OIDCValidator.ValidateToken(t); err == nil { /* accept */ }
      // Fall through to legacy if OIDC validation fails
  }
  claims, err := config.JWTManager.ValidateToken(t)   // <-- HS256 accepted
  ```

  This composes with an open register route into **full auth bypass**: the route mints the token, the fallback
  accepts it, even with the IdP enabled. Migrating to an IdP does not close the legacy path — it must be retired.

Also confirm the secret behind the symmetric path is strong; an empty/weak `JWT_SECRET` allows forging tokens
without touching the register route at all.

**Never send a real email/tenant name** to a register endpoint while probing — empty `{}` only. And do not sweep
`.env` with unscoped repo-root greps; scope to `app/`/`internal/` with `--exclude=.env`.

**Host mismatch trap:** the web origin often does **not** proxy the API. A 404 for `/api/v1/*` on the SPA host
(Nitro/Next error shape: `{error:true,url:...,statusCode:404}`) does not mean the endpoint is absent — probe the real
API host (`api.` / `dev.api.`) separately.

## E. Prove a route-coverage test is non-vacuous — do this every time

A completeness test whose assertion is "in the allow-list **or** covered by default-deny" is a **tautology** — true
of every file including an unclassified one, so it can never fail. The real assertion needs a second explicit
*protected* manifest and must check membership, not absence-from-allow-list. Then prove it bites:

```bash
# Plant a file that belongs to neither list (prefer a temp probe over editing the manifest: it exercises the real
# fs walk and leaves the tree clean).
printf '<template><div>probe</div></template>\n' > app/pages/__nongated_probe.vue
npx vitest run test/utils/publicRoutes.test.ts   # MUST fail (2 failed | 1 passed)
rm app/pages/__nongated_probe.vue
npx vitest run test/utils/publicRoutes.test.ts   # MUST pass (3 passed)
```

Report both runs. A test that stayed green while an unclassified route existed is the finding.

Also count both numbers when a spec cites page totals: `find app/pages -type f -name '*.vue' | wc -l` (all page
files) vs `grep -rl definePageMeta app/pages --include='*.vue' | wc -l` (files with meta). They legitimately
differ; a spec conflating them is a wording bug, not a design bug.

## F. Traps to reject on sight

**`routeRules: {'/path': {appMiddleware: {'00-protected': false}}}` CANNOT disable a `.global` middleware.**
In `node_modules/nuxt/dist/pages/runtime/plugins/router.js` the entry set is seeded with guard **objects**
(`new Set([...globalMiddleware, ...nuxtApp._middleware.global])`) while the `appMiddleware` false branch calls
`middlewareEntries.delete(key)` with a *string* — a no-op. It only adds/removes **named** middleware. Verify against
the installed version before recommending; the config docs list `appMiddleware` without this caveat. Someone will
propose this to exempt `/login` from a default-deny global gate; it will not work.

## G. Report shape

Verdict first. Then: a **matches** table crediting sanctioned idioms (with the doc quote) → **deviations** ranked
by value with `file:line` → **live evidence** (curl output verbatim) → **checked-and-rejected** (traps ruled out,
with the source line that proves it) → **ranked next actions with concrete time estimates** (a guard needing a new
verifier or a client change is hours, not minutes). Close with explicit non-issues: when auth is external
(Keycloak/OIDC) with tokens held against a separate API, the absence of `nuxt-auth-utils`/`requireUserSession` is
**not** a violation — Nitro holds no session to validate. State that so the report doesn't read as an idiom
checklist dump.
