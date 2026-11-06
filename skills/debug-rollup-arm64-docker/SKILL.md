---
name: debug-rollup-arm64-docker
description: Debug @rollup/rollup-linux-arm64-gnu missing on ARM64 Docker with Yarn 1
---

# Debug rollup ARM64 Docker failures

## Symptom

`yarn dev` (Vite) throws:
```
Error: Cannot find module @rollup/rollup-linux-arm64-gnu
```

And/or `yarn install` gives:
```
error @rollup/rollup-darwin-arm64@X.Y.Z: The platform "linux" is incompatible with this module.
```

## Root causes (both must be fixed)

1. **Direct darwin devDependency** — `@rollup/rollup-darwin-arm64` is listed in `package.json` `devDependencies`. Rollup already declares it as an optional dependency; listing it directly makes Yarn treat it as **mandatory on every platform**, so it errors on Linux.

2. **Stray `package-lock.json`** — This project uses Yarn 1 (only `yarn.lock` should exist). npm's lockfile schema corrupts optional dependency resolution for platform-specific packages on ARM64.

## Fix

```bash
# 1. Remove the darwin-specific direct dep
# Edit package.json: delete the "@rollup/rollup-darwin-arm64" line from devDependencies

# 2. Remove stray npm lockfile
rm package-lock.json

# 3. Force re-resolve all deps (ignores Yarn's broken cache)
yarn install --force

# 4. Verify
yarn dev
```

`yarn install --force` is necessary even after removing the dep — Yarn caches the previous resolution state.
