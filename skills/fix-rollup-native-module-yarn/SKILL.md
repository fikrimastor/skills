---
name: fix-rollup-native-module-yarn
description: "Fix missing @rollup/rollup-linux-arm64-gnu native module when yarn dev crashes on Docker ARM64 Linux (Laravel Sail etc.). Diagnosed and fixed by removing stale node_modules, cleaning stray package-lock.json, and reinstalling with yarn install --force."
---

# Fix missing Rollup native module on ARM64 Linux (Docker/Yarn)

## Symptom

`yarn dev` in a Docker container (ARM64 Linux) crashes immediately:

```
Error: Cannot find module @rollup/rollup-linux-arm64-gnu.
npm has a bug related to optional dependencies...
```

Or `ERR_SERVER_ALREADY_LISTEN` / `vite:dep-scan` errors pointing at the rollup native module.

## Root cause

Yarn v1 (`yarn@1.22.22`) mismanages platform-specific optional dependencies when `node_modules` is stale or when `package-lock.json` (from npm) coexists with `yarn.lock`. The native `.node` binary never lands in `node_modules/@rollup/rollup-linux-arm64-gnu/`, so rollup cannot start.

## Fix inside the container

```bash
# 1. Remove stray npm lockfile if it exists (confuses yarn)
rm -f package-lock.json

# 2. Stale node_modules with wrong platform natives
rm -rf node_modules

# 3. Force reinstall — yarn resolves optional deps fresh for this platform
yarn install --force

# 4. Verify the native module landed
ls node_modules/@rollup/rollup-linux-arm64-gnu/rollup.linux-arm64-gnu.node

# 5. Start dev server
yarn dev
```

## Prevention

- Keep only one lockfile. If `packageManager` says `yarn`, delete any `package-lock.json`.
- After switching branches or pulling new deps, run `yarn install --force` in the container.
