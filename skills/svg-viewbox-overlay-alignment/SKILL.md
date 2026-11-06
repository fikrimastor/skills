---
name: svg-viewbox-overlay-alignment
description: "Use when a fixed-viewBox SVG chart with preserveAspectRatio inside a variable-size container has a sibling HTML overlay (legend, tooltip, badge) that needs to visually align with the SVG's actual drawn/letterboxed content, not the outer container's full box."
---

## Symptom

An SVG chart is rendered as `<svg viewBox="0 0 W H" preserveAspectRatio="xMidYMid meet" class="w-full h-full">` inside a `position:relative` wrapper. A sibling HTML element (legend, tooltip, badge) is `position:absolute` inside that same wrapper, sized/positioned against the wrapper's full box (e.g. `top-2 right-2 bottom-2`).

Whenever the wrapper's aspect ratio doesn't exactly match `W:H`, `preserveAspectRatio="xMidYMid meet"` letterboxes (blank margin top/bottom) or pillarboxes (blank margin left/right) the SVG's *drawn content* inside its own box — but the sibling overlay doesn't know that, so it visually runs past the chart's actual top/bottom/left/right edges instead of tracking them.

The same root cause silently breaks pixel math too: a naive `svgSpaceCoord / W * wrapperRect.width` conversion (e.g. for a hover tooltip) assumes no letterboxing and will be off whenever there is any.

## Wrong fix (don't do this)

Don't reach for a `ResizeObserver` on the wrapper that recomputes the "meet" fit math in JS (`scale = min(cw/W, ch/H)`, `fittedHeight = H*scale`, `top = (ch-fittedHeight)/2`) and pipes the result down to the overlay as a new prop. It works, but it's unnecessary machinery: a new prop surface, an async measurement pass (flash-of-wrong-size on first paint), and a second implementation of what CSS already does natively.

## Right fix: shared aspect-ratio box

Wrap the SVG **and** the overlay together in one shared box, instead of positioning the overlay against the wrapper directly:

```html
<div class="w-full h-full flex items-center justify-center"> <!-- was: wrapper directly held the svg -->
  <div class="relative w-full" :style="{ aspectRatio: `${W} / ${H}`, maxHeight: '100%' }">
    <svg :viewBox="`0 0 ${W} ${H}`" class="w-full h-full" preserveAspectRatio="xMidYMid meet">
      <!-- unchanged chart content -->
    </svg>

    <MyOverlay class="absolute top-2 right-2 bottom-2 ..." /> <!-- unchanged overlay component/classes -->
  </div>
</div>
```

Why this works: `aspect-ratio` + `width:100%` (or `w-full`) + `max-height:100%` is the standard declarative "`object-fit:contain`"-equivalent for arbitrary block elements (not just replaced elements like `<img>`/`<svg>`). The browser resolves it as: try `width:100%`, derive `height = width / ratio`; if that height exceeds `max-height`, clamp height to `max-height` and re-derive width from the clamped height. This correctly handles BOTH letterbox (container narrower than ratio) and pillarbox (container wider than ratio) cases via the same single rule — no JS, no measurement pass, no per-orientation branching. `align-items:center; justify-content:center` on the flex parent centers the box exactly like `xMidYMid` does inside the SVG itself.

The overlay component needs **zero changes** — its existing `top-2/right-2/bottom-2` (or whatever) positioning already resolves correctly once it's nested inside the new correctly-fitted `position:relative` box instead of the looser outer wrapper.

## Don't set both left+right unless you want full-width

If the overlay is meant to stay a narrow, shrink-to-fit, right-aligned column (e.g. a legend), keep only `top`/`right`/`bottom` set (no `left`). Setting all four sides (e.g. Tailwind's `inset-*`) with `width:auto` forces the element to *stretch* to fill the full width between left and right — only omit one of left/right when you want shrink-to-fit width.

## Verify

If a live browser isn't available, fetch the page's raw SSR HTML (`GET` the route) and check the rendered `style="aspect-ratio:..."` attribute has the exact expected `W / H` value, and that the tag nesting (wrapper → fitted box → svg + overlay) is structurally correct with no stray v-if/v-else mismatches. This confirms the CSS is wired correctly even without pixel-level visual confirmation.

## Related bugs to check for

Search the same file/component for any hover/tooltip pixel-position math that does `svgSpaceValue / VIEWBOX_DIM * wrapperRect.dimension` — that's the same unaccounted-for-letterbox bug in coordinate-conversion form, not just overlay-sizing form.
