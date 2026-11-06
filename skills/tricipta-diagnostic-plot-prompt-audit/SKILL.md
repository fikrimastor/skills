---
name: tricipta-diagnostic-plot-prompt-audit
description: "Verify the Tricipta diagnostic-plot agent prompts doc faithfully implements the source-of-truth criteria doc (validation gate, per-capability criteria, output format). Use after regenerating or editing either docs/diagnostic-plot-analysis-agent-prompts.md or docs/diagnostic-plot-capabilities-analysis-criteria.md, or when asked to review/verify prompt faithfulness."
---

# Tricipta Diagnostic-Plot Prompt Audit

Cross-doc fidelity audit: does the runtime prompt fleet faithfully carry the source-of-truth criteria so the agent will (1) VALIDATE data, (2) ANALYSE per criteria, (3) emit the specified OUTPUT FORMAT?

## Files (in ~/Sites/afed/tricipta/docs/)
- CRITERIA (source of truth): `diagnostic-plot-capabilities-analysis-criteria.md`
- PROMPTS (runtime, regenerated from criteria): `diagnostic-plot-analysis-agent-prompts.md`
- Maintenance rule: edit criteria first, regenerate prompts, keep thresholds identical. Drift = bug. When the prompt drifts from criteria, fix the PROMPT to match criteria.

## Structure (find by section header, not fixed line numbers — they move)
CRITERIA: §0 global protocol (0.1 units, 0.2 data-quality gate, 0.3 time align, 0.4 thresholds, 0.5 triangulation, 0.6 honesty) · §1 Family1 (1.1-1.11) · §2 Family2 (2.1-2.14) · §3 Family3 (3.1-3.19) · §4 Family4 (4.1-4.2) · §5 assembly guide (5.2 per-cap output template, 5.3 family synthesis, 5.4 global summary, 5.5 hard rules).
PROMPTS: §1 SHARED_PREAMBLE (7 rules) · §2 Family1 · §3 Family2 · §4 Family3 · §5 Family4 · §6 Parent Synthesizer · §7 integration (JSON schema).
Total 46 caps: 11+14+19+2.

## Decomposition (parallel, cap=2)
Grep both docs for section headers first to get current line ranges. Then:
- Scout A (agent:scout, read-only): Families 1+2 cross-doc diff.
- Scout B (agent:scout, read-only): Families 3+4 cross-doc diff.
- Do the global slice inline while scouts run: SHARED_PREAMBLE vs §0.1-0.6; synthesizer §6 vs §5.4; each family OUTPUT TEMPLATE vs §5.2; family-synthesis lines vs §5.3; preamble rule 7 vs §5.5.
Each scout returns: per-family COVERAGE checklist (every cap present, same name/number/order) + DISCREPANCY table (capability | dimension binding|criteria|flags|output | criteria says | prompt says | severity) with file:line cites for HIGH/MED + per-family verdict. Report-only; no edits, no build/lint/test.

## Three audit dimensions per capability
1. COVERAGE: cap present in both, same name/number/order.
2. CRITERIA FIDELITY: DATA BINDING (column semantics, unit traps, derivations, step/duplicate traps) + ANALYSIS CRITERIA (numbered steps, formulas, thresholds) + FLAGS all carried. Flag dropped/added/altered items.
3. OUTPUT: family template expresses what the capability produces.

## Known trap checklist (verify each)
- Cross-reference tokens (§X.Y): easy to transpose. E.g. a confirmed bug was prompts §1.9 flag citing §2.11 (Active Injectors, a COUNT) where criteria says §2.8 (Cumulative Water Injection, a VOLUME). Grep both docs for §2.8 vs §2.11, §1.3, §3.17, §2.10 etc. and confirm each matches criteria.
- Unit factors: ×10^6 (MMbbl↔bbl), ×10^3 (Bscf↔MMscf), ×6.28981, GOR conversions — must be identical.
- CD/PD guards: 'CD>PD impossible', the dataset-wide 1.033 quirk in 7/1989, downtime proxy.
- The 6/1989→7/1989 systemic ramp-up trap (×23/×14 etc.) and duplicate-value/alias traps (H1≡H5, C3≡C5, F3≡F6) — do not sum, verify.
- Formulas verbatim: VRR (§2.12), II=q/(Pbh−Pres) (§2.13), PI=q/ΔP (§2.14), RF=100·oil_cumulative/OOIP + HCPVI (§3.19).
- Family 4 template carve-out is CORRECT: criteria §4.1/§4.2 define a ranking-specific agent-output (rank table/concentration/classifications/shortlist) that overrides generic §5.2. Do not flag as drift.
- §7 JSON schema completeness: it models only the generic 6-field template; it cannot represent Family 4 ranking output — flag if structured JSON is needed for ranking plots.
- Validation-gate ordering: criteria §0 says 'run these checks once per session, before evaluating any capability'; check the preamble frames rules 1-2 as a mandatory upfront gate, not inline caveats.
- Conditional gates: criteria marks facility-capacity / PI-ΔP checks 'only if supplied' — prompt must not present them as unconditional.

## Output of the audit
Verdict table on the 3 questions (validate / analysis-per-criteria / output-format) = PASS / PASS-WITH-GAPS / FAIL, then HIGH bug list (with the exact one-token fix + file:line), MED hardenings, LOW notes. Default REVIEW-ONLY — do not edit the docs unless the user explicitly asks; present fixes as recommendations.
