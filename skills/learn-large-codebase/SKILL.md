---
name: learn-large-codebase
description: "Use when asked to learn/prime/read an entire codebase (learn-codebase skill) but the repo is too large (hundreds of thousands of LOC, 500+ files) to read verbatim in one context window. Provides a parallel-subagent fan-out procedure for genuine full-coverage reading."
---

# Learning a codebase too large for one context window

The `learn-codebase` skill says to read every source file in full. On a small-to-medium repo, do that directly. On a large monorepo (multiple subsystems, 500K+ LOC, 1000+ files), reading everything into your own context is not feasible — instead achieve genuine full-coverage reading via a parallel scout fan-out, then synthesize.

## Procedure

1. **Ground yourself first.** Read root README, CLAUDE.md/AGENTS.md, and any curated architecture doc directly (not delegated) — these are cheap and give you a fast, accurate map to guide partitioning.

2. **Measure before partitioning.** Use `bash`/`find`/`du` to get file counts and KB sizes per subsystem/directory. Identify: what's actually source code (include) vs generated/vendor/binary/data-fixture (exclude — e.g. swagger.json, node_modules, compiled binaries, CSV/JSON data dumps, .nuxt/.output).

3. **Partition into ~20-32 balanced slices** (respect your platform's subagent concurrency cap). Group by:
   - Business/functional domain for modular backends (e.g. group 6-8 feature modules per slice by theme: identity, geo/basin, ingestion, AI, misc).
   - Directory for frontend component/composable/page trees; split large flat directories (100+ files) alphabetically in half/thirds, instructing each agent to self-determine its slice via `sort` so gaps/overlaps are impossible.
   - Root-level shared/core plumbing (config, middleware, models, shared utils) as its own slice(s).
   - One-off script/cmd directories can be split into thirds by directory-name sort, since individual scripts are usually small and repetitive.

4. **Dispatch ALL slices in a single `task()` batch** (not serially, not one-at-a-time) using a read-only scout agent type. Put shared conventions (module patterns, stack, terminology) once in the batch's shared `context`; give each task an exact, self-contained file/directory scope plus a **standardized output contract** so all reports merge cleanly:
   - Scope covered (file list, count, size) — for auditability.
   - Subsystem overview (2-4 sentences).
   - File-by-file notes (purpose, key symbols/endpoints/routes, business logic).
   - Cross-cutting patterns (named once, not per-file).
   - Integration points (what calls in/out).
   - Gotchas / tech debt / dead code / destructive operations.
   - 2-5 most-important files to point a new engineer to.
   Explicitly instruct: read every file in FULL (page with offset/limit for large files), do not sample/skim, do not run builds/tests, read-only.

5. **Drain with repeated `wait` calls** until all slices report `completed`.

6. **Fetch each agent's FULL output** via its artifact URI (previews are truncated — always pull the complete JSON/report before synthesizing).

7. **Synthesize and persist.** Compile the reports into your working understanding for this session. Use a memory/retain tool to save the ~10-15 highest-value *durable* facts for future sessions: architectural decisions, coexisting/competing implementations (e.g. legacy vs new schema both still live), confirmed dead code, destructive operations, migration status (which version is active vs deprecated), and non-obvious traps. Don't retain generic file listings — retain the insights a future session would otherwise have to re-derive.

8. **Report to the user honestly.** State the actual scale (LOC/file counts), the partitioning strategy used, and that full coverage was achieved via parallel delegation rather than literal single-context reading — this is the responsible, transparent way to satisfy "read every file" at a scale where that's only possible in aggregate.
