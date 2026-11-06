---
name: tricipta-paired-pr-frozen-contract-check
description: "Review a Tricipta PR touching API routes/endpoints in asset-web or asset-api: find the paired sibling-repo PR and the frozen contract doc in tricipta/docs/superpowers before judging a route change as unrequested, missing, or broken."
---

# When to use

Any review/verification task where a Tricipta PR changes an API path, endpoint namespace, or request/response contract in `tricipta-edafy-asset-web` or `tricipta-edafy-asset-api` — especially when a reviewer (human or subagent) claims "the backend has no such route" or "this change is unrequested scope creep".

# Why

Tricipta API work ships as **paired PRs** (one web, one api), and the contract that governs them is often an **untracked markdown doc** in the parent docs repo, not git history. Reviewing only the PR under review, or only the sibling checkout's current branch, produces confidently wrong findings.

# Procedure

1. **Find the paired PR, don't infer from the local checkout.** The sibling repo's checked-out branch is frequently an unrelated in-flight effort.
   ```bash
   gh pr list --repo AFED-digital/tricipta-edafy-asset-api --state open --limit 20
   ```
   Match by title/theme (e.g. "insight caching and theme rollup"), then confirm the routes exist on that branch:
   ```bash
   cd <sibling-repo> && git fetch origin
   git grep -n "<route-literal>" $(git for-each-ref --format='%(refname:short)' refs/remotes | head -30) -- '*.go'
   ```
   Echo handler route strings are relative to the group they are registered on — read the `RegisterRoutes` call site in `internal/router/router.go` to get the real prefix (`protected := api.Group("")` = `/api/v1`).
2. **Check for a frozen contract doc before ruling on scope.**
   ```bash
   ls ~/Sites/afed/tricipta/docs/superpowers/specs ~/Sites/afed/tricipta/docs/superpowers/plans
   git -C ~/Sites/afed/tricipta status --short -- docs/superpowers   # ?? = untracked draft
   ```
   Read the namespace/token tables and the *non-goals* carefully; note the doc's `Status:` line (a "Draft for review" weakens a blocker into a should-fix).
3. **Check whether the old routes are deleted on the paired branch** — that is what turns a rename into a lockstep deploy, which is the actual runtime risk:
   ```bash
   git grep -n "<old-route>" <paired-branch> -- '*.go' | head
   ```
   If zero hits, state the merge/deploy order requirement explicitly; a web-only merge 404s against current api `main` (and on the field-dashboard path a 404 may be swallowed into a client-side fallback rather than surfaced as an error).
4. **Timestamp the evidence.** Compare commit times with the doc's mtime — it distinguishes "contradicts a pre-existing frozen contract" from "two same-day efforts to reconcile". Never assert intent, only ordering.
5. **Scope every non-goal before repeating it.** Non-goals written for change A ("This change must leave `<file>` byte-identical") do not bind an unrelated parallel PR. Downgrade such a reviewer finding to a coordination note; keep only the freeze the handover table actually imposes on that work (namespace + module tokens + op shape).

# Reporting

Separate: (a) runtime breakage (lockstep/merge-order, silent-fallback risk), (b) contract deviation vs the frozen table — split *mechanical* (prefix, token names) from *contractual* (op count/shape, new body fields), (c) coordination/design-drift notes. Quote doc line numbers and file:line evidence for each; state doc provenance (tracked vs untracked, draft vs frozen).
