---
name: verify-rendered-svg-diagram-without-vision
description: "Verify a delivered Archify/HTML+SVG diagram is complete and legible when vision is rate-limited or unavailable — or as a cheaper independent check than eyeballing. Use after archify deliver, before reporting a diagram as verified."
---

# Verify a rendered SVG diagram without vision

Turns "the subagent said it looked fine" into measurements you can quote. Complements the containment guidance: containment (scrollHeight vs innerHeight) is NOT a useful gate for archify output — the tool's own bundled examples fail it — so legibility has to be established some other way.

## 1. Content coverage (JSON vs delivered HTML)

Every `participants[].label`, `messages[].label`, `cards[].title` and card item must appear in the HTML. **Do not compare raw strings only** — the renderer HTML-escapes, so a naive check reports false misses (hit this twice: `?code=…&state=…` becomes `&amp;`, and an apostrophe becomes `&#39;`). Accept a label as rendered if ANY of these is present:

```python
variants = {s, html.escape(s, quote=True), html.escape(s, quote=False),
            s.replace("'", '&#39;'), s.replace('&', '&amp;')}
```

Report `N/N` per category. A genuine miss means the deliver silently dropped a step — the failure this check exists to catch.

## 2. Geometry (real browser, not a screenshot)

Serve the directory first — the browser tool blocks `file://`:

```
hub op:"start" application:"python3" args:["-m","http.server","8125"] cwd:"<dir>" ready:{"port":8125}
```

Then in the page compare every text node's `getBoundingClientRect()` against the SVG's own box, and against each other:

- `outsideBox` — text extending past the SVG rect (left/right/top/bottom, ±1px tolerance) = clipped content. Must be 0.
- `overlappingPairs` — two text nodes overlapping >2px on BOTH axes = colliding labels. Must be 0.
- count lifeline/participant labels present.

Always close the tab and stop the server afterwards.

## 3. Visual order (the thing that actually breaks)

Sequence diagrams fail by *ordering*, not by pixels. Read it out of the DOM: sort text nodes by `top` (then `left`), group into rows where `|Δtop| < 6px`, join each row's items, and print them top-to-bottom. Compare that list against the intended message order in the JSON. This catches a redirect drawn as `API→Keycloak` when it should be `Browser→API→Keycloak`.

## Harness trap

In this browser tool **`tab.run(fn)` does not have DOM access** (it throws `document is not defined`). Pass a string expression instead: `tab.evaluate("(...)")`.

## Reporting

State the four numbers (coverage N/N, outsideBox, overlappingPairs, order verified) plus the containment numbers honestly — and say explicitly when a visual/pixel review was NOT performed because vision was unavailable, rather than implying you looked at it.
}
