---
name: security-detector-grandfather-known-gap
description: "Ship a route/endpoint coverage or regression detector that catches a pre-existing security gap without leaving CI red, using a grandfathered known-gap list that cannot rot. Use when adding an e.Routes()-style invariant test while known violations still exist, or when a security fix splits detection from remediation."
---

# Shipping a security detector when known violations already exist

## When to use

You want a class-defeating invariant test ("every route is protected or explicitly public") but the codebase **already violates it**. A naive test ships red; a skipped test is worthless. Use this pattern to ship detection now, remediation later, with CI green.

Pairs with `security-gate-fail-open-review` (designing the gate) and `security-spec-claim-verification` (auditing the spec's claims).

## The pattern

Partition the enumerated set into **exactly-one-of-N categories**, plus a **grandfathered known-gap list**:

1. **Protected** — the real gate (e.g. the group carrying the auth middleware).
2. **Public allow-list** — intentionally unauthenticated (health probes, public auth routes, swagger).
3. **Separately-gated** — protected by a *different* mechanism (webhook secret, API key). **Never** let these into the public allow-list; that silently blesses them as "public" when they are merely "gated elsewhere".
4. **Self-gated** — the route checks its own auth inline and sits outside the middleware group (e.g. a WebSocket handler taking `?token=`). Classify explicitly rather than assuming; it is exactly the sibling path a refactor will touch.
5. **Grandfathered known-gap** — the known violations, each entry naming the issue that will remove it.

### The dual failure condition is the whole point

The test must fail on **both**:

- a route in **none** of the categories (new violation), **and**
- a grandfathered entry that has become protected **or disappeared** (the fix landed → "remove this entry").

Without the second condition the grandfather list becomes a permanent hole. Say this in the AC; it is the property reviewers should check.

## The trap that kills the detector: incomplete enumeration

A first run that fails on a route you never classified looks like a test bug and gets disabled. Enumerate the **whole** table before shipping, and expect two easily-missed classes:

- infrastructure routes registered on the **root** app (swagger UI, docs) rather than the versioned group;
- routes on a bare version group that look covered but are not (check the actual `.Use()` chain, not the group name).

Also check for **duplicate registrations**: the same handler can be mounted on both sides of the auth boundary (once protected, once not). Grep the handler name, not just the path. For those, remediation may be deleting one line, not a refactor.

## Prefer this over the alternatives

- **"Expected to fail on main"** / a skip with a TODO — normalizes a red or ignored suite; hides the gap from dashboards.
- **Adding the offender to the allow-list** — actively wrong; documents the bug as intent.
- **Adding `auth` middleware per page/route** — opt-in coverage, which is the failure mode that created the gap. (Applies to page-level middleware as much as route groups.)

## Splitting detection from remediation

When the gap needs a real refactor, ship **detection** with the security fix and file **remediation** separately, then say which is which in all artifacts. Watch three drift points:

1. The detector's AC must describe the grandfather list, **not** "ships red".
2. The remediation issue must **not** still list the detector as its own work (duplicate commitment).
3. Any "files touched" / diff-scope AC must include the detector's new test file, or it fails on the very file this added.

## Auth ≠ authz

A route passing the detector is only authenticated. Verify the handler also scopes data (tenant predicate, ownership check). Adding middleware to a handler that queries without a tenant filter yields *authenticated but unscoped* — a subtler bug than the one you fixed. Classify the two defects separately in the issue.

## Verification

- Positive control: a locally-constructed unprotected route must be reported as an offender (proves the detector works without depending on the live table).
- Live run: every known offender appears, and nothing else does.
- Re-run after any task that edits the route table itself (middleware wiring, route deletion) — the invariant is over a moving target.
