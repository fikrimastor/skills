---
name: tricipta-bubble-map-engine-capability
description: "Wire a bubble-map capability through the diagnostic plotting engine (well_maps.*) in tricipta-edafy-asset-web: the engine render call + metric_values parsing, unit-scale traps (gas scf/d vs MMscf/d), the injector-vs-producer spatial filter, the design-parity per-string table + tooltip, and the engine→frontend-accumulation→mock fallback chain. Use when adding new Bubble Maps capabilities, wiring engine-computed metrics, or debugging engine-vs-frontend value mismatches."
---

# Bubble Map Engine-Capability Integration (tricipta)

## When to use
Wiring a new bubble-map capability through the plotting engine (`well_maps.*`), porting an existing engine path to a new capability, or debugging why engine values disagree with the map/table/tooltip.

## Wired capabilities (as of 2026-08-24, all verified live)
| Map | Engine capability | Metric | Unit handling |
|---|---|---|---|
| Lp/Tp | `well_maps.lp_tp` | `lp_tp` | engine derives (liquid_cumulative / producing_days) |
| Np/Tp | `well_maps.np_tp` | `np_tp` | engine derives (oil_cumulative / producing_days) |
| Oil Rate | `well_maps.oil_rate_bubble_map` | `oil_rate` | 1:1 (bbl/d) |
| Water Rate | `well_maps.water_rate_bubble_map` | `water_rate` | 1:1 (bbl/d) |
| Liquid Rate | `well_maps.liquid_rate_bubble_map` | `liquid_rate` | 1:1 (bbl/d) |
| Gas Rate | `well_maps.gas_rate_bubble_map` | `gas_rate` | **÷1e6 in web parser** (Go emits scf/d, label MMscf/d) |
| Water Injection | `well_maps.water_injection_rate_bubble_map` | `water_injection_rate` | 1:1 (bbl/d) + `status: "injector"` filter |
| Gas Injection | `well_maps.gas_injection_rate_bubble_map` | `gas_injection_rate` | **÷1e6** + `status: "injector"` filter |
| Water Cut | `well_maps.water_cut_bubble_map` | `water_cut` | Go fraction → bundle ×100 → %; producer mode |
| GOR | `well_maps.gor_bubble_map` | `gor` | `gas_oil_ratio` trend; scf/stb 1:1; producer mode |
| WOR | `well_maps.wor_bubble_map` | `wor` | bbl/bbl 1:1; computed (no DB trend column); producer mode |
| Cumulative Oil | `well_maps.cumulative_oil_bubble_map` | `oil_cumulative` | **÷1e6** (stb → MMstb), producer mode |
| Cumulative Water | `well_maps.cumulative_water_bubble_map` | `water_cumulative` | **÷1e6** (stb → MMstb), producer mode |
| Cumulative Liquid | `well_maps.cumulative_liquid_bubble_map` | `liquid_cumulative` | **÷1e6** (stb → MMstb), producer mode |
| Cumulative Gas | `well_maps.cumulative_gas_bubble_map` | `gas_cumulative` | **÷1e9** (scf → Bscf), producer mode |
| Cumulative Oil Injection | NOT wired — no source data (same as Oil Injection Rate) | `oil_injection_cumulative` | No source; asset-api returns the fixed unavailability statement |
| Oil Injection | NOT wired — no source data (`oil_injection_rate` omitted by Go mapper; ast_production_data has no oil injection column) | | |

The cumulative extractor mode is `_extract_spatial_cumulative`: it applies the producer filter, keeps and counts explicit zero as a genuine never-produced string, and has no all-zero gate (Go handles that state).
The ratio extractor mode is `_extract_spatial_ratio`: it applies the producer filter, excludes blow-ups above GOR > 1e6, WOR > 1000, or WC > 100% with a `ratio_blowup_excluded` diagnostic and `blowup_excluded_points` coverage, keeps explicit zeros as valid values, and records the `naive_sum` flag. WOR is trendless (snapshot-only, no DB trend column); Water Cut and GOR use the `water_cut` and `gas_oil_ratio` trend columns. The insight gate exempts trendless WOR. 

Wiring recipe per capability: add `*_CAPABILITY` const + `engineCapabilityForMetric` case in `bubble-maps-dashboard.vue`; add `REAL_VALUE_METRIC_KEYS` key if missing. The asset-api bundle normalization has Go-side equivalents for the parser scales — ÷1e6 for oil/water/liquid stb → MMstb and ÷1e9 for gas scf → Bscf. The frontend accumulation + mock remain the fallback — never remove them.

## Canonical data flow (verified end-to-end 2026-08, ANSI 114,839 records → 266 points / 222 non-zero)
```
field rows + WSRL locations (web)
  → fetchPlotEngineInlineAll (paginated full-history plot_engine_inline fetch)
  → POST /api/diagnostic-plots/render  (Nuxt proxy → engine :8000)
  → engine marker.metric_values (server-computed)
  → loadEngineMarkerValues parses → resolveRealBubbleValue (engine-first)
  → bubbles + per-string table + CSV + tooltip
```

## Engine contracts (all verified live)
1. **Coordinates mandatory**: `bottomhole_x`/`bottomhole_y` (local CRS preferred) on inline records, else the spatial gate (`canRenderCapabilityWithRecords`) rejects the render → engine never called → silent frontend fallback. This was THE all-zero Lp/Tp bug: Go API previously emitted only `active_days`; WSRL enrichment fixed it.
2. **Status filter trap**: `buildPlotOptions` (diagnosticPlotRequestBuilder.ts) spatial case sends `status` — `"injector"` for `*injection_rate_bubble_map`, `"producer"` for everything else. A producer filter on an injection render drops every point (inferWellStatus marks injectors "injector") → 0 traces, verified live. When wiring an injector-family capability, do NOT forget this — it silently empties the map.
3. **Gas unit trap**: Go mapper emits `gas_rate` and `gas_injection_rate` in **scf/d** (×1e6 via mmToBase); web labels/tables/fallback are **MMscf/d**. Engine does NOT convert (echoes resolved_units only). Parser divides by 1e6: `gasScale = ["gas_rate","gas_injection_rate"].includes(metric) ? 1/1_000_000 : 1` in `loadEngineMarkerValues`. Oil/water/liquid are 1:1 (bbl/d) — no trap.
4. **`producing_days`**: engine derives LTD by cumulating per-month `production_days` (raw DB column, Airflow-computed) itself; Go API carries the per-month column as `production_days` on inline records and `diagnostic_plot` rows (no window-sum precompute). Lp/Tp-Np/Tp need it; rate maps don't care about days.
5. Labels: engine `trace.text` = the record `string` field (Go emits `well_string_code-reservoir`, e.g. "ANSI-A014-SS-I68") — matches the web's `formatLocationWellLabel` + reservoir join. `entity: "string"` required.
6. Anchor: `selected_date` = anchor month (playhead else latest in-range month) — same month the table's Period column shows. Months with no rows (e.g. 2026-09 when ingest stopped at 2026-06) render from last data month; empty-month rows emit 0 + `invalid_formula_rows` diagnostics — data gap, not code.

## Web wiring details
1. `loadEngineMarkerValues()`: `fetchPlotEngineInlineAll` (full history) → minimal spatial `DiagnosticPlotCapability` (name/metric from `engineCapabilityForMetric`) → `renderPlotFromInlineRecords(capability, records, {}, {startDate/endDate = anchor-01/-28})` → parse `neutral_json.traces[0].marker.metric_values` keyed by `text`, applying the gas scale.
2. `resolveRealBubbleValue`: engine-first (label = `${formatLocationWellLabel}-${reservoir}`), frontend accumulation fallback, mock last. Engine failure → empty Map → graceful degradation.
3. Re-render hooks: after `loadProductionData`, on `activePlayheadMonth` change (not while playing), on `[customDateFrom, customDateTo]` change.

## Design-parity per-string table + tooltip
- **String-level table** (Location & Marker, Production Rate, Injection Rate categories — `isStringLevelTable`): No. | Well String | Platform | Reservoir | **Period** | metric(unit). One row per plotted string, UIUX-design parity. Period column kept on ALL tables (user requirement). Injection maps render `—` for producer strings (`format` checks `metricConfig.mode === "injector" && !row.isInjector`), matching UIUX `isActiveWell(d) ? fmt(getVal(d)) : '—'`. Ratio/Cumulative categories keep the monthly production-log table.
- **Tooltip**: header (well), platform · reservoir, WGS84 lon/lat, **Easting/Northing** (dx/dy local metres), **Period · YYYY-MM**, metric + unit. `HoveredPoint.period` from `periodForPoint()` = `activePlayheadMonth ?? latestMonthForPair(...)` — same rule as the table Period, so tooltip and table agree; refreshed on visibleData watch.

## Go backend (asset API)
- `plot_engine_inline` records carry: `production_days` (per-month, null when unset), `reservoir` (ReservoirCode), `bottomhole_x/y` (WSRL FindByPairs enrichment via `enrichPlotEngineInlineCoordinates`, key `WellStringID|ReservoirID`), `status` (injector when water/gas inj PD rate > 0). `oil_injection_rate` deliberately NOT emitted (no column).
- `diagnostic_plot` rows carry `production_days` + `calendar_days` (frontend fallback `cumulativeLifeAverage` needs `row.productionDays` — without it lpTp/npTp resolve 0).
- These landed 2026-08-24 (feature/653-well-test-api). Before: inline records had ONLY `active_days` (calendar month length) — engine path dead, fallback zeroed.

## Verification
- Live engine e2e: page-walk `plot_engine_inline` (or DB export with mapper semantics — water_injection PD rate as-is, gas ×1e6, status injector, string = code-reservoir, coords from ast_well_string_reservoir_locations), POST with spatial options (x/y=bottomhole, selected_date=anchor, filters.status matching family), assert `marker.metric_values`. 2026-08-24: water inj 27/27 non-zero (ANSI 2026-01); gas inj 0 in ANSI (no gas injection data in Azure — data gap, not code).
- SQL cross-check: `((np+wp)*1e6 / sum(production_days))` cumsum per string/reservoir == engine Lp/Tp values.
- Go: `go build ./... && go test ./internal/application/monthlyproductiondatasvc/...`.
- Web: `npx vitest run test/utils/bubbleMapProductionValues.test.ts`; `npx tsc --noEmit -p .nuxt/tsconfig.app.json` (ignore pre-existing useVfm* errors on this branch).
- Dev: page 200 on :3000 after edits (HMR).

## Gotchas
- Go API serves REMOTE Keycloak (`shared.auth.trinovate.io`) — direct-grant tokens from localhost:8180 fail issuer check (401 invalid token). Use DB queries for payload verification, or a browser session token.
- `format` callbacks in CPTable receive `(value, row)` — row needed for injector-dash logic.
- Engine `prepare_frame` does no aggregation — labels are raw `string` values; multi-reservoir strings render one point per reservoir row (dedup NOT engine-side).
