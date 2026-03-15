# RFC-0003: Compile Sage to Rust

- **Status:** Implemented
- **Created:** 2026-03-12
- **Author:** Sage Team

## Summary

Replace the tree-walking interpreter with a compiler that generates Rust source code. Sage programs will compile to native binaries via `rustc`, leveraging Tokio for async execution and the existing Rust ecosystem for HTTP, serialization, and observability.

## Motivation

The current interpreter is suitable for prototyping but has limitations:

1. **Performance overhead** — AST traversal on every execution
2. **No static binaries** — Requires the Sage runtime to be installed
3. **Limited tooling** — No debugger integration, profiling, etc.
4. **Deployment friction** — Users must install Sage CLI to run programs

Compiling to Rust solves these problems while providing:

- Native performance
- Single static binaries
- Access to Rust's async ecosystem (Tokio, reqwest, tracing)
- Memory safety without garbage collection
- Easy distribution (just ship the binary)

## Design Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                        Sage Compiler                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   Source (.sg)                                                  │
│        │                                                        │
│        ▼                                                        │
│   ┌─────────┐                                                   │
│   │  Lexer  │  (sage-lexer, unchanged)                          │
│   └────┬────┘                                                   │
│        │                                                        │
│        ▼                                                        │
│   ┌─────────┐                                                   │
│   │ Parser  │  (sage-parser, unchanged)                         │
│   └────┬────┘                                                   │
│        │                                                        │
│        ▼                                                        │
│   ┌─────────┐                                                   │
│   │ Checker │  (sage-checker, unchanged)                        │
│   └────┬────┘                                                   │
│        │                                                        │
│        ▼                                                        │
│   ┌─────────┐      ┌─────────────────┐                          │
│   │ Codegen │ ───▶ │ Generated Rust  │                          │
│   └─────────┘      │    + Cargo.toml │                          │
│   (NEW CRATE)      └────────┬────────┘                          │
│                             │                                   │
│                             ▼                                   │
│                        cargo build                              │
│                             │                                   │
│                             ▼                                   │
│                      Native Binary                              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### New Crates

| Crate | Purpose |
|-------|---------|
| `sage-codegen` | AST → Rust source code generation |
| `sage-runtime` | Runtime library linked by generated code |

## Language Mapping

### Types

| Sage Type | Rust Type |
|-----------|-----------|
| `Int` | `i64` |
| `Float` | `f64` |
| `Bool` | `bool` |
| `String` | `String` |
| `Unit` | `()` |
| `List<T>` | `Vec<T>` |
| `Option<T>` | `Option<T>` |
| `Agent<T>` | `sage_runtime::AgentHandle<T>` |
| `Inferred<T>` | `T` (resolved at LLM call) |

### Agents

Sage agents become Rust structs with async methods:

**Sage:**
```sage
agent Worker {
    belief value: Int
    belief multiplier: Int

    on start {
        let result = self.value * self.multiplier;
        emit(result);
    }
}
```

**Generated Rust:**
```rust
use sage_runtime::prelude::*;

struct Worker {
    value: i64,
    multiplier: i64,
}

impl Worker {
    fn new(value: i64, multiplier: i64) -> Self {
        Self { value, multiplier }
    }

    async fn on_start(self, ctx: AgentContext<i64>) -> SageResult<i64> {
        let result = self.value * self.multiplier;
        ctx.emit(result).await
    }
}
```

### Spawn & Await

**Sage:**
```sage
let w = spawn Worker { value: 10, multiplier: 2 };
let result = await w;
```

**Generated Rust:**
```rust
let w = sage_runtime::spawn(Worker::new(10, 2)).await;
let result = w.await?;
```

Under the hood, `sage_runtime::spawn` uses `tokio::spawn` and returns an `AgentHandle<T>` wrapping a `JoinHandle` and message channels.

### Functions

**Sage:**
```sage
fn factorial(n: Int) -> Int {
    if n <= 1 {
        return 1;
    }
    return n * factorial(n - 1);
}
```

**Generated Rust:**
```rust
fn factorial(n: i64) -> i64 {
    if n <= 1 {
        return 1;
    }
    return n * factorial(n - 1);
}
```

Direct 1:1 mapping. Sage functions become Rust functions.

### Control Flow

| Sage | Rust |
|------|------|
| `if cond { } else { }` | `if cond { } else { }` |
| `for x in list { }` | `for x in list { }` |
| `while cond { }` | `while cond { }` |
| `return expr;` | `return expr;` |

Control flow maps directly with no transformation needed.

### Infer (LLM Calls)

**Sage:**
```sage
let summary: Inferred<String> = infer("Summarize: {topic}");
```

**Generated Rust:**
```rust
let summary: String = ctx.infer::<String>(&format!("Summarize: {}", topic)).await?;
```

The runtime handles:
- HTTP request to LLM provider
- Response parsing
- Type coercion based on generic parameter

### String Interpolation

**Sage:**
```sage
let msg = "Hello, {name}! Count: {count}";
```

**Generated Rust:**
```rust
let msg = format!("Hello, {}! Count: {}", name, count);
```

### Operators

| Sage | Rust |
|------|------|
| `+`, `-`, `*`, `/` | Same |
| `==`, `!=`, `<`, `>`, `<=`, `>=` | Same |
| `&&`, `\|\|`, `!` | Same |
| `++` (concat) | `format!("{}{}", a, b)` or `a + &b` |

### Built-in Functions

| Sage | Rust |
|------|------|
| `print(msg)` | `println!("{}", msg)` |
| `str(value)` | `value.to_string()` or `format!("{}", value)` |
| `len(list)` | `list.len() as i64` |

## Runtime Library (`sage-runtime`)

A small runtime crate providing:

```rust
// sage-runtime/src/lib.rs

pub mod prelude {
    pub use crate::{AgentContext, AgentHandle, SageResult, SageError};
    pub use crate::spawn;
}

/// Result type for Sage operations.
pub type SageResult<T> = Result<T, SageError>;

/// Error type for Sage runtime errors.
#[derive(Debug, thiserror::Error)]
pub enum SageError {
    #[error("LLM error: {0}")]
    Llm(String),
    #[error("Agent error: {0}")]
    Agent(String),
    #[error("Type error: expected {expected}, got {got}")]
    Type { expected: String, got: String },
}

/// Handle to a spawned agent.
pub struct AgentHandle<T> {
    join: tokio::task::JoinHandle<SageResult<T>>,
    tx: tokio::sync::mpsc::Sender<Message>,
}

impl<T> AgentHandle<T> {
    /// Await the agent's result.
    pub async fn result(self) -> SageResult<T> {
        self.join.await.map_err(|e| SageError::Agent(e.to_string()))?
    }

    /// Send a message to the agent.
    pub async fn send(&self, msg: impl Into<Message>) -> SageResult<()> {
        self.tx.send(msg.into()).await
            .map_err(|e| SageError::Agent(e.to_string()))
    }
}

/// Context passed to agent handlers.
pub struct AgentContext<T> {
    llm: LlmClient,
    result_tx: Option<tokio::sync::oneshot::Sender<T>>,
}

impl<T> AgentContext<T> {
    /// Emit a value to the awaiter.
    pub async fn emit(self, value: T) -> SageResult<T> {
        if let Some(tx) = self.result_tx {
            let _ = tx.send(value.clone());
        }
        Ok(value)
    }

    /// Call the LLM with a prompt.
    pub async fn infer<R: DeserializeOwned>(&self, prompt: &str) -> SageResult<R> {
        self.llm.infer(prompt).await
    }
}

/// Spawn an agent as an async task.
pub async fn spawn<A, T>(agent: A) -> AgentHandle<T>
where
    A: Agent<Output = T> + Send + 'static,
    T: Send + 'static,
{
    let (result_tx, result_rx) = tokio::sync::oneshot::channel();
    let (msg_tx, msg_rx) = tokio::sync::mpsc::channel(32);

    let ctx = AgentContext {
        llm: LlmClient::from_env(),
        result_tx: Some(result_tx),
    };

    let join = tokio::spawn(async move {
        agent.on_start(ctx).await
    });

    AgentHandle { join, tx: msg_tx }
}

/// Trait implemented by generated agent structs.
pub trait Agent {
    type Output;
    fn on_start(self, ctx: AgentContext<Self::Output>) -> impl Future<Output = SageResult<Self::Output>> + Send;
}
```

## Code Generation Strategy

### Output Structure

For a Sage program `example.sg`, generate:

```
target/sage/example/
├── Cargo.toml
├── src/
│   └── main.rs
└── .cargo/
    └── config.toml  (optional, for cross-compilation)
```

### Generated `Cargo.toml`

```toml
[package]
name = "example"
version = "0.1.0"
edition = "2021"

[dependencies]
sage-runtime = { version = "0.1", path = "..." }  # or from crates.io
tokio = { version = "1", features = ["full"] }
serde = { version = "1", features = ["derive"] }
serde_json = "1"
```

### Generated `main.rs`

```rust
//! Generated by Sage compiler. Do not edit.

use sage_runtime::prelude::*;

// === User-defined functions ===

fn factorial(n: i64) -> i64 {
    if n <= 1 {
        return 1;
    }
    return n * factorial(n - 1);
}

// === Agent: Worker ===

struct Worker {
    value: i64,
    multiplier: i64,
}

impl Agent for Worker {
    type Output = i64;

    async fn on_start(self, ctx: AgentContext<i64>) -> SageResult<i64> {
        let result = self.value * self.multiplier;
        ctx.emit(result).await
    }
}

// === Agent: Main (entry point) ===

struct Main;

impl Agent for Main {
    type Output = i64;

    async fn on_start(self, ctx: AgentContext<i64>) -> SageResult<i64> {
        let w = sage_runtime::spawn(Worker { value: 10, multiplier: 2 }).await;
        let result = w.result().await?;
        ctx.emit(result).await
    }
}

// === Entry point ===

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let result = sage_runtime::spawn(Main).await.result().await?;
    println!("{:?}", result);
    Ok(())
}
```

## Codegen Implementation

### AST Visitor Pattern

```rust
// sage-codegen/src/lib.rs

pub struct Codegen {
    output: String,
    indent: usize,
}

impl Codegen {
    pub fn generate(program: &Program) -> String {
        let mut cg = Codegen::new();
        cg.emit_prelude();
        cg.emit_functions(&program.functions);
        cg.emit_agents(&program.agents);
        cg.emit_main(&program.run_agent);
        cg.output
    }

    fn emit_expr(&mut self, expr: &Expr) {
        match expr {
            Expr::Literal { value, .. } => self.emit_literal(value),
            Expr::Binary { op, left, right, .. } => {
                self.emit_expr(left);
                self.emit_binop(op);
                self.emit_expr(right);
            }
            Expr::Call { name, args, .. } => {
                self.emit_call(name, args);
            }
            // ... etc
        }
    }

    fn emit_stmt(&mut self, stmt: &Stmt) {
        match stmt {
            Stmt::Let { name, value, .. } => {
                self.write(&format!("let {} = ", name.name));
                self.emit_expr(value);
                self.writeln(";");
            }
            Stmt::If { condition, then_block, else_block, .. } => {
                self.write("if ");
                self.emit_expr(condition);
                self.emit_block(then_block);
                if let Some(else_branch) = else_block {
                    self.emit_else(else_branch);
                }
            }
            // ... etc
        }
    }
}
```

### Handling Async

Key insight: Only certain operations need `.await`:
- `spawn` → returns handle (no await on spawn itself)
- `await handle` → `.result().await?`
- `infer` → `.await?`
- `send` → `.await?`
- `emit` → `.await`

The codegen must track which expressions are async and propagate accordingly.

## CLI Changes

### New Commands

```bash
# Compile to Rust and build
sage build example.sg
sage build example.sg --release
sage build example.sg --target aarch64-unknown-linux-gnu

# Compile, build, and run
sage run example.sg

# Just generate Rust (for inspection)
sage codegen example.sg --output ./generated/

# Check without compiling (unchanged)
sage check example.sg
```

### Build Modes

| Mode | Description |
|------|-------------|
| `--debug` | Fast compile, slow runtime (default) |
| `--release` | Slow compile, optimized runtime |
| `--emit rust` | Output Rust source only, don't compile |

## Migration Path

### Phase 1: Parallel Implementation
- Keep interpreter working
- Build codegen alongside
- `sage run` uses interpreter
- `sage build` uses codegen

### Phase 2: Feature Parity
- All language features supported in codegen
- Comprehensive test suite passes both paths
- Document any semantic differences

### Phase 3: Default Switch
- `sage run` uses codegen by default
- `sage run --interpret` for legacy mode
- Deprecation warnings

### Phase 4: Interpreter Removal
- Remove `sage-interpreter` crate
- Simplify codebase
- Keep only `sage-codegen` path

## Testing Strategy

### Roundtrip Tests

For each test case:
1. Run with interpreter → result A
2. Compile to Rust → run → result B
3. Assert A == B

```rust
#[test]
fn test_factorial() {
    let source = r#"
        fn factorial(n: Int) -> Int {
            if n <= 1 { return 1; }
            return n * factorial(n - 1);
        }
        agent Main {
            on start { emit(factorial(5)); }
        }
        run Main;
    "#;

    let interpreted = run_interpreted(source);
    let compiled = run_compiled(source);
    assert_eq!(interpreted, compiled);
}
```

### Generated Code Tests

Snapshot testing for generated Rust:

```rust
#[test]
fn codegen_simple_agent() {
    let source = "...";
    let generated = Codegen::generate(parse(source));
    insta::assert_snapshot!(generated);
}
```

## Open Questions

### 1. Incremental Compilation

Should we cache compiled artifacts?

**Option A:** Always regenerate everything (simple, slower)
**Option B:** Hash-based caching like Cargo (complex, faster)

**Recommendation:** Start with A, optimize later.

### 2. Error Messages

How do we map Rust compile errors back to Sage source?

**Option A:** Source maps / `#[line = "..."]` directives
**Option B:** Generate readable Rust with comments
**Option C:** Catch common errors in Sage checker first

**Recommendation:** B + C. Make generated code readable, catch errors early.

### 3. Debug Symbols

Should we support debugging compiled Sage programs?

**Option A:** Debug at Rust level (requires reading generated code)
**Option B:** Proper source maps for debugger integration

**Recommendation:** A for now. B is significant effort.

### 4. Standard Library

Where do builtins live?

**Option A:** All in `sage-runtime`
**Option B:** Generate inline code for simple builtins

**Recommendation:** B for trivial ones (`len`, `str`), A for complex ones (`infer`).

### 5. Cargo Dependency

Users need Rust toolchain installed. Is this acceptable?

**Option A:** Require users to have `rustc` + `cargo`
**Option B:** Bundle `rustc` with Sage distribution
**Option C:** Use pre-compiled `sage-runtime`, ship objects not source

**Recommendation:** A for now. Sage users are developers; they likely have Rust.

## Timeline

| Phase | Duration | Deliverable |
|-------|----------|-------------|
| 1. Prototype | 2 weeks | Basic codegen for simple programs |
| 2. Core features | 3 weeks | All statements, expressions, functions |
| 3. Agents | 2 weeks | spawn, await, emit, message passing |
| 4. LLM integration | 1 week | infer with runtime HTTP client |
| 5. CLI integration | 1 week | `sage build`, `sage run` |
| 6. Testing & polish | 2 weeks | Roundtrip tests, error messages |
| **Total** | ~11 weeks | Production-ready compiler |

## Appendix A: Full Example

### Input (`research.sg`)

```sage
agent Researcher {
    belief topic: String

    on start {
        let summary: Inferred<String> = infer(
            "Write a 2-sentence summary of: {self.topic}"
        );
        emit(summary);
    }
}

agent Main {
    on start {
        let r1 = spawn Researcher { topic: "quantum computing" };
        let r2 = spawn Researcher { topic: "gene editing" };

        let s1 = await r1;
        let s2 = await r2;

        print("Research 1: {s1}");
        print("Research 2: {s2}");
        emit(0);
    }
}

run Main;
```

### Output (`main.rs`)

```rust
//! Generated by Sage compiler. Do not edit.

use sage_runtime::prelude::*;

struct Researcher {
    topic: String,
}

impl Agent for Researcher {
    type Output = String;

    async fn on_start(self, ctx: AgentContext<String>) -> SageResult<String> {
        let summary: String = ctx
            .infer::<String>(&format!(
                "Write a 2-sentence summary of: {}",
                self.topic
            ))
            .await?;
        ctx.emit(summary).await
    }
}

struct Main;

impl Agent for Main {
    type Output = i64;

    async fn on_start(self, ctx: AgentContext<i64>) -> SageResult<i64> {
        let r1 = sage_runtime::spawn(Researcher {
            topic: "quantum computing".to_string(),
        })
        .await;
        let r2 = sage_runtime::spawn(Researcher {
            topic: "gene editing".to_string(),
        })
        .await;

        let s1 = r1.result().await?;
        let s2 = r2.result().await?;

        println!("Research 1: {}", s1);
        println!("Research 2: {}", s2);
        ctx.emit(0).await
    }
}

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let result = sage_runtime::spawn(Main).await.result().await?;
    std::process::exit(result as i32);
}
```

## References

- [RFC-0001: Sage POC](./RFC-0001-poc.md)
- [RFC-0002: Multi-file Projects](./RFC-0002-multi-file.md)
- [Tokio documentation](https://tokio.rs/)
- [The Rust Programming Language](https://doc.rust-lang.org/book/)
