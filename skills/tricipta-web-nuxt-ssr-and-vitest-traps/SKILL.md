---
name: tricipta-web-nuxt-ssr-and-vitest-traps
description: "Diagnose and fix SSR/hydration and raw-vitest failures in tricipta-edafy-asset-web (Nuxt SSR): setInterval/setTimeout during setup 500s, clock-derived text hydration mismatch, missing explicit composable imports, and the build heap flag."
---

# Tricipta asset-web — Nuxt SSR & raw-vitest traps

Applies to `~/Sites/afed/tricipta-edafy-asset-web` (Nuxt 4.4, SSR enabled: `nuxt.config.ts` has `ssr: {}`). Raw vitest = no Nuxt test runtime, no auto-imports.

## 1. "[nuxt] `setInterval` should not be used on the server" → dev SSR 500

Symptom: the page returns a 500 error document naming `setInterval`; the route works in a pure client build.
Cause: a timer scheduled during setup — a module-level `setInterval/setTimeout`, or a `watch(..., { immediate: true })` callback that schedules one. Setup runs during the server render.
Fix in the composable/component:
```ts
watch(src, (v, _o, onCleanup) => {
  if (import.meta.server) return   // server output stays empty, matching the client's pre-timer state
  const id = setInterval(…, 14)
  onCleanup(() => clearInterval(id))
}, { immediate: true })
```
For components that schedule from watchers, guard each scheduling callback the same way, or move the first schedule into `onMounted`.
Verify: `curl -s -o /dev/null -w '%{http_code}' http://localhost:3000/common-platform/dashboard` → 200, and the page text renders.

## 2. Clock-derived text → hydration mismatch

Symptom: none locally (same machine), but a UTC-hosted server renders "Good morning" while the viewer hydrates "Good afternoon"; Vue logs a hydration text mismatch.
Cause: `computed(() => new Date().getHours() …)` evaluated in setup — the server evaluates it once, the client again.
Fix (keeps the copy, no empty first paint):
```ts
function currentPartOfDay() { const h = new Date().getHours(); return h < 12 ? 'morning' : h < 18 ? 'afternoon' : 'evening' }
const partOfDay = useState<string>('landing-part-of-day', () => currentPartOfDay())  // server value ships in the payload
onMounted(() => { partOfDay.value = currentPartOfDay() })                            // correct to the viewer's clock
```
`useState` keys are global per request — pick a unique key per surface.
Check: SSR HTML (`curl … | grep -o 'Good [a-z]*'`) and the client-rendered word agree.

## 3. "<name> is not defined" when a component mounts under vitest

Cause: Nuxt auto-imports exist at runtime only. Fix in the component (not in the test): add the explicit import, e.g.
`import { useDisplayName } from '@/composables/useDisplayName'` — same convention as `useRole`/`useAuthStore` already followed in `CPHeader.vue`.
Watch for Nuxt-global composables too: `test/setup.ts` stubs `useState`, `useRoute`, `navigateTo`, `useHead`, … (if a stub itself throws, e.g. a missing `import { ref } from 'vue'`, fix `test/setup.ts`).

## 4. Build

`npm run build` OOMs on the default node heap (native abort; exit status hidden if piped to `tail`). Use:
```bash
NODE_OPTIONS=--max-old-space-size=8192 npm run build > /tmp/build.log 2>&1; echo "exit=$?"; grep -nE 'Build complete|error|heap out of memory' /tmp/build.log | tail
```

## 5. Quick checklist before claiming a landing/UI change verified

- [ ] `npm run test` (full suite) green; scoped tests for the changed components run individually.
- [ ] SSR route returns 200 and the SSR HTML carries the expected copy.
- [ ] Client render matches SSR (same greeting word), no hydration warning.
- [ ] Narrow + wide layout: `xl` behaviour via a same-origin 1440-wide iframe when the harness pane is fixed-width; stacked reachability via `el.scrollIntoView()` + rect, measuring the real scroll root (`div.overflow-y-auto`), not `document.scrollingElement`.
