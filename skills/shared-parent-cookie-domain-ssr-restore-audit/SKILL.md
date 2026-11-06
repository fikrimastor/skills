---
name: shared-parent-cookie-domain-ssr-restore-audit
description: "Scope a 'make auth cookies visible to both app hosts, then restore SSR' change (shared parent cookie Domain + server-side session validation) in a split-host SPA/SSR stack — verify the cookie parent is actually reachable, find the dual-cookie logout trap, and size the SSR-restore blast radius from source counts."
---

# Shared parent cookie Domain + SSR restore — feasibility audit

Use when someone proposes "add a shared parent domain to the auth cookies so the SSR server can read the session, then turn SSR back on" — typically as the follow-up to a default-deny SSR security fix. That proposal is usually written as a config flag; it is not.

## The one question that gates everything

**Is there a usable shared parent domain?** Two hosts on `a.<suffix>` and `b.<suffix>` can only share a cookie parent if `<suffix>` is a registrable domain (not a public suffix) AND both apps actually live under it.

```bash
# Is the platform domain a public suffix? (browsers will then REFUSE Domain= that value)
curl -s https://publicsuffix.org/list/public_suffix_list.dat -o /tmp/psl.dat
grep -c "<platform-domain-fragment>" /tmp/psl.dat
```

Critical inversion: **absence from the PSL means browsers ACCEPT the domain, not reject it.** So a platform-wide suffix is worse than useless — it broadcasts the session cookie to every unrelated tenant on that platform. Scope to the environment-level suffix, or use a custom domain with a wildcard cert.

Then determine whether the two apps share that parent:

```bash
az containerapp show -n <app-a> -g <rg> --query properties.environmentId
az containerapp show -n <app-b> -g <rg> --query properties.environmentId
# same ID → environment-scoped suffix is a valid parent; different → custom DNS suffix required
```

If the deployment config only exposes the hosts as secret references (`secretref:` in the CI workflow), the answer is **not verifiable from the repo** — say so and send it to the portal. Do not infer it from a shared GitHub `environment:` or resource group.

## Find every cookie write and clear before proposing anything

```bash
grep -rn "SetCookie\|setSessionCookies\|clearSessionCookie" --include=*.go .
```

For each: name, `Path`, `Domain`, `HttpOnly`, `Secure`, `SameSite`, `MaxAge`. Then check whether `Domain` comes from a host derivation:

- Go idiom to look for: `cookieDomain = url.Parse(backendURL).Hostname()` — host-only, never a parent.
- Confirm no `COOKIE_DOMAIN` / `COOKIE_SAMESITE` / `COOKIE_SECURE` env exists yet (grep). If absent, the change adds a new config surface.

## The dual-cookie trap (this is the bug that ships)

A deletion cookie written with the **new** `Domain` does **not** delete the **old host-only** cookie — host-only and domain-scoped cookies with the same name are *different cookie keys*. Logout therefore silently leaves a live session until the access/refresh `MaxAge` ages out. Verification requires clearing **both** variants during rollout.

Checklist:
1. Every clear path repeats the new `Domain` — and additionally writes a host-only deletion for the legacy cookie.
2. `SameSite` needs **no** change: once both hosts share the parent they are same-site, so `Lax` keeps working. Do not let the summary imply a SameSite change is needed.
3. Rollout order: infra (domain + cert) → API (sets the new Domain) → web (flips SSR on). Nothing can flip two steps at once.

## Size the SSR-restore blast radius from counts, not from reading pages

Four questions, each answerable by grep:

1. **Does route-config need per-route entries?** Check whether the protected manifest is `import`ed by the framework config. If it is a test/classification artifact only, inverting the catch-all (`'/**': { ssr: true }`) needs no per-route entries. Grep the config for the manifest import.
2. **Does the state store leak across requests?** A setup-style store (`defineStore('x', () => { … })`) creates refs per instance → per-request isolation for free. An options store with a shared `state` object needs attention. Read the `defineStore` call shape.
3. **Will pages 500 during SSR?** Grep pages/components for `window.` / `document.` / `localStorage` / `new Chart` and classify each hit as *top-level setup* (dangerous) vs *inside `onMounted`/`watch`/handler* (inert) vs *explicitly guarded* (`process.client`, `.client` plugin). A high raw count with zero top-level hits means the risk is hydration mismatch, not 500s — a different problem.
4. **Does server-side session plumbing exist at all?** Grep for `useRequestHeaders` / `useRequestFetch` / forwarded-cookie `$fetch`. Absent means a new server-side data path, not a tweak. Also check the auth plugin's filename: a `.client.ts` plugin never runs during SSR.

## The trap that passes every local test

The access cookie's `MaxAge` is short (derived from the IdP token lifetime) while the refresh cookie runs to 24 h. A server-side gate that probes only `/auth/me` and denies on non-200 bounces **legitimate** users after the access token lapses. The server path must fall back to `POST /auth/refresh` and **forward the resulting `Set-Cookie`** before making its allow/deny decision. Add an explicit "reload after access-cookie expiry" test case.

## Reporting shape

1. Blocking precondition first (the shared parent), with the portal command that answers it.
2. A table of verified facts, each with `file:line`.
3. What changes in each repo — code-small vs rollout-subtle, separated.
4. What does **not** need to change (same-site status, store isolation, guarded pages) — this is where over-scoping gets caught.
5. Ranked sequence with estimates, and the strategic note: if the team never intends to add the server-side gate, the change buys only first-paint polish and should be skipped.
