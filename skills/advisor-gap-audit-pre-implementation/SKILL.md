---
name: advisor-gap-audit-pre-implementation
description: Run an advisor gap audit on spec and plan before any non-trivial implementation kickoff
---

# Pre-implementation advisor gap audit

Triggered before any non-trivial spec/plan is handed off for implementation (multi-file change, new components, new props on shared components, test scaffolding, backend rule changes).

## Why

The advisor (`oracle` agent) gap-audits the spec + plan against the actual code. It catches:

- Claims in the spec that are factually wrong against the codebase (e.g. "the controller already passes X" — verify with a direct read of the Inertia::render call)
- Hidden interaction effects in shared components when a new prop is added (e.g. existing watchers that clobber the new default initialiser)
- Test logic errors that pass for the wrong reason or fail when they should pass (e.g. union-of-windows design where a per-row rejection test ends up matching the global window instead)
- Spec inconsistencies (casing drift, missing fields referenced by new components)

## How

After writing the spec + plan to disk, BEFORE invoking `executing-plans` or `subagent-driven-development`, spawn an `oracle` agent with:

1. Paths to both the spec and the plan (in `docs/superpowers/specs/` and `docs/superpowers/plans/`)
2. Short context (what was just designed, the related prior specs)
3. Specific high-risk areas to check — usually:
   - Any claim in the spec that says "the controller already passes X" or "the modal already supports X" — verify
   - Any plan task that modifies a shared component — check existing watchers / initialisers / lifecycle hooks
   - Any new test that depends on a specific data shape — mentally simulate the test data
   - Any path that has a case-fold duplicate on macOS (e.g. `Tradein` vs `TradeIn`)

4. "Do NOT edit any files. Return a structured gap report with severity (blocker / concern / nit), location, and recommended fix."

## Output contract

Advisor returns a structured list grouped by severity. Each item cites exact line numbers + code snippets. Apply all blockers + key concerns before implementation; nits are optional.

## When to skip

- Single-file change < ~30 lines
- Pure config tweak with no code change
- Spec is < 5 sections and only touches files the agent has already read in full in the same session

## After the audit

The user decides scope ("fix all" / "fix only blockers" / "show me the full report first"). Apply the agreed fixes in place, then proceed to implementation.
