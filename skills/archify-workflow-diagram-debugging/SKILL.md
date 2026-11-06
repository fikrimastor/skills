---
name: archify-workflow-diagram-debugging
description: "Efficiently debug Archify workflow diagram_type layout-constraint validation failures (node exceeds lane bounds, edge-through-node, label-route-clearance, mainPath backward-step) instead of guessing at widths/routes. Use when authoring or fixing an Archify workflow.schema.json diagram and archify validate workflow returns layout/constraint or composition/* errors."
---

# Archify Workflow Diagram Layout-Constraint Debugging

Archify's `workflow` diagram_type layout validator (`archify validate workflow <file> --json`) enforces column/lane geometry that is **not** simply `viewBox_width / num_columns`. These rules were learned empirically across ~20 validate/fix iterations on two real diagrams (a 13-task implementation-plan workflow, and a 9-node cross-repo request-flow diagram with a backward "return to caller" edge).

## Diagnostic loop

1. Write/edit the JSON spec.
2. Run and parse just the essentials, don't read the raw error dump:
   ```bash
   node archify.mjs validate workflow <file> --quality showcase --json 2>&1 | python3 -c "
   import json,sys
   d=json.load(sys.stdin)
   print(d.get('error','OK'))
   for x in d.get('diagnostics',[])[:12]:
       print(x['code'],'|',x['message'][:150])
   "
   ```
3. Fix diagnostics in this priority order — fixing an earlier class often removes several later ones for free:
   1. `layout/constraint` (node exceeds lane bounds, nodes too close, label wider than node)
   2. `clean-flow/edge-through-node`
   3. `composition/proper-crossing`
   4. `composition/*-segment` (micro-segment, short-interior-segment)
   5. `composition/label-route-clearance`
4. Re-run step 2. Repeat until `OK`.
5. Only then: `deliver` the same file, then `visual-check --json` and confirm `containment: pass` at every reported viewport (1440x900 through 2048x1320, light+dark). A diagram can validate but still overflow visually — always run visual-check too.

## Column/lane sizing rules (undocumented, learned by trial)

- **Node width budget depends on column position.** Col 0 and the last column (5, in a 6-column diagram) need width ≤ ~96-100px or you get "exceeds the horizontal bounds of lane". Middle columns (2-4) tolerate up to ~130-150px.
- **Widening `viewBox` does NOT proportionally fix same-lane column crowding.** Going from viewBox width 1500→1800 did not resolve two same-lane adjacent-column nodes overlapping. The intra-lane column pitch has its own, much smaller effective budget than `viewBox[0] / 6` would suggest — don't burn cycles just cranking viewBox wider.
- **SAFE pattern: two nodes at the SAME `col`, DIFFERENT `lane`.** Reliable, used repeatedly with zero collisions (e.g. a validation node and a DB-query node both at `col: 2` in different lanes).
- **RISKY pattern: two nodes in the SAME `lane` at ADJACENT columns** (e.g. col 3 and col 4). Width sums that work for one adjacent pair (96+96=192 at cols 2-3) can still fail for a different adjacent pair (96+110=206, even 96+82=178 at cols 3-4). Don't try to fix this by iteratively shrinking widths — treat repeated "less than 8px apart" errors on the same pair as a structural smell.
- **The real fix for that smell: don't add the second node.** Route the "extra detail" as an additional EDGE back to an EXISTING node in that lane (e.g. a response/return edge back to the node that made the original call) instead of inventing a new box. This is usually also more architecturally accurate — many real systems are "one function does A, calls out, and does B with the result", which is better modeled as one node with an extra return edge than two separate boxes squeezed into adjacent columns.

## `mainPath` rules

- Every consecutive pair in `mainPath` must have a real, same-direction edge between them, or validate reports "has no matching edge".
- Columns referenced by consecutive `mainPath` entries must be non-decreasing; a backward column step reports "moves backward ... use a return edge outside mainPath for loops".
- It is fine, and often necessary, to OMIT a node from `mainPath` (it still renders with all its edges) if including it would require a backward step or a missing edge — `mainPath` only drives the guided "Play story" trace, not every node in the diagram.

## Edge routing

- `route` values: `auto | straight | drop | outside-right | return-left | bottom-channel | up-channel`.
- For an edge that runs BACKWARD (a return/response arrow from a later stage to an earlier one), `up-channel` reliably found a clear corridor. `bottom-channel` and `outside-right` on the same backward edge tended to collide with existing forward-path corridors or intervening nodes.
- `channelX`/`channelY` did NOT reliably separate two overlapping same-corridor edges in practice. Switching the `route` keyword itself is what actually worked — try a different route value before spending cycles tuning channelX/Y.
- When `validate` reports a label-route-clearance or label-overlaps-node error, it very often prints an exact `Suggested fix: labelAt [x, y] or labelDy +N (below); or labelAt [x, y] or labelDy -N (above)`. Paste those exact numbers as the edge's `labelAt`/`labelDy` — this consistently resolved the error on the first try; guessed offsets often didn't and cost another 2-3 iterations.

## CLI gotchas

- `archify.mjs deliver <type> <file> --quality <profile>` has **no `--force` flag** on the installed CLI (confirmed on 0.9.48, despite some docs implying one exists). Omit it — `deliver` overwrites the output path by default.
- `meta.output` in the spec JSON must be a path **relative to the current working directory**, not absolute — an absolute path fails with `output/meta-absolute`.
- **Never hand-edit the delivered HTML file.** It is fully regenerated by `deliver` and hand edits are silently lost on the next run. Always edit the source JSON spec and re-run `validate` → `deliver` → `visual-check`, even for a single-node/single-edge change.
