---
name: tricipta-plot-api-direct-engine-repro
description: "Verify Tricipta diagnostic-plot-api spatial/time-series engine behavior by curling /v1/plots/render directly with a synthetic payload, before trusting multi-repo static-code-tracing theories about a bubble-map, diagnostic-plot, or table value bug. Use before proposing a cross-repo fix when a value spans asset-web + plot-api (or asset-api + plot-api) and the root cause isn't obvious from reading code alone."
---

## Why

Tricipta's plot-api (`tricipta-diagnostic-plot-api`) has several non-obvious indirections that make static code tracing produce confidently wrong root-cause theories:

- `selected_date` (and other options in `PREPARATION_ONLY_OPTIONS`, see `plotting_core/options.py`) is stripped from *render* options — but only **after** `prepare_plot_data()` already applied it (`plotting_core/preparation.py`). Reading `build_spatial()`/`build_plot()` in isolation makes it look like the date is ignored; it isn't, as long as preparation actually runs.
- Preparation only runs if the capability is registered in the YAML registry (`plotting_core/domain/capabilities.yaml`) and `get_capability()` succeeds. If it raises `KeyError`, `api/services/plot_service.py`'s `_render_prepared()` silently skips `prepare_plot_data()` entirely and calls `build_plot()` raw — every preparation-only option (selected_date, filters via `apply_filters`, etc.) becomes a no-op with no error surfaced to the caller.
- The engine's `text` labels (spatial traces) are the raw `entity` column value (e.g. `plot_frame["string"]`), which may not match whatever composite key a caller (e.g. the web app) constructs for its own lookup (e.g. `` `${well}-${reservoir}` ``). A label mismatch silently falls through to a fallback code path rather than erroring.

Given this, a theory like "the engine ignores the date window" or "the parser's last-write-wins picks the wrong row" — built purely from reading Python/TypeScript/Go across three repos — can be **completely wrong**. Verify against the live engine before proposing any fix, especially before touching plot-api itself.

## Procedure

1. **Confirm the container is up**: `docker ps --format '{{.Names}}\t{{.Status}}' | grep -i plot` (expect `plot-api-<id> Up ... (healthy)`). If stale (built before a recent change), rebuild: `docker build -t edafy-asset-diag-api:local . && docker rm -f plot-api-<id> && docker run -d --name plot-api-<id> -p 8000:8000 -e PORT=8000 -e PLOTTING_API_KEY=<key> edafy-asset-diag-api:local`.
2. **Get the API key**: `grep PLOTTING_API_KEY tricipta-diagnostic-plot-api/.env` (dev default is often `plot-twist-api-key-2026`).
3. **Find the real endpoint path** — don't guess from the web's proxy route name (e.g. the web calls its own Nuxt server route `/api/diagnostic-plots/render`, which forwards to plot-api's actual `/v1/plots/render`). Confirm via `grep -rn "@router\." api/routes/*.py` in plot-api.
4. **Build a minimal synthetic payload that mirrors the REAL suspicious data exactly** — same well/string code, same dates, same metric values, pulled from a real DB query if needed (e.g. `docker exec edafy-db psql ...`). Two or three rows is usually enough: one row that should win, one or more decoys that would win under the wrong logic.
   ```bash
   curl -s -X POST http://localhost:8000/v1/plots/render \
     -H "Content-Type: application/json" \
     -H "X-API-Key: <key>" \
     -d '{
       "capability": "<capability.name>",
       "data_source": {"kind": "inline", "records": [ {...row1}, {...row2} ]},
       "units": {},
       "options": {"time": "date", "entity": "string", "x": "bottomhole_x", "y": "bottomhole_y", "selected_date": "<YYYY-MM-DD>", "filters": {"status": "producer"}},
       "output_formats": ["neutral_json"]
     }' | python3 -m json.tool
   ```
5. **Run it twice**: once with the option you suspect is being ignored (e.g. `selected_date`), once without. Compare `neutral_json.traces[0].marker.metric_values` and `.text`. If the engine answers correctly when given the option the caller claims to send, the bug is NOT in the engine — go look at whether the caller (web/Go) actually sends that option at runtime (stale build, a conditional that doesn't fire, a watcher/trigger gap), not at the engine's math.
6. Only propose an engine-side (plot-api) fix once this repro proves the engine itself is wrong for a given input+options combination.

## Payoff

In one session this repro took two `curl` calls to overturn a multi-turn, multi-repo static-trace theory ("engine ignores the date window, parser last-write-wins picks the stale row") and correctly redirect the fix to a one-function frontend trigger-wiring bug (a Vue watcher that skips re-fetching while a playhead animation is running, with nothing forcing a resync when it stops).
