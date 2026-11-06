---
name: verify-tricipta-spec-claims
description: "Verify a Tricipta house-format spec/plan's cited file:line claims against real source before implementation — per-repo read-only scouts, VERIFIED/WRONG/IMPRECISE contract, and the enumeration-gap hunt. Use when asked to review, bless, or sanity-check a spec/plan in tricipta/docs/superpowers before coding."
---

# Verify a Tricipta spec/plan's factual claims

Use when asked to review/bless/implement a spec or plan in `tricipta/docs/superpowers/`.
Goal: does the source actually say what the spec claims? NOT whether the design is good.

## Path trap

Specs/plans live in the **tricipta workspace root**, not the target repo:
- `~/Sites/afed/tricipta/docs/superpowers/specs/<date>-<issue>-<slug>-design.md`
- `~/Sites/afed/tricipta/docs/superpowers/plans/<date>-<issue>-<slug>.md`
- Archify: `<slug>.workflow.json/.html`, `.workflow.visual-check.{html,json,png}`, `.workflow.finalize-summary.json` (check `gates` all pass with one `jq`)

Reading the same path under `tricipta-edafy-asset-web/` fails with "path not found".

## Procedure

1. Read the spec fully (they are long — 200+ lines, claims live in `§2` tables with `file:line`).
2. Check for a companion plan and whether spec decisions (D1..Dn) map to plan Tasks.
3. Dispatch **one read-only scout per repo** touched, in ONE `tasks[]` batch. Each brief:
   - lists the specific claims to verify, one per numbered item, each naming its cited `file:line`
   - requires the contract: `spec claim (cited file:line) | VERIFIED/WRONG/IMPRECISE | actual source line quoted`
   - says to flag a wrong **line number** even when the substance is right
   - tells them to read the stated ref via `git show <ref>:<path>` (checkouts are usually on another branch)
   - tells them to hunt for what the plan's enumeration would MISS
4. **Spot-check the highest-stakes scout claim yourself.** Scout citations can be wrong:
   - one cited `landing.vue:77-79` for `definePageMeta` when the real line was 146.
   - (#1351) a scout claimed "latest migration is 000058" — `git ls-tree origin/main internal/database/migrations/` showed 000059 exists. One `ls-tree` refutes it.
   - (#1351) the web scout had **no shell**, so every branch-only file came back "per spec, not line-verified". If a scout reports it couldn't `git show`, read the branch files yourself with `git show <sha>:<path> | cat -n | sed -n 'A,Bp'` — don't re-dispatch.
5. Report: verdict, defects to fix before implementing (ordered by whether they break execution),
   design risks with fixture evidence, citation fixes, what checked out.

## Scout brief must include

- `READ-ONLY. No edits, no builds, no test runs, no linters, no branch switches.`
- **Secret-safety:** "Do NOT run unscoped repo-root greps — they sweep `.env` and leak credentials
  into the transcript. Scope to `internal/`, `cmd/`, `app/`, `server/` and add
  `--exclude=.env --exclude=*.env`. Never print a secret value; report length or set/not-set only."
- `Mark anything you cannot confirm as [UNVERIFIED] rather than guessing.`

## Highest-value output: enumeration gaps

Specs often add a default-deny rule plus a small allow-list and tell the implementer to "enumerate the rest".
The valuable finding is what that enumeration misses. For Nuxt, glob `app/pages/**/*.vue` and check each
top-level page's `definePageMeta` for `middleware: 'auth'`/`'admin'` — path-prefix middleware does NOT cover
top-level pages (#908: `landing.vue`).

## Recurring Tricipta traps (check every time)

- **"Fixture not in repo" claims:** `git ls-tree -r --name-only <ref> | grep -i <slug>` — asset-web commits
  fixtures under `test/fixtures/` (e.g. `production_forecast_view.json` on the #1206 branch). Then profile it
  with one `jq` (ids, status, null counts, horizons, derived ratios) and test every threshold/finding rule against
  real values — #1351's 3.0 P90/P10 threshold fired on every forecast string (pill permanently Critical), and the
  fixture had no `observed_only` string for the D15 path.
- **"Page runs on a mock" claims:** grep the composable/store for mock/fixture; often the page only calls an unbuilt
  endpoint, so live-verify ACs are impossible without a stub.
- **asset-api migrations are NOT golang-migrate:** a new SQL pair must also get consts + `Up*/Down*` funcs in
  `internal/database/migrations/migrations.go` and an invocation in `internal/database/connection.go`; otherwise it never runs.
- **`apperrors` lives at `internal/apperrors/`**; `New*Error(msg)` sets `code` = category (`NOT_FOUND`) — custom codes
  like `string_not_found` need a new helper.
- **Router unconfigured convention is split:** bubble-map leaves the handler unregistered (404) though its log says
  "will return 503"; diagnostic-plot always registers and 503s. Make the spec pick one.
- **`ResolvedClaims.UserID`** = JWT `sub` only for Keycloak tokens; legacy tokens give a DB UUID.
- **`cfg.InsightCacheTTL`** is env `DIAGNOSTIC_PLOT_INSIGHT_CACHE_TTL` (shared).
- **`llmsvc.Ask` already logs model/latency/tokens under `cmd.Tag`**; fake it via `llmsvc.Client`. Hard-coded LLM
  timeouts make timeout tests slow — require an injectable timeout.
- **asset-web has no `typecheck` script** (scripts: build/dev/generate/preview/test/lint/format).
- **Flex layout wraps:** if a plan wraps an existing card in `flex flex-wrap`, check the card has `flex-1 min-w-0`.
- **LLM "number gate" specs:** a `-?\d+(\.\d+)?` regex splits `1,234.5`, reads `P90/P10`, dates (`2026-05` → `-05`),
  sign-flipped magnitudes and ×100 percent forms as foreign numbers — check the spec's own "untouched" example passes.
- **Response contract vs UI:** fields the panel renders (e.g. `detail`) must be in the response or explicitly read from a prop.

## Also check

- Acceptance criteria that **cannot pass as written** (grep/count ACs contradicted by live code; live ACs with no data source).
- Citations to comments that do not say what the spec claims (attribution drift between files).
- Whether a "blocking" open item is genuinely blocking.
- Internal spec↔plan contradictions (widths, counts like "4th copy" — grep the actual count).
