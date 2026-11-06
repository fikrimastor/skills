---
name: tricipta-bubble-map-insight-debugging
description: "Triage AI Insight failures on the Tricipta Bubble Map dashboard (POST /api/v1/bubble-maps/insight, bubblemapinsightssvc) by log signature -- GORM Pluck+uuid crashes, missing coordinate units, plot-api numpy.bool serialization 500s, stale Redis snapshot cache after a fix, and dev container API-key mismatches. Sibling to tricipta-insight-debugging (the parallel /api/v1/diagnostic-plots/insight pipeline)."
---

# Tricipta Bubble Map AI Insight Debugging

Systematic triage for the `POST /api/v1/bubble-maps/insight` pipeline -- the spatial-analysis
counterpart to `tricipta-insight-debugging` (which covers `/api/v1/diagnostic-plots/insight`).
Different Go service (`bubblemapinsightssvc` vs `diagnosticplotinsightssvc`), same plot-api backend.

## Pipeline map

```
asset-web (BubbleMapInsightPanel.vue)
  -> asset-api POST /api/v1/bubble-maps/insight
     bubblemapinsightssvc.Service.GenerateInsight:
       1. ScopeResolver.Resolve (business codes -> UUIDs, postgres Pluck)
       2. SnapshotCache.Get (Redis, keyed incl. AnalysisVersion) -> LoadBubbleMapSnapshotData on miss
       3. insufficiencyReason(bundle) -- deterministic gate, no plot-api call
       4. plot.AnalyzeBubbleMap -> plot-api POST /v1/analysis/capabilities (well_maps.* profiles)
       5. generateNarrative -> llmsvc
```

## Triage by log signature

| Log signature | Layer | Root cause class | Fix pattern |
|---|---|---|---|
| 500 `unsupported data type: 00000000-...: Table not set` | postgres repo | `Pluck` called without `.Model(&X{})`/`.Table(...)` first | chain `.Model(&Model{})` before `.Distinct/.Where/.Pluck` |
| 500 `converting driver.Value type string ("<uuid>") to a uint8` | postgres repo | `Pluck("col", &scalarUUID)` -- `uuid.UUID` is `[16]byte`, `reflect.Kind()==Array`, GORM's generic scan treats it as "one struct per array element" instead of a `sql.Scanner` scalar | Pluck into `[]uuid.UUID`, take `[0]`; GORM's Pluck contract is always a slice destination, even for the "single value" case |
| panel shows raw `"missing_inputs"` as the reason | Go adapter | adapter collapses `capAnalysis.MissingInputs` list to a static token instead of joining the names | `"Missing inputs: " + strings.Join(missing, ", ")` |
| `insufficient_data` on a capability whose profile needs a metric present in every plotted record (e.g. `bottomhole_x`/`bottomhole_y`) even when the map itself renders that data | asset-api units map | `plotEngineMetricUnits` (plot_engine_mapper.go) has no entry for a metric the analysis profile's `unit_requirements` demands; `UnitsForPlotEngineRecords` silently omits it -> `_missing_units` gate rejects EVERY month, not just data-poor ones | add the metric to `plotEngineMetricUnits`; the gate only needs a non-empty string, not a registry-validated unit (`unit_requirements` values are opaque family tags like `length`, not literal unit strings -- `_missing_units` never checks the value, only presence) |
| plot-api 500 `PydanticSerializationError: Unable to serialize unknown type: <class 'numpy.bool'>` | plot-api extractor | `python_bool_expr and (series == x).all()` -- Python's `and`/`or` return the untouched right-hand operand when short-circuiting doesn't apply, so a numpy-native type (not coerced to Python bool) lands in a dict headed for a pydantic response | wrap in `bool(...)`; same risk for any `and`/`or` mixing a Python bool with a numpy comparison/reduction result (`.all()`, `.any()`, `.sum()` on a boolean series) destined for a response dict |
| fix looks right in code + rebuilt container, but the SAME wrong verdict persists after redeploy | Redis cache | `SnapshotCache` (`bubble-map-snapshot:*`, 24h TTL) is keyed by `AnalysisVersion`, which wasn't bumped when the bundle's *shape* changed (e.g. units map gained keys) -- stale bundles from the pre-fix build keep being served | immediate relief: `docker exec <redis> redis-cli --scan --pattern 'bubble-map-snapshot:*' \| xargs -r docker exec <redis> redis-cli del`; permanent fix: bump `AnalysisVersion` in `bubblemapinsightssvc/request.go` (touches the cross-repo frozen contract fixture mirrored in asset-web + plot-api tests too, so budget for that) |
| plot-api returns 401 right after rebuilding/recreating its dev container | deploy | container recreated with a placeholder/`.env.example` API key instead of the real `.env` value the asset-api container sends (`DIAGNOSTIC_PLOT_API_KEY` in asset-api's `.env` must equal `PLOTTING_API_KEY` on the plot-api container); dev plot-api here has no docker-compose, so the running container's env is whatever was passed to a manual `docker run` at some point in the past | `docker inspect edafy-api --format '{{range .Config.Env}}{{println .}}{{end}}' \| grep DIAGNOSTIC_PLOT_API_KEY` and pass that exact value to `docker run --env PLOTTING_API_KEY=...` when recreating plot-api |

## Gate design: column-presence, not coverage-threshold

`plotting_core/analysis/service.py`'s `inputs_satisfy_profile` / `_missing_units` gates only check
whether a required column/unit is present AT ALL in the payload -- not what fraction of rows have
a valid value. A capability with even 1% coverage passes the gate; only 0% coverage (column
entirely absent -- e.g. every row's rate is NULL for the whole month) fails. The extractor then
drops invalid rows (reported as `dropped_coordinate_points`, `dropped_status_value_points` in
`coverage`) and analyzes only the valid subset. Empirically verified working at 85% coordinate
coverage (676/799 rows). Do NOT "fix" a partial-coverage rejection by adding a coverage threshold
(e.g. "proceed if >=90% have data") -- the existing any-coverage-passes design is already MORE
permissive than a percentage cutoff would be; a 90%-style threshold would be a regression against
real cases that work fine today at 70-85% coverage. A genuine 0%-coverage month is a data problem,
not a gate bug -- verify with a raw SQL count against the source table before assuming the
pipeline is broken: `select count(<rate_col>) from ast_production_data where field_code=... and
time between ... and ...` -- zero means the gate is telling the truth about the data, not lying
about the pipeline.

## Verification recipe

Reproduce end-to-end against the REAL stack (never mocks alone) with a throwaway Go test in
`monthlyproductiondatasvc_test` (external test package -- avoids an import cycle, since
`bubblemapinsightssvc` imports `monthlyproductiondatasvc` and vice versa for the fixed alias types):

1. `gorm.Open` against the real dev DB using the same `.env` DSN the running container uses.
2. Real `postgres.NewMonthlyProductionDataRepository` + `NewWellStringReservoirLocationRepository`
   + `monthlyproductiondatasvc.NewService(repo, locationRepo)` -- not fakes.
3. `svc.LoadBubbleMapSnapshotData` for the exact scope/anchor month from the bug report.
4. Real `plotapi.New(...)` + `bubblemapinsightssvc.NewPlotAnalysisAdapter(client)` against either
   the running dev container (if it already has the fix) or a freshly started local
   `uvicorn api.main:app --port 8010` (since the dev-facing plot-api container has no bind mount
   and silently keeps running pre-fix code until explicitly rebuilt).
5. Delete the throwaway test file after use -- it is a diagnostic, not permanent coverage.
6. After a code fix ships, always ask: does this change the SHAPE of anything cached (units map,
   record fields, response contract)? If yes, flush or version-bump the cache before declaring
   victory -- a correct fix behind a stale cache looks identical to a broken fix.
