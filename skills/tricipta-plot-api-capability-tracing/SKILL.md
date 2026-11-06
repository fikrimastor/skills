---
name: tricipta-plot-api-capability-tracing
description: "Trace how a specific tricipta-diagnostic-plot-api capability (e.g. well_ranking.oil_rate, well_maps.*, chan_diagnostics.*) actually computes and renders, grounding the explanation in source instead of paraphrasing the thin PDF spec or notebook demo. Use when asked to explain, debug, or extend a plot-api capability's mechanics."
---

## Why this matters

`Plots_&_Bubble_Maps_Specifications_*.pdf` and `examples/notebook_quickstart.ipynb` are declarative/demo docs — one sentence per capability, or one worked call. Neither shows sort order, tie-break rules, option defaults, validation, or rendering quirks. For a grounded answer, trace the actual registry-driven pipeline instead of paraphrasing the docs.

## The 5-file trace

1. **`domain/capabilities.yaml`** — grep `- name: <family>.<slug>` or `family: <Family Name>`. Gives: `engine`, `metric`/`x_metric`/`y_metric`, `preprocess_steps` (ordered list of pipeline steps this capability opts into), `control_details` (every option, its default, required-ness), `data_requirements.derived_metric_details` (human_equation for any formula-derived metric).
2. **`plotting_core/preparation.py`** (`prepare_plot_data`) — the fixed pipeline order, each step gated on whether its name appears in `resolved.preprocess`:
   `apply_filters` (~L93) → unit conversion (~L98) → formula derivation/domain aggregation (`materialize_metrics`/`prepare_domain_data`, ~L134-160) → `select_latest_per_entity` if `"latest_per_entity" in preprocess` (~L170) → `rank_entities` if `"rank_entities" in preprocess` (~L185). This order is NOT visible in the YAML or either doc — only in code.
3. **`plotting_core/transforms/<step>.py`** — the actual algorithm for each preprocess step named in the capability's `preprocess_steps`. e.g. `ranking.py:rank_entities` (sorts `[metric, entity]` with `ascending=[False, True]`, `kind="mergesort"`, then `.head(top_n)`; drops rows where metric/entity is NaN before truncating — no diagnostic for those drops); `selection.py:select_latest_per_entity` (one row per entity, at/before `selected_date`, else most recent, with a fallback sort-by-metric tiebreak).
4. **`plotting_core/engines/<engine>.py`** — how the prepared frame becomes a `NeutralPlotResult` (trace kind, x/y/category binding, diagnostics for missing bindings like `missing_bar_binding`/`missing_bar_values`). Confirm engine dispatch via `plotting_core/catalog.py:ENGINE_BUILDERS`.
5. **`plotting_core/renderers/plotly.py`** — engine-specific Plotly quirks not implied by the neutral spec. e.g. bar charts get `figure.update_yaxes(autorange="reversed")` specifically when `neutral.kind == "bar"`, so rank-ordered categories render highest-first at the top — Plotly's default would otherwise put the first list item at the bottom, inverting "Top wells".

## Fast greps

- `grep -n "rank_entities\|latest_per_entity" domain/capabilities.yaml` — which capabilities opt into a given preprocess step (often very few; `rank_entities` is used by exactly the 2 `well_ranking.*` capabilities in the entire ~125-capability registry, nothing else).
- `grep -n "family: <Name>"` in `capabilities.yaml` to enumerate every capability in a family before reading any one in full.
- Cross-check option validity/defaults in `plotting_core/options.py` (e.g. `top_n` is only accepted when `"rank_entities" in capability.preprocess`, else it's flagged as an `unsupported_option` diagnostic; must be a positive int, bool explicitly excluded since `bool` subclasses `int` in Python).

## Worked example (well_ranking family)

- Exactly 2 capabilities: `well_ranking.oil_rate` (direct `oil_rate` column, no derivation) and `well_ranking.cumulative_oil` (derived: cumulative sum of `period_oil` in time order, itself `oil_rate * active_days` when no explicit period/cumulative delta is supplied). Both: engine `bar`, level `well` only, `preprocess: [latest_per_entity, rank_entities]`, `top_n` default 10.
- PDF wording "Top wells ... at the selected date" = the `latest_per_entity` step collapsing each well's full time series to one row at/before `selected_date` BEFORE `rank_entities` sorts those single-row snapshots — ranking never looks at an all-time max, only the snapshot value.
- No "bottom N" / ascending variant is registered anywhere, even though `rank_entities(ascending=True)` is supported by the function signature — `preparation.py`'s call site never passes `ascending`, so every ranking capability is hardcoded descending.

## Path gotcha when tracing across the Tricipta workspace

If `cwd` is the `tricipta` umbrella repo, `tricipta-diagnostic-plot-api` (and the other 4 Tricipta repos) are **sibling workspace roots**, not subdirectories — relative paths like `tricipta-diagnostic-plot-api/domain/capabilities.yaml` 404 in grep/read/glob. Use the absolute path `/Users/fikrimastor/Sites/afed/tricipta-diagnostic-plot-api/...` or a `../`-prefixed relative path.
