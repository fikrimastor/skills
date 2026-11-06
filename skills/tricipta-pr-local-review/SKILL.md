---
name: tricipta-pr-local-review
description: "Review an AFED-digital/Tricipta GitHub PR (or api+web PR pair, or stacked PR chain) read-only (local summary only, never post to GitHub) when the local working tree is dirty — worktree isolation, pre-existing-failure triage, producer-side literal grounding, route-removal swagger/router parity, table-drop/seeder cleanup PRs, stacked merge vs deploy order, cross-repo contract/empty-state checks, cache-key/ETag/URL-query traps, OTel log correlation, per-deployed-branch consumer impact, cleanup verification."
---

# Tricipta PR — Read-Only Local Review

Use when asked to review an AFED-digital/tricipta PR (or an asset-api + asset-web pair, or a stack of PRs) locally (no GitHub comments).

## 0. Standing constraint
Deliver findings in chat (or a markdown file under `tricipta/docs/` if asked for a written report). NEVER `gh pr comment`/`review`/`api` mutations or `git push`.

## 0.5 Dispatch: pick the right agent type
Kanda wants the `@reviewer` agent to do the review. The bundled reviewer prompt allows only read-only git (line ~65: "NEVER edit files or trigger builds"); it has `bash` in its tools, so the restriction lives in the prompt. Check for the user override `~/.omp/agent/agents/reviewer.md`, a copy of bundled v17.2.15 with line 65 edited. It also allows `go build/vet/test` and `npm ci`/`npm run test`/`npm run build`/`npx nuxi typecheck` inside /tmp worktrees. With the override present, dispatch `agent: "reviewer"` and ask it to run verification. Without it, recreate it from the bundled `src/prompts/agents/reviewer.md` (under `~/.bun/install/cache/@oh-my-pi/pi-coding-agent@*/`), or run the verification yourself in parallel. After an omp upgrade, diff the override against the new bundled prompt. Reviewer output can be verdict-only: read `history://<id>` for its yields and any sub-review findings it discarded before concluding.
- The web reviewer may REFUSE source-edit mutation checks ("procedure prohibits source edits"). The Go reviewer can use `go test -overlay` scratch files instead. For web mutation checks, run them yourself: back up the file to /tmp, patch, run the targeted vitest, restore with `cp`, confirm `git status --short` is clean.
- 3 related asset-api PRs (e.g. #200/#202/#205): 2 reviewers is enough — one per independent PR, one for a stacked pair. Write PR bodies to /tmp/prN-api-body.md for them.

## 1. Worktrees, never checkout
```bash
git fetch origin pull/<N>/head:pr-<N> --force
git worktree add /tmp/pr<N>-api-wt pr-<N> --detach
git merge-base pr-<N> origin/main   # drift check
git log --oneline <mergeBase>..origin/main   # what main gained since
```
Read base content with `git show <baseSha>:<path>`; NEVER switch worktree contents with stash/checkout for comparisons. Reuse existing /tmp worktrees only after confirming HEAD and merge-base match. For a web typecheck baseline, also add `/tmp/web-base-wt` at the merge-base.
Stacked PR (base branch ≠ main): review its diff against the parent PR head (`git diff pr-<parent> pr-<N>`), and note the parent relationship.

## 2. Map, then fan out
`gh pr view <N> --json title,body,files,additions,deletions,baseRefName,headRefOid`. Fan read-only slices to subagents, one per repo for a pair. Give each the worktree path, base SHA, the no-GitHub-writes rule, the sibling worktree path (read-only, for contract checks) and a `local://prNNN-review.md` report target. Re-verify subagents' headline claims yourself at file:line before reporting. Files like `/tmp/pr-NNNN-*.md` are usually PR descriptions, not reviews.
Before dispatch, probe the DDM stack (`curl localhost:808{1..4}/health`) so reviewers know whether a live smoke test is possible.

## 3. Verification
Go: `go build ./...`, `go vet ./internal/...`, targeted `go test -count=1` on changed pkgs, `-race` on pkgs with concurrent fetch/cache. DB integration tests failing without Postgres/TLS are pre-existing (router/database tests use testcontainers and do run locally when Docker is up). Note when reviewers piped output through `tail` — exit status then isn't proven.
Web: if package.json/lockfile changed, `npm ci` in the worktree first (stale node_modules shows as `npm ls` invalid/extraneous), then `npm run test`, `npm run build` (`NODE_OPTIONS=--max-old-space-size=8192`), and `npx nuxi typecheck` with a base error-count comparison (baselines: 185 for #1351, 200 on main at 407211a). CI has no typecheck; lint is pre-existing broken (ESLint 9 has no flat config).
**Build hang:** `npm run build` in the main asset-web checkout can hang in the Nitro server stage while `npm run dev` on :3000 shares `node_modules/.cache/nuxt`. Build in a clean `/tmp` worktree instead; after fix commits, `git -C /tmp/<wt> checkout --detach <sha>` and re-run build + typecheck there. Don't skip the build.
Opt-in live fixture generators (`DDM_LIVE_TEST=1`) often hardcode dataset UUIDs that the local DDM no longer has. Report this as stale, not as a regression.

## 4. Producer-side grounding
For consumers switching on producer literals, enumerate the producer's emission sites. For DDM-backed features, ground against `~/Sites/afed/tricipta-production-ai-ddm-api/app/*.py`, not only OpenAPI. The new-well endpoint is `app/spatial_api.py:99` (in-memory, creates no jobs). The Bokor runtime (`bokor_spatial_runtime.py`) forecasts `*_effective_rate_bbl_per_day`, so rate × days-in-month cumulatives are valid.

## 5. Cross-repo pair checks (api + web)
- Contract: paths, query param casing (page endpoints use camelCase `startDate`/`endDate`/`broadStrings`/`datasetVersion`/`fieldId`), JSON keys, nullability, RFC 7807 `code`/`detail`/`upstream_detail`. Diff web fixture key sets against API DTO json tags; flag phantom web fields and fixture enum values the API never emits.
- Dataset pinning: a derived request (insight, feedback) must carry the `dataset_version` the page displays. Otherwise the API resolves its own and feedback can rate a different dataset than the one shown.
- Empty/degenerate states across the boundary (no run, zero pairs, observed-only ranges, 422 from DDM, no observed baseline → must not show a positive label like "Aligned").
- Store/composable races: abort controller cleanup without `ctrl === current`; missing generation guard ON THE SUCCESS PATH too (a run/retry that only checks seq in `finally` still writes a stale result); field/dataset change must bump the run sequence; tests that only switch key (so removing the sequence guard still passes). Add a same-key late-response test.
- Cache-hit metadata: check UI gates on fields that cache hits rewrite (e.g. `meta.source = "cache"`). Gate on a field preserved verbatim.
- Server cache keys: the key must include EVERY input that affects the rendered view, not just the upstream request hash. Example from PR #201: `well_type` (sidetrack vs recompletion) mapped to the same DDM request but was echoed into the view, so cached responses got the wrong label.
- HTTP caching: ETag built from IDs only + If-None-Match checked before the dataset is validated/fetched → 304 for nonexistent resources (`If-None-Match: *` makes it trivial). Strong ETag shared by gzip and identity bodies, and `strings.Contains(Accept-Encoding,"gzip")` ignoring `q=0`, are sibling-wide patterns: note them, don't count them as PR findings.
- Warning/state switches: a `switch` that makes independent warnings mutually exclusive drops the second warning.
- URL query state (Nuxt DDM tabs): (a) tab-leave cleanup that deletes a key SHARED with another tab (e.g. `datasetVersion`, which Connectivity reads) drops that tab's pin; (b) `{...route.query, ...formToQuery()}` merges keep stale optional keys (`modelVersion`, `parentWell`) when a later run omits them. Grep every reader of each deleted or merged key.
- Deploy: new `secretref:` entries need secrets provisioned per Container App.

## 5.5 LLM insight gates (insight services)
- Number gates miss contradictions written without numbers. Check for a direction gate: normalise words to rising/falling/flat, compare 3-way (not a rising↔falling-only helper), skip non-oil subjects (liquid/water/gas), and use whole-word cues so "risk" isn't read as "rise".
- Repaired or fallback outputs must not be cached. Prompt/gate changes must bump the PromptVersion that is part of the cache key.
- Labels must match content: "AI" provenance only when an LLM-authored sentence is actually rendered.

## 5.6 OTel / logging parity (asset-api)
Auto-instrumented: otelecho + request log via `observability.Install`, outbound HTTP via `observability.HTTPTransport()`, redisotel, the GORM tracing plugin, and LLM `llm.generate` spans. Handlers must log through `observability.LoggerFrom(c)`. Service-layer logs on an injected `zap.L()` reach the otelzap bridge WITHOUT trace correlation unless they add `observability.ContextField(ctx)` (+ `trace_id`). Flag completion/summary logs that lack it.

## 6. Live smoke
The local DDM stack (:8081–:8084) can be driven with a throwaway Go harness wiring the real `ddm.Client` + service (a `go test -overlay` scratch file keeps the worktree clean). Reproduce suspected cache collisions live this way. Report any DDM runs or jobs created as side effects.

## 7. Route-removal PRs: swagger/router parity
Load swagger.json and swagger.yaml from git on both sides and compare their (METHOD, path) op sets SEPARATELY — they can diverge (PR #200: JSON+docs.go dropped a still-registered `/insight/dashboard-diagnostic/{field_code}` that YAML kept, so the body's "43 removed" was 44 in JSON). Verify the PR body's op counts. Dump `e.Routes()` via a throwaway test on head and base, diff against swagger, then delete the dump test. Swagger/docs.go conflicts with drifted main are routine: "regenerate after rebase".
Consumer check: asset-web `Header.vue` (default layout, all branches) calls `/v1/notifications*` and opens `/api/v1/ws`; UAT `data-entry.vue` calls area/chemical-usage/esp-monitoring/stock-oil/well-tests. Removing those routes needs a coordinated web PR.

## 7.5 Table-drop / AutoMigrate / seeder cleanup PRs (asset-api)
- Migration numbers: `git ls-tree origin/main internal/database/migrations/` — a new `0000NN_*` may collide with one main added since merge-base (PR #202 vs `000060_ast_insight_feedback`). The runner in `migrations.go` calls migrations by function name, not sorted number, so the collision is a merge conflict + renumber, not a proven runtime skip — don't overstate.
- Fresh-DB proof: run `database.Initialize` on a testcontainer Postgres for base and head, compare `public` table lists; every missing table must be on the drop list.
- Ordering hazard: if the table-drop PR lands before the route-removal PR, still-registered handlers 500 on a fresh DB (no AutoMigrate) — reproduce with a signed-in overlay router test. Also the old seeder (`SeedGORM` → queries `countries`) breaks on the intermediate head.
- Seeder replacement: check first-admin bootstrap (OIDC provisioning refuses users absent from `users` → `ErrUserNotRegistered`, so "provisioned from Keycloak on first login" comments are false), referenced docs actually exist (`docs/iam-runbook.md` was missing), and steps that swallow errors (`SeedFieldsAndPlatforms` only logs; wrapper returns nil).
- Schema guard tests that run with the drop flag on can't catch a reintroduced legacy model — mutation-check by adding a model back.
- Irreversible down migration (RAISE EXCEPTION): acceptable, but tell operators to back up before enabling the drop flag.

## 7.6 Stacked PRs: merge order ≠ deploy order
A PR whose base is another PR's branch, merged "first", lands in that branch, not main. State both: branch merge order and safe deploy order (e.g. route removal #200 → then #202 carrying #205).

## 8. Cross-repo consumer impact — per deployed branch
`git grep` the pattern on origin/main, staging and uat. Check the ancestry of each deployed branch.

## 9. Test quality
For each new test, ask whether it would fail if the fix were reverted. Mutation-check key guards in scratch copies, e.g. `break`→`continue`, or removing a sequence or identity check.

## 10. Cleanup, then verify
Remove only the worktrees YOU created (`git worktree remove --force`), run `git worktree prune`, delete the `pr-<N>` branches and temp files, then re-check `git worktree list` and `git status --short`. Leave other /tmp worktrees and dirty files that predate the review untouched, and say so in the report.

## 11. Report shape
Verdict first → ship/merge order if stacked → per-PR findings table (severity, file:line, evidence; raise severity when the finding breaches a spec line) → verification → test quality → spec/doc drift → cleanup/side effects.
