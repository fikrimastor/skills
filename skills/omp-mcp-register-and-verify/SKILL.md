---
name: omp-mcp-register-and-verify
description: "Register a stdio MCP server into Oh My Pi (OMP) and prove it works — covers the 250ms startup-gate false negative, the xd:// write invocation path, always-apply rule dedupe against context files, and name-shadowing to replace a stale duplicate without editing another tool's config. Use when asked to add an MCP server to OMP, when a newly added server seems not to load, or when scoping MCP guidance to be optional rather than per-request."
---

# Register and verify a stdio MCP server in Oh My Pi

Verified against OMP 18.2.10 (`@oh-my-pi/pi-coding-agent`, installed at
`~/.bun/install/global/node_modules/@oh-my-pi/pi-coding-agent`).

## 1. Pick the right file and check for an existing duplicate first

User scope: `~/.omp/agent/mcp.json` (follows `--profile`). Project scope:
`.omp/mcp.json`. Shape mirrors the existing entries — copy the current file's
format rather than inventing one:

```json
{
  "$schema": "https://raw.githubusercontent.com/can1357/oh-my-pi/main/packages/coding-agent/src/config/mcp-schema.json",
  "mcpServers": {
    "NAME": { "type": "stdio", "command": "npx", "args": ["-y", "PKG"] }
  }
}
```

**Before writing, grep every tool config for the server name.** OMP imports
`~/.claude.json`, `~/.codex/config.toml`, `opencode.json`, `~/.cursor/mcp.json`,
marketplace plugins, and more. A name already present there means you are about
to shadow (or duplicate) it:

```bash
grep -rn '<name>' ~/.claude.json ~/.codex/config.toml \
  ~/.config/opencode/opencode.json ~/.cursor/mcp.json 2>/dev/null | head
```

### Shadowing rule (this is the whole naming decision)

Provider priority: OMP-native `100` > omp-plugins `90` > Claude Code `80` > … .
MCP dedupe is first-wins by **exact server name** (`capability/mcp.ts`;
`isSameMCPConnection` only suppresses *differently-named* identical connections).

- Reuse the **same name** → your entry shadows the older one inside OMP, and the
  other tool's config is left completely untouched. Correct when replacing a
  stale duplicate.
- Use a **different name** → two live servers, duplicate tools, wasted startup.

Confirm which source won by loading the config programmatically (§3) and reading
`_source.provider` / `_source.path` — not by re-reading the JSON you just wrote.

## 2. Confirm the server itself starts

Probe the raw MCP handshake before blaming OMP:

```bash
printf '%s\n' \
 '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2025-06-18","capabilities":{},"clientInfo":{"name":"probe","version":"0"}}}' \
 '{"jsonrpc":"2.0","method":"notifications/initialized"}' \
 '{"jsonrpc":"2.0","id":2,"method":"tools/list","params":{}}' \
 | timeout 90 npx -y PKG
```

Expect `serverInfo.name` and a `tools` array. Note the **exact** tool name — the
runtime id becomes `mcp__<server>_<tool>` with both components lowercased and
sanitized to `[a-z_]`.

## 3. Inspect OMP's real discovery (not the file you wrote)

OMP ships TypeScript sources, so you can call its own loader. Provider
registration is a side effect of importing `discovery/index.ts` — import it
FIRST, or `loadCapability` returns zero items:

```ts
await import("<pkg>/src/discovery/index.ts");
const cmod: any = await import("<pkg>/src/mcp/config.ts");
const cfg = await cmod.loadAllMCPConfigs(process.cwd(), {
  enableProjectConfig: true, filterExa: true, filterBrowser: true,
});
console.log(JSON.stringify(cfg.configs["NAME"]));
console.log(cfg.sources["NAME"].provider, cfg.sources["NAME"].path);
```

Pass `filterBrowser: true` to match a live session — omitting it makes
`playwright` appear and inflates the server count, which looks like your change
added a server.

## 4. Verify by a real tool call — never by a tool listing

**The startup gate is a 250ms race** (`mcp/manager.ts` `STARTUP_TIMEOUT_MS=250`).
A server that hasn't connected AND has no cache entry is left in flight and only
registers later via background `onToolsChanged`. `npx` cold start is ~0.4–0.6s,
so a single headless run can exit before your new server ever appears. Evidence:
`graft` is discovered and mounts interactively, yet was missing from a one-shot
`-p` tool listing.

So: a listing run is not proof of failure. Prove it with an actual call:

```bash
omp -p --no-session --auto-approve \
  "Call the <tool> MCP tool and report exactly what it returns. \
   If no such MCP tool exists, reply exactly NO_<NAME>_MCP."
```

A real result = wired. `NO_..._MCP` = not wired. Or run the listing twice — the
first run populates the tool cache in `~/.omp/agent/agent.db`, the second then
surfaces `DeferredMCPTool` entries.

### Invocation shape (needed to read the evidence)

MCP tools are called by **writing JSON args to an `xd://` URL** — not as a
dedicated tool name. In `--mode=json` a genuine call is:

```json
{"type":"tool_execution_start","toolName":"write",
 "args":{"path":"xd://mcp__server_tool","content":"{...}"}}
```

Therefore:

- count `tool_execution_start` events and read `args.path`;
- do NOT grep the raw JSON stream for the server name — it also matches schema
text and tool-result *content*. Asking the model to read a config file that
contains the server name generated 6 false "invocations" for what was one
`read` call.

## 5. Re-check existing servers (no regression)

Diff the tool inventory before/after, and drive one pre-existing server too:

```bash
omp -p ... "List every mcp__ tool name, comma-separated"   # capture before + after
tr ',' '\n' < after.txt | grep '^mcp__' | sort -u > after.txt
comm -23 before.txt after.txt   # removed = regression; should be empty
comm -13 before.txt after.txt   # added = your new tool
```

Then call an unrelated existing tool (e.g. `xd://mcp__nuxt_get_changelog`) to
prove the shared manager still serves it.

## 6. Making the tool "optional", not per-request

Two independent mechanisms:

- `tools.xdevDocs` defaults to `builtins`, so MCP schemas are NOT inlined in the
  system prompt; they are fetched on demand. Leave it alone for optional tools.
- For usage guidance, add a rule. `~/.omp/agent/rules/<name>.md` with
  `alwaysApply: true` injects the body into every request; a rule with only a
  `description` goes to the advisory **rulebook** bucket (listed in the prompt,
  read on demand via `rule://<name>`), which models routinely ignore.

### always-apply dedupe vs. context files

`dedupeAlwaysApplyRules` (`system-prompt.ts:72-81`) drops an always-apply rule
only when its **entire** normalized block sequence appears **consecutively** in a
prompt source; blocks are split on `/\n{2,}/`.

Consequence: keeping the guidance in BOTH `AGENTS.md` and an always-apply rule is
safe **only if the rule body matches the AGENTS.md section block-for-block**. Add
even one extra section to the rule (e.g. a `### Tool reference` block) and the
contiguous match breaks — the guidance injects twice per request in that
workspace. Verify by rendering, not by reasoning:

```ts
await import("<pkg>/src/discovery/index.ts");
const rmod: any = await import("<pkg>/src/capability/index.ts");
const bmod: any = await import("<pkg>/src/capability/rule-buckets.ts");
const sp: any = await import("<pkg>/src/system-prompt.ts");
const res: any = await rmod.loadCapability("rules", { cwd: TARGET });
const mine = res.items.filter((r: any) => r.name === NAME);
const b = bmod.bucketRules(mine, { addRule: () => false }, {});
const r = await sp.buildSystemPrompt({ cwd: TARGET,
  alwaysApplyRules: b.alwaysApplyRules.map((x: any) => ({ name: x.name, content: x.content, path: x.path })),
  rules: [] });
const text = Array.isArray(r.systemPrompt) ? r.systemPrompt.join("\n") : String(r.systemPrompt ?? "");
console.log((text.match(/<a phrase from the guidance>/g) || []).length);
```

Expect exactly `1` both inside the workspace that has the AGENTS.md section and
outside it.

## 7. Honest limits to report

Guidance in context does NOT guarantee invocation. On a strong reasoning model,
three runs of a deliberately hypothesis-testing task, including with
`alwaysApply` text in context, produced **0** invocations — the model reasoned
natively instead. Report this as "wired, discoverable, and scoped as a guardrail
against overuse", not as "the model will now use it". Also discard converse
tests that produced zero tool calls of ANY kind: they cannot distinguish "rule
ignored" from "no tools needed".

## 8. In-session visibility

The registry loads at session start. A new or changed entry appears in a NEW
session, or after `/mcp reload` in the current one — report that as the user's
next action rather than claiming it is already live. `/mcp list`, `/mcp test
<name>`, and `/mcp reconnect <name>` are the interactive diagnostics.
