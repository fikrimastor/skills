---
name: proving-pr-findings-by-execution
description: Review a PR by extracting its head tree to a temp dir and proving each finding with a throwaway Go/test execution instead of only reading the diff — for code review where a plausible-but-wrong finding is expensive and the local checkout is on a different branch.
---

# Proving PR findings by execution

Use when asked to review a PR (especially read-only, "summary only, don't post to GitHub") in a repo whose local checkout may be on an unrelated/dirty branch. Reading the diff is necessary but not sufficient: diff-reading produces plausible findings, execution produces settled ones.

## 1. Get the real head SHA first

```bash
gh api repos/<owner>/<repo>/pulls/<N> --jq '{head:.head.sha,base:.base.sha,branch:.head.ref}'
```

Never review the local working tree. State the head SHA in every finding.

## 2. Extract the PR-head tree to a temp dir

```bash
rm -rf /tmp/pr<N> && mkdir -p /tmp/pr<N>
gh api "repos/<owner>/<repo>/tarball/<SHA>" > /tmp/pr<N>.tar.gz
tar xzf /tmp/pr<N>.tar.gz -C /tmp/pr<N> --strip-components=1
```

This gives the *complete* file at PR head — resolving every diff truncation — without touching the user's checkout. Prefer `read`/`grep` on `/tmp/pr<N>/...` over re-running `gh api contents` per file. The local repo stays dirty and untouched; that is the point.

## 3. Prove each finding with a throwaway test

Write `zz_proof_test.go` files inside the extracted tree (never the user's repo), then:

```bash
cd /tmp/pr<N> && go test ./path/to/pkg/ -run 'TestProof' -v
```

Log the actual values (`t.Logf`) so the transcript IS the evidence, then judge with `if`/`t.Fatalf`. Use a **control case alongside every negative case** — e.g. prove the new metric is dropped AND that the old metric still forms the theme. A control converts "I think this is broken" into "this differs from its sibling."

Patterns that paid off:
- **Byte-identical payload check**: run the same call twice with one input varying; if the captured prompt/request is `==`, the varying field is provably unused. This settles "is this field wired anywhere?" in one test.
- **Orphan emitter check**: grep every hardcoded string-literal set in the module; count emitter values vs consumer cases.
- **Regex/boundary probes**: call the unexported predicate directly on hyphenated/cased/numeric variants.
- **Unguarded-math probes**: table-drive degenerate inputs (zero, negative, both-zero) through the helper and log the outputs.

## 4. Test-package placement

If the package's tests are `<pkg>_test` (external test package), a `package <pkg>` file CANNOT see their mocks. Put the proof file in the same package as the existing tests (`package <pkg>_test`) and import the package, or copy the mock definitions. A mismatch shows as `undefined: mockX` at build time.

## 5. Also run the PR's own suite

```bash
cd /tmp/pr<N> && go test ./internal/application/... ./internal/interfaces/...
```

A green suite is itself a finding: it proves the defect is untested. Say so explicitly rather than implying coverage exists.

## 6. Cleanup

Delete `/tmp/pr<N>`'s proof files or the whole temp tree when done. Verify the user's repo is untouched (`git status --porcelain`) before reporting.

## Reporting

Separate **blocking** (proven user-visible/data-correctness) from **should-fix** from **verified clean**. For every blocking finding give the proof output verbatim. Include a **verified clean** list — claims you actively tested and found correct is evidence, not politeness. Never report a finding you could not reproduce, and never post anything to the PR unless explicitly asked.

## Known trap

Shell output can return a stale/mismatched block under load. If a result's echoed header does not match the command you issued, re-confirm that specific fact via `read`/`grep` on the extracted tree (or by re-running the test) before it reaches the report. Findings backed by an executed test are immune; findings backed only by a recalled bash block are not.
