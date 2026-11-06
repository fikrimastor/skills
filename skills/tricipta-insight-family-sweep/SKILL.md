---
name: tricipta-insight-family-sweep
description: "When checking whether AI insight answers for each Diagnostic Plots family/capability, or re-verifying after a fix."
---

# Tricipta Diagnostic Plots AI Insight family sweep

Goal: prove, per capability and per scope, whether the insight answers (OK), never fires (INERT),
reports missing inputs, or errors — then group failures by signature. Planning afterwards goes to
the house-format spec/plan (`skill://tricipta-spec-plan-house-format`).

## Setup

- Stack: web `:3000`, asset-api `:8085`, plot-api `:8000`, Keycloak `:8180`. Log in ONCE in the
  shared browser (`skill://tricipta-common-platform-login`); subagents inherit the session. Each
  subagent opens its own named tab.
- Families come from
  `tricipta-edafy-asset-web/app/utils/diagnosticPlotCuratedFamilies.ts` — 4 of them: Rates &
  Production (11), Injection/Pressure/Activity (14), Diagnostic Scatter (19), Well Ranking (2,
  hidden at well scope by design).
- Scope URLs: `/common-platform/<field-slug>/diagnostic-plots`,
  `/<field-slug>/<platform>/diagnostic-plots`, `/<field-slug>/<platform>/<well>/diagnostic-plots`.
  Example angsi / ANDP-B / ANSI-B002; the slug `angsi` maps to `field_code` ANSI. Pick a top-ranked
  well so data exists.

## Fan-out

One subagent per family, at most 2 concurrent, identical output contract:

`Scope | Capability | Chart rendered | Insight result (OK/ERROR/MISSING-INPUTS/EMPTY/INERT/TIMEOUT/NOT-SHOWN) | HTTP | Evidence`

plus failures grouped by error signature with one representative request body. Don't poll `hub
wait` in a loop — results auto-deliver.

## Capture method

- The insight **AUTO-fires** (~400 ms debounce) on capability/scope/period change. There is no
  generate button.
- Hook `window.fetch` in-page and log URLs containing `/insight/diagnostic-plots` into
  `window.__insightLog` as `{reqBody, status, bodyText, ms}`. Install the hook inside the same
  evaluate call that clicks the capability, and reinstall it after every full navigation (SPA-nav
  tabs wedge — use a fresh tab per scope).
- Per capability record: whether the chart rendered (canvas vs "Unable to render" / "Insufficient
  production data"), the insight class, HTTP status, and a panel excerpt.
- Screenshots time out on these dashboards — use DOM text + canvas presence.
- Classes:
  - **INERT** — no request sent; panel shows "Select a field/platform…". Comes from the client
    aggregate gate (web allowlist), not a backend failure.
  - **MISSING-INPUTS** — 200 with `availability.insufficient_data`; read `missing_inputs`.
  - **ERROR** — 422 `unsupported_scope`, or a 502 from contract validation.
  - The interesting class is **chart renders but insight fails**.

## Direct API confirmation (do this before planning fixes)

From one bash/JS script with ONE login (rate limit 0.167/s; lockout after 5 failures):

```bash
TOKEN=$(curl -s -X POST localhost:8085/api/v1/auth/login \
  -H 'Content-Type: application/json' -d '{"username":"testuser","password":"testpass"}' | jq -r .token)
```
then `POST /api/v1/insight/diagnostic-plots` with
`{scope_level, field_code, [platform_code], capability, from_time, to_time}` (ranking also takes
`selected_date`, `top_n`) or `{well_code, capability, from_time, to_time, tubing_string:"ALL", reservoirs:[]}`.

To confirm data absence:
`GET /api/v1/production/aggregate?field_code=..&group_by=field&from_month=..&to_month=..` and check
column max values. Python `requests` is not installed (PEP 668) — use curl/jq or JS fetch.

## Interpreting results

- A 422 at aggregate scope means the capability isn't declared in `curated.go` Scopes; plot-api
  `allowed_scopes` must match too. Unblocking needs all three layers (web allowlist, `curated.go`,
  `analysis_profiles.yaml`).
- Repeat each LLM call 5-6 times: narratives vary between calls and can contradict each other. Check
  the status badge against the text, look for cumulative "change" values below zero, and confirm
  ranking LATEST is the top string's value, not an average.
- Aggregate responses may come from the snapshot cache (`meta.source=cache`) — flush Redis before
  re-verifying a fix.

## Known signatures (2026-09-29)

- 502 `takeaways.<slot> is required for a built card slot` — intermittent LLM omission; the
  asset-api parser contradicts the prompt.
- 422 `INSIGHT_UNSUPPORTED_SCOPE` on exceedance at well scope — an aggregate-only capability the web
  still sends.
- 200 insufficient_data `minimum_points` with a flat chart — an all-zero aggregate (COALESCE to 0)
  dropped by the plot-api explicit-zero gate; check `GET /api/v1/production/aggregate` max values.
- 200 but the narrative contradicts the badge — the model never receives the deterministic status;
  repeat the request 5-6 times to show the text varies.

## Deliverable

Per-scope and per-family tallies, then a findings list grouped by error signature. Triage of the
individual failures → `skill://tricipta-insight-debugging`.
