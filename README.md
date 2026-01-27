# kira-json

A complete JSON library for the [Kira](https://github.com/mrphil/kira) programming language with parsing, serialization, transformation, path-based access, and schema validation.

## Features

- **Parsing** - Full JSON specification compliance with detailed error messages
- **Serialization** - Compact and pretty-printed output
- **Type-safe access** - Predicates and extractors for all JSON types
- **Transformations** - Map, filter, merge, set/remove fields
- **Path-based access** - Query and modify nested values with `user.address.city` or `items[0].name`
- **Schema validation** - Validate JSON structure with type constraints, ranges, and required fields
- **Builder pattern** - Fluent API for constructing JSON objects
- **HashMap objects** - O(1) field lookup performance
- **Resource limits** - Protection against malicious input (depth, string length, array/object size)

## Module Structure

```
src/json/
  types.ki       Core types, predicates, accessors
  parser.ki      Parsing functions
  serializer.ki  Serialization functions
  transform.ki   Map, filter, merge operations
  path.ki        Path-based access
  builder.ki     Builder pattern and constructors
  schema.ki      Schema validation
  equality.ki    Deep equality comparison
```

## Quick Start

```kira
import src.json.types.{ Json, JObject }
import src.json.parser.{ parse }
import src.json.serializer.{ stringify }
import src.json.types.{ get_field, as_string }

effect fn main() -> IO[void] {
    let input: string = "{\"name\": \"Alice\", \"age\": 30}"

    match parse(input) {
        Ok(json) => {
            match get_field(json, "name") {
                Some(val) => {
                    match as_string(val) {
                        Some(name) => { std.io.println(name) }  // "Alice"
                        None => { }
                    }
                }
                None => { }
            }
        }
        Err(e) => { std.io.println("Parse error") }
    }
    return
}
```

## Building JSON

```kira
import src.json.builder.{ builder, with_string, with_int, with_bool, build }

let json: Json = build(
    with_bool(
        with_int(
            with_string(builder(), "name", "Alice"),
            "age", 30),
        "active", true))

// {"name":"Alice","age":30,"active":true}
```

## Documentation

- [Usage Guide](docs/guide.md) - Getting started, examples, and best practices
- [API Reference](docs/api.md) - Complete function documentation
- [Performance Guide](docs/performance.md) - Optimization tips and complexity analysis

## Testing

Run the test suite:

```bash
kira run tests/test_json.ki
kira run tests/test_json_suite.ki
```

## License

MIT
