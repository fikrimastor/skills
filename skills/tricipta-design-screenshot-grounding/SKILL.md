---
name: tricipta-design-screenshot-grounding
description: "When a Tricipta issue says 'match the design' with a screenshot — ground it against prototype source and live app first."
---

# Tricipta design grounding: issue screenshot → prototype source → live app

Use when a Tricipta issue (usually `tricipta-edafy-asset-web`) carries a design screenshot and/or a
`ui.toolkit.asset.trinovate.io` link and you must produce a grounded spec + plan. Run this BEFORE
`skill://tricipta-spec-plan-house-format`; the port itself follows `skill://tricipta-design-port`.

## 1. Recover the image for real

- Anonymous fetch of `https://github.com/user-attachments/assets/...` **404s** (private AFED-digital
  repos).
- ```bash
  TOKEN=$(gh auth token)
  curl -sL -H "Authorization: Bearer $TOKEN" -H "Accept: application/octet-stream" \
    -o /tmp/issue<N>.png -w '%{http_code} %{content_type}\n' <attachment-url>
  ```
  expect `200 image/png`.
- Persist it beside the spec (`docs/superpowers/specs/<date>-<slug>.design.png`) and record
  `shasum -a 256` in the spec's title block — `/tmp` is not durable evidence.

## 2. Never trust the vision pass alone

- A full-image vision read (`read <img>?q=<question>`) invents plausible labels and mis-orders
  blocks: it produced `RECOVERABLE / WITHOUT A RIG / ANNUAL VALUE` tiles that were real but got the
  section order wrong.
- Verify with **region crops** (`PIL.Image.crop` in `eval`, or `ffmpeg -vf crop=`), magnify, re-ask
  per crop, asking for *verbatim transcription* with `[unclear]` markers.
- Then confirm every string against source (step 3). Screenshot = acceptance surface; source =
  implementation authority.

## 3. Pin the prototype page (design source of truth)

Repo: `~/Sites/afed/tricipta-uiux-asset/tricipta-platform/src` (React/Vite). Run it locally
(`npm run dev`, :5173) or probe the deployed bundle.

- Grep distinctive copy — but match the **source casing**: labels live title-case in `data/*.js`
  (`Gas lift now`, `Gap Decomposition`) and are uppercased by CSS. Searching `GAS LIFT NOW` fails.
- Interpolated/escaped copy never greps literally: `envelope this {partOfDay}`,
  `I&apos;ve ranked {ACTIONS.total} moves` → grep the static fragment only.
- Find the **active** composition: `pages/CompanionHome.jsx` has an unconditional `return` with a
  "Chat parked" comment (#808: `CompanionHome.jsx:1360-1368` → `CompanionFieldOptimization.jsx`);
  the real landing is a component that looks like a section, and the rest of the file is dead code.
- Data files mix quoted and unquoted keys (`'GL Reallocation': {…}` vs `Reactivation: {…}`), so
  `^\s*'([^']+)':` silently under-counts. Suspect your own regex before declaring data missing.
- Cross-check the deployed bundle: `curl https://ui.toolkit.asset.trinovate.io/assets/index-*.js`.
  Same strings ⇒ deployed == local main; *missing* chat strings ⇒ minifier dead-code stripping, not
  a newer build. `git log --all` / `git fetch --all` to rule out newer refs (a newer ref touching
  only nav chrome is a non-event).

## 4. Verify layout against the live app, not coordinates

- `browser.open` on the prototype; dev login form is prefilled `zulsyazwan@afed.com.my` / `password`
  (`VITE_DEV_LOGIN=true`) — click `LOGIN`.
- **cmux panes ignore `page.setViewport`/CDP emulation** — don't chase a desktop width. Decide
  per-column placement by **DOM ancestry**: find the leaf text, walk to the nearest `<aside>`, check
  which section titles live in it. At narrow widths every column is full-width so x/y tells you
  nothing.
- `document.body.innerText` omits content clipped by `overflow:hidden` — use `textContent` (reliable
  regardless of pane size).
- Match labels case-insensitively (CSS uppercases them). A 14×14 lucide icon can match a "find the
  svg" selector before the real chart does — scope the query.

## 5. Resolve screenshot-vs-prototype conflicts explicitly (do not silently choose)

The two WILL differ (nav item sets, block placement, brand text). Ask which is authoritative and
record the answer.

- Standing answer (2026-09-11): **the running prototype wins** — #808: 6 nav pills over the
  screenshot's 11; Field Performance in the left banner over the screenshot's right panel;
  ranked queue "50" label vs 8 rendered rows (`ACTIONS.total` is the label); "3 Assets" vs 5
  `PLATFORMS`; donut 132 px component default vs `size={88}` on the landing.
- Counter-current on the same issue: a #808 pass followed the *screenshot* because QA compares
  against it, noting the flip is cheap. When a source states that, it is an acceptance decision,
  not a silent pick — say which you acted on either way.
- The running prototype is the implementation authority unless the user names a Figma frame or
  explicitly says QA compares against the screenshot — never edit from one source without saying
  which one you acted on.
- Record the losing variant as a spec open item with an explicit flip cost, and label any
  unconfirmed default as *inferred*, never as a user decision.
- Propagate a flip to spec **and** plan **and** the archify workflow cards, then re-run
  `validate` → `deliver` → `visual-check`.

## 6. Audit every count/list before writing tests

For each assertion check the data file (the `fieldOptimization.js`-style constants) *and* a live DOM
probe; never inherit a count from the screenshot.

## 7. Check the target app's port traps (these caused real plan bugs)

- **Nav contract**: `app/composables/usePlatformNavigation.ts:15` silently returns when
  `MENU_DESTINATIONS[name]` is absent, and pill labels are often NOT keys (`Overview` /
  `Diagnostic` / `Users` vs `VFM/VPS Overview` / `Diagnostic Plots` / `Manage Users`). Add alias keys
  or navigate by path.
- **Viewport lock**: the prototype root is `h-full … overflow-hidden` with `flex-1 min-h-0` +
  per-column `overflow-hidden` ("nothing here scrolls") — copying it 1:1 clips the stacked layout
  below `xl`. Gate the lock with `xl:` and let the page scroll when stacked.
- **Tailwind v3→v4**: `shadow-3xs`, `py-0.2`, `bg-teal-650`, `rounded-br-xs` do not exist in v4 —
  remap (`shadow-[0_1px_1px_rgb(15_23_42_/_0.04)]`, `py-0.5`). Full mapping table in
  `skill://tricipta-design-port`.
- **Dead helper destinations**: helpers hardcoding `/common-platform/dashboard`
  (e.g. `openCompanionChat`) go stale the moment the landing route is redefined — grep for other
  hardcoded paths before freezing a file as byte-identical.

## 8. Then hand off

Write the house-format spec + plan (`docs/superpowers/{specs,plans}/YYYY-MM-DD-<slug>.md`) and
materialize the archify workflow. The archify tool is a directory, not a binary:

```bash
node ~/.omp/tools/archify/archify/bin/archify.mjs validate workflow <json> --quality showcase --json
node ~/.omp/tools/archify/archify/bin/archify.mjs visual-check <html> --json
```
Gate on 9/9, 0 errors/warnings, containment pass at 1440×900 & 2048×1320. Present and wait for
approval before executing.
