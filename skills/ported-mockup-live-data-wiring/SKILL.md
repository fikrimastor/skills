---
name: ported-mockup-live-data-wiring
description: "Replace mock names/ids in a ported-mockup page (e.g. Tricipta common-platform/AI-insight panels) with real API data while preserving identity joins — separate display strings, join keys, and prose-embedded tokens, then verify in-browser."
---

# Wiring real data into a ported mockup page

Use when a page ported from a mockup (`tricipta-uiux-asset` → `tricipta-edafy-asset-web`, or any
"keep the values, just connect it" ask) must show **real** names/ids from the DB while **all other
values stay mock**. Typical ask: "change the platform names to the real ones in the database, other
values maintain mock."

Core discipline: **resolve at render time, never edit the mock module's literals.**

## Step 1 — classify every mock field into three roles

Grep the mock data module and its consumers, then sort each field:

| Role | Example | Action |
|---|---|---|
| **DISPLAY** | `PLATFORMS[].label`, `FIELD.name`, `ACTIONS.top[].well`, `TOP_PROBLEMATIC_WELLS[].id` | Safe to rewrite via accessor |
| **IDENTITY JOIN KEY** | `PLATFORMS[].id`, `RATE_ESTIMATION[].id`, `FIELD_PERFORMANCE.platforms[].id` | **Leave mock** — rewriting silently breaks joins |
| **PROSE-EMBEDDED** | `ACTION_QUEUE[].action` = `'Optimise choke on A-001-LS — THP …'` | Needs text substitution, not a field swap |

Find join keys by grepping for the id being *compared or passed to a lookup*:
```bash
grep -rn "\.id ===" app/components/<area>/     # joins
grep -rn "getPlatformPct(p.id)\|getRateForPlatform(p.id)" app/components/<area>/
```
A field can be both display and key — if so, keep the key mock and resolve only the label.

**Prose-embedded names are the silent defect.** A row renders a renamed chip directly above a
sentence still naming the old well — visibly self-contradictory. Find them at the producer:
```bash
grep -n "action: '\|issue: '\|outcome: '" app/utils/<mockModule>.ts | grep -E "[A-E]-[0-9]"
```
Check the *consumer* too — a field only matters if it's rendered (`grep -rn "a\.action\|well\.issue"`).

## Step 2 — build a resolver, not literal edits

1. Pure module (no Vue/fetch): `buildLandingNames(pool)` → maps `mockId → realId`,
   `mockLabel → realLabel`. Make the pick **deterministic** (FNV-1a/djb2 over code units); never
   `Math.random()`/`Date.now()`, and never depend on API row order.
2. Composable: fetch via the repo's existing helpers, cache in Nuxt `useState` (**not** a
   module-scoped ref — leaks across SSR requests), expose accessors:
   `fieldName`, `platformLabel(id)`, `wellLabel(label)`, `wellText(prose)`.
3. Fallback = the mock literal, so SSR/first paint and existing tests still pass.
4. Trigger the fetch from **one** parent `onMounted`; children are pure readers (avoids N duplicate
   fetches and mid-swap inconsistency between panels).

`wellText(text)`: substitute all known mock labels, **keys sorted by length descending** so
`A-001-LS` is tried before `A-001`; replace *all* occurrences (`split().join()`).

## Step 3 — traps

- **Pinia-less test mounts.** If the composable calls `useAuthenticatedFetch()`, it reaches
  `useAuthStore()` and throws `getActivePinia() was called but there was no active Pinia` when a
  component test mounts without Pinia. Put *every* helper call (`useApiBase()`,
  `useAuthenticatedFetch()`) **inside the `try`**, and swallow errors → empty maps → mock fallback.
- **Endpoint param typos fail silently.** A wrong query param is ignored, not rejected, and the
  server does an unfiltered scan (observed: 43 s). Confirm params against the handler's
  `// @Param` docs before trusting a slow "empty" response.
- **`limit` may be capped** server-side (e.g. `pagination.DefaultMaxLimit = 100`) regardless of what
  you send — paginate rather than assuming one big page.
- **A hierarchy endpoint may not carry the suffix you need.** `GET /v1/wells/hierarchy` returns bare
  codes (`ANSI-A001`) with no `-LS`/`-SS`; real well+string pairs live in
  `GET /v1/well-string-reservoir-locations?field_id=` (`well_code` + `well_string_code`).
- **Pagination order can be unstable while the set is stable** — always dedupe + sort client-side.
- **Keep the platform↔well pair coherent**: derive the well's platform from the mock label's own
  letter (mock labels satisfy: first char of `A-001-LS` == last char of `ANDR-DP-A`), then draw a
  label only from that platform's pool. Independent mapping yields a well shown under the wrong
  platform.
- **Prefer the identity match when it exists** before the hash fallback — coverage is partial
  (measured ~9/17 labels had an exact real counterpart), so you need both paths.
- **Accessible names**: resolve `aria-label` too; a name that disagrees with visible text is an a11y
  defect. Existing tests asserting the mock aria-label still pass via the fallback.

## Step 4 — verify in the real browser

Unit tests can't prove this. Auth-gated routes: log in via Keycloak (`testuser`/`testpass`), then
pull rendered text and assert on it.

```js
const txt = await tab.evaluate(`document.body.innerText`);
// stale sweep: no mock id survives, real ids present
// chip vs prose: for each row, the chip name must appear in its own prose line
// ref integrity: !/\[object Object\]/.test(txt)   // a ComputedRef interpolated without .value
```

Check explicitly: stale-mock-name list is empty, `[object Object]` absent, and each row's chip name
matches the name inside its prose line. Then run the full suite (`npx vitest run`).

## Reporting
State the endpoint per name source, the live evidence (platform cards, sample names, counts), and
call out any defect you caught mid-flight rather than only the happy path.
