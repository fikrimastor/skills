---
name: security-gate-fail-open-review
description: "Self-review checklist for a security-gate or shared-auth-resource design (e.g. replacing a legacy-fallback, consolidating lazy-init callers) before finalizing a spec or implementation — use when designing/reviewing an enable/disable flag for an auth path, or refactoring two independent lazy-init copies of a validator/client into one shared resource."
---

## When to use

Before finalizing (or when reviewing someone else's) design for:
- A boolean or flag that controls whether a legacy/fallback auth path is accepted.
- Consolidating two independent lazily-initialized copies of an external client/validator (e.g. an OIDC validator initialized once at boot in one place, and separately with its own retry logic in another) into one shared resource.

Grounded in a real case: a JWT middleware fallback-removal fix where a `KeycloakEnabled bool` field, and later a shared OIDC validator resolver, both shipped with real exploitable/availability-breaking defects that only surfaced under structured review — not from reading the diff once.

## The checklist

1. **Zero-value safety.** For any new bool/flag that gates a permissive vs. restrictive path: what does the flag's zero value (`false`, `nil`, `0`) mean? If the zero value means "permissive" (e.g. `KeycloakEnabled: false` silently means "legacy accepted"), an unset field at any construction site reopens the hole. Prefer gating on a real, already-shared reference object (a config pointer other callers already hold) over a separately-computed derived bool — a derived bool can drift from the real state or simply never get set.

2. **Atomic wiring.** If the new gate is load-bearing in file/commit A but its wiring (setting the flag/pointer correctly at every construction site) is planned for a later file/commit B, landing A alone means every caller runs with the gate's *unwired* (usually zero) value until B lands. Fold the wiring into the SAME commit that makes the gate load-bearing. Never split "add the check" from "wire the check" across commits/tasks.

3. **Nil-receiver safety when consolidating shared resources.** If existing callers pass `nil` for an old interface-typed parameter, and you change that parameter's type to a concrete pointer, `nil` still type-checks — nothing forces those call sites to change. If the new type's method dereferences its receiver (e.g. touches a mutex field) without checking for nil first, every one of those pre-existing `nil`-passing call sites panics the first time it exercises that method. Add an explicit `if r == nil { return safeDefault }` guard as the first line.

4. **Lock scope must exclude I/O, especially after widening call frequency.** If you're moving a resource from a rare call path (login-only) onto a hot path (every request), re-audit any lock/mutex around it: holding a lock for the duration of a network call (discovery, health check, etc.) was fine when only 1-in-N requests hit it, but turns into an API-wide serialization/stall the moment every request hits it during an upstream outage. Release the lock before the I/O; cache the last error with a timestamp; add a cooldown before re-attempting, so concurrent/repeated callers fail fast against the cached error instead of each blocking on their own slow retry.

5. **Test/rate-limit compatibility.** If you add a cooldown/rate-limit as part of #4, check every test you're simultaneously mandating for that resource: a "retries until success" test calling `Get()` twice back-to-back will fail by construction if the second call lands inside a hardcoded cooldown window. Make the cooldown an exported/injectable field (or constructor parameter) so tests can zero it out for deterministic assertions, and write a SEPARATE test that specifically proves the cooldown suppresses redundant attempts at its real default value — one test proving "it retries," a distinct test proving "it doesn't retry too often."

## How to apply

Walk through all 5 items explicitly before calling a security-gate design finished, even if it already "looks" reviewed. Verify each concern against actual source (exact field defaults, exact existing call sites, exact timeout values) rather than reasoning abstractly — several of these defects were only confirmed by grepping real call-site counts and reading the actual HTTP client timeout in source, not by inspecting the new code in isolation.
