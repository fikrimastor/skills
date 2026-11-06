---
name: tricipta-diagnostic-plots-axis-table-bugs
description: "When a Diagnostic Plots axis unit/label or Production Data Table columns disagree with spec in asset-web."
---

# Diagnostic Plots: axis unit/label and Production Data Table bugs (tricipta-edafy-asset-web)

Three distinct, unrelated bug classes live in this one feature. Diagnose which one first. They
have already produced issues #590, #592, #594, #596, #664 (all fixed with this playbook) plus
issue-#203 table-column gaps (Cumulative Liquid, WOR). Expect more.

## Where the code lives — check all four files together

1. `app/utils/productionDataToPlotEngine.ts` — `PRODUCTION_METRIC_UNITS` (metric key → unit label
   string) and `transformProductionRowToPlotRecord` (row → chart record; applies an optional scale
   factor via `setMetric(record, key, value, scale)` / `setSumMetric(...)`, default scale 1).
2. `app/utils/productionDataValueFormat.ts` — `PRODUCTION_TABLE_COLUMN_FORMAT` / `PLOT_METRIC_FORMAT`
   and the `formatProductionData*` helpers.
3. `app/utils/productionDataTableColumns.ts` — `PRODUCTION_DATA_COLUMNS`, the master column registry
   (key, label, `format`, `exportValue`, `sortValue`). The on-screen table AND the CSV export
   (`productionDataExportColumns.ts` / `buildProductionExportColumns`) both read this one array.
4. `app/utils/productionDataTableColumnsByCapability.ts` — `CAPABILITY_TABLE_COLUMN_KEYS`
   (capability name → which `PRODUCTION_DATA_COLUMNS` keys to show, in order) and
   `BUBBLE_MAP_CAPABILITY_TO_DIAG_NAME` (Bubble Map capability title → the same capability names).

Charts render through an EXTERNAL service: `server/api/diagnostic-plots/render.post.ts` proxies to
`/v1/plots/render`. That service is NOT in this repo and cannot be edited. It has two independent
behaviours, both provable from `productionDataToPlotEngine.ts` git history (commits `6509e39`,
`844d7f5`):

1. It **echoes the frontend's `units` map verbatim** onto `neutral_json.y_axis.unit`. The map is
   built by `getUnitsForProductionMetrics` from `PRODUCTION_METRIC_UNITS`, keyed by the plot-engine
   record field name (`oil_cumulative`, `liquid_cumulative`, `wor`, `gor`, …) — NOT by the
   capability's dotted name. An entry with a falsy value is skipped (`if (unit)` guard), which
   suppresses the parenthetical unit everywhere (axis title and tooltip).
2. It **independently supplies a human-authored `y_axis.title`** for some metrics, more verbose than
   the product's short form ("Gas-Oil Ratio" vs "GOR", "Water-Oil Ratio" vs "WOR"). Nothing the
   frontend sends influences it — `PlotRenderRequestPayload` has no `title` field.

## Bug class 1 — wrong UNIT (and/or scaled magnitude) on axis/tooltip

Examples: #590 "Cumulative Gas (scf)" should be "(Bscf)"; #592 "Cumulative Liquid (stb)" should be
"(MMbbl)"; #664 tooltip vs table mismatch, e.g. axis showing `1,439,715 stb` where the table's
column shows `1.439715 MMbbl`.

Root cause: the API's raw row fields (`row.np`, `wp`, `wi`, `gp`, `gi`, …) are **already in display
units**, matching `productionDataTableColumns.ts` labels (`"Gp (Bscf)"`, `"Gi (Bscf)"`,
`"Np"`/`"Wp"` in MMbbl). The table prints `row.field` unscaled. A stale `scale` argument
(`MM_TO_BASE = 1_000_000`, `BSCF_TO_SCF = 1_000_000_000`) multiplies the value up to base units
(stb/scf) while `PRODUCTION_METRIC_UNITS` declares that base unit — so chart and table disagree by
exactly the scale factor.

Diagnose:
1. Identify the metric key (e.g. `gas_cumulative`) from the capability shown in the UI.
2. Find its `setMetric`/`setSumMetric` call in `transformProductionRowToPlotRecord`; note the scale
   argument and the row field read.
3. Find that row field's column in `productionDataTableColumns.ts` and read its `label` — that is
   ground truth (the table never scales).
4. Find `PRODUCTION_METRIC_UNITS[key]`. Scaled value + declared unit that don't reproduce the
   table's number and unit = this bug.
5. Cross-check other consumers of the same field (e.g. `bubbleMapDashboardData.ts` unit
   declarations) — they are usually already correct and confirm which unit is real.

Fix:
1. Drop the scale argument (`setMetric(record, key, row.field)`).
2. Set `PRODUCTION_METRIC_UNITS[key]` to the table's real unit string (`"MMbbl"`, `"Bscf"`).
3. Delete a scale constant that becomes unused — grep the whole file for other callers first.
4. Check for a **sibling metric sharing the same scale constant**: `water_injection_cumulative`
   sits next to the `*_cumulative` oil/water/liquid trio; `gas_injection_cumulative` next to
   `gas_cumulative`. They are separate selectable capabilities with the identical latent bug. Ask
   whether to bundle the sibling fix instead of silently expanding or ignoring scope.
5. Dimensionless metric that should show NO unit (e.g. WOR): **delete the
   `PRODUCTION_METRIC_UNITS` entry entirely** (not an empty string — same rendering effect via the
   `if (unit)` guard, but explicit).

NOT this bug — leave alone:
- `gas_volume`: `row.gasVol` (periodic, MMscf) × `MM_TO_BASE` → `"scf"` is correct; the table has a
  separate `"Gas Vol (MMscf)"` periodic column. `MM_TO_BASE` stays in use for it.
- Rate metrics (`*_rate`, `*_cd_rate`, `*_pd_rate`), `oil_volume`, `water_volume` — already unscaled
  and correct; don't add scaling.

Precedent: commits `6509e39` (oil/water/liquid/water-injection cumulative → MMbbl, i.e. #664/#592
and #590's siblings) and `844d7f5` (gas/gas-injection cumulative → Bscf, #590; `BSCF_TO_SCF`
deleted as fully unused).

## Bug class 2 — wrong axis TITLE wording (independent of units)

Example: #594 "Gas-Oil Ratio" should be "GOR".

`app/utils/diagnosticPlotAxisLabels.ts`'s `resolveDiagnosticPlotAxisTitle` deliberately passes
through any title that isn't raw snake_case (comment: "anything else is already a human-authored
label"), so the backend's verbose labels reach the UI unchanged. A test literally named
`"leaves engine-authored labels untouched"` encodes that trust.

Fix: add an entry to the single `AXIS_TITLE_ALIASES: Record<string,string>` map near the top of
`diagnosticPlotAxisLabels.ts` — key = the backend title normalized by `normalizeToken` (lowercase,
hyphens/spaces → underscores: `"Gas-Oil Ratio"` → `"gas_oil_ratio"`), value = the desired short
label (`"GOR"`). History: GOR originally landed as a per-metric token `Set` +
`GOR_AXIS_ALIAS_TOKENS`/`GOR_AXIS_LABEL` short-circuit mirroring `PERIOD_AXIS_TOKENS` and
`CD_PD_RATE_AXIS_LABEL`; adding WOR refactored it into the shared map. Do NOT reintroduce a second
parallel per-metric constant pair — one map is the convention. Export the label constant like its
siblings.

`test/utils/diagnosticPlotAxisLabels.test.ts`'s `"leaves engine-authored labels untouched"` case may
assert the OLD (buggy) passthrough for the exact string you are now overriding. That assertion
embodies the bug: move it into a new dedicated test documenting the override, keep the still-correct
cases in place, and add an end-to-end `formatDiagnosticPlotAxisTitle(rawTitle, unit)` test with the
exact reported string.

## Bug class 3 — wrong / missing Production Data Table columns

Issue #203 ("Rates and Productions Volumes") specifies exact columns per plot, e.g.:

```
Cumulative Liquid | No., Well String, Reservoir, Period, Oil Production (MMbbl), Water Production (MMbbl), Liquid Cumulative (MMbbl)
GOR               | No., Well String, Reservoir, Period, Oil PD (bbl/d), Gas PD (MMscf/d), GOR (scf/bbl)
WOR               | No., Well String, Reservoir, Period, Oil PD (bbl/d), Water PD (bbl/d), WOR
```

`CAPABILITY_TABLE_COLUMN_KEYS[capabilityName]` is a hand-maintained array of
`PRODUCTION_DATA_COLUMNS` keys and it is BOTH the column set and the display order —
`columnsFromCapabilityKeyOrder` sorts strictly by array index, not by `PRODUCTION_DATA_COLUMNS`'
declaration order. Nothing stops an entry listing a sibling capability's fields.

Two failure modes found:
1. **Wrong/borrowed keys** — Cumulative Liquid had `["np","wp","oilVol","waterVol"]` (`oilVol`/
   `waterVol` belong to Cumulative Oil/Water); GOR had `["gor","oilPD","gasCD"]` — wrong order AND
   `gasCD` instead of `gasPD`.
2. **Missing column entirely** for a plot whose Y-axis value is CLIENT-COMPUTED from two other
   fields rather than a raw `DiagnosticPlotRow` field (check the `DiagnosticPlotRow` interface in
   `app/utils/diagnosticPlotsData.ts` — if the metric isn't there, no column key exists). Found for
   Cumulative Liquid (`Np + Wp`) and WOR (`WaterPD / OilPD`); GOR was fine because `gor` is a real
   DB-backed field.

Fix a missing derived column in `app/utils/productionDataTableColumns.ts`:
1. Add a small LOCAL helper that mirrors the EXACT computation the plot engine does, so table and
   chart never disagree — copy `setSumMetric`'s NULL propagation for sums
   (`productionDataToPlotEngine.ts`), or `computeWor`'s NULL propagation + zero guard
   (`if (oilRate <= 0) return 0`) for ratios. Do not import from `productionDataToPlotEngine.ts`;
   replicate the branches locally (existing convention: `sumCumulativeLiquid`, `computeWorRatio`).
   Either input null → whole result null → table shows `NULL_METRIC_DISPLAY`, export `""`.
2. Add the column object to `PRODUCTION_DATA_COLUMNS` with a synthetic `key` that is not a real row
   field. `format`/`exportValue`/`sortValue` all receive `(value, row)` (see `cpTableTypes.ts`'s
   `CPTableColumn`) — compute from `row` and ignore the undefined `value`:
   `format: (_v, row) => formatExact(helper(row.a, row.b))`, `exportValue: (row) => formatExport(helper(row.a, row.b))`,
   `sortValue: (row) => helper(row.a, row.b) ?? 0`.
3. **Formatting convention:** as of issue #659 (merged 2026-08-18) every numeric column except `wc`
   (percent) and `xCoordinate`/`yCoordinate` (locale, fixed digits) uses exact, shortest-round-trip
   formatting — `formatProductionDataExact as formatExact` / `formatProductionDataExport as formatExport`
   with **no digits argument** (`= Number.prototype.toString()`). `formatProductionDataDecimal(value, N)`
   / `.toFixed(N)` / `formatProductionDataLocale` is the OLD, now-wrong style — don't reintroduce it.
   Mirror the plot-engine equivalent (`PLOT_METRIC_FORMAT`) so tooltip and table precision agree.
4. Label: use the issue's literal spec wording for genuinely new columns ("Liquid Cumulative
   (MMbbl)", not "Lp" — "Lp" exists only inside a capability id, never shown to users). Omit the
   unit suffix for dimensionless values (WOR label is just `"WOR"`, matching `Cum. GI/GP` /
   `Inst. GI/GP` style).
5. Fix the `CAPABILITY_TABLE_COLUMN_KEYS` entry's key set AND order. Leave every other entry alone,
   including superficially similar families outside Rates & Production
   (`chan_diagnostics.wor_versus_gor`, `injection_diagnostics.*wor*`).
6. Free bonus: `"<X> Bubble Map"` capabilities resolve through `BUBBLE_MAP_CAPABILITY_TO_DIAG_NAME`
   to the same `well_production.<x>` entry, so one fix covers the Bubble Map table too. Add one
   assertion via `getTableColumnKeysForBubbleMapCapability("<X> Bubble Map")` — don't duplicate the
   fix.

## Tests and verification

Existing test files (read the live file first — never assume you are creating it fresh):
`test/utils/productionDataToPlotEngine.test.ts`, `diagnosticPlotAxisLabels.test.ts`,
`productionDataTableColumns.test.ts`, `productionDataTableColumnsByCapability.test.ts`.

- `productionDataToPlotEngine.test.ts` uses a `makeRow()` helper with all `DiagnosticPlotRow` fields
  nulled. Add one `describe("... (issue NNN)", ...)` block per fix asserting BOTH the unscaled record
  value and the unit from `getUnitsForProductionMetrics(getAvailableProductionMetrics([record]))`.
  Remove any test that locked in the old buggy scaled behaviour as "intentionally untouched".
- `productionDataTableColumnsByCapability.test.ts`: `getTableColumnKeysForCapability` /
  `getTableColumnsForCapability` — the function the Rates & Production tab actually calls
  (`diagnostic-plots-dashboard.vue`'s `productionTableColumns` computed) — had **zero** coverage
  before the 2026-08-18 audit; only the Bubble Map variant was tested. Assert key order AND resolved
  labels against the #203 spec row, plus a negative assertion that the old key is gone. Reuse an
  existing `makeCapability(...)` helper if present; redeclaring it is a duplicate-identifier TS error.
- `npm run test -- <substring>` is a vitest filename substring filter, so
  `npm run test -- productionDataTableColumns` ALSO runs `...ColumnsByCapability.test.ts`.

```bash
npm run test -- productionDataTableColumns productionDataTableColumnsByCapability productionDataToPlotEngine
npx tsc --noEmit          # zero output = pass
```
This repo has no `.vue` component tests (established convention) — pure-function unit tests in
`test/utils/` are the bar.

## Workflow gotchas

- `origin/main` moves fast here. `git fetch origin main` and re-read live files immediately before
  dispatching an implementation subagent, even if you read them earlier in the same session:
  issue #659's exact-value formatting refactor landed mid-session and silently invalidated an
  already-drafted plan. Don't trust an earlier read across a branch switch.
- Two fixes touching disjoint files (axis-label vs table-column mapping never share a file) are a
  genuine parallel `task` batch, not serial work.
- When a second subagent appends to a test file a previous subagent already extended, tell it to
  reuse the existing helpers/imports rather than redeclare them.
- Independently re-run the scoped tests and `npx tsc --noEmit` yourself after a subagent reports
  success — don't trust the reported diff or pass count.
- If 2+ subagents commit on the SAME branch/working tree (not separate worktrees), each must
  `git show --stat HEAD` right after its own commit to confirm it only committed its own files —
  concurrent `git commit` calls in a shared index sweep in a sibling's staged files. Recovery:
  `git reset --soft HEAD~1`, `git reset HEAD <sibling's paths>`, recommit only your own files, and
  message the sibling over `hub` so it doesn't also try to fix the race.
- Branch naming: `fix/<issue-number>-<short-desc>-web`. Non-trivial plans live in
  `docs/superpowers/plans/YYYY-MM-DD-<feature>.md` (writing-plans convention already used here).
- To reword a non-HEAD commit message without an interactive editor: `git reset --hard <target-sha>`
  → `git commit --amend -m "..."` → `git cherry-pick` the follow-up commits back on top (diffs are
  byte-identical so they apply cleanly). Only relabel a commit whose CONTENT is about that issue.

## Confirmed audit status (2026-08-18) for #203's 7 Rates & Production plots

- Fixed: `well_production.cumulative_liquid` (added the `liquidCumulative` column);
  `well_production.gor` (`gasCD`→`gasPD`, reordered) + axis title (#594).
- Audited, already correct: `well_production.cumulative_oil` (`["np","oilVol"]`),
  `cumulative_water` (`["wp","waterVol"]`), `cumulative_gas` (`["gp","gasVol"]`, `Bscf` confirmed
  correct by #590 superseding the spec text's `MMscf`). None of the three was checked for a
  #594-style title bug.
- Not yet audited: `well_production.water_cut`, `well_production.wor`.

## Consumers (fixed automatically — no direct edits)

- `app/pages/common-platform/diagnostic-plots-dashboard.vue` (Rates & Production page)
- `app/composables/useAstProductionData.ts` → `bubble-maps-dashboard.vue`
- `app/utils/diagnosticPlotRequestBuilder.ts`
