---
name: unprotected-route-surface-audit
description: "When asked 'is anything else unauthenticated' or reviewing an auth-fix spec: enumerate routes."
---

# Auditing the unprotected-route surface / verifying security claims

Use when reviewing (or writing) a security spec/plan that closes "an authentication bypass" and claims to have
enumerated the affected surface; for the direct question *"is anything else unauthenticated here?"*; and before
publishing any auth-gate review verdict that rests on "this page is ungated", "reuse the existing endpoint", or
"this guard covers it".

Grounded in a real case: a twice-reviewed 213-line spec correctly identified three bypasses (SSR page rendering, an
ungated `/register` route, a forgeable JWT fallback) and correctly re-derived its page counts — while a
**21-endpoint anonymous data API** sat in the same router file, absent from every document.

## The blind spot this catches

A spec audits **what it can name**: the reported pages, the routes in the incident report, the file in the stack
trace. It does not ask *"what else is unauthenticated?"* — a question with no owner, because naming it requires
enumerating the whole surface, not the interesting part of it.

Pages and routes are also **different inventories**. A spec that exhaustively enumerates 76 page files can still
miss every backend route; a spec that audits named routes can still miss unnamed ones.

Sibling skills: `auditing-nuxt-route-protection` (Nuxt mechanism/doc audit), `nitro-server-gate-nuxt-spa` (the
server gate), `verify-auth-gate-against-devtools-override` (the override verdict).

## Procedure

### 1. Enumerate route registrations, not routes

Find every place a group/route is created and every place middleware is attached:

```bash
grep -n ":= e.Group(\|:= api.Group(\|:= .*Group(\|\.Use(" <router>.go
```

Then list **every** `RegisterRoutes(...)` call with the group identifier it receives. The group identifier is the
finding — one call receiving a different group than its 40 siblings *is* the bug.

Build a table: `line | handler | group | verdict (public-by-design / separate-mechanism / exception)`.

### 2. Probe the contract, not the reading

Send the **same credential-less request** to a route you know is protected and to the suspect route. The
contrasting status codes are the finding:

```bash
for p in /api/v1/known-protected /api/v1/suspect; do
  printf "%-30s -> " "$p"
  curl -s -o /tmp/o -w "HTTP %{http_code}" "$1$p"; echo " | $(head -c 90 /tmp/o)"
done
```

A `200` with a plausible body beats any amount of `grep` reasoning. **Watch for the empty-database trap:** a
`200 {"data":{...zeros...}}` on a dev box means the query *ran and returned nothing*, not that a gate allowed it.
Check the handler for the absence of a tenant predicate to confirm the query is unscoped.

### 3. Find the same-prefix sibling (strongest evidence)

Search for the same URL prefix registered twice by different handlers:

```bash
grep -rn 'Group("/<prefix>' internal/ --include=*.go | grep -v _test
```

A route whose prefix is shared with a protected sibling — e.g. `/dashboard/config` → 401 but `/dashboard/kpis` → 200
— proves **oversight, not intent**, and bounds the finding precisely: "exactly these N routes", not "the whole
prefix".

### 4. Check the non-obvious auth mechanisms

- Middleware with a **`Skipper`** callback — an unconditional bypass that compiles and passes every test.
- Webhook-secret middleware written `if secret != "" { check }` — **fail-open when unset**, often with a
  `log.Println` warning nobody reads.
- Dev/debug routes mounted on `Environment != "production"` — check what the CI/CD pipeline *actually* sets; a
  hardcoded `APP_ENV=development` makes "non-production only" a production mount.
- Health/readiness endpoints returning raw `err.Error()` — cheap anonymous infrastructure disclosure.

### 5. Never trust "the frontend doesn't send auth, so it's public"

**Trap.** A bare `fetch(url)` with no `Authorization` header looks like proof an endpoint is intentionally public.
Check for a global interceptor first:

```bash
grep -rn "window.fetch\|__.*FetchPatched\|originalFetch" app/plugins/
```

If a plugin patches `window.fetch` to inject a Bearer token, the "bare" call **is** authenticated — the evidence
inverts, and the missing middleware becomes *more* clearly an oversight, not less.

### 6. Re-check your own claims before reporting

Every headline claim is a candidate for the same scrutiny. In the source case, of three candidate findings: one was
**withdrawn** (a claimed CSRF hole — the cookies set `SameSite=Lax` and no mutating route was registered as `GET`),
one **downgraded** (a password-grant bypass — the IdP's own brute-force protection still applied), and one
**refuted** by the `window.fetch` check above.

State explicitly which claims are live-proven, source-verified, or UNVERIFIED. Do not let a plausible mechanism
imply an unproven impact.

## Nuxt-specific claims

When the subject is a Nuxt app, the "ungated page", "reuse the existing endpoint", and "global middleware covers it"
claims have their own trap list — see `auditing-nuxt-route-protection`. Verify each such claim there before it
enters a report; this skill owns only the route/endpoint inventory.

## Output

A table: route/mechanism · evidence (file:line or probe output) · severity · confidence. Separate **proven** from
**mechanism-only**. Name the class-defeating gap (the missing inventory step) alongside the individual findings —
fixing N instances without adding the classification test leaves instance N+1 to the next reviewer. Lead with what
you verified live (curl/DOM evidence with status + byte counts), findings **ranked**, each with `file:line` and the
command that proved it. Retractions belong in the same report, stated plainly, naming which earlier claim was wrong
and what refuted it. Never let an inference stand as a finding — the cost of one wrong Critical is the credibility
of the rest.
