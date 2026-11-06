---
name: generate-feature-documentation
description: Generate structured documentation for new features following the docs guidelines. Use when asked to create or update feature documentation.
---

# Generate Feature Documentation

Produce structured feature docs under `docs/01-features/` following the
numbered-context pattern from `~/.claude/docs-guidelines.md`.

## Steps

1. **Analyze** the feature using subagents (`explore`) to map:
   - All controllers (with methods)
   - All models (with fields/relationships)
   - All Vue pages
   - Routes grouped by portal (licensee vs licensing)
   - Migrations, seeders, services, tests
   - Architectural notes (duplicate controllers, missing routes, etc.)

2. **Write** each doc following this per-file structure:
   - `docs/README.md` — main index linking to features
   - `docs/01-features/README.md` — TOC with feature summaries
   - `docs/01-features/01-{feature}.md` — per-feature doc
   - `docs/01-features/02-{feature}.md`
   - `docs/01-features/03-{feature}.md`

3. **Per-feature doc format:**
   - Overview paragraph (brief, wrap at ~100 chars)
   - Bullet list of key properties (type IDs, prefixes, statuses)
   - Flow diagram (if applicable, use ` ```text `)
   - Portal sections (Licensee, Licensing) with route tables
   - Data model with migration/seeders list
   - Key services with file paths
   - Controller listing
   - Test listing
   - Key Decisions list

4. **Table formatting rules:**
   | Header | Second |
   | --- | --- |
   - Compact tables (`| --- |` not `|------|`)
   - No blank lines between header, separator, and body
   - Blank line before and after the entire table

5. **Lint with global config** (never write a local `.markdownlint.json`):
   ```bash
   markdownlint --config ~/.claude/.markdownlintrc docs/**/*.md
   ```

6. **Fix violations:**
   - MD013 line-length: wrap prose at word boundaries
   - MD022/MD031/MD058 blanks: ensure blank lines around headings,
     fenced code blocks, and tables
   - MD040: add `text` or `php` language to fenced code blocks
   - NEVER create or modify `.markdownlint.json` in the repo
   - NEVER disable MD060 via CLI — use proper `| --- |` table separators

## Key rules

- ALWAYS use three-dot diff (`origin/base...HEAD`) to identify changes
- Group docs by feature, not by file type
- Note architectural issues (legacy duplicates, missing routes) in callout blocks
- Every doc page starts with an H1 title and a wrapped introductory paragraph
