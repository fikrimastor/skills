---
name: tricipta-hard-route-rename-review
description: "Review a Tricipta hard route-rename / API namespace-migration PR (tricipta-edafy-asset-api or -asset-web, e.g. /module/insight - /insight/module): verify registration + middleware parity, swagger/doc consistency, stale literals, paired-PR lockstep deploy risk, and doc drift. Local summary only, never GitHub comments."
---

# When to use

A Tricipta PR (asset-api or asset-web) that *renames or relocates API routes* — hard rename, "no aliases" — typically part of an `/insight/{module}`-style namespace migration shipped as a paired web+api PR set.

# Contract of the review

- READ-ONLY. Never `gh pr comment` / `gh pr review` / any GitHub write. Deliver a local chat summary: verdict, findings by severity, each with file:line evidence.
- Verify the PR author's claims yourself where cheap (`go build ./...`, scoped handler tests, swagger path set) — the PR body's verification section is not evidence.

# Procedure

## 1. Metadata first (both PRs)
```bash
gh pr view <N> --repo AFED-digital/tricipta-edafy-asset-api --json isDraft,state,changedFiles,mergeable,mergeStateStatus,statusCheckRollup,reviews,comments
gh pr list --repo AFED-digital/tricipta-edafy-asset-web --state open --limit 20   # find the paired PR
```
`isDraft: true` on either half is a load-bearing fact for the deploy verdict. `changedFiles` can disagree with `git diff --name-only | wc -l` in a truncated listing — count files yourself.

## 2. In-repo audit (dispatch: `reviewer`; two independent slices in ONE parallel `task` call)
- **Registration + middleware parity.** Read the real group in `internal/router/router.go` (`api := e.Group("/api/v1")`, `protected := api.Group("")` + `JWTMiddleware`). Confirm the renamed handlers register on the *same* group as before (`git show main:<handler>`); `router.go` should not be in the PR at all. No wildcard `/insight/:module` may shadow the static routes.
- **Stale literal sweep**: `git grep` the old and new route literals across the whole repo (Go, docs, docker/, .github/, postman/scripts). Report zero-hit as the expected state.
- **Swagger 3-rendering consistency**: `docs/swagger.json`, `docs/docs.go`, `docs/swagger.yaml` must carry the same path set and `tags`; python one-liner over `swagger.json['paths']` is fastest; check for orphan tags.
- **Handler diff purity**: expect only route string + comments + `@Tags`/`@Router`.

## 3. The test trap (asset-api)
`internal/interfaces/http/handler/*_test.go` helpers `newEchoCtx` (bare `e.NewContext`) and `serveTest` (direct `fn(c)` call) **do not route** — the path string in those tests is cosmetic, so a wrong `RegisterRoutes` string passes CI. Route renames are therefore unguarded; say so, and recommend an `e.Routes()` assertion if the spec demands "old path must 404".

## 4. Cross-repo deploy risk (needs git + gh — do NOT send to a shell-less scout)
```bash
cd <sibling-repo> && git fetch origin --quiet          # stale remote refs silently return OLD content
 git grep -n -e 'old/literal' -e 'new/literal' origin/main
```
Answer explicitly: does the sibling's `main` (currently deployed build) still call the deleted paths? Is the paired PR merged/deployed? Impact severity depends on the consumer's 404 handling — check whether the failure renders visibly (error state) or is swallowed into a fallback.

## 5. Frozen contract / spec
`~/Sites/afed/tricipta/docs/superpowers/{specs,plans}/` — may be untracked drafts. Read the namespace/token table, the `Status:` line, the deploy-order mitigation row, and the acceptance criteria; a "ship both together, api first" note converts the deploy window into a *coordination* blocker rather than a design error. Check `gh issue view <N>` for any alias/deprecation requirement.

## 6. Doc correctness (PR-introduced vs pre-existing)
- **Anchor check**: for headings containing a backticked path, compute the GitHub slug locally — `re.sub(r'[^\w\s-]','',h.lower()).strip().replace(' ','-')` — slashes are DROPPED without a separator (`/api/v1/insight/diagnostic-plots` -> `apiv1insightdiagnostic-plots`). Compare against the TOC link; a regression here is a real, one-character PR finding.
- **Table completeness**: an endpoint table edited by the PR should list every route once, matching what handlers actually register.
- Separate **introduced by this PR** (should-fix/blocker) from **pre-existing drift** (one coordination note, do not enumerate as this PR's findings).

# Reporting shape

- BLOCKER: lockstep/deploy window + paired PR draft state, consumer-visible breakage with the 404-handling evidence.
- SHOULD-FIX: defects this PR introduced (broken anchors, stale rows in a table it edited, missing assertions where the spec demands proof).
- COORDINATION NOTE: pre-existing drift, spec-doc staleness.
- NOTE: absent test coverage of the change itself.
Verdict line up front, then a verification table (what was checked -> evidence), then findings.
