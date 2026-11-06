---
name: tricipta-analysis-gate-vs-render-alignment
description: "Audit or align the Tricipta plot-api AI-insight analysis gate (analysis_profiles.yaml required_input_groups) against the render path (capabilities.yaml + formula registry), at BOTH verdict level (ready vs chart draws) and value level (AI number vs chart number), with a 6-second real-entry-point harness. Use when asked whether analysis_profiles.yaml follows capabilities.yaml, whether analysis reuses plot profiles, why the chart renders but the insight says 'Missing inputs', or why AI numbers differ from the chart."
---

# Aligning the Tricipta analysis gate with render-side truth

Repo: `tricipta-diagnostic-plot-api` (+ asset-api Go mapper, asset-web TS mapper for the data side).
Verified 2026-09-23 on branch `feature/748-tuning-plot`. Re-check line numbers before citing.

## Answer-first facts

- `analysis_profiles.yaml` is a HAND-WRITTEN copy. The loader (`analysis/profiles.py:132-152`, `334-364`) only checks that the capability name exists in capabilities.yaml and that every column is a registered metric. Nothing derives groups from capabilities.yaml or the formula registry. No cross-profile gate-vs-render test exists (checked `tests/test_analysis_{profiles,extractors,method_packs,api}.py`).
- Vocabulary differs by layer: capabilities.yaml lists CHART metrics (oil_rate, gor, voidage_ratio); profiles list RAW columns (oil_pd_rate...). The formula registry is the bridge; dump it with `list_formulas()` (inputs / any_inputs / input_groups / dependencies / requires_time / preserve_existing).
- The render side's own requirements export is NOT an oracle either: `describe_capability_contract(...)["required_canonical_columns"]` rejects a PI frame built from `liquid_rate` and a GOR frame with supplied `gor`, both of which render 12 points.
- Gate order (`analysis/service.py:_analyze_one_capability` L207): scope (L232) -> selected_date (L240) -> pre-materialize has_formula group members, policy recalculate (L248-273) -> `inputs_satisfy_profile` (profiles.py:114; AND of groups, OR within) -> `_missing_raw_sources` (L778; liquid_cumulative needs oil+water cumulative, water_cut needs rate sources) -> `_missing_units` (L792; reported AS missing_inputs) -> point count (L364-398; for rank_at_selected_date the snapshot has one row per entity, so ranking/exceedance need >=6 wells) -> `prepare_plot_data` (L405, recalculate) -> extractor -> spatial self-signal (L442). Blanket except (L475) -> insufficient_data with an analysis_error diagnostic and no missing_inputs.
- Render: `prepare_plot_data` (preparation.py:33) defaults to policy `trust` (L48); callers (web `diagnosticPlotRequestBuilder.ts`, Nitro `render.post.ts`) never send `existing_metric_policy`. The alias bridge `_apply_metric_aliases` (call L58, body L245-301: pressure->reservoir_pressure, oil/water/gas_rate->*_pd_rate, one-way) runs only on the render path, before derivation.
- Even under trust, supplied DEPENDENCIES are recalculated when recalculable (`formulas/registry.py:_ensure_metric`, deps loop with allow_supplied_calculated=True). So the Chan GOR chart recalculates gor while the GOR time-series chart keeps the supplied gor.

## Measured results (real entry points)

- 64/128 capabilities have profiles. The other 64 return unsupported_registry: every field_/zone_/string_ capability, reservoir_balance.production_injection_overlay, 4 well_pressure reservoir-pressure/pressure charts, and 7 maps (period_oil, period_water_cut, period_gor, cumulative water/gas injection, reservoir_pressure, pressure).
- Gate-minimal + render-minimal frames: 28 profiles agree everywhere, 36 disagree on >=1 frame.
- Production-shaped frames (Go mapper key sets, nil skipped): verdicts disagree only for `well_injection.active_injectors` (water-only or gas-only injector: chart draws, AI "Missing inputs: gas_injection_rate") and `well_maps.string_location_map` on injector-only snapshots (`method_packs.py:639-641` no_producers, even with plotted injectors).
- Value mismatches on production-shaped frames (same anchor month on both sides): `well_production.gor` AI 468.8 vs chart 586 (= GOR x uptime), `gor_bubble_map` x0.86, `gas_rate_bubble_map` AI x1.17-1.22.

## Misalignment classes

1. Vocabulary: profile *_pd_rate/*_cd_rate vs chart oil_rate/gas_rate/liquid_rate/gor/water_cut/wor (22 profiles). Masked in production because both mappers send both names.
2. AND-vs-OR: active_injectors (formula any_inputs, `formulas/helpers.py:41-45`); rf_versus_hcpvi (profile ANDs all 3 injection cumulatives; the formula sums whichever exist; with none present the chart still draws degenerate x=0 points).
3. Render-only alias bridge (PI, injectivity, Hall, Hearn, WOR/liquid maps).
4. Policy trust vs recalculate: the gor formula's inputs are oil_rate + gas_rate, and BOTH mappers set gas_rate = calendar-day but oil_rate = production-day (Go plot_engine_mapper.go:92,94; web productionDataToPlotEngine.ts:165,167) -> recalculated GOR = true GOR x uptime, unit lost.
5. Analysed column != drawn column: the spatial extractor uses `resolved_columns[0]` when capability.metric is not in the profile's unit_requirements (`method_packs.py:406-407`); time-series falls back at `extractors.py:64-65` when the metric is absent.
6. Hidden requirements / wrong reasons: VRR & HCPVI need Bo/Bw/Bg for any non-zero component (`formulas/helpers.py:65-81`; the voidage formula declares NO inputs, reservoir_balance.py:24); the AI then reports `minimum_points`. The oil_injection_cumulative map is routed to the producer-only cumulative path (`method_packs.py:388-397`, `950-953`).
7. Data layer: the Diagnostic Plots chart is fed by the WEB mapper (`productionDataToPlotEngine.ts`, from asset-api `format=diagnostic_plot` rows); the AI by the GO mapper (`plot_engine_mapper.go` via `analysis.go:84`). Bubble-map engine renders use the Go mapper (`format=plot_engine_inline`) or a compact PD-based web payload (`buildAnchorMonthPayload`). Live differences: injection rates CD (web L191-192) vs PD (Go L117-118); gas rates MMscf/d (web, labelled) vs scf/d (Go). NOT live: THP. The web mapper has a THP branch (L210-213, which also emits `pressure`, aliased to reservoir_pressure by the render), but diagnostic_plot rows always carry Thp nil (asset-api `monthlyproductiondatasvc/mapper.go:82-84`), so THP reaches neither path today [from code, not live-verified]. Do not report a THP chart-vs-AI divergence.

## Harness recipe (~6 s for 64 profiles)

1. Universe frame: 12 monthly rows per string, every column any capability wants, physically consistent (uptime 0.80-0.95; CD = PD x uptime; gas_rate = CD to mimic the mappers; monotonic cumulatives; Bo/Bw/Bg; rock props; coordinates).
2. Gate-minimal sets: product of one pick per required group (x each alternative path).
3. Render-minimal sets: DNF expansion of `capability_metric_names(cap)` through `list_formulas()`: dependencies recursive; inputs not produced by a dependency must be literal; any_inputs and input_groups are OR clauses; alias sources as leaf alternatives; a 'supplied metric' alternative only for columns a mapper can emit; spatial adds bottomhole_x/y. This is a candidate GENERATOR only — never use it (or `_formula_inputs` / `describe_capability_contract`) as the verdict.
4. Judge every candidate with the REAL functions: `analyze_capabilities(frame, [cap], units=..., context=...).capabilities[0]` vs `build_plot_response(frame, capability=cap, time='date', entity=..., units=...)`; render ok = >=1 finite point.
5. Repeat on production-shaped frames (exact Go mapper key sets per string type: oil producer, water injector, gas injector, WAG) and compare the AI latest/total with the chart value for the SAME anchor month.

Traps (each produced a false result once):
- The result attribute is `.capabilities`, not `.analyses`.
- Spatial: pass `x='bottomhole_x', y='bottomhole_y', selected_date=...` (the web does). Metric values live in `trace.marker.metric_values`; trace x/y are coordinates; a missing metric still draws bare dots with `missing_metric`. Without selected_date the render keeps one marker per string-month, so any total/max spans every month while the AI analyses one anchor month — that produced x0.07-x0.87 "mismatches" on 15 maps, 13 of them false.
- Ranking bars are horizontal: values on x, well names on y.
- Cross-sectional profiles need >=6 wells; vary values per well, or `_apply_alias_duplication_gate` collapses identical values across strings.
- Injection maps need status='injector' rows (injector path); producer maps drop non-producers.
- Hall/Hearn/injectivity need bhp > p_res (delta_pressure = bhp - p_res, pressure.py:8-13); PI needs p_res > bhp.
- Run with `PYTHONDONTWRITEBYTECODE=1 venv/bin/python` — the repo tracks .pyc files.
- The render raises KeyError in `transforms/ranking.py:13` when the ranking metric is absent (injector-only field); wrap render calls.

## Correct target

- A conformance test asserting verdict AND value equality over gate-minimal, render-minimal, and both mappers' production shapes. A YAML-vs-YAML equality test misses classes 3-7.
- Write groups in chart-metric vocabulary (the Ratio-family maps already do: `any_of: [wor]`) and let the derivation pre-step resolve raw inputs; or replace groups with one shared `can_render(frame, capability)` built on the render's alias bridge + materialize_metrics with the SAME policy.
- Extractors analyse `capability.metric`; one existing_metric_policy for both paths; fix the gas CD / oil PD basis mix in both mappers.

## Data-side note (asset-api)

`LoadInsightAnalysisData` (monthlyproductiondatasvc/analysis.go:57) reads only `ast_production_data`. The Go mapper emits `producing_days` (from CumulativeProductionDays) but never bottomhole_pressure, tubing_head_pressure or pressure (the latter two appear only in the units map, L34-35), so pressure-based Missing inputs on the AI path is genuine absence.
