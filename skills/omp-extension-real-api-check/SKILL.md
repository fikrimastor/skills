---
name: omp-extension-real-api-check
description: "Diagnose an OMP plugin extension/hook that fails to load (\"Failed to load extension ...: pi.method is not a function\", or a pi.* call sitting behind an unexplained @ts-expect-error) by checking the call against the real installed @oh-my-pi/pi-coding-agent ExtensionAPI/HookAPI surface instead of a locally-invented type."
---

## When to use

- OMP startup logs `Warning: Failed to load extension <path>: <TypeError...>` for a plugin under `~/.omp/plugins/node_modules/<plugin>/`.
- You're writing or reviewing an OMP plugin's `extensions/*.ts` (declared in `package.json`'s `omp.extensions`) or `hooks/*.ts` (`omp.hooks`), and a `pi.<method>(...)` call looks plausible but you're not 100% sure it's real.
- You see `// @ts-expect-error: <method> may not be on <Type>'s typed surface` next to a `pi.*` call. **This comment is a smoking gun** — the compiler already caught a fictional API and someone suppressed it instead of fixing it. Stop and investigate before trusting that call.

## Why this happens

Extension authors often hand-roll a narrow structural type for `pi` (e.g. `pi: { onCommand: (name, handler) => void }`) instead of importing the SDK's real type, usually because they're recalling an API shape from memory/docs/another framework rather than checking the installed package. TypeScript then can't catch the mismatch (or gets silenced via `@ts-expect-error`), and the bug only surfaces at runtime as a `TypeError` — often thrown *before* the rest of the factory function runs, which silently kills every hook/command the same file was about to register, not just the broken call site.

## Procedure

1. **Find the resolved plugin file.** `~/.omp/plugins/node_modules/<plugin>` is typically a symlink to the real source repo (check with `readlink -f`). Always edit the symlink target, not the mirror.
2. **Find the exact installed SDK version's source** (not docs, not memory):
   ```bash
   find / -type d -path "*@oh-my-pi/pi-coding-agent*" -not -path "*/Trash/*" 2>/dev/null
   ```
   Look under `.../src/extensibility/{extensions,hooks}/types.ts` — these are TypeScript SOURCE (not just `.d.ts`), so you can read real implementations, not just signatures.
3. **Know which surface applies:**
   - Files declared under `omp.extensions` in the plugin's `package.json` receive a `ConcreteExtensionAPI` implementing the `ExtensionAPI` interface (`extensibility/extensions/types.ts`). Default export signature: `(pi: ExtensionAPI) => void | Promise<void>`.
   - Files declared under `omp.hooks` receive a narrower `HookAPI` (`extensibility/hooks/types.ts`). Default export: `(pi: HookAPI) => void`.
   - These are DIFFERENT, deliberately non-overlapping surfaces (hooks omit session-control/UI-mutation methods extensions have, by design — see the `fallow-ignore-next-line code-duplication` comments in the source for why they aren't merged). **`ExtensionAPI` is only exported from the `extensions` subpath, not `hooks`** — `import type { ExtensionAPI } from "@oh-my-pi/pi-coding-agent/extensibility/hooks"` silently resolves to nothing useful (grep the `hooks/` source tree for `ExtensionAPI` to confirm zero matches) but won't crash, because `import type` is erased by Bun's transpiler without resolving the module — so this class of bug hides completely until someone actually typechecks.
4. **Cross-check every `pi.<method>(...)` call** against the real interface block (read the whole `ExtensionAPI`/`HookAPI` interface, not just a grep hit — methods are listed together under `// === Event Subscription ===`, `// === Command, Shortcut, Flag Registration ===`, etc.). Frequently-invented-vs-real mismatches:
   - There is **no** `pi.onCommand`. Real: `pi.registerCommand(name, { description?, getArgumentCompletions?, handler })`.
   - A registered command's `handler` signature is `(args: string, ctx: ExtensionCommandContext) => Promise<void>` — `args` is the RAW remaining string after the command name (you parse it yourself), not a pre-parsed object, and the handler returns `void` — you cannot `return` a string to show the user. Output goes through `ctx.ui.notify(message, "info" | "warning" | "error")`.
   - `pi.on(event, handler)` is real and event names are enumerated as overloads on `ExtensionAPI`/`HookAPI` — check the exact event-name string there rather than guessing.
5. **Find a real reference usage in the SDK itself** rather than trusting your reconstruction: `grep -rn '\.registerCommand(' <sdk-src>` turns up first-party call sites (e.g. `src/autoresearch/index.ts`) showing the full working pattern including `ctx.ui.notify` for output.
6. **Regression-test the wiring, not just the extracted logic.** A test suite that only unit-tests pulled-out pure functions (e.g. `handleRetain`, `handleSynthesizeCommand`) never calls the default-exported factory with anything resembling `pi`, so it can't catch a fictional-API crash. Add a test that builds a minimal fake matching ONLY the real method names/shapes the factory actually calls (e.g. `{ on(event, handler) {...}, registerCommand(name, opts) {...} }`), calls the factory with it, and asserts no throw plus the expected registrations happened. Confirm it fails against the buggy code first (reproduces the exact reported error) before applying the fix.
7. **Cheapest true end-to-end check**, no need to trust tests alone: spawn a real ephemeral session and grep its startup output —
   ```bash
   omp -p --no-session --no-tools --max-time 20 "reply OK" 2>&1 | grep -i "failed to load extension"
   ```
   Empty output = the plugin loaded cleanly. Extension/hook loading happens before the model turn, so this catches load failures even though it also spends one small real model call.

## Testability tip (pairs with a `ts-no-any`-style project rule)

If a helper only calls one or two `ExtensionAPI` methods, don't require the whole interface — narrow the parameter to `Pick<ExtensionAPI, "registerCommand">` (or whichever methods are used). A test fake then satisfies the parameter type structurally with **zero casts**. Reserve `as unknown as ExtensionAPI` (with a one-line comment) for the actual extension factory's top-level `pi: ExtensionAPI` parameter, where the full interface is genuinely the contract and a partial stub is a deliberate, well-understood simplification.
