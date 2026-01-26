# JSON Library Production Readiness Plan

## Current Status

The library provides functional JSON parsing, serialization, and manipulation with **375 passing tests** (291 in test_json.ki + 84 in test_json_suite.ki). Recent improvements leverage new compiler features (StringBuilder, std.char.from_i32, std.math.trunc_to_i64). Now includes JSON Schema validation with type constraints, required fields, numeric/string/array length constraints, enum values, and nested validation.

**Note:** Tests updated to use `std.string.length` as character count (not byte count) after compiler update.

**Note:** Kira stdlib API change - `std.string.chars`, `std.string.length`, and `std.string.substring` now return `Result` types. The json.ki library has been updated; test_json.ki needs updating for tests that use these functions directly.

### What Works
- Core types: `Json`, `JsonField`, `JsonError` (structured error types)
- **HashMap-based objects**: `JObject(HashMap)` with O(1) field lookup via `std.map`
- Type predicates: `is_null`, `is_bool`, `is_number`, `is_string`, `is_array`, `is_object`
- Value extraction: `as_string`, `as_number`, `as_int`, `as_bool`, `as_array`, `as_object`, `as_object_map`
- Accessors: `get_field` (O(1)), `get_index`, `size`, `keys`, `values`
- Serialization: `stringify`, `stringify_pretty`, `stringify_pretty_with`
- Parsing: `parse`, `parse_strict`, `parse_with_max_depth` with line/column error tracking
- Transforms: `map_values`, `filter_fields`, `set_field` (O(1)), `remove_field` (O(1)), `merge`, `merge_deep`, `map_array`, `filter_array`
- Unicode escapes: `\uXXXX` parsing and serialization (including surrogate pairs)
- Control character escaping
- Path-based access: `get_path`, `set_path`, `has_path`, `remove_path`
- Deep equality: `equals(a, b)`
- Convenience constructors: `json_null`, `json_bool`, `json_number`, `json_int`, `json_string`, `json_array`, `json_object`, `json_object_from_map`
- Builder pattern: `builder()`, `with_field`, `with_string`, `with_number`, `with_int`, `with_bool`, `with_null`, `with_array`, `with_object`, `build`
- Error context: `format_error_with_context(input, error)` shows source line with caret indicator

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

### 2.1 HashMap-Based Objects ✓
- [x] Changed `JObject(List[JsonField])` to `JObject(HashMap)` using `std.map`
- [x] O(1) field lookup via `std.map.get` (previously O(n) list traversal)
- [x] Preserved insertion order (std.map uses ordered hash map internally)
- [x] Added `json_object_from_map` and `as_object_map` for direct HashMap access
- [x] `as_object` returns `List[JsonField]` for backward compatibility
- [x] Parser uses first-wins semantics for duplicate keys (non-strict mode)
- [ ] Benchmark before/after

**Files:** `src/json.ki`, `tests/test_json.ki`, `examples/*.ki`

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

### 3.3 JSON Schema Validation ✓
- [x] Define schema types (`Schema`, `SchemaProperty`, `SchemaRef`, `SchemaError`)
- [x] `validate(json, schema)` → `Result[void, List[SchemaError]]`
- [x] Type constraints: null, boolean, number, integer, string, array, object
- [x] Required fields validation
- [x] Numeric range: minimum, maximum
- [x] String length: minLength, maxLength
- [x] Array length: minItems, maxItems
- [x] Array items schema validation
- [x] Nested object/array validation with path tracking
- [x] Enum value validation
- [x] Collect all errors mode (not fail-fast)
- [x] Error formatting: `format_schema_error`, `format_schema_errors`
- [x] Schema constructors: `schema_any`, `schema_type`, `schema_string`, `schema_number`, `schema_integer`, `schema_array`, `schema_object`, `schema_property`, `schema_ref`, `schema_enum`
- [x] 30 new tests for schema validation

**Files:** `src/json.ki` (integrated, not separate file), `tests/test_json.ki`

**Deferred:** `pattern` (requires regex), `additionalProperties`

### 3.4 Builder Pattern ✓
- [x] `JsonBuilder` type for fluent object construction
- [x] `builder().with_string("name", "Alice").with_int("age", 30).build()` pattern
- [x] Type-safe construction with `with_field`, `with_string`, `with_number`, `with_int`, `with_bool`, `with_null`, `with_array`, `with_object`

**Files:** `src/json.ki`

### 3.5 Convenience Constructors ✓
- [x] `json_string(s)` → `Json`
- [x] `json_number(n)` → `Json`
- [x] `json_int(n)` → `Json` (from i64)
- [x] `json_bool(b)` → `Json`
- [x] `json_null()` → `Json`
- [x] `json_array(list)` → `Json`
- [x] `json_array_empty()` → `Json`
- [x] `json_object(fields)` → `Json`
- [x] `json_object_empty()` → `Json`

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

### 4.2 Error Context ✓
- [x] Include snippet of input around error location
- [x] Show column indicator with `^`
- [x] `format_error_with_context(input, error)` → `string`

**Files:** `src/json.ki`

---

## Phase 5: Testing & Validation

### 5.1 JSON Test Suite ✓
- [x] Created comprehensive test suite (84 tests)
- [x] Categorized: must accept (43), must reject (36), implementation-defined (6)
- [x] Documented implementation choices (duplicate keys, lone surrogates, overflow handling)

**File:** `tests/test_json_suite.ki`

### 5.2 Property-Based Tests ✓
- [x] Roundtrip: `parse(stringify(json)) == json` (13 roundtrip tests)
- [x] Stringify determinism: same input → same output (2 determinism tests)
- [x] Parse consistency: valid JSON never crashes (2 consistency tests)

**Files:** `tests/test_json.ki`

### 5.3 Edge Case Tests ✓
- [x] Empty input
- [x] Whitespace-only input (4 tests: spaces, tabs, newlines, mixed)
- [x] Maximum nesting depth (4 tests: arrays, objects, mixed)
- [x] String edge cases (2 tests: special chars, unicode escapes)
- [x] Array edge cases (2 tests: multiple elements, mixed types)
- [x] Unicode escape handling (3 tests: basic, non-ASCII, CJK)
- Note: Very long strings/arrays limited by recursive interpreter implementation
- Note: Raw multi-byte Unicode literals have index bugs in parser

**Files:** `tests/test_json.ki`

### 5.4 Fuzzing ✓
- [x] Create fuzz test harness
- [x] Run against parser to find crashes
- [x] Document any issues found

**File:** `tests/fuzz_json.ki`

**Coverage:**
- 103 deterministic malformed inputs (numbers, strings, arrays, objects, edge cases)
- 150 random inputs (100 random strings + 50 JSON-like structures)
- Deep nesting up to 100 levels (arrays and objects)
- Boundary value testing (numeric limits, string escapes)
- Special byte sequences (null bytes, invalid UTF-8)

**Issues Found and Fixed:**
- Fixed `escape_string` to handle new `std.string.chars` Result API
- Parser gracefully handles all tested malformed inputs (returns errors, no crashes)

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
10. ~~**Phase 3.4** - Builder pattern~~ ✓
11. ~~**Phase 3.5** - Convenience constructors~~ ✓
12. ~~**Phase 4.2** - Error context~~ ✓
13. ~~**Phase 5.1** - Test suite (validation)~~ ✓
14. ~~**Phase 5.2** - Property-based tests~~ ✓
15. ~~**Phase 5.3** - Edge case tests~~ ✓
16. ~~**Phase 2.1** - HashMap objects~~ ✓
17. ~~**Phase 3.3** - JSON Schema validation~~ ✓
18. ~~**Phase 5.4** - Fuzzing~~ ✓
19. **Phase 6** - Documentation

---

## Non-Goals (Out of Scope)

- Streaming/incremental parsing (requires different architecture)
- JSON5 or JSONC support (non-standard)
- Binary JSON formats (BSON, MessagePack)
- JSON-LD or JSON-API specifics
