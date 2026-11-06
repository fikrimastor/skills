---
name: tricipta-ddm-sensitivity-insight-live-verify
description: "Live-verify or tune the asset-api DDM drainage-sensitivity AI insight (#1352): 502 ai_unavailable, source=template, gate repair codes, fallback rate, missing rates, budget/timeouts, browser review."
---

# Live-verify DDM drainage-sensitivity insight

## Run it
```bash
cd ~/Sites/afed/tricipta-edafy-asset-api
docker compose build api   # no source mount; rebuild after every code change
SENSITIVITY_INSIGHT_BUDGET=60s INSIGHT_LLM_TIMEOUT=50s docker compose up -d api   # shell override only; never edit/stage docker-compose.yml
T=$(curl -s -X POST http://localhost:8085/api/v1/auth/login -H 'Content-Type: application/json' -d '{"username":"testuser","password":"testpass"}' | jq -r '.access_token // .data.access_token // .token // .data.token')
R=$(curl -s -H "Authorization: Bearer $T" http://localhost:8085/api/v1/ddm/sensitivity-page/refs)
S=$(echo $R | jq -r .screen_id); U=$(echo $R | jq -r .sensitivity_run_id)
curl -s -H "Authorization: Bearer $T" "http://localhost:8085/api/v1/insight/drainage-sensitivity?screenId=$S&sensitivityRunId=$U&refresh=true" -o /tmp/sens.json -w '%{http_code} %{time_total}s\n'
jq -r '.meta|"\(.source) \(.prompt_version) repair=\(.repair)"' /tmp/sens.json
docker logs edafy-api --since 5m | grep 'gate rejected'   # per-attempt gate_failures with section + phrase
```
- Login: max 2 tries (lockout after 5). Route is GET; POST → 404. **There is no `/health` route** (404) — wait on the container's `healthy` status.
- Run 6 times per measurement: the LLM output varies, one pass proves nothing.
- If the bash tool reports "timed out after 900s" at ~30 s wall time on `docker compose build` or long loops, run them via eval Python `subprocess.run(..., timeout=900)` instead.
- Port 8085 may be held by another work stream's host binary (e.g. `/tmp/edafy-api-1355*`, cwd in the -1355 worktree). Check `lsof -nP -iTCP:8085 -sTCP:LISTEN` first. NEVER kill it; publish the container on another port with a compose override file (`ports: !override ["8095:8085"]`, passed via `-f`), and for browser checks rewrite 8085→8095 with a browser init script on the existing :3000 Nuxt.

## Budget wiring
- `SENSITIVITY_INSIGHT_BUDGET` (default 30s): `LoadConfig()` in `orchestration_contracts.go` → `router.go` → handler `context.WithTimeout` covers the whole request.
- Service: ai_unavailable if < `LLMReserve` (2s) remains; each LLM call capped at `min(INSIGHT_LLM_TIMEOUT default 12s, remaining-2s)`; retry skipped if < `RetryMinimumBudget` (5s). Max 2 attempts, then template.

## Read the result
- `502 ai_unavailable`: logs `llm|model call failed` — 403/410 = provider config; `context deadline exceeded` = budget.
- `source=template` + `repair=[codes]`: gate rejected both attempts; tally codes across 6 runs from the WARN logs.
- **High fallback after a new gate rule = usually a prompt↔gate contradiction, not a bad model.** Check: does the per-request checklist (`FormatNarrativeRequirements`) name every id the gate demands (e.g. coverage of all thief producers)? Does a count the gate calls "mislabeled" actually come from a producer/string count (`sensitivityStringCounts`)? Does the prompt's own example text contain a forbidden word ("because")? Is one fact demanded in two sections? Prove each with a throwaway test before changing anything; never weaken a true-positive guard.
- `source=cache`: add `refresh=true`.
- All rates null + `rates_unavailable`: rates SQL error (GORM slice expansion: use `IN ?`, not `unnest(?)`).

## Always
- Read the served sections against `facts` yourself; `repair=[]` does not prove correctness. Past false passes: "all thief producers above 80% WC" when 4 were below; low-rate producer called "cannot currently produce" (gate must check actual rates via `sensitivityHasLiquidRate`, not `rate_flag`); "rates equal to X divided by 519.32" when gas follows a different factor.
- Gate changes: reject test with the live phrase, accept test, template-passes-every-rule test (use served `.facts` as fixture). New hard rules: measure 6 live runs; acceptance = fallback no worse than baseline and no code > 2/6; otherwise demote to prompt-only or fix the contradiction.
- Name test files/fixtures/helpers by content, never by issue number.
- Parallel agents in one worktree: give disjoint file lists, stage explicit paths only, check `git show --stat HEAD` after commit.
- Domain verdicts: plan → implement → measure → review loop with the `production-surveillance-engineer` agent (payload paths, prior findings, rerun cap; browser review at the real panel when asked).
- Report which runs used the shell override/port override; a plain `docker compose up -d api` reverts to `.env` and 8085.
