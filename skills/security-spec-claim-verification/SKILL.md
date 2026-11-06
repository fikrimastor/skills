---
name: security-spec-claim-verification
description: "Audit a security design spec against real source before implementation: verify every cited file:line, re-derive all counts, and hunt for claims that are true in substance but wrong in evidence — including errors in your own earlier review. Use when asked to review/bless a security spec or plan, especially one making claims about which routes/pages are protected."
---

# Security spec claim verification

Verify a security design spec/plan against source **before** implementation. The spec's §2 evidence tables are load-bearing — later decisions rest on them, so a wrong quote propagates into a wrong fix and a wrong acceptance test.

## Why this is not a normal code review

Security specs fail in three specific ways:

1. **Substance right, citation wrong.** A claim that "the handler bypasses the gate" may be true while the cited line numbers are stale. Readers chase the citation and find nothing.
2. **Counts stated loosely.** "61 pages" vs "76 pages" can both be right; the spec silently means one and the reader assumes the other. Wrong counts make the implementer's enumeration instruction unsatisfiable.
3. **A conclusion resting on an unverified premise.** e.g. "these pages are reached only via middleware X" — the mitigation is sound but the stated reason is false, so nobody notices when a *different* page violates it.

## Procedure

### 1. Fan out verification, one scout per repo
Dispatch a read-only scout per repo with a **numbered list of claims**, each with its cited `file:line`. Require the contract: `spec claim (with cited file:line) | VERIFIED / WRONG / IMPRECISE | actual source line (quoted)`. Instruct: flag a wrong **line number** even when substance is right.

Scout briefs must include: **do NOT run unscoped repo-root greps** — they sweep `.env` and leak credentials into the transcript. Scope to `internal/`, `cmd/`, `app/`, `server/`; add `--exclude=.env`; never print secret values (report length or set/not-set).

### 2. Re-derive every count yourself
Never accept a count from the spec or from a scout. Run the actual `find`/`grep -c`. Concretely, for page/route inventories run **two** commands, because they answer different questions:

```bash
find app/pages -name '*.vue' | wc -l                      # total files
grep -rl 'definePageMeta' app/pages --include=*.vue | wc -l  # files with the meta block
```

Then enumerate the actual gap set — do not trust a summary of it:
```bash
for f in $(find app/pages -name '*.vue' | sort); do
  rel=${f#app/pages/}
  case "$rel" in <prefix-covered-dir>/*) continue;; esac
  grep -qE "middleware:\s*(\[|\"|')" "$f" || echo "UNGATED $rel"
done
```
Also check whether the ungated files even have a `definePageMeta` block — if not, "add middleware" means *creating* the block, a materially different edit.

### 3. Test the mitigation's own premise
For each proposed fix, ask "does this actually address every case the spec claims?" The recurring trap: **a rendering-mode change (`ssr:false`) is fail-closed for server-rendered HTML only, never for authorization.** Client-side reachability survives. Verify by asking what an anonymous *browser* sees, not just what `curl` returns.

### 4. Separate vectors need separate assertions
If a fix addresses two vectors, the acceptance criteria need two tests, and at least one must exercise the surface the other cannot. `curl` cannot verify a client-side redirect. Flag any AC that would pass while the bug remains — that is a false green and worth an explicit warning in the spec.

### 5. Check your own prior review
If you already reviewed the spec, re-verify your own added claims. Errors introduced during review are the most dangerous, because they carry a reviewer's authority. Re-derive every number you wrote.

## Output

Group as: **findings that block execution** (an AC that cannot pass, a citation that misdirects, a premise that is false) vs **accuracy fixes** (counts, attributions). State the verdict, then apply corrections only on explicit approval.

## Failure modes to name explicitly

- Claiming a fix covers a case it structurally cannot reach.
- Quoting a line number from a sibling file (adjacent files with similar structure invite this).
- Treating a comment and live code as equivalent — check whether the line is executable.
- Accepting "no other path exists" without enumerating alternatives; if a spec says "the only path is X", grep for other constructors of the same entity.
