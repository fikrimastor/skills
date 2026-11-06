---
name: tricipta-plot-insight-value-reconciliation
description: "Diagnose/fix a Tricipta AI-insight value (Latest card, change, trend) that disagrees with the displayed diagnostic-plot chart or Production Data table — covers multi-reservoir/multi-string aggregation basis, alias-duplicate rows, NULL-scaffold rows, and anomaly-filter latest bugs."
---

# Tricipta plot-insight value reconciliation

Use when an AI-insight number (usually the **Latest** card) does not match what the user sees on the chart or in the table for the same well/date.

## 0. Ground in REAL data first — do not theorise

Synthetic fixtures confirm whatever you already believe; staging data disproves it. Every root cause in this class was found by querying staging.

```bash
# Local dev DB is EMPTY. Query staging via the running container (creds in tricipta-edafy-asset-api/.env).
docker exec edafy-db env PGPASSWORD='<from-.env>' PGSSLMODE=require \
  psql -h psql-production-ai-nonprod.postgres.database.azure.com -p 5432 \
  -U pgadmin -d dev_asset_db -c "<read-only SQL>"
```

Table: `ast_production_data` (note: `monthly_production_data` does not exist). Key columns: `well_code`, `well_string_code`, `reservoir_code`, `time`, `oil_production_day_rate` (`oil_pd_rate`), `oil_calendar_day_rate`, `oil_volume`, `water_*`, `gas_*`.

**Always establish, in this order:**
1. Which rows exist for the well/date in question, and their values.
2. Whether those rows are **identical** (alias duplicates) or **distinct** (genuinely separate intervals).
3. The **last month with non-NULL values** (not the last month with rows).

Ask the user which number is authoritative BEFORE changing shared engine code — chart-vs-engine is a product decision, and the criteria doc can mandate the opposite of the chart.

## 1. Classify the mismatch

| Symptom | Cause |
|---|---|
| Card value equals ONE reservoir (often the lexicographically-last, e.g. `B2.0`), chart shows the sum | Engine flattens entities and reports a positional row instead of aggregating |
| Chart total ≈ 2× (or N×) the true rate | Chart sums ALIAS duplicate rows the engine correctly dedupes |
| Card reports a recent month the user cannot find on the chart | Rows exist but are entirely NULL (scaffold); or latest taken from the anomaly-filtered endpoint |
| Card date is older than the newest month that has data | Latest resolved on the wrong series/entity ordering |

## 2. Aggregation contract (mirrors `app/utils/diagnosticPlotReservoirAggregate.ts`)

Collapse rows to ONE per `(time_key, entity)` where entity = `_entity_column(frame, profile)`:

- **Additive** (`*_rate`, `*_cumulative`, `*_volume`, plus `np/gp/wp/wi/gi`): SUM with skip-null (`min_count=1`) — null only when every contributing value is null.
- **Day counters** (`producing_days`, `active_days`, `calendar_days`): **MAX**, never sum — concurrent reservoir rows are partitions of one completion's clock.
- **Ratios** (`water_cut`, `wor`, `gor`): **recompute** from the summed components; never sum, never carry. `water_cut` uses the chart's fallback order: PD components → volume components → null.
- No-op when every group already has exactly one row.

## 3. Pitfalls that WILL reintroduce the bug

- **Ratio columns must be materialised** before being overwritten. If you exclude them from `sum_columns`/`max_columns` without adding them back, they vanish from the aggregated frame, and `frame[metric]` in the extractors raises `KeyError` → HTTP 502. Set them to null first, then overwrite. (The water_cut profile is `any_of[oil_pd_rate, oil_cd_rate]`, so components can legitimately be absent.)
- **Never carry a ratio via `groupby(...).first()`** — that silently substitutes one arbitrary entity's value (the exact bug being fixed).
- **`latest` must come from the raw (non-anomaly-filtered) series.** In a long decline the newest low reading trips the modified z-score and, if filtered, is replaced by an older higher month.
- **Order feature endpoints chronologically** via `_sort_for_latest_point` ([time, group, entity]). `_sort_series_groups` ([group, entity, time]) puts entity before time, so positional `.iloc[-1]` picks a lexicographic string block.

## 4. Tests must discriminate

A test that passes on the buggy code is worse than no test. Require:

- **Distinct** reservoir values (e.g. `70/30` and `50/50` → pooled `0.4`). Identical values let the alias gate collapse the frame, so aggregation never runs and a first-row carry passes unnoticed.
- Expected values that differ from BOTH the first row's raw value and the arithmetic sum.
- Prove discrimination by temporarily reverting the fix and capturing the failing assertion (`assert 0.3 == 0.4`, `KeyError: 'water_cut'`), then restore.

## 5. Verify

```bash
cd tricipta-diagnostic-plot-api && ./venv/bin/python -m pytest tests/ -q   # bare `python` is NOT on PATH
```
Then end-to-end on the user's real numbers (feed the actual staging rows through `extract_time_series`) and confirm the card value equals the chart value.

## 6. Known open issue

Alias duplicate pairs are duplicated **in the source** (ingest), so the chart and ranking still double-count them even after the engine dedupes. A chart-only dedupe leaves ranking/scatters wrong; the root fix is canonicalizing one code per alias pair at ingest.
