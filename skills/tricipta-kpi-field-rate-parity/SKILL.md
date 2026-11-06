---
name: tricipta-kpi-field-rate-parity
description: "Diagnose or build Tricipta field-level KPI/oil figures (Field KPI Summary, VPS Field Rate Estimation card, Current/Optimised/Uplift) so the numbers match the VPS dashboard — field meter vs VFM sources, the hardcoded ANSI last-timestamp anchor, signed uplift, and the well-string suffix-code trap. Use when a Tricipta card's oil totals disagree with another card, when Current reads 0, when wells appear mixed up, or before changing any oil-rate source in asset-api/asset-web."
---

# Tricipta field-rate parity (Current / Optimised / Uplift)

Use when an oil figure on one Tricipta card disagrees with another, reads 0, or when changing where "actual oil" comes from.

## 1. Know which source holds what

| Figure | Table | Reached through |
|---|---|---|
| Current (field flow meter) | `ast_field_sensor_readings`, `<FIELD>.Oil Rate`, `data_type='NONE'` | `vfmcalculatedsvc.GetFieldSummary` → `FindRTFieldRates` (`vfm_calculated_repository.go:~509-592`) — median over the window |
| Optimised (field VFM) | `ast_vfm_calculated` | same call, `.oil` |
| Per-well status / gas lift | `ast_vps_calculated` (`source='field_derived'` for obs) | direct |
| Per-well last non-zero VFM (SI reactivation) | `ast_vfm_calculated` | direct, `COALESCE(multi_layered, field_derived)` |

**There is no per-well actual oil.** `ast_well_sensor_readings` carries a per-string `<well>-<string>.Oil Rate` tag for only ~1 string per field (ANSI: 1 of 193). Any per-well Current/Gap design is unimplementable — say so before building it.

## 2. Match the card, don't re-derive it

`GetFieldSummary` behaves differently with and without `from_time`:

- no `from_time` → daily median from 00:00 (MYT) of the data's own date;
- `from_time` present → median over `from_time..as_of`.

The VPS dashboard's default "Last 24 Hours" filter sends both, so call it the same way:

```
GET /v1/vfm-calculated/field-summary?field_code=ANSI&source=field_derived
    &from_time=<anchor−24h>&as_of=<anchor>
```

`anchor` = `vfm_last ?? vps_last ?? last` from `GET /v1/vps-dashboard/last-timestamp` — **not** field-summary's own `as_of`. ANSI is hardcoded there (`vpsdashboardservice/service.go:~60-67`, "temporary") to `2026-06-28T23:59:59Z`; using `as_of` instead produced 12,778 vs the card's 12,983. To make the whole page one date, also bound every per-well read with `timestamp <= anchor` and use that anchor as the card's as-of.

Expected live values for ANSI on that anchor: Current 12,983, VFM 7,769, Uplift −5,214 (−40.2%), Reactivation 708, Gas Lift 14.59 MMscf/d.

## 3. Formula (story-literal, signed)

```
current    = round(field meter)      // 0 when absent
optimised  = round(field VFM)        // 0 when absent
uplift     = optimised − current     // only when BOTH exist; may be negative
centre     = max(uplift, 0) + reactivation   // negative uplift is a tile value, not a slice
reactivation == Σ priorities.gain
optimised − current == uplift        // only when both exist
```

A negative uplift must render signed (`-5,214`, no `+` prefix) and in rose, matching the VPS card's red gap.

## 4. The well-string suffix trap

`ast_well_strings.code` is only `LS` / `SS` / `TS` and repeats across wells (ANSI: 103 strings named `LS`). Merging VPS/VFM/sensor rows by code mixes wells and copies one reading onto all of them. Merge on `ast_well_strings.id`:

- VPS/VFM join on `ast_well_strings.well_string_id` (integer surrogate);
- sensor readings join on `ast_well_strings.id` (uuid) — `ast_well_sensor_readings.well_string_id` is a uuid FK here;
- `ast_well_sensor_readings.well_string_code` already holds the full `<well>-<string>`, so it is a label, never a join key;
- display label = `<well name>-<string code>`.

## 5. Query shape (perf)

`ast_well_sensor_readings` is huge (dev: ~600M rows). Filtering by tag code scans a string's whole history; per-well-string `CROSS JOIN LATERAL (… ORDER BY time DESC LIMIT 1)` on `(well_string_id, sensor_tag_id, time DESC)` / `(well_string_id, time DESC)` is index-backed. Drive sensor reads off the well string's oil-rate **tag ids** (`ast_sensor_tags.name LIKE '%.Oil Rate'`), not tag codes.

Status/VFM reads as `LATERAL … LIMIT 1` on `(well_string_id, timestamp DESC)`: 17 s → 1.7 s and 14.6 s → 0.8 s.

## 6. Verifying

```bash
TOKEN=$(curl -s -X POST localhost:8085/api/v1/auth/login -H 'Content-Type: application/json' \
  -d '{"username":"testuser","password":"testpass"}' | jq -r .token)   # max 2 attempts (lockout after 5)
curl -s -H "Authorization: Bearer $TOKEN" localhost:8085/api/v1/insight/kpi-summary/ANSI | jq .data
```

Assert `current == card Flow Meter`, `optimised == card VFM`, `reactivation == Σ priorities.gain`, `uplift + reactivation == donut.centre` (with uplift floored). Cross-check in the browser on `:3000/common-platform/{dashboard,vps-dashboard}`; compare to the same session's VPS card text.

Read the dev DB read-only (`PGOPTIONS='-c default_transaction_read_only=on'`) and keep the password out of argv — `docker exec` a `PGPASSFILE` written via stdin, then delete it.
