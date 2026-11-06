---
name: tricipta-design-parity-audit
description: "When auditing asset-web (:3000) against the uiux prototype (:5173) for design parity via DOM/CSS measurement."
---

# Tricipta design-parity audit: asset-web (:3000) vs uiux prototype (:5173)

Use when verifying `tricipta-edafy-asset-web` against the design implementation
(`tricipta-uiux-asset`, running at `:5173`): design-parity reviews, "doesn't match the design"
reports, port QA for a specific card/section. Produces FINDINGS, not fixes.

## 1. Authenticate, then ASSERT the surface (never skip, re-assert after every navigation)

- Implementation `http://localhost:3000/common-platform/<page>` — Keycloak SSO: click
  `SIGN IN WITH SSO`, then fill `#username` = `testuser`, `#password` = `testpass`, submit `Sign In`
  (lands on Keycloak `:8180`; a browser profile holding an existing session round-trips silently).
- Design `http://localhost:5173/` — dev form prefilled `zulsyazwan@afed.com.my` / `password`
  (`VITE_DEV_LOGIN=true`); click the button whose text is exactly `LOGIN`.
- Both dev servers must be up (`curl -o /dev/null -w '%{http_code}'`). Stack ports: web `:3000`,
  asset-api `:8085`, Keycloak `:8180`, plot-api `:8000`. Never start a second web dev server on
  `:3000`. Sessions drop when a dev server restarts — re-login rather than debugging "the app is
  broken".
- Anchor BOTH sides before any measurement: breadcrumb `Angsi Field / AI Insight` **and** the exact
  section you are measuring in `textContent` — for the landing, a leaf node
  (`[...document.querySelectorAll('*')].find(n => !n.children.length && n.textContent.trim() === 'Field Details')`).
  A signed-out prototype renders its **login card**, a different lockup, which has produced bogus
  "brand metric" findings and a wrong commit that had to be reverted.
- Record `location.href` with every measurement block.

## 2. Measure deterministically, never by eye

1. `tab.evaluate` returning **plain values**: for a text-anchored leaf element capture
   `getComputedStyle` — `fontSize`, `lineHeight`, `fontWeight`, `letterSpacing`, `textTransform`,
   colour, `padding`, `gap`, `borderRadius`, `backgroundColor`, `borderTopColor`, `backdrop-filter`,
   `box-shadow` — plus `getBoundingClientRect()` (w/h/x/y). Diff the numbers; that is the evidence.
2. Anchor by **leaf text**, then compare. Scope probe selectors: the prototype's card `Overview` link
   is 11px/600, not the 11px/500 nav pill of the same name.
3. SVG/shape questions: read attributes (`width`, `viewBox`, `stroke`, `stroke-width`,
   `stroke-dasharray`, `stroke-linecap`) and `<text>` computed sizes on both sides.
4. Icon slots: assert `svg` presence, not box size. `<component :is="X">` with `X` unimported renders
   `<!---->` and the tile still measures the right size — sweep templates for identifiers missing
   from the script block.
5. Vision (`read <png>?q=`) is a **tiebreaker only**. It has invented a KPI delta (`10.9%` — exists
   nowhere in the DOM), a caps wordmark, a 700-weight subtitle, and `+26.3%`/`+2,593` OCR slips.
   Single-image questions work; cross-image comparison does not — stitch two crops into ONE image
   and ask a comparative question. Every vision claim must be confirmed against source/DOM/pixels
   before it is reported.
6. Shapes/crops without numpy: PIL only (`Image`, `crop`, `getdata`, thresholds); sample centre vs
   ring pixels to decide fill polarity (white disc with a star-shaped hole vs filled star).
7. Compare at the **same width regime** on both sides (1440 iframes); attributing a 743 px artifact
   to the design is the single most common false finding.

## 3. Desktop widths despite the ~743 px pane

- The cmux/harness pane is fixed at ~743×994 CSS px and **ignores `page.setViewport` and CDP
  `Emulation.setDeviceMetricsOverride`**. `tab.screenshot()` is therefore a ~765×1024 stacked
  capture (and timeouts on heavy pages) — not a parity tool.
- Inject a **same-origin iframe** at the target width (1440; 1600 for the `2xl` regime) and read
  `iframe.contentDocument`:
  ```js
  const f = document.createElement('iframe');
  f.style.cssText = 'position:fixed;left:0;top:0;width:1440px;height:900px;border:0;opacity:0.01;z-index:-1';
  f.src = 'http://localhost:3000/common-platform/dashboard';
  document.body.appendChild(f);
  // later: f.contentDocument.querySelector(...), f.contentWindow.innerWidth
  ```
  On `:5173` you must click `LOGIN` **inside the iframe's own document** (its auth is React state,
  not a cookie) and re-assert the anchors in-frame before trusting numbers.
- `tab.evaluate` is **synchronous** (cannot await a promise); `tab.run` and `tab.close` frequently
  fail with `not_found: Workspace not found` — use separate calls: inject → sleep ~8-10 s → read →
  remove.
  Prototype HMR full-reloads wipe injected iframes; re-inject when a probe returns "no iframe".
- `innerText` **omits clipped content** → use `textContent` or DOM queries. To prove reachability use
  `getBoundingClientRect()`/`scrollIntoView()` (a page-bottom element legitimately sits above the
  fold when stacked). Find the real scroll container: the shell root may be the scroller
  (`overflow-y-auto`) while `document.scrollingElement` reports no overflow.
- `eval` cells have a ~30 s cap: pass `timeout: 0`, or keep cells short.

## 4. Traps that fabricate findings

- **Login-card vs header lockup.** Prototype signed-out card: logo 24, line 1 17px/17px, subtitle
  `text-[9px] text-slate-400 font-bold`. Its header: logo 32×32, line 1 `font-bold text-lg` =
  18px/28px title case, line 2 `text-[9px] … text-teal-200 font-medium` = 9px/11.25px w500, block
  39 px inside a 56 px bar. Measure inside a `<header>` scope.
- **Viewport lock at narrow widths.** The prototype root is `h-full … overflow-hidden` with
  `flex-1 min-h-0` columns and per-column `overflow-hidden` ("nothing here scrolls"). At 743 px its
  stacked columns collapse (wrappers 0/0/2 px tall, queue section ~26 px, greeting `h2` 0 px wide,
  headings 0 px) because it has no page scroll, only inner panes. Reading that as "the design omits /
  collapses X" is wrong: our app deliberately scrolls instead (intentional §5.4 relaxation).
- **Radius token.** This app is Tailwind v4 + shadcn: `app/assets/css/tailwind.css:7-10,46` sets
  `--radius: 0.625rem`, so `--radius-lg` = 10 px and `--radius-xl` = 14 px — every `rounded-*` renders
  +2 px vs the design's v3 scale (8/12). Fix ports with explicit `rounded-[12px]` / `rounded-[8px]`;
  **never edit `--radius`**. A systematic +2 px is this, not a defect (but verify live: it was NOT
  real for AI insight sections).
- **Colour notation.** `oklch()`/`oklab()` (devtools, v4) vs `rgb()`/`rgba()` (prototype, v3) is
  notation, not a colour delta — resolve to sRGB (canvas or a screenshot pixel) before reporting.
  e.g. `oklch(0.446 0.043 257.28)` = `rgb(71,85,105)` = slate-600.
- **Identical class names render differently across versions**: `text-lg` is 18/28 in the app and
  17/17 in the prototype (custom ui-kit scale); v4 bakes line-heights into size utilities, so a
  parent `leading-*` cannot override them — fix with explicit size/line-height, not the class name.
- **User screenshots may come from a different artifact than the running prototype**: a scrolled
  screenshot hides the header, so don't infer title/layout from it. Say which source each claim is
  measured against.

## 5. Docked AI Diagnostic Insight panels (DDM etc.)

- Prototype `DDMDashboard.jsx:589-599`: frameless full-height aside right of the **whole tab card**
  (top aligned with the tab row), xl 440 px / 2xl 500 px, 12 px gap, title "AI Diagnostic Insight"
  with Diagnosis/AI Chat switch, body gap 10 px, narrative 10.5 px, differentiated section icons.
  Measured rect at 1440×900: `(988,188,440,700)`; at 1600: `(1088,188,500,700)`.
- Asset-web panel root needs class `companion-diagnostic-parity` (teal-200 override → `#CEF0EE`, in
  `common-platform.css`); without it teal borders render `rgb(150,247,228)`.
- **Scroll model:** the prototype page does not scroll — the left column scrolls internally and the
  aside stays put with its own internal scroll. Do NOT use `sticky top-N` on the aside (leaves an
  N-px empty band once the tab row scrolls away) nor `max-h-[720px]` (panel scrolls off, looks
  truncated). Verify by measuring the panel rect at left-column scrollTop 0/mid/max — must be
  constant — and that the footer is reachable by internal scroll. Check other DDM tabs still scroll.
- Prototype also shows a "Live AI" chat badge; asset-web's shared `AiInsightChatView.vue:70` matches.

## 6. Deliver

- Findings table: `region | design (measurement) | implementation (measurement) | delta | severity | suggested fix`.
- Plus a **verified-matching** list, a **"not measured"** list, a separate **"differences caused by
  our intentional deviations"** section (e.g. 6-vs-11 pill row, Field Performance in the left
  banner, our page scrolling where the prototype clips), and an **"accepted deviations — revisit?"**
  section with measurements, and an explicit **retracted claims** list kept visible.
- Mark provenance (who measured what) — a subagent's claim is not evidence until re-measured. A
  read-only subagent pair (one per column/region) works well for the measurement fan-out.
- Gates before claiming done: `npm run test`; and
  `NUXT_IGNORE_LOCK=1 NODE_OPTIONS=--max-old-space-size=8192 npm run build`. Plain `npm run build`
  OOMs (default heap) and Nuxt refuses to start a second concurrent build in the tree ("Another Nuxt
  build is already running", fails in ~2 s — not a compile error; serialise or cancel the stale one).
  Capture the real exit code: `cmd > /tmp/b.log 2>&1; echo exit=$?` — piping to `tail` masks it
  (use `${PIPESTATUS[0]}`), and `&&` chains then continue on failure.
- Subagent hygiene if delegating: give each auditor its own browser tabs, forbid repo-wide git
  commands (`git checkout/restore/stash/reset/clean` — one wiped a sibling's uncommitted work), and
  require path-limited commits.

## 7. Data preconditions

DDM sensitivity/forecast data exists locally only for field **Bokor** (Angsi shows "no sensitivity
screen") — switch the header field selector first.
