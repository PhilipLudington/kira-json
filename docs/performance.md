# JSON Library Performance Guide

This guide covers performance characteristics and best practices for using the Kira JSON library efficiently.

## Table of Contents

- [Data Structure Performance](#data-structure-performance)
- [Parser Performance](#parser-performance)
- [Memory Considerations](#memory-considerations)
- [Best Practices](#best-practices)
- [Limitations](#limitations)

---

## Data Structure Performance

### HashMap-Based Objects

JSON objects use `HashMap` internally for O(1) field access:

| Operation | Time Complexity |
|-----------|-----------------|
| `get_field` | O(1) |
| `set_field` | O(1) |
| `remove_field` | O(1) |
| `keys` | O(n) |
| `values` | O(n) |
| `as_object` (to List) | O(n) |

**Example:**
```kira
// O(1) field lookup - efficient for large objects
match get_field(large_object, "target_field") {
    Some(val) => { /* found in constant time */ }
    None => { }
}
```

### Path-Based Access

Path operations (`get_path`, `set_path`, etc.) traverse the structure:

| Operation | Time Complexity |
|-----------|-----------------|
| `get_path` | O(d) where d = path depth |
| `set_path` | O(d) |
| `has_path` | O(d) |
| `remove_path` | O(d) |

Each segment in the path requires one lookup:
- Object field access: O(1) per segment (HashMap)
- Array index access: O(n) per segment (List traversal)

**Example:**
```kira
// "users[0].profile.name" = 4 segments
// Cost: O(1) + O(n) + O(1) + O(1) where n = array length at users
get_path(json, "users[0].profile.name")
```

### Array Operations

Arrays are stored as `List[Json]` (linked lists):

| Operation | Time Complexity |
|-----------|-----------------|
| `get_index` | O(n) |
| `size` (array) | O(n) |
| `map_array` | O(n) |
| `filter_array` | O(n) |
| Prepend (Cons) | O(1) |

**Recommendation:** For frequent random access to array elements, consider whether your data model might benefit from using objects with numeric string keys for critical paths.

---

## Parser Performance

### StringBuilder Optimization

The parser uses `StringBuilder` for string construction, providing linear-time parsing for strings:

```kira
// Efficient: O(n) for string of length n
let result: Result[Json, JsonError] = parse("{\"long_string\": \"...thousands of chars...\"}")
```

### Depth Limits

The parser has a configurable depth limit (default: 128) to prevent stack overflow:

```kira
// Default: 128 levels of nesting
let result: Result[Json, JsonError] = parse(deep_json)

// Custom limit for untrusted input
let result: Result[Json, JsonError] = parse_with_max_depth(untrusted, 50)
```

**Why 128?** The Kira interpreter has a call stack limit around 1000 frames. Each nesting level in JSON parsing requires multiple stack frames, so 128 provides a safe margin while allowing reasonable nesting.

### Parser Complexity

| Input Type | Time Complexity |
|------------|-----------------|
| Primitives (null, bool, number) | O(n) where n = token length |
| Strings | O(n) where n = string length |
| Arrays | O(m * avg_element_cost) |
| Objects | O(m * avg_field_cost) |
| Overall | O(total input size) |

The parser is single-pass and streaming-friendly in design.

---

## Memory Considerations

### Immutable Data Structures

All transformations create new values (pure functional style):

```kira
let original: Json = parse("{\"a\": 1}") ?? JNull
let updated: Json = set_field(original, "b", json_int(2))
// original is unchanged
// updated is a new value
```

**Implication:** Chains of transformations create intermediate values:

```kira
// Each operation creates a new Json value
let result: Json = remove_field(
    set_field(
        set_field(base, "x", json_int(1)),
        "y", json_int(2)),
    "temp")
```

For many modifications, consider:
1. Using the builder pattern for initial construction
2. Applying transformations in batch where possible

### Object Construction

When building objects with many fields, the builder pattern is efficient:

```kira
// Efficient: fields accumulated in list, converted to HashMap once at build()
let obj: Json = build(
    with_string(
        with_int(
            with_bool(
                builder(),
                "active", true),
            "count", 42),
        "name", "test")
)
```

vs. repeated `set_field` calls:

```kira
// Less efficient: creates new HashMap for each set_field
var obj: Json = json_object_empty()
obj = set_field(obj, "active", json_bool(true))
obj = set_field(obj, "count", json_int(42))
obj = set_field(obj, "name", json_string("test"))
```

### String Serialization

`stringify` and `stringify_pretty` use `StringBuilder` internally for efficient output:

```kira
// Efficient: linear time in output size
let output: string = stringify_pretty(large_json)
```

---

## Best Practices

### 1. Prefer Direct Field Access Over Paths

For single-level access, `get_field` is more efficient than `get_path`:

```kira
// Preferred for top-level fields
match get_field(json, "name") { ... }

// Use paths for deep access
match get_path(json, "user.profile.settings.theme") { ... }
```

### 2. Use Strict Parsing for Untrusted Input

`parse_strict` catches duplicate keys that could indicate malicious input:

```kira
// For API input, config files from users, etc.
let result: Result[Json, JsonError] = parse_strict(user_input)
```

### 3. Set Appropriate Depth Limits

For untrusted input, use conservative depth limits:

```kira
// External API responses
let result: Result[Json, JsonError] = parse_with_max_depth(api_response, 32)

// User-uploaded files
let result: Result[Json, JsonError] = parse_with_max_depth(uploaded_json, 16)
```

### 4. Batch Object Modifications

When making multiple changes, structure your code to minimize intermediate values:

```kira
// Build the complete object at once
let updates: Json = build(
    with_string(
        with_int(
            builder(),
            "new_count", 100),
        "status", "updated")
)
let result: Json = merge(original, updates)
```

### 5. Use Type Predicates Before Extraction

Check types before attempting extraction to avoid unnecessary Option handling:

```kira
if is_string(value) {
    // as_string will return Some
    let s: string = as_string(value) ?? ""
    // process s
}
```

### 6. Consider Structure for Your Access Patterns

If you frequently access the same deep paths, consider restructuring:

```kira
// Instead of repeatedly: get_path(json, "config.db.connection.host")
// Extract the nested object once:
let db_config: Json = get_path(json, "config.db") ?? json_object_empty()
// Then use direct access:
let host: Json = get_field(db_config, "host") ?? JNull
```

---

## Limitations

### Recursive Interpreter

Kira uses a recursive interpreter with a ~1000 frame call stack limit. This affects:

1. **Maximum nesting depth**: Practical limit around 128 levels
2. **Very large arrays**: Processing extremely long arrays may hit stack limits
3. **Deep recursion in transforms**: Chained map/filter operations on large datasets

**Mitigation:** The library's default depth limit (128) is set conservatively.

### No Streaming Parser

The current parser loads the entire JSON into memory. For very large files:

1. Consider splitting data into smaller chunks at the application level
2. Process data incrementally if your use case allows

### Floating Point Numbers

All JSON numbers are stored as `f64`:

```kira
// Integer precision limit: ~2^53
let big_int: Json = json_int(9007199254740993)  // May lose precision

// Use as_int for safe integer extraction
match as_int(json) {
    Some(n) => { /* n is i64 with full precision */ }
    None => { /* not an integer or out of range */ }
}
```

### Array Index Performance

Array access by index is O(n) due to linked list structure:

```kira
// O(n) for each access
get_index(array, 1000)  // Traverses 1000 elements
```

For performance-critical random access, consider alternative data models.

---

## Performance Summary

| Category | Recommendation |
|----------|----------------|
| Object field access | Use `get_field` - O(1) |
| Deep nested access | Use `get_path` - O(depth) |
| Array random access | Avoid if possible - O(n) |
| Object construction | Use builder pattern |
| Multiple modifications | Batch with `merge` |
| Untrusted input | Use `parse_strict` with depth limit |
| Large JSON | Mind memory for immutable transforms |
