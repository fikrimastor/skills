---
name: tricipta-repo-graft-graphify-setup
description: "Use to add graft + graphify to a Tricipta repo that lacks graft/ or graphify-out/, matching the plot-api local-only pattern."
---

# Graft + graphify setup for a Tricipta repo

Reference repo: `~/Sites/afed/tricipta-diagnostic-plot-api`. All AI tooling stays local; only `.gitignore` gets committed.

## Steps (repo root)
1. Create the branch the user names, e.g. `git checkout -b <name>`. Never push unless the user asks.
2. `graft init --dry-run --no-global --agents agents claude`. `--no-global` is required: without it, graft writes `~/.codex/config.toml` and `~/.codex/hooks*`, which affects every repo on the machine.
3. `graft init --no-global --agents agents claude`. This builds `graft/` and writes `.claude/{settings.json,helpers/,skills/graft/}`, `.mcp.json`, `opencode.json`, an `AGENTS.md` graft section, `.ignore`, and a `/graft/` line in `.gitignore`.
4. `PYTHONHASHSEED=0 graphify update .` builds `graphify-out/`. If the repo has `.sql` files they are skipped unless `graphifyy[sql]` is installed.
5. `graphify hook install` installs the post-commit and post-checkout hooks. It also appends `graphify-out/graph.json merge=graphify` to `.gitattributes`. Revert that with `git checkout -- .gitattributes`, because graphify-out is gitignored.
6. `printf '@AGENTS.md' > CLAUDE.md`.
7. In `.gitignore`, replace graft's `/graft/` comment block with this `# AI` block:
```
# AI
CLAUDE.md
graphify-out/
AGENTS.md
.mcp.json
.ignore
opencode.json
graft/
.claude/helpers/
.claude/settings.json
.claude/skills/graft/
```
Omit `.claude/skills/graft/` if the repo already tracks `.claude/skills` content that should be shared.
8. Run `git status --short`. It should show only `.gitignore` plus any pre-existing user changes. Commit `.gitignore` alone; never stage the user's unrelated changes.
9. Smoke test: `graft ask "<domain question>"` and `graphify query "<concept>" --budget 200` should both return real symbols. Confirm nothing was pushed: `git rev-parse --abbrev-ref <branch>@{upstream}` should report no upstream.

## Status
- ddm-api: done on local branch `fm` (2026-11-06).
- asset-chatbot (`tricipta-production-ai-chat-bot`): not set up yet.
