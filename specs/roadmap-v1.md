# RFC-0010: Sage v1.0 — Production Readiness Specification

- **Status:** Draft
- **Created:** 2026-03-15
- **Author:** Sage Contributors
- **Depends on:** RFC-0001, RFC-0002, RFC-0003, RFC-0005, RFC-0006, RFC-0007, RFC-0009

---

## Table of Contents

1. [Overview](#1-overview)
2. [Scope & Philosophy](#2-scope--philosophy)
3. [Error Handling — End-to-End](#3-error-handling--end-to-end)
4. [Standard Library](#4-standard-library)
5. [Agent Supervision & Reliability](#5-agent-supervision--reliability)
6. [Tool & MCP Integration](#6-tool--mcp-integration)
7. [Observability](#7-observability)
8. [Testing Infrastructure](#8-testing-infrastructure)
9. [Tooling — LSP, Formatter, REPL](#9-tooling--lsp-formatter-repl)
10. [Package Ecosystem](#10-package-ecosystem)
11. [Documentation Consistency](#11-documentation-consistency)
12. [v1.0 Completion Checklist](#12-v10-completion-checklist)
13. [Out of Scope for v1.0](#13-out-of-scope-for-v10)

---

## 1. Overview

Sage v0.3.0 has demonstrated the core thesis: agents as first-class language primitives produce programs that are simpler and more readable than equivalent Python framework code. The compiler pipeline (lexer → parser → type checker → Rust codegen) is sound. The core abstractions — `agent`, `spawn`, `await`, `emit`, `infer`, `send`, `receive` — are implemented and working.

**v1.0 is not about adding new abstractions. It is about making the existing ones trustworthy.**

A language is production-ready when a developer can build a system with it and sleep at night. That requires five things:

1. **Failures are handled, not panicked on.** LLM calls fail. Networks fail. Models return garbage. The language must give the programmer clean tools to express what happens when things go wrong.
2. **The runtime is observable.** You cannot trust what you cannot see. Production agent programs must be debuggable — what prompt went out, what came back, which agent failed, why.
3. **Programs can be tested without real LLMs.** Nondeterministic, expensive, slow tests are not tests. The language must provide a first-class mock/test infrastructure.
4. **The development experience is professional.** IDE integration, a formatter, and a debuggable development loop are not luxuries for a language targeting professional developers.
5. **Programs can be composed and reused.** A single-file scripting language cannot build real systems. The module system exists; the package ecosystem must follow.

This document specifies all work required to call Sage 1.0.

---

## 2. Scope & Philosophy

### What v1.0 Is

A complete, trustworthy, developer-ready language for building AI agent systems. Every feature in this RFC should be:

- Fully implemented in the compiler and runtime
- Documented in the guide with examples
- Covered by tests in the standard test suite
- Exercised in at least one example program

### What v1.0 Is Not

v1.0 is not a general-purpose language. Sage remains a domain-specific language for AI agent systems. The following are explicitly deferred:

- Generics / parametric polymorphism
- Algebraic effects
- Session types
- WASM compilation target
- Advanced type inference (Hindley-Milner)
- Operator overloading
- Macros / metaprogramming

These may be appropriate for v2.0 once the agent model has been validated in production.

### Naming Convention for This RFC

Feature areas are given numbered section headings that correspond to sub-RFCs where needed. Where a feature is small enough to specify fully here, no sub-RFC is needed.

---

## 3. Error Handling — End-to-End

**Current state:** RFC-0007 introduced `try`, `catch`, `fails`, and `on error` at the checker and partial codegen level. The `infer` expression panics on all failures. `on stop` is unimplemented. `await` has no timeout. The documentation does not teach error handling at all.

**v1.0 requirement:** A complete, documented, end-to-end error handling story. Programs must not panic on expected failures.

---

### 3.1 The `Error` Type

Sage's built-in `Error` type carries structured information:

```sage
// Built-in. Not user-definable in v1.0.
record Error {
    message: String        // Human-readable description
    kind: ErrorKind        // Categorized failure type
    code: Option<Int>      // HTTP status code, exit code, etc. if applicable
}

enum ErrorKind {
    Network,               // Transport failure (timeout, DNS, TLS)
    Api,                   // Provider returned an error response
    Parse,                 // LLM output could not be parsed to the target type
    Timeout,               // Operation exceeded its deadline
    Agent,                 // Spawned agent failed
    Io,                    // File / stdin / stdout failure
    User,                  // Explicitly raised by sage code via `fail`
    Unknown,               // Catch-all for unclassified errors
}
```

The `Error` type is available in all scopes without import. It is the only type that can be propagated via `try` or caught via `catch`.

---

### 3.2 Fallible Functions and Annotations

A function that may fail is annotated `fails`:

```sage
fn parse_int(s: String) -> Int fails {
    // ...
}
```

Calling a `fails` function without handling the error is a compile-time error (E013, already defined in RFC-0007).

`infer` is implicitly `fails`. It does not require annotation because the compiler knows it is always fallible. The following are equivalent:

```sage
let result: Inferred<String> = infer("...");          // implicit — compiler error if unhandled
let result: Inferred<String> = try infer("...");      // explicit propagation
let result: Inferred<String> = infer("...") catch { "default" };  // explicit recovery
```

For v1.0, **every `infer` call in the guide and examples must use either `try` or `catch`**. The implicit (unhandled) form produces a compiler warning in v1.0 and will become a hard error in v2.0.

---

### 3.3 `try` — Propagation

`try expr` evaluates `expr`. If it succeeds, it returns the value. If it fails, it propagates the `Error` up the call stack.

In an agent handler, propagated errors are caught by the agent's `on error` handler:

```sage
agent Researcher {
    topic: String

    on start {
        let summary: Inferred<String> = try infer(
            "Summarize: {self.topic}"
        );
        emit(summary);
    }

    on error(e: Error) {
        print("Researcher failed: {e.message}");
        emit("(no summary available)");
    }
}
```

If an agent uses `try` but has no `on error` handler, the compiler emits E016 (already defined in RFC-0007).

In a `fails` function, `try` propagates to the caller:

```sage
fn fetch_summary(topic: String) -> String fails {
    let result: Inferred<String> = try infer("Summarize: {topic}");
    return result;
}
```

---

### 3.4 `catch` — Inline Recovery

`catch` handles an error at the call site and provides a recovery value of the same type:

```sage
// Recovery with a default value
let summary: String = infer("Summarize: {topic}") catch { "Summary unavailable." };

// Recovery with access to the error
let summary: String = infer("Summarize: {topic}") catch e { "Error: {e.message}" };

// Recovery with a block
let summary: String = infer("Summarize: {topic}") catch e {
    print("Falling back due to: {e.kind}");
    "Fallback summary."
};
```

The recovery expression must have the same type as the fallible expression. Type mismatch is E015.

---

### 3.5 `fail` — Explicit Failure

Programs can raise explicit errors using the `fail` expression:

```sage
fn require_positive(n: Int) -> Int fails {
    if n <= 0 {
        fail Error {
            message: "Expected positive integer, got {n}",
            kind: ErrorKind.User,
            code: None,
        };
    }
    return n;
}
```

`fail` is an expression of type `Never` — it never returns. It can appear in any expression position.

A shorthand form omits the `Error` record literal when only a message is needed:

```sage
fail "Expected positive integer, got {n}";
// Equivalent to:
fail Error { message: "Expected positive integer, got {n}", kind: ErrorKind.User, code: None };
```

---

### 3.6 `on error` Handler

Defined in RFC-0007. v1.0 completion requirements:

- Fully implemented in codegen (currently partial)
- The handler receives an `Error` value
- After `on error` runs, the agent is considered terminated
- If `on error` calls `emit`, that becomes the agent's result
- If `on error` does not call `emit`, the agent emits `Unit` and any awaiting agents receive an error

```sage
agent Worker {
    on start {
        let data: Inferred<String> = try infer("...");
        emit(data);
    }

    on error(e: Error) {
        // Called if try propagates an error
        print("Worker failed: {e.message}");
        emit("fallback");    // Optional: provide a result anyway
    }
}
```

---

### 3.7 `on stop` Handler — Implementation Required

`on stop` is documented but marked "not yet implemented." v1.0 requires full implementation.

`on stop` runs after `emit` and before the agent terminates. It is intended for cleanup — flushing buffers, logging, releasing resources.

```sage
agent FileProcessor {
    path: String

    on start {
        let content: String = try read_file(self.path);
        let result: Inferred<String> = try infer("Process: {content}");
        emit(result);
    }

    on stop {
        print("FileProcessor for {self.path} completed.");
    }

    on error(e: Error) {
        print("FileProcessor failed: {e.message}");
        emit("error");
    }
}
```

Handler order guarantee: `on start` → (optionally) `on error` → `on stop`.

---

### 3.8 `await` with Timeout

`await handle` blocks indefinitely. This is unacceptable for production. v1.0 introduces timeout syntax:

```sage
// Await with a timeout in milliseconds
let result: String = await r timeout(30000);
// Type: String — if the agent succeeds

// If timeout expires, the result is an Error with kind: ErrorKind.Timeout
let result: String = await r timeout(30000) catch e {
    "Agent timed out after 30s"
};
```

Without `catch`, a timeout propagates as an error and must be handled.

The timeout value must be a compile-time integer literal or a `const` integer. Dynamic timeouts are deferred to v2.0.

---

### 3.9 `infer` Error Taxonomy

The runtime must produce structured errors for all `infer` failure modes. Each maps to an `ErrorKind`:

| Failure Mode | `ErrorKind` | `message` example |
|---|---|---|
| Network unreachable | `Network` | `"Connection refused to api.openai.com"` |
| HTTP 4xx (auth, quota) | `Api` | `"API error 429: rate limit exceeded"` |
| HTTP 5xx (server error) | `Api` | `"API error 503: service unavailable"` |
| Request timeout | `Timeout` | `"LLM call timed out after 30000ms"` |
| JSON parse failure after retries | `Parse` | `"Could not parse response as Summary after 3 retries"` |
| Empty response | `Parse` | `"LLM returned empty content"` |

These error kinds allow programs to implement differentiated retry logic:

```sage
let result = infer("...") catch e {
    match e.kind {
        ErrorKind.Timeout => try infer("..."),       // retry once on timeout
        ErrorKind.Parse   => "unparseable response",  // give up on parse errors
        _                 => fail e,                  // propagate everything else
    }
};
```

---

## 4. Standard Library

**Current state:** The prelude contains `print`, `str`, `len`, `sleep_ms`, and minimal list operations. This is insufficient for any non-trivial program.

**v1.0 requirement:** A complete standard library covering strings, lists, math, I/O, and time. All stdlib functions are available without import (they are part of the prelude).

---

### 4.1 String Module

```sage
// Construction
str(value: T) -> String                    // Convert any primitive to string
repeat(s: String, n: Int) -> String        // "ha".repeat(3) = "hahaha"

// Inspection
len(s: String) -> Int                      // Character count
is_empty(s: String) -> Bool
contains(s: String, sub: String) -> Bool
starts_with(s: String, prefix: String) -> Bool
ends_with(s: String, suffix: String) -> Bool
index_of(s: String, sub: String) -> Option<Int>

// Transformation
trim(s: String) -> String                  // Remove leading/trailing whitespace
trim_start(s: String) -> String
trim_end(s: String) -> String
to_upper(s: String) -> String
to_lower(s: String) -> String
replace(s: String, from: String, to: String) -> String
replace_first(s: String, from: String, to: String) -> String

// Splitting and joining
split(s: String, delim: String) -> List<String>
lines(s: String) -> List<String>           // Split on newlines
join(parts: List<String>, sep: String) -> String

// Slicing
slice(s: String, start: Int, end: Int) -> String  // End is exclusive
chars(s: String) -> List<String>           // Split into individual characters

// Parsing (fallible)
parse_int(s: String) -> Int fails
parse_float(s: String) -> Float fails
parse_bool(s: String) -> Bool fails        // "true"/"false" case-insensitive
```

---

### 4.2 List Module

```sage
// Construction
range(start: Int, end: Int) -> List<Int>   // Exclusive end: range(0, 3) = [0, 1, 2]
range_step(start: Int, end: Int, step: Int) -> List<Int>

// Inspection
len(list: List<T>) -> Int
is_empty(list: List<T>) -> Bool
contains(list: List<T>, value: T) -> Bool
first(list: List<T>) -> Option<T>
last(list: List<T>) -> Option<T>
get(list: List<T>, index: Int) -> Option<T>

// Transformation (return new list — Sage lists are immutable)
map(list: List<T>, f: Fn(T) -> U) -> List<U>
filter(list: List<T>, f: Fn(T) -> Bool) -> List<T>
reduce(list: List<T>, init: U, f: Fn(U, T) -> U) -> U
flat_map(list: List<T>, f: Fn(T) -> List<U>) -> List<U>
flatten(list: List<List<T>>) -> List<T>

// Ordering
sort(list: List<T>) -> List<T>             // T must be Int, Float, or String
sort_by(list: List<T>, f: Fn(T, T) -> Int) -> List<T>  // f returns -1, 0, or 1
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
pop(list: List<T>) -> Option<T>            // Returns last element
concat(a: List<T>, b: List<T>) -> List<T>
unique(list: List<T>) -> List<T>           // Remove duplicates, preserve order
zip(a: List<T>, b: List<U>) -> List<List<T|U>>  // Pairs as 2-element lists
enumerate(list: List<T>) -> List<List<T>>  // [[0, item], [1, item], ...]
```

---

### 4.3 Math Module

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
log(n: Float) -> Float                     // Natural log
log2(n: Float) -> Float
log10(n: Float) -> Float

// Conversion
int_to_float(n: Int) -> Float
float_to_int(n: Float) -> Int              // Truncates toward zero

// Constants
const PI: Float = 3.141592653589793
const E: Float  = 2.718281828459045
```

---

### 4.4 I/O Module

All I/O functions are `fails` — callers must handle errors.

```sage
// File I/O
read_file(path: String) -> String fails        // Reads UTF-8 text
write_file(path: String, content: String) fails
append_file(path: String, content: String) fails
file_exists(path: String) -> Bool              // Does not fail — returns false on error
delete_file(path: String) fails
list_dir(path: String) -> List<String> fails   // Returns filenames (not full paths)
make_dir(path: String) fails                   // Creates directory and parents

// Standard streams
read_line() -> String fails                    // Read one line from stdin (strips newline)
read_all() -> String fails                     // Read all of stdin

// Printing (already in prelude, listed here for completeness)
print(value: T)                                // Prints with newline
print_err(value: T)                            // Prints to stderr with newline
```

**Path handling:** Paths are plain strings. Sage does not have a dedicated `Path` type in v1.0. Relative paths are resolved against the working directory of the running process.

---

### 4.5 Time Module

```sage
// Current time
now_ms() -> Int                            // Unix timestamp in milliseconds
now_s() -> Int                             // Unix timestamp in seconds

// Formatting
format_timestamp(ms: Int, fmt: String) -> String
// fmt uses strftime-style codes: "%Y-%m-%d %H:%M:%S"

// Parsing
parse_timestamp(s: String, fmt: String) -> Int fails  // Returns ms

// Duration helpers (just arithmetic conveniences)
const MS_PER_SECOND: Int = 1000
const MS_PER_MINUTE: Int = 60000
const MS_PER_HOUR: Int   = 3600000
const MS_PER_DAY: Int    = 86400000
```

---

### 4.6 Option Utilities

`Option<T>` is used throughout the stdlib. These helpers make working with it ergonomic:

```sage
// Already in prelude:
is_some(opt: Option<T>) -> Bool
is_none(opt: Option<T>) -> Bool
unwrap(opt: Option<T>) -> T fails          // Fails with ErrorKind.User if None
unwrap_or(opt: Option<T>, default: T) -> T
unwrap_or_else(opt: Option<T>, f: Fn() -> T) -> T
map_option(opt: Option<T>, f: Fn(T) -> U) -> Option<U>
or_option(opt: Option<T>, other: Option<T>) -> Option<T>
```

---

### 4.7 JSON Utilities

LLM output manipulation frequently requires JSON. These are stdlib functions, not language primitives.

```sage
json_parse(s: String) -> String fails      // Validates JSON; returns the input if valid
json_get(json: String, key: String) -> Option<String>   // Extract field as string
json_get_int(json: String, key: String) -> Option<Int>
json_get_float(json: String, key: String) -> Option<Float>
json_get_bool(json: String, key: String) -> Option<Bool>
json_get_list(json: String, key: String) -> Option<List<String>>
json_stringify(value: T) -> String         // Serialize any value to JSON
```

Note: these are intentionally limited. They operate on JSON strings, not a typed JSON value tree. Full JSON manipulation is better done at the `Inferred<RecordType>` level.

---

## 5. Agent Supervision & Reliability

**Current state:** No supervision. Panicking agents propagate errors to awaiters. No restart strategies. `on stop` unimplemented.

**v1.0 requirement:** A practical, usable supervision model. OTP-style supervision trees are post-v1.0; v1.0 delivers a well-defined failure contract and the primitives needed to build reliable pipelines.

---

### 5.1 Agent Failure Contract

When a spawned agent fails (unhandled error in `on start`, or `on error` completes without `emit`), the failure propagates to the awaiting agent as an `Error` with `kind: ErrorKind.Agent`. The `message` field contains the original error message prefixed with the agent name.

```sage
agent Main {
    on start {
        let r = spawn Researcher { topic: "AI" };

        // Without error handling — compile error if Researcher has on error
        // that may not emit, since the result type becomes ambiguous.

        // Correct: handle the potential failure
        let result: String = await r catch e {
            "Research failed: {e.message}"
        };

        print(result);
        emit(0);
    }
}
```

The compiler tracks whether a spawned agent's `on error` handler may not call `emit`. If so, `await` of that agent is treated as fallible and requires error handling. This is enforced as a compiler warning in v1.0 and a hard error in v2.0.

---

### 5.2 Retry Primitive

A built-in `retry` expression retries a block of code on failure:

```sage
// Retry up to 3 times with no delay
let result: String = retry(3) {
    try infer("Summarize: {topic}")
};

// Retry with delay between attempts (milliseconds)
let result: String = retry(3, delay: 1000) {
    try infer("Summarize: {topic}")
};

// Retry only on specific error kinds
let result: String = retry(3, on: [ErrorKind.Network, ErrorKind.Timeout]) {
    try infer("Summarize: {topic}")
};

// Catch after all retries exhausted
let result: String = retry(3) {
    try infer("Summarize: {topic}")
} catch e {
    "All retries failed: {e.message}"
};
```

`retry` is a language keyword, not a function. The block is re-executed on failure. If all retries fail, the last error is propagated.

**Constraints:**
- Retry count must be a compile-time integer literal or `const` (1–10 inclusive).
- Delay must be a compile-time integer literal (0–60000ms).
- The block must be a single expression or a block expression.

---

### 5.3 `on stop` — Full Implementation

As specified in section 3.7. Additional guarantees for v1.0:

- `on stop` is called even when an error handler terminates the agent
- `on stop` has access to `self` (agent state)
- `on stop` cannot call `emit` (doing so is a compile-time error)
- `on stop` should not be `fails` — errors in `on stop` are logged to stderr and ignored

---

### 5.4 Graceful Shutdown

The `sage run` command must handle SIGINT and SIGTERM gracefully:

1. Stop accepting new spawns
2. Allow currently-running agents to reach their next `emit` or error state
3. Call `on stop` for all active agents
4. Exit with code 0

A `--shutdown-timeout` flag gives a maximum time (in seconds) before forceful termination.

---

### 5.5 Agent Pools (Deferred to v1.1)

Agent pools for load distribution are important for production but complex to specify correctly. They are deferred to v1.1.

---

## 6. Tool & MCP Integration

**Current state:** Not implemented. The VISION doc identifies this as a critical gap.

**v1.0 requirement:** First-class `tool` declarations with MCP support.

This is the highest-impact feature for expanding Sage beyond the multi-agent researcher demo pattern. The dominant production AI pattern is "single agent + tools." Without tools, Sage is not viable for the most common production use case today.

---

### 6.1 Tool Declarations

A `tool` is a typed interface to an external capability. Tools are declared at the top level:

```sage
tool FileSystem {
    fn read(path: String) -> String fails
    fn write(path: String, content: String) fails
    fn exists(path: String) -> Bool
    fn list(dir: String) -> List<String> fails
}

tool WebSearch {
    fn search(query: String) -> List<SearchResult> fails
    fn fetch(url: String) -> String fails
}

record SearchResult {
    title: String
    url: String
    snippet: String
}
```

Tool declarations are types — they describe what a tool can do. They are not implementations. Implementations are provided at runtime via configuration or MCP servers.

---

### 6.2 Using Tools in Agents

Agents declare which tools they use with a `uses` clause:

```sage
agent Assistant uses FileSystem, WebSearch {
    query: String

    on start {
        let results: List<SearchResult> = try WebSearch.search(self.query);
        let top = try first(results);

        let content: String = try WebSearch.fetch(top.url);

        let summary: Inferred<String> = try infer(
            "Summarize this content in 3 bullet points:\n{content}"
        );

        try FileSystem.write("output.txt", summary);
        emit(summary);
    }

    on error(e: Error) {
        print("Assistant failed: {e.message}");
        emit("(no output)");
    }
}

run Assistant { query: "latest developments in fusion energy" };
```

Tool calls are always `fails` — they involve external I/O and can always fail.

---

### 6.3 MCP Protocol Support

Tool implementations can be provided via MCP servers. Configuration is in `sage.toml`:

```toml
[project]
name = "my_project"
version = "1.0.0"

[tools]
FileSystem = { mcp = "https://filesystem.mcp.example.com" }
WebSearch  = { mcp = "https://search.mcp.example.com" }
Database   = { mcp = "http://localhost:8080/mcp" }
```

The Sage runtime connects to MCP servers at startup and verifies that the declared tool interface is compatible with the server's capabilities. Incompatible schemas produce a startup error, not a runtime panic.

**CLI tool discovery:**

```bash
# List tools available from configured MCP servers
sage tools list

# Inspect a specific tool's schema
sage tools inspect FileSystem

# Test a tool call directly
sage tools call FileSystem.read --path ./README.md
```

---

### 6.4 Tool Type Checking

The type checker must verify:

- All tool methods called on an agent are declared in the agent's `uses` clause (E030)
- Tool method argument types match the declaration (E031)
- Tool method return types are used correctly (E032)
- `fails` is propagated — calling a tool method without `try` or `catch` is E013

---

### 6.5 Built-in Tool Implementations

For development and testing, Sage ships built-in implementations for common tools that do not require external MCP servers:

| Tool | Description |
|------|-------------|
| `FileSystem` | Local file system access |
| `Stdin` | Reading from standard input |
| `Http` | Basic HTTP GET/POST |
| `Env` | Reading environment variables |
| `Clock` | Time utilities (wraps the stdlib time module) |

Built-in tools are configured in `sage.toml`:

```toml
[tools]
FileSystem = { builtin = "filesystem" }
Http       = { builtin = "http" }
```

---

## 7. Observability

**Current state:** No observability. `print` is the only debugging tool.

**v1.0 requirement:** Structured tracing of agent lifecycle events and LLM calls, consumable by standard tooling.

---

### 7.1 Trace Events

When tracing is enabled, the runtime emits newline-delimited JSON (NDJSON) to stderr (or a configurable file). Each line is a trace event:

```json
{"t": 1742050000123, "kind": "agent.spawn",    "agent": "Researcher", "id": "a1", "beliefs": {"topic": "quantum computing"}}
{"t": 1742050000124, "kind": "infer.start",    "agent": "a1",         "model": "gpt-4o-mini", "prompt_len": 48}
{"t": 1742050001201, "kind": "infer.complete", "agent": "a1",         "model": "gpt-4o-mini", "response_len": 312, "duration_ms": 1077}
{"t": 1742050001202, "kind": "agent.emit",     "agent": "a1",         "value_type": "String"}
{"t": 1742050001203, "kind": "agent.stop",     "agent": "a1",         "duration_ms": 1079}
{"t": 1742050001210, "kind": "agent.error",    "agent": "a2",         "error": {"kind": "Network", "message": "..."}}
```

**Event kinds:**

| Kind | Fields |
|------|--------|
| `agent.spawn` | `agent`, `id`, `beliefs` (field names only, not values, for privacy) |
| `agent.emit` | `agent`, `id`, `value_type` |
| `agent.stop` | `agent`, `id`, `duration_ms` |
| `agent.error` | `agent`, `id`, `error` |
| `infer.start` | `agent`, `id`, `model`, `prompt_len` |
| `infer.complete` | `agent`, `id`, `model`, `response_len`, `duration_ms`, `retries` |
| `infer.error` | `agent`, `id`, `error` |
| `tool.call` | `agent`, `id`, `tool`, `method`, `arg_count` |
| `tool.complete` | `agent`, `id`, `tool`, `method`, `duration_ms` |
| `tool.error` | `agent`, `id`, `tool`, `method`, `error` |
| `message.send` | `from_agent`, `to_agent`, `message_type` |
| `message.receive` | `agent`, `id`, `message_type` |

Prompt content and response content are **not** included in trace events by default. They can be included by setting `SAGE_TRACE_FULL=1`, which the user must explicitly opt into.

---

### 7.2 Enabling Tracing

**CLI flag:**

```bash
sage run program.sg --trace                    # Emit trace events to stderr
sage run program.sg --trace-file trace.ndjson  # Emit to file
sage run program.sg --trace-full               # Include prompt/response content
```

**Environment variable:**

```bash
SAGE_TRACE=1 sage run program.sg
SAGE_TRACE_FILE=trace.ndjson sage run program.sg
```

---

### 7.3 `sage trace` Command

A companion CLI command for working with trace files:

```bash
# Pretty-print a trace file
sage trace pretty trace.ndjson

# Summarize: agent timeline, total infer calls, total cost estimate
sage trace summary trace.ndjson

# Filter to a specific agent
sage trace filter trace.ndjson --agent Researcher

# Show all infer calls and their durations
sage trace infer trace.ndjson

# Calculate total tokens / estimated cost (requires model pricing config)
sage trace cost trace.ndjson
```

---

### 7.4 In-Program Tracing

Programs can emit custom trace events using `trace`:

```sage
agent Coordinator {
    on start {
        trace("Starting coordination pipeline");

        let r = spawn Researcher { topic: "AI" };
        let result = await r;

        trace("Research complete, length: {len(result)}");
        emit(result);
    }
}
```

`trace` is a language keyword (like `print`) that accepts any value. Custom trace events appear in the NDJSON output with `kind: "user"`.

---

## 8. Testing Infrastructure

**Current state:** No `sage test` command. No mock LLM support in the language. Programs are tested by running them against real LLMs.

**v1.0 requirement:** A complete testing framework that allows agent programs to be tested without real LLM calls.

---

### 8.1 Test Files and the `sage test` Command

Test files follow naming conventions:

- Any file in a `tests/` directory
- Any file ending in `_test.sg`

```bash
sage test                          # Run all tests in current project
sage test tests/researcher_test.sg # Run a specific test file
sage test --filter "summarizer"    # Run tests matching a name pattern
sage test --verbose                # Show output for passing tests too
```

A test file is a regular Sage file that may additionally contain `test` blocks.

---

### 8.2 Test Blocks

```sage
test "Researcher emits a non-empty string" {
    let r = spawn Researcher { topic: "quantum computing" };
    let result = await r;
    assert(!is_empty(result));
}

test "Coordinator produces results for both topics" {
    let c = spawn Coordinator {};
    let results = await c;
    assert(len(results) == 2);
    assert(all(results, |r: String| !is_empty(r)));
}

test "parse_int fails on non-numeric input" {
    let result = parse_int("not a number") catch e { -1 };
    assert(result == -1);
}
```

Tests are run in isolation. Each test spawns agents fresh. Tests run concurrently by default.

**Assertion functions:**

```sage
assert(condition: Bool)                        // Fails test if false
assert_eq(a: T, b: T)                         // Fails if a != b, shows both values
assert_ne(a: T, b: T)                         // Fails if a == b
assert_contains(s: String, sub: String)        // Fails if s does not contain sub
assert_fails(f: Fn() -> T)                    // Fails test if f does NOT fail
```

---

### 8.3 Mock LLM

Tests can configure mock LLM responses to avoid real API calls:

```sage
test "Researcher uses the topic in its output" {
    mock_infer {
        when contains(prompt, "quantum computing") {
            return "Quantum computing uses quantum mechanical phenomena...";
        }
        default {
            return "Default mock response.";
        }
    }

    let r = spawn Researcher { topic: "quantum computing" };
    let result = await r;
    assert_contains(result, "quantum");
}
```

**`mock_infer` semantics:**

- `mock_infer` blocks are scoped to the enclosing `test` block
- `when condition { return value; }` matches prompts where `condition` is true
- `prompt` is the rendered prompt string (after interpolation)
- `default { return value; }` is the fallback — required if any `when` clauses exist
- If no `mock_infer` is configured, real LLM calls are made (with a warning)
- If `mock_infer` is configured but no clause matches, the test fails with an error

**Forcing LLM calls in tests:**

```bash
sage test --real-llm                   # Run tests against real LLM regardless of mock_infer
```

---

### 8.4 Mock Tools

Similarly, tools can be mocked in tests:

```sage
test "Assistant writes output to file" {
    mock_tool FileSystem {
        fn write(path: String, content: String) {
            assert_eq(path, "output.txt");
            assert(!is_empty(content));
        }
        fn read(path: String) -> String {
            return "mock file contents";
        }
    }

    mock_infer { default { return "Mocked summary."; } }

    let a = spawn Assistant { query: "fusion energy" };
    await a;
}
```

Tool mocks intercept calls before they reach MCP servers.

---

### 8.5 Test Output Format

```
sage test

Running 12 tests...

  ✓  Researcher emits a non-empty string              (0.8ms)
  ✓  Coordinator produces results for both topics     (1.2ms)
  ✓  parse_int fails on non-numeric input             (0.1ms)
  ✗  Summarizer handles empty input                   (0.3ms)
     FAILED: assert_contains
       expected: result to contain "no content"
         actual: ""
  ✓  ...

Results: 11 passed, 1 failed, 0 skipped
```

Exit code is 0 if all tests pass, 1 if any fail.

---

## 9. Tooling — LSP, Formatter, REPL

**Current state:** No LSP, no formatter, no REPL.

**v1.0 requirement:** A VS Code extension with LSP support, a `sage fmt` command, and a `sage eval` command.

---

### 9.1 Language Server Protocol (LSP)

The Sage LSP server (`sage-lsp`) implements the Language Server Protocol. It is shipped as part of the Sage distribution.

**v1.0 feature set:**

| Feature | Priority |
|---------|----------|
| Syntax highlighting | Required |
| Parse error diagnostics | Required |
| Type error diagnostics | Required |
| Hover: show type of expression | Required |
| Go-to-definition (agents, functions, records) | Required |
| Auto-complete: agent fields, record fields, stdlib functions | Required |
| Code action: add missing error handler | Required |
| Code action: add `try` to unhandled fallible call | Required |
| Find all references | Nice-to-have |
| Rename symbol | Nice-to-have |
| Inline documentation on hover | Nice-to-have |

**VS Code Extension:**

Published to the VS Code Marketplace as `sagelang.sage`. Provides:

- Grammar for syntax highlighting
- Connects to `sage-lsp` automatically
- `sage run` integration (F5 to run current file)
- Output panel for trace events

---

### 9.2 Formatter (`sage fmt`)

The Sage formatter produces canonical Sage source. It is opinionated and non-configurable (in the spirit of `gofmt` and `rustfmt`).

```bash
sage fmt program.sg            # Format in place
sage fmt --check program.sg    # Exit 1 if file would change (for CI)
sage fmt src/                  # Format all .sg files in directory recursively
```

**Formatting rules:**

- Indent with 4 spaces (no tabs)
- Agent body fields aligned to same column
- One blank line between top-level declarations
- Two blank lines before `run` statement
- `on start` before `on message` before `on error` before `on stop`
- String concatenation `++` with spaces either side
- Binary operators with spaces either side
- Closing braces on their own line
- `spawn` initializer fields: one per line if more than two fields, inline if one or two

---

### 9.3 `sage eval` — Interactive Evaluation

`sage eval` provides a quick feedback loop without a full compilation cycle:

```bash
# Evaluate a single expression
sage eval 'str(2 + 2)'
# Output: "4"

# Run a short program inline
sage eval '
  let nums = [1, 2, 3, 4, 5];
  print(sum(nums));
'
# Output: 15

# Run a file as a script (no agents required)
sage eval script.sg
```

`sage eval` does not support `agent` declarations, `spawn`, or `infer` (these require the full compiler pipeline and a runtime). It is intended for quick stdlib and expression testing.

---

### 9.4 `sage check` Improvements

The existing `sage check` command must report:

- All errors with line/column and error code
- All warnings (including unhandled `infer` calls, deprecated syntax)
- A machine-readable JSON output mode (`--json`) for CI integration

```bash
sage check --json program.sg
```

Output format:

```json
{
  "file": "program.sg",
  "errors": [
    { "code": "E013", "message": "unhandled fallible call: infer", "line": 12, "col": 8 }
  ],
  "warnings": [
    { "code": "W001", "message": "belief keyword is deprecated; use bare field syntax", "line": 4, "col": 5 }
  ]
}
```

---

## 10. Package Ecosystem

**Current state:** `sage.toml` has a `[dependencies]` section but no registry and no `sage add` command.

**v1.0 requirement:** A functional package system. A full registry is post-v1.0; v1.0 delivers Git-based packages and local path dependencies.

---

### 10.1 Dependency Specification

```toml
[project]
name = "my_project"
version = "1.0.0"

[dependencies]
sage-research-agents = { git = "https://github.com/example/sage-research-agents", tag = "v1.2.0" }
sage-utils            = { git = "https://github.com/example/sage-utils", branch = "main" }
my-local-lib          = { path = "../my-local-lib" }
```

---

### 10.2 CLI Commands

```bash
sage add git https://github.com/example/package    # Add a git dependency
sage add path ../my-local-lib                      # Add a local path dependency
sage remove package-name                           # Remove a dependency
sage update                                        # Update all dependencies
sage update package-name                           # Update one dependency

sage install                                       # Install all dependencies (run after clone)
```

`sage install` is run automatically before `sage build` and `sage run` if `sage.lock` is out of date.

---

### 10.3 Lock File (`sage.lock`)

`sage.lock` records exact resolved versions (commit hashes for Git dependencies):

```toml
[[package]]
name = "sage-research-agents"
source = "git+https://github.com/example/sage-research-agents"
commit = "a3f8b2c1d9e4f6a7b8c9d0e1f2a3b4c5d6e7f8a9"
```

`sage.lock` must be committed to version control.

---

### 10.4 Package Structure

A Sage package follows the same structure as a Sage project. The library entry point is the file specified in `sage.toml` as `lib`:

```toml
[project]
name = "sage-research-agents"
version = "1.2.0"
lib = "src/lib.sg"           # Library entry point (no run statement)
```

Only `pub` items from the library entry point are importable by dependents.

---

### 10.5 Package Registry (Post-v1.0)

A central registry (`packages.sagelang.io`) is planned for v1.1. v1.0 packages are Git-only. The package manifest format is designed to be forward-compatible with registry-based packages.

---

## 11. Documentation Consistency

**Current state:** Several documented features are either inconsistent with the implementation or describe syntax that has been superseded.

All documentation issues must be resolved before v1.0 is declared.

---

### 11.1 Required Documentation Updates

| Issue | File | Action Required |
|-------|------|-----------------|
| `belief` keyword still used in examples | `guide/src/introduction.md` | Replace with bare field syntax |
| `belief` keyword still used in examples | `guide/src/agents/overview.md` | Replace with bare field syntax |
| `belief` keyword still used in examples | `guide/src/agents/handlers.md` | Replace with bare field syntax |
| `belief` keyword still used in examples | `guide/src/agents/beliefs.md` | Rewrite to reflect RFC-0005 |
| `on stop` documented but marked unimplemented | `guide/src/agents/handlers.md` | Update once implemented |
| `send()` presented as working | `guide/src/agents/messaging.md` | Clarify status until codegen complete |
| Error handling not documented | (missing) | Write new guide section |
| No testing documentation | (missing) | Write new guide section |
| No observability documentation | (missing) | Write new guide section |
| Tool integration not documented | (missing) | Write new guide section |

---

### 11.2 New Guide Sections Required

The v1.0 guide must include all of the following:

```
# Error Handling
- Overview: the error model
- try: propagating errors
- catch: inline recovery
- fail: explicit failure
- on error: agent error handlers
- retry: retrying fallible operations
- Error kinds and taxonomy

# Tools
- What are tools?
- Declaring a tool
- Using tools in agents
- MCP configuration
- Built-in tools reference
- Testing with mock tools

# Observability
- Enabling tracing
- Trace event reference
- The sage trace command
- Custom trace events

# Testing
- Writing test blocks
- Assertions reference
- Mocking LLM calls
- Mocking tools
- Running tests: sage test
- CI integration

# Standard Library
(Full reference for each module)
```

---

### 11.3 Deprecation Warnings

The `belief` keyword must produce a deprecation warning (`W001`) in the compiler output for at least one minor version before being removed. It was removed in RFC-0005 but the docs still teach it. The compiler must emit:

```
warning[W001]: `belief` keyword is deprecated. Use bare field syntax instead.
  --> program.sg:4:5
   |
 4 |     belief topic: String
   |     ^^^^^^ help: remove `belief`: `topic: String`
```

---

## 12. v1.0 Completion Checklist

The following must all be true before the v1.0 tag is cut.

### Compiler & Runtime

- [ ] `on error` handler fully implemented in codegen
- [ ] `on stop` handler fully implemented in codegen
- [ ] `try` and `catch` fully implemented in codegen
- [ ] `fail` expression implemented
- [ ] `retry` expression implemented (count and delay forms)
- [ ] `await` timeout syntax implemented
- [ ] `infer` structured error taxonomy implemented
- [ ] Tool declarations and `uses` clause implemented
- [ ] MCP client integration implemented
- [ ] Built-in tools (FileSystem, Stdin, Http, Env, Clock) implemented
- [ ] Tool type checking (E030, E031, E032) implemented
- [ ] `belief` keyword deprecation warning implemented (W001)
- [ ] `send()` codegen implemented (currently deferred per RFC-0006)
- [ ] Graceful shutdown on SIGINT/SIGTERM implemented

### Standard Library

- [ ] String module complete (section 4.1)
- [ ] List module complete (section 4.2)
- [ ] Math module complete (section 4.3)
- [ ] I/O module complete (section 4.4)
- [ ] Time module complete (section 4.5)
- [ ] Option utilities complete (section 4.6)
- [ ] JSON utilities complete (section 4.7)
- [ ] All stdlib functions covered by tests

### Testing Infrastructure

- [ ] `sage test` command implemented
- [ ] `test` block syntax implemented
- [ ] Assertion functions implemented
- [ ] `mock_infer` implemented
- [ ] `mock_tool` implemented
- [ ] Test output formatter implemented
- [ ] `--real-llm` flag implemented

### Observability

- [ ] Trace event emission implemented
- [ ] `--trace` and `--trace-file` flags implemented
- [ ] `sage trace` subcommands implemented (pretty, summary, filter, infer, cost)
- [ ] `trace()` language keyword implemented

### Tooling

- [ ] `sage-lsp` implements required feature set (section 9.1)
- [ ] VS Code extension published to Marketplace
- [ ] `sage fmt` implemented and tested
- [ ] `sage fmt --check` implemented
- [ ] `sage eval` implemented
- [ ] `sage check --json` implemented

### Package Ecosystem

- [ ] Git-based dependencies implemented
- [ ] Local path dependencies implemented
- [ ] `sage.lock` generation and verification implemented
- [ ] `sage add`, `sage remove`, `sage update`, `sage install` implemented

### Documentation

- [ ] All `belief` keyword occurrences removed from guide
- [ ] Error handling guide section written
- [ ] Tools guide section written
- [ ] Observability guide section written
- [ ] Testing guide section written
- [ ] Standard library reference written
- [ ] All examples in guide use `try`/`catch` on `infer` calls
- [ ] All guide examples compile and run against v1.0

---

## 13. Out of Scope for v1.0

The following are explicitly deferred:

- **Generics / parametric polymorphism** — v2.0
- **Algebraic effects system** — v2.0
- **Session types** — v2.0
- **WASM compilation target** — v2.0
- **Package registry** (`packages.sagelang.io`) — v1.1
- **Agent pools** — v1.1
- **OTP-style supervision trees** — v1.1
- **Broadcast messaging** — v1.1
- **`Inferred<T>` with enforcement rather than coercion** — v2.0
- **DSPy-style prompt optimization** — v3.0
- **Debugging / breakpoints** — v1.1
- **Multi-file test discovery with `#[test]` annotation style** — v1.1
- **Type inference (Hindley-Milner)** — v2.0
- **Operator overloading** — v2.0
- **Macros / metaprogramming** — v2.0

---

## 14. Implementation Notes (2026-03-16)

The following items from this roadmap have been **deferred to v2.0** based on implementation review:

| Feature | Original Section | Reason for Deferral |
|---------|------------------|---------------------|
| **MCP protocol client** | 6.3 | Significant protocol work; Http tool covers immediate needs |
| **Built-in tools: FileSystem, Stdin, Env, Clock** | 6.5 | Can be added incrementally; Http is sufficient for v1.0 |
| **`sage tools` CLI commands** | 6.3 | Depends on MCP integration |
| **LSP: hover, go-to-definition, autocomplete, code actions** | 9.1 | Diagnostics-only is viable for v1.0; IDE smarts are v2.0 |
| **VS Code extension publication** | 9.1 | Requires LSP feature completeness |
| **`sage check --json`** | 9.4 | Nice-to-have for CI; not blocking |

The following are being implemented for v1.0:
- `sage trace` subcommands (pretty, summary, filter, infer, cost)
- `mock_tool` syntax for test blocks
- Local path dependencies (`path = "..."` in sage.toml)

---

*Ward has been patient. v1.0 is what he's been waiting for.*
