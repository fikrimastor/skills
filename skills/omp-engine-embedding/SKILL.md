---
name: omp-engine-embedding
description: "Embed the oh-my-pi engine in a custom host app (web GUI, server, orchestrator) via the in-process SDK or RPC mode — boot sequence, event-stream semantics, verified type gotchas, and how to verify the integration"
---

# Embedding the oh-my-pi engine

Use when building any host on the omp engine: a GUI, a web server, an orchestrator, a Tauri/desktop wrapper. Reference implementation: `~/Sites/personal/ai` (ompgui — Bun server + React SPA over WebSocket).

## Choose the surface

- **SDK in-process** (`@oh-my-pi/pi-coding-agent`, Bun ≥1.3.14): direct `createAgentSession()`, richest access (SessionManager, ModelRegistry, AuthStorage). Use when the host is Bun.
- **RPC subprocess** (`omp --mode rpc`, JSONL over stdio): cross-language or process isolation. Docs: `omp://rpc.md`.

Always read `omp://sdk.md` and ground every call against the installed `.d.ts` under `node_modules/@oh-my-pi/pi-coding-agent/dist/types/` — never invent APIs.

## SDK boot sequence

```ts
import { createAgentSession, discoverAuthStorage, ModelRegistry, SessionManager } from "@oh-my-pi/pi-coding-agent";
const authStorage = await discoverAuthStorage();
const modelRegistry = new ModelRegistry(authStorage);
await modelRegistry.refresh();
const { session } = await createAgentSession({ authStorage, modelRegistry, cwd, sessionManager: SessionManager.create(cwd) });
```

- `modelRegistry.getAvailable()` → model list for pickers.
- `SessionManager.list(cwd)` / `.open(path)` (both async) for session sidebar/switching.
- Postinstalls are blocked by default: `bun pm trust --all` after `bun add` (onnxruntime-node, protobufjs).

## Event-stream semantics (verified)

- Forward `session.subscribe(ev => ...)` events verbatim; clients ignore unknown types defensively.
- Streaming ends ONLY on `agent_end` with `isTerminal !== false`. Retries/fallbacks emit multiple `agent_start` per prompt with one terminal `agent_end`.
- Assistant content blocks are `{ type: "toolCall", id, name, arguments }` — NOT `toolUse`. Tool results are role `toolResult` messages `{ toolCallId, toolName, content: [...] }`.
- `tool_execution_update.partialResult` and `tool_execution_end.result` are AgentToolResult-shaped objects (extract text from `.content` blocks); `partialResult` is a full replacement, NOT a delta — replace output, never append.
- `message_update` carries both `assistantMessageEvent` deltas and a full partial `message`; replacing the in-flight message with `event.message` is far more robust than delta accumulation. Guard replacement by role match.
- The engine echoes every accepted user message via `message_start`. For optimistic UI: tag local placeholders (`__optimistic`) and drop ALL of them when any authoritative message or an error arrives. Reference/text matching against echoes is fragile (content shapes differ; refs get cleared by racing snapshot paths).
- Sessions may contain role `custom` messages (injected system reminders) — hide them. Aborted turns leave empty assistant messages — filter empty-block messages before rendering.

## Type gotchas

- `parseConfiguredThinkingLevel` imports from `@oh-my-pi/pi-coding-agent/thinking` (wildcard subpath export), not the package root. Use it to validate thinking levels; the protocol-level union includes `minimal` which raw `ConfiguredThinkingLevel` typings appear to reject (typings-only mismatch, runtime accepts).
- `session.setModel(model)` is async — await it before broadcasting state, or you snapshot the old model and risk an unhandled rejection.
- `session.beginDispose()` then `await session.dispose()` before creating a replacement session. Disposing mid-stream aborts the active turn — guard session switches while streaming.
- One active session per process by default; multiple concurrent top-level sessions need a private `AgentRegistry` each.

## Verification recipe

1. **Frame-level ground truth first**: a scripted WS/stdio client (`bun -e`) that counts and sequences event frames per prompt. This separates engine/server bugs from rendering bugs in one shot — screenshots cannot.
2. Check raw session transcripts (`~/.omp/agent/sessions/<cwd-slug>/*.jsonl`, jq per-message roles) when UI state and events disagree.
3. Browser-drive the UI for the final pass. React controlled inputs: use the native value setter + `dispatchEvent(new Event("input", { bubbles: true }))`, then a bubbling `keydown` — `tab.fill` desyncs React state silently.
4. Beware concurrent live users on a shared session server: switches/aborts from other tabs pollute test runs.
