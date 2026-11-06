---
name: tricipta-capability-input-contract-audit
description: "Enumerate, per Tricipta diagnostic-plot capability, the inputs required to RENDER the chart versus the inputs required to produce AI INSIGHT analysis — using the two plot-api introspection registries (describe_capability vs analysis_profiles) and the web/asset-api surface allowlists. Use when asked to go through / walk through / enumerate each or every diagnostic-plot capability and say what it needs to plot and what it needs for AI insight; what inputs a capability requires; whether a capability has every input its insight analysis demands; whether a capability is analyzable by the engine at all; which capabilities a dashboard can offer insight for; or checking whether a capability even has an analysis profile before it can be bound into the criteria docs."
---

# Per-capability input contract audit (render vs. insight)

Tricipta has **two independent input contracts per capability**, in two registries, with two different gate functions and two different metric vocabularies. Never answer "what does this capability need" from one of them.

| | Render | Insight (AI analysis) |
|---|---|---|
| Registry | `domain/capabilities.yaml` → `CAPABILITIES` | `plotting_core/domain/analysis_profiles.yaml` → `AnalysisProfile` |
| Introspection | `plotting_core.catalog.describe_capability(cap)` | `plotting_core.analysis.profiles.get_analysis_profile(cap)` |
| Gate | `prepare_plot_data` + option validation | `_analyze_one_capability` |
| Vocabulary | canonical column names (`oil_rate`, `oil_pd_rate`) | profile metric names (`oil_pd_rate`, `producing_days`, `bottomhole_x`) |
| Cardinality | 128 capabilities | **64** profiles |

The vocabularies do not match, and the mismatch bites on coordinates too: every spatial capability's render option defaults are `x: surface_x` / `y: surface_y`, while its analysis groups require `bottomhole_x` / `bottomhole_y`. `oil_rate` (render) vs `oil_pd_rate`/`oil_cd_rate` (analysis) is the same physical quantity under two names; do not assume a column named in one registry exists in the other.

## Authoritative spans

| Thing | Location |
|---|---|
| Capability declarations (128) | `domain/capabilities.yaml`; loader `plotting_core/domain/loaders.py:315-407`; registry `plotting_core/registries/capabilities.py:32` |
| Per-capability render contract | `plotting_core/catalog.py:125` → `plotting_core/capability_contract.py:50-70` (`build_capability_contract` at `:19`) |
| Render HTTP schema | `api/routes/plots.py:17`, `api/schemas/plots.py:19-33` |
| Analysis profiles (64) | `plotting_core/domain/analysis_profiles.yaml`; parser `plotting_core/analysis/profiles.py:67-92,106` |
| Analysis HTTP schema | `api/routes/analysis.py:12`, `api/schemas/analysis.py:34-38` |
| The insight gate | `plotting_core/analysis/service.py:207-470` (`_analyze_one_capability`; `analyze_capabilities` above it at `:174`, `status="ready"` return at `:459`, next def `_unsupported` at `:496`) |
| Generated capability docs | `outputs/export_capability_guidelines.py`; `docs/capability-guidelines/` |
| Web render/insight surfaces | `app/composables/useWellAiInsight.ts:87-111`; `app/utils/wellAiInsightPrompt.ts:102-115` |
| Web curated families (render menu) | `app/utils/diagnosticPlotCuratedFamilies.ts:13-89` |
| Web bubble-map insight allowlist | `app/pages/common-platform/bubble-maps-dashboard.vue:1670-1689` |
| Asset-API insight orchestration | `internal/application/diagnosticplotinsightssvc/insight_service.go:60-153` |
| Asset-API family criteria (LLM) | `internal/application/diagnosticplotinsightssvc/insight_prompt.go:78-123` (4 cases + default) |
| Spatial LLM criteria (bubble maps) | `internal/application/bubblemapinsightssvc/prompt.go` |

`len(list_capabilities())` is authoritative for the registry count — never count YAML by hand or trust the README/notebook. The anchored grep `^- name: <ns>.<slug>$` currently agrees (128/128); a naive grep for `name:` over-counts. The generated `docs/capability-guidelines/facts.json` agrees (128 capabilities / 16 families) and is a fine cross-check, but it describes the *registry* families — not the 4 curated insight families in the criteria docs.

**Neither introspection call is a verdict oracle.** `required_canonical_columns` is a *stated* requirement, not what renders: `well_productivity.productivity_index_time_series` reports `['oil_pd_rate','water_pd_rate','reservoir_pressure','bottomhole_pressure']`, yet a frame built from aliased `liquid_rate` still renders ~12 points (the render's alias bridge + `materialize_metrics` fill in). Same shape on the analysis side for formula-backed groups. Use these contracts to *enumerate* requirements; use a real `build_plot_response` / `analyze_capabilities` call to *judge* a frame. That render-vs-`required_canonical_columns` divergence is the entire subject of `tricipta-analysis-gate-vs-render-alignment`.

## Procedure

### 1. Enumerate both contracts in one pass

The join key is the capability name — identical vocabulary in both registries. Run this from `~/Sites/afed/tricipta-diagnostic-plot-api` (repo uses `venv/`, not `.venv/`, and tracks `.pyc` files — always set `PYTHONDONTWRITEBYTECODE=1` or you dirty the worktree with modified `api/**/__pycache__/*.pyc`):

```bash
cd /Users/fikrimastor/Sites/afed/tricipta-diagnostic-plot-api
PYTHONPATH=. PYTHONDONTWRITEBYTECODE=1 venv/bin/python - <<'PY'
import json
from plotting_core.registries.capabilities import list_capabilities
from plotting_core.catalog import describe_capability
from plotting_core.analysis.profiles import get_analysis_profile, list_analysis_profiles

caps = sorted(c.name for c in list_capabilities())
profiled = {p.capability for p in list_analysis_profiles()}

for cap in caps:
    r = describe_capability(cap)
    opts = r.get("preparation_options") or {}
    try:
        p = get_analysis_profile(cap)
        a = {
            "extractor": p.extractor, "prompt_profile": p.prompt_profile,
            "scope": p.scope, "minimum_points": p.minimum_points,
            "groups": [list(g.any_of) for g in p.required_input_groups],
            "units": dict(p.unit_requirements),
            "optional_context": list(p.optional_context_inputs),
        }
    except KeyError:
        a = None
    print(json.dumps({
        "capability": cap, "engine": r.get("engine"), "family": r.get("family"),
        "render_required": list(r.get("required_canonical_columns") or []),
        "render_alternatives": r.get("input_alternatives") or [],
        "render_derived": [m.get("name") for m in (r.get("derived_metrics") or [])],
        "render_options_required": sorted(k for k, v in opts.items() if v.get("required")),
        "analysis": a,
    }))

print("registry=%d profiles=%d renderable_but_not_analyzable=%d"
      % (len(caps), len(profiled), len([c for c in caps if c not in profiled])), flush=True)
PY
```

Record three numbers before analysing any single capability: registry size, profile size, and the renderable-but-not-analyzable set.

### 2. Report per capability

For each capability produce these rows. Anything not derivable from `describe_capability`/`get_analysis_profile` is marked `[INFERENCE]` or `[GAP]`.

| Column | Source |
|---|---|
| `engine` | `describe_capability().engine` — `time_series` / `diagnostic_scatter` / `spatial` / `bar` |
| Render-required columns | `required_canonical_columns` |
| Render alternatives / derived | `input_alternatives`, `derived_metrics[].name` |
| Required options | `preparation_options` where `required: true` — `time` for time-series families; `entity`, `x`, `y` for every spatial capability (defaults `surface_x`/`surface_y` are **not** applied automatically) |
| Analysis extractor | `get_analysis_profile().extractor` — drives which features are emitted |
| Analysis required input groups | `required_input_groups` — a list of `any_of` groups, **all** groups must be satisfied |
| Analysis unit requirements | `unit_requirements` — caller must supply a unit per resolved metric |
| Optional context inputs | `optional_context_inputs` (e.g. `reservoir_pressure` for oil rate) |
| Minimum points | `minimum_points` — 6 for time-series/scatter, **1** for all spatial |
| Analyzable? | `unsupported_registry` when no profile, or `extractor not in ALL_EXTRACTORS` |

`any_of` semantics matter: `water_cut` needs *both* an oil group and a water group; the rate groups accept `*_pd_rate` **or** `*_cd_rate`, so a CD-only dataset is analyzable but changes which series is analysed.

### 3. Read the surface allowlists, not the registry

The UI never exposes all 128. Both surfaces are curated and both are exhaustive vs. the profile set:

| Surface | Selectable capabilities | Equals |
|---|---|---|
| Diagnostic-plots dashboard | 46 across 4 families (`AST_PRODUCTION_CURATED_FAMILIES`) | the 46 non-spatial profiles — **exact set identity** |
| Bubble-maps dashboard insight | 18 (`SUPPORTED_INSIGHT_CAPABILITIES`) | the 18 spatial profiles — **exact set identity** |

So: registry 128 → analyzable 64 (46 dashboard + 18 spatial) → 64 unanalyzable (all `field_*`, `zone_*`, `string_*`, plus `reservoir_balance.production_injection_overlay`, 4 `well_pressure` reservoir-pressure/pressure charts, and 7 `well_maps.period_*`/pressure/cumulative-injection maps). A capability being renderable says nothing about it being analyzable.

### Family cardinality — four different numbers, all correct

Do not "reconcile" these; they count different things. Report which axis you are on.

| Number | Counts |
|---|---|
| 16 families / 128 capabilities | registry (`capabilities.yaml`, matches `facts.json` totals) |
| 64 capabilities | analysis profiles (`analysis_profiles.yaml`) — ungrouped by family |
| 18 capabilities | spatial profiles, = bubble-maps `SUPPORTED_INSIGHT_CAPABILITIES` (all `well_maps.*`) |
| 4 families / 46 capabilities | curated *insight* families: Rates & Production (11) + Injection, Pressure & Activity (14) + Diagnostic Scatter & Distribution (19) + Well Ranking (2) |

The criteria/prompt docs in `~/Sites/afed/tricipta/docs/` use the 4-family/46-capability axis (`diagnostic-plot-capabilities-analysis-criteria.md`, `diagnostic-plot-analysis-agent-prompts.md`); the 11/14/19/2 split matches `get_analysis_profile().prompt_profile` exactly. The spatial 18 have no criteria-doc section — their binding lives in `bubble-map-capabilities-analysis-criteria.md`.

## Traps

| Trap | Symptom | Real cause |
|---|---|---|
| Chart renders, insight says `insufficient_data` | `missing_inputs: [<metric>]` | Gate 1 (render) and gate 2 (analysis) read different registries. A capability absent from `analysis_profiles.yaml` returns `unsupported_registry` with no explanation. |
| `missing_inputs` names a formula-derivable metric (`producing_days`) | Insight blocked though the chart derived it | Gate is derivation-aware (`service.py:256-278` materializes `has_formula` members before gating), but asset-api emits `active_days`, and DB `production_days` is a **different key** — check `plot_engine_mapper.go` output keys, not DB column names. |
| `missing_inputs: ["scope"]` | Every well returns it | `_scope_satisfied` (`service.py:529`) — profile `scope` (`well`/`field`/`reservoir`/`platform`) vs supplied `analysis_context`. Note `well_ranking.*` profiles are `scope: field` while the capability is per-well. |
| `missing_inputs: ["selected_date"]` | Ranking/exceedance/`latest_per_entity` fail | `_requires_common_selected_date` (`service.py:642`); web only sends `selected_date` when the caller passes it. |
| Units rejected though the chart shows values | Missing-unit diagnostic names the **resolved** metric | `_missing_units` (`service.py:792`) requires a caller unit for every `unit_requirements` entry; in-process derived columns are exempted. The web insight request carries **no** `units` field — asset-api supplies them (`UnitsForPlotEngineRecords`). |
| Spatial capability "needs 6 points" | Wrong gate assumed | All 18 spatial profiles are `minimum_points: 1`; the 46 others are `6`. |
| Assuming 5 LLM family criteria cover all insight | Spatial gets rate/production wording | `insight_prompt.go` switch has **4** cases + `default: familyRatesProductionCriteria`. The 5th prompt profile (`spatial`) lives in plot-api and is surfaced by `bubblemapinsightssvc/prompt.go`, not that switch. |
| Counting 128 from `capabilities.yaml` with `grep -c name:` | Inflated count | Metric/option keys also match; anchor to `^- name: <ns>.<slug>$`. |
| `describe_capability().example_request` pasted as an HTTP body | 422/500 | It is library-shaped (`options` nested, no `data_source`); the HTTP body takes `capability`, `data_source`, `units`, `options`, and top-level `output_formats`. |
| Trusting `required_canonical_columns` as the render verdict | Over-reporting a capability as unrenderable | It is a stated requirement; the alias bridge + `materialize_metrics` satisfy more frames than it lists (see the oracle caveat above). |

## Verification gates

- Registry vs declaration: `len(list_capabilities()) == ` count of `^- name: ` matches.
- Profile/registry join: `analyzable_but_not_renderable` must be empty (it is).
- Surface joins must both be exact set identities (46/46 and 18/18). If either drifts, the skill's denominators are stale — re-derive before reporting.
- For a capability claimed *analyzable*, prove it end to end: POST `/v1/analysis/capabilities` (or `analyze_capabilities` in-process) with a frame shaped exactly like the asset-api mapper output and confirm `status: ready`, not `insufficient_data`.
- For a claimed *required input*, the column must appear in the profile's `required_input_groups` — not merely in `required_canonical_columns` (that would be the render contract).
- Run every python command with `PYTHONDONTWRITEBYTECODE=1`; verify with `git status --porcelain -- api domain plotting_core | grep -c pyc` that the count did not increase.

## Related skills

- **`tricipta-analysis-gate-vs-render-alignment`** — the nearest twin and the complement to this skill. It owns *value/verdict conformance*: does the gate agree with the render on a real frame, and does the AI number match the drawn number. This skill owns the *inventory*: what each registry declares per capability, and which capabilities reach each UI surface. Use both — enumerate here, then conformance-test there. Its `analysis_profiles.yaml` facts (64/128, `unsupported_registry` set, gate order, vocabulary split) were derived independently and agree.
- `tricipta-insight-debugging` — fixing a specific gate rejection / "Missing inputs" report (§2).
- `tricipta-plot-api-capability-tracing` — how a chosen capability actually computes/renders.
- `tricipta-plot-api-capability-docs` / `verified-capability-guideline-corpus` — the committed generator (`outputs/export_capability_guidelines.py`) that produces `facts.json`, which this skill cross-checks against.
- `diagnostic-plot-capability-binding` — the *doc-authoring* leg: binding a rendered datatable's shape into the criteria + prompt docs, one capability at a time. That is a different job from enumerating registry contracts; this skill does not restate it.
