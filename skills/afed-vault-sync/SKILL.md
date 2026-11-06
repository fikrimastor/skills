---
name: afed-vault-sync
description: Use at the end of substantive Afed repo work to ensure Obsidian vault notes are updated manually or via the OMP session_shutdown sync hook.
---

## When to run
Same triggers as `retain`/`learn`: end of a feature, an architecture/design decision, a non-trivial bugfix or refactor, or a research finding worth remembering. Skip trivial Q&A, one-line fixes, and exploratory dead ends.

There are now two paths:

1. **Automatic path:** OMP user hook `~/.omp/agent/hooks/pre/afed-vault-sync-on-shutdown.ts` listens for `session_shutdown` and starts `/Users/fikrimastor/Sites/afed-vault-sync/.venv/bin/python3 -m afed_vault_sync.cli --verbose sync` in the background. Output goes to `~/.local/share/afed-vault-sync/session-shutdown.log`.
2. **Manual fallback:** If the hook/watcher is unavailable, or the user explicitly asks for immediate vault documentation, update the vault note directly with `read`/`edit`/`write` using the workflow below.

## Vault layout
Root: `/Users/fikrimastor/Documents/AFED`
- One folder per repo/project: `ERP/`, `ERP-FE/`, `AFEDTS/`, `McDynect/`, `Mr Churros Dash/`, `WordPress/`, `Lodestar/` (create if missing — check with `read` on vault root first, matching the repo name case-insensitively; add new project folders as needed).
- One folder per repo — never split a repo across two vault folders. If the same repo appears under multiple memory project ids, use the existing single folder.
- One `.md` note per feature/topic within a folder, not one giant log. Reuse an existing note (`edit`) if the topic already has a file; otherwise `write` a new one.
- `Home.md` and per-project `Overview.md` are index/summary pages — update their links when adding a new note, don't duplicate content into them.

## Note schema
```
# <Feature/Topic Title>

> Source: <where this came from, e.g. "Pi session YYYY-MM-DD"> | Branch: `<branch>`

**Branch:** `<branch>`
**Spec:** `<spec path if any>`
**Status:** <COMPLETE | IN PROGRESS | BLOCKED — with a one-line reason>
**Package:** `<key dependency, if relevant>`

**Why:** <one-sentence problem statement>

---

## Key Files
```
<tree of touched/created files with 1-line comments>
```

---

## <Domain-specific sections as needed>

---

## Key Decisions

| Decision | Rationale |
|----------|-----------|

Related notes: [[Other Note]] | [[Overview]]
```

## Manual workflow
1. Identify the repo worked in and map to its vault folder. Check first that no near-duplicate folder exists for the same repo under a different project id; use the existing folder if it does.
2. `read` the target folder to check for an existing note on this topic/feature.
3. Existing note: `edit` to append a dated section (for example `## Update (2026-07-16)`) rather than rewriting the whole file. Use the exact path + `#TAG` from `read`.
4. No existing note: `write` a new `.md` file following the schema above.
5. If the topic connects to another repo/note, add `[[wikilink]]` cross-references in both directions.
6. Do this in addition to, not instead of, `retain`/`learn`; memory and vault serve different audiences.

## Troubleshooting automatic sync
- Run one-shot sync manually:
  ```bash
  cd /Users/fikrimastor/Sites/afed-vault-sync
  ./.venv/bin/python3 -m afed_vault_sync.cli --verbose sync
  ```
- Check shutdown hook log: `~/.local/share/afed-vault-sync/session-shutdown.log`.
- Hook discovery path is user-level OMP native config: `~/.omp/agent/hooks/pre/afed-vault-sync-on-shutdown.ts`.
