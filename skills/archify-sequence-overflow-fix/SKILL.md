---
name: archify-sequence-overflow-fix
description: "Fix Archify sequence-diagram validation off-canvas errors and visual-check containment (vertical overflow) failures by adjusting viewBox, not node size. Use when authoring an archify sequence diagram and validate reports a message/segment 'outside the readable timeline/canvas', or visual-check containment is 'fail' with scrollHeight innerHeight."
---

# Archify sequence: off-canvas + containment overflow fixes

Two distinct failures hit archify **sequence** diagrams. Fix each by moving the viewBox, never by shrinking nodes/labels or hiding overflow.

## 1. validate: message/segment 'outside the readable timeline/canvas' (layout/constraint)
The renderer reserves bottom margin tied to viewBox HEIGHT:
- message `y` max ≈ `viewBoxHeight - 83`
- segment `to` max ≈ `viewBoxHeight - 45`
If your lowest message `y` or segment `to` exceeds those, validation errors. Fix: **raise viewBox height** so `height >= lowestSegmentTo + 45` (and `>= lowestMessageY + 83`). Messages need `y >= 160`.

## 2. visual-check: containment 'fail', overflowY true, scrollHeight > innerHeight
Root cause is counterintuitive: the SVG **scales up to the container width**, so a near-square viewBox (e.g. 900x860) upscaled to a 1440px desktop becomes very tall and overflows 900px height.
- **Fix: WIDEN the viewBox width** (height fixed). Wider viewBox => smaller upscale factor => shorter rendered height. This is the opposite of intuition (widening makes it *shorter* on screen).
- Behaviour is non-linear (there's a max-width/letterbox), so iterate empirically: change width by ~300-500px, re-deliver, re-visual-check, read the per-viewport scrollHeight vs innerHeight. The binding viewport is usually 1440x900.
- `column_fit: "spread"` distributes the extra width across participant lanes so a wide viewBox with few (e.g. 7) participants doesn't look sparse.
- Worked example: 7 participants, 13 messages needed viewBox width ~1560 (from 900) to pass containment at 1440/1600/1920/2048, with height 860.

## Loop
After EACH viewBox change: `validate` (target 9/9, 0 err/warn) -> `deliver` -> `visual-check --json`, then actually read a screenshot. One geometry knob per round. Content-accuracy (e.g. correct endpoint paths in labels) is NOT caught by the validator — verify labels against source manually.
