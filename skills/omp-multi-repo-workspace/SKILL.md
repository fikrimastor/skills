---
name: omp-multi-repo-workspace
description: "Use when a user wants to work across multiple repos/folders in one OMP (Oh My Pi) session, asks to mirror another project's folder into the current directory, or asks for a VS Code-style multi-root workspace in OMP."
---

## Problem

User wants another project's folder "available" in/alongside the current OMP session — often phrased as "mirror this folder here," "multi-repo workspace," or "like VS Code multi-root workspace." Default instinct (git clone / rsync / symlink) is usually the wrong tool.

## Fast path — don't grep `omp://` docs for "workspace"

Grepping the internal `omp://` doc corpus for "workspace"/"multi-root" mostly returns noise: Rust/Bazel cargo workspace crates, GitLab Duo workspace/namespace fields, LSP `workspace/didChangeConfiguration`, config-dir workspace inheritance. The actual answer is one command away:

```bash
omp --help | grep -i dir
```

## The answer: `--add-dir`

```
--add-dir=<value>   Add a workspace directory beyond the working directory (repeatable)
```

- Registers the path as `additionalDirectories` in the session header (see `omp://session.md`) — the native equivalent of VS Code's "Add Folder to Workspace."
- No copy, no symlink, no clone. Repeatable: `--add-dir pathA --add-dir pathB`.
- Combine with an existing session instead of losing it: `omp --continue --add-dir <path>` or `omp --resume <id> --add-dir <path>`.
- If the target is a git repo with subdirectories that are just plain folders (not submodules — verify with `git -C <subdir> remote -v` matching the parent's remote), one `--add-dir` at the repo root covers everything under it.

## Why cross-repo tool calls already "just work" even without it

OMP tools (`read`, `bash`, `edit`, `grep`, `glob`) accept any absolute path regardless of session cwd — they are not sandboxed to cwd. Default `tools.approvalMode` is `yolo`, which auto-approves read/write/exec everywhere with zero prompts. So an agent can already `bash -C /other/repo ...` or `read /other/repo/...` mid-session with no setup. `--add-dir` mainly buys: (1) explicit workspace-root registration/metadata for the session and TUI, and (2) removing per-call approval prompts in stricter modes (`always-ask`/`write`) where paths outside cwd would otherwise gate.

## Don't confuse with `omp worktree`

`omp worktree [list|clear]` manages **agent-created** git worktrees under `~/.omp/wt` — these come from the `isolated` flag on `agent()`/subagent `task` calls (per-subagent edit isolation), not from anything the user sets up manually. Irrelevant to "I want to work across two of my own repos."

## When a real isolated checkout is still needed

If the user wants an independent working tree (separate branch, safe to edit concurrently with whatever's checked out in the original clone) rather than just shared live access, that's still plain `git worktree add <path> [-b branch | --detach]` — orthogonal to `--add-dir`, and the two stack fine (point `--add-dir` at the worktree path instead of the original repo).

## Decision order

1. Does the user just need *this agent* to read/edit another repo's files in the current conversation? → Nothing to set up, already works (empirically verify with one cross-path `bash`/`read` call if in doubt).
2. Does the user want a persistent, named second workspace root for the session (VS Code multi-root feel)? → `omp --add-dir <path>` (add `--continue`/`--resume` to keep the current session).
3. Does the user need an independent, separately-editable/committable checkout? → `git worktree add <path> [-b branch|--detach]`, optionally also registered via `--add-dir`.
4. Only reach for `git clone` / `rsync` / symlink when the copy must be fully decoupled (different remote, safe to delete without touching the original repo's worktree metadata) — state the disk/network cost (full history + object store duplicate) before doing it.
