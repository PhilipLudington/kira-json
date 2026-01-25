# JSON Library Production Readiness Plan

## Current Status

The library provides functional JSON parsing, serialization, and manipulation with 108 passing tests. Recent improvements leverage new compiler features (StringBuilder, std.char.from_i32, std.math.trunc_to_i64).

### What Works
- Core types: `Json`, `JsonField`, `JsonError` (structured error types)
- Type predicates: `is_null`, `is_bool`, `is_number`, `is_string`, `is_array`, `is_object`
- Value extraction: `as_string`, `as_number`, `as_int`, `as_bool`, `as_array`, `as_object`
- Accessors: `get_field`, `get_index`, `size`, `keys`, `values`
- Serialization: `stringify`, `stringify_pretty`, `stringify_pretty_with`
- Parsing: `parse` with line/column error tracking
- Transforms: `map_values`, `filter_fields`, `set_field`, `remove_field`, `merge`, `merge_deep`, `map_array`, `filter_array`
- Unicode escapes: `\uXXXX` parsing and serialization
- Control character escaping

---

## Phase 1: JSON Specification Compliance

### 1.1 UTF-16 Surrogate Pair Support ✓
- [x] Handle surrogate pairs in unicode escapes (`\uD800`-`\uDFFF`)
- [x] Parse `\uD83D\uDE00` as single emoji character
- [x] Add tests for emoji and extended Unicode
- [x] Lone surrogates replaced with U+FFFD (replacement character)

**Files:** `src/json.ki` (parse_unicode_escape_str, is_high_surrogate, is_low_surrogate, combine_surrogates, parse_4_hex_digits)

### 1.2 Number Handling Edge Cases
- [x] Reject leading zeros (e.g., `007` is invalid JSON)
- [x] Handle very large numbers (overflow to infinity - rejected)
- [x] Handle very small numbers (underflow to zero - allowed)
- [x] Reject `NaN` and `Infinity` in parsing
- [x] Test scientific notation edge cases (`1e999`, `1e-999`)

**Files:** `src/json.ki` (parse_number)

### 1.3 String Validation
- [x] Reject unescaped control characters (0x00-0x1F) in input
- [x] Validate UTF-8 sequences in input strings (handled by Kira's string type)
- [x] Handle bare `\` at end of string as error

**Files:** `src/json.ki` (parse_string_contents_builder, is_control_char_str)

### 1.4 Duplicate Key Handling ✓
- [x] Define behavior for duplicate keys (first-wins for `parse`, error for `parse_strict`)
- [x] Add `parse_strict` that rejects duplicates (`DuplicateKey` error)
- [x] Add `parse_strict_with_max_depth` for custom depth limits
- [x] Document the behavior (see tests for examples)

**Files:** `src/json.ki` (parse_object_fields, parse_strict, parse_strict_with_max_depth, DuplicateKey error variant)

---

## Phase 2: Performance Optimization

### 2.1 HashMap-Based Objects (Requires std.map)
- [ ] Create `JsonObjectMap` type using `std.map`
- [ ] O(1) field lookup instead of O(n) list traversal
- [ ] Preserve insertion order for serialization
- [ ] Benchmark before/after

**Files:** `src/json.ki` (new type, update accessors)

### 2.2 Parser String Building
- [x] Use StringBuilder in `parse_string_contents_str`
- [x] Currently uses recursive `std.string.concat` (O(n²)) - FIXED
- [ ] Benchmark improvement

**Files:** `src/json.ki` (parse_string_contents_str, parse_string_contents_builder)

### 2.3 Stack Safety ✓
- [x] Add depth limit parameter to parser (`parse_with_max_depth`)
- [x] Default reasonable limit (512 levels via `default_max_depth()`)
- [x] Return error on excessive nesting (`MaxDepthExceeded` error variant)
- [ ] Consider iterative parsing for deeply nested structures (optional, recursive works for 512)

**Files:** `src/json.ki` (parse_value, parse_array, parse_object)

---

## Phase 3: API Enhancements

### 3.1 Path-Based Access ✓
- [x] `get_path(json, "user.address.city")` → `Option[Json]`
- [x] `get_path(json, "users[0].name")` → `Option[Json]`
- [x] `set_path(json, "user.name", value)` → `Json`
- [x] `has_path(json, "user.name")` → `bool`
- [x] `remove_path(json, "user.name")` → `Json`
- [x] Path parsing with error handling (`PathError` type)

**Files:** `src/json.ki`

### 3.2 Equality and Comparison
- [x] `equals(a, b)` → `bool` (deep structural equality)
- [x] Handle object key order (should be order-independent)
- [x] Handle number comparison (floating point issues)

**Files:** `src/json.ki`

### 3.3 JSON Schema Validation
- [ ] Define schema type
- [ ] `validate(json, schema)` → `Result[void, ValidationError]`
- [ ] Support type constraints, required fields, patterns
- [ ] Collect all errors vs fail-fast mode

**New file:** `src/json_schema.ki`

### 3.4 Builder Pattern
- [ ] `JsonBuilder` type for fluent object construction
- [ ] `object().field("name", string("Alice")).field("age", number(30)).build()`
- [ ] Type-safe construction

**Files:** `src/json.ki`

### 3.5 Convenience Constructors
- [ ] `json_string(s)` → `Json` (alias for `JString(s)`)
- [ ] `json_number(n)` → `Json`
- [ ] `json_bool(b)` → `Json`
- [ ] `json_array(list)` → `Json`
- [ ] `json_object(fields)` → `Json`

**Files:** `src/json.ki`

---

## Phase 4: Error Handling Improvements

### 4.1 Structured Error Types ✓
- [x] Create error variant type with 7 variants:
  - `UnexpectedEof` - end of input with expected token and context
  - `UnexpectedToken` - wrong character with expected vs found
  - `UnterminatedString` - missing closing quote
  - `InvalidEscape` - bad escape sequence with reason
  - `ControlChar` - unescaped control character with code
  - `InvalidNumber` - invalid number with value and reason
  - `TrailingContent` - extra content after JSON value
- [x] Add `format_error(JsonError) -> string` for human-readable messages
- [x] Add `error_line(JsonError) -> i32` and `error_column(JsonError) -> i32` helpers
- [x] Update parser to use structured errors throughout
- [x] Add 11 tests for pattern matching on error variants

**Files:** `src/json.ki`, `tests/test_json.ki`

### 4.2 Error Context
- [ ] Include snippet of input around error location
- [ ] Show column indicator (e.g., `  ^--- here`)
- [ ] `format_error(input, error)` → `string`

**Files:** `src/json.ki`

---

## Phase 5: Testing & Validation

### 5.1 JSON Test Suite
- [ ] Run against [JSONTestSuite](https://github.com/nst/JSONTestSuite)
- [ ] Categorize: must accept, must reject, implementation-defined
- [ ] Document any deviations from spec

**New file:** `tests/test_json_suite.ki`

### 5.2 Property-Based Tests
- [ ] Roundtrip: `parse(stringify(json)) == json`
- [ ] Stringify determinism: same input → same output
- [ ] Parse consistency: valid JSON never crashes

**Files:** `tests/test_json.ki`

### 5.3 Edge Case Tests
- [ ] Empty input
- [ ] Whitespace-only input
- [ ] Maximum nesting depth
- [ ] Very long strings (>1MB)
- [ ] Very long arrays (>10000 elements)
- [ ] Unicode edge cases (BOM, surrogates, RTL)

**Files:** `tests/test_json.ki`

### 5.4 Fuzzing
- [ ] Create fuzz test harness
- [ ] Run against parser to find crashes
- [ ] Document any issues found

**New file:** `tests/fuzz_json.ki`

---

## Phase 6: Documentation

### 6.1 API Reference
- [ ] Document all public functions with examples
- [ ] Document error conditions
- [ ] Document performance characteristics

**New file:** `docs/api.md`

### 6.2 Usage Guide
- [ ] Getting started example
- [ ] Parsing examples
- [ ] Building JSON programmatically
- [ ] Transformation examples
- [ ] Error handling examples

**New file:** `docs/guide.md`

### 6.3 Performance Guide
- [ ] When to use HashMap objects
- [ ] Memory considerations
- [ ] Streaming large files

**New file:** `docs/performance.md`

---

## Priority Order

1. ~~**Phase 2.2** - Parser string building~~ ✓
2. ~~**Phase 1.2** - Number edge cases~~ ✓
3. ~~**Phase 1.3** - String validation~~ ✓
4. ~~**Phase 3.2** - Equality~~ ✓
5. ~~**Phase 4.1** - Structured errors~~ ✓
6. ~~**Phase 3.1** - Path access~~ ✓
7. ~~**Phase 2.3** - Stack safety~~ ✓
8. ~~**Phase 1.1** - Surrogate pairs (full Unicode)~~ ✓
9. ~~**Phase 1.4** - Duplicate key handling~~ ✓
10. **Phase 5.1** - Test suite (validation)
11. **Phase 2.1** - HashMap objects (if perf needed)

---

## Non-Goals (Out of Scope)

- Streaming/incremental parsing (requires different architecture)
- JSON5 or JSONC support (non-standard)
- Binary JSON formats (BSON, MessagePack)
- JSON-LD or JSON-API specifics
