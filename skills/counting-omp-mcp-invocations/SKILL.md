---
name: counting-omp-mcp-invocations
description: "Count and verify Oh My Pi (OMP) MCP tool invocations correctly — key on args.path (xd:// URLs), never toolName — and get an authoritative tool inventory from agent.db's mcp_tools cache instead of the model's narrated tool list. Use when verifying an MCP server is actually invoked, auditing whether a server is mounted/reachable, or checking that a config change did not regress other servers."
---

# Counting and verifying OMP MCP invocations

Two counting methods that look reasonable and are both wrong. Use the ones below instead.

## Trap 1: MCP calls are not `toolName`-keyed

MCP tools are dispatched as a **generic write** to an internal URL. An MCP call appears as:

```json
{"type":"tool_execution_start","toolName":"write",
 "args":{"path":"xd://mcp__sequentialthinking_sequentialthinking",
         "content":"{\"thought\":\"...\"}"}}
```

So a counter that keys on `toolName == "mcp__..."` **always returns 0** — including on runs that used the tool heavily. Count on the path:

```python
import json, collections, sys
c = collections.Counter()
for line in open(sys.argv[1]):
    line = line.strip()
    if not line:
        continue
    try:
        ev = json.loads(line)
    except ValueError:
        continue
    if ev.get("type") == "tool_execution_start":
        p = (ev.get("args") or {}).get("path", "")
        c[(ev.get("toolName"), p if str(p).startswith("xd://") else "")] += 1
for k, v in c.items():
    print(v, k)
```

Run `omp -p --no-session --auto-approve --mode=json "<prompt>"` and parse that stream. Built-in tools show as their real name (`read`, `bash`) with an empty path component; MCP shows as `write` + an `xd://` path.

Beware substring greps over the raw transcript: tool *schemas and descriptions* are echoed into the stream, so `grep -c sequentialthinking` matches text, not calls. Filter on event type first.

## Trap 2: the model's tool list is not an inventory

Asking "list every MCP tool" returns a **narration**: it under-reports (servers silently missing), and it emits bare server-local names (`sequentialthinking`, `get-changelog`) rather than `mcp__`-prefixed runtime names. Do not diff narrated lists before/after a config change — you will draw wrong regression conclusions.

Authoritative inventory is OMP's own tool cache:

```bash
cd ~/.omp/agent && sqlite3 agent.db \
  "select key, value from cache where key like 'mcp_tools:%';"
```

Keys are `mcp_tools:<server>`. Each value is JSON `{"version":1,"configHash":"...","tools":[{"name":"..."}]}`. Count tools as `len(tools)` per server.

## Trap 3: a stale cache entry does not mean a broken server

`configHash` mismatches (i.e. `MCPToolCache.get` returning `null`) only mean the **fast-startup path** falls back to a live connect — which is then subject to the 250ms startup gate (`manager.ts STARTUP_TIMEOUT_MS = 250`). It says nothing about whether the server works. Most stale entries are simply unreachable or auth-gated servers (HTTP 401, unset env vars). Never report cache staleness as a regression.

To prove a config edit did **not** regress other servers, use OMP's real cache class (do not reimplement its hashing — a hand-rolled `stableStringify` will not match and will produce a confusing false negative):

```ts
await import("<omp>/src/discovery/index.ts");   // registers providers
const cmod = await import("<omp>/src/mcp/config.ts");
const tmod = await import("<omp>/src/mcp/tool-cache.ts");
const { Database } = await import("bun:sqlite");
const db = new Database("/Users/<you>/.omp/agent/agent.db");
const storage = { getCache: k => (db.query("select value from cache where key = ?").get(k))?.value ?? null };
const cache = new tmod.MCPToolCache(storage);
const cfg = await cmod.loadAllMCPConfigs(cwd, { filterExa: true, filterBrowser: true });
// non-null == config unchanged and server healthy at last connect
const defs = await cache.get(serverName, cfg.configs[serverName]);
```

`<omp>` is `~/.bun/install/global/node_modules/@oh-my-pi/pi-coding-agent`. Run with `bun`.

## The 250ms startup gate

Servers not connected within 250ms with no cache entry are left in flight and register later via `onToolsChanged` — so a single `omp -p` run can exit before a slow stdio server appears. Since a `npx`-launched server costs 0.4–0.6s cold, verify with **two consecutive runs** (the first writes the cache, the second surfaces a deferred tool) or an interactive `/mcp list`. Do not conclude failure, or start editing config, from one run.

Also note OMP's provider precedence when a server appears in several tools' configs: OMP-native (`~/.omp/agent/mcp.json`) is priority 100, Claude Code 80. Dedupe is first-wins by exact name, so a same-named OMP-native entry cleanly shadows an entry in `~/.claude.json` **within OMP only**, leaving the other tool's config untouched.

## Config-change checklist

1. Edit the OMP-native file (user scope: `~/.omp/agent/mcp.json`).
2. `loadAllMCPConfigs` → confirm the server resolves and its `source.provider` is `native`.
3. Confirm the pre-existing entries still resolve (they will — one added key cannot alter another server's config).
4. Two consecutive `omp -p` runs; count dispatches by `args.path`.
5. Confirm no other tool's config file was rewritten (check mtimes).
