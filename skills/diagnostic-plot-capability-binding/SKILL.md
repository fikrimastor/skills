---
name: diagnostic-plot-capability-binding
description: "Bind a diagnostic-plot capability's datatable structure into the Tricipta analysis criteria + agent prompt docs, one capability at a time — including auditing existing criteria against the table shape. Use when Kanda provides a datatable structure (columns, units, sample rows) for a capability and wants the LLM analysis output enhanced."
---

# Diagnostic Plot Capability Binding Loop

Enhance LLM analysis output for one diagnostic-plot capability by binding its actual datatable structure into both docs. One capability per session — focused discussion, then repeat.

## Source docs (keep in sync — drift is a bug)
- Criteria (source of truth): `~/Sites/afed/tricipta/docs/diagnostic-plot-capabilities-analysis-criteria.md`
- Prompts (runtime fleet): `~/Sites/afed/tricipta/docs/diagnostic-plot-analysis-agent-prompts.md`
- 46 capabilities across 4 families: F1 Rates & Production (11), F2 Injection/Pressure/Activity (14), F3 Scatter & Distribution (19), F4 Well Ranking (2).

## Input from user (per capability)
1. Capability name + family.
2. Datatable structure: columns, units, granularity (daily/monthly), time basis (calendar vs producing days), sample rows, pagination/record count, any metadata the API sends alongside.

## Analysis pass (before editing — this is the value)
1. **Shape**: long vs wide format; which columns are keys (WELL STRING, RESERVOIR, TIMESTAMP) vs values. Multi-series tables MUST be grouped by key — never analysed as one series.
2. **Dual-column traps**: CD vs PD columns (downtime proxy = 1 − CD/PD); cumulative vs rate columns (cumulatives identical across bases, rates differ).
3. **Shared step-up trap**: a step shared by ALL series at the same date = aggregation/unit change (§0.1), not reservoir behavior.
4. **Gaps**: missing months = no data; no interpolation, no derivatives across gaps.
5. **What the LLM can compute directly** vs derivable (decline fit, breakthrough point, slopes) vs missing (state as gaps, never guess).
6. **Criteria audit (MUST)**: re-read the capability's existing criteria against the table shape and fix mismatches, not just add bindings. Known traps:
   - Event correlation claims without an events/status column → qualify as cross-capability inference only.
   - Cyclicity/slugging claims on monthly data → bound to seasonal/annual resolution.
   - Flags referencing columns that don't exist (e.g. "with active well status") → point to the activity series that substitutes.
   - Aggregation bias: string-level totals mix vintages/statuses → decline rates on aggregated data are biased low; agent must state aggregation level with every decline estimate.

## Edits (both docs, same pass)
### Criteria doc — add `- **Data binding (…)**` subsection after `- **Data required**`:
- Schema: format, columns, units, keys.
- Series identification rule (group by key).
- Primary vs secondary series + why.
- Derived proxies (e.g. downtime ratio).
- Gap/step traps specific to this table.
- Cross-checks to other capabilities.
- Gaps to state in output.
- Aggregation bias note.
Then fix the criteria bullets/flags/output lines that the table shape invalidates.

### Prompt doc — add `DATA BINDING (…):` block at the top of the capability's numbered section, before the first criterion:
- Same content, compressed to prompt voice (imperative, bulleted).
- Mirror every criteria fix (flags, event correlation, cyclicity bounds, aggregation bias).
- Also update the family INPUT CONTRACT if the table shape is new (e.g. long-format note).

## Verification
- Grep both docs for the capability name + binding marker; confirm the binding content matches (thresholds/units identical).
- Grep for each corrected phrase in both docs — criteria fix and prompt mirror must both be present.
- Confirm the capability's remaining criteria text is untouched (binding is additive).

## Output to user
- Structural analysis of the table (shape, traps, computable/derivable/missing).
- Criteria audit findings (what the old criteria got wrong for this table shape).
- Diff summary of both edits.
- Next capability prompt.
