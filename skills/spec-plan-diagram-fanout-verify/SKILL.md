---
name: spec-plan-diagram-fanout-verify
description: "Fan out parallel subagents to materialize a Tricipta (or similar house-format) spec+plan into an Archify threat-model/architecture diagram, a workflow pipeline, and a visualize dashboard — then independently re-verify every artifact instead of trusting the subagent receipt. Use when asked to visualize a plan, diagram the threats if a plan is NOT implemented, or produce diagram dashboards for a security/feature spec."
---

# Spec/Plan Diagram Fanout + Verify

Turn a reviewed spec + implementation plan into three visual artifacts via parallel subagents, then verify them yourself. Grounded in the Tricipta `#908` run (2026-09-16).

## Layout contract (freeze it before dispatching)

All artifacts go in the **workspace root**, never inside the target product repo:

| Artifact | Path |
|---|---|
| Threat/architecture diagram | `docs/superpowers/specs/<date>-<issue>-<slug>-threat-model.architecture.json` + `.html` |
| Workflow pipeline | `docs/superpowers/plans/<date>-<issue>-<slug>.workflow.json` + `.html` |
| Plan dashboard | `docs/superpowers/plans/<date>-<issue>-<slug>.visualize.html` |

Archify writes `visual-check.*` PNG/JSON/HTML sidecars itself — allow those, forbid every other new file.

## Fan out (2 concurrent max, one `tasks[]` batch)

One brief per artifact, each self-contained:

- **Tool path**: `node ~/.omp/tools/archify/archify/bin/archify.mjs`. Authoring rule: read `schemas/<type>.schema.json`, `schemas/common.schema.json`, and ONE matching example, then match that example's shape. Do not read renderer internals.
- **No `sources[]`** for any multi-repo system — the repository-evidence gate needs one public GitHub repo + `--repo-root`, so it hard-fails. Put `file:line` in `sublabel`/`tag`/cards instead.
- **Grounding mandate**: confirm every drawn edge against current source with grep/read before drawing it; drop or correct any claim that no longer holds and report the discrepancy. Never draw an invented relationship.
- **Loop**: `validate <type> <json> --quality showcase --json` → 9/9, 0 errors, 0 warnings, ≤1 geometry control per repair round → `deliver` → `visual-check` → open one saved PNG with an image reader.
- **Visualize dashboard**: copy `~/.agents/skills/visualize/examples/template.html`, fill slots, delete the Mockup section if no UI design was discussed, and *render it* (serve + screenshot) rather than asserting it renders.

## Content requirement that actually mattered

A threat diagram must model the plan's **residual** state, not fixed-vs-unfixed. Enumerate, per threat: which decision/task closes it, which threat **survives** because a task is blocked on an open question, and which mitigations are **out-of-band** (ops actions, pipeline defects) that no PR covers. A diagram that implies the plan closes every path misstates the risk posture.

## Verify yourself — never accept the receipt

1. `shasum -a 256` every artifact vs the subagent's reported hashes.
2. Re-run `validate` on both JSONs; parse the JSON `ok` + count `checks[]` (exit code alone is not the signal).
3. Threat diagram: assert no component carries `sources[]`; print components/connections/cards and check each edge label against the spec.
4. Dashboard: `grep -c '{{'` → 0; `grep -nE 'https?://|<link |<script src|@import'` → none; count `<h2>` sections, `.step`, `.risk`, open checkboxes.
5. Read one PNG per diagram to judge legibility and confirm no clipped text.
6. `git status --porcelain` — prove only the contracted paths changed and pre-existing dirty files (check mtimes) were not swept in.

## Do not gate on containment

`visual-check` containment fails whenever a `cards` block exists — archify's own bundled examples fail it identically (workflow 2052/900, dataflow 1312/900). Target `validate` 9/9 + `deliver` exit 0, then report the residual `scrollHeight` per viewport honestly. Never shrink viewBox, nodes, or cards to chase a zero the reference artifacts never reach.

## Receipt to report

Per artifact: diagram_type, path, specification/artifact sha256, validation N/N, truthful visual_review (state containment result even when it fails), correction_rounds (≤2). Plus the grounding table (claim → file:line, or DROPPED and why) and a note that nothing was committed unless asked.
