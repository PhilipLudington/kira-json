# Project Bugs

## [ ] Bug 1: MatchFailed runtime error in JSON parsing/serialization

**Status:** Blocking

**Description:** The kira-json library has a runtime bug that causes `error.MatchFailed` when parsing or serializing JSON. This prevents all library functionality from working.

**Symptoms:**
```
Runtime error: error.MatchFailed
Error: error.RuntimeError
```

**Reproduction:**
```kira
module test

import src.json.parser.{ parse }
import src.json.types.{ Json, JsonError, JNull }

effect fn main() -> IO[void] {
    let result: Result[Json, JsonError] = parse("null")
    match result {
        Ok(json) => { std.io.println("OK") }
        Err(_) => { std.io.println("Error") }
    }
    return
}
```

**Affected operations:**
- Parsing any valid JSON (`"null"`, `"42"`, `"{}"`, etc.)
- Serializing JSON values with `stringify()`
- Running `examples/json_demo.ki`
- Running `tests/test_json.ki`

**Investigation notes:**
- All source files pass `kira check` (type checking succeeds)
- Error occurs at runtime, not compile time
- The serializer's `stringify_entries_to_builder` function uses tuple destructuring (`entry.0`, `entry.1`) from `std.map.entries()` which may be involved
- Likely a Kira runtime issue with tuple field access or `std.map.entries()`

**Discovered:** 2026-01-27
