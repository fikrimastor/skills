# Application layer — `internal/application/<name>svc/`

Reference: `wellsvc` (full read). Corroborated against `fieldsvc`, `sensortagsvc`, `platformsvc`,
`wellstringsvc`, `reservoirsvc`, `tagtypesvc`, `unitofmeasurementsvc`, and the four ingest services
(`productioningestsvc`, `welltestingestsvc`, `fieldsensoringestsvc`, `wellsensoringestsvc`).

## File layout

```
internal/application/<name>svc/
  service.go   # Service struct + NewService(repo...) + use-case methods
  dto.go       # Create/Update/Delete Command structs + Response struct(s)
  mapper.go    # exported To<Name>Response / To<Name>ResponseList
  ports.go     # OPTIONAL — locally-defined interfaces for external adapters (ingest services only)
  service_test.go
```

## Service method shape — CRUD contexts

`internal/application/wellsvc/service.go` (signatures, full file is ~295 lines):

```go
func NewService(repo well.Repository, fieldPlatformRepo fieldplatform.Repository) *Service

func (s *Service) Create(ctx context.Context, cmd CreateWellCommand, createdBy uuid.UUID) (*WellResponse, error)
func (s *Service) List(ctx context.Context, filter well.Filter, params well.ListParams) ([]WellResponse, int64, error)
func (s *Service) GetByID(ctx context.Context, id uuid.UUID) (*WellResponse, error)
func (s *Service) Update(ctx context.Context, id uuid.UUID, cmd UpdateWellCommand, updatedBy uuid.UUID) (*WellResponse, error)
func (s *Service) Delete(ctx context.Context, id uuid.UUID, cmd DeleteWellCommand, deletedBy uuid.UUID) error
```

This exact shape repeats in `fieldsvc`, `platformsvc`, `sensortagsvc`, `wellstringsvc`, `reservoirsvc`,
`wellstringreservoirsvc`, `tagtypesvc`. Three load-bearing, contradiction-worthy facts:

1. **`context.Context` is always first** — consistent with copilot-instructions §4.1.
2. **The actor is an explicit `uuid.UUID` parameter** (`createdBy`/`updatedBy`/`deletedBy`), never extracted
   from `ctx` inside the service. `internal/tenant/` does not exist; there is no `tenant.FromContext(ctx)` call
   anywhere in the repo (grep-confirmed). The handler resolves the actor via `actorFromContext(c echo.Context)`
   (`internal/interfaces/http/handler/auth_helpers.go`) and passes the `uuid.UUID` straight into the service
   call. Mirror this — do not add a context-extraction helper to a new service.
3. **`List` takes the domain's own `Filter` + `ListParams` directly** — there is no `XFilterQuery`
   application-layer DTO with pagination/sort fields (contra copilot-instructions §4.2). The handler builds
   `well.Filter{...}` and `well.ListParams{...}` from query params (using `internal/pagination.FromQuery()` only
   for the page/limit math) and passes them straight through.

## Non-CRUD service shapes

Not every service is CRUD. Match the shape to the use case, not to a template:

- **Read-only / dashboard services** (`vfmcalculatedsvc`, `vpsdashboardsvc`, `monthlyproductiondatasvc`,
  `fieldplatformsvc`): no actor params, methods named for the query they answer
  (`GetPlatformSummary`, `GetWellRateEstimation`, `ListDiagnosticPlotRows`, `FindFieldsByPlatformID`, …).
- **Data-ingestion / write-only services** (`vpscalculatedsvc`, `vpsqcedsvc`): `Insert`/`InsertBatch`/`List`/
  `GetByID`, no actor params, sometimes a `filter.Validate()` call before hitting the repository.
- **Event-driven ingest pipelines** (`productioningestsvc`, `welltestingestsvc`, `fieldsensoringestsvc`,
  `wellsensoringestsvc`): `UploadFile(ctx, cmd, io.Reader)`, `TriggerIngest(ctx, cmd)`, `GetBatch(ctx, id)`,
  `HandleBlobCreated(ctx, blobURL, contentLength, contentMD5)` — driven by Azure Blob Storage / Event Grid, not
  a human actor. These depend on locally-defined interfaces in a `ports.go` file rather than a domain
  repository alone:

  `internal/application/productioningestsvc/ports.go` (full file):

  ```go
  package productioningestsvc

  type UploadResult = wellsensoringestsvc.UploadResult
  type DagRun = wellsensoringestsvc.DagRun

  type BlobStore interface {
  	Upload(ctx context.Context, key, contentType string, r io.Reader, size int64) (UploadResult, error)
  	DeleteBlob(ctx context.Context, key string) error
  }

  type AirflowClient interface {
  	TriggerDagRun(ctx context.Context, dagID string, conf map[string]any) (DagRun, error)
  }
  ```

  The concrete implementations live in `internal/infrastructure/blobstore/azureblob/client.go` and
  `internal/infrastructure/airflow/client.go` — same inversion-of-control rule as a repository: the
  application layer defines the interface it needs, infrastructure implements it, `router.go` wires the
  concrete type in.

## DTOs — `dto.go`

Naming matches copilot-instructions §4.2 for commands/responses:

- `Create<Name>Command`, `Update<Name>Command`, `Delete<Name>Command` — struct tags include `json:"..."` and
  `validate:"..."` (`go-playground/validator/v10` tag syntax), e.g. `sensortagsvc.CreateSensorTagCommand` uses
  `validate:"required,uuid"`, `validate:"omitempty,oneof=..."`, `validate:"min=..."`.
- `<Name>Response` — output DTO.

**But `validate` tags are consumed only by the HTTP handler**, never by the application service. Grep-confirmed:
`go-playground/validator` is imported by 15 handler files and by zero files under `internal/application/`.
Inside a service method, validation is manual and value-object-driven:

- `uuid.Parse(...)` on incoming ID strings → `apperrors.NewValidationError(...)` on failure.
- Value-object parsing via the domain's `ParseXxx` functions (`well.ParseWellStatus`, `platform.ParsePlatformType`, …).
- `time.Parse("2006-01-02", ...)` for date strings.
- FK existence checks via a repository method like `repo.PlatformExists(ctx, id)` before calling `Create`.

There is **no `XFilterQuery` DTO** — do not invent one for a new context; pass the domain `Filter`/`ListParams`
straight from handler to service, as `well_handler.go` does.

## Mappers — `mapper.go`

```go
func ToWellResponse(w *well.Well) *WellResponse { /* reads exported fields directly */ }
func ToWellResponseList(wells []*well.Well) []WellResponse
```

Exported (capitalized), one function per response shape, reads domain entity fields directly (no getters to
call, since domain fields are exported — see `references/domain.md`).

## What's absent — don't build these speculatively

- **No tenant scoping.** Zero hits for `Tenant`/`tenant_id` anywhere in `internal/application/`. `tenant_id` is
  set on the JWT/echo context by `internal/interfaces/http/middleware/jwt.go` but never read back — it is
  currently auth/audit-context-only, not a query-scoping mechanism. Do not add a `tenantID` parameter to a new
  service method unless the task is explicitly "add multi-tenancy" (a cross-cutting change, not a
  per-context one).
- **No domain-event publishing.** Zero hits for `Publish(`/`EventPublisher`. The `HandleBlobCreated` methods on
  ingest services handle external Azure Event Grid webhooks, not an internal domain-event bus — don't conflate
  the two.
- **No cross-repo transaction orchestration from the application layer.** Zero hits for `dbcontext` under
  `internal/application/`. Each service method calls one repository write per use case; `dbcontext.WithTx` (see
  `references/infrastructure.md`) is defined but has zero callers anywhere in the repo today. If a new use case
  genuinely needs multi-repository atomicity, that's a deliberate infra addition to discuss, not something to
  wire up silently by copying the copilot-instructions.md `Transactor` example (which doesn't exist here).
- **Zero `gorm`/`echo`/`net/http` imports** anywhere under `internal/application/` — keep it that way.
