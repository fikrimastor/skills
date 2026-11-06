---
name: phantom-modified-file-blocking-checkout
description: "Diagnose a file git reports as modified that is byte-identical to HEAD, where git stash appears to succeed but branch checkout still aborts with 'Your local changes to the following files would be overwritten by checkout' — the CRLF-blob vs .gitattributes eol=lf mismatch. Use when checkout is blocked and re-stashing has no effect, when a file shows M with no real edits, or before running git checkout -f on such a file."
---

# Phantom-modified file blocking checkout

## Symptom

- `git checkout <branch>` → `error: Your local changes to the following files would be overwritten by checkout: <file>` / `Aborting`
- `git stash save` prints `Saved working directory and index state WIP on ...` — **but the next checkout aborts identically**
- The same file reappears as ` M` afterward; repeating the stash changes nothing

## Root cause (be precise)

Git decides "modified" by hashing the **attribute-normalized** worktree content and comparing it to the index blob. When a file's **stored blob is CRLF** but attributes demand `eol=lf` (e.g. `.gitattributes` containing `*.yml text eol=lf`), the normalized form (LF) can never equal the CRLF blob → a permanent one-way mismatch.

**A CRLF blob under `eol=lf` is necessary but NOT sufficient for the dirty flag.** The flag only appears once git re-reads the file (stat-cache miss from a touch, edit, or a checkout that rewrote it). A CRLF-blob file whose cached stat still matches reports **clean** — so a repo can hold many latent mismatches that only surface one at a time. Verify this: a sibling CRLF-blob file in the same repo will report clean.

**Why `git stash` can never fix it:** the worktree file is byte-identical to the committed blob, so the stash captures the phantom diff and restores the phantom. Git re-checks on the next checkout and aborts again. A stash is not a workaround; repeated stashing only manufactures duplicate no-op stash entries.

## Diagnostic ladder

Run these before touching anything:

```bash
# 1. Is any of it REAL? Empty output = line-endings only, nothing to preserve.
git diff --ignore-cr-at-eol --stat -- <file>

# 2. Canonical one-liner: index-eol / worktree-eol / attr-eol
#    e.g.  i/crlf  w/crlf  attr/lf   <- blob and attributes contradict
git ls-files --eol <file>

# 3. Proof the mismatch is attribute-normalization, not stat-cache staleness.
#    raw == HEAD  but  path-filtered != HEAD  =>  normalization mismatch
git hash-object --no-filters <file>            # raw worktree bytes
git hash-object --path=<file> <file>           # after attribute filters
git rev-parse HEAD:<file>                      # committed blob
```

If step 1 is **empty**, no real work is at risk. Do not invent a preservation step.

## Safe-fix procedure

1. **Prove the discard is lossless** (all three, not just the first):
   - worktree bytes == HEAD blob: `cmp <(git show HEAD:<file>) <file>` or matching md5s
   - no untracked path collides with the target tree — check **exact paths AND directory prefixes**:
     ```bash
     git ls-tree -r --name-only <target-branch> > /tmp/tree.txt
     git ls-files --others --exclude-standard | while read -r p; do
       grep -qxF "$p" /tmp/tree.txt && echo "COLLISION: $p"
     done
     ```
   - snapshot untracked md5s so you can prove nothing was lost afterward
2. `git checkout -f <branch>`
3. **Verify after**: re-hash untracked files against the snapshot, confirm stash count unchanged, back up the file first (`cp <file> /tmp/`) for cheap insurance.

## Confirm it is repo-wide, not local

```bash
git clone -q --branch <branch> --single-branch . /tmp/scratch
cd /tmp/scratch && git status --porcelain | wc -l
```
A fresh clone that comes up dirty proves the mismatch is **committed**, reproducible for every collaborator. Single-branch clones lack other branches — `git fetch origin <branch>:refs/heads/<branch>` before round-trip switching tests.

## Durable fix

Renormalize the **blobs**, not the attributes (changing `eol=lf` would only hide the drift and re-dirty everyone):

```bash
git add --renormalize .
git commit -m "chore: normalize line endings against .gitattributes"
```

Rewrites every mismatched blob — do this on a **dedicated cleanup branch**, never inside an unrelated feature PR.

## Traps

- Don't drop stashes blindly after a stash-retry storm. Each failed attempt leaves a duplicate. Check each with `git diff --ignore-cr-at-eol --numstat stash@{i}^ stash@{i}` — **0 files means pure line-ending noise, safe to drop**; nonzero means real edits, keep it.
- `git checkout -- <file>` succeeds (exit 0) yet the file stays ` M` — that is expected here and is not a failed command. Don't loop on it.
- Don't report the cause as "stat-cache staleness" alone: `git update-index --refresh` / `git status` re-reads and re-flags it. The attribute/blob contradiction is the durable cause.
