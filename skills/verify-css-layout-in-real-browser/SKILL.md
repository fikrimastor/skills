---
name: verify-css-layout-in-real-browser
description: "Verify a CSS/layout change (spacing, margins, flex/grid, Tailwind utility conflicts) in a real browser with computed styles, because jsdom/vitest applies no stylesheet and will green-light a visually broken change. Use when a UI change \"passes tests\" but looks wrong on screen, when Tailwind v4 utilities seem ignored, or before claiming any visual/spacing fix is verified."
---

# Verify CSS/layout changes in a real browser

## Why

jsdom (and therefore vitest/@vue/test-utils, jest) does **not** load the
stylesheet. A component test proves DOM *structure* only. It will happily pass
while the thing renders flush, invisible, collapsed, or off-screen.

Real failure this catches: paragraphs were correctly split into multiple `<p>`
(test asserted `length > 1` — passed) but rendered **with zero gap**, because a
utility was overridden by specificity. Structure correct, appearance broken.

Never claim a spacing/layout fix is verified on the basis of a jsdom test.

## Step 1 — read the compiled selector, not the class name

Utility class names lie about specificity. Inspect the **built** CSS:

```bash
# find the chunk containing the utility
grep -rl "space-y-2" .output/public/_nuxt/*.css

# extract the actual rule
python3 -c "
import re,sys
s=open('<chunk>.css').read()
for m in re.finditer(r'[^{}]*\.space-y-2[^{}]*\{[^}]*\}', s):
    print(repr(m.group(0))[:200]); break
"
```

**Tailwind v4 trap:** utilities compile through `:where()`, which contributes
**zero specificity (0,0,0)**:

```
:where(.space-y-2>:not(:last-child)){...margin-block-end:...}
```

So *any* plain class (specificity 0,1,0) — e.g. `m-0`, `mt-0`, a reset, a
component scoped style — silently beats `space-y-*`, `divide-*`, and similar
`:where()`-wrapped utilities.

Rule of thumb: when a sibling/reset class sets a property, prefer an **explicit
normal-specificity utility** (`mt-2`, `mb-2`, `gap-2`) over `space-y-*`.

## Step 2 — measure computed styles in the real page

Use the browser tool's in-page evaluator. **`tab.evaluate` runs in page
context; `tab.run` does not** (`document is not defined`).

```js
const out = await tab.evaluate(`(() => {
  const host = document.createElement("div");
  host.style.cssText = "position:fixed;left:-9999px";
  host.innerHTML = \`
    <div id="probe" class="<container classes>">
      <p class="<candidate classes>">One.</p>
      <p class="<candidate classes>">Two.</p>
    </div>\`;
  document.body.appendChild(host);
  const m = [...document.querySelectorAll("#probe > p")].map(p => getComputedStyle(p).marginTop);
  host.remove();
  return JSON.stringify({ m, spaced: m.slice(1).every(v => parseFloat(v) > 0) });
})()`);
```

Injecting a probe into the live page measures against the **real stylesheet**
without needing to navigate to the exact feature route (useful when the screen
is behind auth or requires selection state). Build the probe with the actual
container + child classes so you measure the real cascade.

Do this for pre-fix and post-fix side by side in one call — the contrast is the
proof.

## Step 3 — guard it in the test suite

Computed styles are unreachable from jsdom, but **class names are**. Assert the
class contract so the bug cannot silently return:

```ts
paras.slice(1).forEach((p) => expect(p.classes()).toContain("mt-2"));
paras.forEach((p) => expect(p.classes()).not.toContain("m-0"));
```

Then **prove the guard fails** on the broken code: temporarily reintroduce the
bug, run the test, confirm red, restore. A guard that has never failed is not a
guard.

## Checklist

- [ ] Built CSS grepped for the real selector + specificity
- [ ] Computed style measured in a real browser, pre-fix vs post-fix
- [ ] Class-contract test added
- [ ] Guard demonstrably fails on the reintroduced bug
- [ ] Full suite run (structure tests still green)
- [ ] No blanket claim of "verified" resting only on jsdom
