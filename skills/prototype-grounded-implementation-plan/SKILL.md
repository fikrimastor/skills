---
name: prototype-grounded-implementation-plan
description: "Write an implementation plan whose code blocks are compiled and tested first in a throwaway git worktree of origin/main, so the plan ships verified code rather than guesses. Use when writing a writing-plans or house-format plan for a Go or other compiled repo, especially when the change introduces new library APIs or pinned dependency versions, or when the main checkout is dirty or on another branch."
---

# Prototype-grounded implementation plan

## Why
Plans written from reading code alone embed wrong API calls, wrong counts and vacuous tests. On asset-api #1024 this happened 8 times: a tracing provider that produced zero spans, a wrong client count, an env default the Collector rejected, a privacy test that passed on bad code, and others. Every one was caught only by running code.

## Procedure
1. **Isolate.** Never touch the user's checkout. Fetch, then `git -C <repo> worktree add --detach /tmp/<slug> origin/main`. Diff the spec's base SHA against the new head for the files the spec cites; re-read any that changed.
2. **Check pins in a scratch module.** `go mod init scratch && go get <all pins>`. Confirm core versions are not bumped (`grep` go.mod). Then read the exact APIs you will call in `$(go env GOMODCACHE)`: option funcs, interfaces, and which options disable which behaviour.
3. **Prototype in the worktree, test-first.** Write each test, see it fail for the expected reason, then implement.
   - `go mod tidy` prunes modules not yet imported. Re-`go get` when a later file imports them. In the plan, put each `go get` in the task that first imports the module.
   - Extra module paths may be needed for `go.sum`, e.g. `gorm.io/plugin/opentelemetry/tracing@v`.
4. **Prove each test is not vacuous.**
   - Mutate the implementation back to the old or bad behaviour and confirm the test goes red. Then restore it.
   - Common vacuity: the code logs through a global (`zap.L()`) that the test never replaces. Fix with `t.Cleanup(zap.ReplaceGlobals(obs))`.
   - Assert "empty before" for flush/shutdown tests.
5. **Negative-control config validators.** Feed a deliberately broken config and confirm a non-zero exit before trusting a "pass". An unset env var often validates as empty or 0.
6. **Run the repo's existing suites that the change touches**, including Docker ones. Attribute failures: check whether it is pre-existing on base (e.g. a wrong env var name in a TestMain) or caused by you (e.g. a newly visible route in a route-coverage test).
7. **Write the plan from the files, not from memory.** In a py eval, read the prototype files and embed them verbatim with `code(lang, Path(...).read_text())`. After fixing a test, splice the new text into the plan by replacing the old block. Scan for `TBD|TODO|<placeholder>`.
8. **Record deviations.** Add a "Spec corrections found during grounding" table (C1..Cn: spec text / evidence / plan). Add a task that updates the spec.
9. **Clean up.** `git worktree remove --force /tmp/<slug> && git worktree prune`, delete scratch dirs, and confirm the user's checkout status is unchanged.
10. Store non-obvious library gotchas with `learn`.

## Plan conventions worth keeping
- Execution-waves block with at most 2 parallel executors. Only one task in a wave edits `go.mod`.
- Global Constraints: worktree command, path-limited commits, no gofmt sweeps, exact pins, frozen identifiers, merge gate.
- An AC trace table mapping each spec acceptance row to a named test.
