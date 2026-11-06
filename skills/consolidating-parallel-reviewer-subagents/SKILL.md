---
name: consolidating-parallel-reviewer-subagents
description: "Use after dispatching multiple parallel task agent:\"reviewer\" subagents to review a PR/diff and synthesizing their findings into one report — covers recovering from a crashed subagent, reconciling contradictory or overlapping claims between reviewers against actual source, and downgrading findings contradicted by local code comments before presenting a final verdict."
---

## When to use

After fanning a PR/diff review out to multiple parallel `task` subagents with `agent: "reviewer"` (one per file-group), before writing the consolidated report back to the user.

## Before dispatching

- Verify the diff yourself first: confirm current branch, and that `git diff <base>...<branch> --stat` reproduces the exact file list/line counts the request claims. Takes one bash call, prevents sending reviewers to review the wrong range.
- Spot-check trivial files (`.gitignore`, one-line config adds) directly yourself in the same pass — cheap, and stops a reviewer agent from burning many minutes researching a 4-line diff's provenance.
- Group files by locality/module; pair each new/changed test file with its implementation file in the same reviewer's assignment.
- Put every cross-file integration contract (shared types, prop contracts consumed by another group's component, known prior bugs from project history/memory relevant to specific files) into the shared `context` field once. Reviewers do not share conversation history with each other or with you — anything they need to judge correctness must be explicit, since context is the only place instructions apply to the whole batch without duplication.
- Dispatch all reviewers in one `tasks[]` array — never serialize independent file groups.

## While waiting

- `hub wait` repeatedly; results settle asynchronously and out of order. The largest diff is not guaranteed to finish last, but often does — don't block downstream work on submission order.

## Recovering a crashed reviewer

- A reviewer can error (`status: failed (exit N)`) after doing real, valid tool-call work (git diffs run, files read, web research done) — the crash is often in generating the final structured answer, not in the investigation itself.
- Read `history://<agent-id>` (or a wide line range if the default render truncates) to see exactly what it already verified before deciding next step.
- For a trivial slice (small file count, small diff), it is faster and just as grounded to finish the remaining 1-2 open questions yourself with direct tool calls than to re-spawn a whole subagent.
- For a substantial slice, re-spawn with the same assignment rather than guessing at what the crashed agent would have concluded.

## Reconciling overlapping/contradictory claims

- Each reviewer only reads its assigned files in depth; it may read other files read-only for context but doesn't verify them as rigorously.
- Watch for two reviewers making claims that both hinge on the same code path but were derived from different files (e.g., a composable reviewer claims a page renders an error at a specific line; the page's reviewer, working from the page's own diff, found that exact block was commented out by the same patch).
- Before presenting either claim as settled fact in the consolidated report, directly `read` the disputed line range yourself. This is cheap (one or two tool calls) and prevents shipping a report with an internal contradiction, or worse, presenting a now-dead-code path as a live user-facing bug.
- Once reconciled, state the corrected, unified picture explicitly — don't just pick one reviewer's version silently.

## Downgrading disputed findings

- A reviewer's "bug" finding can be wrong not because the reviewer erred, but because the reviewer's assigned scope didn't include the comment/doc that explains the behavior is intentional.
- Reviewers self-report a confidence score per finding — a noticeably low score on one finding (e.g., 0.45 versus 0.7-0.9 on everything else in the same report) is a strong signal to verify that specific one yourself before repeating it as fact.
- To verify: read the exact function/lines cited, and check immediately-surrounding comments for stated rationale. If a comment explicitly documents the flagged behavior as a deliberate design choice, do not present the finding as a confirmed bug.
- Reframe as "disputed — needs author clarification," cite both the reviewer's reasoning and the contradicting evidence, and let the author/user make the final call rather than silently dropping or silently accepting the finding.

## Final report shape

Separate findings into: blocking (must-fix, user-visible or data-correctness), should-fix (real bugs, non-blocking), disputed (needs clarification, evidence on both sides), nits (optional polish). Always include a "verified clean" section — call out specific previously-known concerns (from project history/memory) that this round of review actually confirmed as fixed, not just a list of new problems. This is itself evidence, not a courtesy.
