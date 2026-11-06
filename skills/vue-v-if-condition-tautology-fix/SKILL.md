---
name: vue-v-if-condition-tautology-fix
description: "Diagnose and fix a Vue/Nuxt v-if (or v-show) condition that visibly does not take effect, especially !== combined with ||, and verify it with a component-mounting regression test."
---

# Vue `v-if` condition tautology — diagnose, fix, verify

Use when a Vue/Nuxt conditional render "doesn't work": an element shows (or hides) on every route regardless of the condition, or adding a second value to the condition changes nothing.

## 1. Suspect the logic before the framework

Check for a tautology — a condition that is true for *all* inputs:

```
// ALWAYS TRUE — no string equals both 'a' and 'b'
(a !== 'a') || (a !== 'b')

// ALWAYS FALSE
(a === 'a') && (a === 'b')
```

The classic shape is `!==` with `||`. The apparent intent is "hide on these paths", which needs `&&` (or better, membership). Do not debug routing, router normalisation, or `route.path` reactivity first — this is pure boolean arithmetic and is usually the whole bug.

Also check whether the condition was already dead: a single-value version (`x !== 'a' || x !== 'b'`) fails identically, so a prior commit that "hid" the element may never have worked.

## 2. Fix structurally, not by flipping the operator

A `&&` chain scales badly and re-invites the error. Move the decision into a named predicate so the template holds one non-negatable expression:

```ts
// navGroups.ts (or any sibling util module)
const SIDEBAR_HIDDEN_PATHS: Record<string, true> = {
  "/app/dashboard": true,
  "/app/dashboard-wip": true,
};

export function shouldShowSidebar(path: string): boolean {
  const normalized =
    path.length > 1 && path.endsWith("/") ? path.slice(0, -1) : path;
  return SIDEBAR_HIDDEN_PATHS[normalized] !== true;
}
```

```vue
<script setup lang="ts">
const showSidebar = computed(() => shouldShowSidebar(route.path));
</script>

<template>
  <CPSidebar v-if="showSidebar" />
</template>
```

Notes:
- Prefer `Record<string, true>` over `Set` for a static, literal, string-keyed lookup unless the repo already uses `Set` for this.
- Normalise trailing slashes only if a sibling helper (e.g. an `activeNavFromPath`) already treats `/dashboard` and `/dashboard/` as the same thing — otherwise the two helpers disagree about the route.
- Keep it in a *pure* util module, not the layout, so it stays testable and reusable by other shells.
- If a repo rule forbids one-line wrapper functions, inline the normalisation inside the predicate rather than extracting a helper.

## 3. Verify by mounting the component — helper tests are not enough

A test of `shouldShowSidebar` alone **passes against the broken layout**, because the bug is in the template expression. Mount the real component and assert on rendered output:

```ts
import { computed, ref } from "vue";
import { mount } from "@vue/test-utils";
import AppLayout from "@/layouts/app.vue";

vi.mock("@/components/Header.vue", () => ({
  default: { name: "Header", template: '<header data-testid="header" />' },
}));
vi.mock("@/components/Sidebar.vue", () => ({
  default: { name: "Sidebar", template: '<aside data-testid="sidebar" />' },
}));

// Raw Vitest has no Nuxt auto-imports — stub before mounting.
vi.stubGlobal("computed", computed);
vi.stubGlobal("ref", ref);

function mountAt(path: string) {
  vi.stubGlobal("useRoute", vi.fn(() => ({ path, query: {}, params: {}, meta: {} })));
  return mount(AppLayout, { slots: { default: '<main data-testid="slot" />' } });
}

it("hides the rail on full-bleed routes", () => {
  for (const path of ["/app/dashboard", "/app/dashboard-wip", "/app/dashboard/"]) {
    expect(mountAt(path).find('[data-testid="sidebar"]').exists()).toBe(false);
  }
});

it("renders rail and content elsewhere", () => {
  const wrapper = mountAt("/app/other");
  expect(wrapper.find('[data-testid="sidebar"]').exists()).toBe(true);
  expect(wrapper.find('[data-testid="slot"]').exists()).toBe(true);
});
```

Then **prove the guard is real**: revert the template to the old tautology, confirm the test FAILS, restore the fix, confirm it passes, and run the full suite.

## Pitfalls

- A test that only exercises the extracted predicate does not guard the template — the original bug survives it.
- Raw Vitest + `@vitejs/plugin-vue` (no `@nuxt/test-utils`) means `computed`, `ref`, `useRoute`, `useState`, `navigateTo`, etc. must be stubbed manually; missing `computed`/`ref` surfaces as `ReferenceError: ref is not defined` from an unrelated imported module.
- Mount the *imported* component, not a stub of it, or the assertion is vacuous.
- If the hidden-route list must grow (A/B variants), add the path to the predicate — do not duplicate the whole page component just to hide chrome.
