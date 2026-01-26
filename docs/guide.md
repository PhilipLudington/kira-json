# JSON Library Usage Guide

A practical guide to using the Kira JSON library for parsing, building, transforming, and validating JSON.

## Table of Contents

- [Getting Started](#getting-started)
- [Parsing JSON](#parsing-json)
- [Building JSON](#building-json)
- [Accessing Values](#accessing-values)
- [Transforming JSON](#transforming-json)
- [Path-Based Access](#path-based-access)
- [Error Handling](#error-handling)
- [Schema Validation](#schema-validation)
- [Security: Parsing Untrusted Input](#security-parsing-untrusted-input)

---

## Getting Started

Import the library in your Kira module:

```kira
import src.json.{
    Json, JNull, JBool, JNumber, JString, JArray, JObject,
    JsonField, JsonError, JsonBuilder,
    parse, parse_strict, stringify, stringify_pretty,
    get_field, get_path, set_field, set_path,
    as_string, as_number, as_int, as_bool, as_array,
    is_null, is_string, is_object, type_name,
    json_string, json_number, json_int, json_bool, json_null,
    json_array, json_object, json_object_empty,
    builder, with_string, with_int, with_bool, with_array, build,
    format_error, format_error_with_context,
    equals, merge, merge_deep
}
import std.list.{ List, Cons, Nil }
```

### Minimal Example

```kira
effect fn main() -> IO[void] {
    // Parse JSON
    let input: string = "{\"name\": \"Alice\", \"age\": 30}"
    match parse(input) {
        Ok(json) => {
            // Access a field
            match get_field(json, "name") {
                Some(name_val) => {
                    match as_string(name_val) {
                        Some(name) => { std.io.println(name) }  // "Alice"
                        None => { }
                    }
                }
                None => { }
            }

            // Serialize back
            std.io.println(stringify(json))
        }
        Err(e) => {
            std.io.println(format_error(e))
        }
    }
    return
}
```

---

## Parsing JSON

### Basic Parsing

The `parse` function handles most JSON parsing needs:

```kira
let result: Result[Json, JsonError] = parse("{\"key\": \"value\"}")
match result {
    Ok(json) => {
        // Successfully parsed
        std.io.println(stringify(json))
    }
    Err(e) => {
        // Parse error with line/column info
        std.io.println(format_error(e))
    }
}
```

### Strict Parsing

Use `parse_strict` to reject duplicate keys:

```kira
// This fails with DuplicateKey error
let result: Result[Json, JsonError] = parse_strict("{\"a\": 1, \"a\": 2}")
match result {
    Ok(_) => { }
    Err(DuplicateKey { key: k, line: l, col: c }) => {
        std.io.println("Duplicate key: " + k)
    }
    Err(e) => {
        std.io.println(format_error(e))
    }
}
```

### Parsing with Depth Limits

Protect against deeply nested JSON (e.g., from untrusted sources):

```kira
// Limit nesting to 50 levels
let result: Result[Json, JsonError] = parse_with_max_depth(untrusted_json, 50)
match result {
    Err(MaxDepthExceeded { depth: d, max_depth: m, line: _, col: _ }) => {
        std.io.println("Nesting too deep")
    }
    _ => { }
}
```

### Handling All JSON Types

```kira
let json: Json = parse(input) ?? JNull
match json {
    JNull => { std.io.println("null") }
    JBool(b) => {
        if b { std.io.println("true") }
        else { std.io.println("false") }
    }
    JNumber(n) => { std.io.println(std.float.to_string(n)) }
    JString(s) => { std.io.println(s) }
    JArray(items) => { std.io.println("array with items") }
    JObject(_) => { std.io.println("object") }
}
```

---

## Building JSON

### Using Constructors

Build JSON values directly with constructor functions:

```kira
// Primitives
let null_val: Json = json_null()
let bool_val: Json = json_bool(true)
let num_val: Json = json_number(3.14)
let int_val: Json = json_int(42)
let str_val: Json = json_string("hello")

// Arrays
let arr: Json = json_array(Cons(json_int(1), Cons(json_int(2), Cons(json_int(3), Nil))))
let empty_arr: Json = json_array_empty()

// Objects
let person: Json = json_object(Cons(
    JsonField { key: "name", value: json_string("Alice") },
    Cons(
        JsonField { key: "age", value: json_int(30) },
        Nil
    )
))
```

### Using the Builder Pattern

The builder pattern provides a fluent API for object construction:

```kira
let person: Json = build(
    with_bool(
        with_int(
            with_string(
                builder(),
                "name", "Alice"),
            "age", 30),
        "active", true)
)
// {"name": "Alice", "age": 30, "active": true}
```

### Nested Objects with Builder

```kira
let address_builder: JsonBuilder =
    with_string(
        with_string(
            builder(),
            "city", "Seattle"),
        "country", "USA")

let person: Json = build(
    with_object(
        with_string(
            builder(),
            "name", "Alice"),
        "address", address_builder)
)
// {"name": "Alice", "address": {"city": "Seattle", "country": "USA"}}
```

### Arrays in Builder

```kira
let tags: List[Json] = Cons(json_string("admin"), Cons(json_string("user"), Nil))
let user: Json = build(
    with_array(
        with_string(
            builder(),
            "name", "Alice"),
        "roles", tags)
)
// {"name": "Alice", "roles": ["admin", "user"]}
```

---

## Accessing Values

### Field Access

```kira
let json: Json = parse("{\"user\": {\"name\": \"Alice\"}}") ?? JNull

// Direct field access (O(1))
match get_field(json, "user") {
    Some(user) => {
        match get_field(user, "name") {
            Some(name_val) => {
                match as_string(name_val) {
                    Some(name) => { std.io.println(name) }
                    None => { }
                }
            }
            None => { }
        }
    }
    None => { }
}
```

### Array Access

```kira
let json: Json = parse("[1, 2, 3]") ?? JNull

match get_index(json, 0) {
    Some(first) => {
        match as_int(first) {
            Some(n) => { std.io.println(std.int.to_string(n)) }  // 1
            None => { }
        }
    }
    None => { }
}
```

### Type Checking

```kira
if is_object(json) {
    // Safe to use get_field
}

if is_array(json) {
    // Safe to use get_index
}

// Get type as string
let t: string = type_name(json)  // "object", "array", "string", etc.
```

### Getting Object Keys and Values

```kira
match keys(json) {
    Some(key_list) => {
        // Iterate over keys
    }
    None => { }  // Not an object
}

match size(json) {
    Some(n) => { std.io.println("Size: " + std.int.to_string(n)) }
    None => { }  // Not an array or object
}
```

---

## Transforming JSON

### Setting Fields

```kira
let json: Json = parse("{\"x\": 1}") ?? JNull
let updated: Json = set_field(json, "y", json_int(2))
// {"x": 1, "y": 2}

// Overwrite existing field
let modified: Json = set_field(json, "x", json_int(10))
// {"x": 10}
```

### Removing Fields

```kira
let json: Json = parse("{\"a\": 1, \"b\": 2}") ?? JNull
let without_b: Json = remove_field(json, "b")
// {"a": 1}
```

### Merging Objects

```kira
let base: Json = parse("{\"a\": 1, \"b\": 2}") ?? JNull
let overlay: Json = parse("{\"b\": 20, \"c\": 30}") ?? JNull

// Shallow merge (overlay fields override base)
let merged: Json = merge(base, overlay)
// {"a": 1, "b": 20, "c": 30}
```

### Deep Merging

```kira
let config1: Json = parse("{\"db\": {\"host\": \"localhost\", \"port\": 5432}}") ?? JNull
let config2: Json = parse("{\"db\": {\"port\": 5433}}") ?? JNull

// Deep merge preserves nested fields
let merged: Json = merge_deep(config1, config2)
// {"db": {"host": "localhost", "port": 5433}}
```

### Mapping Arrays

```kira
let numbers: Json = parse("[1, 2, 3]") ?? JNull

let doubled: Json = map_array(numbers, fn(item: Json) -> Json {
    match as_number(item) {
        Some(n) => { return json_number(n * 2.0) }
        None => { return item }
    }
})
// [2, 4, 6]
```

### Filtering Arrays

```kira
let numbers: Json = parse("[1, 2, 3, 4, 5]") ?? JNull

let evens: Json = filter_array(numbers, fn(item: Json) -> bool {
    match as_int(item) {
        Some(n) => { return n % 2 == 0 }
        None => { return false }
    }
})
// [2, 4]
```

### Filtering Object Fields

```kira
let json: Json = parse("{\"name\": \"Alice\", \"_internal\": true, \"age\": 30}") ?? JNull

// Remove fields starting with underscore
let filtered: Json = filter_fields(json, fn(key: string, _: Json) -> bool {
    return not std.string.starts_with(key, "_")
})
// {"name": "Alice", "age": 30}
```

---

## Path-Based Access

Paths provide a convenient way to access nested values using dot notation and array indices.

### Path Syntax

- `field` - Object field
- `field.nested` - Nested field
- `array[0]` - Array index
- `users[0].name` - Combined

### Getting Values by Path

```kira
let json: Json = parse("{\"users\": [{\"name\": \"Alice\"}, {\"name\": \"Bob\"}]}") ?? JNull

match get_path(json, "users[0].name") {
    Some(name_val) => {
        match as_string(name_val) {
            Some(name) => { std.io.println(name) }  // "Alice"
            None => { }
        }
    }
    None => { std.io.println("Path not found") }
}
```

### Setting Values by Path

```kira
let json: Json = parse("{\"user\": {\"name\": \"Alice\"}}") ?? JNull

// Update existing path
let updated: Json = set_path(json, "user.age", json_int(30))
// {"user": {"name": "Alice", "age": 30}}
```

### Checking Path Existence

```kira
if has_path(json, "user.email") {
    std.io.println("Email exists")
} else {
    std.io.println("No email")
}
```

### Removing by Path

```kira
let json: Json = parse("{\"user\": {\"name\": \"Alice\", \"temp\": true}}") ?? JNull
let cleaned: Json = remove_path(json, "user.temp")
// {"user": {"name": "Alice"}}
```

---

## Error Handling

### Pattern Matching on Errors

```kira
let result: Result[Json, JsonError] = parse(invalid_input)
match result {
    Ok(json) => { /* success */ }
    Err(UnexpectedEof { expected: exp, context: ctx, line: l, col: c }) => {
        std.io.println("Unexpected end of input")
    }
    Err(UnexpectedToken { expected: exp, found: fnd, line: l, col: c }) => {
        std.io.println("Unexpected token: " + fnd)
    }
    Err(UnterminatedString { line: l, col: c }) => {
        std.io.println("String not closed")
    }
    Err(InvalidEscape { sequence: seq, reason: rsn, line: l, col: c }) => {
        std.io.println("Bad escape: " + seq)
    }
    Err(ControlChar { code: cd, line: l, col: c }) => {
        std.io.println("Unescaped control character")
    }
    Err(InvalidNumber { value: val, reason: rsn, line: l, col: c }) => {
        std.io.println("Invalid number: " + val)
    }
    Err(TrailingContent { found: fnd, line: l, col: c }) => {
        std.io.println("Extra content after JSON")
    }
    Err(MaxDepthExceeded { depth: d, max_depth: m, line: l, col: c }) => {
        std.io.println("Too deeply nested")
    }
    Err(DuplicateKey { key: k, line: l, col: c }) => {
        std.io.println("Duplicate key: " + k)
    }
}
```

### Formatted Error Messages

```kira
match result {
    Err(e) => {
        // Simple message
        std.io.println(format_error(e))

        // Message with source context
        std.io.println(format_error_with_context(input, e))
        // Output:
        // Unexpected token at line 1, column 10: expected value, found '}'
        // {"name": }
        //          ^
    }
    Ok(_) => { }
}
```

### Error Position Information

```kira
match result {
    Err(e) => {
        let line: i32 = error_line(e)
        let col: i32 = error_column(e)
        std.io.println("Error at " + std.int.to_string(line) + ":" + std.int.to_string(col))
    }
    Ok(_) => { }
}
```

---

## Schema Validation

### Basic Type Validation

```kira
import src.json.{ Schema, SchemaError, schema_type, validate, format_schema_errors }

let string_schema: Schema = schema_type("string")

match validate(json_string("hello"), string_schema) {
    Ok(_) => { std.io.println("Valid") }
    Err(errors) => { std.io.println(format_schema_errors(errors)) }
}

match validate(json_int(42), string_schema) {
    Ok(_) => { }
    Err(errors) => {
        // "Type mismatch at '$': expected string, got number"
        std.io.println(format_schema_errors(errors))
    }
}
```

### String Length Constraints

```kira
import src.json.{ schema_string }

// Username: 3-20 characters
let username_schema: Schema = schema_string(Some(3), Some(20))

match validate(json_string("ab"), username_schema) {
    Err(errors) => {
        // "Minimum length violation at '$': length 2 is less than minimum 3"
    }
    Ok(_) => { }
}
```

### Number Range Constraints

```kira
import src.json.{ schema_number, schema_integer }

// Score: 0.0 to 100.0
let score_schema: Schema = schema_number(Some(0.0), Some(100.0))

// Age: integer 0 to 150
let age_schema: Schema = schema_integer(Some(0.0), Some(150.0))

match validate(json_number(3.14), age_schema) {
    Err(errors) => {
        // "Integer required at '$': value 3.14 is not an integer"
    }
    Ok(_) => { }
}
```

### Array Validation

```kira
import src.json.{ schema_array }

// Array of strings, 1-5 items
let tags_schema: Schema = schema_array(
    Some(schema_type("string")),  // item schema
    Some(1),                       // minItems
    Some(5)                        // maxItems
)

let tags: Json = json_array(Cons(json_string("a"), Cons(json_int(1), Nil)))
match validate(tags, tags_schema) {
    Err(errors) => {
        // "Type mismatch at '$[1]': expected string, got number"
    }
    Ok(_) => { }
}
```

### Object Validation

```kira
import src.json.{ schema_object, schema_property, SchemaProperty }

let user_schema: Schema = schema_object(
    Some(Cons("name", Cons("email", Nil))),  // required fields
    Some(Cons(
        schema_property("name", schema_string(Some(1), Some(100))),
        Cons(
            schema_property("email", schema_string(Some(5), None)),
            Cons(
                schema_property("age", schema_integer(Some(0.0), Some(150.0))),
                Nil
            )
        )
    ))
)

let user: Json = parse("{\"name\": \"Alice\"}") ?? JNull
match validate(user, user_schema) {
    Err(errors) => {
        // "Missing required field 'email' at '$'"
    }
    Ok(_) => { }
}
```

### Enum Validation

```kira
import src.json.{ schema_enum }

let status_schema: Schema = schema_enum(
    Cons(json_string("active"),
    Cons(json_string("inactive"),
    Cons(json_string("pending"), Nil)))
)

match validate(json_string("unknown"), status_schema) {
    Err(errors) => {
        // "Enum violation at '$': value not in allowed set"
    }
    Ok(_) => { }
}
```

### Nested Validation

Validation errors include the full path to the problematic value:

```kira
let order_schema: Schema = schema_object(
    Some(Cons("items", Nil)),
    Some(Cons(
        schema_property("items", schema_array(
            Some(schema_object(
                Some(Cons("name", Cons("price", Nil))),
                Some(Cons(
                    schema_property("name", schema_type("string")),
                    Cons(schema_property("price", schema_number(Some(0.0), None)), Nil)
                ))
            )),
            Some(1),
            None
        )),
        Nil
    ))
)

let order: Json = parse("{\"items\": [{\"name\": \"Widget\", \"price\": -5}]}") ?? JNull
match validate(order, order_schema) {
    Err(errors) => {
        // "Minimum violation at '$.items[0].price': value -5 is less than minimum 0"
    }
    Ok(_) => { }
}
```

### Handling Multiple Errors

Schema validation collects all errors (not fail-fast):

```kira
let json: Json = parse("{\"name\": 123, \"age\": \"thirty\"}") ?? JNull
match validate(json, user_schema) {
    Err(errors) => {
        // Multiple errors returned
        std.io.println(format_schema_errors(errors))
        // Type mismatch at '$.name': expected string, got number
        // Type mismatch at '$.age': expected integer, got string
    }
    Ok(_) => { }
}
```

---

## Security: Parsing Untrusted Input

When parsing JSON from untrusted sources (user input, external APIs, uploaded files), use resource limits to prevent denial-of-service attacks.

### The Problem

Malicious JSON can exhaust system resources:

- **Deep nesting**: `[[[[[[...]]]]]]` causes stack overflow
- **Huge strings**: `"aaa...aaa"` (millions of characters) exhausts memory
- **Large arrays**: `[1,1,1,...,1]` (millions of elements) exhausts memory
- **Large objects**: `{"a":1,"b":1,...}` (thousands of keys) exhausts memory

### The Solution: Use `parse_with_limits`

```kira
import src.json.{ parse_with_limits, default_limits, ParseLimits }

// Parse with sensible default limits (recommended)
let result: Result[Json, JsonError] = parse_with_limits(untrusted_input, default_limits())
```

The default limits are:
- Max nesting depth: 128 levels
- Max string length: 10 million characters
- Max array items: 1 million elements
- Max object fields: 100 thousand keys

### Custom Limits

For stricter requirements, define custom limits:

```kira
let strict_limits: ParseLimits = ParseLimits {
    max_depth: 20,               // Shallow nesting only
    max_string_length: Some(10000),   // 10KB strings max
    max_array_items: Some(1000),      // 1000 items max
    max_object_fields: Some(100)      // 100 fields max
}

let result: Result[Json, JsonError] = parse_with_limits(input, strict_limits)
```

### Handling Limit Errors

```kira
match parse_with_limits(input, default_limits()) {
    Ok(json) => { /* safe to use */ }
    Err(StringTooLong { length: len, max_length: max, ... }) => {
        std.io.println("String too long: " + std.int.to_string(len) + " chars (max " + std.int.to_string(max) + ")")
    }
    Err(TooManyArrayItems { count: cnt, max_items: max, ... }) => {
        std.io.println("Array too large: " + std.int.to_string(cnt) + " items (max " + std.int.to_string(max) + ")")
    }
    Err(TooManyObjectFields { count: cnt, max_fields: max, ... }) => {
        std.io.println("Object too large: " + std.int.to_string(cnt) + " fields (max " + std.int.to_string(max) + ")")
    }
    Err(MaxDepthExceeded { depth: d, max_depth: max, ... }) => {
        std.io.println("Nesting too deep: " + std.int.to_string(d) + " levels (max " + std.int.to_string(max) + ")")
    }
    Err(e) => {
        std.io.println("Parse error: " + format_error(e))
    }
}
```

### Combining with Strict Mode

For maximum safety, combine limits with strict mode:

```kira
let result: Result[Json, JsonError] = parse_strict_with_limits(input, default_limits())
```

This rejects:
- Inputs that exceed resource limits
- Duplicate object keys (strict RFC 8259 compliance)

### When to Use Each Parser

| Parser | Use Case |
|--------|----------|
| `parse` | Trusted input, development, internal data |
| `parse_strict` | When duplicate keys should be errors |
| `parse_with_limits` | Untrusted input, external APIs |
| `parse_strict_with_limits` | Untrusted input requiring RFC compliance |
