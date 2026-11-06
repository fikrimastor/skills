---
name: llm-json-contract-enforcement
description: "When an LLM returns structured JSON that fails contract validation: classify repairable vs rejecting violations, coerce/clamp what's deterministic, retry only transient failures"
---

# LLM JSON Output Contract: Repair vs Reject

Decision rule for enforcing an LLM output contract (structured JSON insight/report from a weak model, e.g. llama-8b via OpenAI-compat APIs). Classify each violation, then repair cheaply or reject — never retry blindly, never reject what a deterministic transform can fix.

## Classify the violation

| Violation | Action | Why |
|---|---|---|
| Prose/markdown around JSON | Salvage first `{...}` with stdlib decoder | JSON itself is valid |
| Corrupted JSON inside the object (stray chars, unbalanced) | Retry the LLM call **once** | Transient sampling noise; second sample usually parses |
| Over-length list (e.g. 7 findings, max 5) | Clamp to first N | Items are independent; a superset of the contract |
| Below-minimum optional list | Accept 0 — no minimum | Sparse evidence legitimately yields few/none; forcing minimum invites fabrication |
| Number where string expected (scalar type slip) | Custom UnmarshalJSON that coerces (json.Number to preserve source text) | Renders identically; type slip ≠ malformed |
| Wrong enum, missing required field, empty required string | Hard fail with actionable error | Deterministic violation; retry cannot fix |
| Wrong topic/salience (summary about the wrong block) | Fix the prompt: explicit primary-focus rule + annotate payload blocks | Model latches onto whichever block has most card-shaped numbers |

## Prompt-layer prevention

- Pin structured output at the provider when supported (`response_format: {"type":"json_object"}` — Databricks FM APIs and DeepSeek support it; the gate is **case-sensitive**: literal lowercase `json` must appear across ALL messages, not just the system prompt; Anthropic supports only `json_schema`, not `json_object`).
- State which payload block owns which output section (cards/title/summary ← primary analysis only; other blocks corroborate inside narrative).
- For optional lists, say "include only with concrete evidence; empty array is correct" — otherwise models pad to look compliant.

## Retry scoping

Retry ONLY transient failures (transport, 5xx, malformed JSON). Never retry deterministic failures (schema violations, validation errors) — a second sample reproduces them. Infrastructure layer retries transport; application layer retries content quality; keep the two separate.

## Anti-patterns

- Failing the whole response on one repairable scalar (e.g. `"value": 1` vs `"1"`).
- Minimum-count gates on optional evidence lists.
- Retrying schema violations (wastes latency, deterministic).
- Flattening provider errors into generic "service unavailable" (debugging nightmare — surface provider message in logs AND consider exposing it in the response).
- Forgetting that containerized apps need image rebuild, not just restart, when there's no source bind-mount.

## Verification pattern

Table-test each repair path separately: coercion (number→string), clamping (7→5), acceptance (0 and 2 items), and rejection (wrong enum, empty required field, malformed item among valid). Include the exact production payload class as a named regression test.
