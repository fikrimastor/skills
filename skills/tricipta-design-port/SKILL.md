---
name: tricipta-design-port
description: "When porting tricipta-uiux-asset React mockups into tricipta-edafy-asset-web Vue SFCs (Tailwind v3→v4, keyframes)."
---

# Porting React mockups to tricipta-edafy-asset-web

Design repo (read-only source): `~/Sites/afed/tricipta-uiux-asset/tricipta-platform/src/pages/` —
JSX mockups like `BubbleMapDashboard.jsx`, `DiagnosticPlotsDashboard.jsx`, `VPSDashboard.jsx`,
`DDMDashboard.jsx` are the spec. Target: `tricipta-edafy-asset-web/app/pages/common-platform/*.vue`
with shared components in `app/components/common-platform/`.

Ground the design first (`skill://tricipta-design-screenshot-grounding` when a screenshot is
involved); verify the result live (`skill://tricipta-web-live-verification`).

## Steps

1. Read the full design JSX + the current Vue page side by side; identify deltas — don't
   re-implement what already matches.
2. Port to Vue 3 `<script setup lang="ts">` SFC conventions: computed/ref/watch, `v-for` keys, no
   JSX, lucide-vue-next icons. Mockup text and values are the spec — port them verbatim, don't
   invent.
3. Remap Tailwind classes (web = **v4**, design = **v3.4.19**; its `tailwind.config` extends only
   colors/font/animation/keyframes, so several design utilities are **no-ops** — don't copy them
   believing they render):

| design (v3) | use in web (v4) | note |
|---|---|---|
| `shadow-2xs` | `shadow-sm` | v4 `shadow-sm` = v3 `shadow` (heavier) |
| `shadow-3xs` | `shadow` (or drop) | doesn't exist in v4 — a bare `shadow-3xs` draws nothing |
| `shadow-sm` | `shadow-xs` | scale step shifted |
| `backdrop-blur-sm` | `backdrop-blur-xs` | 4 px vs 8 px |
| `rounded-sm` | `rounded-xs` | v3 = 2 px; web's `rounded-sm` = `calc(var(--radius) - 4px)` ≈ 6 px |
| `bg-teal-650` | `bg-teal-600` | non-stock shade |
| `rounded-br-xs` / `rounded-bl-xs` | `rounded-br-sm` / `rounded-bl-sm` | non-stock |

   Unchanged, copy as-is: `shadow-lg`/`shadow-xl`, `backdrop-blur-md`,
   `rounded-lg`/`rounded-xl`/`rounded-full`/`rounded-2xl`, and the
   slate/teal/purple/indigo/amber/emerald palette. Arbitrary values from v3 that don't exist in v4:
   `shadow-[0_1px_1px_rgb(15_23_42_/_0.04)]`, `py-0.2` → `py-0.5`.
   The web overrides radius tokens (`--radius: 0.625rem`, `--radius-xl +4px`) so `rounded-xl`
   renders 14 px vs the design's 12 px — keep the app token for consistency with sibling cards and
   record the residual instead of hard-coding pixel radii.
4. Port keyframes: append to `app/assets/css/common-platform.css` (already has `.animate-fade-in`):
   drawerAppear (`.drawer-animate-appear`), sectionReveal (`.insight-section`), kpiPop (`.kpi-card`),
   findingReveal (`.finding-item`), purpleShineSweep (`.animate-purple-gradient`). Keep class names
   identical to the design repo's `index.css`. The web's Tailwind v4 `@theme` declares only
   `--radius-*`/`--color-*`, so any non-stock animation/shadow token must be defined explicitly.
5. AI Insight drawer pattern (platform-wide in design: Bubble Maps, Diagnostic Plots, VPS, DDM):
   purple button + anchored popover drawer with caret; header (title + chat toggle + close); three
   modes — card explorer (purple gradient exec summary + module selection flow), module report
   (numbered sections 01-05 with staggered reveal, KPI cards, typewriter analysis text), chat
   (message bubbles, quick chips, 600 ms fake reply). Build as a self-contained component taking
   `{field, capabilityLabel, open}` + a `close` emit.
6. Reuse existing web components first: InsightLogoIcon, CPTable, LoadingIndicator,
   TubingStringSelector, ReservoirMultiSelect, BubbleMapLegend, BubbleGeoMap. For CPTable changes
   keep new props optional so other consumers are unaffected.

## Vue/Nuxt port traps

- **Viewport lock**: the prototype root is `h-full … overflow-hidden` with `flex-1 min-h-0` +
  per-column `overflow-hidden` ("nothing here scrolls"). Copying it 1:1 clips the stacked layout
  below `xl` — gate the lock with `xl:` and let the page scroll when stacked.
- **Nav contract**: `app/composables/usePlatformNavigation.ts:15` silently returns when
  `MENU_DESTINATIONS[name]` is absent, and pill labels often aren't keys (`Overview` /
  `Diagnostic` / `Users` vs `VFM/VPS Overview` / `Diagnostic Plots` / `Manage Users`). Add alias keys
  or navigate by path.
- **`<script setup>` cannot contain ES module VALUE exports** — `export const X = ...` fails with
  "`<script setup> cannot contain ES module exports". Only type exports are legal. A shared constant
  needed by both an SFC prop default and a page fallback must live in a module
  (`app/utils/*.ts`). Verify empirically with `@vue/compiler-sfc`'s `parse` + `compileScript`.
- **Dead helper destinations**: helpers hardcoding `/common-platform/dashboard`
  (e.g. `openCompanionChat`) go stale the moment the landing route is redefined — grep for other
  hardcoded paths before freezing a file as byte-identical.
- An unimported identifier in `<component :is="X">` renders `<!---->`; the tile still measures the
  right size, so assert `svg` presence in tests rather than box dimensions.

## Verification

- Read back edited hunks (no build/lint per repo workflow). `npx eslint` cannot run in this repo
  (`.eslintrc.json` contains `export default`; no flat config) — use `npx vitest run` + the dev
  server compiling the page + a browser pass with no console errors.
- Confirm the utilities you used actually resolved: read `getComputedStyle` in the page
  (`backdrop-filter`, `box-shadow`, `border-radius`) — a class that doesn't exist in v4 silently
  draws nothing.
- Proving the ported page renders live (SSO, hidden-tab ArcGIS, tile traffic) →
  `skill://tricipta-web-live-verification`.
