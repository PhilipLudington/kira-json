# JSON Parser Library for Kira

A pure functional JSON parsing, manipulation, and serialization library demonstrating Kira's ADTs, pattern matching, and higher-order functions.

## Overview

This library provides:
- **Core JSON ADT** - Type-safe representation of JSON values
- **Parser** - Recursive descent parser from string to ADT
- **Accessors** - Safe field/index access with Option types
- **Transformers** - Higher-order functions for JSON manipulation
- **Serializer** - Convert ADT back to JSON string

## 1. Core Types

```kira
// JSON value - the 6 JSON types as a sum type
type Json =
    | JNull
    | JBool(bool)
    | JNumber(f64)
    | JString(string)
    | JArray(List[Json])
    | JObject(List[JsonField])

// Object field (key-value pair)
type JsonField = {
    key: string,
    value: Json
}

// Parse error with location
type ParseError = {
    message: string,
    line: i32,
    column: i32
}

// Internal parser state
type ParserState = {
    input: string,
    pos: i32,
    line: i32,
    col: i32
}

// Internal parse result
type ParseResult[T] =
    | ParseOk(T, ParserState)
    | ParseErr(ParseError)
```

## 2. Public API

### Parsing

```kira
/// Parse a JSON string into a Json value
/// Returns Result[Json, ParseError] for error handling
pub fn parse(input: string) -> Result[Json, ParseError]
```

### Accessors

```kira
/// Get a field from a JSON object by key
pub fn get_field(json: Json, key: string) -> Option[Json]

/// Get an element from a JSON array by index
pub fn get_index(json: Json, index: i32) -> Option[Json]

/// Extract string from JSON, returns None if not a string
pub fn as_string(json: Json) -> Option[string]

/// Extract number from JSON, returns None if not a number
pub fn as_number(json: Json) -> Option[f64]

/// Extract integer from JSON number, returns None if not integer
pub fn as_int(json: Json) -> Option[i64]

/// Extract boolean from JSON, returns None if not a boolean
pub fn as_bool(json: Json) -> Option[bool]

/// Extract array from JSON, returns None if not an array
pub fn as_array(json: Json) -> Option[List[Json]]

/// Extract object fields from JSON, returns None if not an object
pub fn as_object(json: Json) -> Option[List[JsonField]]

/// Type checking predicates
pub fn is_null(json: Json) -> bool
pub fn is_bool(json: Json) -> bool
pub fn is_number(json: Json) -> bool
pub fn is_string(json: Json) -> bool
pub fn is_array(json: Json) -> bool
pub fn is_object(json: Json) -> bool

/// Get the type name of a JSON value
pub fn type_name(json: Json) -> string

/// Get the number of elements in an array or fields in an object
pub fn size(json: Json) -> Option[i32]

/// Get all keys from a JSON object
pub fn keys(json: Json) -> Option[List[string]]

/// Get all values from a JSON object
pub fn values(json: Json) -> Option[List[Json]]
```

### Transformers

```kira
/// Map a function over all values in a JSON structure (recursive)
pub fn map_values(json: Json, f: fn(Json) -> Json) -> Json

/// Filter fields in a JSON object (non-recursive)
pub fn filter_fields(json: Json, pred: fn(string, Json) -> bool) -> Json

/// Filter fields recursively in all nested objects
pub fn filter_fields_deep(json: Json, pred: fn(string, Json) -> bool) -> Json

/// Merge two JSON objects (second object's fields override first)
pub fn merge(a: Json, b: Json) -> Json

/// Deep merge two JSON objects (recursively merges nested objects)
pub fn merge_deep(a: Json, b: Json) -> Json

/// Set a field in a JSON object (returns new object)
pub fn set_field(json: Json, key: string, value: Json) -> Json

/// Remove a field from a JSON object
pub fn remove_field(json: Json, key: string) -> Json

/// Map over array elements
pub fn map_array(json: Json, f: fn(Json) -> Json) -> Json

/// Filter array elements
pub fn filter_array(json: Json, pred: fn(Json) -> bool) -> Json
```

### Serialization

```kira
/// Convert JSON to compact string (no extra whitespace)
pub fn stringify(json: Json) -> string

/// Convert JSON to pretty-printed string with 2-space indentation
pub fn stringify_pretty(json: Json) -> string

/// Convert JSON to pretty-printed string with custom indent size
pub fn stringify_pretty_with(json: Json, indent_size: i32) -> string
```

## 3. Parser Implementation

The parser uses a recursive descent approach following patterns from `examples/simple_parser.ki`.

### Character Utilities

```kira
/// Create initial parser state from input string
fn init_state(input: string) -> ParserState

/// Create a parse error at current position
fn make_error(state: ParserState, message: string) -> ParseError

/// Check if we've reached end of input
fn is_eof(state: ParserState) -> bool

/// Peek at current character without consuming
fn peek(state: ParserState) -> Option[char]

/// Advance parser state by one character
fn advance(state: ParserState) -> ParserState

/// Skip whitespace (space, tab, newline, carriage return)
fn skip_whitespace(state: ParserState) -> ParserState

/// Check if character is a digit
fn is_digit(c: char) -> bool
```

### Literal Parsers

```kira
/// Parse null literal
fn parse_null(state: ParserState) -> ParseResult[Json]

/// Parse boolean literal (true/false)
fn parse_bool(state: ParserState) -> ParseResult[Json]

/// Parse JSON number (integer, decimal, exponent)
fn parse_number(state: ParserState) -> ParseResult[Json]

/// Parse JSON string with escape sequences
fn parse_string(state: ParserState) -> ParseResult[string]
```

### Compound Parsers

```kira
/// Parse JSON array: [ value, value, ... ]
fn parse_array(state: ParserState) -> ParseResult[Json]

/// Parse JSON object: { "key": value, ... }
fn parse_object(state: ParserState) -> ParseResult[Json]

/// Parse any JSON value (dispatches based on first character)
fn parse_value(state: ParserState) -> ParseResult[Json]
```

### Dispatch Logic

The `parse_value` function examines the first non-whitespace character:
- `n` → parse_null
- `t` or `f` → parse_bool
- `"` → parse_string
- `[` → parse_array
- `{` → parse_object
- `-` or digit → parse_number

## 4. Example Usage

### Parsing JSON

```kira
effect fn parse_example() -> void {
    let json_str: string = "{\"name\": \"Alice\", \"age\": 30, \"active\": true}"

    match parse(json_str) {
        Ok(json) => {
            std.io.println("Parsed successfully!")

            // Access fields
            match get_field(json, "name") {
                Some(name_json) => {
                    match as_string(name_json) {
                        Some(name) => {
                            std.io.println("Name: " + name)
                        }
                        None => {
                            std.io.println("Name is not a string")
                        }
                    }
                }
                None => {
                    std.io.println("No name field")
                }
            }
        }
        Err(e) => {
            std.io.println("Parse error at line " + to_string(e.line) +
                         ", column " + to_string(e.column) + ": " + e.message)
        }
    }
}
```

### Building JSON Programmatically

```kira
fn build_user() -> Json {
    let user: Json = JObject(
        Cons(JsonField { key: "id", value: JNumber(1.0) },
        Cons(JsonField { key: "name", value: JString("Bob") },
        Cons(JsonField { key: "emails", value: JArray(
            Cons(JString("bob@example.com"),
            Cons(JString("bob.smith@work.com"), Nil))
        ) }, Nil)))
    )
    return user
}
```

### Transforming JSON

```kira
fn transform_example() -> void {
    let original: Json = JObject(
        Cons(JsonField { key: "price", value: JNumber(10.0) },
        Cons(JsonField { key: "quantity", value: JNumber(5.0) },
        Cons(JsonField { key: "name", value: JString("Widget") }, Nil)))
    )

    // Double all numbers
    let doubled: Json = map_values(
        original,
        fn(j: Json) -> Json {
            var result: Json = j
            match j {
                JNumber(n) => {
                    result = JNumber(n * 2.0)
                }
                _ => { }
            }
            return result
        }
    )

    // Filter to only numeric fields
    let numbers_only: Json = filter_fields(
        original,
        fn(key: string, value: Json) -> bool {
            var result: bool = false
            match value {
                JNumber(_) => { result = true }
                _ => { }
            }
            return result
        }
    )
}
```

### Merging JSON Objects

```kira
fn merge_example() -> Json {
    let defaults: Json = JObject(
        Cons(JsonField { key: "theme", value: JString("light") },
        Cons(JsonField { key: "fontSize", value: JNumber(14.0) },
        Cons(JsonField { key: "language", value: JString("en") }, Nil)))
    )

    let user_prefs: Json = JObject(
        Cons(JsonField { key: "theme", value: JString("dark") },
        Cons(JsonField { key: "showSidebar", value: JBool(true) }, Nil))
    )

    // User preferences override defaults
    let merged: Json = merge(defaults, user_prefs)
    return merged
    // Result: { theme: "dark", fontSize: 14, language: "en", showSidebar: true }
}
```

### Pretty Printing

```kira
effect fn print_example() -> void {
    let json: Json = build_user()

    // Compact output
    std.io.println(stringify(json))
    // {"id":1,"name":"Bob","emails":["bob@example.com","bob.smith@work.com"]}

    // Pretty output
    std.io.println(stringify_pretty(json))
    // {
    //   "id": 1,
    //   "name": "Bob",
    //   "emails": [
    //     "bob@example.com",
    //     "bob.smith@work.com"
    //   ]
    // }
}
```

## 5. Implementation Plan

### Phase 1: Core Types & Accessors
1. Create `examples/json.ki` with type definitions
2. Implement all accessor functions (type checks, field access)
3. Test by building JSON values manually

### Phase 2: Serializer
1. Implement `stringify` with escape handling
2. Implement `stringify_pretty` with indentation
3. Test round-trip with manual JSON values

### Phase 3: Parser
1. Implement character utilities and parser state
2. Implement literal parsers (null, bool, number, string)
3. Implement compound parsers (array, object)
4. Test with various JSON inputs

### Phase 4: Transformers
1. Implement `map_values`, `filter_fields`
2. Implement `merge`, `merge_deep`
3. Implement array operations

### Phase 5: Documentation & Demo
1. Create `examples/json_demo.ki` with comprehensive examples
2. Update documentation

## 6. Design Decisions

### Pure Functions Only
All parsing and transformation functions are pure (`fn`, not `effect fn`). Only the demo's I/O operations use effects.

### Option Types for Safe Access
All accessors return `Option[T]` rather than panicking on missing keys or type mismatches. This follows Kira's explicit error handling philosophy.

### Result Types for Parsing
The parser returns `Result[Json, ParseError]` with detailed error information including line and column numbers.

### Immutable Transforms
All transformation functions return new JSON values rather than mutating. This matches Kira's functional design.

### List-Based Objects
Objects are represented as `List[JsonField]` rather than a hash map. This is simpler and matches Kira's standard library patterns. For most JSON use cases, O(n) field lookup is acceptable.

### Explicit Type Parameters
Higher-order functions use explicit type parameters (e.g., `std.list.map[Json, Json](...)`) following Kira's no-inference policy.

## 7. Future Extensions

- **JSONPath support** - Query JSON with path expressions like `$.store.book[0].title`
- **Schema validation** - Validate JSON against a schema definition
- **Streaming parser** - Parse large files without loading entirely into memory
- **Custom serialization hooks** - User-defined type serialization
