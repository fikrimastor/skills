---
name: verifying-omp-mcp-server-wiring
description: "Verify an Oh My Pi (OMP) MCP server is actually wired and used — count invocations by the xd:// args.path (not toolName), read the authoritative tool inventory from ~/.omp/agent/agent.db's mcp_tools cache, and prove existing servers were not staled by a config change. Use when asked to add/verify an MCP server in OMP, or when a tool appears missing or unused."
---

# Verify an OMP MCP server is wired and used

Two counters lie: `toolName` and the model's own narration. Use neither.

## 1. Add the server

OMP-native config wins precedence, and dedupe is first-wins by exact name:
`~/.omp/agent/mcp.json` (user, priority 100) beats `~/.claude.json`
(Claude Code, priority 80). Reusing an existing server's exact name shadows
it inside OMP only, without editing the other tool's file.

```json
{ "type": "stdio", "command": "npx", "args": ["-y", "<package>"] }
```

## 2. Prove the server speaks MCP (before wiring)

```bash
printf '%s\n' \
 '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2025-06-18","capabilities":{},"clientInfo":{"name":"p","version":"0"}}}' \
 '{"jsonrpc":"2.0","method":"notifications/initialized"}' \
 '{"jsonrpc":"2.0","id":2,"method":"tools/list","params":{}}' \
 | npx -y <package>
```

Expect `result.serverInfo.name` and the real tool names (often kebab-case).

## 3. Count real invocations CORRECTLY — the critical step

OMP dispatches MCP calls as a **`write` tool call to an `xd://` URL**:

```json
{"type":"tool_execution_start","toolName":"write","args":{"path":"xd://mcp__<server>_<tool>","content":"{...}"}}
```

So keying on `toolName` returns **0 always** and yields a false
"the tool was never used" conclusion. Scan `args.path`:

```python
import json
c=collections.Counter()
for line in open(session_jsonl):
    ev=json.loads(line)
    if ev.get("type")=="tool_execution_start":
        c[(ev["toolName"], (ev.get("args") or {}).get("path",""))]+=1
```

Also: `--mode=json` prints the stream; a grep for the server name over raw
JSON matches *schema and description text*, not calls. Filter on event type
AND path.

Run the same probe twice. The startup gate is a 250ms race, but a cache hit
serves a `DeferredMCPTool` immediately — a cached stdio server is callable on
the first run. Two runs is cheap insurance against a false negative.

## 4. Authoritative inventory (never trust model narration)

A model asked to list its MCP tools returns an **incomplete** list. Read the
cache instead:

```bash
sqlite3 ~/.omp/agent/agent.db \
  "select key from cache where key like 'mcp_tools:%';"
```

Each value is JSON with a `tools[]` array. For a correctness check, use the
real class rather than a reimplementation — its `configHash` uses OMP's own
`stableStringify`, which a hand-rolled version will not match:

```ts
await import("<omp>/src/discovery/index.ts"); // registers providers (required)
const { MCPToolCache } = await import("<omp>/src/mcp/tool-cache.ts");
const cfg = await config.loadAllMCPConfigs(cwd, { filterExa: true, filterBrowser: true });
const db = new Database("~/.omp/agent/agent.db");
const cache = new MCPToolCache({ getCache: k => db.query(
  "select value from cache where key = ?").get(k)?.value ?? null });
await cache.get(name, cfg.configs[name]); // null => config changed/stale
```

`null` for a pre-existing server means its config was staled — that is the
regression signal. `null` for servers that already failed auth (401) or have
an unset env var is pre-existing, not caused by your change.

Note: load the `discovery/index.ts` module first or `loadCapability` returns
0 items — provider registration is an import side effect.

## 5. Scoping guidance (optional-tool instructions)

- Rulebook rule (`description` present, no `alwaysApply`): only name+description
  are listed; the model often never reads it — advisory only.
- `alwaysApply: true`: full content injected into the prompt.
- `dedupeAlwaysApplyRules` drops a rule only when its **entire** block sequence
  (split on blank lines) appears **contiguously** in a prompt source. Adding any
  extra section absent from the context file breaks dedupe and double-injects.
  Verify with the real `buildSystemPrompt` and count occurrences in the output.

## 6. Report honestly

Invocations require the `xd://` path key. A `0` from a `toolName`-keyed
counter is a measurement bug, not evidence. Guidance being installed is not
evidence it triggers — say which you actually measured.
