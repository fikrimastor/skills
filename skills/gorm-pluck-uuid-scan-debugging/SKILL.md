---
name: gorm-pluck-uuid-scan-debugging
description: "Diagnose and fix GORM Pluck() failures when the destination is google/uuid.UUID — 'unsupported data type: uuid: Table not set' or 'converting driver.Value type string ... to a uint8'. Use when a Go+GORM repository method scanning a UUID column via Pluck panics/errors with either signature."
---

# GORM Pluck + google/uuid.UUID scan failures

Two distinct, stacked GORM behaviors cause this — they present as different error messages and must both be fixed.

## Signature 1: `unsupported data type: 00000000-0000-0000-0000-000000000000: Table not set, please set it like: db.Model(&user) or db.Table("users")`

**Cause:** the query chain never called `.Model(&SomeModel{})` or `.Table("...")` before `.Pluck(...)`. With no table bound, GORM's `Pluck` falls back to `Statement.Parse(dest)` to infer a table from the *destination* type. `uuid.UUID` is `type UUID [16]byte` — not a struct — so `schema.Parse` fails with `ErrUnsupportedDataType`, and the error prints the (usually still-zero) destination value, which is why the zero UUID shows up in the message even though the real bug has nothing to do with a zero ID.

**Fix:** chain `.Model(&YourGormModel{})` (or `.Table("table_name")`) before `.Pluck(...)`, matching every other query in the same repository file.

```go
// BROKEN — no table bound
err := db.WithContext(ctx).Distinct("field_id").Where("field_code = ?", code).Pluck("field_id", &fieldID).Error

// FIXED
err := db.WithContext(ctx).Model(&SomeModel{}).Distinct("field_id").Where("field_code = ?", code).Pluck("field_id", &fieldID).Error
```

## Signature 2: `sql: Scan error on column index 0, name "...": converting driver.Value type string ("c83bd066-...") to a uint8: invalid syntax`

This appears *after* fixing signature 1 — it was unreachable before because the query never got as far as executing SQL.

**Cause:** `Pluck`'s destination MUST be a slice (`*[]T`), never a bare scalar (`*T`) for anything outside GORM's small built-in fast-path list (`*int`, `*string`, `*bool`, `*time.Time`, `*sql.NullXxx`, etc. — see `gorm.io/gorm/scan.go`'s `Scan` function, the first `switch dest := db.Statement.Dest.(type)` case). `uuid.UUID` isn't in that list, so GORM falls into its generic reflection-based scan path. There it inspects `reflect.Kind()` of the dereferenced destination type: `uuid.UUID` is `[16]byte`, so `Kind() == reflect.Array`. GORM's generic scanner treats *any* Array/Slice-kind destination as "one row per element" — it takes `.Elem()` of the array type (→ `byte`/`uint8`) as the per-row type, and tries to scan the entire UUID string into a single `uint8`, which fails.

This is true **even though `*uuid.UUID` implements `sql.Scanner`** — GORM's Array/Slice dispatch happens before it would ever consult that interface on the whole value.

**Fix:** Pluck into a slice of UUIDs, then read `[0]`. This matches GORM's actual documented contract for `Pluck` ("scan into a slice") and is exactly how existing working `Pluck` calls into `[]string` in the same codebase behave — a slice-of-scalar Pluck dest works because iterating per-row and appending one scalar each time is the intended code path.

```go
// BROKEN — scalar dest, even with Model() set
var fieldID uuid.UUID
err := db.Model(&SomeModel{}).Distinct("field_id").Where(...).Limit(1).
    Pluck("field_id", &fieldID).Error

// FIXED — slice dest
var fieldIDs []uuid.UUID
err := db.Model(&SomeModel{}).Distinct("field_id").Where(...).Limit(1).
    Pluck("field_id", &fieldIDs).Error
if err != nil { return nil, err }
if len(fieldIDs) == 0 || fieldIDs[0] == uuid.Nil {
    return nil, ErrNotFound // Pluck never raises ErrRecordNotFound on 0 rows
}
return &fieldIDs[0], nil
```

## Diagnostic recipe

1. Grep the repository file for `Pluck(` calls where the destination variable was declared as a bare `uuid.UUID` (not `[]uuid.UUID`) — `var \w+ uuid\.UUID` followed by a `Pluck(` within the same function.
2. Confirm whether `.Model(...)`/`.Table(...)` appears anywhere in that call chain — if not, that's signature-1's root cause; add it first.
3. Re-run and check whether signature 2 now appears (it was masked until signature 1 was fixed) — if so, switch the destination to a slice.
4. Verify against a real DB with real data, not just mocks/fakes — this bug class is invisible to unit tests using fake repository interfaces (the interface signature `(*uuid.UUID, error)` doesn't change; only the internal Pluck destination type does), so only an integration-level check against a real Postgres exercises it.
