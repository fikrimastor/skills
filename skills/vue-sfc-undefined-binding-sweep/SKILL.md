---
name: vue-sfc-undefined-binding-sweep
description: "Audit Vue/Nuxt SFC templates for identifiers not defined in script setup — silent _ctx.* lookups that pass vitest/CI — by compiling with @vue/compiler-sfc and collecting unresolved bindings. Use when reviewing a Vue/Nuxt change touching page↔component prop/event wiring, or when a template prop/handler may be misspelled."
---

# When to use

Reviewing a Vue 3 / Nuxt SFC change where template bindings (`:prop="x"`, `@event="handler"`) may reference identifiers that don't exist in `<script setup>`. This class of defect **cannot fail vitest, lint, or the production build**: the SFC compiler emits unknown identifiers as `_ctx.<name>` lookups, so the app silently renders undefined props / drops events (dev-only warning: "Property X was accessed during render but is not defined on instance"). Spot-checking with grep is not enough — this sweep is exhaustive and uses the compiler's own binding resolution.

# Procedure

1. Write `/tmp/sweep-sfc-bindings.cjs` (adjust `repo` path; see script below).
2. Run `node /tmp/sweep-sfc-bindings.cjs` from the repo (needs `@vue/compiler-sfc` in `node_modules`, which any Nuxt app has).
3. For each changed SFC the output lists `unresolved (_ctx.*)` — every one is an identifier the compiler could not resolve from `<script setup>`. Cross-check each hit against the script's declarations; a genuine miss is a real bug.
4. Interpret: zero hits = template fully wired. Hits on `$`-prefixed names (instance props like `$slots`) and Nuxt **auto-imported composables used only in templates** are false positives — check `.nuxt/types/imports.d.ts` before ruling.

# Why it works (and what it catches)

- `compileScript(descriptor, { inlineTemplate: true })` inlines the render function into the setup scope, so resolvable bindings appear as plain identifiers (`.value` rewrites for refs) while unresolved ones are prefixed `_ctx.`.
- Caught in practice: PR #883 — `diagnostic-plots-dashboard.vue` bound `wellTypedNarrative`/`wellTypingComplete`/`activeCapabilityName`/`handleViewFullAnalysis`/`handleBackToCard`, none defined; 650/650 tests passed because the page spec stubbed the child panel (`WellAiInsightDrawer: true`). The Bubble Maps page, same PR, compiled to zero hits.
- Bonus: the compiled output shows exactly how assignments were rewritten, e.g. `@update:chat-input="alias = $event"` where `const alias = chat.input` compiles to `alias.value = $event` — use this to settle "is this ref-write correct?" questions empirically instead of from memory.

# Script

```js
const repo = "/ABSOLUTE/PATH/TO/REPO";
const { parse, compileScript } = require(repo + "/node_modules/@vue/compiler-sfc");
const fs = require("fs");
const files = process.argv.slice(2); // pass changed SFC paths, or hardcode a list
for (const rel of files) {
  const file = repo + "/" + rel;
  const { descriptor, errors } = parse(fs.readFileSync(file, "utf8"), { filename: file });
  if (errors?.length) { console.log(rel, "parseErrors:", errors.map(e => e.message)); continue; }
  let compiled;
  try { compiled = compileScript(descriptor, { inlineTemplate: true, id: "sweep" }); }
  catch (e) { console.log(rel, "compileError:", String(e.message || e)); continue; }
  const hits = new Set();
  for (const m of compiled.content.matchAll(/_ctx\.([A-Za-z_$][\w$]*)/g)) hits.add(m[1]);
  console.log(rel, "unresolved:", JSON.stringify([...hits].sort()));
}
```

Extend the script with a `needle` filter over `compiled.content.split("\n")` to print codegen evidence for specific bindings (ref rewrites, `_unref(...)`, handler closures).

# Limits

- Reports bindings unresolved at **compile time**; it cannot prove semantic correctness (right ref, right event name, handler doing the right thing).
- Components registered globally by Nuxt resolve via `resolveComponent`, so they do not appear as hits — absence of a hit does not prove the component exists.
- Template-only usage of a page `defineExpose`'d value or a slot prop is out of scope.
