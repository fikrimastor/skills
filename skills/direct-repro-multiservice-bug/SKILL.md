---
name: direct-repro-multiservice-bug
description: "Use when a bug report spans multiple services (web frontend, API/backend, downstream engine) and static code tracing across the repos stalls or loops without converging on root cause — reproduce directly by querying the real database for actual data and POSTing the exact payload straight to the internal/backend service, bypassing the web UI, auth, and log archaeology, to isolate root cause fast."
---

## When to use

A bug report spans multiple services (web frontend → API/backend → downstream engine/service), and static code tracing across the repos — reading handlers, DTOs, resolvers, extractors — is not converging on a root cause, or you find yourself re-deriving the same hypothesis repeatedly without new evidence. This is also the right move if a loop-guard or similar mechanism flags repetitive reasoning without progress.

## The problem with static tracing here

Reading code across 3+ repos to guess which capability/request/field caused a bug is slow and speculative: you're reasoning about what SHOULD happen, not what DID happen. Each hypothesis ("maybe the web sent the wrong capability", "maybe the container is stale", "maybe the profile is misconfigured") requires another read-through to rule out, and it's easy to loop without new evidence.

## The fix: reproduce directly against the real backend

1. **Confirm which database the running service actually uses.** Don't assume the local dev DB has data — check `.env`/config for `DB_HOST`/`DB_NAME`. A local Docker Postgres can be completely empty while the running app actually connects to a staging/remote DB. Query whichever one the app's config says it uses.
2. **Query the real database directly** for the exact rows involved in the bug report (the specific entity ID, date range, etc.) — in the shape the failing service actually consumes. Check the mapper/serializer code first for the exact field names it emits (e.g. `date`, `well`, `oil_pd_rate`) so your reproduction payload matches production exactly.
3. **Build the exact request payload** the calling service would construct — same JSON shape, same field names, same required context fields (API keys, capability/entity IDs, etc.). Read the actual mapper/DTO code, don't guess the shape.
4. **POST directly to the internal/backend service** (bypassing the web UI, auth middleware, and any gateway/BFF layer) using its own API key if it needs one. This cleanly isolates whether the bug is in that service's own logic vs. upstream request construction — a manually-verified-correct payload proves the callee's behavior in isolation.
5. **Read the full response, especially diagnostics/evidence arrays, not just the top-level status field.** Structured diagnostics (gate names, excluded-row counts, error codes) in a well-instrumented service usually name the exact internal step that caused the failure — grep straight to that code, don't re-trace the whole call graph.

## Why this works

- Replaces N speculative code-reading passes with one grounded data point.
- Cleanly separates "is this a caller bug" (wrong payload/capability sent) from "is this a callee bug" (wrong internal logic) — a hand-built, verified-correct payload posted straight to the callee proves the callee's behavior in isolation, no upstream ambiguity.
- Diagnostic/evidence fields in a well-instrumented service are often self-documenting about root cause — read them before grepping source for hypotheses.

## Worked example

Bug report: an AI-insight feature (web → API → downstream plot-analysis engine) showed "insufficient data — missing inputs: multiple_entities" for a well-level oil-rate time series that visibly had real underlying data in the UI. Static tracing across the web dashboard's capability-selection logic, the API's capability resolver, and the engine's profile config didn't converge — reasoning looped across the same hypotheses (wrong capability sent? stale container? misconfigured profile?) without new evidence.

Direct reproduction broke the loop in 3 tool calls:
1. Queried the real (staging) database for the well's actual rows.
2. Built the exact request payload the API would construct (verified field names against the mapper source).
3. POSTed straight to the engine service's analysis endpoint with its API key.

The response's diagnostics array showed a data-cleaning gate had zeroed out all input rows (explicit-zero filtering + alias-duplication detection), and a separate unconditional entity-count check — meant only for cross-sectional/ranking capabilities — mislabeled the resulting empty frame with the wrong missing-input code instead of falling through to the correct "not enough points" branch. Root cause found and localized to one specific unscoped conditional, from a single reproduction call.
