---
name: visualize-from-brainstorming
description: Generate a self-contained HTML dashboard from a spec or brainstorm via the visualize template
---

# Visualize from brainstorming — fill the template

Triggered when the user says "visualize" / "visualize it" / "use visualize it skill" during a brainstorm or right after writing a spec.

## Template location

`~/.claude/skills/visualize/examples/template.html` (also at `~/.agents/skills/visualize/examples/template.html`). Never write CSS or the layout by hand — copy the template and fill the slots.

## Where the output goes

`docs/<feature>/<title>.html` — kebab-case, matches the spec filename. Create the dir if missing. Example: a spec at `docs/superpowers/specs/2026-07-07-sama-erp-journey-design.md` produces `docs/superpowers/sama-erp-journey.html`.

## Copy + fill

1. `cp ~/.claude/skills/visualize/examples/template.html docs/<feature>/<title>.html`
2. Replace header slots: `{{TITLE}} {{FEATURE}} {{ONE_LINE_GOAL}} {{STATUS}} {{DATE}}` and the `<title>` tag.
3. Fill the 5 sections (delete any not discussed):
   - **Goal & Success Criteria** — `<p>` + `<ul>` of measurable bullets
   - **Scope** — `.grid2` with `.panel.in` / `.panel.out` columns
   - **Flow** — `.flow` container with `.step` children (one per step)
   - **Mockup** — `.mock` block with ASCII layout (or `<img>`)
   - **Risks** — `.risk.high` / `.med` / `.low` with `.tag` for sev/likely
   - **Decisions & Open Questions** — `.q.decided` (checked disabled) vs `.q` (interactive)
4. Replace the footer `{{DATE}}` with the real date and a Source link to the spec.

Use `SWAP N.=M:` ops on the section bodies — never retype the surrounding CSS/JS or the empty `<section>` shells.

## Verify

1. `browser.open({url: 'file:///path/to/dashboard.html'})`
2. `tab.evaluate(() => Array.from(document.querySelectorAll('section h2')).map(h => h.innerText))` — should return 6 sections in order: Goal, Scope, Flow, Mockup, Risks, Decisions.
3. `tab.screenshot({fullPage: true})` to confirm layout (don't save to disk unless asked).

## Hard rules (from the template)

- **No external refs.** No `<script src>`, `<link href>`, `@import url(...)`, webfonts, or remote `url()`. Must open offline.
- **Never fabricate.** Label inferences as `Assumption:`. Unknowns go in Open Questions.
- **Mini.** Drop any section not discussed; don't pad.
- The localStorage checkbox toggle works at `file://` — the script's try/catch handles sandboxed cases.

## Commit

Single commit: `docs(visualize): <feature> dashboard`.
