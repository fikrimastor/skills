---
name: tricipta-marker-value-defect-debug
description: Diagnose Lp/Tp / Np/Tp (and bucket-cumulative) marker-value defects in the Tricipta Bubble Maps stack (asset-web + plot-api) when map/tooltip/table values disagree with the bubble colour — use before proposing fixes.
---

# Tricipta Marker-Value Defect Procedure

When a Bubble Maps marker (Lp/Tp, Np/Tp, or any engine-computed spatial metric) shows values that disagree with its colour/colorbar or table, do NOT guess (e.g. "÷30 days-in-month"). Follow this verified procedure.

## Symptom pattern (observed 2026-09)
- Bubble colour/colorbar domain plausible (e.g. 0–4,108 stb/d), but tooltip/table show values orders of magnitude larger (e.g. 839,606 stb/d).
- Field `Σ` on the header sums the (wrong) displayed values.

## Root cause chain (verified by live reproduction)
1. asset-web `buildPlotOptions` (`app/utils/diagnosticPlotRequestBuilder.ts`) sets `options.selected_date` for every `spatial` capability.
2. plot-api `prepare_plot_data` (`plotting_core/preparation.py`) runs `select_latest_per_entity` **before** formula materialization → only 1 row per string survives.
3. The frontend inline payload (`transformProductionRowToPlotEngineRecord`) emits `active_days` but NOT `production_days` → the engine derives `producing_days = cumsum(active_days)` over that single surviving row = one month (~28–31 days), not life-to-date.
4. Engine formula `lp_tp/np_tp = cumulative ÷ producing_days` → a ~1-month average of the life cumulative, ~30× off.
5. The bubble COLOUR is correct because the frontend fallback (`valueDomainForPairs` / `cumulativeLifeAverage` in `app/utils/bubbleMapProductionValues.ts`) accumulates LTD producing days properly (test-verified: `test/utils/bubbleMapProductionValues.test.ts`).

## Verification procedure (reproduce before fixing)
1. In `tricipta-diagnostic-plot-api`: `venv/bin/python` with a synthetic 3-month bucket (cumulatives + `active_days`, no `production_days`) → `prepare_plot_data(..., selected_date=..., entity="string", filters={"status": ...})` → observe `producing_days` = one month and `lp_tp` ≈ cumulative ÷ 30.
2. Repeat without `selected_date` (multi-row) → correct LTD cumsum.
3. Re-derive observed UI numbers from §5.3 cumulative values ÷ ~30 to confirm the match before claiming the mechanism.

## Fix options (pick per blast radius)
- **Frontend (recommended, smallest):** in `resolveRealBubbleValue` (bubble-maps-dashboard.vue) skip `engineMarkerValues` for `lpTp`/`npTp` so the verified frontend fallback (`resolveBubbleValueForMonth`) drives value + colour alike; or drop `selected_date` from spatial options for bucket-cumulative metrics.
- **Engine (widest):** derive `producing_days` before the selected-date cut in `prepare_plot_data` — fixes all consumers but touches shared engine logic.

## Analysis-validity rules while the defect is live
- Ranking / ordering across strings: VALID (constant wrong divisor, order-preserving).
- Lp/Tp-vs-Np/Tp ratio: VALID (divisor cancels → cum-liquid ÷ cum-oil).
- Absolute `stb/d` from tooltip/table: NEVER quote — read the colour scale or recompute `cumulative ÷ LTD producing_days`.

## Related divergences (distinct mechanisms, don't conflate)
- GOR/WOR maps: oil→0 blow-ups hijack the map colourbar while the table caps them saner → trust the table, exclude blow-ups.
- Cumulative maps: map (life-to-date) vs table (date-windowed) disagree only on narrowed windows; agree at full range.
