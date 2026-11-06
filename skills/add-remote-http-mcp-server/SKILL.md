---
name: add-remote-http-mcp-server
description: "Register a hosted HTTP MCP server across OMP, Claude Code, and Codex with the right per-client dialect, and verify it by an actual tool call rather than by reading config. Use when asked to 'add MCP server name' or wire a URL like https://nuxt.com/mcp into coding agents."
---

# Add a hosted HTTP MCP server to coding agents

Applies to remote/HTTP MCP endpoints (e.g. `https://nuxt.com/mcp`). Each client
takes a DIFFERENT key for the same server; pasting one client's block into
another silently produces a stdio server with no `command`, or a schema error.

## 1. Probe the endpoint first

Never wire a URL without confirming it speaks MCP and is reachable:

```bash
curl -sS -i -X POST <URL> \
  -H 'Content-Type: application/json' \
  -H 'Accept: application/json, text/event-stream' \
  -d '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2025-06-18","capabilities":{},"clientInfo":{"name":"probe","version":"0"}}}' \
  --max-time 20
```

Expect HTTP 200 + a `result.serverInfo.name`. Then `method: "tools/list"` to
learn the real tool names (they are usually kebab-case, e.g.
`list-documentation-pages`, not snake_case).

## 2. Per-client dialect — the key differences

| Client | File | Shape |
|---|---|---|
| OMP | `~/.omp/agent/mcp.json` (user) or `.omp/mcp.json` (project) | `"type": "http"`, `"url": ...` |
| Claude Code | `~/.claude.json` user scope or `.mcp.json` project | `"type": "http"`, `"url": ...` |
| Codex | `~/.codex/config.toml` | `[mcp_servers.NAME]` with `url = "..."`, `enabled = true` |
| OpenCode | `~/.config/opencode/opencode.json` | `"type": "remote"`, `"url": ...` |
| Cursor / VS Code | `.cursor/mcp.json` / `.vscode/mcp.json` | `"type": "http"` (`servers` key for VS Code, not `mcpServers`) |

OpenCode is the outlier: it uses `remote`/`local`, not `http`/`stdio`. Writing
`"type": "http"` there is wrong.

## 3. Write only what is needed

- **User scope, not project scope**, when the server is generic (docs, language
  reference). Project files like `.mcp.json` in a repo are usually tracked in
  git — adding a universal server there creates permanent diff churn in every
  checkout.
- **Check before writing.** Grep each target file for the server name/URL first;
  a client may already have it, and a duplicate entry is pure churn:

```bash
grep -c -i '<url-or-name>' ~/.claude.json ~/.codex/config.toml \
  ~/.config/opencode/opencode.json 2>/dev/null
```

- Prefer the client's own CLI when it exists, so the file is written correctly
  and any needed bookkeeping happens: `claude mcp add --scope user --transport http <name> <url>`.

## 4. Verify by behavior, not by reading config

Reading the file back proves nothing; a malformed or mis-discovered entry still
looks correct. Prove it with a real tool call in a fresh non-interactive run:

```bash
omp -p --auto-approve "Call the <name> MCP tool '<real-tool-name>' and report only the result. If no <name> MCP tool exists, reply exactly NO_<NAME>_MCP."
```

A concrete number back = wired. `NO_..._MCP` = not wired, regardless of what the
config file says. For Claude Code, `claude mcp list` shows
`<name>: <url> (HTTP) - ✔ Connected`. For Codex TOML, validate it parses:

```bash
python3 -c "import tomllib;d=tomllib.load(open('<path>','rb'));print(d['mcp_servers'])"
```

## 5. A/B test when a client allegedly inherits another client's config

Docs claiming cross-tool MCP discovery are unreliable. To settle it: run the
verification prompt, then move the client's own config file aside and run the
SAME prompt again. Same result → the file is not load-bearing. `NO_..._MCP` →
the file is required, and the cross-tool claim is false for that client.

```bash
mv ~/.omp/agent/mcp.json /tmp/mcp-backup.json   # disable
# ... re-run verification ...
mv /tmp/mcp-backup.json ~/.omp/agent/mcp.json   # ALWAYS restore
```

Always restore the file in the same shell invocation you moved it, or the
client is left silently broken.

## 6. In-session visibility

The MCP registry loads at session start. New or changed entries do NOT appear in
the current session until `/mcp reload` (OMP) — new sessions pick them up
automatically. Report this as the user's next action rather than claiming the
server is immediately usable in the session where it was added.
