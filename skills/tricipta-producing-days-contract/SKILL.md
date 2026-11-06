---
name: tricipta-producing-days-contract
description: "Understand/fix Tricipta producing_days across plot-api (Python engine), asset-api (Go), and asset-web when a Chan scatter (WOR/GOR vs Producing Days) or Np/Tp,Lp/Tp denominator uses calendar days instead of real monthly production days. Use for producing_days/production_days token confusion, the Go plot_engine_mapper active_days defect, cut-before-derive, or an asymmetric well_code/entity-scope filter across sibling repository methods."
---

# Tricipta producing_days contract & fixes

Three distinct tokens, NEVER aliased together. Get this straight before touching any Chan scatter or Np/Tp/Lp/Tp code.

| Token | Repo | Meaning | Where |
|---|---|---|---|
| `production_days` | asset-api (Go + Airflow) | PER-MONTH, = volume ÷ production-day-rate (oil→liquid→water) | ingest `airflow/plugins/fpc_monthly_ingest/ingest.py:613` `_production_days`; DB col `production_days`; DTO `monthlyproductiondatasvc/dto.go:43` `ProductionDays *float64` |
| `producing_days` | plot-api (Python) | CUMULATIVE = `cumsum(active_days)` per entity/group in time order, OR trusted as-supplied | `plotting_core/formulas/activity.py:32` (`preserve_existing=True`, `source_policy: either`) |
| `active_days` | plot-api | per-month building block; fallback `days_in_month × uptime` | `activity.py:25-30` |

Engine has **zero** `production_days` occurrences and **no alias** for it (`preparation.py` `_apply_metric_aliases` bridges only pressure/PD-rate names). The backend's real per-month value is orphaned at the engine.

`lp_tp`/`np_tp` formulas (`plotting_core/formulas/well_performance.py:40-58`) declare `producing_days` as a named dependency alongside `liquid_cumulative`/`oil_cumulative` — so a correctly-supplied `producing_days` flows straight through to these ratios via `_ensure_metric`'s recursive dependency resolution, which honors each metric's OWN `preserve_existing` flag independent of the top-level `existing_metric_policy` ("trust" vs "recalculate") passed to `materialize_metrics`. **Do not assume `existing_metric_policy="recalculate"` discards a supplied producing_days value — it doesn't, when `preserve_existing=True` is set on the metric (true for producing_days).** That policy only matters for metrics WITHOUT `preserve_existing`, or for the top-level requested metric itself when it's already present pre-derivation.

## Chan `wor_versus_producing_days` render contract
- y = `wor` = `water_pd_rate / oil_pd_rate` (`production.py:37-41`)
- x = `producing_days` = `cumsum(active_days)` per entity/group
- identity/time (`analysis_profiles.yaml:583`): `date` (time_key), entity=`string` else `well`, group=`reservoir`
- gate: any_of[oil_pd_rate|oil_cd_rate], any_of[water_pd_rate|water_cd_rate], any_of[producing_days]

Same shape for `gor_versus_producing_days` (gas rates), `*_derivative_*` (Δ/Δproducing_days), and `well_productivity.{lp_tp,np_tp}_*_versus_days`.

## The classic defect (Go mapper — active_days aliasing)
`asset-api/internal/application/monthlyproductiondatasvc/plot_engine_mapper.go:68` sets `active_days = daysInMonthFromTime(rec.Time)` (calendar 30/31), while the REAL `rec.ProductionDays` goes to the ignored `production_days` key (`:72`). Result: engine x = cumsum(calendar days), not real producing days.

**Fix:** route `rec.ProductionDays → active_days`, guard `nil → 0` (NOT 30 — a down/0-0 month contributes 0), drop the orphan `production_days` key. Data is already in hand; it's a one-line reroute. Frontend has the same pattern: `productionDataToPlotEngine.ts:151` emits `active_days = daysInMonthFromIso`.

## A second Go defect class: asymmetric entity-scope filter across sibling CTE methods
A repository method that builds a raw-SQL CTE with a window function already correctly partitioned by entity (e.g. `PARTITION BY well_string_reservoir_id, well_string_id`) can still carry a leftover **unconditional** single-entity filter (`WHERE well_code = ?`) copied from an older single-well-scoped sibling query. Example: `monthly_production_data_repository.go`'s `attachCumulativeProductionDaysForAnalysis` (~L357-451) applies `WHERE well_code = ?` unconditionally, while sibling methods in the SAME FILE (`attachCumulativeProductionDays` ~L300-303, `FindLatestForAnalysis` ~L471-473, `ListRangeAggregates`/`ListMonthlyTrend`) all correctly guard it: `if filter.WellCode != "" { query += " AND well_code = ?"; args = append(args, filter.WellCode) }`. For field/platform-scope requests, `filter.WellCode==""` (scope_resolver.go leaves it empty for those levels) → the CTE matches 0 rows → the Go mapper's `if rec.CumulativeProductionDays != nil { record["producing_days"] = ... }` never sets the key at all for ANY row in that scope → plot-api has nothing to preserve → falls back to deriving via cumsum(active_days) over the anchor-month-cut frame (~30 days) → wrong life-to-date ratios (Lp/Tp, Np/Tp) for field/platform-scoped AI-insight requests specifically (well-scoped requests are unaffected since they pass a real well_code).

**Diagnostic signature:** compare the suspect method against sibling methods in the same file — if siblings guard the same predicate conditionally and the suspect doesn't, that asymmetry is the bug. Fix mirrors the sibling's exact guard (low-risk, convention-matching, doesn't change well-scoped behavior).

**Don't forget cache invalidation:** if the fixed value feeds a cached AI-insight response (e.g. `bubblemapinsightssvc`), bump the cache-key version field (e.g. `AnalysisVersion` in `request.go`) or flush the relevant Redis key pattern — otherwise the stale wrong value keeps being served from cache until TTL.

## Footguns
- NEVER send per-month `producing_days` directly: `preserve_existing=True` makes the engine trust it as cumulative → wrong.
- Denominator/x MUST stay cumulative (life-to-date). Np/Tp, Lp/Tp are cumulative÷cumulative productivity ratios. "Obey monthly production_days" = obey it as the per-month INPUT atom the engine cumsums, not the final value.
- Life-to-date = from FIRST production, not window start. To avoid cumsum resetting at the date-window start, send the well's full history (window filters display only) OR pre-cumulate in Go with a carry-in seed and send `producing_days` directly.
- cut-before-derive: `preparation.py:123` `select_latest_per_entity` runs before `:149` `materialize_metrics`; for spatial `well_maps.lp_tp/np_tp` (`_uses_selected_date` true) this collapses `producing_days` to one ~30-day month → `life_cumulative ÷ ~30`. Chan `diagnostic_scatter` caps skip the cut. The analysis-service path (`analysis/service.py`) has an equivalent anchor-month snapshot (`_snapshot_at_anchor_month`) that filters ROWS only (not columns) — a per-row producing_days value that's ALREADY correct (e.g. from a Go window-function CTE) survives this snapshot fine; the ~30-day collapse only happens when producing_days must be DERIVED from active_days after the cut, i.e. when it wasn't supplied at all.
- Before blaming `existing_metric_policy`, check whether the metric in question has `preserve_existing=True` — if so, the policy is a red herring; go look for why the value is ABSENT from the frame in the first place (usually an upstream Go/mapper bug), not why it's supposedly being "overwritten".

## Grounding order
Work per-repo from its own graph: `graphify query` / `graft ask` inside plot-api, asset-api, asset-web. Grep all variants: `producing_days|production_days|producingDays|productionDays|ProducingDays|ProductionDays`.
