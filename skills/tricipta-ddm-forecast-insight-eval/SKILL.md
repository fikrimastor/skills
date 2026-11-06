---
name: tricipta-ddm-forecast-insight-eval
description: When judging or regression-testing the DDM Production Forecast AI Insight output quality before/after an asset-api change.
---

# DDM Production Forecast AI Insight — output evaluation

Use for `GET /api/v1/insight/production-forecast?broad_string_id=…` (asset-api `internal/application/productionforecastinsightsvc`).

## 1. Judge with real responses, not code
1. Pick 5–6 strings covering: steady, declining, rising water cut, wide P10–P90, not-forecastable, short/noisy history. Record why each was picked.
2. Save raw response + underlying history/forecast numbers per string.
3. Check every claim in `summary`/`analysis`/findings against the numbers: direction (observed vs forecast), confidence vs horizon, units, P10=low/P90=high, invented causes.
4. Rule-based findings/cards are normally correct; LLM prose is where errors live.

## 2. Before/after measurement over the whole field
1. Record the running API's commit/image identity in the manifest BEFORE the run; never rebuild mid-run. Sequence: baseline finishes → then implementer rebuilds.
2. No cache-bypass param: `docker exec edafy-redis redis-cli --scan --pattern 'insight:production-forecast-insight*' | xargs DEL` (also `-nf*`). Leave model-index keys.
3. Run all strings (Bokor: 173, 58 forecastable). Capture per string: meta.source, summary_source, analysis_source, ai_available, repairs, finding codes, latency.
4. Rebuild only the api service with the user's compose file untouched: `docker compose build api && docker compose up -d api`; prove new code is live with one response.
5. Rerun with `--no-flush-cache` to prove caching still works.

## 3. Attribution rules
- Deterministic code changes (e.g. analysis_source → 100% deterministic, PromptVersion bump) are hard facts.
- LLM summary repair/replacement counts and LLM latency are NOT attributable: no temperature is set. Pin temperature 0 or run ≥3 passes and quote the spread.

## 4. Known traps
- Cache condition must depend on summary source + repairs only.
- Bump `PromptVersion` (prompt.go) on any prompt/response contract change.
- Prompt unit labels must match metric units (slopes are bbl/d per month).
- `high_uncertainty` at ratio 3.0 fires on nearly every Bokor string.
