---
name: ompgui-project-guide
description: "Project map and operational reference for ompgui (Bun server + React SPA embedding the oh-my-pi coding-agent engine) at ~/Sites/personal/ai. Use when starting/restarting/debugging ompgui, adding features to it, or writing/updating its docs."
---

# ompgui — Project Guide

Location: `/Users/fikrimastor/Sites/personal/ai` (package name `ompgui`, private).

## What it is

Bun server embeds the oh-my-pi coding-agent engine in-process (`@oh-my-pi/pi-coding-agent` SDK, `createAgentSession`) and serves a React SPA over one WebSocket (`/ws`), opencode-style UX. No build step — Bun bundles the TSX via HTML import.

Full current-state reference: **`README.md`** at project root (written 2026-08-18, covers features/architecture/protocol/routes). Keep it updated when adding features instead of re-deriving everything from scratch next time.

## Architecture

```
Browser SPA (React, web/app.tsx)
  │ WebSocket /ws, JSON frames — shared/protocol.ts is the frozen contract
  ▼
Bun server (server/index.ts, Bun.serve on 127.0.0.1:PORT)
  │ in-process SDK calls
  ▼
AgentSession → JSONL session files on disk (SDK-managed)
  ▲
  ├── server/session-agents.ts → parses JSONL for the Subagents panel
  └── server/trajectory.ts     → parses JSONL for the Trajectory tab
```

- `server/engine.ts` boots the SDK once at startup: `discoverAuthStorage()` (reads existing oh-my-pi credentials — this app has no login flow of its own), `ModelRegistry`, `refresh()`.
- One active `AgentSession` at a time; every connected browser tab mirrors/controls the same session (like multiple TUI attachments) — there is no per-tab/per-client session isolation.
- Engine settings persist via the SDK to `settings.getAgentDir()/config.yml` — this path is **dynamic** (resolved via `settings.getAgentDir()` at runtime). Never hardcode `~/.omp/config.yml` or `~/.config/oh-my-pi/...` — both were wrong guesses made by parallel research agents in one session; verify against `server/index.ts` (`buildSettingsSnapshot()`) if it matters.

## Run / restart

Managed as a hub process named `ompgui`. If `hub ps` shows a prior `ompgui` job as `exited`, restart it by name rather than guessing a fresh command:

```
hub start ompgui --application bun --args run,dev --cwd /Users/fikrimastor/Sites/personal/ai --ready '{"log":"running at","port":3117}'
```

- `bun run dev` = `bun --hot server/index.ts` (hot reload — use for active dev).
- `bun run start` = `bun server/index.ts` (no hot reload).
- Default port `3117` (`shared/protocol.ts#DEFAULT_PORT`), override with `PORT` env. `WORKSPACE` env sets the agent's working directory (defaults to `process.cwd()`), also seeds `Settings.init({cwd})` and the initial session.
- Ready signal: stdout log line `running at` (server also prints `WebSocket: ws://localhost:<PORT>/ws` and the workspace path).

## Key files

(Line counts as of 2026-08-18 — re-`wc -l` before quoting exact numbers in future docs, they drift.)

- `server/index.ts` (856 ln) — HTTP routes (`/`, `/ws`, `/session-log?path=`, static `/styles.css` `/app.tsx` `/web/*`) + WS message routing.
- `shared/protocol.ts` (200 ln) — **source of truth** for the WS contract (`ClientMsg`/`ServerMsg` + shared types: `SnapshotState`, `AgentInfo`, `TrajectoryEntry`, `SessionAgentInfo`, `SettingsSnapshot`, etc). Read this directly, don't trust paraphrase, before describing the protocol anywhere.
- `server/session-agents.ts` (482 ln) / `server/trajectory.ts` (235 ln) — stateless JSONL parsers, re-read from disk on every request; no server-side caching to worry about.
- `web/app.tsx` (1,637 ln) — single `App` component, ~22 `useState` hooks, no external state lib, purely WebSocket-driven; view switching (chat/trajectory) via a plain `view` state var, no router.
- `docs/superpowers/specs/2026-08-15-ompgui-design.md` — original v1 design spec. Implementation has since outgrown it (Subagents panel, Trajectory audit, Settings modal, Plan mode toggle were all added after v1, and multi-session concurrency was explicitly deferred there). Treat `README.md` as current-state truth, the spec as historical intent only.

## Lesson: cross-verify parallel research before writing docs

When dispatching parallel scouts to research server-side vs. frontend for documentation, they can disagree on derived facts they didn't directly quote (e.g. one scout claimed settings persist to `~/.omp/config.yml`, another claimed `~/.config/oh-my-pi/agent_dir/config.yml` — both were wrong; the actual value is the dynamic `settings.getAgentDir()/config.yml` computed at runtime). Any factual claim two sources disagree on, or that reads like a paraphrase rather than a quote, → read the actual source line yourself before committing it to a deliverable. Don't average or guess between conflicting subagent claims.
