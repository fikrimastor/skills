---
name: wayfinder-protocol-relative-url-port-fix
description: "When Wayfinder-generated routes use protocol-relative URLs (//host/path) and the browser resolves them to a different port (80 instead of dev port), apply a path-only transformation in form post handlers."
---

# Wayfinder Protocol-Relative URL Port Drop Fix

## Problem
Inertia + Wayfinder + Vite dev setup where:
- App runs on `http://localhost:8081` (or any non-80 port)
- Wayfinder generates URLs like `//localhost/platforms/auth/store` (protocol-relative, no port)
- Inertia's `form.post(url)` does `new URL(url, window.location.origin)` internally
- `new URL('//localhost/path', 'http://localhost:8081')` returns `http://localhost/path` (port DROPPED because protocol-relative URL specifies the host)
- Form posts to port 80, fails with `ERR_CONNECTION_REFUSED`

## Fix
In Vue components that call `form.post(store.url())`, transform the URL to path-only before posting:

```ts
// Wayfinder emits absolute or protocol-relative URLs from the route domain.
// The generated `//localhost/...` form makes the browser drop the dev port
// (8081) and POST to port 80. Resolve against the current origin so the
// request stays on the same host:port as the page.
const toRelativePath = (url: string): string => {
    // Already a path-only URL? Keep as-is.
    if (url.startsWith('/') && !url.startsWith('//')) {
        return url;
    }
    // Strip host, keep just the path + query string.
    try {
        return new URL(url, window.location.origin).pathname
            + new URL(url, window.location.origin).search;
    } catch {
        return url;
    }
};

const submit = () => {
    form.post(toRelativePath(store.url()), {
        onFinish: () => form.reset('password'),
    });
};
```

**Key gotcha:** the early-return guard MUST be `url.startsWith('/') && !url.startsWith('//')`. If you only check `url.startsWith('/')`, the protocol-relative `//host/path` returns true (starts with `/`) and the URL is returned unchanged, breaking the fix.

## Why other approaches fail
- **Adding port to `tenancy.central_domains`**: breaks route matching (Laravel's `Request::getHost()` doesn't include port).
- **`URL::forceRootUrl('http://localhost:8081')`**: Wayfinder's `domain()` short-circuits on the route's registered domain before reading `forcedRoot`.
- **`public/build/hot` file with port**: only affects `@vite()` script tags, not Wayfinder route generation.
- **Inertia's URL behavior is intentional** — `new URL('//host/path', base)` follows RFC 3986 where the protocol-relative URL's host overrides the base's host (port lost if not in protocol-relative).

## Verification
After fix:
- Browser console / network tab: POST goes to `http://localhost:8081/path` (correct port)
- Server log: `data` field populated with form values
- Successful login → redirect to dashboard

## Related stack
- Laravel 12 with `stancl/tenancy` (central_domains used for route registration)
- `wayfinder` for Vue route TypeScript generation
- Inertia.js v2 with Vue 3
- Vite 7 dev server (port 5173 in container, 15173 on host via Docker)
- Sail (laravel.test on host port 8081 → container 80 via `artisan serve`)
