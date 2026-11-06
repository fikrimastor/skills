---
name: deepseek-api-contract
description: "Use when wiring, debugging, or costing DeepSeek API calls (model ids, thinking mode, max_tokens, pricing, key verification) — training-data assumptions about this API are stale and cause 400s."
---

# DeepSeek API contract

Model names and parameters from training data are **wrong** for the current API. Fetch
`https://api-docs.deepseek.com/api/create-chat-completion` and `.../quick_start/pricing`
before writing or debugging a client, and verify a key with `GET {base}/models` before any
long run.

## Model ids

- `deepseek-flash` (DeepSeek-V4.1-Flash) — cheapest
- `deepseek-v4-pro` — stronger
- **Not valid:** `deepseek-chat`, `deepseek-reasoner`. Legacy names still accepted:
  `deepseek-v4-flash`, `deepseek-v4-flash-vision-exp`.
- Base URL `https://api.deepseek.com` (OpenAI-compatible `/chat/completions`; Anthropic
  format at `/anthropic`).

## Reasoning is a parameter, not a model

- `thinking: {type: "enabled"|"disabled"}` — **defaults to `enabled`**.
- `reasoning_effort: none|low|high|max` — default `high`; `none` disables thinking.
- For prose, documentation, extraction, or anything not needing deliberation, send
  `reasoning_effort: "none"` explicitly. Otherwise reasoning tokens bill as output **and**
  consume the output budget, and `content` can come back short or empty.
- `temperature` has no effect in thinking mode (it is not rejected); `top_p` is raised to
  ≥0.95 in thinking mode and fixed at 1.0 otherwise; `frequency_penalty` and
  `presence_penalty` are deprecated no-ops.

## Output budget

- `max_tokens` range 1–393216 for both models; 1M context.
- Unset default: 8K (non-thinking) / 64K (thinking) — set it explicitly per request when
  outputs vary in size, rather than pinning one ceiling that truncates long outputs.
- Always check `finish_reason`: `"length"` means truncated. Do not persist a truncated or
  empty response, and do not mark the work as done/completed — retry with a larger budget.

## Pricing (per 1M tokens; peak = exactly 2× off-peak)

Peak hours: 01:00–04:00 and 06:00–10:00 UTC, Mon–Fri.

| Model | In (cache miss) | In (cache hit) | Out |
|---|---|---|---|
| `deepseek-flash` | $0.15 / $0.30 | $0.003 / $0.006 | $0.60 / $1.20 |
| `deepseek-v4-pro` | $0.66 / $1.32 | $0.022 / $0.044 | $1.98 / $3.96 |

(cache-miss/hit shown off-peak / peak.) Cache-hit input is ~50× cheaper, so a shared prompt
preamble across batched requests is worth structuring for.

## Checklist before a batch run

1. `GET {base}/models` with the key — confirms the key AND the live model ids. A bad key
   yields an opaque `401 Authentication Fails`; a bad model id yields a 400.
2. Reject placeholder/truncated keys locally before the call (the docs' own `sk-...` sample
   is the usual accidental paste; real keys are ~35 chars).
3. Decide thinking mode deliberately (`reasoning_effort: "none"` for prose).
4. Size `max_tokens` per request; handle `finish_reason == "length"` as a failure, not a result.
5. Cost from tokens × the table above, and state whether the run is off-peak or peak.
