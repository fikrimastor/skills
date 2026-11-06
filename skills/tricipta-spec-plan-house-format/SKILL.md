---
name: tricipta-spec-plan-house-format
description: "Write the Tricipta house-format design spec + implementation plan (+ Archify workflow materialization) for an issue before coding, review that they align, and narrow scope correctly when part of the work is deferred to another developer"
---

# Tricipta house-format spec + plan (documents before code)

## When
Any non-trivial Tricipta issue/feature/bug, especially spanning repos (`tricipta-edafy-asset-web`, `tricipta-edafy-asset-api`, `tricipta-diagnostic-plot-api`, `tricipta-uiux-asset`). The user expects documents first — ad-hoc implementation gets reverted ("revert changes, i need to have a spec and implementation").

## Artifacts and paths
- Spec: `~/Sites/afed/tricipta/docs/superpowers/specs/YYYY-MM-DD-<slug>-design.md`
- Plan: `~/Sites/afed/tricipta/docs/superpowers/plans/YYYY-MM-DD-<slug>.md`
- Workflow materialization (workspace rule, `tricipta/AGENTS.md`): same basename + `.workflow.json` / `.workflow.html` + `.workflow.visual-check.*`, authored with the local tool `~/.omp/tools/archify` (skills `archify-diagrams`, `archify-workflow-lane-budget`). The binary may not be on PATH — invoke `node ~/.omp/tools/archify/archify/bin/archify.mjs validate workflow <json> --quality showcase --json` (expect 0 errors/0 warnings) and `visual-check <html> --json` (containment pass at 1440x900 and 2048x1320, light and dark).

## Spec structure (keep the order)
1. Title block: Issue link, Date, `Status: Draft for review`, Repos, Design source.
2. **Goal** — which issue bullets are satisfied; state module/surface scope up front.
3. **Current state (verified <date>)** — inventories with exact `file:line` and a `This change` column per row (endpoints, web call sites, components).
4. **Scope** — in scope / out of scope; out-of-scope must name the exact files someone else owns. List every generated artifact the change produces (swagger `docs/**`, helper scripts) or A10's diff-scope proof cannot pass.
5. **Decisions** table — Choice + Rationale per row; add considered alternatives for load-bearing choices. Include the small stuff that bites (e.g. prototype-only Tailwind classes such as `shadow-3xs`/`py-0.2` that do not exist and must be remapped).
6. **Target design** — contract tables (old → new), component inventory + frozen interfaces, data-section mapping from real result types (never prototype mock numbers), visual tokens. For a device-visible card, map each DOM region (badge, description, tiles, action line) to its source — audit the real component, do not assume.
7. **Risks** + mitigation.
8. **Acceptance criteria** — executable (grep counts, 404 assertions, build/test commands, browser checks).
9. **Open items for review** — the decisions the user may flip cheaply.

## Plan shape
- Header: Goal / Architecture / Tech Stack / Spec path.
- **Global Constraints**: repo + branch + head SHA, unrelated uncommitted files that must never be staged, do-not-touch list, frozen identifiers, "skip formatters/linters/project-wide suites, run only the task's targeted tests, commit per task".
- One task per coherent slice with: **Files** (exact paths + line refs), **Interfaces** (Consumes / Produces), checkbox **Steps**, **Verify** (exact commands + expected counts), **Commit** (exact message).
- Order: contract/API slice → callers → UI slices → verification/docs/PR.
- Subagents implement one task each; the plan is the contract they are handed, so a task must be self-sufficient: every code step contains complete compiling code, and every changed exported signature lists ALL call sites (including tests) in its Files.

## Scope narrowing (when part of the work is deferred)
1. Spec §2 tables mark each row in-scope vs `deferred — untouched`.
2. Spec §3.2 names the deferred files and requires them byte-identical.
3. Spec §5.1 splits "migrated now" from a **frozen handover contract** (future identifiers + registration pattern for the next developer).
4. Plan global constraints repeat the do-not-touch list; the verification task adds a `git diff origin/main --stat` proof that no deferred file changed.
5. Regenerate the Archify workflow so deferred work sits in its own lane with zero edges into the main path.

## Review loop (do this before handing the plan to executors)
1. **Freeze the input.** Read `proc://` and kill leftover agents; a subagent reported "completed" can still be live and keep committing to the same doc (seen: one silently reverted an earlier fix). Record the plan SHA and review that SHA.
2. Dispatch read-only reviewers split by slice (API tasks vs web+verification+workflow), each told: verdict `aligned | aligned-after-fixes | not aligned`, findings table with `severity / plan line / spec ref / exact fix`, and an AC→task coverage matrix. Tell them which deviations are intentional so they are not re-litigated.
3. Apply fixes with **one** agent per file (never two on the same doc), using enumerations from the report, not prose.
4. Re-verify with a fresh reviewer on the fix commit, and audit `git diff <pre-fix-sha> <fix-sha> -- <path>` hunk by hunk — commit messages stop describing their contents after a few rounds.
5. Only then offer execution (subagent-per-task vs inline).

## Verification traps that repeatedly bite
- `swag init -g cmd/server/main.go --output docs` — regenerate after every DTO/handler change and commit `docs/docs.go|swagger.json|swagger.yaml`, or CI flags stale swagger.
- `jq` precedence in A7-style equality checks: `A[:n] | map(x) == B[:n] | map(x)` pipes a boolean into `map`. Use `jq -s -e '((.[0]....)[:$n] | map(.well_string)) == ((.[1]....)[:$n] | map(.well_string))'` over two temp files.
- Env-var defaults make "unset ⇒ omit" dead code; state one rule in the spec (unset ⇒ default, omission only at an explicit sentinel such as `=0`).
- Go DTO vs TS type drift: diff the frozen response shape field-by-field (names, nullability, per-slot provenance, `[]` vs `null`) between the API task and the web types task.

## Gate
Present spec + plan (+ diagram) and wait for explicit approval before executing; do not start coding while the spec says `Draft for review`. Only the user can accept the open items.
