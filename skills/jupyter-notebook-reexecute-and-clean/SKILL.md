---
name: jupyter-notebook-reexecute-and-clean
description: "Re-execute a checked-in .ipynb to replace stale or error cell outputs, then strip the execution harness's noise (metadata.execution timestamps, execution_count churn) so the git diff shows only real content changes. Use when a committed notebook's stored outputs are stale or hold a traceback, or after editing notebook source."
---

# Re-execute a committed notebook and clean the diff

A checked-in notebook's saved outputs show what was true when it was last executed, not
what the code does now. Re-running refreshes them — but the harness also writes
metadata that pollutes the diff and can silently drop cells you excluded.

## 1. Snapshot the true "before"

Do this FIRST, before any edit, so you can later diff intent vs noise:

```bash
cp path/to/notebook.ipynb /tmp/nb_pre_exec.ipynb
```

Also diff against `git show HEAD:<path>` — the working tree may already carry the user's
owned uncommitted edits that you must not claim as yours.

## 2. Edit source programmatically, never with a text `Edit` tool

Notebook source lines live inside a JSON string array, so a cell line reads as
`"        \"oil_rate\": oil_rate,\n",` in the raw file. Anchor-based text edits fail with
"close enough match" errors. Load, mutate, dump:

```python
import json, nbformat
nb = json.load(open(p))
cell = next(c for c in nb['cells'] if 'MARKER' in ''.join(c['source']))
src = ''.join(cell['source'])
assert src.count(old) == 1          # guard against silent no-op / multi-match
cell['source'] = src.replace(old, new).splitlines(keepends=True)
json.dump(nb, open(p,'w'), indent=1, ensure_ascii=False)
open(p,'a').write('\n')
```

Assigning a plain `str` to `cell['source']` also works — nbformat normalizes it.

## 3. Execute, excluding cells you cannot run offline

`%pip install` cells need the network and rewrite environment-specific paths. Exclude,
execute, then re-insert verbatim:

```python
from nbclient import NotebookClient
nb = nbformat.read(p, as_version=4)          # after converting to plain str sources
pip = next(c for c in nb.cells if ''.join(c['source']).strip().startswith('%pip'))
nb.cells.remove(pip)
nb.metadata.pop('widgets', None)
NotebookClient(nb, timeout=600, kernel_name='python3', allow_errors=True,
               resources={'metadata': {'path': <notebook's dir>}}).execute()
nb.cells.insert(<original index>, pip)
nbformat.write(nb, p)
```

`allow_errors=True` does NOT mean ignore failures — it means the run completes so you can
enumerate them. Always assert on the result:

```python
errs = [(i, o.get('ename'), o.get('evalue'))
        for i, c in enumerate(nb.cells) for o in c.get('outputs', [])
        if o.get('output_type') == 'error']
assert not errs, errs
```

`resources={'metadata':{'path': ...}}` matters: cells that read relative files resolve
against the notebook's directory, not the process cwd.

## 4. Strip harness metadata

Execution adds per-cell timestamps that no author intended to commit:

```python
for c in nb['cells']:
    if 'execution' in c.get('metadata', {}):
        del c['metadata']['execution']
    c.setdefault('metadata', {})
```

Restarting the kernel also makes `execution_count` collide with the restored `%pip` cell
(both `1`). Renumber sequentially over cells that actually have outputs, set the rest to
`None`, and keep each `execute_result`'s own count in sync with its cell.

## 5. Verify, then read the diff as a reviewer

Run `nbformat.validate(nbformat.read(p, as_version=4))`, and confirm structure survived:
cell count and per-type counts unchanged, cell `id`s byte-identical to `HEAD`, and the
restored `%pip` source equal to its snapshot.

Then compare outputs against the snapshot and classify each difference as intended or
noise. Two real traps:

- **Cascade edits.** Fixing a data-source cell legitimately changes downstream outputs
  (diagnostic counts, resolved units, alias messages). Enumerate them and confirm each is
  explained by your change — do not assume a large diff is all noise.
- **Check every changed cell, not just the one you fixed.** A data change ripples; a
  reviewer will find the cells you did not look at.

Finally, check `git status` for byproducts of running the interpreter — tracked
`__pycache__/*.pyc` and `.DS_Store` get rewritten by an import. Revert them
(`git checkout -- <dirs>`); they are not part of the work.
