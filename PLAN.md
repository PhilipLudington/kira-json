# JSON Parser Library for Kira - Implementation Plan

A pure functional JSON parsing, manipulation, and serialization library demonstrating Kira's ADTs, pattern matching, and higher-order functions.

## Phase 1: Core Types & Accessors

Create the foundation with type definitions and accessor functions.

- [x] Create `src/json.ki` with core type definitions
  - `Json` sum type (JNull, JBool, JNumber, JString, JArray, JObject)
  - `JsonField` record type for key-value pairs
  - `ParseError` record type with line/column info
  - `ParserState` record type for internal parser state
  - `ParseResult[T]` sum type for parse results

- [x] Implement type checking predicates
  - `is_null(json: Json) -> bool`
  - `is_bool(json: Json) -> bool`
  - `is_number(json: Json) -> bool`
  - `is_string(json: Json) -> bool`
  - `is_array(json: Json) -> bool`
  - `is_object(json: Json) -> bool`
  - `type_name(json: Json) -> string`

- [x] Implement value extractors
  - `as_string(json: Json) -> Option[string]`
  - `as_number(json: Json) -> Option[f64]`
  - `as_int(json: Json) -> Option[i64]`
  - `as_bool(json: Json) -> Option[bool]`
  - `as_array(json: Json) -> Option[List[Json]]`
  - `as_object(json: Json) -> Option[List[JsonField]]`

- [x] Implement field/index accessors
  - `get_field(json: Json, key: string) -> Option[Json]`
  - `get_index(json: Json, index: i32) -> Option[Json]`
  - `size(json: Json) -> Option[i32]`
  - `keys(json: Json) -> Option[List[string]]`
  - `values(json: Json) -> Option[List[Json]]`

- [x] Test accessors with manually constructed JSON values

## Phase 2: Serializer

Implement JSON to string conversion with proper escaping and formatting.

- [x] Implement string escape handling
  - Escape quotes, backslashes, newlines, tabs, carriage returns
  - Handle Unicode escapes if needed

- [x] Implement `stringify(json: Json) -> string`
  - Compact output with no extra whitespace
  - Handle all JSON types correctly

- [x] Implement `stringify_pretty(json: Json) -> string`
  - Pretty-printed with 2-space indentation
  - Proper newlines and formatting

- [x] Implement `stringify_pretty_with(json: Json, indent_size: i32) -> string`
  - Custom indentation size support

- [x] Test serializer with various JSON structures

## Phase 3: Parser

Implement recursive descent parser for JSON strings.

- [x] Implement parser state utilities
  - `init_state(input: string) -> ParserState`
  - `make_error(state: ParserState, message: string) -> ParseError`
  - `is_eof(state: ParserState) -> bool`
  - `peek(state: ParserState) -> Option[char]`
  - `advance(state: ParserState) -> ParserState`
  - `skip_whitespace(state: ParserState) -> ParserState`

- [x] Implement character utilities
  - `is_digit(c: char) -> bool`
  - `is_hex_digit(c: char) -> bool` (for unicode escapes)

- [x] Implement literal parsers
  - `parse_null(state: ParserState) -> ParseResult[Json]`
  - `parse_bool(state: ParserState) -> ParseResult[Json]`

- [x] Implement number parser
  - `parse_number(state: ParserState) -> ParseResult[Json]`
  - Handle integers, decimals, and exponent notation
  - Handle negative numbers

- [x] Implement string parser
  - `parse_string(state: ParserState) -> ParseResult[string]`
  - Handle escape sequences: `\"`, `\\`, `\/`, `\b`, `\f`, `\n`, `\r`, `\t`
  - Handle unicode escapes: `\uXXXX`

- [x] Implement compound parsers
  - `parse_array(state: ParserState) -> ParseResult[Json]`
  - `parse_object(state: ParserState) -> ParseResult[Json]`
  - Handle empty arrays/objects
  - Handle trailing commas (optional, per JSON spec)

- [x] Implement main parse dispatch
  - `parse_value(state: ParserState) -> ParseResult[Json]`
  - Dispatch based on first character
  - `parse(input: string) -> Result[Json, ParseError]` public API

- [x] Test parser with various valid JSON inputs

- [x] Test parser error handling with invalid inputs

## Phase 4: Transformers

Implement higher-order functions for JSON manipulation.

- [x] Implement value mapping
  - `map_values(json: Json, f: fn(Json) -> Json) -> Json`
  - Recursively apply function to all values

- [x] Implement field filtering
  - `filter_fields(json: Json, pred: fn(string, Json) -> bool) -> Json`
  - `filter_fields_deep(json: Json, pred: fn(string, Json) -> bool) -> Json`

- [x] Implement object operations
  - `set_field(json: Json, key: string, value: Json) -> Json`
  - `remove_field(json: Json, key: string) -> Json`

- [x] Implement merge operations
  - `merge(a: Json, b: Json) -> Json` - shallow merge
  - `merge_deep(a: Json, b: Json) -> Json` - recursive merge

- [x] Implement array operations
  - `map_array(json: Json, f: fn(Json) -> Json) -> Json`
  - `filter_array(json: Json, pred: fn(Json) -> bool) -> Json`

- [x] Test transformers with complex nested structures

## Phase 5: Documentation & Demo

Create comprehensive examples and documentation.

- [x] Create `examples/json_demo.ki` with usage examples
  - Parsing JSON from strings
  - Building JSON programmatically
  - Transforming JSON structures
  - Merging JSON objects
  - Pretty printing output

- [x] Add inline documentation comments to all public functions

- [x] Test full round-trip: parse -> transform -> stringify

## Notes

### Design Decisions
- All functions are pure (`fn`, not `effect fn`) except I/O demos
- Option types for safe access (no panics on missing keys)
- Result types for parsing with line/column error info
- Immutable transforms (all functions return new values)
- List-based objects for simplicity (O(n) lookup is acceptable)
- Explicit type parameters for all generic operations

### File Structure
```
src/
  json.ki          # Main library with all types and functions
examples/
  json_demo.ki     # Usage examples and demonstrations
tests/
  test_json.ki     # Comprehensive test suite (97 tests)
```
