---
name: nitro-server-gate-nuxt-spa
description: "When securing Nuxt SPA (ssr:false) routes server-side: Nitro gate design and coverage."
---

# Server-side route gating for a Nuxt SPA (`ssr:false`) via Nitro

Answers: *can Nitro gate the routes, what breaks, what does it actually cover, and is the change small?* Prove
every claim on the running **production** build (`node .output/server/index.mjs`), never dev alone — dev and prod
differ, and Nuxt 4 docs are **silent** on the `ssr:false` × server-middleware interaction.

Sibling skills: `verify-auth-gate-against-devtools-override` (proving/refuting the reported override) and
`auditing-nuxt-route-protection` (the wider route-protection audit this gate is one layer of).

## 0. Ground the docs, then distrust them on the gap

Via the Nuxt MCP (`get-documentation-page`; a `sections` array avoids pulling the huge `nuxt-config` page):

- `/docs/4.x/directory-structure/server` — server middleware *"will run on every request before any other server
  route to add or check headers"*; examples only, never `sendRedirect`/`getCookie`.
- `/docs/4.x/guide/concepts/rendering` — `ssr` is defined only as *"Disables server-side rendering of the HTML"*.
  **No statement that it blocks server-side gating.**
- `/docs/4.x/guide/recipes/sessions-and-authentication` — *"Client-side middleware is helpful for the user, but
  without server-side protection your data can still be accessed."*
- Hybrid rendering requires a **server output**: `nuxt build` + `NITRO_PRESET=node-server` + `.output/server/index.mjs`.
  `nuxt generate`/static hosting kills route rules silently.
- `ssr:false` wants `~/spa-loading-template.html` (rendering doc) — missing file = blank first paint.

## 1. Plant a probe that reports its own decision

```ts
// server/middleware/zz-probe-gate.ts — TEMPORARY, delete after capture
export default defineEventHandler((event) => {
  setResponseHeader(event, "x-zz-probe", "fired");
  setResponseHeader(event, "x-zz-probe-decision", isPublic(path) ? "public" : "gated");
});
```

```bash
npx nuxt build && PORT=3101 NODE_ENV=production node .output/server/index.mjs &
curl -D- http://localhost:3101/<protected-route>     # require x-zz-probe: fired
```

Non-vacuous control: record the anonymous byte count with NO gate. A correct `ssr:false` protected response is a
small **constant** (e.g. 2107 B; `<div id="__nuxt"></div>` + `data-ssr="false"`) and **identical across different
protected paths**, vs a public `ssr:true` page (~21 kB). Identical byte counts across routes = nothing rendered
server-side.

`sendRedirect` from middleware works on such routes: expect `302` + `location:` + a ~107 B body, **not** a 200
shell. Ignore the docs' "middleware should not respond" phrasing — `sendRedirect`/`createError` are standard Nitro
utilities.

## 2. Coverage matrix — measure which requests reach the middleware

Do not assume; results are inconsistent across sources and must be probed per route:

| Request kind | Nitro middleware |
|---|---|
| document (`Sec-Fetch-Dest: document`) | **fires** |
| XHR to `server/api/**` (`Sec-Fetch-Dest: empty`) | **fires** |
| real client-side SPA transition (`router.push`) | **never fires — no request exists** |
| served static asset (`public/*`, e.g. `/favicon.ico`, `/logo.png`) | bypasses entirely |
| `/_nuxt/*.js` asset | **inconsistent**: one measurement says it reaches middleware, another says it bypasses — probe it |

Two prior measurements disagree on two rows: one recorded `/_nuxt/*` reaching the middleware, another that it
bypasses; one recorded a "client-side nav route fetch" as firing, another that a real `router.push` never fires.
Both are reconcilable (`router.push` makes no request at all, so the "fetch" row was measuring a synthetic XHR) but
the asset row is genuinely per-route. **Probe your own app for each row and cite the probe**; do not quote this
table as fact.

**The synthetic-request trap:** `curl -H 'Sec-Fetch-Dest: empty' /protected-route` *fires* the middleware and looks
like client-nav is covered. It is not — `router.push` issues no HTTP request at all. Verify honestly: log
`console.log('[ZZREQ]', url.pathname)` in the probe, build, restart for a clean log, then in a browser drive a
genuine transition (`const app = window.useNuxtApp(); await app.$router.push('/protected')`) and grep the log. If
the only line is the initial document load, SPA nav is invisible to Nitro and the gate cannot stop it — that is
also why a gate must never be exempted from `/api/**`: a proxy injecting a server-held upstream API key, reachable
anonymously, is the real data leak.

Never rely on the asset-exemption clause as load-bearing; it is defence-in-depth only.

## 3. Trust signal — can middleware read the session?

Grep the app for a server-held session (`grep -rn "useSession|requireUserSession|getUserSession|nuxt-auth-utils"`).
Empty + client-only `isAuthenticated` + `if (process.server) return` in every app middleware ⇒ the whole trust chain
is client state and the client gate is cosmetic. The Nitro gate must derive trust independently: read the IdP's
cookie with `getCookie(event, ...)` and validate it **server-to-server** against the API
(`$fetch(api + '/auth/me', { headers: { cookie } })`) — structurally outside any browser-side override.

### 3a. Credential reachability decides whether a prod gate is even possible — check BEFORE proposing it

| Read | Where | Decides |
|---|---|---|
| Frontend's API base | web CD workflow (`NUXT_PUBLIC_API_BASE` / `runtimeConfig.public`) | same-origin or cross-origin? `public` runtimeConfig is baked into the client → cross-origin |
| Cookie `Domain` the API sets | API cookie helper (`cookieDomain = u.Hostname()` of `BACKEND_URL`) | which host the browser returns it to |
| Hosts in the deployed topology | CD secret refs (`secretref:api-base`, `secretref:be-url`) | whether those two hosts are the same origin |

**If the cookie is scoped to the API host and the web host is a different origin, the web server sees no
credential and a gate there cannot work in production.** It can appear to work locally because both hosts are
`localhost` and cookies ignore ports — the classic false green. Precondition for a production server-side gate:
same-origin BFF, or a shared parent-cookie `Domain`. Name that precondition, or scope the gate to dev explicitly.

Cookie matching rule: host `H` receives `Domain=D` iff `H == D` or `H.endsWith('.' + D)`. Sibling subdomains
(`dev.asset.example.io` + `dev.api.asset.example.io`) share only their tier root (`asset.example.io`) or the
registrable domain (`example.io`) — scope to the tier root, or a multi-tenant suffix hazard applies (§6).

Also: a **preflight (`OPTIONS`) in the incident traffic proves cross-origin**, and `Sec-Fetch-Site: same-site`
means same registrable domain but a *different* origin — do not read same-site as same-origin, or you drop the
cookie-Domain fix and the gate fails closed for every real user. Credentialed cross-origin XHR needs BOTH
`Access-Control-Allow-Origin` AND `Allow-Credentials`; grep *every* CORS registration and check its env guard — an
`APP_ENV == "local"`-gated `CORSWithConfig` plus a CD workflow hardcoding `APP_ENV=development` means deployed
environments emit no CORS headers at all.

## 4. Gate design

```ts
// server/middleware/00-auth-gate.ts  (numeric prefix; runs before other server middleware)
import { PUBLIC_ROUTES } from "~~/app/utils/publicRoutes";   // reuse the app's allow-list

const isPublic = (p) => PUBLIC_ROUTES.some((r) => p === r || p.startsWith(`${r}/`));

export default defineEventHandler(async (event) => {
  const path = getRequestURL(event).pathname;
  const isApi = path.startsWith("/api/");
  const dest = getHeader(event, "sec-fetch-dest");
  const isDocument = !dest || dest === "document";

  if (!isApi && !isDocument) return;                      // assets, payloads
  if (!isApi && isPublic(path)) return;                   // public allow-list

  const apiBase = useRuntimeConfig(event).public.apiBase;

  const access = getCookie(event, "edafy_token");
  if (access) {
    try { await $fetch(`${apiBase}/v1/auth/me`, { headers: { cookie: `edafy_token=${access}` } }); return; }
    catch { /* fall through to refresh */ }
  }

  const refresh = getCookie(event, "edafy_refresh");
  if (refresh) {
    try {
      const res = await $fetch.raw(`${apiBase}/v1/auth/refresh`, {
        method: "POST", headers: { cookie: `edafy_refresh=${refresh}` },
      });
      for (const sc of res.headers.getSetCookie?.() ?? []) appendResponseHeader(event, "set-cookie", sc);
      return;
    } catch { /* deny below */ }
  }

  if (isApi) throw createError({ statusCode: 401, statusMessage: "Unauthorized" });
  return sendRedirect(event, `/login?redirect=${encodeURIComponent(path)}`, 302);
});
```

Rules that are load-bearing:

- **Exempt non-document traffic first, and be explicit.** A gate keyed only on the public allow-list will 302 the
  redirect target's OWN JS/CSS for anonymous users, breaking the very page it redirects to. Verify by curling every
  `/_nuxt/*` referenced in the login HTML with `Sec-Fetch-Dest: script`.
- **Never exempt `/api/**` as a class.** "Skip `/api/**`" and "exempt non-document" are different rules;
  conflating them leaves the data proxy open while the page looks protected. Skip `/api/**` deliberately only if
  the gate is for documents, and say so.
- **Deny semantics differ by kind:** `401` for XHR (a `302` feeds login HTML to `$fetch` and the client sees a parse
  error, not an auth failure); `302` for documents.
- **Forward the refresh `Set-Cookie`** to the browser before allowing, or the renewed access cookie never lands and
  every subsequent load re-runs a server-side refresh. **Assert the renewal, not the 200:** the forwarded refresh
  cookie's `Max-Age` must be slightly under the full lifetime (e.g. `28791` vs `28800`) — a passthrough of the old
  value proves nothing, and a test that only checks `HTTP 200` will not catch a discarded `Set-Cookie`.
- **Keep the client gate and the `process.server` bail** — client middleware remains the UX half; the Nitro gate is
  the boundary. No `ssr` flip needed.

## 5. Verification cases (production build, curl) — publish this table

1. anon document → protected: `302` + `location:`
2. anon document → public route: `200` with real HTML
3. valid access cookie: `200`
4. **expired access + valid refresh cookie**: `200` *and* ≥1 `set-cookie` forwarded with a short `Max-Age`
5. forged access cookie, no refresh: `302`
6. anon XHR → `/api/**`: **`401`** (assert a small body — an error, not kilobytes of data)
7. authed XHR → `/api/**`: pass-through
8. login page's own assets + favicon: `200`
9. client-nav fetch (`Sec-Fetch-Dest: empty`): `200`

Then delete the probe, `rmdir server/middleware` if it was empty, confirm `git status --short` is clean in
`server/`, and re-curl to show the route restored. **A hot-reloading dev process can keep serving a deleted
probe** — restart the server before concluding.

## 6. The multi-tenant suffix hazard (cookie `Domain` widening)

The rule is *never a PUBLIC SUFFIX or multi-tenant provider suffix* — not "never the bare registrable domain".
`azurecontainerapps.io` is absent from the Public Suffix List yet shared by every Azure tenant, and browsers
**accept** it (not being a suffix is why), so the dangerous case is silently accepted. Confirm against the list:

```bash
grep -x "<domain>" public_suffix_list.dat   # from github.com/publicsuffix/list
```

Scope Azure-native FQDNs to `<environment-unique-id>.<region>.azurecontainerapps.io`. A customer-owned registrable
domain (e.g. `trinovate.io`) is legal and safe, because only `io` is a PSL entry. A suffix absent from the PSL also
means sibling hosts are same-site, so SameSite is not the blocker there.

## 7. Deploy order

The web gate fails **closed**, so ship the cookie-Domain change to the API first, then verify the cookie actually
arrives on the web origin *before* merging the web gate:

```bash
curl -D- -H 'Sec-Fetch-Dest: document' -H "Cookie: edafy_token=$TOK" \
  https://<web-host>/<protected> | head -1     # must be 200, not 302
```

## 8. Honest scope note

Data endpoints on the **API origin itself** are not fixed by this, a shared cookie parent is visible to sibling
apps under that parent, and a client-side bypass needs API-side token validation rather than the gate. Do not
present an SSR-shell improvement as a fix for a client-side bypass.
