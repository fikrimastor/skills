---
name: verify-auth-gate-against-devtools-override
description: "When a DevTools override (fake 200 on /auth/me) is said to beat the gate: prove or refute."
---

# Prove or refute a DevTools response-override auth bypass

Use when a pentest/report says "I overrode `GET /auth/me` in DevTools and reached the protected page", when asked
"can attackers bypass the login page / how do we prevent this override", when a client-side gate is suspected
cosmetic, or before publishing a verdict either way.

## Reframe first (say this before anything else)

A response override, localStorage edit, or monkey-patched fetch runs **inside the attacker's own browser, after the
network**. No CSP, header, or code change reaches it. Shipping bytes to a machine means that machine's owner can
lie about them. So do not answer "how do we block this." Answer two separable questions:

1. **Is the demonstrated PoC actually capable of anything?** (usually: no)
2. **What can an unauthenticated attacker do WITHOUT the override?** (where the real finding lives)

The requirement is that the override be **inert** — the spoof yields a UI shell over denied/empty data. "Blocked"
is not a goal.

## Step 1 — read the gate, do not test it yet

Find the route guard (middleware/guard/interceptor) and the value it reads. The cosmetic signature is a three-link
chain:

- middleware trusts `store.isAuthenticated`
- store defines it as `computed(() => !!user.value)`
- `user` is populated from the `/auth/me` response body

If those chain, the override replaces exactly the trusted value — the attacker passed their own gate. Quote all
three file:line. Rendering a protected shell client-side is a self-deception, not an escalation.

Then ask whether a **server-held session** exists — this is the root-cause question:

```bash
grep -rn "useSession\|replaceUserSession\|getUserSession" app/ server/
```

- **Empty** → no server-held session; the client gate can only ever be cosmetic. (Tricipta asset-web, verified
  2026-09-21.)
- **Hits** → a sealed server cookie (e.g. `nuxt-auth-utils`: `useSession(event, { password, name })`) plus a
  server endpoint the client asks (`GET /api/_auth/session`). The browser cannot decrypt or forge the seal, so the
  override wins nothing. (dynect-hrms-web: `dynect_session`, client `useUserSession().loggedIn`.)

Compare the rest of the shape too: does the browser call the backend directly, or through a server proxy that
injects the token? Dynect only exposes `server/api/backend/[...path].ts` which attaches
`session.secure.accessToken`; a browser that cannot reach the API directly can't be overridden into fetching real
data.

Classify what the PoC rendered — bundled or fetched? If the "leaked" content is a compile-time literal it is
already public in the downloaded JS, so a spoof that merely *renders* it proves nothing:

```bash
grep -rl "<the-secret-looking-string>" .output/public/_nuxt/ 2>/dev/null \
  || grep -rl "<string>" dist/assets/ 2>/dev/null
```

Trace the component: a `utils/*Data.ts` / `constants` / `mock*` import means authored demo data. Watch for the
**silent-fallback trap**: a composable that fetches real names/ids but resets to bundled mock values in its
`catch`, so a failed or absent API call still renders plausible real-looking identifiers.

## Step 2 — the harness browser tool is the wrong instrument

The OMP/cmux `tab` page surface exposes only url, title, viewport, goto, evaluate, content, locator, `$`,
waitForSelector, waitForFunction, waitForResponse, screenshot. There is **no** `setRequestInterception`, no
`evaluateOnNewDocument`, no `cookies`, no `browser.newPage`, no `route`. Do not retry it. Also `tab.evaluate` is
**synchronous-only** — an async closure throws `returned a Promise, but this surface evaluates synchronously`; use
`tab.run` or raw CDP for anything async.

Launch Chrome yourself with a **fresh** profile (a reused `--user-data-dir` carries a live session cookie and turns
"logged-in user reached a page" into a false bypass report):

```bash
rm -rf /tmp/chrome-poc-*; TMP=$(mktemp -d /tmp/chrome-v-XXXX)
"/Applications/Google Chrome.app/Contents/MacOS/Google Chrome" --headless=new \
  --remote-debugging-port=9335 --user-data-dir="$TMP" \
  --no-first-run --no-default-browser-check "about:blank" >/tmp/chrome-v.log 2>&1 &
sleep 4; curl -s --max-time 5 http://127.0.0.1:9335/json/list | head -c 200
```

In `eval` (JS), open a `WebSocket` to the page target's `webSocketDebuggerUrl` and implement `send(method, params)`:

```js
const list = await (await fetch("http://127.0.0.1:9335/json/list")).json();
const page = list.find(t => t.type === "page");
const ws = new WebSocket(page.webSocketDebuggerUrl);
let seq = 0; const pending = new Map();
const send = (m, p = {}) => new Promise((res, rej) => {
  const id = ++seq; pending.set(id, { res, rej });
  ws.send(JSON.stringify({ id, method: m, params: p }));
});
ws.addEventListener("message", (ev) => {
  const msg = JSON.parse(ev.data);
  if (msg.id && pending.has(msg.id)) { const { res, rej } = pending.get(msg.id); pending.delete(msg.id);
    msg.error ? rej(new Error(JSON.stringify(msg.error))) : res(msg.result); }
});
await new Promise(r => ws.addEventListener("open", r));
await send("Page.enable"); await send("Runtime.enable"); await send("Network.enable");
```

### Run A — anonymous baseline FIRST (no interception)

```js
await send("Storage.clearDataForOrigin", { origin: "http://localhost:3000", storageTypes: "all" });
await send("Page.navigate", { url: "http://localhost:3000/login" });
// then navigate the protected route and assert it redirects to /login?redirect=…
```

Confirm anonymity before claiming anything: `document.cookie`, `localStorage.getItem('auth-token')`,
`localStorage.getItem('auth-user')` all empty. Without a clean profile, a logged-in session makes "no redirect" look
like a bypass. The baseline must be an observed redirect, not an assumption — and without it "the override was
defeated" is unfalsifiable.

### Run B — intercept with `Fetch.enable` at the Response stage

```js
await send("Fetch.enable", { patterns: [{ urlPattern: "*", requestStage: "Response" }] });
ws.addEventListener("message", async (e) => {
  const m = JSON.parse(e.data);
  if (m.method !== "Fetch.requestPaused") return;
  const p = m.params;
  try {
    if (p.responseStatusCode === undefined || p.request.method === "OPTIONS")
      return void await send("Fetch.continueRequest", { requestId: p.requestId });
    const isDoc = (p.responseHeaders || []).some(h => /^content-type$/i.test(h.name) && /html/i.test(h.value));
    if (isDoc) return void await send("Fetch.continueRequest", { requestId: p.requestId });
    if (!p.request.url.includes("/auth/me"))
      return void await send("Fetch.continueRequest", { requestId: p.requestId });
    fired.push({ url: p.request.url, at: Date.now(), orig: p.responseStatusCode });  // PROOF IT FIRED
    await send("Fetch.fulfillRequest", {
      requestId: p.requestId, responseCode: 200,
      responseHeaders: [
        { name: "Content-Type", value: "application/json" },
        { name: "Access-Control-Allow-Origin", value: "http://localhost:3000" },
        { name: "Access-Control-Allow-Credentials", value: "true" }],
      body: Buffer.from(JSON.stringify({ user: { id: "1", name: "Security Tester", role: "admin" }, role: "admin" })).toString("base64"),
    });
  } catch {}
});
```

Non-negotiables in that handler:

- **`responseStatusCode === undefined` is the request stage**, not the response stage — continue it and do not count
  it. `OPTIONS` preflights also arrive; a naive `includes("/auth/me")` match otherwise hijacks the preflight.
  Preferring `requestStage: "Response"` avoids most of this.
- **`fulfillRequest` does NOT inherit the original response headers.** Omitting `Access-Control-Allow-Origin` on a
  cross-origin XHR makes the override fail as a CORS error — a false negative that reads as "the bypass doesn't
  work". Copy or reconstruct the CORS headers.
- **Continue document/HTML responses untouched** or you replace the app shell and learn nothing.
- Bodies are base64: `Buffer.from(JSON.stringify(payload)).toString("base64")`.

Then navigate the protected route and read the outcome:

```js
await send("Runtime.evaluate", { expression:
  `JSON.stringify({url: location.href, user: localStorage.getItem('auth-user'), text: document.body.innerText.slice(0,200)})`,
  returnByValue: true })
```

Also record `localStorage` keys and `document.cookie` to confirm the profile stayed anonymous.

## Step 3 — the verdict hinges on the firing proof, never on the URL alone

Landing on `/login?redirect=…` is *exactly* what a plain anonymous load does — it proves nothing about the
override. Log every intercepted `/auth/me` with its **original** `responseStatusCode` (expect 401) and a timestamp,
and assert the count incremented **during that navigation**. Two hits at original 401 proves the override fired on
that navigation (stores commonly call `/auth/me` once in `recoverSessionFromCookie()` and again after refresh — the
attacker must satisfy both; a single interception can leave the session half-built). If the interceptor was scoped
to an earlier document and never re-armed, the redirect is just the normal gate.

Quote both the log and the rendered page. Without that counter, publish neither verdict.

- **`127.0.0.1` vs `localhost`**: session cookies set `Domain=localhost` are never sent to `127.0.0.1`, so `/auth/me`
  401s, the store stays unauthenticated, and a real bypass looks unreproducible. Drive the *app* at
  `http://localhost:<port>` (the CDP endpoint itself stays `127.0.0.1:9335`).
- **Batch tool calls returning another command's output.** Parallel bash batches produced mismatched
  command/result pairs in practice. Run each probe you intend to cite as its own call and check the echoed label
  matches.
- **No firing proof** ⇒ the normal gate reads as a defeated override.

For enumerating the whole route surface (not just the routes this page calls) see
`unprotected-route-surface-audit`; for full Nuxt route-protection audits see `auditing-nuxt-route-protection`.

## Step 4 — then find the real exposure (the override is a distraction)

Probe with NO credentials; the publishable finding is what an anonymous client with no browser at all can fetch:

```bash
# one probe per curl call, echo a label so a mismatched result is caught
printf 'kpis='; curl -s -o /dev/null -w '%{http_code}\n' http://localhost:8085/api/v1/dashboard/kpis
printf 'control='; curl -s -o /dev/null -w '%{http_code}\n' http://localhost:8085/api/v1/zzz-nope
```

- **Always run a catch-all control path.** Echo (Go) `Group.Use` registers a wildcard not-found handler carrying
  that group's middleware (`echo.Any(path.Clean(prefix+""))` and `prefix+"/*"` → `NotFoundHandler`), so every
  unmatched path under the prefix 401s instead of 404ing — an anonymous 401 does not prove a route exists or is
  protected. A 401 without a control proves nothing; a 200 does prove a real route. Static routes win over the group
  wildcard, which is why registered middleware-less routes still 200 while unmatched paths 401.
- **Prove a deleted route with a valid token** (expect 404), never anonymously.
- Grep the route registrations for the actual middleware attachment, not path spelling:
  `grep -n 'Group("' src/router/*.go | head -40`.
- **Do not overread the 401 into a spec's acceptance criteria.** An AC asserting "route deleted → 404 anonymously" is
  wrong under Echo's catch-all — verify the AC itself rather than testing against it.
- **`ssr:false` is not a gate.** It hides the HTML shell but still ships the client bundle; verify by comparing
  anonymous SSR bytes for a protected route against a public one (a ~1.7 KB shell vs a ~21 KB real page).
- **`/_nuxt/app/middleware/x.global.js` 404 proves nothing** — Nuxt bundles global middleware into the app entry, it
  is never served at that path. Do not infer "gate not loaded".

Also test whether the server sees the session cookie at all:

```bash
curl -s -o /dev/null -w 'anon=%{http_code}\n' http://localhost:3000/protected
curl -s -o /dev/null -w 'forged=%{http_code}\n' -H 'Cookie: session=forged-garbage' http://localhost:3000/protected
```

Identical byte counts ⇒ the server ignores the cookie ⇒ no server-side gate exists to build on.

**Before recommending "add auth middleware to the group," check the callers.** A gate that breaks the feature will
not ship:

```bash
grep -rn 'fetch(' src/ | grep -v '\$api\|useFetch\|apiFetch\|__.*FetchPatched'
```

A bare `fetch(url)` sends **no `Authorization` header**. Check for a global interceptor first
(`grep -rn "window.fetch\|__.*FetchPatched\|originalFetch" app/plugins/`) — if a plugin patches `window.fetch` to
inject the Bearer, the "bare" call *is* authenticated and the missing middleware is more clearly an oversight. If
no interceptor exists, the remediation is paired: **gate the server route AND route the callers through the
authenticated client** (or inject the header in a shared helper). Say so, or you hand over a fix that bricks the
feature.

## Step 5b — locate the real server-side fix (verify by diff, not by commit message)

Enumerate the API route groups and find the ones mounted **outside** the authenticated group:

```bash
grep -n ":= e.Group(\|:= api.Group(\|:= .*Group(\|\.Use(" <router>.go
git diff --stat origin/main..<branch> -- <handler dir>   # empty = handler untouched
git show <branch>:<router> | grep -n "<group> := "        # api.Group vs protected.Group
git show <branch>:<handler> | grep -c -i tenant           # 0 on both refs = no tenant scoping
```

Then answer three separate questions — they fail independently:

1. **Is the group gated?** (auth middleware inherited via the parent)
2. **Is there tenant scoping?** (gating alone still allows cross-tenant reads)
3. **Does the client send the credential?** (a gated endpoint 401s every chart if the client doesn't attach the
   bearer)

A fix that answers only (1) is partial; say so explicitly.

Prove **why the override is inert** from source — does the auth-response handler assign the *token*, not just the
user object?

```bash
grep -n "token.value = " app/stores/auth.js
```

If the auth endpoint body populates `user` but the token comes from a cookie callback or `localStorage`, then after
an override: `isAuthenticated === true` and `token === null` → no `Authorization` header → every protected API call
401s → empty shell over zero data. That is the whole answer to "is this a breach".

## Step 6 — recommend, ranked

1. **Keep `ssr:false`, add the server gate.** A document-level `server/middleware/auth-gate.ts` that reads the
   session cookie, forwards it server-side to the auth endpoint, and redirects on non-200. Nitro→API is
   server-to-server, structurally outside the browser's override stack. Do not re-enable SSR just for this: it
   requires server-side data fetching and production cookie-`Domain` correctness, and getting the Domain wrong
   *re-opens* the leak. (A short probe proves Nitro middleware fires on `ssr:false` documents — see
   `nitro-server-gate-nuxt-spa`.)
2. **Close the anonymous data surface.** The gate protects pages, not endpoints. Probe every route the page calls
   with `curl` and no token; a 200 is a real leak and makes the override unnecessary. Framework catch-alls apply
   (see Step 4).
3. **Session-expiry trap in any server gate.** If the access cookie is short (e.g. `edafy_token` Max-Age=900) and
   the refresh cookie is long (24h), a gate that only probes `/auth/me` will 302 a *valid* session to login after
   ~15 min idle. The gate must fall back to `POST /auth/refresh` with the refresh cookie and forward the resulting
   `Set-Cookie` to the browser. Add a "reload after access-cookie expiry" test case.

**Never recommend** override detection, a client-side `authenticatedToken` flag, or a client-side JWT signature
check. All live in the browser and fall to the same 30-second override.

And **never conclude "it's just cosmetic, nothing to do"**: a cosmetic gate frequently sits beside a real
unauthenticated endpoint. Hunt Step 4 before closing the ticket.

## Measuring SSR / hydration regressions with the same harness (no interception)

Flip `routeRules` to `ssr: true`, then classify `Runtime.consoleAPICalled` messages by ROOT CAUSE (`Hydration
.*mismatch`, `Failed to resolve component`, `accessed during render but is not defined`, `missing template or
render function`, `No match found for location`) and **dedupe by message, not by occurrence**.

Traps that produced wrong numbers in practice:

- **Clear the buffer BEFORE `Page.navigate`, not after.** Clearing mid-navigation leaks the previous route's
  messages into the next route's bucket. Symptom: an impossible per-route split contradicting the raw dump (route
  A=5, B=0 on one run; 5/5 on the next).
- **All-empty results are a broken capture, not zero defects.** If a config-independent, auth-independent warning
  (`Failed to resolve component: Info`) that you measured before comes back empty, the CDP listener did not attach
  — discard the run, never publish it as "costs nothing".
- **A status-code sweep hides broken-but-200 pages.** 2/75 non-200 can coexist with 15 routes carrying `[Vue warn]`
  in the payload and 5 with real hydration errors. Count body content and console output too.
- **Anonymous loads 401 on every page**, so any filter counting "any console error" inflates with client-auth
  noise. Filter on hydration/render signatures only.
- **Redirect stubs double-count.** An `index.vue` doing `await navigateTo('/other')` reports the target page's
  warnings under its own route. Dedupe by page.
- **Compare against the `ssr:false` baseline.** A warning present in both configs is pre-existing, not SSR cost —
  this separates genuine defects from noise.

## Cleanup and verification before concluding

- Restore the config file and confirm: `git diff <file>` empty, and the live probe shows the original behaviour.
- Delete any temporary probe route/middleware; confirm with `git status --short` and a path search for the probe
  name.
- Quote probe results only from runs whose echoed label matches the command; re-run sequentially when a batch
  result looks like another call's output (parallel bash batches produced mismatched command/result pairs in
  practice).

## Tricipta workspace specifics (edafy asset-web / asset-api)

Use these when triaging inside the Tricipta repos — they are the concrete instances of the procedure above.

**The achievable property, stated for this workspace:**

> **No real data is readable without a server-verified token.** A protected page that renders an empty shell over
> 401s is **already correct**. Redirect effort from "block the override" (impossible) to "make the override inert".

Three real bypass paths: (1) DevTools response override of `/auth/me` (status→200 + fake user JSON); (2) DevTools
console mutating the Pinia store directly; (3) direct API access, bypassing the UI entirely.

**NOT a path:** `localStorage.setItem('auth-user', …)`. In tricipta-edafy-asset-web `getItem("auth-user")` is never
read; only `getItem("auth-token")` is (api.client.js:94, auth.client.js:54, stores/auth.js:148,183,335), and a
forged token fails signature verification. Verify this before adding it to a vector list.

**Verification ladder (run in order, evidence not inference):**

```bash
# 1. Client gate is purely local state — read it, don't test it
#    app/middleware/00-protected.global.js:48-51  -> authStore.isAuthenticated
#    app/stores/auth.js:24                         -> computed(() => !!user.value)

# 2. SSR gate (D4): anonymous curl must be an EMPTY SHELL, not leaked content
curl -s -o /tmp/a.html -w '%{http_code} %{size_download}\n' localhost:3000/common-platform/dashboard
grep -qi "Angsi\|PETRONAS\|Good afternoon\|Dashboard" /tmp/a.html && echo LEAK || echo ok
curl -s -o /tmp/l.html localhost:3000/login   # public -> should be LARGE (real SSR)

# 3. Server authorization — the only enforcement
curl -s -o /dev/null -w 'anon:   %{http_code}\n' localhost:8085/api/v1/dashboard/kpis   # want 401
TOKEN=$(curl -s -X POST localhost:8085/api/v1/auth/login -H 'Content-Type: application/json' \
  -d '{"email":"testuser@example.com","password":"testpass"}' | jq -r .token)
curl -s -o /dev/null -w 'authed: %{http_code}\n' -H "Authorization: Bearer $TOKEN" \
  localhost:8085/api/v1/dashboard/kpis                                                    # want 200
```

**Traps that produce wrong conclusions here:**

| Trap | Effect | Guard |
|---|---|---|
| Echo `Group.Use()` installs `Any(prefix+"/*")` catch-all carrying group middleware | Unknown paths under a protected prefix return **401**, so an anonymous 401 on a deleted route looks like a gate | Verify deletion with a **valid token** (404) |
| Stale `.output/` from before the SSR fix | Falsely reproduces the leak | Compare artifact mtime to branch HEAD; prefer `npm run dev` / fresh build |
| Branch rebased mid-investigation | SHAs and line numbers drift | Re-ground before citing (observed `6de3a90`→`5f5fef1`) |
| Empty local dev DB (`wells=0`, `ast_production_data=0`) | Payloads are zeroed → mistaken for "no leak" | The leak **is** the anonymous `200`; zeros are a data artifact |
| `useAiInsightLandingNames.ts` silent mock-name fallback | Unauth shell shows real-looking labels (`Angsi`, `PETRONAS Carigali`) → tester reports a breach | Landing cards are bundled literals from `app/utils/aiInsightLandingData.ts`, in any build — not data |
| `docker exec edafy-db psql -U postgres` | `role "postgres" does not exist` | Use `-U user -d tricipta-edafy-asset-api-db` |

**Known deliverables to map findings onto:** asset-api#163 — the real exposure (`router.go:730-751` `dashboardGroup`
outside `protected`, 21 routes, no tenant predicate in `internal/domain/dashboard/handler.go:37-49`; fix is
three-part: gate the group **+** add tenant scoping **+** make the client send the bearer — gating alone 401s every
chart). asset-web#927 — `server/api/diagnostic-plots/*`, no session check, injects server-side `X-API-Key`.
asset-api webhook fail-open — `webhook_secret.go:28` `if secret != ""` skips the check entirely when unset.
`/internal/dev/simulate-blob-event` — mounted when `APP_ENV != "production"`, and `cd.yml:77` hardcodes
`development`, so live in every tier. `GET /swagger/*` — anonymous (`router.go:168`). asset-web#923 — **performance
only**; does not fix the client gate (explicitly keeps D8) and could *reopen* the SSR leak if the shared
parent-cookie `Domain` is wrong; never route a security answer through it.

## Reporting shape

1. Two separable questions: the override is unpreventable; is the *server* exposed? (Reframe.)
2. What the spoof rendered and why it proves nothing (bundled vs fetched, with the bundle grep as evidence);
   anonymous-vs-override table with the firing proof (count, URL, original status) attached.
3. The trusted client-state chain, with exact file:line.
4. The real anonymous surface table (endpoint → anonymous status → what it returns) plus the catch-all control
   caveat, and what is NOT fixed.
5. Ranked remediation with the specific missing server-side check named, plus the paired client change.
