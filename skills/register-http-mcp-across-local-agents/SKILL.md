---
name: register-http-mcp-across-local-agents
description: "Register one HTTP MCP server into every local coding agent (OMP, Claude Code, Codex, OpenCode, Cursor, VS Code) using each client's own dialect, then empirically verify mounting instead of trusting config files."
---

# Register an HTTP MCP server across local agents

## When
User says "add <X> mcp <url>" and the machine has several agents installed. Goal: one server, all clients, each verified.

## Step 0 — Probe the endpoint first (30s)

Prove the server exists before touching config:

```bash
curl -sS -i -X POST <URL> \
  -H 'Content-Type: application/json' \
  -H 'Accept: application/json, text/event-stream' \
  -d '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2025-06-18","capabilities":{},"clientInfo":{"name":"probe","version":"0"}}}' \
  --max-time 20
```

200 + `serverInfo.name` = real. Then `tools/list` to enumerate names, and `tools/call` to prove one actually works. Note whether it needs auth (no `auth` block = none).

## Step 1 — Discover what already exists (don't duplicate)

Grep every candidate location for the server name/URL **before** writing:

```bash
for f in ~/.omp/agent/mcp.json ~/.config/opencode/opencode.json \
         ~/.codex/config.toml ~/.claude.json ~/.cursor/mcp.json \
         ~/.codeium/windsurf/mcp_config.json ~/.gemini/settings.json \
         <project>/.mcp.json <project>/.omp/mcp.json; do
  echo "--- $f"; grep -i -c "<name-or-host>" "$f" 2>/dev/null
done
```

Also `claude mcp list` (shows live health per server). Prefer **user scope** over project files when the user works across many repos — project files are tracked and create diff churn.

## Step 2 — Write, using each client's own dialect

The HTTP shape is **not** uniform. Copy-pasting between clients silently breaks:

| Client | File | HTTP shape |
|---|---|---|
| OMP | `~/.omp/agent/mcp.json` | `{"type":"http","url":"..."}` |
| Claude Code | `~/.claude.json` (via CLI) | `{"type":"http","url":"..."}` |
| Codex | `~/.codex/config.toml` | `[mcp_servers.X]` + `url = "..."` (no `type` key) |
| OpenCode | `~/.config/opencode/opencode.json` | `{"type":"remote","url":"..."}` ← **remote, not http** |
| Cursor | `.cursor/mcp.json` | `{"type":"http","url":"..."}` |
| VS Code | `.vscode/mcp.json` | top-level `servers` (not `mcpServers`) |
| Gemini CLI | `~/.gemini/settings.json` | `{"httpUrl":"..."}` |

Prefer the client's own CLI when it has one (`claude mcp add --scope user --transport http <name> <url>`) — it writes the right shape.

For OMP, add the schema line for editor validation:
```json
{"$schema":"https://raw.githubusercontent.com/can1357/oh-my-pi/main/packages/coding-agent/src/config/mcp-schema.json","mcpServers":{}}
```

TOML edit: append a new `[mcp_servers.<name>]` block at EOF; validate with `python3 -c "import tomllib; print(tomllib.load(open(p,'rb'))['mcp_servers'])"`.

## Step 3 — Verify mounting, not config (the part that matters)

Config presence ≠ mounted tool. Three checks:

1. **OMP**: `omp -p --auto-approve "Call the <name> MCP tool '<known-tool>' … and report only the result. If no <name> tool exists, say NO_<NAME>_MCP."` — must return real data, not the sentinel.
2. **In-session**: `eval` (python) `print([n for n in dir(tool) if '<name>' in n.lower()])` → `[]` means not mounted; MCP config loads at session start, so instruct the user to `/mcp reload`.
3. **Claude**: `claude mcp list | grep -i <name>` → `✔ Connected`.

## Traps

- **Never assume cross-tool translation works.** `omp://mcp-config.md` lists many client files as discovery sources; an A/B test disproved discovery from `~/.config/opencode/opencode.json`. **A/B test**: move the OMP-native file aside (`mv ~/.omp/agent/mcp.json /tmp/bak`), re-run the probe, then restore. This distinguishes "my file works" from "something else already provided it".
- A server may exist in one client but be suppressed per project (`disabledMcpServers` in `~/.claude.json` `projects.<path>`). Report this; don't silently "fix" it.
- Some endpoints return metadata stubs alongside real entries (e.g. Nuxt's 309 entries → 286 real pages after dropping `.navigation`). Count and say which is which.
- Missing `"type":"http"` on a remote server makes OMP treat it as stdio and error `requires "command" field`.
