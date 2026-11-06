---
name: tricipta-spec-open-item-answer-propagation
description: "Propagate a reviewer's answers to a Tricipta spec's §8 open items (O1-O7) through the spec, the implementation plan, the archify workflow/threat artifacts, and the visualize dashboard — including a decision reversal (e.g. gate - delete a route), the true deletion blast radius, and the frozen-list collision it creates. Use when someone answers open items or reverses a decision in docs/superpowers/ specs/plans."
---

## When

A Tricipta house-format spec (`docs/superpowers/specs/<date>-<issue>-<slug>-design.md`) has §8 "Open items for review" answered, and/or a decision in §4 (D1..Dn) is reversed. The answer must reach **every** derived artifact or they silently disagree:

- spec `.md` and plan `.md`
- `plans/<...>.workflow.json` + `.workflow.html` (archify)
- `specs/<...>.architecture.json` + `.architecture.html` for a threat model
- `plans/<...>.visualize.html` (the /visualize dashboard)

## Sequence

### 1. Settle the answers first, in one `ask`
One `ask` call with every §8 item (O1..On as `questions[]`), O-blocking and O-urgent first, each with `options[]` and a `recommended` index. Record "no answer" as a real state — do not invent a default for a skipped item.

### 2. Verify every load-bearing claim in source BEFORE propagating
Never propagate on trust. If the answer says "X already handles it", prove X exists end to end (frontend call site -> route registration -> handler -> application service -> the branch that makes it work) and report the file:line chain. State the **condition**: paths often work only because the UI sends a particular flag (e.g. `sso_only: true`), while the other branch fails hard without a configured secret. Flattening that to "admins just use page Y" loses the real constraint.

### 3. Enumerate the true blast radius of a deletion/reversal
A decision like "delete the route instead of gating it" is far wider than the route line. Build an ordered checklist:

- route registration line (and whether the registering **function** survives because it also registers siblings)
- handler function **and its swagger annotation block** (`@Router`, `@Param`, `@Success/@Failure`)
- request-DTO / command types used only by it
- application-service method (check its callers exhaustively — `lsp references` or `grep -rn`, not ranked search)
- error sentinels and helpers whose only callers were inside the deleted code (check each; some look shared but are not)
- **checked-in generated artifacts** (e.g. `docs/docs.go`, `docs/swagger.json`, `docs/swagger.yaml`) — regenerate with the repo's documented command (`swag init -g cmd/server/main.go --output docs`), never hand-edit
- README / docs prose and route tables
- explicitly KEEP and name anything still referenced

### 4. Fix the frozen-list collision — mandatory
The plan's Global constraints `Do-not-touch` list usually names the very implementation the new decision deletes. Narrow it (e.g. `service.go`'s `Login` implementation and helpers it shares with the login path — not `Register/Login`), and mirror the boundary in the spec's §3.2 out-of-scope. Never leave "frozen" recorded in one place and "deleted" in another.

### 5. Update expectation values, not just prose
A gated route expects `403`; a deleted one expects `404`. Verify the actual status by smoke-probing the framework (Echo: `echo.New()` + `httptest` + a `POST` to the unregistered path) rather than assuming. Update the AC/verification text and the Task verify steps, and drop any "only run this if Task N shipped" conditionals that existed only because of the blocker.

### 6. Preserve history
Keep the original recommendation text beneath the answer, labelled `**Historical recommendation (superseded <date>):** ...`. Mirror the docs' existing `**Corrected <date>:**` style. Never silently rewrite a reviewed decision.

### 7. Update the derived artifacts
- **archify**: edit the JSON, then `validate <type> <json> --quality showcase --json` (target 9/9, 0 errors, 0 warnings), `deliver`, `visual-check`. Artifact hashes change — re-record them. Do NOT chase visual-check containment (the tool's own bundled examples fail it).
- **visualize dashboard**: OPEN rows that were answered become `checked disabled` rows prefixed `ANSWERED <date> —`; genuinely unanswered rows (including "no answer recorded") stay unchecked. Update the flow cards, in/out scope, risks (mark resolved/obsolete ones rather than deleting them), and the footer. Leave the template's CSS/JS untouched.

### 8. Verify by grep, then render
Grep every doc for the stale markers (`403`, `BLOCKED`, `gated on`, `rate-limit <route>`, the old task title). Expect the only survivors to be explicitly labelled superseded/historical. Then actually render the dashboard in a browser and assert the DOM: section count, step/risk counts, decided vs open row counts, and that the open set equals exactly the items still unanswered.

## Traps

- Ranking search (`graft`/ranked MCP search) is top-N, not exhaustive — use it to find candidates, then exhaustively `grep -rn` before claiming a symbol is dead.
- `stale403: true` in a DOM assertion is often the deliberate "expected 404, not the old 403" string. Check the hit before treating it as a defect.
- Answering an item does not close it if it needs an external owner (ops/infra rotation): mark it answered-but-deferred and keep it OPEN in the residual list.
- Filing the follow-up issue is part of the work when the answer is "open a follow-up" — cite the real issue number in the spec row instead of "recommend a follow-up".
