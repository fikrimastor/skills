---
name: livewire-computed-stale-after-mutation
description: "Diagnose and fix a Livewire action whose state change doesn't take effect until a reload (Approve/Submit button still rendered after clicking it) — stale #[Computed] memoization, not a broken template guard"
---

# Symptom

Click an action (Approve, Submit, Delete) and the UI keeps showing the pre-action state for a control that should now be hidden or disabled — the status badge updates, but the button is still there. A manual page reload fixes it.

# Diagnosis

1. **Check the template guard first.** Find the render site (e.g. `@if($this->canApprove)`). If the guard is present and correct, the condition is not the problem — do not edit the template.
2. **Disambiguate staleness vs. bad condition.** Load the same page fresh in the already-changed state (e.g. a record that is already approved). If the control is correctly hidden on fresh load, the guard works and this is memo staleness. If it is still visible, the condition itself is wrong.
3. **Root cause.** Livewire `#[Computed]` properties are memoized for the whole request. If the action method reads the computed *before* the mutation — typically its own authorization check, `if (! $this->canApprove) { abort(403); }` — that read seeds the cache with the pre-mutation value. The post-action re-render happens in the *same* request and reuses the cached value.

# Fix

Invalidate every computed that derives from the mutated state, immediately after the state change and after any `->refresh()`:

```php
$this->submission->update([...]);
$this->submission->refresh();

unset($this->canApprove, $this->canAmend);
```

Patch all affected computeds, not just the obvious one — same-state computeds (canAmend vs canApprove) stay stale together and leave controls that `abort(403)` when clicked, since they re-check in a fresh request where the value is correct.

This is the established repo idiom — look for sibling `unset($this->a, $this->b)` calls after mutations and follow them.

# Verification

- Drive the test through the Livewire action so the same-request re-render path is exercised:
  `Livewire::test('pages::some.page', [...])->call('approve')->assertDontSeeHtml('<needle>')`.
  A `->get(route(...))` test re-renders fresh and **passes even with the bug present**.
- Assert a needle unique to the control (e.g. `wire:click="approve"`). Never use a label that is a substring of other rendered text — `assertDontSee('Approve')` collides with an "Approved" badge.
- **Falsify:** remove the `unset`, re-run, confirm the test fails, restore. A test that passes both with and without the fix pins nothing.
- Verify live in the browser: click the action and read the DOM *without navigating* — confirm both the control and the dependent affordances are gone.
