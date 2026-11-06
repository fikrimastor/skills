---
name: tricipta-bubble-map-capability-binding
description: "Bind a Tricipta bubble-map (well_maps.*) capability's real UI render into the bubble-map analysis-criteria doc, one capability per screenshot — the spatial counterpart to diagnostic-plot-capability-binding. Use when Kanda attaches a Bubble Map screenshot and wants the criteria/data-binding enhanced for the #683 AI-insight prompt work."
---

# Tricipta Bubble Map Capability Binding Loop

Enhance the bubble-map analysis-criteria doc by binding one capability's ACTUAL rendered UI (from a screenshot) into it. One capability per turn. Spatial counterpart to the `diagnostic-plot-capability-binding` skill.

## Source doc (source of truth)
`~/Sites/afed/tricipta/docs/bubble-map-capabilities-analysis-criteria.md` — 18 UI maps across 5 families. Companion (later): `bubble-map-analysis-agent-prompts.md` (mirror; drift = bug).

## Scope facts (don't re-derive)
- Engine registry has 25 `well_maps.*`; the dashboard surfaces 18 (`asset-web app/utils/bubbleMapDashboardData.ts` → BUBBLE_CATEGORIES + MAP_METRICS). Wired value keys in `bubbleMapEngineCapabilities.ts`.
- Every map = spatial snapshot at an anchor month; one bubble per (WELL STRING, RESERVOIR). Table is string-level snapshot for ALL families (`isStringLevelTable` true for all 5 categories; the monthly-log branch is dead).

## Read from EACH screenshot (bind these verbatim)
1. Capability title + family; anchor month (playhead); header `Σ <total> <unit>` and `N strings` count.
2. Colorbar: unit, domain min→max, ticks, colormap (Oil Rate & Cumulative Oil = plasma; everything else = viridis). Colorbar domain is the metric's OVERALL data max (fixed across months), NOT the anchor-month max.
3. Table exact header casing `NO. | WELL STRING | PLATFORM | RESERVOIR | PERIOD | <METRIC> (<UNIT>)`; sort dir; a few desc-top rows with real values.
4. Tooltip (if shown): label · platform·reservoir · WGS84 4dp · Easting/Northing abs metres · Period · metric value (String Location's last row is Producer/Injector ROLE, not a metric).
5. Whether MAP (tooltip/colorbar) == TABLE for the same well.

## Known traps to check every time
- **Marker maps (Lp/Tp, Np/Tp):** table column is ~140–200× the map/colorbar — OPEN BUG (marker-table value builder). Trust the map/colorbar.
- **GOR/WOR:** blow up as oil→0 (observed ANSI-A022R01-LS: GOR 2.59B, WOR 127) — map shows blow-up + hijacks colorbar/Σ; table capped/saner. Trust table, exclude blow-ups.
- **Cumulatives:** map↔table divergence is a date-WINDOW artifact — agrees on full range; on narrow window prefer map/Σ.
- **Ratio Σ (WC/GOR/WOR):** naive sum — meaningless; never cite as field ratio. **Rate/cumulative Σ:** valid field total (Cum Liquid Σ = Σoil+Σwater).
- **All-zero month:** latest (2026-09) is a rate data-lag → rate maps Σ0/all-zero (gate: report gap, step playhead to 2024-09/2025-04/2025-10). Life-to-date markers & cumulatives stay populated (immune).
- **Gas:** UI is MMscf/d (÷1e6 already applied; trap only on raw engine scf/d); gas RATE has a negative colorbar floor (allocation artifacts → exclude); cumulative gas is Bscf.
- **Injection maps:** injector-mode; producers render hollow + table `—` (tooltip may show 0). `N strings` = injector count (0 when none). Oil Injection (rate & cumulative) = NO source → honest empty state (0 strings, all `—`), NOT mock. Water/Gas injection empty is month-specific/field-specific, not permanent.

## Edit procedure (per capability, in the criteria doc)
1. Rebind the `Data binding` bullets: bubble schema (metric, engine cap, unit, mode, colormap, observed colorbar domain+ticks, Σ, anchor month), exact table header + observed rows, exact tooltip, traps, gaps.
2. Audit the `Analysis criteria` when asked ("check/audit criteria"): add gates (all-zero for rates, no-injectors for injection, data-availability for ratios, exclude-blow-ups for GOR/WOR), qualify directional claims as planform-only (§0.6), and add any analysis the map uniquely enables.
3. Promote genuinely global findings to §0 (colorbar-fixed, negatives=artifact, map chrome/scope, tooltip format) and cross-cutting build guidance to §6.
4. Use the `edit` tool with multiple PUT hunks in ONE call using ORIGINAL line numbers; re-read to get fresh line numbers/tag between calls.

## Report each turn
Structural read (anchor/Σ/colorbar/table/tooltip) + audit findings (what the old text got wrong) + which map↔table trust rule applies + diff summary. Then prompt for the next capability.
