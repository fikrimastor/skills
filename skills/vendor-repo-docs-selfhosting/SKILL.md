---
name: vendor-repo-docs-selfhosting
description: "Use when replacing a hosted/cloud repo-documentation generator (Qoder RepoWiki, DeepWiki, etc.) with a self-hosted or local pipeline, or when an AI-generated doc corpus (e.g. .qoder/repowiki/) has gone stale and needs a freshness/update loop."
---

# Replacing a hosted repo-docs generator locally

Goal: keep an AI-generated documentation corpus (RepoWiki-style) updating without the vendor's service. The generator is the replaceable part; the plan, the per-page briefs, and the resume anchor are usually already sitting in the export.

## 1. Inventory the export before designing anything

```bash
find .qoder -type f | wc -l; find .qoder -type f ! -name '*.md' | sort   # find the non-prose artifacts
find .qoder -type f | sed 's|/[^/]*$||' | sort | uniq -c | sort -rn      # files per unit
du -sk .qoder; git ls-files .qoder | wc -l; grep -n qoder .gitignore
```

Generated corpora are typically 50% stubs by count (module cards) and 0% stubs in the narrative tree — measure file sizes per tree (min/median/max, count under 400 B) to find which tree is worth keeping. Per-file size distribution decides the scope, not the file count.

## 2. Decode the contract — look for four things

1. **The plan / editorial policy** (e.g. `wiki_plan.yaml`): scope globs, `notes[]` with house rules (layering, tenancy, glossary requirements), `documents[]` briefs. This is the prompt that produced the corpus.
2. **The per-page briefs**: often stored verbatim in the machine-readable index (e.g. `wiki_catalogs[].prompt`). Reusing them preserves coverage and the quality bar — no re-planning.
3. **The machine-readable index**: the one JSON/YAML with per-item `{path, line_range}` and a `name → files` edge. It is the input to any drift check and any incremental regeneration.
4. **The resume anchor**: a stored last-indexed commit SHA + branch (e.g. `wiki_repo.last_commit_id`, `extend_info.branch`, a `recovery_checkpoint`). Check it still resolves (`git cat-file -t <sha>`) — it means a replacement resumes incrementally instead of starting from zero.

## 3. Measure drift with TWO metrics, never one

Citation targets carry URL fragments (`file://path/to/file.go#L470-L640`). Strip the `file://` prefix and the `#fragment` before testing existence, then report separately:

- dead targets: referenced path no longer exists
- out-of-bounds: `range_end > file line count` (the real staleness signal)

A naive existence check on the raw link reports catastrophic false drift (measured 85.9% vs a true 1.2%). Verify against the index's own `{path, line_range}` rows as a cross-check.

## 4. Measure churn-vs-cadence before proposing incremental anything

For each time window, count changed files and how many doc units cite any of them:

```bash
git log --since=1.week.ago --name-only --pretty=format:   # then intersect with the page→file index
```

If the curve is flat (in a high-churn repo: 64% of units touched at 1 week, 73% at 5 weeks, because ~100 files change weekly), **do not build an incremental engine expecting 10x** — it buys 25-35%. Design for cheap full regeneration, a smaller corpus, or a slower cadence. Also check whether a top-N page cut is even available: if citation mass is diffuse (top-20 units holding ~20% of citations), narrowing must be editorial, not mechanical.

## 5. Check for a vendor-local path before building anything

Verify against primary sources: is the vendor CLI proprietary/login-gated, and does its command surface contain a wiki command at all? Also confirm where state lives — if the export carries `repo_id`, `progress_status`, and encrypted `raw_data`, generation is server-side and there is no offline mode. `npm view <pkg>` shows the license and publisher.

## 6. Evaluate replacements with ONE decisive fact each

Not feature lists. The facts that actually discriminate:

- **incremental mechanism** (git-HEAD tracked & diffed? or full regen every run? quote the source file)
- **provider flexibility** (can it point at any OpenAI-compatible base URL / your own gateway? name the config key)
- **language support** (is the AST genuinely multi-language, or `.py`-only in practice?)
- **maintenance recency** (last commit date; is it superseded by a successor project?)

## 7. Prefer the already-installed local toolchain for the free tier

A deterministic structural layer costs $0 and needs no key: `graft build` (wiring graph + per-file cards; `graft check` exits 1 on drift and is *"Pure I/O + hashing: no LLM, no network"*) and `graphify update` / `graphify export wiki` (AST-only, changed files only).

**Verify capabilities against the installed artifact, never against piped/truncated `--help`.** A `help | head -40` cut off an entire `export` subcommand family and produced a wrong claim in a delivered document. Recipe: resolve the launcher (`cat $(command -v tool)` → shebang → venv path), then grep the package source for the subcommand name (`grep -n 'wiki' <pkg>/cli.py`), then run it.

## 8. Deliver three tiers + one explicit decision

- **Free tier**: deterministic structure + graph/wiki export, always fresh, sub-second.
- **Paid tier**: the narrative corpus, deliberately scoped to units carrying editorial judgment — do not default to all of them.
- **Gate**: a script recomputing the §3 metrics from the index, exiting non-zero past a threshold; wire into CI as a warning first, then ratchet to blocking.
- **Decision to surface, never decide silently**: track the corpus in git (so it is reviewable and diffable) or shrink it and let the free tier carry breadth. An untracked generated corpus dies with the next `git clean -xfd` and no reviewer can diff it.

## Pitfalls

- `.gitignore` membership is the first thing to check: an untracked corpus is invisible to CI and review regardless of quality.
- Citation line ranges rot faster than prose (14.1% out of bounds in ~5 weeks of churn) — make range validity part of the gate, not just file existence.
- Coverage decays differently from staleness: count changed files cited in *no* page (measured 133/169). New packages silently have no documentation and a "refresh stale pages" loop will never notice.
- Community/analysis data can lag the graph it was built from; re-run the clustering step before exporting a wiki, or articles reference dropped nodes.
