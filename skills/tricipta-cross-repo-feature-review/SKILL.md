---
name: tricipta-cross-repo-feature-review
description: "Review a feature spanning the Tricipta diagnostic-plot stack (plot-api Python + asset-api Go DDD + asset-web Nuxt) across its three feature branches, reconciling the API contract across repo boundaries."
---

# Tricipta Cross-Repo Feature Review

Use when a feature (e.g. #NNN) spans two or more of the Tricipta repos and you must review it as one system, not three isolated diffs. The value is reconciling the **cross-repo API contract** and the **compounding risks** no single-repo reviewer can see.

## Repo topology
- `tricipta-diagnostic-plot-api` — Python FastAPI stateless engine (`POST /v1/analysis/*`). Strict Pydantic `extra=forbid` schemas.
- `tricipta-edafy-asset-api` — Go, Clean Arch + DDD. Review new code against `.github/copilot-instructions.md` (dependency direction, domain purity, tenant scoping, apperrors RFC7807, no `Save()`, `.Select("*")`, local handler interface). Calls plot-api and ddm-api; exposes REST to web.
- `tricipta-edafy-asset-web` — Nuxt/Vue SPA. Dashboards live in `app/pages/common-platform/*.vue`; DDM pages in `app/components/ddm/*`. Confirm paths with `git ls-files`.
- `tricipta-production-ai-ddm-api` — DDM upstream (spatial :8083 serves `/v1/sensitivity/results/{runId}`; pair fields produced in `app/sensitivity.py`). Its committed OpenAPI does NOT list pair fields (rank_in_field, vector_class…) — trace to `app/*.py`, not the openapi json.
- asset-chatbot is not wired into any insight feature (as of 2026-11); don't spend review time there unless grep shows a reference.

## Procedure
1. **Discover the work**: branch, `status --short`, `log --oneline -3` across roots. Feature branches are usually one commit → range `HEAD~1..HEAD`.
2. **Read the spec** `tricipta/docs/superpowers/specs/<date>-<NNN>-*-design.md` — §7 has executable acceptance criteria (A1..An) and §8 open items; review against them.
3. **graphify/graft first** per repo rules, then read the diff for line-level review.
4. **Fan out one reviewer per repo in parallel** (2-concurrent cap). Instruct each to DOCUMENT its side of each API boundary (route, request fields+units, response shape, error/empty semantics) so you can reconcile.
5. **Run verification yourself in the parent.** Reviewer subagents have repeatedly refused to run build/tests (claiming a /tmp-worktree-only restriction) and return "not run". Parent runs: asset-api `go build ./... && go vet <pkgs> && go test <pkgs>`; asset-web `npx vitest run <changed test files>`. Note: asset-web ESLint is broken repo-wide (no `eslint.config.*` under ESLint 9) — pre-existing, not a regression.
6. **Prove reviewer findings by execution**: for Go gate/regex claims, write a throwaway `zz_review_probe_test.go` in the same package calling the unexported function AND the top-level entry (e.g. `Gate` via existing test helper `gateSummaryViolations`), run `go test -run TestZZReviewProbe -v -count=1`, then `rm` it and confirm `git status` clean. Function-level positives can be masked by other checks — always test the full entry point.
7. **Parent reconciliation**: line up request/response field-for-field across seams; hunt compounding risks.
8. **UI parity** (if asked): delegate to an audit using `tricipta-design-parity-audit` skill; prototype lives at `tricipta-uiux-asset/tricipta-platform/src`. DDM tabs (incl. Sensitivity = tab key `diagnostics` in `pages/DDMDashboard.jsx`) share ONE generic DDM insight aside (~L589–741) — compare against it plus the merged sibling panel (e.g. `DdmForecastInsightPanel.vue`). Spec-driven content (sections/tables/footer) is an accepted deviation; judge chrome/layout.

## Output
Per-repo findings table (P0-P3, file:line, problem, proof, fix), verdict; reconciled contract; minimal merge-blocking fix list.

## Gotchas learned
- Prior P0s in asset-api were typed-nil-through-interface + redis fail-open — check `isNil()` guards and nil-client cache guards.
- LLM narrative gates (`sensitivityinsightsvc/gate.go`): nominal-form exemptions ("increase in/of injection") let modal recommendations through ("An increase in injection is recommended." passed in #1352). Probe nominalized + modal/passive-recommend phrasings ("is advised", "should be considered").
- Rounding allowlists that only accept exactly-N-decimals reject valid coarser roundings (0.288 → "0.3" rejected) → silent template fallback.
- Insight panels porting: check `companion-diagnostic-parity` root class (teal token override), frameless shell, no fixed max-height, 10.5px narrative, 2xl width 500px — #1352 missed all of these.
- plot-api engine may define a plot differently than criteria/prompts docs — flag spec↔engine drift.
