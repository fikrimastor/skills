# Infrastructure layer — persistence, transactions, errors, config, external adapters

## Repository implementation — `internal/infrastructure/persistence/postgres/<name>_repository.go`

Reference: `well_repository.go` (full file, 375 lines). Corroborated against `field_repository.go`,
`reservoir_repository.go`, `sensor_tag_repository.go`, `tag_type_repository.go`,
`unit_of_measurement_repository.go`.

```go
package postgres

// GORM model lives in the SAME file as the repository, not a shared models package.
type WellModel struct { /* gorm-tagged fields */ }

func (WellModel) TableName() string { return "ast_wells" }

type wellRepository struct{ db *gorm.DB }

// Constructor returns the DOMAIN interface type, not the concrete struct.
func NewWellRepository(db *gorm.DB) well.Repository {
	return &wellRepository{db: db}
}

func (r *wellRepository) FindByID(ctx context.Context, id uuid.UUID) (*well.Well, error) {
	db := dbcontext.DBFromContext(ctx, r.db)

	var m WellModel
	err := db.First(&m, "id = ?", id).Error
	if err != nil {
		if errors.Is(err, gorm.ErrRecordNotFound) {
			return nil, well.ErrNotFound
		}
		return nil, err
	}
	return toDomainWell(&m)
}

func (r *wellRepository) Create(ctx context.Context, w *well.Well) error {
	db := dbcontext.DBFromContext(ctx, r.db)
	m := toWellModel(w)
	return db.Create(&m).Error
}
```

Non-negotiable, confirmed universally (20/20 repository files check `dbcontext.DBFromContext` first —
`vps_dashboard_repository.go` is the sole exception, using `r.db.WithContext(ctx)` directly; treat that as a gap
in existing code, not a pattern to copy):

1. **`db := dbcontext.DBFromContext(ctx, r.db)` is the first line of every method.** This is the real
   (and only) transaction-propagation mechanism —
   `internal/dbcontext/dbcontext.go` (full file):

   ```go
   package dbcontext

   type txKey struct{}

   func DBFromContext(ctx context.Context, fallback *gorm.DB) *gorm.DB {
   	if tx, ok := ctx.Value(txKey{}).(*gorm.DB); ok && tx != nil {
   		return tx
   	}
   	return fallback.WithContext(ctx)
   }

   func WithTx(ctx context.Context, tx *gorm.DB) context.Context {
   	return context.WithValue(ctx, txKey{}, tx)
   }
   ```

   There is **no `Transactor` interface, no `GORMTransactor`, no `internal/txmanager` package** — that part of
   `.github/copilot-instructions.md` §13 is fictional. `WithTx` itself has zero call sites in the repo today
   (nothing currently begins a tx and injects it) — `DBFromContext` still matters because it's what every
   repository calls, and it transparently falls back to `r.db.WithContext(ctx)` when no tx is present.

2. **GORM model lives in the same file as its repository**, not a shared `models` package — this is a
   deliberate departure from copilot-instructions §5.1's separate-file suggestion; follow the actual file
   layout (one `<name>_repository.go` per aggregate, model + repo + converters together).

3. **Converters**: `toXModel(x *domain.X) XModel` (domain → persistence, no error — construction from an
   already-valid entity can't fail) and `toDomainX(m *XModel) (*domain.X, error)` (persistence → domain, MUST
   return an error and MUST NOT discard one). `well_repository.go`'s doc comment on `toDomainWell` states the
   rule explicitly: *"Never calls NewWell — avoids re-running invariants and ID generation."* Build the struct
   with exported fields directly instead. **Not every existing repository follows the error-return half of this
   rule** — `field_repository.go`'s `toDomainField` returns `*Field` with no error and silently discards
   `ParseZone`/`ParseStatus` failures. That is a pre-existing bug, not a pattern — new converters must return
   `(*X, error)` and propagate every `ParseXxx` failure.

4. **No sort-column whitelist exists anywhere** (`sortColumns` map / `safeSortColumn()` from
   copilot-instructions §5.1–5.2 is aspirational — zero hits repo-wide). If a new repository needs
   user-controlled `ORDER BY`, don't silently skip whitelisting just because no precedent exists — un-sanitized
   sort input is a real SQL-injection-adjacent risk regardless of what prior code did.

5. Soft delete uses GORM's `gorm.DeletedAt` field plus an explicit `DeleteReason`/`DeletedBy` pair set via a
   dedicated `SoftDelete(ctx, id, deletedBy, reason)` repository method — not a generic `Delete`.

## `apperrors` — the real RFC 7807 implementation

Three files: `internal/apperrors/errors.go`, `categories.go`, `problem.go`. This part matches
copilot-instructions §8 closely — safe to trust that section, verified against source.

**`Problem` struct** (`problem.go`, full file):

```go
type Problem struct {
	Type     string           `json:"type"`
	Title    string           `json:"title"`
	Status   int              `json:"status"`
	Detail   string           `json:"detail,omitempty"`
	Instance string           `json:"instance,omitempty"`
	TraceID  string           `json:"trace_id,omitempty"`
	Code     string           `json:"code,omitempty"`
	Errors   []FieldViolation `json:"errors,omitempty"`
}

type FieldViolation struct {
	Field   string `json:"field"`
	Message string `json:"message"`
}
```

**`CategoryError` constructors** (`categories.go`) — use these from handlers/services for anything that isn't a
registered domain error:

```go
apperrors.NewUnauthorizedError(msg string)   // 401
apperrors.NewForbiddenError(msg string)      // 403
apperrors.NewNotFoundError(msg string)       // 404
apperrors.NewConflictError(msg string)       // 409
apperrors.NewUnprocessableError(msg string)  // 422
apperrors.NewBadRequestError(msg string)     // 400
apperrors.NewInternalError(msg string)       // 500
```

Plus `apperrors.NewCategoryError(cat, msg)` for a custom category, and `.WithViolations([]FieldViolation)` to
attach field-level errors (used by `NewValidationProblem`, see `references/interfaces.md`).

**`ResolveProblem(err error) *Problem`** (`errors.go:116-172`) — 4-tier resolution priority, in order:

1. `*CategoryError` (via `errors.As`) — category-driven status/title/code/violations.
2. `DomainError` (via `errors.As`) — looks up the HTTP status registered by the domain package's
   `init(){ apperrors.RegisterCode(code, status) }`; falls back to 500 if the code was never registered.
3. `*pgconn.PgError` (via `errors.As`) — maps Postgres SQLSTATE codes (e.g. `23505` unique-violation → 409,
   `23503` FK-violation → 422) via `pgErrorToResponse`.
4. Generic fallback — 500, `Code: "INTERNAL_ERROR"`, `Detail: err.Error()` (this raw detail is scrubbed for 5xx
   in production by `ProblemErrorHandler`, not here — see `references/interfaces.md`).

`RegisterCode(code string, status int)` is what every domain package's `errors.go` `init()` calls — this is how
a domain-defined `ErrXxx` sentinel ends up mapped to the right HTTP status without the domain importing
`net/http`.

`apperrors.NewValidationError(msg string) error` also exists as a shared, code-generic validation error
(`ErrorCode() == "VALIDATION_ERROR"`) for one-off input validation (e.g. bad UUID string) that doesn't warrant a
domain-specific error code — used inside application services.

## `internal/pagination/pagination.go`

`Params{Page, Limit}` + `Offset()` + `FromQuery(c echo.Context) Params`-style helper. Imported by **11 handler
files only** — never by a repository or application service (see `references/domain.md`/`application.md` for
why: domains define their own local `ListParams`). Use it exactly the way `well_handler.go` does: parse query
params with it in the handler, then translate into the target domain's `ListParams`/`Filter` before calling the
service.

## Config, auth, external adapters — one-liners

- `internal/config/config.go`: `Load() (*Config, error)` — `godotenv.Load` + `getEnv` helpers, returns an error
  instead of panicking (no `Must`-style startup panic pattern here, despite copilot-instructions §7 suggesting
  one — `router.Setup()` calls `Load()` and handles the error itself).
- `internal/auth/jwt.go`: `JWTManager.GenerateToken(userID, tenantID uuid.UUID, email, role string) (string,
  error)`; `JWTClaims{UserID, TenantID, Email, Role}` — `tenantID` IS embedded in the token (for potential
  future use / audit) even though nothing downstream currently reads it back out of the request context to
  scope a query.
- `internal/infrastructure/blobstore/azureblob/client.go`: Azure Blob Storage adapter implementing the ingest
  services' `BlobStore` port (`Upload`, `DeleteBlob`, `EnsureContainer`).
- `internal/infrastructure/airflow/client.go`: Airflow REST API adapter implementing the ingest services'
  `AirflowClient` port (`TriggerDagRun`).

Both adapters follow the same inversion pattern as repositories: the interface (`BlobStore`/`AirflowClient`) is
defined in the application layer's `ports.go` (see `references/application.md`); the concrete implementation
lives in infrastructure and gets wired into the service constructor in `router.go`.

## Migrations

`internal/database/migrations/*.sql` — numbered, zero-padded 6-digit prefix + short description + `.up.sql`/
`.down.sql` pair, e.g. `000047_vps_provenance_revert_varchar.{up,down}.sql`. Peripheral to this skill; see the
global `go-backend-reviewer` skill's migration-review phase for the full rule set (idempotency, `ast_` table
prefix, rollback requirements).
