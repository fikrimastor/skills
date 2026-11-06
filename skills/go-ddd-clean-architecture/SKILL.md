---
name: go-ddd-clean-architecture
description: Use when writing NEW Go backend code in this repo (tricipta-edafy-asset-api) — a new internal/domain/<name> bounded context, an internal/application/<name>svc use case, an internal/infrastructure/persistence/postgres/<name>_repository.go, or an internal/interfaces/http/handler/<name>_handler.go. Grounded in the actual internal/ code (the `well` bounded context is the canonical reference), not the aspirational rules in .github/copilot-instructions.md. For reviewing already-written PRs/diffs against these rules, use the global go-backend-reviewer skill instead.
---

# Go DDD + Clean Architecture — as actually implemented in this repo

This repo runs two coexisting architectural eras inside `internal/domain/`. `.github/copilot-instructions.md`
describes an idealized Clean Architecture that **does not fully match real code** — it was written ahead of (or
diverged from) the implementation. This skill documents what is **actually there**, verified by reading the
source, so new code matches the codebase instead of the aspirational doc.

> ⚠️ **Never pattern-match against `internal/domain/<name>/` blindly.** About half of those directories are
> legacy vertical-slice code. Check the classification table below before copying anything.

## The two eras

| | NEW — layered Clean Architecture (follow this) | LEGACY — flat vertical slice (never copy) |
|---|---|---|
| Domain package contents | `entity.go`, `value_objects.go`, `repository.go` (interface only), `errors.go`, optional `service.go`/`mocks/` | `entity.go`-ish content **plus** `handler.go`, `service.go`, `mapper.go`, `repository.go` (impl), `dto.go` — everything in one package |
| Imports in domain package | stdlib, `uuid`, `internal/apperrors` (errors only) | `gorm.io/gorm`, `github.com/labstack/echo/v4`, `net/http`, `internal/core/models` — dependency-rule violations throughout |
| HTTP errors | `apperrors.Problem` via `ProblemErrorHandler` | `c.JSON(http.StatusInternalServerError, map[string]interface{}{...})` written directly in the handler |
| Wired in `internal/router/router.go` as | `postgres.NewXRepository(db)` → `xsvc.NewService(repo)` → `authHandler.NewXHandler(svc)` → `RegisterRoutes` | `x.NewHandler(db)` — a raw `*gorm.DB` passed straight into the domain package |

**Confirmed NEW-layered contexts** (use `well` as the primary reference, cross-check against 2-3 others when a
pattern seems ambiguous): `well`, `field`, `platform`, `sensortag`, `tagtype`, `unitofmeasurement`, `reservoir`,
`wellstring`, `wellstringreservoir`, `fieldplatform`, `monthlyproductiondata`, `wellsensorreading`,
`fieldsensorreading`, `vfmcalculated`, `vpscalculated`, `vpsqced`, `vpsdashboard`, `fieldsensoringest`,
`wellsensoringest`, `productioningest`, `welltestingest`.

**Confirmed LEGACY flat contexts** (read-only reference for understanding old code; never extend, never imitate):
`area`, `entity`, `production`, `well_test`, `well_allocation`, `well_analytics`, `stock_oil`, `chemical_usage`,
`esp_monitoring`, `export`, `notification`, `layout_share`, `license`, `realtime`, `reservoir_vrr`,
`production_deferment`, `dashboard_config`, `allocation_run`, `admin_users`, `authentication`. (`dashboard/` was
not deep-classified — inspect its file list before treating it as either.)

`internal/core/models/` is the legacy GORM model package (`BaseModel`, `User`, `Tenant`, `Entity`, `WellTest`,
etc.) consumed only by legacy flat packages. NEW-layered contexts never import it — each defines its own
GORM model in its `infrastructure/persistence/postgres/*_repository.go` file instead.

## Canonical reference chain — read these four files together before writing a new context

1. `internal/domain/well/entity.go`, `value_objects.go`, `repository.go`, `errors.go`
2. `internal/application/wellsvc/service.go`, `dto.go`, `mapper.go`
3. `internal/infrastructure/persistence/postgres/well_repository.go`
4. `internal/interfaces/http/handler/well_handler.go`, plus the wiring block in `internal/router/router.go`
   (search `Clean Architecture: well bounded context`)

## Adding a new bounded context — checklist

1. **Domain** (`internal/domain/<name>/`) — see `references/domain.md`. Entity with exported fields, `New<Name>(...) (*Name, error)` factory, `(n *Name) Update(...) error` mutator, domain-local `Filter`/`ListParams` in `repository.go`, `errors.go` with `domainError` + `init(){ apperrors.RegisterCode(...) }`. No `TenantID` field, no getters, no shared-package imports.
2. **Application** (`internal/application/<name>svc/`) — see `references/application.md`. `service.go` with `Create/List/GetByID/Update/Delete` methods, actor (`createdBy`/`updatedBy`/`deletedBy`) passed as an explicit `uuid.UUID` parameter (never extracted from `ctx`). `dto.go` with `Create<Name>Command`/`Update<Name>Command`/`Delete<Name>Command`/`<Name>Response`. `mapper.go` with exported `To<Name>Response`/`To<Name>ResponseList`.
3. **Infrastructure** (`internal/infrastructure/persistence/postgres/<name>_repository.go`) — see `references/infrastructure.md`. GORM model + `TableName()` in the same file, `dbcontext.DBFromContext(ctx, r.db)` as the first line of every method, `toXModel`/`toDomainX` converters (`toDomainX` returns `(*X, error)`, never discards a `ParseXxx` error).
4. **Interfaces** (`internal/interfaces/http/handler/<name>_handler.go`) — see `references/interfaces.md`. Swaggo-annotated methods, a package-local narrow `<Name>Service` interface, `New<Name>Handler(service)`, `RegisterRoutes(g *echo.Group)`. Errors always `return`ed (`echo.NewHTTPError`, `NewValidationProblem(extractFieldViolations(err))`, or bare `return err`) — never `c.JSON` for an error path.
5. **Wire it** in `internal/router/router.go`'s `Setup()` — that function, not `cmd/server/main.go`, is the real composition root. Add a `// --- Clean Architecture: <name> bounded context ---` block mirroring the `well` block, then call `<name>Handler.RegisterRoutes(protected)` alongside the others.
6. **Tests** — see `references/interfaces.md#testing`. Domain/application/handler tests use the standard `testing` package with hand-written mocks (no testify, no table-driven requirement). Infra repository tests use `testcontainers-go` (`postgres:16-alpine`), following the pattern in `internal/infrastructure/persistence/postgres/testmain_test.go`.

## Load on demand

- `references/domain.md` — entity/value-object/repository/errors patterns, full era classification detail, domain events (none exist).
- `references/application.md` — service/DTO/mapper shape, actor threading, ingest-service `ports.go` pattern, validation split (handler-only).
- `references/infrastructure.md` — GORM repository pattern, `dbcontext` transaction propagation, full `apperrors` RFC 7807 implementation, config/auth/external adapters.
- `references/interfaces.md` — router composition root, handler pattern, auth context keys, middleware inventory, mock generation (`mockery`), testing conventions.

## Known contradictions with `.github/copilot-instructions.md`

Read `.github/copilot-instructions.md` for intent/history, but when it disagrees with the code below, **the code
wins**. Do not "fix" new code toward the instructions doc without being asked — that would break the repo's
actual conventions.

| Instructions doc says | Real code does |
|---|---|
| §3.1a: `Well` uses an unexported guarded `status` field + `Status()` getter | All entity fields are exported, no getters, `Update()` validates then mutates directly |
| §3.3/§12: every repository method takes `tenantID`; multi-tenancy enforced everywhere | No repository method takes `tenantID`; no entity has a `TenantID` field; `tenant_id` is set on the JWT/echo context but never read back anywhere (grep-confirmed zero consumers) |
| §4.1: application service calls `tenant.FromContext(ctx)` | `internal/tenant/` doesn't exist; actor id is passed as an explicit `uuid.UUID` parameter into the service method |
| §4.1/§4.2: `pagination.Params` passed to repos; `XFilterQuery` DTOs with `SortBy`/`SortDir` | Each domain package defines its own local `ListParams{Page,Limit}` + `Filter`; `internal/pagination.Params` is imported only by HTTP handlers (11 of them), never by application or infra code; no sort/`XFilterQuery` DTOs exist |
| §5.2: sort-column whitelist map + `safeSortColumn()` per repository | Does not exist anywhere in the repo — aspirational |
| §7: DI wiring lives in `cmd/server/main.go` | Wiring lives in `internal/router/router.go`'s `Setup()`; `main.go` just calls it |
| §9: domain/app tests use `testify` + table-driven style | Plain `testing` package, individual `TestXxx` functions, hand-written mocks |
| §13: `internal/txmanager` + `Transactor` interface + `GORMTransactor` | Doesn't exist. Real mechanism is `internal/dbcontext` (`DBFromContext`/`WithTx`); `WithTx` itself has zero call sites today — treat as a hook for future orchestration, not something to wire up speculatively |
| §15: `EventPublisher` + domain events published after persistence | No domain-event infrastructure exists anywhere |
| §17: mocks generated with `mockgen`/`moq` | Repo uses `mockery` exclusively (see `references/interfaces.md`) |

## Relationship to `go-backend-reviewer`

`go-backend-reviewer` (global skill) is the audit checklist for reviewing a diff/PR against architecture,
security, CI, deployment, migrations, and observability rules. This skill is the authoring guide for producing
new code that already matches the codebase before review. Use this one while writing; use that one while
reviewing.
