---
name: nuxt-readiness-await-middleware-port
description: "Port a session-readiness-await route-guard pattern (dynect-hrms-web style) into another Nuxt app whose auth store owns the session, and prove each ported branch is load-bearing. Use when a Nuxt app's route middleware reads isAuthenticated/loggedIn before the session restore settles, or when asked to make one app's middleware 'the same as' another's."
---

# Porting a readiness-await route guard between Nuxt apps

Two Nuxt apps can share a guard *shape* while differing in where the session lives. Port the shape, not the mechanism — and prove each ported branch actually changes behavior.

## 1. Identify the session owner in the target app

The source pattern reads a readiness flag and awaits a memoized fetch:

```ts
// source (nuxt-auth-utils app, e.g. dynect-hrms-web)
const { loggedIn, ready, fetch } = useUserSession()
if (!ready.value) await fetch()
if (!loggedIn.value) return navigateTo(`/login?redirect=${to.fullPath}`)
```

Find the target's equivalent of each of the three:

| Need | Source | Target equivalent to find |
|---|---|---|
| readiness flag | `ready` | is there a `ready`/`initialized`/`hydrated` ref on the auth store? (often NONE — add one) |
| restore | `fetch()` | the store's `initializeAuth()`-style method |
| auth predicate | `loggedIn` | `isAuthenticated`, `!!user.value`, etc. |

Grep the store for the predicate and the restore before editing:

```bash
grep -n 'isAuthenticated\|initializeAuth\|ready\|initialized' app/stores/auth.js
```

**Do not copy `nuxt-auth-utils` idioms into an app that lacks the module.** Check `package.json` for `nuxt-auth-utils` first; if absent, there is no `useUserSession`, no `requireUserSession`, and no server-readable session.

## 2. Add readiness to the store (if missing)

The whole bug is that the auth predicate is false for BOTH "anonymous" and "not yet restored". Only a readiness flag distinguishes them.

```js
// app/stores/auth.js
const initialized = ref(false)
let initPromise = null

// Extract the existing restore body into an unguarded inner fn, then wrap it:
const runInitializeAuth = async () => { /* original body, unchanged */ }

const initializeAuth = async () => {
  if (initialized.value) return true
  initPromise ??= runInitializeAuth().finally(() => {
    initialized.value = true
    initPromise = null
  })
  return initPromise
}
```

Expose `initialized`, the wrapped `initializeAuth`, and the guard-escape `resetInitialized()`.

### Two invariants that are easy to get wrong

1. **`initialized` must SURVIVE logout.** Deliberate logout resolves the session *to anonymous* — it is resolved, not unresolved. If `clearSession()` resets it to `false`, a guest gate on `/login` takes its "not ready" branch, falls through the already-signed-in check, and the dashboard's auth gate bounces the user straight back → `login → dashboard → login` loop. Reset only via `resetInitialized()`.
2. **Find plugin branches that clear the session without restoring it.** An `if (session_expired) { clearUser(); }` branch leaves `initialized === false`, so middleware then kicks off a restore and quietly re-establishes the session the branch just dropped. That branch must set `initialized = true` explicitly.

## 3. Port each guard

Apply the same three lines to every client-side gate (named + `.global`). Keep any server bail the target already relies on:

```js
if (process.server) return          // keep if the target's restore is client-only
const authStore = useAuthStore()
if (!authStore.initialized) await authStore.initializeAuth()   // the port
// then the original predicate + redirect
```

**Server bail is app-specific — justify it, do not copy the source's silence.** An app whose session is server-readable (nuxt-auth-utils) omits the bail. An app whose restore is client-only AND whose protected routes are `ssr:false` MUST keep it: a server check reads `user === null` and redirects authenticated users away. State which case you are in, in a comment.

## 4. Test at module level, then prove non-vacuity

Middleware doing `import { useAuthStore } from '~/stores/auth'` cannot be driven with `vi.stubGlobal('useAuthStore', …)` — direct ES imports bypass globals. Use `vi.mock` (hoisted, so static imports still work):

```ts
vi.mock('~/stores/auth', () => ({ useAuthStore: () => store }))

// Mutable store fixture — an unresolved store reports the predicate FALSE,
// and the awaited restore flips it. A static `isAuthenticated: true` fixture
// makes the readiness test vacuous.
const store = {
  isAuthenticated: opts.isAuthenticated,
  initialized: opts.initialized,
  initializeAuth: vi.fn(async () => {
    store.initialized = true
    store.isAuthenticated = opts.authenticatedAfterInit ?? opts.isAuthenticated
  }),
}
```

Static `import` of the middleware is then fine — do not reach for `await import()`, project lint rules usually forbid it and `vi.mock` is already hoisted.

**Then prove each branch is load-bearing** — delete the ported branch, expect failures, restore:

```bash
cp app/middleware/gate.js /tmp/g.bak
python3 - <<'PY'
p='app/middleware/gate.js'; s=open(p).read()
b="""  if (!authStore.initialized) {\n    await authStore.initializeAuth();\n  }\n\n"""
assert b in s, 'branch not found'
open(p,'w').write(s.replace(b,''))
PY
npx vitest run test/middleware/    # MUST fail
cp /tmp/g.bak app/middleware/gate.js
npx vitest run test/middleware/    # MUST pass
```

If removal does not fail anything, the test is vacuous — fix the fixture, not the test name.

## 5. Verify live, both directions

Unit tests cannot see the real redirect chain. With the stack up (`:3000` web, `:8085` api, `:8180` keycloak):

- anonymous cold load of a protected route → `/login?redirect=<path>`
- authenticated cold load of the same route → renders real content, no bounce
- authenticated cold load of `/login` → lands on the dashboard (the guest inverse)
- `/login?logged_out=1` → STAYS on login, no bounce back

## 6. Do not delete a global default-deny net when "matching" an opt-in source

If the source app's guards are opt-in (a *named* `middleware/auth.ts`, not `.global`), its safety rests on layers the target may lack — server-readable session, in-template permission guards. In a target where auth state is client-only and `ssr:false` renders nothing server-side, an un-opted-in page is fully reachable. Port the guard *body*, keep the global net.
