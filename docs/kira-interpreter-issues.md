# Kira Interpreter Issues Report

**Date:** 2026-01-20
**Kira Version:** (as installed at `/usr/local/bin/kira`)
**Platform:** macOS Darwin 24.6.0
**Project:** kira-json (JSON library implementation)

## Summary

During development and testing of a JSON parsing library for Kira, several interpreter issues were discovered. This document details each issue with reproduction steps and workarounds where available.

---

## Issue 1: `std.float.to_string` Not Found

**Severity:** High
**Category:** Missing Standard Library Function

### Description

Calling `std.float.to_string(n)` where `n` is an `f64` value results in a runtime error:

```
Runtime error: error.FieldNotFound
```

### Reproduction

```kira
effect fn main() -> IO[void] {
    let n: f64 = 42.5
    let s: string = std.float.to_string(n)  // ERROR
    std.io.println(s)
    return
}
```

### Expected Behavior

The function should convert a floating-point number to its string representation.

### Workaround

None available. Number-to-string conversion is not possible, limiting JSON serialization capabilities.

### Suggested Fix

Implement `std.float.to_string(f64) -> string` in the standard library, or document the correct module path if it exists elsewhere.

---

## Issue 2: `std.string.equals` Not Found

**Severity:** High
**Category:** Missing Standard Library Function

### Description

Calling `std.string.equals(a, b)` to compare two strings results in a runtime error:

```
Runtime error: error.FieldNotFound
```

### Reproduction

```kira
effect fn main() -> IO[void] {
    let a: string = "hello"
    let b: string = "hello"
    if std.string.equals(a, b) {  // ERROR
        std.io.println("equal")
    }
    return
}
```

### Expected Behavior

The function should return `true` if two strings are equal, `false` otherwise.

### Workaround

Use the `==` operator for string comparison:

```kira
if a == b {
    std.io.println("equal")
}
```

### Suggested Fix

Either implement `std.string.equals` or document that `==` is the idiomatic way to compare strings in Kira.

---

## Issue 3: Multi-line `Cons` Expressions Parse Error

**Severity:** Medium
**Category:** Parser

### Description

Multi-line `Cons` constructor expressions with closing parentheses on separate lines cause parse errors.

### Reproduction

```kira
// This FAILS to parse:
let list: List[i32] = Cons(
    1,
    Cons(
        2,
        Nil
    )
)

// Error: expected ')' after variant arguments
```

### Expected Behavior

Multi-line expressions should parse correctly with standard indentation.

### Workaround

Write `Cons` expressions on a single line or use intermediate variables:

```kira
// Single line (works)
let list: List[i32] = Cons(1, Cons(2, Nil))

// Or use intermediate variables (works)
let inner: List[i32] = Cons(2, Nil)
let list: List[i32] = Cons(1, inner)
```

### Suggested Fix

Update the parser to handle newlines within parenthesized expressions for sum type constructors.

---

## Issue 4: String Output Double-Escaping

**Severity:** Low
**Category:** Runtime / IO

### Description

When printing strings that contain quotes, the output appears to have extra escaping or quote characters.

### Reproduction

```kira
effect fn main() -> IO[void] {
    let s: string = "hello"
    std.io.println(s)  // Output: "hello" (with quotes)
    return
}
```

### Observed Output

```
"hello"
```

With nested strings in data structures, this compounds:

```
"{""""\"""name""\""": ""\"""Alice""\"""}"
```

### Expected Output

```
hello
```

Or for JSON:

```
{"name": "Alice"}
```

### Workaround

None. This is a cosmetic issue that doesn't affect program logic but makes output harder to read.

### Suggested Fix

Review the `std.io.println` implementation to ensure strings are printed without automatic quoting, or provide a separate function for raw string output.

---

## Issue 5: Segmentation Fault on Program Exit

**Severity:** Medium
**Category:** Memory Management

### Description

Programs that run successfully still produce segmentation faults during cleanup/exit:

```
Segmentation fault at address 0x102820008
???:?:?: 0x19bcf90a0 in ??? (libsystem_platform.dylib)
???:?:?: 0x10245eda3 in _symbols.table.SymbolTable.deinit (???)
```

### Reproduction

Any program, including the minimal example:

```kira
effect fn main() -> IO[void] {
    std.io.println("Hello")
    return
}
```

### Observed Behavior

1. Program executes correctly and produces expected output
2. After `main` returns, segfault occurs during symbol table cleanup
3. Exit code is non-zero despite successful execution

### Impact

- CI/CD pipelines may incorrectly report failures
- Makes it difficult to distinguish between runtime errors and successful runs

### Suggested Fix

Debug the `SymbolTable.deinit` function in the interpreter. The crash appears to be in memory deallocation during cleanup.

---

## Issue 6: Memory Leak Warnings

**Severity:** Low
**Category:** Memory Management

### Description

Programs produce numerous memory leak warnings on exit:

```
error(gpa): memory address 0x101300300 leaked:
???:?:?: 0x101081697 in _typechecker.checker.TypeChecker.resolveAstType (???)
```

### Impact

- Noisy output obscures actual program output
- Indicates potential memory management issues in the interpreter

### Suggested Fix

Review allocation/deallocation patterns in the type checker and interpreter. Consider using arena allocation for AST nodes if not already doing so.

---

## Issue 7: Cross-Module Function Calls May Fail

**Severity:** High
**Category:** Module System

### Description

Calling functions from imported modules sometimes results in `FieldNotFound` errors, even when the function is correctly exported with `pub`.

### Reproduction

```kira
// src/json.ki
module json

pub let is_null: fn(Json) -> bool = fn(json: Json) -> bool {
    match json {
        JNull => { return true }
        _ => { return false }
    }
}
```

```kira
// examples/demo.ki
import src.json

effect fn main() -> IO[void] {
    let val: src.json.Json = src.json.JNull
    let result: bool = src.json.is_null(val)  // Sometimes ERROR
    return
}
```

### Workaround

Define functions inline in the same file rather than importing from modules.

### Suggested Fix

Investigate the module resolution and symbol lookup for imported functions. The issue may be related to how qualified names (`src.json.is_null`) are resolved at runtime.

---

## Environment Details

```
Platform: darwin (macOS)
OS Version: Darwin 24.6.0
Kira binary: /usr/local/bin/kira
```

## Test Files

The following files in this repository can be used to reproduce these issues:

- `examples/simple_demo.ki` - Working demo with workarounds applied
- `examples/json_demo.ki` - Original demo (exhibits issues)
- `src/json.ki` - Full JSON library (import issues)
- `tests/test_json.ki` - Test suite

## Recommendations

1. **Priority 1:** Fix `std.float.to_string` - essential for numeric output
2. **Priority 2:** Fix cross-module function calls - essential for code organization
3. **Priority 3:** Fix segfault on exit - affects CI/CD reliability
4. **Priority 4:** Fix multi-line parsing - affects code readability
5. **Priority 5:** Fix string output quoting - affects user experience
