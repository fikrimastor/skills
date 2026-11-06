---
name: tricipta-house-plan-implemented-audit
description: "Audit whether a Tricipta house-format implementation plan (docs/superpowers/plans/*.md) is actually implemented across its repos — run each task's own Verify gate, then per-acceptance-criterion test hunt, instead of trusting commit messages or the docs' self-declared status. Use when asked 'is this plan already implemented?' or 'does the plan exist as code?'."
---

# Tricipta house-plan implemented? audit

Answer "is the plan already implemented?" with executable evidence per task and per acceptance criterion. Commit messages, PR titles, and the knowledge base's self-declared status are NOT evidence.

## Procedure

### 1. Locate the plan and its sibling spec
```bash
glob docs/superpowers/plans/*<slug>*.md   # plan
ls docs/superpowers/specs/*<slug>*.md     # design spec
```
Read the plan's **Global Constraints** block first: it names the branch and HEAD per repo, the unrelated uncommitted files that must stay unstaged, and the frozen identifiers. Those are your comparison baseline.

### 2. Map task -> repo -> gate
Each task has `Repo:`, `Files:`, `Steps`, `Verify`, `Commit:`. Extract the Verify block verbatim per repo; that is the gate you will run.

### 3. Confirm the branches and commits (cheap, do this before running anything)
```bash
for r in <repoA> <repoB> <repoC>; do
  echo "=== $r ==="; git -C $r branch --show-current
  git -C $r log --oneline @{u}..HEAD   # unpushed set
done
```
Expect "nothing implemented" to look like only the spec/plan docs commits, and "implemented" to look like one commit per plan task with the plan's exact `Commit:` message.

### 4. Run every Verify gate verbatim
Use the plan's own commands, in order, one repo at a time:
- Python: `./venv/bin/python -m pytest tests/... -q` -> report the tail line with counts.
- Go: `go build ./... && go test ./internal/application/<svc>/...` -> use `-count=1` to defeat the test cache; without it you may report `(cached)` as if it were a fresh run.
- Nuxt/Vue: `npx vitest run <path>` -> report `Tests N passed` AND `VITEST_EXIT=$?`.

### 5. Per-acceptance-criterion test hunt
For each numbered AC, grep for the assertion that would fail without the change. Name `test_function` + `file:line` + PASS, or state MISSING plainly. Do not accept a nearby test as covering a different AC.

Grep for the *shape* of the required test, not its topic word:
- "drives two contradictory LLM responses" -> grep `responses:` / response-queue usage, not `Determinism`.
- "asserts byte-identical features" -> grep the `not in` / `== expected_features` assertion, not the word `invariance`.
- "asserts a numeric label rendered" -> grep the literal expected string, not the label constant.

### 6. Verify cited file:line claims against source
Open the cited range. A plan is often written before the code, so a claim can be true in substance but wrong in evidence. Check especially:
- field optionality (`omitempty`, pointer vs value, `Literal[...] | None`)
- whether a called-once helper is called unconditionally or behind a branch
- whether a struct carries every field the plan's output format needs

### 7. Report as a delta table
One row per plan task: `Repo | branch | commits | gate result | IMPLEMENTED/PARTIAL/MISSING`. Then a separate numbered list of gaps, each with file:line. Include doc drift (files still marked "Target:" after implementation, stale plan wording) as gaps — the plan frequently mandates a doc edit that never landed.

## Traps

- **Truncated plan artifacts.** Reading a long plan via the `read` tool truncates lines (`[Some lines truncated to 768 chars]`) and can elide the file paths inside a bullet. The plan's stated test path may therefore look wrong. Confirm every path with `glob` before reporting the plan as incorrect.
- **Go test cache.** `go test` prints `ok ... (cached)`; always re-run with `-count=1` before claiming green.
- **Unit test satisfying a service-level AC.** A builder unit test that calls the function twice proves determinism of the builder, not of the service. The plan usually asks for the service-level version; report the missing one.
- **Doc self-declaration.** A knowledge base or spec that says `Status: IMPLEMENTED` is a claim, not evidence. Check its own glossary/changelog for lingering "Target:" wording — that mismatch is itself a finding.
- **Plan checkboxes.** `grep -c '^- \[ \]'` on an executed plan often still returns the full count (nobody ticks them). Do not use checkbox state as the implementation signal.
- **Stale-looking shell output.** If repeated bash calls appear to return identical blocks, re-run the single decisive command with an explicit exit echo and a second distinct command in the same call. Do not mark a gate green on output you cannot attribute to a specific invocation.
- **Frozen-contract deviation.** Where the plan freezes an exact output format and the code renders a subset, count it as an implementation gap, not plan-text drift — and check whether the plan's format is even satisfiable from the DTO the code receives.

## Acceptance
Report per-task verdicts plus a numbered gap list, each gap carrying file:line and the gate or AC it violates. Ask the user before changing code on a branch whose contract the plan froze.
