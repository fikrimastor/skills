---
name: auditing-endpoint-removability
description: "Decide whether an HTTP endpoint can be safely deleted — exhaustively map consumers, separate a dead route from a load-bearing service, and discriminate real route existence behind blanket middleware. Use when asked 'can we remove these legacy endpoints', 'is this route dead code', or before any API-surface deletion."
---

# Auditing Endpoint Removability

Answers "is it safe to delete this endpoint?" with evidence. The failure mode this
prevents: deleting a "legacy" route that is actually the live integration, or
deleting a dead *route* whose *service* is the only implementation of a core
capability.

## Rule 0 — reject the framing

Users usually arrive with a premise: *"these are legacy because we moved to X"*.
Test it, never inherit it. In one real audit, "/auth is legacy now that we use
Keycloak" was backwards — **6 of 8 routes in that namespace *were* the Keycloak
integration**. The legacy code lived inside two handlers, not in the routes.

State the premise explicitly, then let evidence confirm or kill it.

## Step 1 — Consumer map (exhaustive, multi-repo)

A consumer is anything that issues the request. Grep **every repo in the
workspace**, not the obvious frontend:

| Consumer class | Where to look |
|---|---|
| Frontend app code | `app/`, `src/`, stores, composables, plugins |
| **CLI / scripts** | `scripts/*.sh` — easy to miss, easy to break |
| **Runbooks / docs** | `airflow/`, `docs/*.md` — operators follow these literally |
| **CI/CD** | `.github/workflows/` — an env var or curl in a deploy step |
| **Tests** | unit/integration tests pinning behavior |
| **Built output** | `.output/`, `dist/` — confirms a deploy actually ships the call |
| Generated clients | swagger/docs.go, openapi specs (documentation, **not** a consumer) |

Then **classify every hit by runtime reachability**. A hit in `.qoder/`,
`graphify/`, `graft/`, `swagger`, or a design doc is documentation — it does not
keep a route alive. A hit in `scripts/` or a workflow **does**.

```bash
# scoped sweep, secret-safe (never an unscoped repo-root grep)
grep -rn --include=*.{ts,js,vue,go,py,sh,yml} -E "/v1/auth/" \
  repo-a repo-b repo-c 2>/dev/null \
  | grep -v node_modules | grep -v '/.git/' | grep -v 'graft/' \
  | grep -v '\.qoder/' | grep -v '\.output/'
```

## Step 2 — Route vs service (the expensive mistake)

**A route with zero callers is not automatically dead code.** Check whether the
handler's *service* is the sole implementation of a capability that other entry
points also need.

Checklist for a "dead" route:
1. Does the handler call a service that **creates/mutates a core aggregate**?
2. Is there **another route** that does the same thing? (`grep` the router for
   sibling resources — absence of a `/tenants` route is itself a finding.)
3. Do seeders, migrations, or CLIs create that aggregate independently?
4. If the service were deleted, what capability disappears entirely?

Worked example — a route with no caller that must NOT have its service deleted:
`POST /auth/register` (no working frontend caller, since the page invoked a
nonexistent store method) was the **only** code path creating a named `Tenant`
with a unique subdomain and `role: admin`. The sole alternative created a
`"default"` singleton with `role: viewer`. Deleting the service = no tenant can
ever be onboarded.

**Report route and service as separate rows with separate verdicts.**
`"route removable / service load-bearing"` is a common and important answer.

## Step 3 — Legacy branch vs legacy route

Dual-mode handlers hide the legacy path in an `else` of the *same* route:

```go
if h.keycloak != nil && h.keycloak.Enabled() { return h.loginViaKeycloak(c, &req) }
// legacy path below — same route, different mode
```

When you see a runtime feature flag, determine which side of it is authoritative
in **each deployed environment**. Grep the flag across `docker-compose.yml`,
`.env.example`, and CI/CD (`cd.yml` secret injection) — a flag that is always
set in practice makes that branch the live path.

Then: **the route stays, the branch goes.** Say which one you mean.

## Step 4 — Live discrimination traps

Probing endpoints live has three traps that produce confidently wrong answers.

**Trap A — blanket middleware masks route existence.**
If auth middleware rejects before routing, *every* path returns 401, including
bogus ones. Use a deliberately nonexistent control path:

```bash
# identical bodies == middleware, NOT proof the route exists
curl -s $API/api/v1/auth/forgot-password        # {"detail":"missing authorization header"}
curl -s $API/api/v1/auth/xyzzy-does-not-exist   # {"detail":"missing authorization header"}
```
Existence is only provable from source (`RegisterPublicRoutes`) or by finding a
JWT-exempt sibling (`/auth/login` returned 302, so it bypasses the middleware).

**Trap B — same path, different server.** A sibling repo may host `/auth/*` on
ts own BFF with its own client. Verify *which* server and *which* client before
counting hits as consumers.

**Trap C — mutating probes.** Prefer non-destructive discrimination:
- Send a bogus resource id: `404 Not found` proves the route reaches the backend.
- Send a real id only when the operation is idempotent and reversible.
- After any probe, re-read the resource and confirm nothing changed, then report it.

## Step 5 — Decide per environment, and surface the intent question

Some verdicts are not technical. When the honest answer is "this is a product
decision", say so as a **one-line question** instead of guessing:

> *"Is tenant creation intentionally admin/ops-provisioned now, or is
> self-service onboarding still a requirement?"*

Do **not** try to settle an intent question by querying a dev/staging database —
if the running service points at a different host than the local container, local
rows describe nothing that matters. State the question and hand it over.

## Output shape

A per-endpoint table, then the traps, then the decision fork:

| Target | Verdict | Why |
|---|---|---|
| `POST /auth/logout` | delete | real machinery, **zero callers**; GET variant covers all |
| `POST /auth/register` **route** | decides | no HTTP caller |
| `Register` **service** | do not delete | only tenant bootstrap path |
| `POST /auth/login` | keep route, cut `else` branch | routes to IdP; used by scripts + runbooks |

Separate **"safe now (zero risk)"** from **"needs a decision"**. Never present a
product decision as a cleanup.

## Pitfalls

- Counting documentation hits (swagger, generated wikis, `.qoder/`) as consumers.
- Assuming the obvious frontend is the only consumer — miss `scripts/` and CI and
  you break automation silently.
- Treating "the UI never calls it" as "nothing calls it".
- Deleting a route and its service in one commit without checking Step 2.
- Probing live and treating a uniform 401 as evidence — run the control path.
- Guessing at an intent question instead of asking it.
- **Secret safety:** never run an unscoped repo-root grep (no `--include`, no
  `--exclude`) — it sweeps `.env` and dumps plaintext credentials into the
  transcript. Scope to `internal/`/`app/`, or add `--exclude=.env`, or mask with
  `sed -E 's/(PASSWORD|SECRET|TOKEN)=.*/\1***/'`.
