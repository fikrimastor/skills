---
name: laravel-appearance-theme-integration
description: "Use when building light/dark/system appearance switching in Laravel Inertia Vue Tailwind apps, porting starter-kit theme controls, or fixing dark-mode flash, Tailwind dark variant, cookie/localStorage sync, and first-paint theme drift."
---

# Laravel Appearance Theme Integration

## Overview
Build theme switching as one contract across CSS, the first HTML paint, Laravel middleware, and Vue state. A toggle alone is incomplete if it flashes, ignores `system`, or stores a browser-only value the Blade shell cannot read.

## When to Use
- Laravel 11/12 apps using Inertia Vue and Tailwind that need light, dark, and system appearance modes.
- Porting Laravel starter-kit/shadcn-style appearance controls to another project.
- Fixing dark-mode flash, `dark:` utilities not applying, or theme state drifting between reloads and Inertia navigation.

## Integration Contract

| Layer | Required outcome |
|---|---|
| Tailwind CSS | Class-driven `dark:` variant and semantic tokens for `:root` and `.dark`. |
| Laravel middleware | Share an `appearance` value with the root Blade view from a readable cookie. |
| Cookie encryption | Exclude the appearance cookie from encryption because Vue writes it directly. |
| Root Blade shell | Apply the cookie value on `<html>` and run a pre-Vite system-mode script before first paint. |
| Vue composable | Own `light | dark | system`, resolve system, sync localStorage + cookie, and update `<html class="dark">`. |
| UI | Thin settings/user-menu controls consume the composable; they do not duplicate storage logic. |

## Implementation Recipe

1. Define shared types: `Appearance = 'light' | 'dark' | 'system'` and `ResolvedAppearance = 'light' | 'dark'`.
2. Add Tailwind v4 class dark mode in the CSS entry: `@custom-variant dark (&:is(.dark *));`. Keep theme colors as CSS variables: map Tailwind tokens in `@theme inline`, define light values in `:root`, dark values in `.dark`, then base-apply `bg-background text-foreground`.
3. Create `HandleAppearance` middleware that runs in the web group before Inertia rendering. Coerce the cookie to `light`, `dark`, or `system`, then share it with the root view as `appearance`.
4. In `bootstrap/app.php`, add `appearance` to `encryptCookies(except: [...])`, then append `HandleAppearance::class` before `HandleInertiaRequests::class` in the web middleware group.
5. In the root Inertia Blade file, set `<html ... @class(['dark' => ($appearance ?? 'system') == 'dark'])>`. In `<head>`, before `@vite`, add a small inline script: if the server value is `system` and `matchMedia('(prefers-color-scheme: dark)').matches`, add `dark` to `document.documentElement`.
6. Create a Vue composable with SSR guards. It should keep a module-level `appearance` ref, compute `resolvedAppearance`, initialize from localStorage or `system`, listen for system preference changes, and expose `updateAppearance(value)` that writes both `localStorage.setItem('appearance', value)` and `document.cookie = 'appearance=' + encodeURIComponent(value) + ';path=/;max-age=31536000;SameSite=Lax'` before calling `updateTheme(value)`.
7. Call the theme initializer once from the app entry. The Blade script covers first paint; the root app keeps the `<html>` class stable across Inertia navigation; components should not re-read storage.
8. Add UI last: a settings page segmented control for Light/Dark/System and optional user-menu quick toggle using `resolvedAppearance`. Use existing routing conventions/Wayfinder when linking to the appearance page.

## Reference Example

```php
// app/Http/Middleware/HandleAppearance.php
public function handle(Request $request, Closure $next): Response
{
    $appearance = $request->cookie('appearance');

    View::share('appearance', in_array($appearance, ['light', 'dark', 'system'], true) ? $appearance : 'system');

    return $next($request);
}
```

```php
// bootstrap/app.php
$middleware->encryptCookies(except: ['appearance']);
$middleware->web(append: [
    HandleAppearance::class,
    HandleInertiaRequests::class,
]);
```

```blade
<html lang="{{ str_replace('_', '-', app()->getLocale()) }}" @class(['dark' => ($appearance ?? 'system') == 'dark'])>
<head>
    <script>
        (function () {
            const appearance = @json($appearance ?? 'system');
            if (appearance === 'system' && window.matchMedia('(prefers-color-scheme: dark)').matches) {
                document.documentElement.classList.add('dark');
            }
        })();
    </script>
    @vite(['resources/js/app.ts', "resources/js/pages/{$page['component']}.vue"])
</head>
```

## Verification

- Light, Dark, and System controls update `<html class="dark">` immediately.
- Reload keeps the selected mode; `localStorage.appearance` and the `appearance` cookie match.
- System mode follows OS color-scheme changes without selecting a fixed light/dark value.
- A hard reload in dark/system-dark has no light flash before Vue mounts.
- Tailwind `dark:` utilities visibly apply; if not, check the CSS `@custom-variant` directive first.
- In Laravel tests or smoke checks, assert the `appearance` cookie is unencrypted/readable and the root Blade receives the expected `$appearance` value.

## Red Flags and Fixes

| Rationalization / mistake | Fix |
|---|---|
| "The composable toggles it on mounted" | Add the root Blade pre-paint script. |
| "localStorage is enough" | Also write a readable cookie so Laravel can render the shell correctly. |
| Forgetting cookie encryption exception | Add `appearance` to `encryptCookies(except: [...])`. |
| Adding `dark` class but no Tailwind variant | Add Tailwind v4 `@custom-variant dark`. |
| Duplicating localStorage logic in components | Keep storage/class updates in one composable. |
| Treating `system` as a stored resolved value | Store `system`; resolve to light/dark at runtime. |
| "Use a DB preference now, just in case" | Avoid DB storage unless cross-device sync is explicitly required. |
