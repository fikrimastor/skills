---
name: asset-api-swagger-regen
description: Regenerate Swagger docs for tricipta-edafy-asset-api (swag init with documented flags only) and verify the spec actually updated
---

# Regenerate Swagger docs — tricipta-edafy-asset-api

## Command (use EXACTLY this)

```bash
go install github.com/swaggo/swag/cmd/swag@v1.16.6   # match go.mod pin
"$(go env GOPATH)/bin/swag" init -g cmd/server/main.go -o docs
```

`swag` is usually NOT on PATH on this workstation (`command not found: swag`) — install, then call it via `$(go env GOPATH)/bin/swag`. A leading `warning: failed to get package name in dir: ./` is harmless.

Source of truth for flags: `.agents/rules/code-style.md:353`.

## Why: do NOT add --parseDependency / --parseInternal

The extended flags `--parseDependency --parseInternal` **break** generation on this repo:
swag fails with `ParseComment error ... cannot find type definition: apperrors.Problem`
and aborts before writing, leaving stale docs in place with no obvious failure.
The minimal invocation parses `apperrors.Problem` fine.

## Verify

1. Output ends with `create docs.go / swagger.json / swagger.yaml`, zero `ParseComment error` lines.
2. Grep `docs/swagger.json` for any newly-annotated route path / changed description — absence means silent failure.
3. Diff the right base: if docs/ are already modified (staged or unstaged) in the working tree, `git status` looks unchanged after regen. Compare regenerated output against the working tree (`git diff` + `git diff --cached`); identical = docs were already current, report that rather than "updated".
4. `go build ./...` clean.

## Gotchas

- Handler annotations live on handler funcs (`@Router /x/y [post]`), mounted in `internal/router/router.go`.
- Commit `docs/docs.go`, `docs/swagger.json`, `docs/swagger.yaml` together with the annotation/DTO changes that caused them; don't commit them alone when the source change is still uncommitted.
