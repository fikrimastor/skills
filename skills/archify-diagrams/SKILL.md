---
name: archify-diagrams
description: "Create validated interactive HTML architecture/workflow/sequence/dataflow/lifecycle diagrams with the locally installed Archify tool (~/.omp/tools/archify). Use when asked for an architecture diagram, system map, workflow, sequence, or dataflow visualization."
---

# Archify diagram authoring (local install)

Tool lives at `~/.omp/tools/archify/archify` (git clone of tt-a1i/archify, pure Node, no install step). CLI: `node ~/.omp/tools/archify/archify/bin/archify.mjs`. Sanity check: `node ~/.omp/tools/archify/archify/bin/archify.mjs doctor`. `inspect` / `validate --layout-json` return geometry for architecture diagrams ONLY.

## Workflow

1. Determine diagram type: architecture | workflow | sequence | dataflow | lifecycle.
2. Read the matching schema (`schemas/<type>.schema.json`), `schemas/common.schema.json`, and ONE matching example (`examples/<type>*.json`). Read only those files.
3. Ground facts first: for real systems, verify topology from code/config (docker-compose, .env, routers, proxy routes) with file:line evidence — never invent edges. Re-verify scout-reported line numbers yourself; scouts drift by 2-10 lines.
4. Author the candidate JSON: `meta.quality_profile: "showcase"`, English copy, omit subtitle/visual_preset/engineering_profile/animation/brand unless asked, ≤12 nodes, sparse semantic edge labels, cards for supporting detail. Add `meta.legend.entries` only to fix a misleading default label (see below).
5. Validate loop after EVERY edit: `node bin/archify.mjs validate <type> <candidate.json> --quality showcase --json` — target 9/9 checks, 0 errors, 0 warnings. Apply diagnostics by code/subject/supportedFixes; max ONE geometry control per repair round.
6. Deliver: `node bin/archify.mjs deliver <type> <candidate.json> <output.html> --quality showcase --json` (atomic; non-zero exit = failure). Hash in `artifact.sha256`.
7. Visual evidence: `node bin/archify.mjs visual-check <output.html> --json`. Then actually inspect a screenshot with an image reader; report truthful visual_review status.
8. Content audit without vision: HTML-entity-decode the delivered HTML and assert every spec label/sublabel/tag/card item/legend label/view note is a substring. Also re-read every card claim against source — the validator never checks truth.

## Known gotchas

- Component `sources[]` triggers the repository-evidence gate: requires `meta.repository` {url, revision} pinned to ONE public GitHub repo + `--repo-root` to a git-verified checkout of that repo. For multi-repo systems, omit `sources` (put verified facts in labels/sublabels) or the validate/deliver hard-fails.
- Desktop overflow (scrollHeight > innerHeight at 1440x900 etc.): repair by compacting spacing or removing redundant content — never shrink nodes or hide overflow. Re-validate + re-deliver after any change; hashes change.
- Do not read renderer internals (geometry.mjs etc.) before the first candidate; only for unresolved diagnostics.
- `visual-check` exit 0 with `ok:false` still means a must-fix overflow. `containment.viewports` lists LIGHT theme only; overall `status` also fails on `captures.screenshots[*]` DARK captures, which render ~10px taller at 1440x900. Read both lists.

## Dataflow geometry (measured 2026-09-23, archify 2.15.0)

- Stage centres are FIXED at x = 100 + 215·stage. Verified identical node rects at viewBox widths 1080, 1400 and 1500 — widening the viewBox does NOT widen inter-stage gutters, it only adds blank right margin and shrinks text. A 5-stage dataflow wants viewBox width 1080.
- Row centres at y = 157 + 114·row; default node 112×58, so the vertical gap between rows is 56px — room for one 16px label band.
- Label width ≈ 6.2 px/char at node-label size, ≈ 5.6 px/char for flow labels, ≈ 3.6 px/char for sublabels. Default node width 112 fits an 18-char label; 136 fits 21; 150 fits 23. Tags are NOT drawn in the box (hover/passport only).
- Inter-stage gutter = 215 − (wA/2 + wB/2). With 150-wide nodes it is 65px, too narrow for most horizontal flow labels (label-overlaps-node error). Recipe that validated first try with 23-char method names:
  - keep every same-row flow horizontal and every same-stage flow vertical; allow diagonals only where a corridor has free row gaps (the renderer puts the label mid-corridor, inside the row gap);
  - horizontal flows between wide nodes: `labelDy: 53` (label below the arrow, in the row gap) — the validator suggests this exact value;
  - a side exit that must drop to a lower row: `fromSide: "bottom", toSide: "left"` (clean L through an empty cell);
  - two flows leaving the same side of one node create port offsets → micro-segment and label-clearance errors; re-source one of them (e.g. from a config node) instead of fighting routes.
- Card grid is 3 columns up to 1920px: a 4th card wraps to a new row and breaks containment at 1440x900. Use ≤3 cards, ≤3 items each, items ≤80 chars.
- viewBox height 540 passes all 8 visual cells for a 3-row, 5-stage dataflow with 3 short cards (600 → 560 → 540 was needed once when cards were long).

## Legend defaults can lie

Dataflow default legend labels: emphasis "primary data", dashed "async batch", database "data store", default "data flow". If dashed means optional/side input, "async batch" is wrong — relabel via `meta.legend: {mode: "auto", entries: {dashed: {label: "side input"}, security: {label: "early exit"}, ...}}`. Adding it passed validate 9/9 and all visual cells unchanged in two diagrams.

## Sequence notes

- The SVG scales to a ~960px content column at 1440, so a 1560-1600 wide viewBox gives small message text — the price of passing containment (narrower viewBox = taller page). `column_fit: "spread"` spreads participants.
- Message `note`s are not drawn by default (hover/lens only); put must-see facts in labels or cards.

## Handoff receipt format

diagram_type / output path / specification_sha256 / artifact_sha256 / validation N/N / visual_review / correction_rounds (≤2).
