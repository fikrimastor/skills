---
name: tricipta-plot-api-capability-docs
description: "Author or refresh per-capability documentation for the Tricipta diagnostic plot API from the live registry — fact pack via describe_capability, a committed render-verification generator, and the traps (level drops columns, spatial coordinates, HTTP output_formats 500, PDF-vs-code formula drift). Use when asked to document capabilities, explain the plot API to developers, or when capability docs/facts.json go stale."
---

## When to use

Documenting or refreshing `docs/capability-guidelines/` in `tricipta-diagnostic-plot-api`, or answering "what does capability X need / how do I call it" for a developer.

## Ground rules

- **Count from the registry, not the notebook.** `len(list_capabilities())` is authoritative (128 as of 2026-09; the notebook's printed 118 is stale).
- **`plotting_core.describe_capability(name)` is the per-capability contract.** It returns `required_canonical_columns`, `input_alternatives`, `derived_metrics[].human_equation`, `preparation_options` (with `required`/`default`), and `example_request`. Never hand-parse `domain/capabilities.yaml` (25k lines, drifts).
- **Code beats the PDF.** `Plots_&_Bubble_Maps_Specifications_29 June.pdf` is materially wrong on several formulas; extract it with `pdftotext -layout` only for product intent and note every disagreement in the doc.
- **Never invent output.** Quote a measured render.

## Procedure

1. **Orient.** `ls plotting_core/engines/ transforms/` and read `plotting_core/capability_contract.py` (`build_capability_contract`, `describe_capability_contract`) to see what the contract exposes.

2. **Write/keep the generator in the repo**, not in `/tmp` — the docs cite measured numbers that must be regenerable:
   `outputs/export_capability_guidelines.py`.
   It must (a) build a physically-valid fixture, (b) for each capability call `describe_capability()` + `build_plot_response()`, (c) write a fact pack JSON, (d) support `--check` that exits non-zero if any capability has an error-severity diagnostic or `plotted_rows == 0`.

   ```bash
   cd ~/Sites/afed/tricipta-diagnostic-plot-api
   PYTHONPATH=. python -m outputs.export_capability_guidelines --check
   # expect: 128 capabilities across 16 families / error diagnostics: 0 / empty plots: 0
   ```

3. **Repair the fixture.** `make_synthetic_monthly_data()` is a demo, not a fixture. It needs:
   - `reservoir_pressure` distinct from `bottomhole_pressure` — and by well type (producers: reservoir > bottomhole; injectors: bottomhole > reservoir), because `pi` needs reservoir − bottomhole > 0 while `injectivity_index`/`hall_*` need bottomhole − reservoir > 0.
   - `surface_x`/`surface_y` aliases for the spatial coordinate defaults.
   - static reservoir inputs: `ooip`, `area`, `porosity`, `net_pay`, `thickness`, `initial_water_saturation`, `water_saturation`, `bo`/`bw`/`bg`.
   - `sbhp` distinct from the flowing pressures.
   - oil/gas injection rates + cumulatives on injectors (the sample only carries water injection).
   - Do NOT add `producing_days` — it is derived correctly as a per-well cumulative sum of `active_days`.

4. **Encode the option rules in `recommended_options()`:**
   - Pass `level` ONLY for `field_/zone_/string_ (production|injection)` time-series families.
   - For `engine == "spatial"`: never pass `level`; pass explicit `x`/`y`; add `selected_date`.
   - Add `selected_date` for `latest_per_entity` capabilities and `rank_entities` capabilities.
   - Add `top_n` only when `rank_entities` is in `preprocess`.

5. **Organise chapters by what you are drawing, not by registry family.** The `field/zone/string` families are the same metrics at different aggregation levels — one chapter with a **level matrix** beats six near-duplicate docs. A workable split:
   - `field-zone-string-production.md`, `field-zone-string-injection.md` (+ reservoir-balance time series)
   - `well-time-series.md` (all `engine == time_series` and name starts with `well_`)
   - `diagnostic-scatter.md` (ALL `engine == diagnostic_scatter`, spanning families)
   - `well-ranking.md` (bar), `well-maps.md` (spatial)
   Verify the partition is exact: union == all capabilities, no overlap.

6. **Chapter template** (juniors read tables, not prose): title + orientation → `At a glance` (every capability, one row) → `ELI5` → `Why this exists` → `Inputs you must supply` (with the failure diagnostic per column) → `How each capability is computed` (exact `human_equation`) → `Runnable examples` (measured output) → `Traps and failure modes` → `API calls` → `Source of truth` (file anchors).

7. **Verify.** Re-run `--check`; then mechanically assert:
   - every expected capability ID appears in its chapter;
   - every `human_equation` appears (verbatim preferred);
   - code fences balanced; no TODO/placeholder;
   - every diagnostic code named exists in source: `grep -rhno 'code="[a-z_]*"' plotting_core/ | sort -u` **plus** `plotting_core/registries/diagnostics.py` (some codes are declared there, e.g. `missing_formula_inputs`, `invalid_formula_rows`);
   - extract every fenced JSON body containing `data_source` from the docs and POST it through `fastapi.testclient.TestClient(app)` expecting 200.

## The traps that must appear in the docs

| Trap | Symptom | Fix |
|---|---|---|
| `level=` on spatial/scatter | `error: missing_coordinates` / `missing_plot_values`; 0 plotted. `level` routes through `prepare_domain_data`, which drops non-metric columns | Drop `level` |
| Spatial without `x`/`y` | `error: missing_coordinates`. Registry marks them `required: true` with defaults `surface_x`/`surface_y`, but defaults are NOT applied | Pass real coordinate columns |
| Omitting `level` on a `field_/zone_/string_` capability | Silent: 72 per-well points instead of 12 aggregated | Always pass the matching `level` |
| Sample's `reservoir_pressure == bottomhole_pressure` | `invalid_formula_rows` → empty for `pi`, `injectivity_index`, `hall_*`, `voidage_ratio` | Supply genuinely different pressures |
| No `sbhp` column | `warning: missing_metric`; `sbhp` is a distinct supplied metric, not an alias | Supply `sbhp` |
| Ratio metrics on mixed well populations | `invalid_formula_rows` + `missing_metric_values`; injectors carry `null` metric values | `filters={"status": "producer"}` |
| `output_formats` inside `options` on HTTP | **HTTP 500**, `TypeError: got multiple values for keyword argument 'output_formats'` | Top-level field |
| Registry `example_request` copied into a POST body | Fails: it is library-shaped (no `data_source`, `output_formats` nested) | Copy only its `options` block |
| `top_n` on a non-ranking capability | `error: unsupported_option` | Only the 2 `well_ranking.*` capabilities accept it |
| Omit `selected_date` on ranking | Silent: each well snapshots at a different month | Always pass `selected_date` |

## Formula corrections to state explicitly

| PDF says | Code does |
|---|---|
| `liquid_rate = oil_rate + water_rate` | `oil_pd_rate + water_pd_rate` |
| `wor = water_rate / oil_rate` | `water_pd_rate / oil_pd_rate` |
| `PI = liquid_rate / (pressure - bottomhole_pressure)` | `liquid_rate / (reservoir_pressure - bottomhole_pressure)` |
| `injectivity_index = water_injection_rate / (bottomhole_pressure - pressure)` | same shape but `delta_pressure = bottomhole_pressure - reservoir_pressure` |
| `hall_integral = cumulative_sum((bottomhole_pressure - pressure) * elapsed_days)` | integrates `delta_pressure` over elapsed time, per entity |
| Chan GOR as "GOR divided by 1000" | explicit metric `gor_for_chan = gor * 0.001` |

## Delegation

Fan out chapters to subagents only up to the concurrency cap (the harness caps at 2; over-dispatching past it drops writers silently — verify each expected file exists afterwards). Each brief must carry: exact output path, the explicit capability list, the fixture traps, the required section order, and the instruction to quote `facts.json` `verified.*` rather than inventing output. Slices must be disjoint files.

## Pitfalls seen in practice

- Writing the harness in `/tmp` makes every "verified" number unmaintainable — commit it.
- A grep for a code string can miss a backticked or differently-worded claim; audit by *reading* the flagged region, and re-verify any claim a reviewer disputes instead of trusting your earlier grep.
- The bundled sample's `oil_pd_rate == oil_rate` alias hides the PD-vs-CD distinction that bites on real data.
- `describe_capability().example_request` defaults `top_n` to 10 and `category` to `well`.
