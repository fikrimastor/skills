---
name: archify-dark-theme-containment-overflow
description: "Diagnose and fix Archify containment overflow that appears only in the dark theme at 1440x900, where the reported containment.viewports list omits dark rows and the real cause is container height, not title or label wrapping."
---

# Archify dark-theme containment overflow

Applies to any Archify type (dataflow, workflow, architecture) where
`visual-check` reports `containment.status: fail` at the smallest viewport.

## Trap: the failure is dark-theme-only and easy to misread

`visual-check --json` puts light rows in `containment.viewports` and dark rows
only in `captures.screenshots`. A dark-only overflow therefore shows
`containment.status: fail` with an apparently all-`ok:true` viewport list.
Always read the dark rows from `captures.screenshots` before concluding which
viewport failed.

## Trap: editing copy does not change container height

Measured on a 5-stage showcase dataflow: `scrollHeight` at 1440x900 was **928px
before and 928px after** shortening the title. Title/label/sub-label edits are
irrelevant to containment — do not spend repair rounds there. Confirm with a
number, not a hunch: re-run `visual-check` and compare the exact `scrollHeight`.

## Fix order (one geometry change per round)

1. Reduce `meta.viewBox` height, e.g. `[1500, 700]` -> `[1500, 620]`. This is the
   reliable lever; it compacts the whole layout.
2. If still failing, compact grid spacing (gapX/gapY) or drop redundant content.
3. Never shrink nodes or hide overflow.

## Verify

After each single change: `deliver` then `visual-check`, and compare every
`scrollHeight` against `innerHeight` for both themes, not just the light rows.
Done means all four viewports report `ok:true` at both themes with
`visual-check` exiting 0.
