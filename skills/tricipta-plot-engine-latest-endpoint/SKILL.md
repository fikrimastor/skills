---
name: tricipta-plot-engine-latest-endpoint
description: "Diagnose and fix a Tricipta plot-engine defect where features.latest (the \"Latest\" card value) disagrees with the change/trend endpoints or reports a stale value as current — the entity-before-time sort trap in _sort_series_groups. Use when an AI Insight or diagnostic-plot value looks wrong for a multi-string (tubing_string=ALL) well."
---

# Tricipta plot-engine `latest` endpoint defects

## Symptom triggers
- AI Insight "Latest" card value disagrees with the same insight's headline / Period Δ.
- A stale value (e.g. a 2018 rate) is presented as the current/terminal state.
- Value looks plausible for one tubing string but wrong for the well.
- Any wrong-looking value on a well selected with `tubing_string=ALL`.

## Repo
`~/Sites/afed/tricipta-diagnostic-plot-api`, file `plotting_core/analysis/extractors.py`.

## The trap
`_sort_series_groups` sorts by `[group, entity, time]` — **entity before time**. When
multiple tubing strings (LS/SS/TS) share one frame, rows arrive as contiguous
per-entity blocks, so any positional `.iloc[-1]` resolves to the
lexicographically-last **string's** terminal row, not the newest date.

`_time_series_features` derives `features.latest` positionally, so it inherits
this — producing an insight that contradicts itself (e.g. Latest `1.097 bbl/d`
from SS Jun-2018 while change says `388 -> 23 bbl/d` from LS).

The chronological helper already exists: `_sort_for_latest_point`, which sorts
`[time, group, entity]`. It was originally used only by the scatter path.

## Correct fix
Make the time-series/cumulative preparation sort chronologically:

- In `_prepare_series_frame`, use `_sort_for_latest_point` instead of
  `_sort_series_groups`.
- Apply the same to the cumulative extractor's explicit sort.
- Leave `_sort_series_groups` itself alone — ranking/tie ordering still uses it.
- Do **not** touch `_detect_anomalies` semantics, scatter selection (already
  chronological), or ranking (intentionally top-ranked, not time-ordered).

Per-group row sequencing is unchanged by a time-major sort, so
`_all_series_monotonic`, `_detect_shared_ramp`, and the alias / CD-PD gates
(which groupby per entity) keep working — but verify them anyway.

## Wrong fix to reject on sight
"Make `latest` share the anomaly-filtered endpoint used by change/trend"
(i.e. switch `clean.iloc[-1]` to `filtered.iloc[-1]`). This only *looks* fixed:
its fixture passes because the other entity's values happen to be anomaly-flagged
and get dropped, and it still returns the stale row when that entity's values are
ordinary.

## Verification recipe (do this, don't just read the code)
Write a throwaway probe that exercises the real path and print the sorted tail:

```python
import sys; sys.path.insert(0, ".")
import pandas as pd
from plotting_core.analysis.extractors import _sort_for_latest_point
from plotting_core.analysis.profiles import get_analysis_profile

prof = get_analysis_profile("well_production.oil_rate")
ls_dates = pd.date_range("2018-01-01", periods=8, freq="YS")
frame = pd.DataFrame({
    "date": ls_dates.tolist() + [pd.Timestamp("2018-06-01"), pd.Timestamp("2018-07-01")],
    "well": ["A013"] * 10,
    "string": ["LS"] * 8 + ["SS"] * 2,
    "oil_rate": [388., 350., 300., 250., 200., 150., 100., 23., 20., 25.],
})
o = _sort_for_latest_point(frame, prof)
print(o.tail(3)[["date", "string", "oil_rate"]])
```

Run it with `./venv/bin/python` (there is no bare `python` on this machine) and
assert the terminal row is the **newest date** (`2025-01-01 LS 23.0`), not the
last entity block. Test **both** variants — other-entity values that are
anomalies (`[1000, 900]`) and ordinary ones (`[20, 25]`). The ordinary variant is
the one that catches the wrong fix; the anomaly variant passes either way.

## Regression gate
`./venv/bin/python -m pytest tests/ -q` — expect all green (238 at time of
writing). The shared extractor feeds all 46 capabilities, so a full run is the
real gate, not just the touched test file.

## Related
- The deterministic recency signal lives in asset-api: `computeDataRecency`
  (`diagnosticplotinsightssvc`) derives `{latest_data_date, window_end,
  months_stale, stale}` from `features.latest.date` vs the user's window end and
  ships it in `meta.data_recency`. Prefer that deterministic signal over prompt
  instructions when the goal is "never present stale data as current".
