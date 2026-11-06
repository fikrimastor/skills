---
name: generated-repo-docs-audit
description: "Audit, validate, or replace an LLM/vendor-generated documentation corpus in a repo (Qoder .qoder/repowiki, DeepWiki, graphify wiki, generated wiki sets): decode its machine-readable contract, measure citation drift, decide incremental vs full regeneration, and evaluate free self-hosted alternatives. Use when asked to refresh/keep-update generated docs, or when a repo carries a generated wiki whose freshness is in question."
---

# Generated Repo-Docs Corpus: Audit & Maintenance

Use when a repo carries a generated documentation corpus (Qoder `.qoder/repowiki`, DeepWiki, graphify wiki, similar) and the question is: is it still valid, how do we keep it fresh, and can we drop the vendor?

## Step 1 — Decode the corpus as a contract, not prose

Read the machine-readable index first; it defines everything and is usually plain files even when the generator is proprietary.

Typical shape (Qoder RepoWiki export):
- **Plan/policy file** — scope globs, editorial notes (layering rules, "every domain page MUST end with a glossary"), per-document goals/templates.
- **Metadata JSON** — page table, and critically: per-page `prompt` (the page brief), `dependent_files` (comma-separated), a `{path, line_range}` snippet registry, and a repo row holding `last_commit_id` — **the resume anchor** the vendor diffs from on re-run.
- **Rendered pages** with `file://path#La-Lb` citations.

Two consequences: (a) the stored per-page briefs are a free quality baseline — reuse them instead of re-planning; (b) confirm the anchor still resolves (`git cat-file -t <sha>`). A gc'd anchor costs you the baseline and forces a cold rebuild.

Also inventory **both** trees if present — narrative pages and module cards are often separate, and one is frequently mostly stubs. Measure sizes; do not assume both matter.

## Step 2 — Measure drift; never eyeball it

Four cheap metrics:
1. **Anchor lag** — `git rev-list --count <sha>..HEAD`, `git diff --name-only <sha>..HEAD`.
2. **Citation line-range validity** — % of `#La-Lb` anchors whose end exceeds the target file's line count.
3. **Dead targets** — referenced paths that no longer exist.
4. **Coverage gap** — changed files cited in *no* page. Usually the bigger problem: it means new code was never documented, which a "refresh stale pages" loop does not fix.

Parsing gotchas that produce false headline numbers:
- Strip the `#La-Lb` fragment (and malformed `#La-b`) **before** any existence check, or every valid anchor reports missing.
- List-like fields may be comma-separated **strings**, not arrays — iterate `value.split(",")`, not the value.
- Distinguish "target file gone" from "range now past EOF"; report them separately.

## Step 3 — Measure churn before designing incrementality

Build a page→file reverse index from the citations, then compute pages-affected for 1/2/3/5-week windows. If the curve is flat (high churn + dense citations), incremental regeneration buys little over a full pass — say so with numbers before proposing a diff engine. Also plot commits/week; a repo with 100+ changed files/week couples most pages to most of the tree.

## Step 4 — Evaluate the free/deterministic layers honestly

- **Check what is already installed** before proposing new dependencies. Typical free layers: `graft` (`graft check` = deterministic drift gate, exit 1, no key/network; `--deep` = LLM pass, takes any OpenAI-compatible base URL) and `graphify` (`update` = AST-only incremental, no key).
- **Verify capabilities from source, never from `--help | head`.** Subcommands are often documented only in a per-subcommand usage string. Real case: `graphify export wiki` works, yet appears in no top-level help line — it is reachable only via the `export` subcommand list in the package's CLI source. A truncated help read produces a confident, wrong "this tool cannot do X".
- **Judge generated artifacts by provenance, not presentation.** Extract the source files each section actually references and compare against its label. Tool-preset sections plus one templated sentence per section ("X groups implementation around a, b, c") are browsable but mislabel the repo. Also check whether community/section labels come from an LLM naming pass — they can be flat wrong.
- **Check whether the artifact is meant to be committed at all.** Some tools deliberately write a gitignored local cache (`node_modules`-style). That is an agent-context artifact, not a doc set, and it cannot substitute for one.

## Step 5 — Decide: track, shrink, or regenerate

- Untracked generated docs are unreviewable, invisible to CI, and die on fresh clone or `git clean -xfd`. If they are meant to be read, track them (next to the repo's existing docs convention).
- If the real consumer is agents, deterministic layers already cover it for free — shrink the human corpus to a small curated narrative set rather than regenerating megabytes forever.
- A **freshness gate** belongs in CI regardless of generator choice: recompute the Step 2 metrics, exit non-zero past a threshold. It is the only piece that catches silent rot, and it is short because the metadata index already holds every `{path, line_range}`.

## Traps

- Truncated `--help` → false "feature missing" conclusions. Re-check in source.
- Vendor CLI under the same brand ≠ a local generator: check the package's license field and publisher, and whether the feature's state is server-side.
- Filename- or community-derived titles mix test scaffolding (`mock*`, `*_test.go`, `testing.T`) in with production code — verify before citing an artifact as documentation.
- Narrative genre cannot be faked by an index: symbol lists, connection counts, and call tables answer "where is X / who calls it", not "how does this system work and why". State which genre a replacement actually produces.
