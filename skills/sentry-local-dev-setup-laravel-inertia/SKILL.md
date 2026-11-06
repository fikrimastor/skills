---
name: sentry-local-dev-setup-laravel-inertia
description: "Diagnose and fix Sentry v10 SDK not working in local Laravel + Inertia + Vue dev: missing DSN, Vite env cache, 429 rate limit, and user tracking via router.on('success'). Use when a dev reports \"Sentry not capturing browser errors\" or \"429 Too Many Requests\" from ingest.sentry.io, or asks how to track logged-in user in Sentry events."
---

# Sentry Local Dev Setup & Debugging — Laravel + Inertia + Vue + Sail/Docker

## When to use this skill

Use when ANY of these are true in a Laravel + Inertia + Vue + Sail (Docker) project:
- Developer reports "Sentry not capturing errors" or "no events in dashboard"
- Browser DevTools shows HTTP 429 from `*.ingest.sentry.io` or `*.ingest.de.sentry.io`
- Developer asks "how to track user (id/name/email) in Sentry events"
- Sentry init code is present in `resources/js/app.js` but `Sentry.isInitialized()` returns false or no events appear
- After adding `SENTRY_LARAVEL_DSN` / `VITE_SENTRY_DSN` to `.env`, nothing changes in the served bundle

## The three failure modes (ordered by frequency)

### 1. Vite caches `.env` at server startup — adding new env vars requires container restart

**Symptom:** `import.meta.env.VITE_SENTRY_DSN` is `undefined` at runtime even though `.env` has the value. Vite serves `app.js` with `import.meta.env = {...}` block but `VITE_SENTRY_DSN` key is missing.

**Diagnostic:**
```bash
curl -sS "http://localhost:5173/resources/js/app.js" | head -1
# Should print: import.meta.env = {"BASE_URL": "/", ..., "VITE_SENTRY_DSN": "https://..."}
# If VITE_SENTRY_DSN is missing → Vite hasn't picked up the .env change
```

**Fix:**
```bash
docker restart mrchurros-dash-vite-1
# Or: sail restart vite
# Or: kill the node process and run `npm run dev` again
```

**Root cause:** Vite reads `.env` once at dev server startup and caches in memory. File watcher (HMR) does NOT reload env changes. This is a Vite design choice, not a bug.

### 2. Sentry v10 browser SDK doesn't auto-extract user from Inertia's auth share

**Symptom:** Errors are captured (check `Sentry.isInitialized()` returns true and `sentryWrapped` appears in error stacks) but the User section in Sentry dashboard is empty.

**The fix — three pieces:**

**Piece 1: Import the singleton router from `@inertiajs/core` (NOT from `@inertiajs/vue3`):**
```js
import { router } from "@inertiajs/core";
```

**Piece 2: Listen to the 'success' event and read the new Page's auth.user:**
```js
const syncSentryUser = (page) => {
    if (!import.meta.env.VITE_SENTRY_DSN) return;
    const user = page?.props?.auth?.user;
    if (user) {
        Sentry.setUser({
            id: String(user.id),
            username: user.name,
            email: user.email,
        });
    } else {
        Sentry.setUser(null);
    }
};
router.on('success', (event) => syncSentryUser(event.detail.page));
```

**Piece 3: Sync on initial page load too (router.on('success') only fires on navigations):**
```js
createInertiaApp({
    setup({ el, App, props, plugin }) {
        // ... createApp chain ...
        if (import.meta.env.VITE_SENTRY_DSN) {
            Sentry.init({ /* ... */ });
            syncSentryUser(props.initialPage);  // <- initial page payload
        }
        return app.mount(el);
    },
});
```

**Why not just use `usePage()` composable inside a watch?** It works, but requires Vue setup context, which the `setup()` callback already provides. The router.on approach is simpler and reactive to all navigations without needing a watch.

**Email in Sentry requires `sendDefaultPii: true`:**
```env
SENTRY_SEND_DEFAULT_PII=true
VITE_SENTRY_SEND_DEFAULT_PII=true
```

### 3. HTTP 429 from Sentry = sampling too high in dev

**Symptom:** DevTools Network tab shows `POST .../envelope/...` returning `429 (Too Many Requests)`. Errors may or may not reach dashboard.

**Root cause:** `tracesSampleRate: 1.0` + `replaysOnErrorSampleRate: 1.0` in dev. Sentry's free plan has 5,000 events/month cap plus burst limits. One Inertia navigation = 5-15 events. 10 navigations = 50-150 events. Multiple devs sharing one project hits the cap fast.

**Fix for local dev `.env`:**
```env
# Server side (ok to keep 1.0 for Laravel debugging)
SENTRY_TRACES_SAMPLE_RATE=1.0
SENTRY_PROFILES_SAMPLE_RATE=0.0
SENTRY_REPLAY_SAMPLE_RATE=0.0
SENTRY_REPLAY_ON_ERROR_SAMPLE_RATE=0.0

# Browser side (cap aggressively in dev)
VITE_SENTRY_TRACES_SAMPLE_RATE=0.0       # 0 for dev, 0.1 to peek, never 1.0
VITE_SENTRY_REPLAY_SAMPLE_RATE=0.0
VITE_SENTRY_REPLAY_ON_ERROR_SAMPLE_RATE=0.0

# Per-developer environment namespace to avoid sharing quota
SENTRY_ENVIRONMENT=local-kanda
VITE_SENTRY_ENVIRONMENT=local-kanda
```

After changing `.env`, restart Vite container: `docker restart mrchurros-dash-vite-1`.

## The canonical `app.js` template (Laravel + Inertia v2 + Sentry v10)

```js
import "./bootstrap";
import "../css/app.css";

import * as Sentry from "@sentry/vue";
import { createApp, h } from "vue";
import { createInertiaApp } from "@inertiajs/vue3";
import { router } from "@inertiajs/core";  // <-- singleton for event listening
import { createPinia } from "pinia";
// ... other imports ...

const syncSentryUser = (page) => {
    if (!import.meta.env.VITE_SENTRY_DSN) return;
    const user = page?.props?.auth?.user;
    if (user) {
        Sentry.setUser({
            id: String(user.id),
            username: user.name,
            email: user.email,
        });
    } else {
        Sentry.setUser(null);
    }
};
router.on('success', (event) => syncSentryUser(event.detail.page));

createInertiaApp({
    resolve: (name) => resolvePageComponent(`./Pages/${name}.vue`, import.meta.glob("./Pages/**/*.vue")),
    setup({ el, App, props, plugin }) {
        const app = createApp({ render: () => h('div', [h(App, props)]) })
            .use(plugin)
            // ... other plugins ...
            ;

        const sentryDsn = import.meta.env.VITE_SENTRY_DSN;
        if (sentryDsn) {
            Sentry.init({
                dsn: sentryDsn,
                sendDefaultPii: import.meta.env.VITE_SENTRY_SEND_DEFAULT_PII,
                environment: import.meta.env.VITE_SENTRY_ENVIRONMENT || undefined,
                release: import.meta.env.VITE_SENTRY_RELEASE || undefined,
                tracesSampleRate: Number(import.meta.env.VITE_SENTRY_TRACES_SAMPLE_RATE ?? 0),
                replaysSessionSampleRate: Number(import.meta.env.VITE_SENTRY_REPLAY_SAMPLE_RATE ?? 0),
                replaysOnErrorSampleRate: Number(import.meta.env.VITE_SENTRY_REPLAY_ON_ERROR_SAMPLE_RATE ?? 0),
                integrations: [
                    Sentry.browserTracingIntegration(),
                    Sentry.vueIntegration({ app, tracingOptions: { trackComponents: true } }),
                ],
            });
            syncSentryUser(props.initialPage);  // initial load
        }

        return app.mount(el);
    },
});
```

## Diagnostic flow (use in this order)

1. **Is Sentry init running?** Inject a console.log inside the `if (sentryDsn)` block:
   ```js
   console.log('[Sentry] init running, DSN =', sentryDsn);
   Sentry.init({ /* ... */ });
   console.log('[Sentry] isInitialized =', Sentry.isInitialized?.());
   ```
   Reload page, check browser console. If you see "init running" but NOT "isInitialized = true", DSN is invalid or the integration list is malformed.

2. **Is Vite serving the right env?** `curl -sS http://localhost:5173/resources/js/app.js | head -1` — look for `VITE_SENTRY_DSN` in the `import.meta.env = {...}` block.

3. **Is the event reaching Sentry?** Open DevTools Network, filter `sentry.io`, trigger an error: `setTimeout(() => { throw new Error('x'); }, 50)`. Look for POST status 200 (success) or 429 (rate limited).

4. **Is user being attached?** In browser console: `Sentry.getCurrentScope().getUser()` — should return `{id, username, email}` after navigating to a logged-in page. Returns `null` if not set.

5. **Server-side check:** `php artisan tinker --execute='var_dump(\Sentry\captureMessage("test"));'` — returns an EventId object if working, throws if DSN is wrong.

## What does NOT work (avoid these mistakes)

- `Sentry.setIntegrationsForVueApp(app)` — does not exist in @sentry/vue v9+. Use `Sentry.vueIntegration({ app })` inside the integrations array at init time. The `app` must be passed at init, so `Sentry.init` must be called INSIDE `setup({...})` after the app is created.
- Calling `Sentry.init` at module top-level (outside `setup()`) — `vueIntegration({ app })` requires `app` to be in scope. Top-level call means `app` doesn't exist yet → init fails silently or vueIntegration is bound to no app.
- Setting `SENTRY_RELEASE` to a long git SHA on every commit — Sentry groups events by release; do this in CI/deploy, not locally.
- Relying on `Sentry.flush()` before page navigation without `keepalive: true` — events may be cancelled. Sentry v10 sets `keepalive` automatically based on payload size, but the browser may still drop very large envelopes.
- Using `window.Sentry` after import — ES module imports don't bind to `window`. Use `Sentry.getCurrentScope()` or expose Sentry via `window.Sentry = Sentry` if you need global access (don't do this in production code).

## When the user reports "I see 429" specifically

Don't increase sampling. Don't disable the SDK. The right response is:
1. Set `VITE_SENTRY_TRACES_SAMPLE_RATE=0.0` and `VITE_SENTRY_REPLAY_ON_ERROR_SAMPLE_RATE=0.0` in `.env`
2. Set per-developer `SENTRY_ENVIRONMENT=local-<initials>` in `.env`
3. Restart Vite container
4. Confirm 429s stop, errors still flow (check Sentry Issues dashboard filtered by environment)

If 429s continue after sampling=0 and unique environment, the DSN may be invalid (rare) or shared with a production deployment hitting the same rate limit bucket. Ask user to verify DSN in Sentry dashboard: Settings → Projects → Client Keys (DSN) → verify the public DSN matches what's in their `.env`.

## Files to check first when debugging

- `resources/js/app.js` — Sentry init code location and pattern
- `.env` and `.env.example` — DSN and sampling config
- `config/sentry.php` — server-side defaults (rarely the cause, but check if `tracing.default_integrations` is disabled)
- `vite.config.js` — `sentryVitePlugin` for source map upload (separate from event capture)
- `docker-compose.yml` / `compose.yaml` — Vite container config; should mount `.` to `/var/www/html` so Vite sees `.env` changes
- `app/Http/Middleware/HandleInertiaRequests.php` — should share `auth.user` with Inertia (standard Breeze setup; if not present, Sentry can't see the user)
