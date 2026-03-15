# kira-json — Typed Deserialization Plan

## Overview

Add typed deserialization to kira-json: convert `Json` values into user-defined Kira types (product types and sum types) with clear error reporting. This bridges the gap between the untyped `Json` ADT and application-level types like `User`, `Todo`, etc.

Reference: DESIGN.md, src/json/types.ki
Current status: Phase 3 complete. Phase 4 next.

## Design Constraints

Kira has no runtime reflection, macros, or code generation. Typed deserialization must be explicit — users write (or compose) deserializer functions per type. The library's job is to make this concise, safe, and consistent.

The approach: a `decode` module providing field extraction helpers that return `Result[T, DecodeError]` and compose with `?` for clean error propagation.

```kira
// Goal: what user code looks like
import json.decode.{required_string, required_int, optional_bool}

let decode_user: fn(Json) -> Result[User, DecodeError] =
    fn(json: Json) -> Result[User, DecodeError] {
        let name: string = required_string(json, "name")?
        let age: i64 = required_int(json, "age")?
        let active: bool = optional_bool(json, "active") ?? true
        return Ok(User { name: name, age: age, active: active })
    }
```

---

## Phase 0: DecodeError Type and Field Extractors ✅
**Status:** Complete (2026-03-15)

**Goal:** Core decode module with error type and required/optional field helpers for all JSON primitive types.
**Estimated Effort:** 1–2 days

### Deliverables
- `src/json/decode.ki` — new module
- `DecodeError` sum type with field context
- `required_*` helpers for string, number (f64), int (i64), bool
- `optional_*` helpers returning `Option[T]`
- `format_decode_error` for human-readable messages

### Tasks
- [x] Define `DecodeError` sum type (completed 2026-03-15):
  - `MissingField { field: string, context: string }`
  - `TypeMismatch { field: string, expected: string, actual: string, context: string }`
  - `InvalidValue { field: string, message: string, context: string }`
- [x] Implement required field extractors (completed 2026-03-15):
  - `required_string(Json, string) -> Result[string, DecodeError]`
  - `required_number(Json, string) -> Result[f64, DecodeError]`
  - `required_int(Json, string) -> Result[i64, DecodeError]`
  - `required_bool(Json, string) -> Result[bool, DecodeError]`
  - `required_field(Json, string) -> Result[Json, DecodeError]` (raw Json)
- [x] Implement optional field extractors (completed 2026-03-15):
  - `optional_string(Json, string) -> Option[string]`
  - `optional_number(Json, string) -> Option[f64]`
  - `optional_int(Json, string) -> Option[i64]`
  - `optional_bool(Json, string) -> Option[bool]`
  - `optional_field(Json, string) -> Option[Json]`
- [x] Implement `format_decode_error(DecodeError) -> string` (completed 2026-03-15)
- [x] Write tests: `tests/test_decode.ki` — 53 tests (completed 2026-03-15)
  - Happy path for each extractor
  - Missing field returns MissingField error
  - Wrong type returns TypeMismatch error with actual type name
  - Optional returns None for missing/wrong-type fields

### Testing Strategy
One test file (`test_decode.ki`) covering all extractors with happy path, missing field, and type mismatch cases. Minimum 30 tests.

---

## Phase 1: Nested Object and Array Decoders ✅
**Status:** Complete (2026-03-15)

**Goal:** Decode nested objects and arrays of decoded values.
**Estimated Effort:** 1–2 days

### Deliverables
- Nested object decoding via non-generic field extractors + user composition
- Array decoding via field extractors + user-defined typed iterators
- Error context chaining (e.g., `"users[2].address.city"`)

### Design Note
Kira does not support exporting generic functions across module boundaries (type parameters
are not resolved on import). The API was adapted: instead of generic `required_object[T]`,
the module exports non-generic building blocks (`required_object_field`, `required_array_field`,
`with_context`, `index_context`) that users compose with their own typed decoders. This
delivers the same functionality with slightly more verbose user code.

### Tasks
- [x] Implement nested object field extractors (completed 2026-03-15):
  - `required_object_field(Json, string) -> Result[Json, DecodeError]` — validates field is an object
  - `optional_object_field(Json, string) -> Option[Json]`
- [x] Implement array field extractors (completed 2026-03-15):
  - `required_array_field(Json, string) -> Result[List[Json], DecodeError]` — validates field is an array
  - `optional_array_field(Json, string) -> Option[List[Json]]`
- [x] Add context chaining to DecodeError (completed 2026-03-15):
  - `with_context(DecodeError, string) -> DecodeError` — prepends path segment
  - `index_context(i32) -> string` — formats array index as `"[N]"`
  - Composition pattern: user wraps decoder errors with `with_context(e, field_name)`
- [x] Write tests: 28 Phase 1 tests (completed 2026-03-15)
  - Nested objects, arrays of primitives, arrays of objects, error path reporting
  - Full path: `"team[1].address"` for deeply nested errors

### Testing Strategy
Test nested User-with-Address pattern, array of Users, error messages showing full path. Minimum 20 tests. Achieved 28 tests.

### Phase 1 Readiness Gate
Before Phase 2, these must be true:
- [x] All Phase 0 and 1 tests pass (81 total)
- [x] Error context correctly reports nested paths like `"users[0].email"`

---

## Phase 2: Convenience Decoders and Coercions ✅
**Status:** Complete (2026-03-15)

**Goal:** Common patterns: nullable fields, default values, enum strings, numeric coercions.
**Estimated Effort:** 1–2 days

### Deliverables
- Nullable field support (`null` → `None`, value → `Some(T)`)
- Default value support
- String-to-enum mapping (via `required_string_in`)
- Numeric coercion helpers (f64 → i32 with range checking)

### Design Notes
- `nullable_object` adapted to non-generic `nullable_object_field` (same pattern as Phase 1)
- `required_enum` adapted to `required_string_in(Json, string, List[string])` — validates string is in allowed set, user maps to enum type
- `required_f32` removed: Kira's `f32` type exists but has no runtime f64→f32 conversion

### Tasks
- [x] Implement nullable decoders (completed 2026-03-15):
  - `nullable_string(Json, string) -> Result[Option[string], DecodeError]`
  - `nullable_int(Json, string) -> Result[Option[i64], DecodeError]`
  - `nullable_number(Json, string) -> Result[Option[f64], DecodeError]`
  - `nullable_bool(Json, string) -> Result[Option[bool], DecodeError]`
  - `nullable_object_field(Json, string) -> Result[Option[Json], DecodeError]`
- [x] Implement default-value decoders (completed 2026-03-15):
  - `string_or(Json, string, string) -> Result[string, DecodeError]`
  - `int_or(Json, string, i64) -> Result[i64, DecodeError]`
  - `bool_or(Json, string, bool) -> Result[bool, DecodeError]`
  - `number_or(Json, string, f64) -> Result[f64, DecodeError]`
- [x] Implement string-to-enum mapping (completed 2026-03-15):
  - `required_string_in(Json, string, List[string]) -> Result[string, DecodeError]`
- [x] Implement safe numeric coercion (completed 2026-03-15):
  - `required_i32(Json, string) -> Result[i32, DecodeError]` — error if out of i32 range
- [x] Write tests: 37 Phase 2 tests (completed 2026-03-15)

### Testing Strategy
Test null vs missing vs present for nullable fields. Test defaults. Test enum mapping with valid and invalid strings. Test numeric overflow detection. Minimum 25 tests. Achieved 37 tests.

---

## Phase 3: Serialization (Encode) ✅
**Status:** Complete (2026-03-15)

**Goal:** Symmetric encode support — convert Kira types back to `Json` values.
**Estimated Effort:** 1 day

### Deliverables
- `src/json/encode.ki` — new module
- Primitive encoders and object/array builders that mirror the decode API
- Bonus: `collect_fields` helper for building objects with optional fields

### Tasks
- [x] Implement primitive encoders (completed 2026-03-15):
  - `encode_string(string) -> Json`
  - `encode_int(i64) -> Json`
  - `encode_number(f64) -> Json`
  - `encode_bool(bool) -> Json`
  - `encode_null() -> Json`
- [x] Implement composite encoders (completed 2026-03-15):
  - `encode_object(List[JsonField]) -> Json`
  - `encode_field(string, Json) -> JsonField`
  - `encode_array(List[Json]) -> Json`
- [x] Implement optional encoders (completed 2026-03-15):
  - `encode_optional(Option[Json]) -> Json` — Some(v) → v, None → JNull
  - `encode_optional_field(string, Option[Json]) -> Option[JsonField]` — None omits the field
  - `collect_fields(List[Option[JsonField]]) -> List[JsonField]` — filters None entries
- [x] Write tests: 47 Phase 3 tests (completed 2026-03-15)
  - 12 primitive encoder tests, 9 composite encoder tests, 7 optional encoder tests, 19 roundtrip tests

### Testing Strategy
Roundtrip tests: encode a Kira type, stringify, parse, decode, assert equality. Minimum 15 tests. Achieved 19 roundtrip tests (47 total).

### Phase 3 Readiness Gate
Before Phase 4, these must be true:
- [x] Encode → stringify → parse → decode roundtrip works for nested types
- [x] All 165 tests across decode/encode pass (118 decode + 47 encode)

---

## Phase 4: Integration with kira-http Examples

**Goal:** Update kira-http examples to use the real decode/encode API instead of pseudo-code.
**Estimated Effort:** 0.5 days

### Tasks
- [ ] Update `kira-http/examples/api_client.ki` to use `json.decode.*` for response parsing
- [ ] Update `kira-http/examples/rest_api.ki` to use `json.encode.*` for request/response bodies
- [ ] Verify both examples pass `kira check`

### Testing Strategy
Both examples compile with `kira check`. Manual smoke test of the API patterns.

---

## Risk Register

| Risk | Impact | Likelihood | Mitigation |
|------|--------|------------|------------|
| Kira `?` operator doesn't work with custom error types | High | Low | DecodeError is a standard sum type; `?` works with any `Result[T, E]` |
| Closure capture in array decoders hits compiler bugs | Medium | Medium | v0.4.0 fixed closure capture; test early with array decoder |
| Generic type parameters on decoder functions cause segfaults | Medium | Medium | Avoid importing generic `get` from http.client (known crash); use explicit type annotations |
| Performance of deeply nested decoding | Low | Low | Kira's recursive pattern matching is efficient; JSON depth rarely exceeds 10 |

## Timeline

Phase 0 (field extractors) → Phase 1 (nesting/arrays) → Phase 2 (convenience) → Phase 3 (encode) → Phase 4 (kira-http integration).

Phases 0–1 are the core deliverable. Phases 2–3 add polish. Phase 4 closes the loop with kira-http.
