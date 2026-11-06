---
name: tricipta-bubble-map-criteria-doc
description: "Author or extend the Tricipta bubble-map (spatial) capabilities analysis-criteria doc for AI-insight prep (#683), the spatial analogue of the diagnostic-plot criteria doc."
---

# Tricipta Bubble Map Capabilities Analysis-Criteria Doc

Use when preparing the AI-insight (RAG) groundwork for the Well Level Bubble Map Plot (asset-web #683) — building or extending `docs/bubble-map-capabilities-analysis-criteria.md` in `~/Sites/afed/tricipta/docs`. This is the spatial-snapshot analogue of `diagnostic-plot-capabilities-analysis-criteria.md`.

## Ground scope from source, not synthesis (verify every count)
- UI maps: `tricipta-edafy-asset-web/app/utils/bubbleMapDashboardData.ts` → `BUBBLE_CATEGORIES` (category→map names) + `MAP_METRICS` (key/unit/label/mode/digits). Count = sum of category items. As of 2026-09: 18 maps, 5 categories (3+4+3+3+5).
- Engine registry: `tricipta-diagnostic-plot-api/plotting_core/domain/capabilities.yaml` — `grep -cE '^- name: well_maps\.'` (25). Engine-only = registry − UI (7).
- Wired value keys: `app/utils/bubbleMapEngineCapabilities.ts` `engineCapabilityForMetric` switch cases (`grep -cE 'case "'`). Keys with no case = mock/fallback only.
- Table shape: `app/pages/common-platform/bubble-maps-dashboard.vue` `isStringLevelTable` computed — if true for all categories, ALL maps use the one-row-per-string snapshot table.

## Key domain facts (re-verify each time; code changes)
- Bubble map = spatial SNAPSHOT at an anchor month (playhead else latest in-range), one bubble per (well string, reservoir). NOT a time series — forbid trend/decline claims; route temporal questions to the diagnostic #682 pipeline.
- Value semantics: rate (production/injection), ratio/life-average (WC/GOR/WOR, Lp/Tp, Np/Tp), cumulative. Lp/Tp = liquid_cumulative/producing_days; Np/Tp = oil_cumulative/producing_days (stb/d).
- No-source placeholders (never analyse, return unavailability): Oil Injection Rate, Cumulative Oil Injection (oil injection not measured).
- Gas unit trap: engine scf/d → web ÷1e6 → MMscf/d. Cumulative gas = Bscf; cumulative liquids = MMstb.
- Only oil/water/liquid_rate are ANCHOR_MONTH_ONLY; others need full history fed to the engine.
- Injector-mode maps: status:"injector" filter; producers render '—' (not 0).

## Doc structure (mirror the diagnostic doc)
1. Header + scope-reconciliation note (engine count vs UI count, list engine-only).
2. §0 Global SPATIAL Analysis Protocol: snapshot semantics, units/normalization, spatial data-quality gate, spatial thresholds (Pareto/VRR/offset), cross-map triangulation (same-snapshot pairs), honesty rules, per-capability template.
3. One section per UI map: Purpose / Data required / Data binding (bubble schema, engine capability name, unit+trap, mode, wiring status; point identification; table shape; tooltip; traps; gaps) / Analysis criteria (spatial) / Flags / Agent output.
4. §6 Implementation handoff (coverage, RAG input shape, snapshot-vs-trend, unit guards, next step = prompts doc).

## Verify before claiming done
- `grep -cE '^### [0-9]'` = protocol subsections + capability sections.
- No stray old counts; capability names match capabilities.yaml.
- Do NOT ship unverified bases (e.g. injection producing-day vs calendar) as source-of-truth — mark as not re-verified.

## Next
Author `bubble-map-analysis-agent-prompts.md` mirroring the criteria doc (one prompt block per capability, single string-level-table INPUT CONTRACT, §0 as shared preamble). Keep the two in sync — drift is a bug.
