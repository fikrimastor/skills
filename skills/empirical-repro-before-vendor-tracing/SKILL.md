---
name: empirical-repro-before-vendor-tracing
description: "Use when a bug involves unexplained state mutation (a file, timestamp, or record changing) in an app built on a vendored/third-party SDK in node_modules, before or instead of statically tracing through that SDK's source to find the write site."
---

## Problem

A file/record mutates (mtime bumps, a field changes, an entry appears) from an action that looks read-only (opening/viewing/selecting something), and the mutation is caused by a vendored dependency (node_modules) rather than your own repo code. Statically tracing a large third-party SDK's source (thousands of lines, many indirections: constructors calling helpers calling other classes) to find the exact write site is slow and easy to get lost in — deferred/conditional writes, wrapper methods, and re-exports all obscure the actual call path.

## Technique

Reproduce empirically against the real running process instead of reading more source:

1. **Baseline**: capture the target file's identity before the action — `stat -f '%m %Sm' <file>`, `md5 -q <file>` (or equivalent checksum), `wc -l < <file>`. Cheap, exact, no ambiguity.
2. **Act for real**: perform the exact action through the actual running app (browser tool clicking the real UI, CLI invocation, etc.) — not a hand-rolled script that approximates the code path. Hand-rolled repros can miss the exact option combination that triggers the write.
3. **Recheck immediately, then again after a delay**: some writes are async/deferred (e.g. background title generation, debounced flush). A clean immediate recheck does not rule out a delayed write — recheck again after real time has passed (tens of seconds to a couple minutes) before concluding "no write happened."
4. **If changed, diff the content**, not just the mtime: `tail -N <file>` before vs after, or a real diff. The appended/changed content usually names the mechanism directly (e.g. an entry with `type`/`reason` fields), which turns "where in 3000 lines of SDK source is this coming from" into a precise `grep` for that literal string.
5. **Only then**, if still unclear, grep the vendored source for the literal token found in step 4 (entry type name, function name) — this is a targeted, bounded search instead of an open-ended read.

## Gotchas hit while doing this

- **Browser tool + live-updating pages**: if the page receives push updates (WebSocket, polling), a snapshot/ref captured in one `run` call can be stale by the next `run` call — re-renders invalidate `aria-ref`s. Observe (query/snapshot) and act (click) inside the **same** `run` call, or re-query fresh state immediately before acting.
- **`Element.closest(selector)` matches the element itself** if it satisfies the selector, not just ancestors — a `class="session-name"` element matches `[class*=session]` and `.closest('[class*=session]')` on it returns itself, not the intended parent row. Use an exact class name or `parentElement` when you mean "the container, not this node."
- **A clean/unchanged recheck right after the action is not proof of "no write"** if the mechanism could be async — this cost an extra investigation round in the ompgui case (the write landed ~90s later, from a session-switch bookkeeping append, not from the open itself).

## When to stop tracing source and switch to this

As soon as static tracing crosses roughly one "layer" without a clear answer (e.g. you've read the direct call site and it looks clean, but the callee is itself calling into 500+ more lines you haven't read) — stop, and spend that time on an empirical before/after check instead. It is very often faster and always conclusive, whereas source-reading can rule out candidates without ever finding the actual site.
