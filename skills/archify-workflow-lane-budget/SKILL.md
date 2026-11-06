---
name: archify-workflow-lane-budget
description: "Decide how many lanes an Archify workflow diagram should have BEFORE authoring it, so fixing a same-lane node collision doesn't trade it for a containment (page-height) overflow failure. Use when planning an Archify workflow diagram for a plan/spec with more than ~6 conceptual steps, or when archify validate/deliver pass cleanly (N/N checks, 0 errors) but visual-check reports containment fail with scrollHeight far exceeding innerHeight at all tested viewports."
---

# Archify Workflow Lane Budget

Sibling to `archify-workflow-lane-collision-fix` and `archify-workflow-diagram-debugging` — read those first for collision/layout-constraint mechanics. This skill is about the decision to make *before* those fixes are needed: how many lanes to plan for.

## The trap

`archify-workflow-lane-collision-fix` correctly says: two nodes in the SAME lane at adjacent columns collide (`less than 8px apart in lane X`), and the reliable fix is giving the second node its own dedicated lane. This is true and works — `validate`/`deliver` will pass cleanly (N/N checks, 0 errors).

But applying that fix mechanically to every node in a multi-step plan (e.g. one lane per step in a 9-step implementation plan) produces a diagram that is honestly laid out but far too TALL. Lanes stack vertically; 9 lanes at ~140px each is ~1270px of SVG alone, before header/legend/cards. `visual-check` will then report `containment: fail` at every viewport (1440x900 through 2048x1320) with scrollHeight 2-3x the viewport height — a completely different failure class than the one you just fixed, and `validate`/`deliver` give no warning about it (containment is only checked by the separate `visual-check` command).

## The rule

Budget lanes by CONCEPT COUNT, not STEP COUNT:

- **≤4-6 lanes total.** One lane per phase, system, or actor (e.g. "Plot API", "Asset API", "Asset Web", "Verification") — not one per individual edit/test/fix within that phase.
- **One node per lane is the safe default** for a linear/sequential plan. It trivially satisfies the collision-fix's "different lane" safety rule with zero risk, and keeps total page height bounded.
- **Push itemized, step-level detail into the `cards` array**, not into more nodes/lanes. Cards render as a scrollable list below the diagram and don't affect the diagram's own SVG height or `layout/constraint` checks at all — they're the correct home for "5 sub-bullets of what actually changed in this phase", not 5 extra lane rows.
- If a genuine multi-node lane is unavoidable (rare), you can space same-lane nodes non-adjacently (gap of 2+ columns) instead of adjacent — but note 3 same-lane nodes with mandatory 2-column gaps need a 5-column span (e.g. cols 0,2,4), which combined with an already-used column range from a neighboring lane can exceed the 6-column max (`col` is 0-5 per schema) — do the arithmetic before committing to more than 2 nodes in one lane.

## Diagnostic signal

If `archify validate` and `archify deliver` both report a clean N/N pass with 0 errors/warnings, but `archify visual-check <output>.html --json` reports `"ok": false` and `containment.viewports[*].ok: false` with `scrollHeight` roughly 2-3x `innerHeight` across ALL 4 tested viewports (not just the smallest) — that ratio, uniform across viewport sizes, is the signature of "too many stacked lanes", not a fixable per-node layout tweak. Don't iterate on node widths, viewBox height, or trimming card text (card text only moves the needle a few percent — confirmed empirically, 1595px→1511px, ~5%, for trimming ~16 bullet items down to ~8). Re-author with fewer lanes instead.

## Honest reporting

Even a well-compressed 4-lane diagram with a `cards` section can show mild residual containment "fail" (page requires scrolling past the diagram+legend to reach the cards) — this is normal document scroll, not clipped or broken content. Confirm by actually reading a screenshot (`visual-check` saves PNGs) before deciding whether to keep compressing or to report the gap honestly. Never force it to zero via aggressive `viewBox` shrinking — that violates the tool's own anti-pattern guidance ("never shrink nodes or hide overflow") and produces an illegibly cramped diagram instead of a true fix.
