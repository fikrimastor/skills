---
name: tricipta-plot-api-rebuild-and-env-recovery
description: "Diagnose and fix \"error when rebuilding the docker image\" for tricipta-diagnostic-plot-api (plot API on :8000): prove the Dockerfile builds cold, then find the real break in the build+rm+run chain (missing gitignored .env, wrong container name, silently disabled API key) and restore the local stack with a verified authenticated render."
---

# Plot API rebuild failure — diagnose and restore

Symptom: user reports an error "bila try rebuild docker image" for `tricipta-diagnostic-plot-api`. The named culprit (the Dockerfile) is usually innocent.

## 1. Never judge the build from a cached run

```bash
cd ~/Sites/afed/tricipta-diagnostic-plot-api
docker build --no-cache -t edafy-asset-diag-api:nocache-probe . 2>&1 | tail -30
docker build --platform linux/amd64 -t edafy-asset-diag-api:amd64-probe . 2>&1 | tail -30
```

A run where every step says `CACHED` proves **nothing**. Force a cold build before blaming the Dockerfile. Both of the above pass on this repo (cold arm64 and cross amd64) — so if they pass, the error is downstream, not in the Dockerfile.

## 2. Reproduce the user's real chain, not a bare build

Check `~/.zsh_history` (or `grep -a "diag-api" ~/.zsh_history | tail`) for the actual command. The recurring one is a 3-part `&&` chain:

```sh
docker build -t edafy-asset-diag-api:local . \
  && docker rm -f plot-api-dev \
  && docker run -d --name plot-api-dev -p 8000:8000 --env-file .env edafy-asset-diag-api:local
```

Break it at each step:

| Step | Failure mode here |
|---|---|
| `docker build` | fine (see §1) |
| `docker rm -f <name>` | history sometimes names `plot-api-682` while `run` creates `plot-api-dev` → wrong target removed, later `run` dies `name already in use` |
| `docker run --env-file .env` | **`.env` does not exist** — gitignored (`.gitignore:2`), only `.env.example` ships → `docker: --env-file: open .env: no such file or directory`, exit 125 |

Reproduce it exactly:

```bash
ls -la .env            # expect: No such file or directory
git check-ignore -v .env
docker run -d --name envprobe --env-file .env edafy-asset-diag-api:local   # exit 125
```

Because the chain is `&&`-joined *after* the `rm`, the failure leaves **port 8000 empty** and Diagnostic Plots looks broken. Confirm: `lsof -nP -iTCP:8000 -sTCP:LISTEN`.

## 3. Second-order trap: API key silently disabled

Without `PLOTTING_API_KEY`, the container logs `API key protection is disabled because PLOTTING_API_KEY is not set` — auth is OFF while asset-api/web still send the key. An env-less `docker run` therefore "works" but is wide open.

## 4. Fix

Create `.env` from the `.env.example` local-dev block. The key must be identical across three repos:

```sh
# tricipta-diagnostic-plot-api/.env
PLOTTING_API_KEY=plot-twist-api-key-2026
PORT=8000
APP_ENV=development
PUBLIC_BASE_URL=http://127.0.0.1:8000
```

| Repo | Var |
|---|---|
| `tricipta-diagnostic-plot-api/.env` | `PLOTTING_API_KEY` |
| `tricipta-edafy-asset-api/.env` | `DIAGNOSTIC_PLOT_API_KEY` (base `http://host.docker.internal:8000`) |
| `tricipta-edafy-asset-web/.env` | `NUXT_DIAGNOSTIC_PLOT_API_KEY` |

`.env` is gitignored — never `git add` it; edit it in place.

## 5. Restore and verify (evidence, not assumption)

```bash
docker rm -f plot-api-dev >/dev/null 2>&1
docker run -d --name plot-api-dev -p 8000:8000 --env-file .env edafy-asset-diag-api:local
curl -s -o /dev/null -w '%{http_code}\n' http://127.0.0.1:8000/v1/capabilities                          # expect 401 (auth ON)
curl -s -o /dev/null -w '%{http_code}\n' -H 'X-API-Key: plot-twist-api-key-2026' http://127.0.0.1:8000/v1/capabilities   # expect 200
curl -s http://127.0.0.1:8000/health                                                                    # {"status":"ok"}
curl -s -X POST http://127.0.0.1:8000/v1/plots/render -H 'Content-Type: application/json' \
  -H 'X-API-Key: plot-twist-api-key-2026' \
  -d '{"capability":"well_production.oil_rate","data_source":{"kind":"sample"},"output_formats":["neutral"]}' | head -c 120
# expect {"schema_version":"1.0", ...}
```

Then clean up probe artifacts so they don't shadow the user's stack:

```bash
docker rmi edafy-asset-diag-api:nocache-probe edafy-asset-diag-api:amd64-probe
docker rm -f diag-probe envprobe envprobe2
```

Restoring the tagged image `edafy-asset-diag-api:local` from a rebuilt source tree is the correct end state — that tag is what `run` uses.
