# Domain layer — `internal/domain/<name>/`

Reference contexts (all NEW-layered, all verified by direct read): `well`, `field`, `sensortag`, `wellstring`,
`platform`, `reservoir`, `tagtype`, `unitofmeasurement`, `fieldplatform`, `monthlyproductiondata`.

## File layout

A NEW-layered domain package has exactly:

```
internal/domain/<name>/
  entity.go          # struct + New<Name>() factory + Update() mutator
  value_objects.go    # string-backed enums with ParseXxx(), sometimes value_object.go (singular)
  repository.go       # Repository interface + domain-local ListParams/Filter
  errors.go           # domainError + sentinel vars + init() RegisterCode
  service.go          # OPTIONAL — pure domain logic with no I/O (rare; e.g. vfmcalculated)
  *_test.go
```

No `handler.go`, `dto.go`, `mapper.go`, or repository *implementation* ever lives here — those belong to
`interfaces/http/handler`, `application/<name>svc`, and `infrastructure/persistence/postgres` respectively.

## Entity pattern — exported fields, not guarded getters

`internal/domain/well/entity.go:11-31`:

```go
type Well struct {
	ID              uuid.UUID
	WellID          int // integer surrogate key (ast_wells.well_id), used by VPS/VFM tables
	PlatformID      uuid.UUID
	FieldPlatformID uuid.UUID // FK to ast_field_platforms
	FieldID         uuid.UUID // denormalized, derived from field_platform.field_id
	UWI             string
	Name            string
	Code            *string
	WellType        *WellType
	Status          *WellStatus
	SpudDate        *time.Time
	Geometry        json.RawMessage
	DeleteReason    *string
	Version         int
	CreatedBy       uuid.UUID
	UpdatedBy       uuid.UUID
	CreatedAt       time.Time
	UpdatedAt       time.Time
	DeletedAt       *time.Time
}
```

Every field is exported. No unexported/guarded field, no getter method, anywhere in any NEW-layered context —
this directly contradicts `.github/copilot-instructions.md` §3.1a, which prescribes an unexported `status`
field + `Status()` getter specifically for `Well`. Do not add guarded fields to new entities; match what's here.

Construction is a factory returning `(*T, error)`:

```go
func NewWell(platformID, fieldPlatformID, fieldID uuid.UUID, uwi, name string, code *string,
	wellType *WellType, status *WellStatus, spudDate *time.Time, geometry json.RawMessage,
	createdBy uuid.UUID) (*Well, error)
```

Mutation is a single `Update(...)  error` method that validates every argument first and only assigns fields
after all validation passes — "the entity is not mutated on failure" (doc comment in `entity.go:84`). There is
no `SetXxx`/reconstitution-only setter pattern; infrastructure mappers build the struct directly with exported
fields (see `references/infrastructure.md`).

`Version int` and `CreatedBy`/`UpdatedBy uuid.UUID` are present on every entity that supports optimistic locking
and audit trail — include them on new aggregate roots that get updated. `TenantID` is **not** present on any
entity — do not add it unless multi-tenancy is being deliberately introduced as a cross-cutting feature (it
isn't today; see the SKILL.md contradictions table).

## Value objects — `ParseXxx` for every string-backed enum

`internal/domain/sensortag/value_objects.go` (full file, 39 lines):

```go
type SensorType string

const (
	SensorTypePressure    SensorType = "pressure"
	SensorTypeTemperature SensorType = "temperature"
	SensorTypeFlow        SensorType = "flow"
	SensorTypeChoke       SensorType = "choke"
	SensorTypeLevel       SensorType = "level"
	SensorTypeOther       SensorType = "other"
)

var validSensorTypes = map[SensorType]struct{}{ /* one entry per const */ }

// AllowedSensorTypes returns the slice of valid sensor type strings for error messages.
func AllowedSensorTypes() []string { return []string{"pressure", "temperature", "flow", "choke", "level", "other"} }

// ParseSensorType validates and returns a SensorType from a raw string.
func ParseSensorType(s string) (SensorType, error) {
	st := SensorType(s)
	if _, ok := validSensorTypes[st]; !ok {
		return "", ErrInvalidSensorType
	}
	return st, nil
}

func (st SensorType) String() string { return string(st) }
```

This exact shape (typed string const block + `valid<Enum>s` set + `AllowedXxx() []string` + `ParseXxx(s string)
(X, error)` + `String()`) repeats for every string-backed enum across `well` (`WellType`, `WellStatus`),
`wellstring`, `platform` (`ParsePlatformType`), `reservoir` (`ParseZone`), `field`, etc. Never cast a raw string
to the enum type directly (`SensorType(s)` outside `ParseXxx`) in application/infrastructure code — always call
`ParseXxx` so invalid values are rejected consistently.

## Repository interface — domain owns its own pagination, no tenant param

`internal/domain/well/repository.go` (full file):

```go
// ListParams carries pagination parameters for list queries.
// Defined here so the domain has no dependency on shared packages.
type ListParams struct {
	Page  int
	Limit int
}

func (p ListParams) Offset() int { return (p.Page - 1) * p.Limit }

type Filter struct {
	PlatformID      *uuid.UUID
	FieldID         *uuid.UUID
	FieldPlatformID *uuid.UUID
	Status          *WellStatus
	WellType        *WellType
	BBox            *[4]float64
}

type Repository interface {
	FindAll(ctx context.Context, filter Filter, params ListParams) ([]*Well, int64, error)
	FindByID(ctx context.Context, id uuid.UUID) (*Well, error)
	Create(ctx context.Context, w *Well) error
	Update(ctx context.Context, w *Well) error
	SoftDelete(ctx context.Context, id uuid.UUID, deletedBy uuid.UUID, reason string) error
	PlatformExists(ctx context.Context, platformID uuid.UUID) (bool, error)
	FindHierarchy(ctx context.Context) ([]RegionNode, error)
}
```

Two load-bearing facts, confirmed across every NEW-layered context (`field`, `sensortag`, `wellstring`,
`platform`, `reservoir`, `tagtype`, `unitofmeasurement`, `fieldplatform`, `wellstringreservoir`,
`monthlyproductiondata`):

1. **Pagination is domain-local**, never `internal/pagination.Params`. The comment on `ListParams` is explicit
   about why: "so the domain has no dependency on shared packages." `internal/pagination.Params` exists and is
   real, but it is consumed **only** by HTTP handlers (`pagination.FromQuery()` parses the query string into it,
   then the handler maps it into the domain's own `ListParams`). Never import `internal/pagination` from a
   domain or application package.
2. **No `tenantID` parameter anywhere.** `FindByID(ctx, id)`, not `FindByID(ctx, tenantID, id)`. `SoftDelete`
   takes `deletedBy` (an actor id, for audit) but that is not tenant scoping.

`Filter` expresses business predicates only, matching the intent (if not the tenant field) of
copilot-instructions §3.3. Return types for list-style read models (e.g. `well.RegionNode`/`FieldNode`/
`PlatformNode`/`WellNode` for the hierarchy endpoint) also live in `repository.go` next to the interface when
they're query-shaped rather than aggregate-shaped.

## Domain errors — this part of the instructions doc IS accurate

`internal/domain/well/errors.go` (full file):

```go
package well

import "edafy-asset-backend/internal/apperrors"

func init() {
	apperrors.RegisterCode("WELL_NOT_FOUND", 404)
	apperrors.RegisterCode("WELL_PLATFORM_NOT_FOUND", 422)
	apperrors.RegisterCode("WELL_PLATFORM_FIELD_MISMATCH", 422)
}

type domainError struct {
	code    string
	message string
}

func (e *domainError) Error() string     { return e.message }
func (e *domainError) ErrorCode() string { return e.code }

var (
	ErrNotFound              = &domainError{code: "WELL_NOT_FOUND", message: "well not found"}
	ErrPlatformNotFound      = &domainError{code: "WELL_PLATFORM_NOT_FOUND", message: "referenced platform does not exist"}
	ErrPlatformFieldMismatch = &domainError{code: "WELL_PLATFORM_FIELD_MISMATCH", message: "platform does not belong to the specified field"}
)
```

Reproduce this exactly for a new context: unexported `domainError{code, message}` implementing
`apperrors.DomainError` (`Error() string` + `ErrorCode() string`), package-level `ErrXxx` sentinels, HTTP status
registered via `apperrors.RegisterCode` inside `init()`. `apperrors.ResolveProblem` picks these up automatically
(see `references/infrastructure.md`).

## Domain events and domain services — mostly absent

No `internal/domain/*/event.go` file exists anywhere in the repo. There is no `EventPublisher` interface, no
`Publish(...)` call, no `WellDeactivated`-style event. Do not build one speculatively — copilot-instructions §15
is entirely aspirational here.

A handful of contexts (`vfmcalculated`, `unitofmeasurement`) have a `service.go` with pure computation (unit
conversion, calculated-field derivation) and no I/O — this matches copilot-instructions §3.4 (stateless, no
context.Context, no DB/HTTP calls). Add one only when genuine cross-entity logic doesn't belong on a single
aggregate; most contexts don't need it.

## `internal/core/models/` — legacy only

`internal/core/models/` (`base.go`, `user.go`, `tenant.go`, `entity.go`, `welltest.go`, `tank.go`, …) is the
pre-refactor GORM model package: exported structs with `gorm:"..."` tags, `BeforeCreate`/`BeforeUpdate` hooks, a
`Pagination` struct baked into `base.go`. It is imported by LEGACY flat domain packages
(`well_test/repository.go`, `well_test/service.go`, etc.) and by `internal/testutils/database.go` (which
`AutoMigrate`s `models.Tenant`, `models.User`, `models.TenantUser` for test fixtures — that part is still live
infrastructure, not dead code). No NEW-layered domain or infrastructure package imports `internal/core/models` —
each NEW-layered repository defines its own GORM model locally instead (see `references/infrastructure.md`).
Never import `internal/core/models` from a new domain/application package.
