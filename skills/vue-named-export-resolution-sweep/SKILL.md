---
name: vue-named-export-resolution-sweep
description: "Statically verify that every named import in changed Vue/Nuxt app files resolves to a real export in the target module — catches the class of wrong-import/deleted-export bug that vitest (esbuild interop → undefined) and sometimes vue-tsc both pass, which then 500s the page in the browser."
---

Use when verifying a Vue/Nuxt branch that adds, removes, renames or moves exports — or after any scripted edit that rewrites an import block. Cheap, read-only, and the only gate that catches this class before the browser does.

## Why it is needed

- Vitest/vite-node interop resolves a missing named export to `undefined` rather than erroring. A page can pass 1000+ unit tests and still 500 in the browser with `Importing binding name 'X' is not found` (Vite/Nitro ESM link check).
- Real instance: a page imported `PLATFORM_TOTAL_STROKE` from `@/utils/diagnosticPlotTraceStyle`; it was exported by `@/utils/diagnosticPlotApi`. Every test passed; the page returned `500 Internal Server Error`.
- Running the dev server with a stale module cache can mask or fake it — restart the dev server before concluding.

## Procedure

1. List the app files the branch touches: `git diff --name-only origin/main..HEAD -- app`.
2. For each, extract every `import { ... } from "@/..."` / `from "~/..."`, resolve the specifier against `app/` (try `.ts`, `.js`, and a directory index), and check each imported name against `export (declare )?(async )?(const|let|function|type|interface|class|enum) <name>` or a re-export list `export { ... <name> ... }`.
3. Report every miss as `(file, name, module)`; a positive control (temporarily assert one known-good name fails) proves the sweep is not vacuous.

Reference implementation (Python, no deps):

```python
import re, pathlib, subprocess
files = subprocess.check_output(["git","diff","--name-only","origin/main..HEAD","--","app"], text=True).split()
for fp in files:
    src = pathlib.Path(fp).read_text()
    for m in re.finditer(r'import\s+(?:type\s+)?\{([^}]*)\}\s+from\s+"([@~]/[^"]+)"', src):
        rel = m.group(2)[2:]
        mod = next((c for c in (pathlib.Path("app")/(rel+e) for e in (".ts",".js","")) if c.is_file()), None)
        if not mod: continue
        text = mod.read_text()
        for name in [x.strip().replace("type ","").split(" as ")[0].strip() for x in m.group(1).split(",") if x.strip()]:
            declared = re.search(r'export\s+(declare\s+)?(async\s+)?(const|let|function|type|interface|class|enum)\s+'+re.escape(name)+r'\b', text)
            reexported = re.search(r'export\s*\{[^}]*\b'+re.escape(name)+r'\b', text)
            if not (declared or reexported):
                print(fp, name, "->", mod)
```

## Companions

- Auto-imported Vue components and Nuxt composables are a different failure class (`_ctx.foo` undefined bindings) — see the `vue-sfc-undefined-binding-sweep` skill.
- After any scripted insertion of tests, grep the `it("...")` titles for duplicates: vitest runs same-named tests twice without failing.
- Keep it read-only so it can run while another agent edits the tree.
