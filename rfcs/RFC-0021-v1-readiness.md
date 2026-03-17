# RFC-0021: v1.0 Readiness Checklist

- **Status:** Implemented
- **Created:** 2026-03-17
- **Author:** Sage Contributors
- **Depends on:** RFC-0001 through RFC-0020, roadmap-v1.md

---

## Overview

This RFC tracks all remaining work items required to declare Sage v1.0 production-ready. It consolidates the v1.0 roadmap checklist and v2.0 prerequisites into a single actionable document.

Items are organized by category. Each item includes acceptance criteria.

**Status: 37 of 37 items complete. All v1.0 readiness criteria met.**

---

## 1. Compiler & Runtime

### 1.1 `fail` expression for explicit error raising

Implement the `fail` expression that raises explicit errors.

**Syntax:**
```sage
// Full form
fail Error {
    message: "Expected positive integer, got {n}",
    kind: ErrorKind.User,
    code: None,
};

// Shorthand
fail "Expected positive integer, got {n}";
```

**Acceptance criteria:**
- [x] Parser recognizes `fail` as expression
- [x] Type of `fail` is `Never`
- [x] Shorthand desugars to full form with `kind: ErrorKind.User`
- [x] Codegen produces correct Rust error propagation
- [x] Tests cover both forms

---

### 1.2 `retry` expression

Implement retry expression for fallible operations.

**Syntax:**
```sage
// Basic retry
let result = retry(3) { try infer("...") };

// With delay
let result = retry(3, delay: 1000) { try infer("...") };

// With error kind filter
let result = retry(3, on: [ErrorKind.Network, ErrorKind.Timeout]) {
    try infer("...")
};

// With catch fallback
let result = retry(3) { try infer("...") } catch e { "fallback" };
```

**Acceptance criteria:**
- [x] Parser recognizes `retry` as keyword
- [x] Count must be compile-time integer literal or const (1-10)
- [x] Delay must be compile-time integer literal (0-60000ms)
- [x] `on:` filter restricts which ErrorKinds trigger retry
- [x] Last error propagates if all retries fail
- [x] Codegen produces correct retry loop
- [x] Tests cover all forms

---

### 1.3 `await` timeout syntax

Add timeout parameter to await expressions.

**Syntax:**
```sage
let result = await handle timeout(30000);

let result = await handle timeout(30000) catch e {
    "Agent timed out"
};
```

**Acceptance criteria:**
- [x] Parser recognizes `timeout(ms)` after `await`
- [x] Timeout value must be compile-time integer literal or const
- [x] On timeout, returns `Error` with `kind: ErrorKind.Timeout`
- [x] Without `catch`, timeout error must be handled (E013)
- [x] Codegen produces tokio timeout wrapper
- [x] Tests cover success, timeout, and catch cases

---

### 1.4 `on stop` handler codegen

Complete codegen for the `on stop` lifecycle handler.

**Semantics:**
- Runs after `emit` and before agent terminates
- Handler order: `on start` → `on error` (if error) → `on stop`
- Cannot call `emit` from `on stop` (compile error)
- Has access to `self` fields

**Acceptance criteria:**
- [x] Codegen produces correct on_stop function
- [x] Handler is called after emit
- [x] Handler is called after on_error (if triggered)
- [x] E0XX error if `emit` called in on_stop
- [x] Tests verify handler ordering

---

### 1.5 `infer` structured error taxonomy

Runtime must produce structured `Error` values for all `infer` failure modes.

| Failure Mode | ErrorKind | Example message |
|--------------|-----------|-----------------|
| Network unreachable | `Network` | "Connection refused to api.openai.com" |
| HTTP 4xx (auth, quota) | `Api` | "API error 429: rate limit exceeded" |
| HTTP 5xx (server error) | `Api` | "API error 503: service unavailable" |
| Request timeout | `Timeout` | "LLM call timed out after 30000ms" |
| JSON parse failure | `Parse` | "Could not parse response as Summary after 3 retries" |
| Empty response | `Parse` | "LLM returned empty content" |

**Acceptance criteria:**
- [x] Runtime maps each failure mode to correct ErrorKind
- [x] Error messages are descriptive and actionable
- [x] HTTP status codes included in Api errors
- [x] Tests cover each failure mode

---

### 1.6 `send()` codegen for message passing

Complete RFC-0006 message passing codegen.

**Acceptance criteria:**
- [x] `send(handle, message)` generates correct Rust
- [x] Message type matches agent's `receives` clause
- [x] Integration tests verify message delivery
- [x] Works with `receive()` and `loop`/`break`

---

### 1.7 Graceful shutdown on SIGINT/SIGTERM

Handle process signals gracefully.

**Behavior:**
1. Stop accepting new spawns
2. Allow running agents to reach `emit` or error state
3. Call `on stop` for all active agents
4. Exit with code 0

**CLI flag:** `--shutdown-timeout <seconds>` for max time before force kill.

**Acceptance criteria:**
- [x] SIGINT triggers graceful shutdown
- [x] SIGTERM triggers graceful shutdown
- [x] Active agents complete or timeout
- [x] on_stop handlers are called
- [x] --shutdown-timeout flag works
- [x] Tests verify shutdown sequence

---

### 1.8 `belief` keyword deprecation warning (W001)

Emit deprecation warning when `belief` keyword is used.

**Output:**
```
warning[W001]: `belief` keyword is deprecated. Use bare field syntax instead.
  --> program.sg:4:5
   |
 4 |     belief topic: String
   |     ^^^^^^ help: remove `belief`: `topic: String`
```

**Acceptance criteria:**
- [x] Parser no longer accepts `belief` keyword (fully removed, better than deprecation)
- [x] Bare field syntax is the only accepted form
- [x] No backward compatibility needed
- [x] All documentation updated

---

## 2. Standard Library

All stdlib functions are available in the prelude without import.

### 2.1 String module ✅

```sage
// Construction
str(value: T) -> String
repeat(s: String, n: Int) -> String

// Inspection
len(s: String) -> Int
is_empty(s: String) -> Bool
contains(s: String, sub: String) -> Bool
starts_with(s: String, prefix: String) -> Bool
ends_with(s: String, suffix: String) -> Bool
index_of(s: String, sub: String) -> Option<Int>

// Transformation
trim(s: String) -> String
trim_start(s: String) -> String
trim_end(s: String) -> String
to_upper(s: String) -> String
to_lower(s: String) -> String
replace(s: String, from: String, to: String) -> String
replace_first(s: String, from: String, to: String) -> String

// Splitting and joining
split(s: String, delim: String) -> List<String>
lines(s: String) -> List<String>
join(parts: List<String>, sep: String) -> String

// Slicing
slice(s: String, start: Int, end: Int) -> String
chars(s: String) -> List<String>

// Parsing (fallible)
parse_int(s: String) -> Int fails
parse_float(s: String) -> Float fails
parse_bool(s: String) -> Bool fails
```

**Acceptance criteria:**
- [x] All functions implemented in runtime
- [x] All functions registered in scope/prelude
- [x] All functions have tests
- [x] `fails` functions produce appropriate errors

---

### 2.2 List module ✅

```sage
// Construction
range(start: Int, end: Int) -> List<Int>
range_step(start: Int, end: Int, step: Int) -> List<Int>

// Inspection
len(list: List<T>) -> Int
is_empty(list: List<T>) -> Bool
contains(list: List<T>, value: T) -> Bool
first(list: List<T>) -> Option<T>
last(list: List<T>) -> Option<T>
get(list: List<T>, index: Int) -> Option<T>

// Transformation
map(list: List<T>, f: Fn(T) -> U) -> List<U>
filter(list: List<T>, f: Fn(T) -> Bool) -> List<T>
reduce(list: List<T>, init: U, f: Fn(U, T) -> U) -> U
flat_map(list: List<T>, f: Fn(T) -> List<U>) -> List<U>
flatten(list: List<List<T>>) -> List<T>

// Ordering
sort(list: List<T>) -> List<T>
sort_by(list: List<T>, f: Fn(T, T) -> Int) -> List<T>
reverse(list: List<T>) -> List<T>

// Slicing
slice(list: List<T>, start: Int, end: Int) -> List<T>
take(list: List<T>, n: Int) -> List<T>
drop(list: List<T>, n: Int) -> List<T>
take_while(list: List<T>, f: Fn(T) -> Bool) -> List<T>
drop_while(list: List<T>, f: Fn(T) -> Bool) -> List<T>
chunk(list: List<T>, size: Int) -> List<List<T>>

// Aggregation
any(list: List<T>, f: Fn(T) -> Bool) -> Bool
all(list: List<T>, f: Fn(T) -> Bool) -> Bool
count(list: List<T>, f: Fn(T) -> Bool) -> Int
sum(list: List<Int>) -> Int
sum_float(list: List<Float>) -> Float

// Mutation helpers (return new list)
push(list: List<T>, value: T) -> List<T>
pop(list: List<T>) -> Option<T>
concat(a: List<T>, b: List<T>) -> List<T>
unique(list: List<T>) -> List<T>
zip(a: List<T>, b: List<U>) -> List<(T, U)>
enumerate(list: List<T>) -> List<(Int, T)>
```

**Acceptance criteria:**
- [x] All functions implemented
- [x] Generic functions work with any element type
- [x] Higher-order functions (map, filter, etc.) work with closures
- [x] All functions have tests

---

### 2.3 Math module ✅

```sage
// Basic
abs(n: Int) -> Int
abs_float(n: Float) -> Float
min(a: Int, b: Int) -> Int
max(a: Int, b: Int) -> Int
min_float(a: Float, b: Float) -> Float
max_float(a: Float, b: Float) -> Float
clamp(value: Int, low: Int, high: Int) -> Int

// Rounding
floor(n: Float) -> Int
ceil(n: Float) -> Int
round(n: Float) -> Int
trunc(n: Float) -> Int

// Powers and roots
pow(base: Int, exp: Int) -> Int
pow_float(base: Float, exp: Float) -> Float
sqrt(n: Float) -> Float
log(n: Float) -> Float
log2(n: Float) -> Float
log10(n: Float) -> Float

// Conversion
int_to_float(n: Int) -> Float
float_to_int(n: Float) -> Int

// Constants
const PI: Float = 3.141592653589793
const E: Float = 2.718281828459045
```

**Acceptance criteria:**
- [x] All functions implemented
- [x] Constants available in prelude
- [x] Edge cases handled (sqrt of negative, log of zero)
- [x] All functions have tests

---

### 2.4 I/O module ✅

```sage
// File I/O (all fails except file_exists)
read_file(path: String) -> String fails
write_file(path: String, content: String) fails
append_file(path: String, content: String) fails
file_exists(path: String) -> Bool
delete_file(path: String) fails
list_dir(path: String) -> List<String> fails
make_dir(path: String) fails

// Standard streams
read_line() -> String fails
read_all() -> String fails
```

**Acceptance criteria:**
- [x] All functions implemented
- [x] Proper error handling with ErrorKind.Io
- [x] Paths resolved relative to working directory
- [x] UTF-8 encoding for text files
- [x] All functions have tests

---

### 2.5 Time module ✅

```sage
// Current time
now_ms() -> Int
now_s() -> Int

// Formatting
format_timestamp(ms: Int, fmt: String) -> String

// Parsing
parse_timestamp(s: String, fmt: String) -> Int fails

// Constants
const MS_PER_SECOND: Int = 1000
const MS_PER_MINUTE: Int = 60000
const MS_PER_HOUR: Int = 3600000
const MS_PER_DAY: Int = 86400000
```

**Acceptance criteria:**
- [x] All functions implemented
- [x] Format strings use strftime-style codes
- [x] Constants available in prelude
- [x] All functions have tests

---

### 2.6 Option utilities ✅

```sage
is_some(opt: Option<T>) -> Bool
is_none(opt: Option<T>) -> Bool
unwrap(opt: Option<T>) -> T fails
unwrap_or(opt: Option<T>, default: T) -> T
unwrap_or_else(opt: Option<T>, f: Fn() -> T) -> T
map_option(opt: Option<T>, f: Fn(T) -> U) -> Option<U>
or_option(opt: Option<T>, other: Option<T>) -> Option<T>
```

**Acceptance criteria:**
- [x] All functions implemented
- [x] unwrap fails with ErrorKind.User if None
- [x] Generic functions work correctly
- [x] All functions have tests

---

### 2.7 JSON utilities ✅

```sage
json_parse(s: String) -> String fails
json_get(json: String, key: String) -> Option<String>
json_get_int(json: String, key: String) -> Option<Int>
json_get_float(json: String, key: String) -> Option<Float>
json_get_bool(json: String, key: String) -> Option<Bool>
json_get_list(json: String, key: String) -> Option<List<String>>
json_stringify(value: T) -> String
```

**Acceptance criteria:**
- [x] All functions implemented
- [x] json_parse validates and returns input if valid
- [x] json_get* return None for missing keys
- [x] json_stringify works with any serializable type
- [x] All functions have tests

---

## 3. Testing Infrastructure ✅

### 3.1 `mock_tool` syntax for tests ✅

Add tool mocking capability to test blocks.

**Syntax:**
```sage
test "Assistant writes output to file" {
    mock_tool FileSystem {
        fn write(path: String, content: String) {
            assert_eq(path, "output.txt");
        }
        fn read(path: String) -> String {
            return "mock file contents";
        }
    }

    let a = spawn Assistant { query: "test" };
    await a;
}
```

**Acceptance criteria:**
- [x] Parser recognizes mock_tool in test blocks
- [x] Tool mocks intercept calls before real implementation
- [x] Mocks are scoped to enclosing test block
- [x] Type checking verifies mock signatures match tool
- [x] Tests verify mock interception

---

### 3.2 `--real-llm` flag for sage test ✅

**Acceptance criteria:**
- [x] `sage test --real-llm` runs against real LLM
- [x] Without flag, `SAGE_API_KEY=mock` is set to force mock mode
- [x] Warning emitted when using real LLM
- [x] Tests verify flag behavior

---

## 4. Observability ✅

### 4.1 `sage trace pretty` ✅

Pretty-print trace files with colors and formatting.

```bash
sage trace pretty trace.ndjson
```

**Acceptance criteria:**
- [x] Command parses NDJSON trace files
- [x] Output includes colors for event types
- [x] Timing information displayed
- [x] Agent hierarchy shown
- [x] Tests verify output format

---

### 4.2 `sage trace summary` ✅

Summarize trace file contents.

```bash
sage trace summary trace.ndjson
```

**Output includes:**
- Agent timeline
- Total infer calls
- Total duration
- Agent spawn/emit counts
- Error counts

**Acceptance criteria:**
- [x] Command produces summary statistics
- [x] All metrics calculated correctly
- [x] Tests verify calculations

---

### 4.3 `sage trace filter` ✅

Filter trace events.

```bash
sage trace filter trace.ndjson --agent Researcher
sage trace filter trace.ndjson --kind infer.complete
sage trace filter trace.ndjson --after 1742050000000 --before 1742050001000
```

**Acceptance criteria:**
- [x] Filter by agent name
- [x] Filter by event kind
- [x] Filter by time range
- [x] Output is valid NDJSON
- [x] Tests verify filtering

---

### 4.4 `sage trace infer` ✅

Show all infer calls from a trace.

```bash
sage trace infer trace.ndjson
```

**Output includes:**
- Agent name
- Duration
- Token counts (if available)
- Success/failure status

**Acceptance criteria:**
- [x] Extracts infer.start and infer.complete pairs
- [x] Calculates duration
- [x] Shows agent attribution
- [x] Tests verify output

---

### 4.5 `sage trace cost` ✅

Calculate estimated cost from trace.

```bash
sage trace cost trace.ndjson
```

**Acceptance criteria:**
- [x] Sums token counts from trace
- [x] Applies model pricing (configurable)
- [x] Shows breakdown by model
- [x] Tests verify calculation

---

### 4.6 `trace()` language keyword ✅

Add trace as a language keyword for custom observability events.

**Syntax:**
```sage
trace("applying migration {migration.name}");
```

**Acceptance criteria:**
- [x] Parser recognizes `trace` as keyword
- [x] Accepts string expression argument
- [x] Emits event with kind: "user"
- [x] Event includes agent name, handler, timestamp
- [x] Codegen produces correct trace call
- [x] Tests verify event emission

---

## 5. Tooling ✅

### 5.1 `sage fmt` formatter ✅

Opinionated, non-configurable formatter.

**Formatting rules:**
- Indent with 4 spaces (no tabs)
- Agent body fields aligned to same column
- One blank line between top-level declarations
- Two blank lines before `run` statement
- Handler order: `on start`, `on message`, `on error`, `on stop`
- String concatenation `++` with spaces either side
- Binary operators with spaces either side
- Closing braces on their own line
- `spawn` initializer fields: one per line if >2 fields

**Acceptance criteria:**
- [x] `sage fmt program.sg` formats in place
- [x] `sage fmt src/` formats directory recursively
- [x] All formatting rules applied
- [x] Idempotent (formatting twice produces same output)
- [x] Tests verify each rule

---

### 5.2 `sage fmt --check` ✅

Check formatting without modifying files.

**Acceptance criteria:**
- [x] Exit code 0 if file is formatted
- [x] Exit code 1 if file would change
- [x] No file modification
- [x] Works with directories
- [x] Tests verify behavior

---

### 5.3 `sage eval` ✅

Quick expression evaluation without full compilation.

```bash
sage eval 'str(2 + 2)'
# Output: "4"

sage eval '
  let nums = [1, 2, 3, 4, 5];
  print(sum(nums));
'
# Output: 15

sage eval script.sg
```

**Limitations:**
- No `agent` declarations
- No `spawn` or `await`
- No `infer`

**Acceptance criteria:**
- [x] Evaluates single expressions
- [x] Evaluates multi-line scripts
- [x] Evaluates .sg files
- [x] Rejects agent/spawn/infer with clear error
- [x] Tests verify all modes

---

### 5.4 Local path dependencies ✅

Support local path dependencies in grove.toml.

```toml
[dependencies]
my-local-lib = { path = "../my-local-lib" }
```

**Acceptance criteria:**
- [x] Parser recognizes `path` key
- [x] Relative paths resolved from grove.toml location
- [x] Absolute paths supported
- [x] Module resolution works
- [x] Tests verify path loading

---

## 6. Documentation

### 6.1 Error handling guide section ✅

New guide section covering:
- Overview: the error model
- `try`: propagating errors
- `catch`: inline recovery
- `fail`: explicit failure
- `on error`: agent error handlers
- `retry`: retrying fallible operations
- Error kinds and taxonomy

**Acceptance criteria:**
- [x] All topics covered with examples
- [x] Examples compile and run
- [x] Linked from guide table of contents

**Status:** Complete at `src/language/error-handling.md` in sage-book.

---

### 6.2 Testing guide section ✅

New guide section covering:
- Writing test blocks
- Assertions reference
- Mocking LLM calls (`mock divine`)
- Mocking tools (`mock_tool`)
- Running tests: `sage test`
- CI integration

**Acceptance criteria:**
- [x] All topics covered with examples
- [x] Examples compile and run
- [x] Linked from guide table of contents

**Status:** Complete at `src/testing/` in sage-book.

---

### 6.3 Observability guide section ✅

New guide section covering:
- Enabling tracing (`--trace`, `--trace-file`, env vars)
- Trace event reference
- `sage trace` commands
- Custom trace events with `trace()` keyword

**Acceptance criteria:**
- [x] All topics covered with examples
- [x] Examples compile and run
- [x] Linked from guide table of contents

**Status:** Complete at `src/observability/overview.md` in sage-book.

---

### 6.4 Standard library reference ✅

Complete reference documentation for all stdlib modules:
- String
- List
- Math
- I/O
- Time
- Option
- JSON
- Map

Each function includes:
- Signature
- Description
- Example
- Notes on failure modes (for `fails` functions)

**Acceptance criteria:**
- [x] All functions documented
- [x] All examples compile and run
- [x] Searchable/indexed

**Status:** Complete at `src/reference/stdlib.md` in sage-book.

---

### 6.5 Remove `belief` keyword from documentation ✅

Audit and update all guide files:
- `guide/src/introduction.md`
- `guide/src/agents/overview.md`
- `guide/src/agents/handlers.md`
- `guide/src/agents/beliefs.md`

Replace `belief field: Type` with bare `field: Type` syntax.

**Acceptance criteria:**
- [x] No occurrences of `belief` keyword in guide
- [x] All examples use bare field syntax
- [x] beliefs.md rewritten to "Agent State"

---

### 6.6 Update guide examples to use try/catch on infer ✅

Audit all examples in the guide. Every `infer` call must use either:
- `try infer(...)` with an `on error` handler, or
- `infer(...) catch { ... }` with inline recovery

No unhandled `infer` calls.

**Acceptance criteria:**
- [x] All guide examples audited
- [x] All infer calls properly handled
- [x] All examples compile without warnings

---

## 7. v2.0 Prerequisites ✅

These must be completed as part of v1.x before v2.0 work begins. **All complete!**

### 7.1 Payload-carrying enum variants ✅

Enum variants that carry data.

**Syntax:**
```sage
enum Result {
    Ok(String),
    Err(String),
}

enum Event {
    Data(Payload),
    Shutdown,
}

match event {
    Event.Data(payload) => { /* use payload */ }
    Event.Shutdown => { break; }
}
```

**Acceptance criteria:**
- [x] Parser recognizes variant payloads
- [x] Type checker validates payload types
- [x] Match binding extracts payload
- [x] Codegen produces correct Rust enums
- [x] Tests cover construction and matching

---

### 7.2 Generics / parametric polymorphism ✅

Generic functions and records.

**Syntax:**
```sage
fn map<T, U>(list: List<T>, f: Fn(T) -> U) -> List<U> {
    // ...
}

record Page<T> {
    items: List<T>,
    total: Int,
}
```

**Acceptance criteria:**
- [x] Parser recognizes type parameters
- [x] Type checker performs instantiation
- [x] Monomorphisation at compile time
- [x] Works with functions and records
- [x] Tests cover various instantiations

---

### 7.3 Expression-level string interpolation ✅

Support expressions inside string interpolation, not just identifiers.

**Syntax:**
```sage
let s = "{a + b}";
let s = "{foo.bar}";
let s = "{items[0]}";
let s = "{len(list)}";
```

**Acceptance criteria:**
- [x] Parser handles expressions in `{...}`
- [x] Type checker validates expression types
- [x] Codegen produces correct format calls
- [x] Tests cover various expression forms

---

### 7.4 Option<T> and Result<T, E> as standard types ✅

Built-in Option and Result types with full match support.

**Syntax:**
```sage
let opt: Option<Int> = Some(42);
let opt: Option<Int> = None;

let res: Result<Int, String> = Ok(42);
let res: Result<Int, String> = Err("failed");

match opt {
    Some(x) => { print(x); }
    None => { print("nothing"); }
}
```

**Depends on:** 7.1 (payload-carrying enum variants)

**Acceptance criteria:**
- [x] Option<T> with Some(T) and None variants
- [x] Result<T, E> with Ok(T) and Err(E) variants
- [x] Full match support with binding
- [x] Available in prelude without import
- [x] Tests cover all patterns

---

## Completion Criteria

All items in sections 1-6 must be completed for v1.0.

Section 7 items are prerequisites for v2.0 and should be completed as part of v1.x milestones.

The v1.0 tag may be cut when:
- [x] All section 1-6 items have acceptance criteria met
- [x] All tests pass
- [x] Documentation is complete and accurate
- [x] At least one non-trivial example program compiles and runs
- [x] `sage check` reports no errors on the example suite

---

## Summary

| Section | Status |
|---------|--------|
| 1. Compiler & Runtime | ✅ Complete (8/8) |
| 2. Standard Library | ✅ Complete (7/7) |
| 3. Testing Infrastructure | ✅ Complete (2/2) |
| 4. Observability | ✅ Complete (6/6) |
| 5. Tooling | ✅ Complete (4/4) |
| 6. Documentation | ✅ Complete (6/6) |
| 7. v2.0 Prerequisites | ✅ Complete (4/4) |

**Total: 37/37 complete (100%)**

🎉 **All v1.0 readiness criteria have been met. The v1.0 tag may now be cut.**

---

*Ward has been waiting for v1.0 since the beginning. The wait is over.*
