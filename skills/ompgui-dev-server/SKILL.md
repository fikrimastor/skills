---
name: ompgui-dev-server
description: "Start, stop, restart, or check status of the ompgui (Oh My Pi web GUI) Bun dev server at ~/Sites/personal/ai. Use when asked to \"up\", start, stop, restart, or run the web app/GUI in this project."
---

## What it is

`ompgui` (project root `/Users/fikrimastor/Sites/personal/ai`) is a Bun fullstack app: a Bun server (`server/index.ts`) serving a React SPA (`web/app.tsx`) that acts as a web GUI for the Oh My Pi coding agent — settings, subagent view, trajectory audit, session management via `@oh-my-pi/pi-coding-agent`.

Layout:
- `server/` — `index.ts` (entry, `Bun.serve` + WebSocket), `session-agents.ts`, `trajectory.ts`, `engine.ts`, `sessions.ts`
- `web/` — `app.tsx` (React SPA), `styles.css`, `index.html`
- `shared/protocol.ts` — WS client/server message types, `DEFAULT_PORT = 3117`

## Starting it

This is a long-running dev server — MUST launch via `hub`, never bare `bash`/`&`.

```
hub start
  name: ompgui
  application: bun
  args: ["run", "dev"]
  cwd: /Users/fikrimastor/Sites/personal/ai
  ready: { log: "running at", port: 3117, timeout: 20 }
```

`bun run dev` = `bun --hot server/index.ts` (hot reload on edits to `server/`, `web/`, `shared/` — use this for active development).
`bun run start` = `bun server/index.ts` (no hot reload) — only if hot reload is undesired.

If a hub process named `ompgui` already exists (running or exited), reuse the same name — `hub start` on an exited/completed name restarts it; don't invent a new name.

## URLs

- App: http://localhost:3117
- WebSocket: ws://localhost:3117/ws

Port comes from `PORT` env var, default 3117 (`shared/protocol.ts` `DEFAULT_PORT`).

## Managing it

- Status: `hub ps` (lists all) or `hub describe ompgui`
- Logs: `hub logs ompgui`
- Stop: `hub stop ompgui`

## Trigger phrases

"up the web", "start the web app", "run ompgui", "start the GUI" (when cwd is or references this project) all mean: start this hub process.
