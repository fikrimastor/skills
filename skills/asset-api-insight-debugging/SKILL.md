---
name: asset-api-insight-debugging
description: "Diagnose and fix asset-api /diagnostic-plots/insight failures: Databricks json gates, stale Docker images, scope mismatches, schema clamping, and context-block prompt hijacks"
---

# Asset-API Insight Endpoint Debug Playbook

Failure modes and fixes for `POST /api/v1/diagnostic-plots/insight` (asset-api → Databricks llama-8b), verified 2026-09-01 on `feature/682-insight-api-v2`.

## Read the failure shape from the log first

| Log signature | Meaning | Fix locus |
|---|---|---|
| 502 `malformed LLM JSON output: ...` | Model output corrupt/prose-wrapped | plot-api/asset-api parse layer |
| 503 `LLM service unavailable` | Provider rejected or unreachable — read the wrapped provider error in the same log line | llmsvc → provider |
| 400 `messages must contain the word "json"` | Databricks json_object gate | prompts |
| 422 `unsupported_scope` | Curated scope mismatch | curated.go + analysis_profiles.yaml |
| 502 `invalid findings/cards count` or `invalid status enum` | Schema violation (parseValidated) | never retried; clamp if safely repairable |
| Summary topic ≠ selected chart | Context-block attention hijack | prompt PRIMARY FOCUS rule |

## Known traps

1. **Docker has no source bind-mount.** `docker compose build api && docker compose up -d api` is required after commits. A container restart is NOT enough; stale-image mismatch masquerades as "the fix didn't work". Log caller paths under `/app/` confirm containerized execution.
2. **Databricks `json_object` gate is case-sensitive**: lowercase `json` must appear in the *combined messages*. Uppercase `JSON` alone fails. The user prompt is the classic miss — marshalled evidence payload does not contain the word. Both system AND user prompts must carry it.
3. **Two context layers can disagree**: asset-api `internal/domain/chartcatalogue/curated.go` (Scope, ContextIDs) vs plot-api `plotting_core/domain/analysis_profiles.yaml` (scope, entity_strategy). When flipping one, flip the other, and update the hard-coded counts in `curated_test.go` (41/5-style roster assertions) plus the field-scope list in `capability_resolver_test.go`.
4. **Retry policy is asymmetric**: exactly one retry, only on `ParseLLMNarrativeSyntax` failure (corrupt JSON). `parseValidated` schema violations are deterministic — never retried. Exception: findings count >5 is clamped to first 5 (independent evidence, superset of contract); <3 stays 502.
5. **`fakeLLMClient` in insight_service_test.go supports `responses` sequences** for ordered bad-then-good retry tests; `resp` alone returns the same payload every call (breaks retry tests).
6. **response_format plumbing**: `AskCommand.ResponseFormat *string` → `GenerateRequest` → `chatCompletionRequest.ResponseFormat map[string]string` with `omitempty` (nil = field absent). `"json_object"` works for Databricks llama + DeepSeek (DeepSeek also needs "json" in prompt); Anthropic supports only `json_schema` under a different wire shape — the Anthropic client silently drops the field, which is safe but unguaranteed.

## Verification sequence

```bash
go build ./...
go test ./internal/application/diagnosticplotinsightssvc/ ./internal/application/llmsvc/ ./internal/infrastructure/llm/ ./internal/domain/chartcatalogue/
# Known pre-existing failures (clean tree, unrelated): TestConfig_DiagnosticPlotAPI_Defaults, TestVPSMigrations_ApplyAndRepositoryInsertFind
```

Then rebuild the container before judging any fix live.
