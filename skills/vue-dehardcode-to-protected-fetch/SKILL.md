---
name: vue-dehardcode-to-protected-fetch
description: "Convert a Vue/Nuxt component (or family of components) from static hardcoded module-level data to data fetched from a protected/authenticated API, with minimal diff and correct loading/error/empty states. Use when asked to move mock/hardcoded data behind auth, remove a client-bundled data fixture, or replace a MockDataBadge/USE_..._MOCK-style fallback with real fetched data."
---

## When this applies

A component (or a small family of sibling components, e.g. one parent rendering several children) currently does:

```ts
import { PRODUCTION, ACTIONS, PLATFORMS, ... } from '@/utils/someHardcodedData'
```

and uses those constants directly in both `<script setup>` and `<template>`. The goal is to make the *values* come from an authenticated fetch, while keeping the diff small and never letting the page fabricate real-looking data when the fetch fails or hasn't completed (e.g. because the requester isn't really authenticated — client-side auth gates can be spoofed, so the data itself must be the enforcement boundary).

## The core trick: same-named `ComputedRef`s keep templates untouched

Vue's `<script setup>` SFC compiler auto-unwraps a **top-level** `ref`/`computed` binding when it's referenced in the `<template>` (`PRODUCTION.gap` compiles to `_unref(PRODUCTION).gap`). Exploit this:

1. Define a composable that fetches once (client-only, `useState`-backed for cross-component sharing, idempotent `ensureLoaded()` with in-flight dedupe — mirror any existing composable in the codebase that already does an authenticated fetch, e.g. one that fetches paginated real names).
2. The composable exposes computed refs **named exactly like the old constants** (`PRODUCTION`, `ACTIONS`, `PLATFORMS`, ...), each `computed(() => raw.value.someKey)`, where `raw` starts at a deep-zero/empty default and only gets replaced wholesale on a successful fetch.
3. In each consuming component: replace `import { X } from '@/utils/hardcoded'` with `const { X, ... } = useTheNewComposable()`. **Templates need zero edits.** Only script-scope usage (inside `computed()`, plain functions, top-level `const`) needs `.value` inserted, because refs do NOT auto-unwrap in script code, only in the template.
4. Watch for **setup-time plain consts derived from the data** (e.g. `const segments = SOME_DATA.levers.reduce(...)`) — these must become `computed(() => SOME_DATA.value.levers.reduce(...))`, or they freeze at whatever the ref held when the module first evaluated and never react to the fetch resolving.

This keeps a 5-component, ~100-reference migration to a handful of import-line + `.value`-insertion edits per file instead of rewriting every template expression.

## Empty/loading/error: gate once at the top, not per-leaf-component

If the leaf/child components are **only ever mounted from one parent** (verify this first — grep for their usages), gate the data-dependent region **once** in that parent:

```vue
<template v-if="loaded">
  <!-- children assume data is present and well-shaped; no per-property null guards needed -->
</template>
<div v-else-if="!error">Loading…</div>
<div v-else>Data is unavailable for this session.</div>
```

This is far cheaper and safer than null-guarding every property access across every leaf component (division-by-zero on empty aggregates, `.find()` returning undefined, indexing an empty `Record`, etc.).

Two traps in the gate itself, both easy to get backwards:
- **Gate the loading branch on `!error`, not on a separate `loading` boolean.** `loaded` and `loading` both start `false` before `onMounted`'s async fetch has a chance to flip `loading` true — so a naive `v-else-if="loading"` / `v-else` still flashes "unavailable" on first paint. `error` stays `null` until a fetch has genuinely, terminally failed, so `!error` correctly covers both "not started yet" and "in flight."
- **Never let the composable populate the data ref on a non-2xx response.** On `!response.ok` or a thrown error: set `error.value`, leave `loaded.value` false, leave the data ref at its current (empty) value, and `return` — do not partially apply. This is the actual security-relevant line: it's what makes a spoofed client-side auth state harmless, since the real data only ever appears after a *successful* authenticated response.

## Verifying you actually removed the leak

Deleting the source module is necessary but not sufficient proof — always confirm the bytes are gone from the shipped artifact:
1. Build the production client bundle.
2. Grep it for a **canary string that is provably unique to the deleted data** — check it doesn't also appear in JSDoc comments/examples in files you *kept*, and isn't a value your own formatting code could independently produce at runtime (e.g. a big round number). The safest canary is an opaque reference code or ID that only ever existed in the deleted data, never as example text anywhere else in the kept source tree.
3. A zero-match `grep -rl <canary> .output/` (client and server chunks) is the actual proof; a passing unit test or a correct-looking browser render is necessary but not sufficient on its own.

## Adjacent gotcha: any code that derives its "known set of IDs/keys" from the hardcoded module

If some other utility (e.g. a mock-id-to-real-name mapping function) imports the hardcoded constants purely to enumerate the set of mock IDs/labels it knows about, change its signature to accept that set as a parameter instead of importing it — otherwise deleting the hardcoded module breaks that utility silently (empty enumeration, not a compile error) or forces you to keep a dead re-export around.
