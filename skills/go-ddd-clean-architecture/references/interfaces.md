# Interfaces layer, composition root, and testing

## Composition root — `internal/router/router.go`, NOT `cmd/server/main.go`

`cmd/server/main.go` is a thin entrypoint. **All** repo → service → handler wiring happens inside
`internal/router/router.go`'s `Setup()` function — this contradicts copilot-instructions §7, which shows the
wiring living in `main.go`.

Real wiring for `well` (`internal/router/router.go:213-234`, alongside sibling contexts):

```go
// --- Clean Architecture: field bounded context ---
fieldRepo := postgres.NewFieldRepository(db)
fieldSvc := fieldsvc.NewService(fieldRepo)
fieldHandler := authHandler.NewFieldHandler(fieldSvc)

// --- Field-Platform junction repository (read-only in phase 1) ---
fieldPlatformRepo := postgres.NewFieldPlatformRepository(db)

// --- Clean Architecture: platform bounded context ---
platformRepo := postgres.NewPlatformRepository(db)
platformSvc := platformsvc.NewService(platformRepo, fieldPlatformRepo)
platformHandler := authHandler.NewPlatformHandler(platformSvc)

// --- Clean Architecture: well bounded context ---
wellRepo := postgres.NewWellRepository(db)
wellSvc := wellsvc.NewService(wellRepo, fieldPlatformRepo)
wellHandler := authHandler.NewWellHandler(wellSvc)
```

Followed later in the same function by `wellHandler.RegisterRoutes(protected)` where `protected` is the
JWT-guarded `*echo.Group`. Add a new bounded context the same way: one `// --- Clean Architecture: <name>
bounded context ---` comment block, `postgres.New<Name>Repository(db)` → `<name>svc.NewService(repo, ...)` →
`authHandler.New<Name>Handler(svc)`, then `RegisterRoutes` alongside the rest.

Legacy flat contexts are wired completely differently right next to these blocks — `x.NewHandler(db)`, passing
the raw `*gorm.DB` straight into the domain package (e.g. `areaHandler := area.NewHandler(db)`,
`wellTestHandler := well_test.NewHandler(db)`). That's the fastest visual tell for "this is legacy, don't
imitate it" when skimming `router.go`.

## Handler pattern — `internal/interfaces/http/handler/<name>_handler.go`

Reference: `well_handler.go`. Corroborated against `field_handler.go`, `sensor_tag_handler.go`.

```go
// <Name>Service is the local narrow interface for this handler — enables mock injection.
type WellService interface {
	Create(ctx context.Context, cmd wellsvc.CreateWellCommand, createdBy uuid.UUID) (*wellsvc.WellResponse, error)
	// ... one method per use case this handler actually calls
}

type WellHandler struct{ service WellService }

func NewWellHandler(service WellService) *WellHandler { return &WellHandler{service: service} }

func (h *WellHandler) RegisterRoutes(g *echo.Group) {
	wells := g.Group("/wells")
	wells.GET("", h.List)
	wells.GET("/:id", h.GetByID)
	wells.POST("", h.Create)
	wells.PUT("/:id", h.Update)
	wells.DELETE("/:id", h.Delete)
}
```

Every exported handler method is preceded by full swaggo/`swag` annotations — `@Summary`, `@Description`,
`@Tags`, `@Produce`/`@Accept`, one `@Param` per query/path/body parameter, `@Success`, one `@Failure` per
plausible status with `{object} apperrors.Problem`, `@Security BearerAuth`, `@Router`. Copy this block for every
new handler method; it drives the generated Swagger UI at `/swagger/index.html`.

### Error handling — three, and only three, ways to fail

```go
// 1. Input/format errors (bad path param, malformed UUID) — protocol concern, not business logic:
return echo.NewHTTPError(http.StatusBadRequest, "invalid well ID format")

// 2. Struct-tag validation failures on a Command DTO:
if err := h.validator.Struct(cmd); err != nil {
	return NewValidationProblem(extractFieldViolations(err))   // → 422 with errors[] array
}

// 3. Everything from the service (domain errors, CategoryErrors, unexpected failures):
resp, err := h.service.Create(c.Request().Context(), cmd, actorID)
if err != nil {
	return err   // ProblemErrorHandler resolves it — see below
}
```

**Never `c.JSON(http.StatusXxx, map[string]interface{}{...})` for an error path.** That pattern is confirmed
(~100+ grep hits) throughout the LEGACY flat handlers (`internal/domain/*/handler.go` — `area`, `entity`,
`production`, `admin_users`, etc.) and is exactly what NEW-layered handlers must not do.

`NewValidationProblem`/`extractFieldViolations` (`internal/interfaces/http/handler/validation.go`, full file):
converts `validator.ValidationErrors` → `[]apperrors.FieldViolation` (snake_case field names via a
PascalCase/camelCase → snake_case converter, human-readable per-tag messages for `required`/`oneof`/`min`/
`max`/`email`/`uuid`/`gt`/`gte`/`lt`/`lte`/`url`), then wraps them in
`apperrors.NewCategoryError(apperrors.CategoryValidation, msg).WithViolations(violations)`.

`ProblemErrorHandler(logger *zap.Logger, env string) echo.HTTPErrorHandler`
(`internal/interfaces/http/handler/problem_handler.go`) is registered once as `e.HTTPErrorHandler` and handles
every error, including echo's own (route-not-found, JWT-middleware 401/403) via `BuildProblem`: resolves an
`*apperrors.Problem` (via `apperrors.ResolveProblem` — see `references/infrastructure.md` for the 4-tier
priority), fills `Instance` from the request path and `TraceID` from the active OTel span, scrubs `Detail` for
5xx responses when `env == "production"`, logs 4xx as `Warn` / 5xx as `Error` with a stack field, writes
`application/problem+json`. `internal/interfaces/http/handler/errors.go`'s `WriteError` helper is deprecated —
never call it in new code.

## Auth context keys — resolved, not a bug

`internal/interfaces/http/middleware/jwt.go` sets **all** of these on the echo context on every authenticated
request (both the legacy-JWT and OIDC code paths): `claims`, `claims_type`, `"user_id"`, `"auth_user_id"`,
`"tenant_id"`, `"email"`, `"role"`, `"token"`.

Only `"user_id"` is ever read back, via
`internal/interfaces/http/handler/auth_helpers.go`:

```go
const contextKeyUserID = "user_id"

func actorFromContext(c echo.Context) (uuid.UUID, bool) {
	raw, ok := c.Get(contextKeyUserID).(string)
	if !ok || raw == "" { return uuid.Nil, false }
	id, err := uuid.Parse(raw)
	if err != nil { return uuid.Nil, false }
	return id, true
}
```

Every CRUD handler calls this to get the `createdBy`/`updatedBy`/`deletedBy` value it passes into the service.
`"auth_user_id"` and `"tenant_id"` are set but have zero consumers anywhere in `internal/interfaces` or
`internal/application` — not a bug, just unused-so-far context (`tenant_id` in particular is reserved for
future multi-tenancy work, see SKILL.md contradictions table). Use `actorFromContext` verbatim in new handlers;
don't invent a second helper.

## Middleware inventory — `internal/interfaces/http/middleware/`

| File | Purpose |
|---|---|
| `jwt.go` | Dual-mode JWT/OIDC auth middleware; sets the context keys above |
| `ratelimit.go` | IP-based rate limiting on the login endpoint |
| `webhook_secret.go` | Constant-time secret validation for Azure Event Grid webhook endpoints |

## Mock generation — `mockery`, not `mockgen`/`moq`

Every generated mock file starts with `// Code generated by mockery. DO NOT EDIT.` (e.g.
`internal/interfaces/http/handler/mocks/mock_well_test_ingest_service.go`,
`internal/application/welltestingestsvc/mocks/mock_airflow_client.go`,
`internal/domain/welltestingest/mocks/mock_repository.go`). This directly contradicts
copilot-instructions §17 (`mockgen`/`moq`). No `//go:generate` directives or `.mockery.yaml` were found in the
repo tree — mocks are regenerated via an explicit CLI invocation:

```
mockery --name=<InterfaceName> --dir=./internal/<layer>/<context>/ --output=./internal/<layer>/<context>/mocks/ --outpkg=mocks
```

## Testing conventions {#testing}

Three genuinely different patterns coexist by layer — match the layer, not a single global rule:

- **Domain** (`internal/domain/well/entity_test.go`): standard `testing` package only, individual `TestXxx`
  functions with `t.Fatalf`/`t.Errorf` guards. **No `testify`, no table-driven requirement** — contradicts
  copilot-instructions §9.
- **Application** (`internal/application/wellsvc/service_test.go`): same — standard `testing`, individual
  `TestXxx` functions, but with **hand-written mock structs** (`mockWellRepo`, `mockFieldPlatformRepo`)
  implementing the domain `Repository` interface directly in the test file — not `mockery`-generated, despite
  `mockery` being the generator used elsewhere (domain/welltestingest, application/*ingestsvc, and handler
  mocks). Follow whichever precedent already exists in the specific package being touched; for a brand-new
  `*svc` package, hand-written mocks matching `wellsvc`'s style are the safer default.
- **Handler** (`internal/interfaces/http/handler/well_handler_test.go`): hand-written `mockWellService`,
  `newEchoCtx(method, path, body)` + `serveTest(c, fn)` local helpers, `ProblemErrorHandler` registered on the
  test `echo.Echo` instance, standard `testing` package.
- **Infrastructure** (`internal/infrastructure/persistence/postgres/testmain_test.go`): the one place that
  **does** match copilot-instructions §9 — `testcontainers-go` with `postgres:16-alpine`, a single `TestMain`
  spins up one container for the whole `postgres_test` package, shared `testDB *gorm.DB` across tests.
- **`internal/testutils/`** (`config.go`, `database.go`, `server.go`) is a *separate*, older test-infra path:
  connects to a real Postgres via DSN (not testcontainers), `SetupTestDB(t)` + `AutoMigrate`s the legacy
  `internal/core/models` (`Tenant`, `User`, `TenantUser`), `SetupTestServer(t)` builds a full JWT-authenticated
  Echo server for integration-style tests, `GenerateTestToken(userID, tenantID, email, role)` for auth fixtures.
  Two DB-test strategies coexist in this repo; use `testcontainers-go` (matching `testmain_test.go`) for new
  repository tests, and `testutils.SetupTestServer` only when a test genuinely needs a full authenticated HTTP
  round-trip.
