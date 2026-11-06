---
name: tricipta-input-contract-doc-generation
description: "Generate or regenerate the committed Tricipta diagnostic-plot render-vs-insight input contract corpus (docs/diagnostic-plot-capability-input-contract.md) from the live plot-api registries, and keep its generator honest. Use when asked to build/enumerate input-requirement tables per capability, list what each capability needs to plot vs for AI insight, or refresh that doc after capabilities.yaml / analysis_profiles.yaml changes."
---

# Input-contract doc generation (render vs. insight)

Builds `docs/diagnostic-plot-capability-input-contract.md` in `~/Sites/afed/tricipta` from the
plot-api registries. Generator: `docs/scripts/capability-input-contract.py`. Never hand-edit the doc.

Sibling job, different scope: `tricipta-capability-input-contract-audit` covers *auditing* what the
registries declare. This skill covers *generating and maintaining the committed artifact*.

## Two registries, two vocabularies

| | Render (chart draws) | Insight (AI analysis) |
|---|---|---|
| Registry | `plotting_core/domain/capabilities.yaml` | `plotting_core/domain/analysis_profiles.yaml` |
| Introspection | `plotting_core.catalog.describe_capability(cap)` | `plotting_core.analysis.profiles.get_analysis_profile(cap)` |
| Gate | `prepare_plot_data` + option validation | `_analyze_one_capability` (`analysis/service.py:207`) |

128 capabilities / 64 profiles = 46 non-spatial (11 `rates_production`, 14 `injection_pressure`,
19 `diagnostic_scatter`, 2 `well_ranking`) + 18 `spatial`. The other 64 capabilities have no profile
and always return `unsupported_registry`.

## Generator requirements

Four things the script must do or it produces a wrong artifact:

1. **Re-exec into the plot-api venv.** `pandas` is not on the system interpreter. Test `import pandas`,
   and on `ModuleNotFoundError` `os.execv(PLOT_API/'venv/bin/python', ...)`.
2. **`sys.dont_write_bytecode = True`** plus `PYTHONDONTWRITEBYTECODE=1`. That repo *tracks its `.pyc`
   files*; a plain import dirties ~55 files in the worktree. Verify with a before/after md5 of
   `find plotting_core api -name '*.pyc' -exec stat -f %m {} \;` — must be identical.
3. **Self-check that cannot go vacuous.** `build()` must raise if the ⚠/⛔ classes come out empty, if a
   named expected capability is missing from its class, or if a known-satisfiable capability
   (`well_production.liquid_rate`, `well_maps.np_tp`, `well_maps.lp_tp`) is flagged. An always-False
   predicate otherwise silently publishes an unmarked doc.
4. **`--check` mode** comparing committed vs generated; wire it next to the freshness gate in CI.

## The two gap classes (both probe-verified — never infer from YAML alone)

**⚠ ALIAS (14 rows)** — render contract asks the legacy name, profile demands PD/CD. The bridge
`_apply_metric_aliases` (`preparation.py:245`) is called only at `preparation.py:58` inside
`prepare_plot_data`; `_analyze_one_capability` never calls it, and `has_formula('oil_pd_rate')` is
`False`, so the derivation-aware gate (`analysis/service.py:256-278`) cannot rescue it. **Terminal** —
send the PD/CD metric explicitly. Rows: `well_production.{oil,water,gas}_rate`, `gor`, `water_cut`,
`well_ranking.oil_rate`, `well_maps.{oil,water,gas}_rate_bubble_map`, 5 chan/injection-diagnostics.

**⛔ CONTRACT (3 rows)** — different metrics, no bridge, no formula:

| Capability | Render | Insight | Why |
|---|---|---|---|
| `well_maps.liquid_rate_bubble_map` | `oil_pd_rate`+`water_pd_rate` | `liquid_pd_rate` | `has_formula('liquid_pd_rate')=False`; not in the alias table |
| `reservoir_balance.voidage_replacement_ratio` | *no canonical columns* | 6 rate names | satisfiable **only if** the frame carries the rates (formula closure derives `voidage_ratio`) |
| `reservoir_balance.rf_versus_hcpvi` | 6 reservoir cols | +3 injection cumulatives | asset-api never emits `oil_injection_*` |

Classify by **input group**, not whole profile — a bubble map carries coordinate groups alongside its
rate group and only the rate group is blocked.

Implement reachability as: `render_contract_columns` (stated requirement ∪ alternatives ∪ **derived
metrics' `required_inputs`** ∪ option-supplied columns) → `formula_closure` (fixed point over
`list_formulas()`; this is what keeps `voidage_replacement_ratio` and `np_tp` from being false-flagged)
→ compare each `required_input_groups` entry.

## Four generator bugs to avoid (each produced a wrong table here)

1. **Alias dict direction.** `{render_name: (insight_names,)}` — testing insight names against dict
   *keys* never matches, so the flag never fires. Guard with a non-vacuity self-check.
2. **Seeding the closure with alias-expanded columns.** Analysis does not alias; expanding makes every
   PD/CD target look reachable and empties the ⚠ class. Closure seed = raw render contract.
3. **Units column from `unit_requirements` wholesale.** `_missing_units` (`service.py:792`) iterates the
   *resolved* columns only — one member per group, the first declared present one
   (`_resolve_profile_columns`, `service.py:741`). An unused alternative is not owed a unit. Render the
   column per group: resolved member's unit + parenthesised alternatives.
4. **Dropping `required_input_alternatives`.** Reading only `required_input_groups` loses
   `well_productivity.productivity_index_time_series`, whose real clause is the alternative.

## Markdown emission

Escape `|` as `\|` in every cell. Verify afterwards: group lines into contiguous `|`-blocks and assert
uniform *unescaped* pipe count per block (a single global expected count is wrong — 2/4/6-column
tables all appear).

## Probe traps when verifying a claim

- **Context must match the profile's scope.** `well_code` on a spatial profile (or its absence for
  `scope: well`) fails on `scope` *before* the metric check — re-run with correct context rather than
  reporting a contract gap.
- `rank_at_selected_date` needs `selected_date` + ≥2 entities; `anchor_month_all_producers` needs
  `anchor_month` + a `status` column carrying `producer`.
- **Synthetic frames under-report derivable metrics.** Omitting `active_days` makes `np_tp` look
  unreachable; the real asset-api mapper always emits it.
- Prove a claim by running the gate twice: render-shaped frame → failure, then add the named metric →
  `ready`.
