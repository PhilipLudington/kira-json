# JSON Library Production Readiness Plan

## Current Status

**All phases complete.** The library provides functional JSON parsing, serialization, and manipulation with **375+ passing tests** across multiple test files. The codebase has been modularized into 8 logical submodules under `src/json/` for better maintainability.

**Module Structure:**
- `src.json.types` - Core types, predicates, accessors
- `src.json.parser` - Parsing functions
- `src.json.serializer` - Serialization functions
- `src.json.transform` - Transformation functions
- `src.json.path` - Path-based access
- `src.json.builder` - Builder pattern and constructors
- `src.json.schema` - Schema validation
- `src.json.equality` - Deep equality comparison

**Note:** Tests updated to use `std.string.length` as character count (not byte count) after compiler update.

**Note:** Kira stdlib API change - `std.string.chars`, `std.string.length`, and `std.string.substring` now return `Result` types. The json library has been updated accordingly.

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
- [x] Default reasonable limit (128 levels via `default_max_depth()`)
- [x] Return error on excessive nesting (`MaxDepthExceeded` error variant)
- [ ] Consider iterative parsing for deeply nested structures (optional, recursive works for 128)

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

## Phase 6: Documentation ✓

### 6.1 API Reference ✓
- [x] Document all public functions with examples
- [x] Document error conditions
- [x] Document performance characteristics

**File:** `docs/api.md`

### 6.2 Usage Guide ✓
- [x] Getting started example
- [x] Parsing examples
- [x] Building JSON programmatically
- [x] Transformation examples
- [x] Error handling examples
- [x] Schema validation examples

**File:** `docs/guide.md`

### 6.3 Performance Guide ✓
- [x] When to use HashMap objects
- [x] Memory considerations
- [x] Best practices for large JSON handling

**File:** `docs/performance.md`

---

## Phase 7: Code Quality Improvements

Based on comprehensive code quality review (score: 8.9/10). These improvements address architectural refinements, security hardening, and maintainability enhancements.

### 7.1 Resource Limits for Untrusted Input ✓

Add configurable limits to prevent resource exhaustion attacks from malicious JSON input.

**New Types:**
```kira
pub type ParseLimits = {
    max_depth: i32,
    max_string_length: Option[i32],
    max_array_items: Option[i32],
    max_object_fields: Option[i32]
}
```

**New Functions:**
- [x] `parse_with_limits(string, ParseLimits) -> Result[Json, JsonError]`
- [x] `parse_strict_with_limits(string, ParseLimits) -> Result[Json, JsonError]`
- [x] `default_limits() -> ParseLimits` - sensible defaults for untrusted input
- [x] `unlimited_limits() -> ParseLimits` - no limits (use with caution)

**New Error Variants:**
- [x] `StringTooLong { length: i32, max_length: i32, line: i32, col: i32 }`
- [x] `TooManyArrayItems { count: i32, max_items: i32, line: i32, col: i32 }`
- [x] `TooManyObjectFields { count: i32, max_fields: i32, line: i32, col: i32 }`

**Implementation:**
- [x] Add limit checks in `parse_string_contents_builder`
- [x] Add limit checks in `parse_array_elements` (via `parse_array_elements_counted`)
- [x] Add limit checks in `parse_object_fields` (via `parse_object_fields_map_counted`)
- [x] Update `ParserState` to include limits
- [x] Add tests for each limit type (16 new tests)
- [x] Update docs/api.md with new functions
- [x] Update docs/guide.md with security recommendations

**Files:** `src/json.ki`, `tests/test_json.ki`, `docs/api.md`, `docs/guide.md`

### 7.2 Refactor Surrogate Pair Handling ✓

Extract surrogate pair logic from `parse_unicode_escape_str` to reduce nesting depth from 5 levels to 2-3.

**Refactoring Complete:**
- [x] Extract `code_point_to_string: fn(i32) -> string` - converts code point to string or replacement char
- [x] Extract `try_parse_low_surrogate: fn(ParserState) -> Option[(i32, ParserState)]` - attempts to parse `\uXXXX` low surrogate
- [x] Extract `parse_surrogate_pair: fn(i32, ParserState) -> ParseResult[string]` - handles high surrogate with optional low pair
- [x] Simplify `parse_unicode_escape_str` to delegate to helpers (reduced from ~80 lines to ~15 lines)
- [x] Maintain identical behavior (lone surrogates → U+FFFD)
- [x] All 375 existing tests pass (291 in test_json.ki + 84 in test_json_suite.ki)

**Files:** `src/json.ki`

### 7.3 Modularize Source Code ✓

Split `src/json.ki` (3,626 lines) into logical submodules for better maintainability.

**Final Structure:**
```
src/
├── json.ki              (documentation only, ~80 lines - no re-exports, Kira doesn't support them)
├── json/
│   ├── types.ki         (Json, JsonField, JsonError, type predicates, accessors, ~600 lines)
│   ├── parser.ki        (parse functions, ParserState, ~1200 lines)
│   ├── serializer.ki    (stringify functions, ~200 lines)
│   ├── schema.ki        (Schema types, validate, ~500 lines)
│   ├── transform.ki     (map, filter, merge, set/remove, ~300 lines)
│   ├── path.ki          (get_path, set_path, PathError, ~350 lines)
│   ├── builder.ki       (JsonBuilder, with_* functions, convenience constructors, ~250 lines)
│   └── equality.ki      (equals function, ~80 lines)
```

**Implementation Steps:**
- [x] Create `src/json/` directory
- [x] Extract types to `src/json/types.ki`
- [x] Extract parser to `src/json/parser.ki`
- [x] Extract serializer to `src/json/serializer.ki`
- [x] Extract schema validation to `src/json/schema.ki`
- [x] Extract transformations to `src/json/transform.ki`
- [x] Extract path operations to `src/json/path.ki`
- [x] Extract builder to `src/json/builder.ki`
- [x] Extract equality to `src/json/equality.ki`
- [x] Update `src/json.ki` to documentation-only (Kira doesn't support re-exports)
- [x] Update all imports in tests and examples
- [x] Verify all tests pass after refactoring

**Import Pattern (Kira doesn't support re-exports):**
```kira
import src.json.types.{ Json, JNull, JString, ... }
import src.json.parser.{ parse, parse_strict }
import src.json.serializer.{ stringify, stringify_pretty }
import src.json.transform.{ set_field, remove_field, merge }
import src.json.path.{ get_path, set_path }
import src.json.builder.{ json_object, builder, with_string, build }
import src.json.schema.{ validate, schema_object, schema_type }
import src.json.equality.{ equals }
```

**Files:** `src/json/*.ki`, `tests/*.ki`, `examples/*.ki`, `benches/*.ki`

### 7.4 Performance Benchmarks ✓

Created benchmark suite to track performance characteristics and verify complexity claims.

**Benchmark Categories:**
- [x] Parse time vs. input size (100 bytes, 1KB, 10KB)
- [x] Stringify time vs. structure complexity
- [x] Field access: HashMap O(1) verification (confirmed: 10 fields = 1000 fields = 0.007ms)
- [x] Path access time vs. depth (linear: ~0.15ms per level)
- [x] Array operations vs. length (O(n) confirmed)
- [x] Schema validation time vs. complexity

**Implementation:**
- [x] Create `benches/bench_json.ki`
- [x] Use `std.time.now()` for timing
- [x] Generate test data of various sizes
- [x] Run benchmarks and document baseline
- [x] Add benchmark results to `docs/performance.md`

**Key Findings:**
- HashMap O(1) verified: identical access times regardless of object size
- Array O(n) confirmed: times scale with position/length
- Parser has significant interpreter overhead (~5ms for 100 bytes)
- Serialization is fast (~0.4ms for small objects)

**Files:** `benches/bench_json.ki`, `docs/performance.md`

### 7.5 Standardize Internal Naming ✓

Established consistent naming conventions for internal helper functions.

**Naming Conventions:**
| Suffix | Usage | Example |
|--------|-------|---------|
| `_str` | String-based variant | `is_digit_str`, `peek_str` |
| `_acc` | Accumulator recursion | `list_reverse_acc`, `format_schema_errors_acc` |
| `_impl` | Internal implementation | `parse_impl`, `validate_impl` |
| `_to_builder` | StringBuilder output | `escape_chars_to_builder` |

**Implementation:**
- [x] Audit all internal functions for naming consistency
- [x] Rename inconsistent functions (backwards compatible - internal only)
  - `parse_internal` → `parse_impl`
  - `parse_internal_with_limits` → `parse_impl_with_limits`
  - `list_length_json` → `length_json`
  - `list_append_schema_errors` → `append_errors`
  - Removed duplicate `list_has_key` (identical to `has_key`)
- [x] Document conventions in code comments (added at top of json.ki)

**Files:** `src/json.ki`

### 7.6 Add Performance Documentation in Code ✓

Add inline documentation for operations with non-obvious performance characteristics.

**Locations to Document:**
- [x] `get_index` - O(n) array traversal warning
- [x] `size` for arrays - O(n) list length
- [x] `as_object` - O(n) HashMap to List conversion
- [x] `map_array`, `filter_array` - O(n) with new list allocation
- [x] Path operations with array indices - O(n) per index segment (`get_path`, `set_path`, `has_path`, `remove_path`)

**Example Format:**
```kira
/// Gets an element from a JSON array by index.
///
/// **Performance:** O(n) where n is the index. Arrays are linked lists,
/// requiring traversal to reach the target index. For frequent random
/// access, consider using objects with string indices.
pub let get_index: fn(Json, i32) -> Option[Json]
```

**Files:** `src/json.ki`

---

## Phase 7 Priority Order

1. **7.1** - Resource limits (security hardening)
2. **7.2** - Surrogate pair refactor (complexity reduction)
3. **7.4** - Performance benchmarks (measurement before optimization)
4. **7.6** - Performance docs in code (quick win)
5. **7.5** - Naming standardization (maintainability)
6. **7.3** - Modularization (large refactor, do last)

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
19. ~~**Phase 6** - Documentation~~ ✓
20. ~~**Phase 7.1** - Resource limits for untrusted input~~ ✓
21. ~~**Phase 7.2** - Refactor surrogate pair handling~~ ✓
22. ~~**Phase 7.4** - Performance benchmarks~~ ✓
23. ~~**Phase 7.6** - Performance documentation in code~~ ✓
24. ~~**Phase 7.5** - Standardize internal naming~~ ✓
25. ~~**Phase 7.3** - Modularize source code~~ ✓

---

## Non-Goals (Out of Scope)

- Streaming/incremental parsing (requires different architecture)
- JSON5 or JSONC support (non-standard)
- Binary JSON formats (BSON, MessagePack)
- JSON-LD or JSON-API specifics
