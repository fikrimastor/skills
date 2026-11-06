---
name: esri-basemap-identity-verification
description: "Determine which Esri/ArcGIS basemap or tile service a map surface actually renders — from source, from git history across all refs, and by pixel-matching a live render against a direct tile fetch. Use when asked what basemap/map type is in use, when a map looks like it changed style, or when a dark map is suspected of being a dark basemap."
---

# Verifying an Esri/ArcGIS basemap identity

Use when answering "what map type does this use?", "did the basemap change?", or when a map renders unexpectedly dark/light and a basemap swap is suspected.

## 1. Read the source, don't infer from the look

Three distinct Esri constructions exist and they take different code paths — check which one it is:

- **Built-in id**: `new Map({ basemap: 'oceans' })` — a named Esri basemap.
- **Custom `Basemap`** (hybrid): `new Basemap({ baseLayers: [...], referenceLayers: [...] })` with `TileLayer` instances. This is how "Imagery with Labels" is built — **World_Imagery ships no labels**, so an explicit `Reference/World_Boundaries_and_Places` layer is required.
- **Third-party tiler**: Leaflet `L.tileLayer(url, ...)` — same Esri services, different library. Don't conflate the library with the basemap.

Grep for: `basemap:`, `Basemap(`, `TileLayer`, `arcgisonline`, `L.tileLayer`.

## 2. Git-archaeology the basemap across ALL refs

The current branch may not contain the change. Sweep every ref for the literal values:

```bash
git log --all --oneline -S "World_Imagery"
git log --all --oneline -S "Dark_Gray" -S "dark-gray" -S "World_Dark"
# enumerate every basemap literal that has ever existed in the file
git rev-list --all | while read c; do
  git show $c:path/to/Map.vue 2>/dev/null | grep -oE "basemap: \"[^\"]+\"|World_[A-Za-z_]+"
done | sort -u
```

An empty `-S` result is meaningful evidence: the value **never existed** in this repo.

## 3. CRITICAL TRAP — ArcGIS REST tile URL is `{z}/{y}/{x}`

```
https://server.arcgisonline.com/ArcGIS/rest/services/<Folder>/<Service>/MapServer/tile/{z}/{y}/{x}
```

`{y}` comes BEFORE `{x}`. Passing `{z}/{x}/{y}` silently returns a **placeholder tile** — a uniform light-gray image averaging about `(205,205,205)`. This is the tell. Two other ways to get that same malformed-URL symptom:

- Shell field-splitting a label into the coordinate vars (e.g. `while read lbl z y x` over a line containing `(`) leaves a var empty → malformed path → placeholder.
- Any transposition that still returns HTTP 200 (the service 200s the wrong tile rather than 404ing).

**Rule out the placeholder before drawing conclusions.** If every sample returns an *identical* average color, the URL is malformed, not the imagery. Build and fetch the URL inside ONE process (Python `urllib`, not shell interpolation) so field-splitting can't corrupt it.

Web-mercator tile math (z/y/x):

```python
import math
n = 2**z
x = int((lon + 180) / 360 * n)
lr = math.radians(lat)
y = int((1 - math.log(math.tan(lr) + 1/math.cos(lr)) / math.pi) / 2 * n)
```

## 4. Prove it against the live render

Do not stop at the tile fetch — cross-check the running app:

1. Open the map page and screenshot it.
2. Sample pixels **inside the map panel** (avoid markers/bubbles/legend), e.g. with PIL `getpixel`.
3. Fetch the tile at the field's coordinates from each candidate service.
4. **Exact match wins.** A live interior matching a direct fetch is decisive; two independent sources agreeing beats any amount of inference.

Also confirm there is no CSS override before blaming styling — grep the component's scoped `<style>` and `:deep(.esri-view)` rules; they often only set width/height and accent colors.

## 5. Don't misread satellite imagery as a dark basemap

The single most common false conclusion: **World_Imagery over deep water renders near-black** (RGB ~`14,36,49` at z10). An offshore field therefore looks like it is using a dark-gray basemap. Land on the same service renders olive/green (~`83,85,46`).

To discriminate:
- Sample the same zoom at an **inland** coordinate. Ocean-black + land-green ⇒ satellite imagery, not a dark basemap.
- Check the **attribution string** in the render. Esri credits name the actual service (e.g. "Earthstar Geographics | Esri, HERE, Garmin" ⇒ World_Imagery), whereas a dark-gray basemap would not carry those.
- Assert the negative: if no `Dark_Gray`/`World_Dark` literal exists in any ref, it is not a dark basemap.

## Reporting

State (a) the SDK/library and the exact composition, with `file:line`; (b) the before/after commit pair with dates if the basemap changed, or state that it never changed; (c) for a darkness/lightness question, the pixel evidence that settles it; (d) any sibling map surfaces still on a different basemap (a real inconsistency worth flagging). Keep the ocean-vs-land contrast as supporting color, not as the load-bearing proof.
