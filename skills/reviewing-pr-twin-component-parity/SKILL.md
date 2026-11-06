---
name: reviewing-pr-twin-component-parity
description: "Review a large Vue/Nuxt PR containing copy-paste twin components by adding an explicit parity slice, probing layout claims in a real browser, and hunting dead bindings — instead of trusting per-file reviewer verdicts. Use when a PR adds/rewrites sibling components (VFM vs VPS, table vs well panels), when reviewers return APPROVE on files you suspect diverge, or before endorsing a green CI on a repo with no typecheck gate."
---

# Reviewing PRs with twin components (parity + layout + dead bindings)

Copy-paste twin components are the highest-yield bug class in large Vue/Nuxt PRs, and the
default per-file review fan-out is structurally blind to it.

## 1. Never split a twin pair across reviewer slices

Two near-identical siblings (e.g. `VfmWellAiInsightPanel.vue` ↔ `VpsWellAiInsightPanel.vue`,
`VfmTableAiInsightPanel.vue` ↔ `VpsTableAiInsightPanel.vue`) assigned to two different
agents means **nothing cross-checks them**. The most valuable defect is asymmetric wiring.

Real example — the twins were wired **inversely**, both halves broken for opposite reasons:

| Twin | Declares event | Emits it | Parent listens |
|---|---|---|---|
| `VfmWellAiInsightPanel.vue:55` | yes | **never** (`emit(` count = 0) | yes (`VFMWellDetail.vue:307`) |
| `VpsWellAiInsightPanel.vue:498` | yes | **yes** | **no listener** (`VpsWellSensorDetail.vue:448-457`) |

Vue **component events do not bubble**. A declared-but-never-emitted event makes the
parent's handler dead; an emitted-but-unlistened event makes the click a silent no-op.

**Do:** give one reviewer an explicit **parity mandate** — diff the twins and report every
divergence in fallback handling, `meta.source` checks, request field names, `ctaPath`
normalization, and enabled/disabled controls. Divergence with no stated reason is a finding.

## 2. Probe layout claims in a real browser at a definitive height

A reviewer can read a Tailwind class string, reason "looks fine", and return APPROVE while
the layout is fully broken. Do not accept that over reproducible geometry.

The failure shape: a rail (`h-full shrink-0`) plus a data column (`flex-1 min-h-0`) inside a
**column-direction** flex container that has a **definite height** (a `h-screen` layout
ancestor). The rail's `height:100%` resolves against the container, and the column with
`flex: 1 1 0%` + `min-height:0` gets **0px** — the grid is invisible below the breakpoint.
Above the breakpoint the container becomes `flex-row` and the same markup is correct.

Reproduce in one call, no app boot:

```js
const html = `<!doctype html><style>
body{margin:0;height:900px}
.shell{display:flex;flex-direction:column;gap:12px;height:100%;min-height:0}
.left{flex:1 1 0%;min-width:0;min-height:0;overflow-y:auto}
.rail{width:100%;flex-shrink:0;height:100%;max-height:100%;min-height:0}
</style>
<div class="shell"><div class="left"><div style="height:400px">TABLE</div></div><div class="rail">PANEL</div></div>`;
const tab = await browser.open({ name: "probe", url: "about:blank" });
await tab.evaluate(`document.open();document.write(${JSON.stringify(html)});document.close();`);
display(await tab.evaluate(`document.querySelector('.left').getBoundingClientRect().height`));
await tab.close();  // height 0 == collapsed
```

Then **re-run with `flex-direction: row`** to prove the bug is breakpoint-scoped, and confirm
the real ancestor gives a definite height (`h-screen`, not `min-h-screen`) — otherwise the
percentage resolves against content and nothing collapses.

**Carry your own probe result into the synthesis at its true severity**, regardless of what
reviewer verdicts say. An empirical `height: 0` outranks a subagent's APPROVE.

## 3. Hunt dead bindings and dead fields by diffing head vs merge-base

- **Orphaned DOM queries:** grep the head for `querySelector` / `watch` targets whose
  template binding the PR deleted. A watcher still querying `[data-theme-hit="true"]` after
  `:data-theme-hit` was removed = silently dead autoscroll.
- **Declared-but-never-populated optional fields:** if a PR adds `metric?: string` to a type
  and a consumer does exact-match lookup (`why.find(i => i.metric === metric)`), check every
  builder actually sets it. Unset → the section renders empty forever.
- **Unreachable handlers:** a `function handleX()` with zero template references while its
  twin is wired live (`@click="handleSend...`). Grep the handler name; one hit (the
  definition) means dead.
- **Blank containers on empty state:** `<div class="bg-white border ...">` wrapping a
  `v-for` with no `v-if` on the wrapper renders bordered empty cards.

## 4. Measure the typecheck delta yourself — CI may not gate on it

A PR can be green on every CI check and still add type errors when the workflow gates only
`lint (continue-on-error)` → `test` → `build`. Diff type errors against the merge-base rather
than trusting CI, but **normalize line numbers** (they shift) or you will misattribute
pre-existing errors:

```bash
for r in base head; do
  vue-tsc --build --force 2>&1 | grep -E '^\S+\.(vue|ts)\([0-9]+,[0-9]+\): error TS' \
    | sed -E 's/\([0-9]+,[0-9]+\)/|/' | sed 's/[0-9]//g' | sort > /tmp/$r.txt
done
comm -13 /tmp/base.txt /tmp/head.txt   # genuinely new
comm -23 /tmp/base.txt /tmp/head.txt   # genuinely fixed
```

Confirm the merge-base actually equals `origin/main` (`git rev-parse origin/main <base>`)
before attributing anything — and verify a suspicious identifier really is pre-existing with
`git show origin/main:<file>`, not by assumption.

## 5. Test-quality checks specific to this shape

- A new test that **asserts the broken state** (`expect(button).toBeDisabled()`) locks the bug
  in rather than catching it. Flag it.
- Check whether the changed assertion targets **now-dead code**: when the PR removes a
  function's only production caller but keeps the export, the test file adapts to test dead
  code. Either delete both or note the retention explicitly.
