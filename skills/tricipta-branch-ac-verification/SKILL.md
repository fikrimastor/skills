---
name: tricipta-branch-ac-verification
description: "Verify a Tricipta asset-web feature branch against the originating issue's acceptance criteria in the harness browser, and draft the PR from measured evidence. Use when asked to \"check whether the branch fulfils the ACs\" or to create the PR for a Tricipta issue branch — covers the build-clobbers-dev-server trap, poisoned-tab false negatives, and refuting sub-verifier claims."
---

# Verifying a Tricipta branch against an issue's acceptance criteria

Use when asked "does this branch fulfil issue #NNN's acceptance criteria?" or to then create its PR, for `tricipta-edafy-asset-web` (Nuxt, Keycloak-gated). Main owns the verdict; fan the measurement slices to subagents.

## 1. Ground the ACs before measuring

- Read the issue (`issue://AFED-digital/<repo>/<N>`) for the literal AC text. Quote ACs verbatim in each subagent brief.
- Look for a prior approved **spec** in the sibling workspace `~/Sites/afed/tricipta/docs/superpowers/specs/` and its plan in `plans/`. These carry an executable check table (A1..A9) and — critically — an **accepted-deviations** section. Distinguish "deviates from the approved spec" (fine) from "deviates from the literal issue screenshot" (still a finding, because issue ACs often say *implement exactly as shown*).
- Planning docs live in the `tricipta` workspace, **not** in the component repos. Never link them as if they were in-repo paths — `tricipta-edafy-asset-web/docs/` contains only `bubble-maps-esri/` and `653-well-test-web/`; such links 404 on GitHub.

## 2. Prove data-module parity mechanically

When a branch ports a prototype data file, don't eyeball it — bundle both and deep-compare:

```bash
cd <repo> && node_modules/.bin/esbuild --bundle app/utils/<x>.ts \
  --outfile=/tmp/cmp/impl.mjs --format=esm --platform=node
node_modules/.bin/esbuild --bundle <prototype>/src/data/<x>.js \
  --outfile=/tmp/cmp/design.mjs --format=esm --platform=node
```

then `isDeepStrictEqual` every export and any derived export (`RATE_TOTALS`-style `.map`). This settles AC2/"content matches" in one shot. Note the prototype's `formatCount` may not re-export through the bundle — check before asserting on it.

## 3. The build-clobbers-dev-server trap (cost a whole verification cycle)

`npm run build` in this repo writes `.nuxt/`, which the **already-running `nuxt dev`** is serving. Even a *cancelled* build leaves the dev server's Vite module graph corrupt, presenting as:

- client-side nav / clicks → `500 Internal Server Error\nImporting a module script failed.`
- a reload loop (page `performance.timeOrigin` changes between probes)
- **`curl` of the route still returns 200** — only the module graph 500s, so HTTP status cannot detect it.

Rules:
- Never run `npm run build` while a dev server is up **and** subagents are driving it. Serialise: let them settle, or stop the dev server first.
- Recovery: restart `nuxt dev` (`kill` the pid, then start under `hub` with `ready: {port: 3000}` — readiness may time out because it binds IPv6 `[::1]` only; confirm with `lsof -nP -iTCP:3000 -sTCP:LISTEN`).
- For authoritative interaction evidence, prefer the **production preview** (`skill://tricipta-web-live-verification` §1: build, then serve `.output/server/index.mjs` on :3001 with `NODE_ENV=development` and the CORS-allowed port). The same reload loop can reproduce there too — see next.

## 4. Poisoned tabs produce false negatives — always re-test on a fresh tab

A tab that lived through the corrupt-server window keeps failing after the server is fixed. Symptom: `document.querySelectorAll('button').length` collapses to 0 and `performance.timeOrigin` jumps every probe. Open a **fresh named tab** (`browser.open({name: "fresh1", ...})`) before concluding any control is broken. Do not attribute a click failure to branch code until a clean tab reproduces it.

## 5. Browser harness recipes that actually work here

- `tab.evaluate(...)` — synchronous, **cannot await a promise**. Returning a Promise errors with "this surface evaluates synchronously". Split inject → wait → read into separate calls, or run the async sequence inside `tab.run(async ({page}) => …)` as one cell (< 30 s ceiling).
- `page.createCDPSession`, `page.on(...)`, `tab.observe()`, `tab.id(ref).click()` are **not available** on this surface — capture console/page errors by injecting `window.addEventListener('error'/'unhandledrejection')` plus wrapping `console.error`/`console.warn` in-page, then read the array back.
- Widths: the pane is a fixed ~743 px and `setViewport` is a no-op. Inject a same-origin iframe of the target width, wait ~14 s, measure inside `contentDocument`. For 1440 assert `documentElement.scrollWidth <= clientWidth + 1`.
- A9-style "reachable when stacked": `scrollIntoView({block:'center'})` on the target then assert in-viewport **and** walk ancestors for `overflow:hidden` clipping. Scrolling fully to the bottom legitimately pushes earlier elements out of view — don't misread that as clipped.
- Network (AC5): `performance.getEntriesByType('resource')` and filter; watch out for `/api/` false matches inside `node_modules/@microsoft/applicationinsights-*/otel/api/...` dev-module URLs.

## 6. Refute sub-verifier claims; never relay them

Subagent reports are claims, not evidence. In one run both a functional and a parity agent reported failures that direct measurement refuted:
- "Top 5/10 toggle does not function" — actually `[data-testid="problematic-well"]` count goes 5 → 10.
- "N solutions expand does not function" — actually `aria-expanded` flips `false → true` with `BEST FIT` + `Ref …` rendered.
- "AI Chat does not navigate" — poisoned tab; on a clean preview tab it reaches `/common-platform/companion`.
- A parity agent marked AC3 "PASS" while only ever measuring the 743 px stacked pane — that is not AC3 evidence; re-measure the real viewports.
Re-run each contested claim yourself and state the correction explicitly. Also verify their artifact paths exist before citing them.

## 7. AC5 ("load time not negatively affected")

Evidence that holds: landing route chunk size (find it with `grep -rl <a-unique-data-string> .output/public/_nuxt/*.js`, then raw + `gzip -c | wc -c`), no new `package.json`/lockfile entries, zero added `/api/` requests on mount, zero console errors. Label any claim about *perceived* load as `[INFERENCE]` unless you diff first-paint against `origin/main`.

## 8. Draft-PR hygiene for these branches

- Recon first: `git remote -v`, upstream, `git ls-remote --heads origin <branch>`, `gh pr list --head <branch> --state all`, and recent merged PR titles for convention (`baseRefName` is `main`).
- Confirm nothing untracked leaks: `git ls-tree -r --name-only HEAD | grep -cE '^(\.nuxt|\.output|dist)/'` must be 0, and changed files should all sit under `app/` or `test/`.
- Put the measured evidence table, the accepted-deviations section (with any still-open sign-off item), and the known issues in the body. Surface real defects rather than hiding them; flag `Closes #N` vs `Refs #N` when an AC is only met against the prototype and not the literal screenshot.
