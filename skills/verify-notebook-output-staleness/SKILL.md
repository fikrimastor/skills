---
name: verify-notebook-output-staleness
description: "Verify whether a checked-in Jupyter notebook's saved outputs still match current code, and measure chart claims from a PDF at native raster resolution instead of trusting downscaled renders or vision reads. Use when asked to explain or trust a notebook/spec doc whose outputs may predate recent commits."
---

# Verify notebook stored-output staleness

A notebook's saved cell outputs show **what was true when it was last executed**, not
what the code does now. `git log` on the notebook shows nothing if only the code
moved — the drift is invisible in `git status`.

## Procedure

### 1. Find the outputs that can drift
Filter saved outputs to the ones whose shape comes from code:

```python
import json
nb = json.load(open("path/to/notebook.ipynb"))
for i, c in enumerate(nb["cells"]):
    for out in c.get("outputs", []):
        pj = out.get("data", {}).get("application/vnd.plotly.v1+json")
        if pj:
            print(i, c.get("execution_count"), pj["layout"].get("yaxis"))
```

### 2. Re-run the same call and diff
Reproduce the cell's exact call in the repo venv, then compare the **stored JSON
key set** against the live one. Two independent markers beat one:

- **Absent key** — stored output lacks a key current code always writes
  (e.g. no `autorange`, no `metadata`).
- **String mismatch** — stored `trace.name` / `xaxis.title.text` is a raw metric
  slug where current code emits a display label.

The string comparison is the stronger marker: no interpretation, just equality.

### 3. Date the change with `git log -S`
Attribute the drift to a commit by searching for the symbol the current code uses:

```bash
git log -S'metric_label' --format='%h %ad %s' --date=short -- path/to/emitter.py
```

This dates the emitting change; comparing its date to the notebook's mtime tells
you whether the notebook predates it.

## Measuring charts printed in a PDF

Never reason about chart values from a downscaled render or from a vision/LLM read
of a screenshot — models misread bar order and axis labels at low resolution.

1. Find the panel: `pdfimages -list -f PAGE -l PAGE doc.pdf` — a chart exported as
   an image appears as a ~500x400 RGB raster; vector text appears as `<word` entries
   in `pdftotext -bbox`.
2. Extract at native resolution: `pdfimages -png -f PAGE -l PAGE doc.pdf /tmp/p`.
3. Measure pixels, not impressions — decode the PNG (ffmpeg to rawvideo, or sips to
   BMP, if PIL is unavailable) and compute per-bar widths, e.g. thresholding the
   plotly default blue `#1f77b4`.
4. Assert monotonicity explicitly rather than eyeballing:
   `all(w[i] <= w[i+1] for i ...)` vs the reverse.

## Provenance rules

- A bitmap carries **no generator provenance**. An embedded raster in a slide cannot
  be attributed to a library, version, or commit. Say what the image *shows*; never
  what *produced* it.
- Keep pre/post-fix claims attached only to artifacts you can diff — a text file
  against a live run, or a commit against the code. Not to a picture.
- Axis-title style differences between a slide and current code are **suggestive,
  not proof**.

## Reporting

State the markers as a table (signal | stored | current | introduced-by), and name
which claims are observations of a rendered artifact vs. verified against code.
Retract provenance claims you cannot support rather than softening them.
