# RFC-0007: Error Handling

- **Status:** Implemented
- **Created:** 2026-03-13
- **Author:** Sage Contributors

---

## Table of Contents

1. [Summary](#1-summary)
2. [Motivation](#2-motivation)
3. [Design Goals](#3-design-goals)
4. [Language Design](#4-language-design)
5. [Checker Rules](#5-checker-rules)
6. [Runtime Semantics](#6-runtime-semantics)
7. [Codegen](#7-codegen)
8. [New Error Codes](#8-new-error-codes)
9. [Implementation Plan](#9-implementation-plan)
10. [Open Questions](#10-open-questions)
11. [Alternatives Considered](#11-alternatives-considered)

---

## 1. Summary

This RFC introduces first-class error handling to Sage. Currently, LLM inference failures, network errors, and agent panics crash the entire program. This is acceptable for a POC but unusable in production.

The design introduces three complementary mechanisms:

- **`fails`** — a function annotation marking it as fallible
- **`catch { }`** — an inline recovery block at the call site
- **`try`** — propagation of failures up the call stack
- **`on error(e)`** — an agent-level handler that catches any unhandled failure in that agent

These mechanisms are enforced at compile time. Every fallible call must be either caught or explicitly propagated — unhandled failures are a type error.

---

## 2. Motivation

Today, this program silently crashes if the API is down:

```sage
agent Researcher {
    topic: String

    on start {
        let summary: Inferred<String> = infer(
            "Write a 2-sentence summary of: {self.topic}"
        );
        emit(summary);
    }
}
```

There is no way to recover. The agent panics, and if it is the entry agent, the whole program exits with an opaque error.

In a real system you need to handle this. A research pipeline should be able to fall back, retry, or emit a meaningful error message rather than crashing silently.

---

## 3. Design Goals

1. **Explicit, not implicit.** Errors must be acknowledged at the call site. Silent swallowing is not possible.
2. **Fits the agent model.** Agents already have an event handler model (`on start`, `on message`, `on stop`). Error handling should feel like a natural extension: `on error`.
3. **Readable by non-Rust developers.** `try` and `catch` are familiar from many languages. `fails` is a plain English annotation.
4. **Enforced at compile time.** The checker rejects programs where fallible calls are unhandled.
5. **Composable.** `catch` handles locally. `try` delegates upward. They can be combined.

---

## 4. Language Design

### 4.1 Marking functions as fallible: `fails`

Any function that can fail at runtime is annotated with `fails` in its signature:

```sage
fn fetch(url: String) -> String fails {
    // implementation may fail
}
```

`fails` is part of the function's type. The checker tracks which functions are fallible and enforces handling at every call site.

Built-in fallible operations (`infer`, `send`, and any future I/O functions) are implicitly `fails`. They do not need to be declared — the checker knows about them as part of the prelude.

Functions without `fails` are guaranteed not to fail. They may call other fallible functions internally, but only if those failures are fully handled inside the function body.

### 4.2 Inline recovery: `catch { }`

The `catch` block immediately follows a fallible call and provides a recovery expression:

```sage
let summary = infer("Summarise: {self.topic}") catch {
    "Summary unavailable."
};
```

The `catch` block must produce a value of the same type as the successful call. The combined expression is non-fallible — the failure has been fully handled.

To access the error in the `catch` block, bind it with `catch(e)`:

```sage
let summary = infer("Summarise: {self.topic}") catch(e) {
    print("Inference failed: {e}");
    "Summary unavailable."
};
```

`e` is of type `Error` — a built-in type representing a runtime failure, with a `.message` field of type `String`.

`catch` blocks can also call fallible operations themselves, in which case those must also be handled:

```sage
let summary = infer("Summarise: {self.topic}") catch(e) {
    infer("Provide a one-sentence default about: {self.topic}") catch {
        "Summary unavailable."
    }
};
```

### 4.3 Propagation: `try`

`try` propagates a failure upward without handling it locally:

```sage
agent Researcher {
    topic: String

    on start {
        let summary = try infer("Summarise: {self.topic}");
        emit(summary);
    }

    on error(e) {
        print("Researcher failed: {e.message}");
        emit("unavailable");
    }
}
```

`try` is valid in two contexts:

**Context A — inside an agent that has an `on error` handler.** Failure routes to `on error`.

**Context B — inside a `fails` function.** Failure propagates to the caller.

```sage
fn summarise(topic: String) -> String fails {
    let result = try infer("Summarise: {topic}");
    result
}
```

Using `try` outside both contexts is a compile error (E014).

### 4.4 Agent error handler: `on error(e)`

`on error` is a new event handler for agents, alongside `on start`, `on message`, and `on stop`:

```sage
agent Researcher {
    topic: String

    on start {
        let summary = try infer("Summarise: {self.topic}");
        emit(summary);
    }

    on error(e) {
        print("Researcher failed: {e.message}");
        emit("unavailable");
    }
}
```

Rules for `on error`:

- There can be at most one `on error` handler per agent.
- The parameter `e` is of type `Error`.
- The handler **must** call `emit` with a value of the same type as the `on start` handler, or call `emit` with a default, or re-raise with `try` (routing the error to a supervisor — see §10).
- If an agent has no `on error` handler and a `try` is used inside it, the checker emits E014.
- `on error` itself may use `catch` for local recovery. It may not use `try` (there is nowhere further to propagate — see Open Questions §10.1 on supervision).

### 4.5 The `Error` type

A built-in type available without import:

```sage
// Accessible inside catch(e) and on error(e)
e.message   // String — human-readable description
e.kind      // ErrorKind — category of error (see below)
```

`ErrorKind` is an enum (not user-extensible in this RFC):

| Variant | Meaning |
|---------|---------|
| `Llm` | LLM API failure (network, rate limit, timeout) |
| `Agent` | A spawned agent failed and was awaited |
| `Runtime` | Other runtime failure |

Usage:

```sage
on error(e) {
    if e.kind == ErrorKind.Llm {
        print("LLM unavailable: {e.message}");
    }
    emit("fallback");
}
```

### 4.6 Failing agents and `await`

When an agent fails (its `on start` panics, or a `try` fires with no `on error` handler), the failure is captured in the `AgentHandle`. Awaiting that handle propagates the failure to the awaiting context:

```sage
agent Main {
    on start {
        let r = spawn Researcher { topic: "quantum computing" };
        let result = try await r;  // propagates if Researcher failed
        print(result);
        emit(0);
    }

    on error(e) {
        print("Pipeline failed: {e.message}");
        emit(1);
    }
}
```

`await` on a failed agent handle is itself a fallible operation. The failure must be handled with `catch` or `try`, just like any other fallible call.

### 4.7 Complete example

```sage
agent Researcher {
    topic: String

    on start {
        // try propagates failure to on error
        let summary = try infer("Write a 2-sentence summary of: {self.topic}");
        emit(summary);
    }

    on error(e) {
        print("Research failed for topic '{self.topic}': {e.message}");
        emit("No summary available.");
    }
}

agent Coordinator {
    on start {
        let r1 = spawn Researcher { topic: "quantum computing" };
        let r2 = spawn Researcher { topic: "CRISPR gene editing" };

        // Each await may fail if Researcher had no on error handler,
        // but since it does, we always get a String back.
        let s1 = await r1;
        let s2 = await r2;

        print("Summary 1: {s1}");
        print("Summary 2: {s2}");
        emit(0);
    }
}

run Coordinator;
```

Because `Researcher` has an `on error` handler that always emits a `String`, `await r1` is **not** fallible from `Coordinator`'s perspective — the checker knows the agent is always guaranteed to emit. This is an important property: a well-handled agent presents as infallible to its caller.

---

## 5. Checker Rules

### 5.1 Fallibility tracking

The checker maintains a `is_fallible: bool` flag for every function and built-in. Built-ins `infer`, `send`, and `await` (of a fallible handle) are always fallible. User functions are fallible iff their signature includes `fails`.

### 5.2 Call site enforcement

Every call to a fallible function must be one of:

- Directly followed by `catch { }` → non-fallible result, fully handled
- Prefixed with `try` → propagates upward
- Neither → **E013: unhandled fallible call**

### 5.3 `try` context validation

`try` is valid only when:

- Inside an agent body (any handler) where that agent declares `on error` → **routes to `on error`**
- Inside a `fails` function → **propagates to caller**

`try` outside both → **E014: `try` used outside fallible context**

### 5.4 `on error` emit type validation

The checker already infers an agent's emit type from its `on start` handler. The `on error` handler must emit the same type, or a compatible type. Mismatches → **E015: `on error` emit type mismatch**

### 5.5 `on error` uniqueness

An agent may declare at most one `on error` handler. Duplicates → **E016: duplicate `on error` handler**

### 5.6 `fails` function using `try`

A function using `try` must be declared `fails`. If it is not → **E017: `try` in non-`fails` function** (hint: add `fails` to signature).

### 5.7 Infallible agents

An agent with an `on error` handler that always emits is considered **infallible at the await site**. The checker propagates this: `await r` is only flagged as fallible if the spawned agent has no `on error` handler.

---

## 6. Runtime Semantics

### 6.1 `catch` execution

At runtime, if the expression before `catch` succeeds, the `catch` block is never evaluated. If it fails, the `catch` block runs and its value becomes the result. The error is discarded unless bound with `catch(e)`.

### 6.2 `try` execution

At runtime, if the expression succeeds, execution continues normally. If it fails, control jumps immediately to the `on error` handler (in the agent case) or unwinds to the caller (in the `fails` function case).

### 6.3 `on error` invocation

When a failure reaches an agent's `on error` handler, the handler runs in the same agent context. `self` is still accessible. The handler must complete with an `emit` call — if it does not, the agent panics unconditionally (this is a bug in user code and is not recoverable by design).

### 6.4 Agent failure propagation

If an agent has no `on error` handler and a `try` fires, the agent fails. The failure is stored in the `AgentHandle`. Any `await` on that handle will surface the failure to the awaiting context.

If an agent has an `on error` handler, the handle always resolves to a value. From the outside, the agent is infallible.

---

## 7. Codegen

### 7.1 `fails` functions

Functions marked `fails` generate Rust functions returning `SageResult<T>`:

**Sage:**
```sage
fn summarise(topic: String) -> String fails {
    let result = try infer("Summarise: {topic}");
    result
}
```

**Generated Rust:**
```rust
fn summarise(topic: String) -> sage_runtime::SageResult<String> {
    let result = ctx.infer::<String>(&format!("Summarise: {}", topic)).await?;
    Ok(result)
}
```

### 7.2 `catch` blocks

**Sage:**
```sage
let summary = infer("Summarise: {self.topic}") catch(e) {
    print("Failed: {e.message}");
    "unavailable"
};
```

**Generated Rust:**
```rust
let summary = match ctx.infer::<String>(&format!("Summarise: {}", self.topic)).await {
    Ok(v) => v,
    Err(e) => {
        println!("Failed: {}", e.message());
        "unavailable".to_string()
    }
};
```

### 7.3 `try`

**Sage:**
```sage
let summary = try infer("Summarise: {self.topic}");
```

**Generated Rust (in agent with `on error`):**
```rust
let summary = ctx.infer::<String>(&format!("Summarise: {}", self.topic)).await
    .map_err(|e| ctx.handle_error(e))?;
```

`ctx.handle_error` stores the error for routing to the `on error` handler after the current handler unwinds.

### 7.4 `on error` handler

**Sage:**
```sage
on error(e) {
    print("Failed: {e.message}");
    emit("unavailable");
}
```

**Generated Rust:**
```rust
async fn on_error(self, e: sage_runtime::SageError, ctx: AgentContext<String>)
    -> SageResult<String>
{
    println!("Failed: {}", e.message());
    ctx.emit("unavailable".to_string()).await
}
```

The runtime calls `on_error` when `on_start` returns an `Err` that originated from a `try` expression.

### 7.5 `SageError` in runtime

The `sage-runtime` crate needs a new public error type:

```rust
#[derive(Debug, thiserror::Error)]
pub enum SageError {
    #[error("LLM error: {0}")]
    Llm(String),
    #[error("Agent error: {0}")]
    Agent(String),
    #[error("Runtime error: {0}")]
    Runtime(String),
}

impl SageError {
    pub fn message(&self) -> &str { ... }
    pub fn kind(&self) -> ErrorKind { ... }
}

pub enum ErrorKind { Llm, Agent, Runtime }
```

`SageResult<T>` is already `Result<T, SageError>` in the runtime — this RFC formalises it as a stable public type.

---

## 8. New Error Codes

| Code | Name | Description |
|------|------|-------------|
| E013 | `UnhandledFallibleCall` | A fallible function was called without `catch` or `try` |
| E014 | `TryOutsideFallibleContext` | `try` used where there is no `on error` handler or `fails` function to propagate to |
| E015 | `OnErrorEmitTypeMismatch` | `on error` emits a different type than `on start` |
| E016 | `DuplicateOnErrorHandler` | An agent declares `on error` more than once |
| E017 | `TryInNonFailsFunction` | `try` used inside a function not marked `fails` |

---

## 9. Implementation Plan

### Phase 1 — Lexer & Parser ✅
- [x] Add `fails` keyword token (`KwFails`)
- [x] Add `try` keyword token (`KwTry`)
- [x] Add `catch` keyword token (`KwCatch`)
- [x] Add `error` keyword token (`KwError`)
- [x] Add `Error` type keyword (`TyError`)
- [x] Add `on error` as a new `EventKind` variant
- [x] Extend `FnDecl` AST node with `is_fallible: bool`
- [x] Add `Try` and `Catch` to the `Expr` enum
- [x] Parse `catch(e)` binding as `Catch { error_bind: Option<Ident>, recovery: Box<Expr> }`
- [x] Add `TypeExpr::Error` for the Error type

### Phase 2 — Checker ✅
- [x] Track `is_fallible` on all function entries in `SymbolTable`
- [x] Track `in_fallible_context` during type checking
- [x] Track `agent_has_error_handler` during type checking
- [x] Implement `try` context validation (E014 - try in non-fallible function)
- [x] Implement missing error handler check (E016 - try without on error handler)
- [x] Implement `catch` type compatibility checking (E015)
- [x] Add Error error types (E013-E016)
- [x] Track fallibility at `await` sites (requires error handling context)
- [x] Implement call-site enforcement for unhandled fallible calls (E013)

### Phase 3 — Runtime ✅
- [x] Stabilise `SageError` with `message()` and `kind()` accessors
- [x] Add `ErrorKind` enum to public API (`Llm`, `Agent`, `Runtime`)
- [x] Add helper constructors (`SageError::llm()`, `SageError::agent()`, etc.)

### Phase 4 — Codegen ✅
- [x] Add `TypeExpr::Error` -> `sage_runtime::SageError` mapping
- [x] Generate `SageResult<T>` return types for `fails` functions
- [x] Generate `match` arms for `catch` blocks (Ok/Err unwrapping)
- [x] Generate `?` propagation for `try` expressions
- [x] Generate `on_error` method on agent structs
- [x] Generate runtime dispatch from `on_start` failure to `on_error` in main()

---

## 10. Open Questions

### 10.1 Supervision trees

Should an agent's `on error` be able to re-raise to a parent agent? This would enable Erlang-style supervision:

```sage
// Hypothetical future syntax
agent Supervisor {
    on start {
        let worker = spawn Worker { ... };
        let result = try await worker;  // catches worker failure
        // decide whether to restart, fail, or emit fallback
        emit(result);
    }

    on error(e) {
        // top-level fallback
        emit("system degraded");
    }
}
```

This is already partially supported by the current design — a spawned agent that has no `on error` handler will surface its failure to the awaiting agent via `try await`. The open question is whether an agent with `on error` should be able to *choose* to re-raise upward. Deferred to a future RFC on supervision.

### 10.2 Retry built-in

A common pattern is retrying a failing LLM call. Should there be language-level retry support?

```sage
// Hypothetical
let summary = retry(3) infer("Summarise: {self.topic}") catch {
    "unavailable"
};
```

This could be implemented as a runtime function for now rather than syntax. Deferred.

### 10.3 Error context chaining

Should errors be chainable (like `anyhow::Context` in Rust)?

```sage
let result = try fetch(url) context("while fetching research data");
```

This would enrich the error message without changing its kind. Low priority — deferred.

### 10.4 Panics vs `fails`

Currently, out-of-bounds list access and integer division by zero cause a Rust panic, which bypasses the error handling model entirely. A future RFC should decide whether these become `fails`-style errors or remain panics. Out of scope here.

---

## 11. Alternatives Considered

### 11.1 `Result<T, Error>` type (Rust-style)

Exposing `Result<T, E>` as a first-class type was considered and rejected. It requires developers to understand algebraic data types and `match` expressions. The `try`/`catch`/`fails` model achieves the same safety with more familiar vocabulary and less syntactic noise.

### 11.2 Exceptions

Unchecked exceptions (Java/Python style) were rejected because they make error handling easy to ignore. The core goal is *enforced* acknowledgement at compile time.

### 11.3 `?` operator directly

Using `?` as a propagation operator (identical to Rust) was considered. Rejected because it is a symbolic operator with no obvious meaning to developers who don't know Rust, which conflicts with the design principle "readable by humans and LLMs equally."

### 11.4 Only `on error`, no `try`/`catch` in functions

A simpler design would restrict error handling to agents only — functions cannot fail, agents handle everything via `on error`. Rejected because it prevents composing reusable fallible utilities as functions, forcing all error-prone logic into agent bodies.

---

*This RFC brings Sage from a fragile POC to a language where failures are part of the contract, not an afterthought.*
