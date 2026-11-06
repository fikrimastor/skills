---
name: tricipta-insight-debugging
description: "When a Tricipta AI insight returns 502/503/422, shows 'Missing inputs', or its grounding gate misjudges claims."
---

# Tricipta AI Insight debugging

Systematic triage for the insight pipeline. Four distinct failure classes; diagnose which one before
touching code.

## Pipeline map

```
asset-web (WellAiInsightDrawer.vue, useWellAiInsight.ts)
  -> asset-api POST /api/v1/diagnostic-plots/insight
     diagnosticplotinsightssvc: capability_resolver -> monthlyproductiondatasvc loader
       -> plot-api POST /v1/analysis/capabilities   (analysis gate)
       -> llmsvc.Ask (Databricks OpenAI-compat)
       -> ParseLLMNarrative (syntax + schema validation)
```

Other modules (`/api/v1/insight/<kind>`) follow the same shape.

## 1. Triage by log signature

| Log signature | Layer | Root cause class | Fix pattern |
|---|---|---|---|
| 502 `malformed LLM JSON output` | ParseLLMNarrative | corrupted/prose-wrapped JSON | `salvageJSONObject`, then ONE retry on syntax failures only (`ParseLLMNarrativeSyntax`); schema violations never retry |
| 400 `must contain the word "json"` | Databricks gate | `json_object` mode requires a literal lowercase `json` in the messages (case-sensitive) | add lowercase `json` to BOTH system and user prompt |
| 502 `invalid cards count` / `invalid findings count` / `cannot unmarshal number` | parseValidated | schema violation with repairable content | clamp (>5 findings → first 5), coerce (`flexString` accepts numeric scalars); zero findings is legitimate |
| 502 `takeaways.<slot> is required for a built card slot` | asset-api parser | intermittent LLM omission; parser contradicts the prompt | align prompt/schema per §2 |
| 422 `unsupported_scope` / `INSIGHT_UNSUPPORTED_SCOPE` | asset-api resolver | `curated.go` Scope disagrees with chart behaviour, or an aggregate-only capability the web still sends at well scope | flip Scope in `curated.go` AND `scope`/`entity_strategy` in plot-api `analysis_profiles.yaml` together; remove the web entry |
| 503 / insufficient_data `minimum_points` while the chart renders | loader/gate | chart and API filter the same concept on different columns; or an all-zero aggregate (COALESCE to 0) dropped by plot-api's explicit-zero gate | diff the two filter paths column-by-column; verify real data with SQL (see §4) |
| 200 but the narrative contradicts the badge | prompt | the model never receives the deterministic status | repeat the same request 5-6 times to show text varies; then §2/§3 |

Repair-vs-reject rule for LLM contract enforcement:
- **Repairable (never 502):** over-length lists (clamp), wrong scalar type (coerce), missing optional
  arrays (default empty).
- **Reject (502):** wrong enum values, missing required fields, malformed JSON after salvage.
- **Retry:** only transient syntax failures, exactly once. Test contract: schema failure →
  deterministic 1 call; malformed JSON → 2 calls (`fakeLLMClient.responses` sequence).

## 2. "Missing inputs: <metric>" while the chart renders fine

Symptom: chart plots well; the drawer shows `NO INSIGHT AVAILABLE — Insufficient Data, Missing
inputs: <metric>`.

Two pipelines, two gates:
- Render path `POST /v1/plots/render` → `prepare_plot_data` → `materialize_metrics` **derives**
  formula-backed metrics (e.g. `producing_days` = cumsum of `active_days`).
- Analysis path `analyze_capabilities` in `plotting_core/analysis/service.py` gates
  `required_input_groups` (from `analysis_profiles.yaml`) against **raw frame columns, before
  derivation**.
- Asset-api mapper `internal/application/monthlyproductiondatasvc/plot_engine_mapper.go` emits
  `active_days` but NOT `producing_days` (it emits DB's `production_days`, a different key).

Diagnose:
1. Reproduce in-repo with a minimal frame shaped like the mapper output (`active_days` present, no
   `producing_days`) and call `analyze_capabilities` directly with `venv/bin/python` (the repo uses
   `venv/`, not `.venv/`).
2. Inspect `status`/`missing_inputs`. If the named metric is formula-derivable (`has_formula` in
   `plotting_core/formulas/`), the gate is not derivation-aware.
3. Also check the units gate: `_missing_units` requires caller-supplied units for every
   `unit_requirements` entry — callers can never supply units for metrics they never sent.

Fix (implemented in `service.py`, commit `d1c5492`):
- Pre-materialize absent-but-derivable required-group members via `materialize_metrics` before
  `inputs_satisfy_profile`; merge `DATAFRAME_UNITS_ATTR` units back into the caller's units map.
- Pass `exempt=derived_columns` to `_missing_units` so in-process derived columns skip the
  caller-unit check.

Verify: two regression tests (derivable input passes the gate → ready; genuinely missing still
reports insufficient_data); full pytest + CI-equivalent `compileall` + import smoke; an end-to-end
repro with the exact asset-api record shape flipping insufficient_data → ready; `graphify update .`
after code changes (repo rule).

## 3. Tuning a grounded gate live (asset-api)

Symptom: every run serves the template / returns `gate_rejected`, or a passing gate lets false claims
through.

1. **Get the facts before touching prompts.** Log every gate failure per attempt at WARN as
   `code (section: "phrase")`, never the full narrative. Without the phrases, tuning is guesswork.
2. **Rebuild before every live check** (no source mount):
   `docker compose build api && docker compose up -d api`. Never stage the user's
   `docker-compose.yml` hunks.
3. **Token:** `POST :8085/api/v1/auth/login {testuser/testpass}` → `.token`. Reuse it; the account
   locks after 5 failures.
4. **Force a fresh generation:** `GET /api/v1/insight/<kind>?...&refresh=true`. Read `meta.source`,
   `meta.repair`, latency.
5. **Classify each logged phrase.** False positive → fix the gate (digit ids like `S01`, rounding
   tolerance narrower than the allowlist, imperative verbs "Confirm/Check/Verify" hitting the
   certainty rule). True violation → fix the prompt (numbers spelled out, missing rate citation,
   missing required warning). Never loosen rules that block false claims.
6. **Make attempt 1 pass:** put the requirements in the first prompt as a short checklist containing
   the pre-computed exact values — ids, rates, water cuts, the contradiction sentence data.
7. **Make the retry corrective:** pass the facts; per failure include the section, the offending
   phrase and the required content. Keep `meta.repair` as unique codes only.
8. **A passing gate is not proof.** Read the served `source=ai` text against facts every time.
   Universal claims ("all above 80%") and omitted required warnings need deterministic,
   sentence-scoped rules, each with a test using the exact live sentence.
9. **Budget:** each attempt ≈10 s at an 18k-char prompt, 2 attempts. A 15 s budget is too short; the
   default is now 30 s. A 502 `ai_unavailable` is a provider/budget problem — check logs for
   `context deadline exceeded`, 403, or 410 (retired model), and `docker exec edafy-api printenv` for
   the budget.
10. **Bump versions:** `PromptVersion` on every prompt change, `OutputSchemaVersion` when the
    response shape changes, so cache keys change.

### Raising the pass rate without loosening the gate

Policy: **AI-only.** On failure return `source=unavailable` plus a reason code. Never serve a
template narrative, and never cache unavailable results. Keep the facts tables in the response so the
panel still renders data.

1. **Measure live first** — at least 8 real `Service.Insight` generations, tabulate source and repair
   codes per run. Bench replays alone don't count as yield evidence.
2. **Classify each repair code** as a genuine error (wrong number/owner, invented ID, contradiction,
   wrong section population, same-well called inter-well) or an honest omission/style issue. Keep
   genuine-error rules strict.
3. **Pre-round facts** to the exact precision the gate checks: weight 2 dp, distance whole m, rate
   whole bbl/d, percent 1 dp. Decide thresholds on raw values, publish rounded ones — the model
   copies raw floats otherwise.
4. **Pin the request:** `temperature=0` and `response_format=json_object`. Extract the JSON object
   from surrounding prose. Fill sections fully determined by facts (limitations, seeded checks)
   server-side when the model omits them; gate any it supplies.
5. **Retry with feedback:** list each violation code, the offending snippet, and one corrective
   instruction per code. Never resend the identical prompt.
6. **Completeness via copying, not recall:** put backend-composed `must_mention` sentences in facts,
   one per required item (verdicts, field rates, top pairs). The gate checks them with a normalized
   substring + numeric equivalence (`0.5` ≡ `0.50`) under one `gate_missing_required` code, and the
   retry quotes any missing string verbatim. Provide single-ID rate clauses so the model avoids
   "48 and 164 bbl/d for A and B" sentences.
7. **One definition for every derived count** (e.g. "at ceiling" = equal at published precision).
   Sentences, gate and table must agree or the composed text becomes a factual error.
8. **Gate entity claims against relations, not phrases:** check same-well assertions against
   `same_parent`, and bind numbers to the ID named in the same clause, both before and after the
   number.
9. **Re-measure live**, then run a browser review with the production-surveillance-engineer agent
   fact-checking every AI run against the on-screen tables.

Pitfalls: each `must_mention` rule added without live measurement cut yield sharply (70% → 33%); a
copied `must_mention` text has near-zero analytical value — if diagnosis is needed, require one
composed comparison sentence (water cut on both strings of the top pair); Databricks OAuth on
401/403 → invalidate the cached token and retry once; fix UI tables through one shared component and
audit every table (fixing a single selector missed the injector table).

## 4. Direct API proof (cheaper than the browser)

One login (rate limit 0.167/s; the account locks after 5 failures):
```bash
TOKEN=$(curl -s -X POST localhost:8085/api/v1/auth/login \
  -H 'Content-Type: application/json' -d '{"username":"testuser","password":"testpass"}' | jq -r .token)
```
Note the field is `.token`. Then `POST /api/v1/insight/diagnostic-plots` with
`{scope_level, field_code, [platform_code], capability, from_time, to_time}` (ranking also takes
`selected_date`, `top_n`) or the well body
`{well_code, capability, from_time, to_time, tubing_string:"ALL", reservoirs:[]}`.

Confirm data absence with
`GET /api/v1/production/aggregate?field_code=..&group_by=field&from_month=..&to_month=..` and check
column max values. Don't use Python `requests` — not installed, PEP 668 managed; use curl/jq or JS
fetch.

Interpretation: a 422 at aggregate scope means the capability isn't declared in `curated.go` Scopes;
plot-api `allowed_scopes` must match too — unblocking needs all three layers
(web allowlist, `curated.go`, `analysis_profiles.yaml`). Aggregate responses may come from the
snapshot cache (`meta.source=cache`) — flush Redis before re-verifying a fix.

## 5. Runtime gotchas

- asset-api runs in Docker **without a source bind-mount**: `docker compose build api && docker
  compose up -d api` — a restart does NOT pick up commits. plot-api runs locally with
  `uvicorn --reload`, so commits apply on restart automatically.
- Databricks FM API supports `response_format` for any chat model, but Anthropic-hosted Claude
  supports only `json_schema` (not `json_object`). `ResponseFormat` is opt-in per call (nil = field
  omitted).
- Multi-string wells: idle strings report zeros that pass `notna()` gates; scatter "latest" must be
  the rightmost x, not an alphabetical tie-break of `iloc[-1]`.
- See `skill://tricipta-insight-family-sweep` to prove coverage across all capabilities after a fix.
