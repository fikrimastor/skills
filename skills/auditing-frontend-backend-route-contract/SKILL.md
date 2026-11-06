---
name: auditing-frontend-backend-route-contract
description: "Mechanically diff a frontend's outbound HTTP URLs against a backend's registered routes to find 404-class contract mismatches and genuinely-unused routes — includes the extraction traps that produce confidently wrong verdicts. Use when asked 'are these endpoints used in the web repo', 'is this route dead', or before an API-surface change."
---

# Auditing a Frontend↔Backend Route Contract

Answers "does the frontend actually call these routes?" with a mechanical diff. The
value is in the traps: a naive grep produces a verdict that looks authoritative and
is wrong in both directions — inventing dead routes that are live, and hiding live
mismatches behind phantom routes.

Complements `auditing-endpoint-removability` (which covers route-vs-service and
live-probing). Use this one for the static two-sided diff.

## Output shape

Two tables, then calibration:

1. **Forward** — frontend URLs with no matching registered route (the 404s). This is
   usually the real ask.
2. **Reverse** — registered routes with no frontend caller. Label explicitly as
   *route-level observation only*: zero callers ≠ deletable, and the service behind
   it may be load-bearing.

Every row carries `file:line` on both sides. Report forward mismatches first; they
are live bugs, reverse entries are cleanup candidates.

## The four extraction traps

Each of these silently corrupts the result. Handle all four.

**1. Route truth lives in two places.** Not just the central router. Also each
handler's `RegisterRoutes(g *echo.Group)` method, reached via a call in the router
(`x.RegisterRoutes(protected)`). Read both, or you miss whole route families.

**2. Dual registration via a shared closure.** Some domains register both spellings:

```go
register := func(g *echo.Group) { g.GET("/countries", h.GetCountries) /* ... */ }
register(g.Group("/area"))
register(g.Group("/areas"))
```

A router-only read reports `/areas/*` as a mismatch when it is registered. **This is
the single most common false positive.** Grep every handler for `Group("/` before
trusting a plural/singular mismatch.

**3. Never-invoked registration methods.** A `RegisterRoutes` that the router never
calls (commented out, superseded, or a package never imported) registers **nothing**.
Counting its routes invents routes that do not exist — and those phantoms then
*mask* real mismatches, because the frontend's URL appears satisfied. Extraction must
be invocation-driven: parse the router for which methods it actually calls.

Attribute each call to a handler by **receiver type** (`x := pkg.NewFooHandler()` →
`FooHandler`), not by Go package: large repos put dozens of handlers in one
`package handler`, so the package name is not an identity.

**4. Frontend URL construction has two forms.** Literal templates
(`` `/v1/foo/bar` ``) and path-builder helpers:

```js
const api = (path) => `${apiBase}/v1${path}`
api("/auth/refresh")                     // invisible to a literal-only grep
const api2 = (path) => `${base}/v1/${path}`
api2('license/status')
```

Helper-based callers that go unresolved inflate the reverse list with **live routes** —
which is exactly how someone deletes a working endpoint. Detect builders by matching a
local name bound to a template starting with the api base, then expand its string-literal
arguments. Only treat it as a builder when the binding matches, or you will pick up an
unrelated `api(` client and create phantom paths.

Also: inside a template, `${x}` **preceded by `/`** is a path parameter (keep walking);
any other interpolation, or a `?`, ends the static path. Treating them alike mangles
paths (or loses trailing segments like `/notifications/:id/read`).

## Normalize before comparing

Strip the version prefix on both sides, and map every `:param`/`:{interp}` segment to
one placeholder on both sides so `/foo/:id` matches `/foo/${x}`.

## Calibrate, or the output is worthless

A diff has no ground truth — so pin invariants and fail loudly:

- **Known-answer pair**: one confirmed mismatch must appear forward, its real route
  reverse. If either is missing, extraction is broken.
- **Known false positive**: a dual-registered family must NOT appear forward.
- **Counts**: record backend-route and frontend-template counts; a *drop* in frontend
  templates after a change means helper resolution regressed.

Ship these as a `--self-check` mode returning a non-zero exit code. Then verify it can
fail: temporarily disable one extractor feature and confirm the check trips. A
self-check that cannot fail is decoration.

## Reporting

- Separate **documentation hits** (`swagger/`, generated wikis, `graft/`,
  `graphify-out/`) from real consumers — docs are not callers.
- Separate **other services** from the contract under audit: Nuxt server routes,
  OTLP/telemetry endpoints, and localhost-bound exporters will look like mismatches
  and are not.
- Watch for commented-out calls: they are often deliberate TODOs, not live traffic.
- Distinguish a URL-shape near-miss from a real route: `/v1/production` is not
  satisfied by `/v1/production/data`.
- Record the false positives you ruled out, with the reason. Otherwise the next
  reader re-derives them and may act on them.
- A route with zero callers plus a frontend call to a *different* path often means the
  frontend was written against an API that never shipped — check whether both sides
  landed in the same initial commit.

## Pitfalls

- Trusting a subagent's sweep without re-verifying each row; a single missed
  registration pattern propagates an error rate into the whole table.
- Reading only the central router.
- Counting a handler's routes without checking the router invokes it.
- Assuming the obvious frontend is the only consumer — check scripts, cron, CI,
  Airflow DAGs, and runbooks.
- Recommending "repoint to X" without proving X is registered (it may live in the same
  never-invoked handler that caused the bug).
- Deleting a route that has no caller but whose service is the only implementation of
  a capability.
- **Secret safety:** never run an unscoped repo-root grep; scope to source dirs or
  exclude `.env`, and mask suspected secrets in output.
