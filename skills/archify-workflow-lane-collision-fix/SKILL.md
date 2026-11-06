---
name: archify-workflow-lane-collision-fix
description: "Debug Archify workflow-diagram validation failures — same-lane adjacent-column node collisions, mainPath/trace-spine backward-column errors, and label-route-clearance overlaps. Use when archify validate workflow file.json reports 'less than 8px apart in lane X', 'moves backward from col A to col B', or repeated label-clearance failures that don't resolve with width/viewBox changes."
---

## Symptom

`archify validate workflow <file>.json --quality showcase --json` reports one or more of:

- `Nodes "A" and "B" are less than 8px apart in lane "L"` — for two nodes in the SAME lane at ADJACENT columns.
- `mainPath step "X" -> "Y" moves backward from col A to col B`.
- Repeated `composition/label-route-clearance` collisions that keep reappearing after each labelDx/labelDy nudge.

## Root cause 1: same-lane adjacent-column collisions are column-index-specific, not size-specific

This is a real archify layout-engine quirk, confirmed empirically: two nodes in ONE lane at adjacent column indices can collide regardless of node width (tested 60-118px) or overall `viewBox` width (tested 1500-2600px). The SAME lane can simultaneously have safe adjacent pairs at OTHER column-index boundaries (e.g. cols 2-3 and 4-5 fine, cols 3-4 broken) in the exact same diagram. Shifting the whole pair left/right (e.g. to cols 1-2) does NOT fix it — the specific index-boundary is broken, not the absolute position.

**Do NOT waste cycles on:**
- Shrinking node `width` (tested down to 60px, still collides if the index-boundary is bad).
- Widening `viewBox` (tested up to 2600px wide, no effect on this specific failure).
- `yOffset` vertical stacking to dodge horizontal crowding — undocumented for workflow node placement in this tool version; produces NEW errors (`collides with the title or boundary of lane`) rather than fixing the original one.

**Reliable fix:** give the second node its OWN dedicated lane instead of sharing the crowded lane with the first. If both nodes conceptually belong to the same service/system (e.g. "Asset API - Go"), split it into two lane rows with related labels, e.g. `"Asset API \u00b7 Go (Echo)"` and `"Asset API \u00b7 Go (dispatch)"`. Different-lane-same-column pairs have NEVER shown this collision in testing — only same-lane-adjacent-column pairs are affected.

After adding a new lane: place it LAST in the `lanes` array (bottom of the diagram) if the new node is downstream in the flow, so upstream edges (e.g. a `bottom-channel`-routed edge between earlier nodes) don't route through the new lane's row and get flagged as `clean-flow/edge-through-node`.

## Root cause 2: representing a "call, response, second call" (A calls B, B responds, A then calls C) pattern

`mainPath` requires: (a) every consecutive pair has a real matching edge, (b) columns are non-decreasing across consecutive mainPath steps (equal is fine, decreasing is a hard failure — error text literally says "use a return edge outside mainPath for loops").

This means you CANNOT put `[..., B, A, C, ...]` in mainPath if A's column is lower than B's (revisiting A after B is a backward step). If you need the trace/story animation to visit B (so the animated narrative doesn't skip the intermediate call), AND you need C's incoming arrow to originate from a distinctly-labeled "A-side" node (so the diagram doesn't visually imply B calls C directly) — you need a genuinely separate node for the post-return dispatch logic, positioned at column >= B's column. Do not try to reuse A's own node id at a later mainPath position (backward-column error) and do not accept a direct B->C edge with just a clarifying label (it still visually implies B calls C).

Pattern that works: `A(colN) -> B(colN+1)`, then a `role: "return"`, dashed edge `B -> A2` where `A2` is the new orchestrator node (same conceptual service as A, different lane per Root cause 1's fix, column >= B's column), then `A2 -> C` (the real dispatch edge) and `mainPath` includes `[..., A, B, A2, C, ...]` — all forward or equal columns, every pair has a real edge.

## Root cause 3: label-route-clearance whack-a-mole

When a label collides with a node or another edge's label/route, PREFER the validator's own suggested fix verbatim (it reports exact `labelAt: [x, y]` or `labelDy: +N` values in the error text) over guessing `labelDx`/`labelDy` offsets by feel — guessed offsets frequently just move the collision to a different node/edge, requiring several more rounds. Apply the suggested `labelAt` array directly as `{"labelAt": [x, y]}` on the edge.

## No geometry introspection for workflow diagrams

`archify inspect workflow <file>.json` and `archify validate workflow <file>.json --layout-json` both fail with `--layout-json is currently supported for architecture diagrams only` (exit code 2). There is no way to dump computed node/edge rects for `workflow` diagrams in this tool version — only `architecture` diagrams support that. Debugging workflow layouts must proceed purely from the validator's plain-text error messages and iterative small edits; do not spend time trying to find a JSON/rect dump for workflow types, it does not exist.

## Workflow

1. `archify validate workflow <file>.json --quality showcase --json`, pipe through a small Python snippet that prints `error` truncated to ~800 chars — full error blobs are verbose and repeat the same message across `error` and `diagnostics`.
2. Fix ONE class of error per iteration (structural node-position errors first, since fixing those changes all downstream geometry and invalidates label-position guesses).
3. Once `validate` returns `OK`/`ok: true`, `archify deliver workflow <file>.json --quality showcase` to write the HTML, then `archify visual-check <output>.html --json` and check `containment.status` across all 4 tested viewports (1440x900, 1600x1000, 1920x1080, 2048x1320) — a diagram can pass `validate` (internal SVG geometry) yet still overflow the outer HTML page if `viewBox` height is too tall for the smaller viewports. Fix by lowering `meta.viewBox[1]` (height) and re-delivering; this does not require touching node/edge geometry since it just rescales the whole diagram.
