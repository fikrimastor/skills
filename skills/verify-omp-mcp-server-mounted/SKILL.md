---
name: verify-omp-mcp-server-mounted
description: "Verify an MCP server is actually mounted, connected, and callable in the Oh My Pi (OMP) harness — using agent.db's tool cache and real xd:// dispatch instead of the model's narrated tool list, which under-reports. Use when adding an MCP server to OMP, when an MCP tool seems missing, or when auditing \"did my MCP change break anything\"."
---

# Verify an MCP server is really mounted in OMP

OMP = Oh My Pi (`omp`). Config lives in `~/.omp/agent/mcp.json` (user) or `.omp/mcp.json` (project).

## The two traps that produce wrong verdicts

**1. Never trust the model's narrated tool list.** Asking `omp -p "list your MCP tools"` under-reports (it omitted `graft`, `pencil`, `phpstorm`, `postgres`, `playwright` in a real session). It also lists *bare* names (`sequentialthinking`), not `mcp__`-prefixed ones, and OMP may filter the tool-result display so the raw list is invisible. Do not diff before/after inventories this way.

**2. MCP tool calls are NOT `toolName` "mcp__…".** They are dispatched as:

```json
{"type":"tool_execution_start","toolName":"write",
 "args":{"path":"xd://mcp__<server>_<tool>","content":"{...}"}}
```

A counter keyed on `toolName` (or on the substring `mcp__` in the raw line) returns **0 forever**. Key on `args.path` starting with `xd://mcp__`, or on `write` events.

```python
import json, collections
c = collections.Counter()
for line in open(transcript):
    line = line.strip()
    if not line: continue
    try: ev = json.loads(line)
    except: continue
    if ev.get("type") == "tool_execution_start":
        p = (ev.get("args") or {}).get("path", "")
        c[(ev["toolName"], p if str(p).startswith("xd://") else "")] += 1
print(c)   # ('write','xd://mcp__server_tool'): 3
```

Run probes with `omp -p --no-session --auto-approve --mode=json`.

## Source of truth: agent.db tool cache

```bash
sqlite3 ~/.omp/agent/agent.db "select key from cache where key like 'mcp_tools:%';"
```

Each value is `{"version":1,"configHash":"<sha256>","tools":[{"name":...}]}`. Count tools here — that is real mount data.

## Is it a live server or only a cache entry?

`MCPManager` has a 250ms startup gate (`STARTUP_TIMEOUT_MS` in `mcp/manager.ts`). A server that hasn't connected in that window is served from cache as a `DeferredMCPTool`, else left in flight.

Prove the cached entry matches the *current* config by using the real cache class — do **not** reimplement `stableStringify` (it will not match and gives a false negative):

```ts
await import("~/.bun/install/global/node_modules/@oh-my-pi/pi-coding-agent/src/discovery/index.ts");
const cmod = await import(".../src/mcp/config.ts");       // loadAllMCPConfigs
const tmod = await import(".../src/mcp/tool-cache.ts");   // MCPToolCache
const { Database } = await import("bun:sqlite");
const db = new Database("/Users/<you>/.omp/agent/agent.db");
const storage = { getCache: k => (db.query("select value from cache where key = ?").get(k))?.value ?? null };
const cache = new tmod.MCPToolCache(storage);
const cfg = await cmod.loadAllMCPConfigs(cwd, { filterExa: true, filterBrowser: true });
const defs = await cache.get("myserver", cfg.configs["myserver"]);
console.log(defs !== null);   // true => cache valid for current config, fast path works
```

Importing `discovery/index.ts` first is **required** — loading a capability without it returns 0 items.

## Precedence: override another tool's server without editing it

OMP-native config = priority 100; Claude Code (`~/.claude.json`) = 80. Capability dedupe is **first-wins by exact server name** (`capability/mcp.ts`, `equivalent: isSameMCPConnection`).

To replace a stale server another client owns, **reuse the exact same name** in the OMP-native file. A differently-named key gives you TWO live servers and duplicate tools. Verify the winning source:

```ts
console.log(cfg.sources["myserver"].provider, cfg.sources["myserver"].path);
// -> "native" "/Users/you/.omp/agent/mcp.json"
```

## Verification ladder (cheap → authoritative)

1. **Liveness** — raw JSON-RPC over stdin, no client needed:
   ```bash
   printf '%s\n' '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2025-06-18","capabilities":{},"clientInfo":{"name":"probe","version":"0"}}}' '{"jsonrpc":"2.0","method":"notifications/initialized"}' '{"jsonrpc":"2.0","id":2,"method":"tools/list","params":{}}' | npx -y <package>
   ```
2. **Discovery** — `loadAllMCPConfigs` (after importing `discovery/index.ts`).
3. **Cache** — `MCPToolCache.get` as above.
4. **Real call** — `omp -p --mode=json "…call the tool…"`, then count `xd://mcp__…` write paths. 2 consecutive runs to rule out the startup-gate race.

## Regression scope

- "NO/STALE cache" for a server means the fast-startup path falls back to a live connect — **not** that it broke. Servers showing this are usually auth-gated or unreachable (401s, unset env), pre-existing.
- Adding one key to `~/.omp/agent/mcp.json` cannot alter any other server's resolved config.
- Prefer a real call to an untouched server (e.g. an existing one's changelog tool) as the no-regression evidence.

## Agent guidance for an OMP MCP tool

Rules live in `~/.omp/agent/rules/<name>.md`. Bucketing (`capability/rule-buckets.ts`): TTSR (`condition`/`astCondition`) > `alwaysApply: true` > rulebook (`description` only, listed in prompt, read on demand via `rule://`).

- Rulebook-only guidance is **advisory and often ignored** — it is listed, not injected.
- Use `alwaysApply: true` when the guidance must actually be in context.
- `dedupeAlwaysApplyRules` (system-prompt.ts) drops a rule only if its full normalized block sequence — split on blank lines — appears **contiguously** in a prompt source. Adding an extra section (e.g. a `### Tool reference`) breaks the match, so identical text in AGENTS.md AND a rule double-injects. Keep the rule body a verbatim match if you want dedupe.
- `tools.xdevDocs` defaults to `builtins`, so MCP schemas stay on-demand rather than shipping every request.
