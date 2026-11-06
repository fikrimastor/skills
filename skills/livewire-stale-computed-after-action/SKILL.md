---
name: livewire-stale-computed-after-action
description: "Fix and regression-test Livewire UI that stays stale after its own action succeeds — a button, panel, or control still rendered post-action that a page reload would hide. Use when an action works (DB updates, badge flips) but the control reacting to it doesn't, or when writing Livewire action tests."
---

# Livewire: stale `#[Computed]` UI after a successful action

## Symptom

A control that should disappear once an action succeeds is still rendered after the action completes — but a full page reload renders it correctly. Verbose mode: every side effect landed (DB row updated, status badge already shows the new value), only the *reactive* control is wrong.

## Root cause

`#[Computed]` props are memoized **for the duration of one request**. If the action method *reads* the computed prop before mutating the state it derives from — classically as its own authorization guard — that read seeds the cache with the pre-mutation value. The post-action re-render happens in the *same* request, so the template's (perfectly correct) `@if($this->canApprove)` re-reads the stale value.

```php
public function approve(): void
{
    if (! $this->canApprove) {      // ← memo seeded TRUE here
        abort(403);
    }

    $this->submission->update([...]);   // state changes
    $this->submission->refresh();
    // re-render: @if($this->canApprove) reads the cache → still true
}
```

If the action never read the prop, the memo would first populate at render time — *after* the mutation — and be correct. The action's own guard is what poisons it.

## Diagnose before editing

1. **Check the template guard first** — it is usually already correct. Don't "fix" a guard that says `@if($this->canApprove)`.
2. **Load the page fresh in the already-final state** (a record already approved/locked). If the control is correctly *absent* on a fresh load, the guard is fine and staleness is the only remaining explanation. One step, distinguishes both hypotheses.
3. Enumerate every `#[Computed]` that derives from the state the action mutates, and check which of them the action reads before mutating. Any that are *also* gated in the template (edit affordances, add-entry panels) share the bug.

## Fix

Invalidate immediately after the state change, next to it:

```php
$this->model->refresh();

unset($this->canApprove, $this->canAmend);
```

Follow the codebase's existing convention if one exists — grep for `unset($this->` in sibling components first.

**Audit siblings.** A computed derived from mutated state is a bug at *every* action that mutates that state, not just the reported one. Check each action in the component. Components that read the same state via a direct query (`->exists()`) or a plain non-`#[Computed]` method invoked inside `with()` re-evaluate every render and are safe — say so explicitly rather than assuming.

## Regression test

- **Drive it through the action, not a fresh request.** `Livewire::test('pages::x.y', ['model' => $model])->call('approve')` on one component instance exercises the same-request re-render. A `->get(route(...))` re-renders from scratch and **passes even with the bug present**.
- **Pin the control, never its label.** If a status badge renders "Approved", `assertSee('Approve')` matches the badge and the assertion is meaningless in both directions. Use a markup needle unique to the control: `wire:click="approve"`.
- **Assert both directions in one chain** — present before, absent after. The "present" half proves the needle genuinely matches the rendered control.
- **Falsify it.** Temporarily remove the `unset(`, re-run, confirm the test fails at the post-action assertion, then restore. Without this the test may be vacuous — and this is the only thing that proves it catches the bug.
- Satisfy `mount()`/guard preconditions the component requires (e.g. mount may `abort(403)` unless the acting user's `client_id` matches the record owner's).

## Assertion helper gotcha

`assertSeeHtml($v)` is **not** identical to `assertSee($v, escape: false)`. `assertSeeHtml` calls `$this->html()`, whose signature is `html($stripInitialData = false)` — so it matches the raw response *including* the `wire:snapshot` blob. `assertSee($v, escape: false)` passes `true` and strips it. Prefer the latter when the only thing you want to match is rendered markup; it can only match real markup, not serialized state.
