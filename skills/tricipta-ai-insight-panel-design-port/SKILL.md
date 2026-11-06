---
name: tricipta-ai-insight-panel-design-port
description: "Port the toolkit \"AI Diagnostic Insight\" panel design into tricipta-edafy-asset-web (issue #871 family): design source spans, prototype-to-Nuxt traps, docked-shell contract, and the spec-first gate this workspace demands."
---

# Port the toolkit AI Insight panel design into asset-web

Use when continuing issue #871 (AI Insight panel design port / unified insight endpoint) or porting any docked "AI Diagnostic Insight" panel from the toolkit prototype into `tricipta-edafy-asset-web` (Bubble Maps, Diagnostic Plots, and later VFM/VPS/DDM).

## Step 0 — spec-first gate (this workspace)

The user reverts ad-hoc implementation and asks for "a spec and implementation". Before code:

1. `~/Sites/afed/tricipta/docs/superpowers/specs/YYYY-MM-DD-<slug>-design.md` — Goal, Current state grounded with `file:line`, Scope in/out, Decisions table, Target design (frozen interfaces), Risks, Acceptance criteria, Open items.
2. `~/Sites/afed/tricipta/docs/superpowers/plans/YYYY-MM-DD-<slug>.md` — Goal, Global Constraints (repo + branch + head, "skip project-wide suites", commit message per task), then Tasks with `- [ ]` steps, files, interfaces, verification, commit.
3. Materialize the plan with Archify (`~/.omp/tools/archify`): `.workflow.json` + `.workflow.html` + visual-check; `archify validate workflow <file> --quality showcase` must pass. Read the installed `archify-workflow-lane-budget` skill before choosing lane count.

#871 artifacts: `specs/2026-09-10-ai-insight-ui-endpoint-design.md`, `plans/2026-09-10-ai-insight-ui-endpoint.md` (status: draft, held for user review — check §8 open items before implementing).

## Design source (verified)

- Prototype = `~/Sites/afed/tricipta-uiux-asset/tricipta-platform` (React 19 + Vite, the app behind https://ui.toolkit.asset.trinovate.io). `library-uikit/` is the component library and has **no** insight surfaces — do not look there.
- Panel markup: `src/pages/BubbleMapDashboard.jsx:345-658` (state/handlers :695-829, mount :1395-1416); `src/pages/DiagnosticPlotsDashboard.jsx:325-583` (state :614-893, mount :3048-3067).
- Category arrays: `BubbleMapDashboard.jsx:135-178`; `DiagnosticPlotsDashboard.jsx:218-292`.
- CSS: `src/index.css` `.custom-scrollbar:37-58`, `.animate-fade-in:190-192`, `.animate-purple-gradient:761-794`.
- Prototype behaviour to know: chat is local-only (canned keyword reply after a 600 ms `setTimeout`, no fetch); typewriter 16 ms/char (bubble) / 18 ms (diagnostic); KPI/summary numbers are hardcoded mock — never port them.

## Port traps

- Prototype utilities that do **not** exist in Tailwind: `shadow-3xs`, `py-0.2` (silent no-ops). Remap to `shadow-2xs` / `py-0.5`. Valid in the repo's Tailwind v4.1.16: `shadow-2xs`, `rounded-xs`, `rounded-br-xs`, `rounded-bl-xs`, arbitrary values like `w-[340px] text-[11.5px]`.
- Already defined in `app/assets/css/common-platform.css` — do not re-add: `.custom-scrollbar` (L17-40), `.animate-fade-in` (L164-176), `.animate-purple-gradient` (L613-621), `.drawer-animate-appear` (L542).
- `app/components/common-platform/InsightLogoIcon.vue` already exists (props `size`, `className`, `fill`, default teal).
- Panel mount points today: `bubble-maps-dashboard.vue:2386-2398` (absolute popover, toggle `aiInsightOpen` L182, button L2367-2384) and `diagnostic-plots-dashboard.vue:1072-1077` (`WellAiInsightDrawer`, owns `internalOpen`, mounted in the toolbar).
- `issue://871` does not resolve from the `tricipta/` workspace root — use `issue://AFED-digital/tricipta-edafy-asset-web/871`.

## Frozen interfaces (as drafted in the #871 spec — re-read the spec before use)

- Docked shell `AiInsightPanelShell.vue`: props `subtitle`, `statusLabel`, `chatTitle`, `thinkingLabel`, `chatPlaceholder`, `chatChips`, `messages`, `thinking`, `chatInput`, `activeTab: "overview" | "chat"`; emits `update:activeTab`, `update:chatInput`, `send(text)`; default slot = dashboard-owned Overview body. Sizing `w-[340px] xl:w-[380px] 2xl:w-[420px] shrink-0 h-full`.
- Shared composables: `useTypewriter(source, speedMs=16)` → `{ text, isComplete }`; `useInsightChatMock({ greeting, rules, fallback, replyDelayMs })` → `{ messages, thinking, input, send, reset }`.
- Endpoints (if approved): `POST /api/v1/insights/{bubble-maps|diagnostic-plots|vfm-table|vps-well-table|vps-platform|vps-field}` replacing the six `/insight` routes; hard rename, swagger tag `insights`; web call sites `useBubbleMapInsight.ts:100`, `useWellAiInsight.ts:102`, `useVfmTableAiInsight.ts:45`, `useVpsTableAiInsight.ts:61`, `useVpsAiInsight.ts:82-83`. Swagger regen: `swag init -g cmd/server/main.go -o docs` (never add `--parseDependency`).

## Verification

Browser pass on both dashboards at 1440 and 2048: panel docks without overlapping map/chart; tabs switch; typewriter runs once per result; chat chips/composer append bubbles and reply ~600 ms later with **no network request**; every status branch (`inert|generating|insufficient|unavailable|error`) renders; retry re-fires; category Apply switches the page's active capability.
