# JSON Library API Reference

A pure functional JSON parsing, manipulation, and serialization library for Kira.

## Table of Contents

- [Types](#types)
- [Parsing](#parsing)
- [Serialization](#serialization)
- [Type Predicates](#type-predicates)
- [Value Extraction](#value-extraction)
- [Accessors](#accessors)
- [Transformations](#transformations)
- [Path-Based Access](#path-based-access)
- [Equality](#equality)
- [Constructors](#constructors)
- [Builder Pattern](#builder-pattern)
- [Error Handling](#error-handling)
- [Schema Validation](#schema-validation)

---

## Types

### Json

The core sum type representing any JSON value.

```kira
pub type Json =
    | JNull
    | JBool(bool)
    | JNumber(f64)
    | JString(string)
    | JArray(List[Json])
    | JObject(HashMap)
```

**Note:** `JObject` uses `HashMap` for O(1) field lookup.

### JsonField

A key-value pair for JSON objects.

```kira
pub type JsonField = {
    key: string,
    value: Json
}
```

### JsonError

Structured error type for parsing failures. Enables pattern matching on specific error categories.

```kira
pub type JsonError =
    | UnexpectedEof { expected: string, context: string, line: i32, col: i32 }
    | UnexpectedToken { expected: string, found: string, line: i32, col: i32 }
    | UnterminatedString { line: i32, col: i32 }
    | InvalidEscape { sequence: string, reason: string, line: i32, col: i32 }
    | ControlChar { code: i32, line: i32, col: i32 }
    | InvalidNumber { value: string, reason: string, line: i32, col: i32 }
    | TrailingContent { found: string, line: i32, col: i32 }
    | MaxDepthExceeded { depth: i32, max_depth: i32, line: i32, col: i32 }
    | DuplicateKey { key: string, line: i32, col: i32 }
    | StringTooLong { length: i32, max_length: i32, line: i32, col: i32 }
    | TooManyArrayItems { count: i32, max_items: i32, line: i32, col: i32 }
    | TooManyObjectFields { count: i32, max_fields: i32, line: i32, col: i32 }
```

### ParseLimits

Resource limits for parsing untrusted JSON input. Used to prevent resource exhaustion attacks.

```kira
pub type ParseLimits = {
    max_depth: i32,                      // Maximum nesting depth (default: 128)
    max_string_length: Option[i32],      // Max characters per string (None = unlimited)
    max_array_items: Option[i32],        // Max items per array (None = unlimited)
    max_object_fields: Option[i32]       // Max fields per object (None = unlimited)
}
```

### PathError

Error type for path parsing failures.

```kira
pub type PathError =
    | EmptyPath
    | InvalidIndex { segment: string }
    | UnterminatedBracket { segment: string }
```

### JsonBuilder

Builder type for fluent JSON object construction.

```kira
pub type JsonBuilder = {
    fields: List[JsonField]
}
```

---

## Parsing

### parse

Parses a JSON string with default settings (128 max depth, first-wins for duplicate keys).

```kira
pub let parse: fn(string) -> Result[Json, JsonError]
```

**Example:**
```kira
let result: Result[Json, JsonError] = parse("{\"name\": \"Alice\", \"age\": 30}")
match result {
    Ok(json) => { /* use json */ }
    Err(e) => { std.io.println(format_error(e)) }
}
```

### parse_strict

Parses JSON and rejects duplicate keys with a `DuplicateKey` error.

```kira
pub let parse_strict: fn(string) -> Result[Json, JsonError]
```

**Example:**
```kira
let result: Result[Json, JsonError] = parse_strict("{\"a\": 1, \"a\": 2}")
// Returns Err(DuplicateKey { key: "a", ... })
```

### parse_with_max_depth

Parses JSON with a custom maximum nesting depth.

```kira
pub let parse_with_max_depth: fn(string, i32) -> Result[Json, JsonError]
```

**Example:**
```kira
let result: Result[Json, JsonError] = parse_with_max_depth(deep_json, 50)
// Returns Err(MaxDepthExceeded {...}) if nesting exceeds 50 levels
```

### parse_strict_with_max_depth

Combines strict parsing (no duplicate keys) with custom max depth.

```kira
pub let parse_strict_with_max_depth: fn(string, i32) -> Result[Json, JsonError]
```

### default_max_depth

Returns the default maximum nesting depth (128).

```kira
pub let default_max_depth: fn() -> i32
```

### default_limits

Returns sensible default limits for parsing untrusted JSON input.

```kira
pub let default_limits: fn() -> ParseLimits
```

**Default values:**
- `max_depth`: 128
- `max_string_length`: Some(10,000,000) (10 million characters)
- `max_array_items`: Some(1,000,000) (1 million items)
- `max_object_fields`: Some(100,000) (100 thousand fields)

### unlimited_limits

Returns unlimited limits (no restrictions except max_depth of 128). Use with caution on untrusted input.

```kira
pub let unlimited_limits: fn() -> ParseLimits
```

### parse_with_limits

Parses JSON with custom resource limits. Use for parsing untrusted input to prevent resource exhaustion.

```kira
pub let parse_with_limits: fn(string, ParseLimits) -> Result[Json, JsonError]
```

**Example:**
```kira
// Parse with default limits (recommended for untrusted input)
let result: Result[Json, JsonError] = parse_with_limits(untrusted_input, default_limits())

// Parse with custom limits
let limits: ParseLimits = ParseLimits {
    max_depth: 50,
    max_string_length: Some(1000),
    max_array_items: Some(100),
    max_object_fields: Some(50)
}
let result: Result[Json, JsonError] = parse_with_limits(input, limits)
```

**Possible errors:**
- `StringTooLong` - A string exceeded `max_string_length`
- `TooManyArrayItems` - An array exceeded `max_array_items`
- `TooManyObjectFields` - An object exceeded `max_object_fields`
- `MaxDepthExceeded` - Nesting exceeded `max_depth`

### parse_strict_with_limits

Combines strict mode (rejects duplicate keys) with resource limits. Recommended for parsing untrusted input that requires RFC 8259 compliance.

```kira
pub let parse_strict_with_limits: fn(string, ParseLimits) -> Result[Json, JsonError]
```

**Example:**
```kira
let result: Result[Json, JsonError] = parse_strict_with_limits(input, default_limits())
match result {
    Ok(json) => { /* use json */ }
    Err(DuplicateKey { key: k, ... }) => { std.io.println("Duplicate key: " + k) }
    Err(StringTooLong { ... }) => { std.io.println("String too long") }
    Err(e) => { std.io.println(format_error(e)) }
}
```

---

## Serialization

### stringify

Converts JSON to a compact string representation.

```kira
pub let stringify: fn(Json) -> string
```

**Example:**
```kira
let json: Json = json_object(Cons(make_field("name", JString("Alice")), Nil))
let s: string = stringify(json)
// Result: {"name":"Alice"}
```

### stringify_pretty

Converts JSON to a formatted string with 2-space indentation.

```kira
pub let stringify_pretty: fn(Json) -> string
```

**Example:**
```kira
let s: string = stringify_pretty(json)
// Result:
// {
//   "name": "Alice"
// }
```

### stringify_pretty_with

Converts JSON to a formatted string with custom indentation.

```kira
pub let stringify_pretty_with: fn(Json, i32) -> string
```

**Example:**
```kira
let s: string = stringify_pretty_with(json, 4)  // 4-space indentation
```

---

## Type Predicates

### is_null

```kira
pub let is_null: fn(Json) -> bool
```

### is_bool

```kira
pub let is_bool: fn(Json) -> bool
```

### is_number

```kira
pub let is_number: fn(Json) -> bool
```

### is_string

```kira
pub let is_string: fn(Json) -> bool
```

### is_array

```kira
pub let is_array: fn(Json) -> bool
```

### is_object

```kira
pub let is_object: fn(Json) -> bool
```

### type_name

Returns the JSON type as a string: "null", "boolean", "number", "string", "array", or "object".

```kira
pub let type_name: fn(Json) -> string
```

**Example:**
```kira
type_name(JNumber(42.0))  // "number"
type_name(JArray(Nil))    // "array"
```

---

## Value Extraction

All extraction functions return `Option[T]` for safe access.

### as_string

```kira
pub let as_string: fn(Json) -> Option[string]
```

### as_number

```kira
pub let as_number: fn(Json) -> Option[f64]
```

### as_int

Returns the number as an integer if it has no fractional part.

```kira
pub let as_int: fn(Json) -> Option[i64]
```

### as_bool

```kira
pub let as_bool: fn(Json) -> Option[bool]
```

### as_array

```kira
pub let as_array: fn(Json) -> Option[List[Json]]
```

### as_object

Returns object fields as a list (for backward compatibility).

```kira
pub let as_object: fn(Json) -> Option[List[JsonField]]
```

### as_object_map

Returns the underlying HashMap for direct access.

```kira
pub let as_object_map: fn(Json) -> Option[HashMap]
```

---

## Accessors

### get_field

Gets a field value from an object by key. O(1) lookup.

```kira
pub let get_field: fn(Json, string) -> Option[Json]
```

**Example:**
```kira
match get_field(json, "name") {
    Some(val) => { /* use val */ }
    None => { /* field not found */ }
}
```

### get_index

Gets an element from an array by index.

```kira
pub let get_index: fn(Json, i32) -> Option[Json]
```

### size

Returns the size of an array or object.

```kira
pub let size: fn(Json) -> Option[i32]
```

### keys

Returns the keys of an object.

```kira
pub let keys: fn(Json) -> Option[List[string]]
```

### values

Returns the values of an object.

```kira
pub let values: fn(Json) -> Option[List[Json]]
```

---

## Transformations

### map_values

Applies a function to all values in an object.

```kira
pub let map_values: fn(Json, fn(Json) -> Json) -> Json
```

### filter_fields

Filters object fields by a predicate.

```kira
pub let filter_fields: fn(Json, fn(string, Json) -> bool) -> Json
```

### filter_fields_deep

Recursively filters object fields.

```kira
pub let filter_fields_deep: fn(Json, fn(string, Json) -> bool) -> Json
```

### set_field

Sets or updates a field in an object. O(1) operation.

```kira
pub let set_field: fn(Json, string, Json) -> Json
```

**Example:**
```kira
let updated: Json = set_field(json, "age", JNumber(31.0))
```

### remove_field

Removes a field from an object. O(1) operation.

```kira
pub let remove_field: fn(Json, string) -> Json
```

### merge

Merges two objects. Fields from the second object override the first.

```kira
pub let merge: fn(Json, Json) -> Json
```

### merge_deep

Recursively merges two objects. Nested objects are merged rather than replaced.

```kira
pub let merge_deep: fn(Json, Json) -> Json
```

### map_array

Applies a function to each element of an array.

```kira
pub let map_array: fn(Json, fn(Json) -> Json) -> Json
```

### filter_array

Filters array elements by a predicate.

```kira
pub let filter_array: fn(Json, fn(Json) -> bool) -> Json
```

---

## Path-Based Access

Paths use dot notation for fields and bracket notation for array indices: `"user.addresses[0].city"`

### get_path

Gets a value at the specified path.

```kira
pub let get_path: fn(Json, string) -> Option[Json]
```

**Example:**
```kira
let city: Option[Json] = get_path(json, "user.addresses[0].city")
```

### set_path

Sets a value at the specified path, creating intermediate objects/arrays as needed.

```kira
pub let set_path: fn(Json, string, Json) -> Json
```

### has_path

Checks if a path exists.

```kira
pub let has_path: fn(Json, string) -> bool
```

### remove_path

Removes the value at the specified path.

```kira
pub let remove_path: fn(Json, string) -> Json
```

### parse_path

Parses a path string into segments.

```kira
pub let parse_path: fn(string) -> Result[List[PathSegment], PathError]
```

### format_path_error

Formats a path error as a human-readable string.

```kira
pub let format_path_error: fn(PathError) -> string
```

---

## Equality

### equals

Deep structural equality comparison. Objects are compared regardless of key order.

```kira
pub let equals: fn(Json, Json) -> bool
```

**Example:**
```kira
let a: Json = parse("{\"x\": 1, \"y\": 2}") ?? JNull
let b: Json = parse("{\"y\": 2, \"x\": 1}") ?? JNull
equals(a, b)  // true
```

---

## Constructors

Convenience functions for creating JSON values.

### json_null

```kira
pub let json_null: fn() -> Json
```

### json_bool

```kira
pub let json_bool: fn(bool) -> Json
```

### json_number

Creates a number from f64.

```kira
pub let json_number: fn(f64) -> Json
```

### json_int

Creates a number from i64.

```kira
pub let json_int: fn(i64) -> Json
```

### json_string

```kira
pub let json_string: fn(string) -> Json
```

### json_array

```kira
pub let json_array: fn(List[Json]) -> Json
```

### json_array_empty

```kira
pub let json_array_empty: fn() -> Json
```

### json_object

Creates an object from a list of fields.

```kira
pub let json_object: fn(List[JsonField]) -> Json
```

### json_object_from_map

Creates an object from a HashMap.

```kira
pub let json_object_from_map: fn(HashMap) -> Json
```

### json_object_empty

```kira
pub let json_object_empty: fn() -> Json
```

### make_field

Creates a JsonField from a key and value.

```kira
pub let make_field: fn(string, Json) -> JsonField
```

---

## Builder Pattern

Fluent API for constructing JSON objects.

### builder

Creates a new empty builder.

```kira
pub let builder: fn() -> JsonBuilder
```

### with_field

Adds a field with any JSON value.

```kira
pub let with_field: fn(JsonBuilder, string, Json) -> JsonBuilder
```

### with_string

Adds a string field.

```kira
pub let with_string: fn(JsonBuilder, string, string) -> JsonBuilder
```

### with_number

Adds a number field (f64).

```kira
pub let with_number: fn(JsonBuilder, string, f64) -> JsonBuilder
```

### with_int

Adds an integer field (i64).

```kira
pub let with_int: fn(JsonBuilder, string, i64) -> JsonBuilder
```

### with_bool

Adds a boolean field.

```kira
pub let with_bool: fn(JsonBuilder, string, bool) -> JsonBuilder
```

### with_null

Adds a null field.

```kira
pub let with_null: fn(JsonBuilder, string) -> JsonBuilder
```

### with_array

Adds an array field.

```kira
pub let with_array: fn(JsonBuilder, string, List[Json]) -> JsonBuilder
```

### with_object

Adds a nested object from another builder.

```kira
pub let with_object: fn(JsonBuilder, string, JsonBuilder) -> JsonBuilder
```

### build

Builds the final JSON object.

```kira
pub let build: fn(JsonBuilder) -> Json
```

**Example:**
```kira
let person: Json = build(
    with_int(
        with_string(
            builder(),
            "name", "Alice"),
        "age", 30))
// {"name": "Alice", "age": 30}
```

---

## Error Handling

### format_error

Converts a JsonError to a human-readable string.

```kira
pub let format_error: fn(JsonError) -> string
```

### error_line

Returns the line number from a JsonError.

```kira
pub let error_line: fn(JsonError) -> i32
```

### error_column

Returns the column number from a JsonError.

```kira
pub let error_column: fn(JsonError) -> i32
```

### format_error_with_context

Formats an error with the source line and a column indicator.

```kira
pub let format_error_with_context: fn(string, JsonError) -> string
```

**Example:**
```kira
let input: string = "{\"name\": }"
let result: Result[Json, JsonError] = parse(input)
match result {
    Err(e) => {
        std.io.println(format_error_with_context(input, e))
        // Output:
        // Unexpected token at line 1, column 10: expected value, found '}'
        // {"name": }
        //          ^
    }
    Ok(_) => { }
}
```

---

## Schema Validation

### Schema Types

```kira
pub type Schema = {
    type_constraint: Option[string],  // "null", "boolean", "number", "integer", "string", "array", "object"
    required: Option[List[string]],   // Required field names (objects)
    properties: Option[List[SchemaProperty]],  // Property schemas (objects)
    minimum: Option[f64],             // Min value (numbers)
    maximum: Option[f64],             // Max value (numbers)
    min_length: Option[i32],          // Min string length
    max_length: Option[i32],          // Max string length
    min_items: Option[i32],           // Min array length
    max_items: Option[i32],           // Max array length
    items: Option[SchemaRef],         // Array item schema
    enum_values: Option[List[Json]]   // Allowed values
}

pub type SchemaProperty = {
    name: string,
    schema: SchemaRef
}

pub type SchemaRef =
    | SchemaInline(Schema)

pub type SchemaError =
    | TypeMismatch { expected: string, actual: string, path: string }
    | MissingRequired { field: string, path: string }
    | MinimumViolation { value: f64, minimum: f64, path: string }
    | MaximumViolation { value: f64, maximum: f64, path: string }
    | MinLengthViolation { actual: i32, min_length: i32, path: string }
    | MaxLengthViolation { actual: i32, max_length: i32, path: string }
    | MinItemsViolation { actual: i32, min_items: i32, path: string }
    | MaxItemsViolation { actual: i32, max_items: i32, path: string }
    | EnumViolation { path: string }
    | IntegerRequired { value: f64, path: string }
```

### Schema Constructors

#### schema_any

Creates a schema that validates anything.

```kira
pub let schema_any: fn() -> Schema
```

#### schema_type

Creates a type-only schema.

```kira
pub let schema_type: fn(string) -> Schema
```

**Example:**
```kira
let string_schema: Schema = schema_type("string")
```

#### schema_string

Creates a string schema with optional length constraints.

```kira
pub let schema_string: fn(Option[i32], Option[i32]) -> Schema
```

**Example:**
```kira
let username_schema: Schema = schema_string(Some(3), Some(20))  // 3-20 chars
```

#### schema_number

Creates a number schema with optional range constraints.

```kira
pub let schema_number: fn(Option[f64], Option[f64]) -> Schema
```

#### schema_integer

Creates an integer schema with optional range constraints.

```kira
pub let schema_integer: fn(Option[f64], Option[f64]) -> Schema
```

#### schema_array

Creates an array schema with optional item schema and length constraints.

```kira
pub let schema_array: fn(Option[Schema], Option[i32], Option[i32]) -> Schema
```

**Example:**
```kira
let numbers_schema: Schema = schema_array(Some(schema_type("number")), Some(1), Some(10))
```

#### schema_object

Creates an object schema with required fields and property schemas.

```kira
pub let schema_object: fn(Option[List[string]], Option[List[SchemaProperty]]) -> Schema
```

#### schema_property

Creates a schema property (name + schema pair).

```kira
pub let schema_property: fn(string, Schema) -> SchemaProperty
```

#### schema_ref

Wraps a schema in a SchemaRef for recursive definitions.

```kira
pub let schema_ref: fn(Schema) -> SchemaRef
```

#### schema_enum

Creates an enum schema that accepts only specified values.

```kira
pub let schema_enum: fn(List[Json]) -> Schema
```

**Example:**
```kira
let status_schema: Schema = schema_enum(Cons(JString("active"), Cons(JString("inactive"), Nil)))
```

### validate

Validates JSON against a schema. Collects all errors.

```kira
pub let validate: fn(Json, Schema) -> Result[void, List[SchemaError]]
```

**Example:**
```kira
let schema: Schema = schema_object(
    Some(Cons("name", Cons("age", Nil))),  // required fields
    Some(Cons(
        schema_property("name", schema_string(Some(1), None)),
        Cons(schema_property("age", schema_integer(Some(0.0), Some(150.0))), Nil)
    ))
)

match validate(json, schema) {
    Ok(_) => { std.io.println("Valid!") }
    Err(errors) => { std.io.println(format_schema_errors(errors)) }
}
```

### format_schema_error

Formats a single schema error.

```kira
pub let format_schema_error: fn(SchemaError) -> string
```

### format_schema_errors

Formats multiple schema errors as a newline-separated string.

```kira
pub let format_schema_errors: fn(List[SchemaError]) -> string
```
