---
name: chart-table-ranking-parity
description: "When a ranking/bar chart's order, labels or tooltip disagree with the table/CSV, or a horizontal bar shows 0,1,2…"
---

# Server-ranked chart vs client-built table

## Symptom family
- Chart is sorted descending (ranker lives in the backend engine) but the table keeps API order, so zero/empty rows appear first.
- Table row numbers (`index + 1`) reflect API order, not rank order.
- Chart draws every entity (hundreds of bars) while the table looks fine, or vice versa.
- Chart and CSV disagree about which rows and in what order.

## Diagnose (in order)
1. Find the surface that ORDERS: grep the render/analysis request for the engine's ranking option (`top_n`, `rank_entities`, `selection_strategy`). If the sort happens server-side, assume the client table is NOT sorted.
2. Find the surfaces that are BUILT CLIENT-SIDE: the table-row mapper, the chart-record mapper, the CSV/export column builder. They usually take the same fetched array and disagree only because nobody sorted it.
3. Confirm whether a render-side limit can fix it. It usually cannot: the limit affects the chart's data, not a table built from the raw fetched rows. Any client-side surface needs its own cap.

## Fix (the single-list rule)
Compute ONE canonical list on the client and drive every surface from it:
- the ranking metric, converted to the display unit;
- drop null / undefined / non-finite / zero (non-producing) rows;
- sort descending by the metric, with a deterministic tie-break on a stable identity (entity/string code);
- cap to the display limit (export the constant, e.g. `FIELD_RANKING_TOP_N = 20`);
- return the ordered list PLUS both the eligible count (before truncation) and the fetched candidate count, so the UI can render `Showing top 20 of 223 eligible …`.

Then feed that list to: chart records, table rows, row numbering, footer count, and CSV export. Also send the same limit to the render request when the engine supports it, so the chart cannot draw every entity.

## Verify
- A test that fails before the fix for each surface: chart order, table order/numbering, cap, footer numbers, export rows.
- Adversarial check: reverse the input order and assert the output order is unchanged; seed a null and a zero row and assert both are dropped; seed a tie and assert the tie-break is stable.
- Assert eligible-count and shown-count separately (they differ by construction).

## Chart.js horizontal-bar tooltip trap
When a bar chart sets `indexAxis: "y"`:
- `context.parsed.y` = the CATEGORY INDEX (0, 1, 2…), not the value;
- `context.parsed.x` = the real value.

A shared tooltip helper written for vertical charts reads `parsed.y` and will show 0,1,2… on every horizontal bar. Fix by branching on the chart type (`type === "bar"` / horizontal axis) and formatting `parsed.x` with the X-axis unit and title. Check every tooltip path: the shared label callback AND any hand-rolled multi-point/dual-series tooltip loop, which usually reads `parsed.y` too. Pin it with a component test that asserts the real value appears (it would show the index if regressed).

## Do not
- Do not fix the mismatch by changing the backend's ordering or adding a filter param to a frozen API contract — make the client authoritative for what it displays and exports.
- Do not change a shared record mapper's identity field globally to fix one chart type; scope the change to the capability family that needs it (e.g. only `*_ranking.*`) so time-series paths and legend/toggle identity are untouched.
- Do not apply ranking order/filter/limit to a scatter or exceedance capability that merely groups by entity.
