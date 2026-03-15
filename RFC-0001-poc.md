# Sage Language — POC RFC
**RFC-0001 | Proof of Concept Specification**
Version: 0.1.0 | Author: Pete Pavlovski | Status: Implemented

---

## Table of Contents

1. [Overview](#1-overview)
2. [Goals and Non-Goals](#2-goals-and-non-goals)
3. [Language Design Specification](#3-language-design-specification)
4. [Runtime Semantics](#4-runtime-semantics)
5. [Compiler Architecture](#5-compiler-architecture)
6. [Project Structure](#6-project-structure)
7. [Dependency Manifest](#7-dependency-manifest)
8. [Ordered Task List](#8-ordered-task-list)
9. [Demo Program](#9-demo-program)
10. [Success Criteria](#10-success-criteria)
11. [Out of Scope](#11-out-of-scope)

---

## 1. Overview

**Sage** is a general-purpose programming language where agents are first-class citizens at the language level — not a library, not a framework, but a semantic primitive baked into the compiler and runtime. It is implemented in Rust, uses the `.sg` file extension, and targets professional software developers building AI-native systems.

This document specifies the Proof of Concept (POC) scope: the minimum viable language subset, compiler pipeline, and runtime required to demonstrate that agents-as-primitives is a meaningfully different programming model from using Python frameworks like LangChain or CrewAI.

### Core thesis

The POC must answer one question convincingly: **can a language where agents, beliefs, and LLM inference are first-class constructs produce programs that are simpler, safer, and more readable than equivalent Python framework code?**

If yes, Sage has a reason to exist.

---

## 2. Goals and Non-Goals

### Goals

- Define a minimal but real Sage language grammar (not a toy)
- Build a working lexer, parser, type checker, and tree-walking interpreter in Rust
- Execute multi-agent programs with real LLM calls (OpenAI-compatible API)
- Demonstrate agent spawning, typed beliefs, message passing, and supervision
- Produce a compelling demo runnable with `sage run examples/research.sg`
- Establish a codebase structure that can grow into a full language

### Non-Goals for POC

- LLVM or Cranelift codegen (tree-walking interpreter only)
- Algebraic effects system (designed for, deferred to v0.2)
- Session types (deferred to v0.2)
- Package manager
- LSP / IDE integration
- Standard library beyond a minimal prelude
- Garbage collector (arena allocation only in POC)
- Imports / modules (single-file programs only)
- Generics / parametric polymorphism

---

## 3. Language Design Specification

### 3.1 Philosophy

Sage syntax follows three rules:

1. **Explicit over implicit.** Every type boundary is annotated. No hidden coercions. No implicit imports.
2. **Flat over nested.** Agents are top-level. No deeply nested blocks. Readable at a glance.
3. **Readable by humans and LLMs equally.** Regular grammar. Keyword-heavy. No symbolic magic.

### 3.2 Type System (POC Subset)

#### Primitive types

| Type     | Description                        | Example literal      |
|----------|------------------------------------|----------------------|
| `Int`    | 64-bit signed integer              | `42`, `-7`           |
| `Float`  | 64-bit IEEE 754                    | `3.14`, `-0.5`       |
| `Bool`   | Boolean                            | `true`, `false`      |
| `String` | UTF-8 string                       | `"hello"`            |
| `Unit`   | No value (void equivalent)         | implicit             |

#### Compound types (POC)

| Type       | Description                                  | Example                    |
|------------|----------------------------------------------|----------------------------|
| `List<T>`  | Homogeneous ordered list                     | `List<String>`             |
| `Option<T>`| Nullable value                               | `Option<String>`           |
| `Agent<T>` | Handle to a running agent of type T          | `Agent<Researcher>`        |

#### Inference type (special)

`Inferred<T>` — the return type of an `infer` expression. Wraps a value in a
stochastic context, signaling to the compiler and developer that the value
originates from a language model call and may be non-deterministic.

```sage
let summary: Inferred<String> = infer("Summarize {topic}")
```

### 3.3 Grammar (POC EBNF)

```ebnf
program         ::= top_level_decl* run_stmt

top_level_decl  ::= agent_decl | fn_decl

agent_decl      ::= "agent" IDENT "{" agent_body "}"

agent_body      ::= belief_decl* handler_decl*

belief_decl     ::= "belief" IDENT ":" type_expr

handler_decl    ::= "on" event_kind block

event_kind      ::= "start"
                  | "message" "(" IDENT ":" type_expr ")"
                  | "stop"

fn_decl         ::= "fn" IDENT "(" param_list? ")" "->" type_expr block

param_list      ::= param ("," param)*

param           ::= IDENT ":" type_expr

block           ::= "{" stmt* "}"

stmt            ::= let_stmt
                  | expr_stmt
                  | assign_stmt
                  | return_stmt
                  | if_stmt
                  | for_stmt

let_stmt        ::= "let" IDENT (":" type_expr)? "=" expr

assign_stmt     ::= IDENT "=" expr

return_stmt     ::= "return" expr?

if_stmt         ::= "if" expr block ("else" (block | if_stmt))?

for_stmt        ::= "for" IDENT "in" expr block

expr_stmt       ::= expr

expr            ::= infer_expr
                  | spawn_expr
                  | await_expr
                  | send_expr
                  | emit_expr
                  | call_expr
                  | binary_expr
                  | unary_expr
                  | primary_expr

infer_expr      ::= "infer" "(" string_template ("->" type_expr)? ")"

spawn_expr      ::= "spawn" IDENT "{" field_init_list? "}"

await_expr      ::= "await" expr

send_expr       ::= "send" "(" expr "," expr ")"

emit_expr       ::= "emit" "(" expr ")"

call_expr       ::= IDENT "(" arg_list? ")"
                  | "self" "." IDENT "(" arg_list? ")"

binary_expr     ::= expr binary_op expr

unary_expr      ::= unary_op expr

primary_expr    ::= IDENT
                  | "self" "." IDENT
                  | literal
                  | "(" expr ")"
                  | list_literal

binary_op       ::= "+" | "-" | "*" | "/" | "==" | "!=" | "<" | ">"
                  | "<=" | ">=" | "&&" | "||" | "++"

unary_op        ::= "!" | "-"

literal         ::= INT_LIT | FLOAT_LIT | BOOL_LIT | STRING_LIT

string_template ::= STRING_LIT  (* may contain {ident} interpolations *)

list_literal    ::= "[" (expr ("," expr)*)? "]"

field_init_list ::= field_init ("," field_init)*

field_init      ::= IDENT ":" expr

type_expr       ::= "Int" | "Float" | "Bool" | "String" | "Unit"
                  | "List" "<" type_expr ">"
                  | "Option" "<" type_expr ">"
                  | "Inferred" "<" type_expr ">"
                  | "Agent" "<" IDENT ">"
                  | IDENT
```

### 3.4 Core Language Constructs

#### 3.4.1 Agent declaration

```sage
agent Researcher {
    belief topic: String
    belief max_words: Int

    on start {
        let result: Inferred<String> = infer(
            "Summarize {self.topic} in under {self.max_words} words"
        )
        emit(result)
    }

    on message(msg: String) {
        let refined: Inferred<String> = infer(
            "Refine this summary based on feedback '{msg}': {self.topic}"
        )
        emit(refined)
    }

    on stop {
        print("Researcher shutting down")
    }
}
```

**Semantics:**
- `belief` declares typed agent state, initialized at spawn time.
- `on start` runs immediately when the agent is spawned.
- `on message(x: T)` runs when the agent receives a message of type T.
- `on stop` runs during graceful shutdown.
- `self.belief_name` accesses the agent's own beliefs.
- `emit(value)` sends the value to the agent's supervisor/awaiter.

#### 3.4.2 LLM inference primitive

```sage
let answer: Inferred<String> = infer("Answer this question: {question}")
```

**Semantics:**
- `infer(template)` is a language keyword, not a function call.
- The template is a string literal with `{ident}` interpolation — identifiers
  are resolved in the current scope.
- Return type is always `Inferred<T>` where T defaults to `String` unless
  annotated otherwise.
- In the POC, `Inferred<T>` unwraps transparently for downstream use. The
  type annotation exists to mark provenance, not to restrict operations.
- The runtime dispatches the call to the configured LLM backend
  (default: OpenAI-compatible API, configurable via `SAGE_LLM_URL` and
  `SAGE_API_KEY` environment variables).

#### 3.4.3 Spawn and await

```sage
// Spawn returns an Agent<T> handle
let r: Agent<Researcher> = spawn Researcher {
    topic: "Byzantine fault tolerance",
    max_words: 100
}

// Await blocks until the agent emits a value and terminates
let result: String = await r
```

**Semantics:**
- `spawn AgentName { field: value, ... }` creates and starts a new agent,
  initializing its beliefs from the field list. All declared beliefs must be
  provided.
- `await expr` suspends the current agent until the awaited agent emits its
  value and terminates. Returns the emitted value.
- An agent that never calls `emit` and whose `on start` block completes will
  cause `await` to return `Unit`.

#### 3.4.4 Message passing

```sage
// Send a message to a running agent
send(r, "Please be more concise")

// Receive happens via on message handler in the target agent
```

**Semantics:**
- `send(agent_handle, value)` enqueues a message in the target agent's mailbox.
- Messages are processed in FIFO order.
- The type of the value must match the type declared in the agent's
  `on message(x: T)` handler. A type mismatch is a compile-time error.
- In the POC, an agent without an `on message` handler silently drops messages
  (warning emitted by compiler).

#### 3.4.5 Built-in functions (prelude)

| Function             | Signature                          | Description                          |
|----------------------|------------------------------------|--------------------------------------|
| `print`              | `(String) -> Unit`                 | Print line to stdout                 |
| `print_fmt`          | `(String, ...args) -> Unit`        | Formatted print (basic)              |
| `len`                | `(List<T>) -> Int`                 | Length of a list                     |
| `push`               | `(List<T>, T) -> List<T>`          | Append to list (returns new list)    |
| `join`               | `(List<String>, String) -> String` | Join strings with separator          |
| `int_to_str`         | `(Int) -> String`                  | Convert Int to String                |
| `str_contains`       | `(String, String) -> Bool`         | Substring check                      |
| `sleep_ms`           | `(Int) -> Unit`                    | Sleep N milliseconds (async-safe)    |

#### 3.4.6 Top-level functions

```sage
fn greet(name: String) -> String {
    return "Hello, " ++ name
}
```

**Semantics:**
- Functions are pure in the POC (no agent spawning from within functions in
  this phase — agents are top-level constructs only).
- `++` is string concatenation.
- Recursion is allowed.

#### 3.4.7 Run statement

```sage
run Coordinator
```

**Semantics:**
- Exactly one `run` statement per program. Must appear at the bottom of the
  file.
- Names the entry-point agent. The runtime spawns this agent and waits for it
  to complete.
- `run` initializes the agent with no beliefs (entry agents must have no
  required beliefs, or all beliefs must have defaults — POC: entry agents
  must have zero beliefs).

### 3.5 Comments

```sage
// Single line comment — the only supported comment style in POC
```

### 3.6 String interpolation

Strings containing `{identifier}` patterns are treated as templates:

```sage
let name: String = "Alice"
let greeting: String = "Hello, {name}!"
// greeting == "Hello, Alice!"
```

Interpolation works in regular strings and in `infer` templates. Nested
expressions (`{a + b}`) are not supported in the POC — identifier-only
interpolation only.

### 3.7 Complete example program

```sage
// Multi-agent research summarizer

agent Researcher {
    belief topic: String

    on start {
        let summary: Inferred<String> = infer(
            "Write a concise 2-sentence summary of: {self.topic}"
        )
        emit(summary)
    }
}

agent Synthesizer {
    belief summaries: List<String>

    on start {
        let combined: String = join(self.summaries, "\n---\n")
        let synthesis: Inferred<String> = infer(
            "Synthesize these research summaries into one coherent paragraph:\n{combined}"
        )
        emit(synthesis)
    }
}

agent Coordinator {
    on start {
        let topics: List<String> = [
            "quantum computing",
            "CRISPR gene editing",
            "nuclear fusion energy"
        ]

        let r1: Agent<Researcher> = spawn Researcher { topic: "quantum computing" }
        let r2: Agent<Researcher> = spawn Researcher { topic: "CRISPR gene editing" }
        let r3: Agent<Researcher> = spawn Researcher { topic: "nuclear fusion energy" }

        let s1: String = await r1
        let s2: String = await r2
        let s3: String = await r3

        let summaries: List<String> = [s1, s2, s3]
        let synth: Agent<Synthesizer> = spawn Synthesizer { summaries: summaries }
        let report: String = await synth

        print("=== Research Report ===")
        print(report)
    }
}

run Coordinator
```

---

## 4. Runtime Semantics

### 4.1 Execution model

Each agent runs as an independent **tokio task**. Agent tasks communicate via
**tokio channels** (mpsc for messages, oneshot for the emit/await pair).

```
┌─────────────────────────────────────────────────────────────┐
│                        Sage Runtime                         │
│                                                             │
│  ┌──────────────┐    spawn     ┌──────────────┐            │
│  │  Coordinator │ ──────────►  │  Researcher  │            │
│  │   (task)     │              │   (task)     │            │
│  │              │ ◄──(emit)──  │              │            │
│  │              │   await      └──────────────┘            │
│  └──────────────┘                                           │
│          │                                                   │
│         run                                                  │
│          │                                                   │
│  ┌───────▼──────────────────────────────────────────────┐  │
│  │                  tokio runtime                        │  │
│  │  (single-threaded in POC, multi-thread in future)    │  │
│  └───────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

### 4.2 Agent lifecycle

```
spawned → on_start running → [on_message* running] → emit → terminated
                                                          ↓
                                                    awaiter unblocked
```

States:
- **Spawned**: tokio task created, beliefs initialized
- **Running**: `on start` block executing
- **Waiting**: `on start` has completed, agent is listening for messages
- **Terminated**: `emit` has been called, task exits

An agent terminates after calling `emit`. Message handlers may call `emit`
too, which also terminates the agent.

### 4.3 Supervision (POC: minimal)

In the POC, supervision is simple:
- If an agent's task panics, its awaiter receives an error and the runtime
  prints a supervision event to stderr.
- No restart strategy in POC (just fail-fast with clear error message).
- Budget supervision is not in POC scope.

Full OTP-style supervision trees are a v0.2 feature.

### 4.4 LLM backend

The runtime dispatches `infer` calls via **reqwest** to an OpenAI-compatible
chat completions endpoint.

**Configuration (environment variables):**

| Variable         | Default                                 | Description             |
|------------------|-----------------------------------------|-------------------------|
| `SAGE_LLM_URL`   | `https://api.openai.com/v1/chat/completions` | LLM endpoint       |
| `SAGE_API_KEY`   | _(required)_                            | API key                 |
| `SAGE_MODEL`     | `gpt-4o-mini`                           | Model name              |
| `SAGE_MAX_TOKENS`| `1024`                                  | Max tokens per call     |
| `SAGE_TIMEOUT_MS`| `30000`                                 | Request timeout (ms)    |

**Ollama compatibility:** Set `SAGE_LLM_URL=http://localhost:11434/v1/chat/completions`
and `SAGE_API_KEY=ollama` to use a local Ollama server.

**Infer call format:** Each `infer` dispatches as a single user message with
the rendered template as content. System prompt is fixed:
`"You are a helpful, concise assistant. Respond only with the requested output, no preamble."`

### 4.5 Async behavior

- `await agent_handle` yields the current tokio task until the awaited agent
  emits. This is non-blocking — other agents continue to run.
- Multiple `spawn` calls followed by multiple `await` calls run the spawned
  agents concurrently (they are all started before any is awaited).
- `sleep_ms(n)` is `tokio::time::sleep` under the hood.

---

## 5. Compiler Architecture

### 5.1 Pipeline overview

```
source.sg
    │
    ▼
┌─────────┐
│  Lexer  │  logos crate — tokenizes source into a flat Token stream
└────┬────┘
     │  Vec<Token>
     ▼
┌─────────┐
│ Parser  │  chumsky crate — produces typed AST with span info
└────┬────┘
     │  Program (AST)
     ▼
┌──────────────┐
│ Name Resolver│  Resolves identifiers, builds symbol table
└──────┬───────┘
       │  Program + SymbolTable
       ▼
┌──────────────┐
│ Type Checker │  Hand-rolled bidirectional type checker
└──────┬───────┘
       │  TypedProgram
       ▼
┌─────────────┐
│ Interpreter │  Tree-walking evaluator with async support
└──────┬──────┘
       │
       ▼
   execution
```

### 5.2 AST node types (Rust)

```rust
// Core AST nodes — simplified for spec clarity

pub struct Program {
    pub agents: Vec<AgentDecl>,
    pub functions: Vec<FnDecl>,
    pub run_agent: Ident,
    pub span: Span,
}

pub struct AgentDecl {
    pub name: Ident,
    pub beliefs: Vec<BeliefDecl>,
    pub handlers: Vec<HandlerDecl>,
    pub span: Span,
}

pub struct BeliefDecl {
    pub name: Ident,
    pub ty: TypeExpr,
    pub span: Span,
}

pub struct HandlerDecl {
    pub event: EventKind,
    pub body: Block,
    pub span: Span,
}

pub enum EventKind {
    Start,
    Message { param_name: Ident, param_ty: TypeExpr },
    Stop,
}

pub struct Block {
    pub stmts: Vec<Stmt>,
    pub span: Span,
}

pub enum Stmt {
    Let { name: Ident, ty: Option<TypeExpr>, value: Expr, span: Span },
    Assign { name: Ident, value: Expr, span: Span },
    Return { value: Option<Expr>, span: Span },
    If { condition: Expr, then_block: Block, else_block: Option<Block>, span: Span },
    For { var: Ident, iter: Expr, body: Block, span: Span },
    Expr { expr: Expr, span: Span },
}

pub enum Expr {
    Infer { template: StringTemplate, result_ty: Option<TypeExpr>, span: Span },
    Spawn { agent: Ident, fields: Vec<FieldInit>, span: Span },
    Await { handle: Box<Expr>, span: Span },
    Send { handle: Box<Expr>, message: Box<Expr>, span: Span },
    Emit { value: Box<Expr>, span: Span },
    Call { name: Ident, args: Vec<Expr>, span: Span },
    SelfField { field: Ident, span: Span },
    Binary { op: BinOp, left: Box<Expr>, right: Box<Expr>, span: Span },
    Unary { op: UnOp, operand: Box<Expr>, span: Span },
    List { elements: Vec<Expr>, span: Span },
    Literal { value: Literal, span: Span },
    Ident { name: Ident, span: Span },
}
```

### 5.3 Type checker rules

The type checker performs a single forward pass. Key rules:

**Belief access:**
- `self.x` resolves to the belief `x` of the enclosing agent.
- Using `self` outside an agent handler is a compile error.

**Infer expression:**
- `infer(template)` has type `Inferred<String>` by default.
- `infer(template -> T)` has type `Inferred<T>`.
- In the POC, `Inferred<T>` is assignment-compatible with `T` (transparent
  unwrapping). The type annotation is tracked for informational purposes.

**Spawn expression:**
- `spawn A { f1: v1, ... }` checks that all beliefs of agent `A` are provided,
  no extras, and each value's type matches the belief's declared type.
- Result type is `Agent<A>`.

**Await expression:**
- `await expr` where `expr: Agent<A>` returns the type of the value passed to
  `emit` in agent `A`'s handlers. The type checker infers this by scanning the
  agent's `emit` calls. If multiple `emit` calls exist with different types,
  it's an error. If no `emit` call exists, type is `Unit`.

**Send expression:**
- `send(handle, msg)` where `handle: Agent<A>` checks that `A` has an
  `on message(x: T)` handler and that `msg` has type `T`.

**Message handler parameter:**
- `on message(x: T)` introduces `x: T` into the block's scope.

### 5.4 Error reporting

All errors use **miette** for rich, human-readable output with source spans.

Required error codes for POC:

| Code    | Description                                         |
|---------|-----------------------------------------------------|
| E001    | Unexpected token                                    |
| E002    | Undefined identifier                                |
| E003    | Type mismatch                                       |
| E004    | Missing belief in spawn                             |
| E005    | Extra field in spawn                                |
| E006    | Await on non-agent type                             |
| E007    | Send type mismatch                                  |
| E008    | No `run` statement                                  |
| E009    | Multiple `run` statements                           |
| E010    | `self` used outside agent handler                   |
| E011    | Emit type conflict (multiple emit types in agent)   |
| E012    | Entry agent has required beliefs                    |

Example error output:

```
Error[E003]: type mismatch
  --> examples/research.sg:14:32
   |
14 |     let r = spawn Researcher { topic: 42 }
   |                                        ^^ expected String, found Int
   |
   = note: belief `topic` is declared as String in agent Researcher
```

### 5.5 Interpreter (tree-walking evaluator)

The interpreter uses a **Value** enum to represent runtime values:

```rust
pub enum Value {
    Int(i64),
    Float(f64),
    Bool(bool),
    String(String),
    Unit,
    List(Vec<Value>),
    AgentHandle(AgentHandle),
    Inferred(Box<Value>),  // transparent wrapper
}
```

Agent execution context:

```rust
pub struct AgentContext {
    pub name: String,
    pub beliefs: HashMap<String, Value>,
    pub mailbox: mpsc::Receiver<Value>,
    pub emit_tx: oneshot::Sender<Value>,
}
```

Environment (scope chain):

```rust
pub struct Env {
    pub vars: HashMap<String, Value>,
    pub parent: Option<Box<Env>>,
    pub agent_ctx: Option<Arc<AgentContext>>,
}
```

---

## 6. Project Structure

```
sage/
├── Cargo.toml                  # workspace root
├── Cargo.lock
├── README.md
├── LICENSE
│
├── crates/
│   ├── sage-lexer/             # logos-based lexer
│   │   ├── Cargo.toml
│   │   └── src/
│   │       ├── lib.rs
│   │       ├── token.rs        # Token enum + logos derives
│   │       └── tests.rs
│   │
│   ├── sage-parser/            # chumsky-based parser
│   │   ├── Cargo.toml
│   │   └── src/
│   │       ├── lib.rs
│   │       ├── ast.rs          # AST node definitions
│   │       ├── parser.rs       # chumsky parser combinators
│   │       └── tests.rs
│   │
│   ├── sage-types/             # shared type definitions
│   │   ├── Cargo.toml
│   │   └── src/
│   │       ├── lib.rs
│   │       ├── ty.rs           # TypeExpr, resolved Type
│   │       └── span.rs         # Span type
│   │
│   ├── sage-checker/           # name resolution + type checker
│   │   ├── Cargo.toml
│   │   └── src/
│   │       ├── lib.rs
│   │       ├── resolver.rs     # name resolution pass
│   │       ├── checker.rs      # type checking pass
│   │       ├── env.rs          # type environment
│   │       └── tests.rs
│   │
│   ├── sage-interpreter/       # tree-walking interpreter + runtime
│   │   ├── Cargo.toml
│   │   └── src/
│   │       ├── lib.rs
│   │       ├── value.rs        # Value enum
│   │       ├── env.rs          # runtime environment
│   │       ├── eval.rs         # expression evaluator
│   │       ├── agent.rs        # agent task spawning + lifecycle
│   │       ├── llm.rs          # LLM backend (reqwest)
│   │       ├── prelude.rs      # built-in functions
│   │       └── tests.rs
│   │
│   └── sage-cli/               # binary entry point
│       ├── Cargo.toml
│       └── src/
│           └── main.rs         # CLI: sage run <file>
│
├── examples/
│   ├── hello.sg                # minimal: single agent, no LLM
│   ├── infer.sg                # single agent with LLM call
│   ├── two_agents.sg           # spawn + await between two agents
│   └── research.sg             # full demo: coordinator + multi-agent
│
└── tests/
    ├── lexer_tests.rs
    ├── parser_tests.rs
    ├── checker_tests.rs
    └── interpreter_tests.rs
```

---

## 7. Dependency Manifest

### Root `Cargo.toml`

```toml
[workspace]
members = [
    "crates/sage-lexer",
    "crates/sage-parser",
    "crates/sage-types",
    "crates/sage-checker",
    "crates/sage-interpreter",
    "crates/sage-cli",
]
resolver = "2"
```

### `sage-lexer/Cargo.toml`

```toml
[dependencies]
logos = "0.14"

[dev-dependencies]
pretty_assertions = "1"
```

### `sage-parser/Cargo.toml`

```toml
[dependencies]
sage-lexer = { path = "../sage-lexer" }
sage-types = { path = "../sage-types" }
chumsky = { version = "0.9", features = ["label"] }

[dev-dependencies]
pretty_assertions = "1"
```

### `sage-checker/Cargo.toml`

```toml
[dependencies]
sage-parser = { path = "../sage-parser" }
sage-types = { path = "../sage-types" }
miette = { version = "5", features = ["fancy"] }
thiserror = "1"
```

### `sage-interpreter/Cargo.toml`

```toml
[dependencies]
sage-parser = { path = "../sage-parser" }
sage-checker = { path = "../sage-checker" }
sage-types = { path = "../sage-types" }
tokio = { version = "1", features = ["full"] }
reqwest = { version = "0.12", features = ["json"] }
serde = { version = "1", features = ["derive"] }
serde_json = "1"
miette = { version = "5", features = ["fancy"] }
thiserror = "1"
```

### `sage-cli/Cargo.toml`

```toml
[dependencies]
sage-lexer = { path = "../sage-lexer" }
sage-parser = { path = "../sage-parser" }
sage-checker = { path = "../sage-checker" }
sage-interpreter = { path = "../sage-interpreter" }
tokio = { version = "1", features = ["full"] }
clap = { version = "4", features = ["derive"] }
miette = { version = "5", features = ["fancy"] }
```

---

## 8. Ordered Task List

Tasks are grouped into milestones. Each task has an ID, estimated effort in
hours (for a senior Rust developer), and explicit dependencies.

---

### Milestone 1: Project Scaffolding
**Goal:** Working Cargo workspace, CI, and the ability to `cargo build` cleanly.

---

#### TASK-001 — Initialize Cargo workspace
**Effort:** 1h | **Deps:** none

- Create repo with `Cargo.toml` workspace definition
- Add all six crate stubs with empty `lib.rs` / `main.rs`
- Verify `cargo build` succeeds (zero warnings policy)
- Add `.gitignore`, `LICENSE` (MIT), `README.md` stub

---

#### TASK-002 — Set up CI (GitHub Actions)
**Effort:** 1h | **Deps:** TASK-001

- Workflow: `cargo check`, `cargo test`, `cargo clippy -- -D warnings`
- Run on push to `main` and all PRs
- Pin Rust toolchain to stable with `rust-toolchain.toml`

---

#### TASK-003 — Define shared types crate (`sage-types`)
**Effort:** 2h | **Deps:** TASK-001

- Define `Span { start: usize, end: usize, source: Arc<str> }`
- Define `Ident(String, Span)`
- Define `TypeExpr` enum (all primitive and compound types)
- Write unit tests for TypeExpr equality and display

---

### Milestone 2: Lexer
**Goal:** Source text → flat token stream, with full test coverage.

---

#### TASK-004 — Define Token enum
**Effort:** 2h | **Deps:** TASK-003

Define all tokens using `logos`:

```rust
#[derive(Logos, Debug, PartialEq, Clone)]
pub enum Token {
    // Keywords
    #[token("agent")]   KwAgent,
    #[token("belief")]  KwBelief,
    #[token("on")]      KwOn,
    #[token("start")]   KwStart,
    #[token("stop")]    KwStop,
    #[token("message")] KwMessage,
    #[token("infer")]   KwInfer,
    #[token("spawn")]   KwSpawn,
    #[token("await")]   KwAwait,
    #[token("send")]    KwSend,
    #[token("emit")]    KwEmit,
    #[token("run")]     KwRun,
    #[token("fn")]      KwFn,
    #[token("let")]     KwLet,
    #[token("return")]  KwReturn,
    #[token("if")]      KwIf,
    #[token("else")]    KwElse,
    #[token("for")]     KwFor,
    #[token("in")]      KwIn,
    #[token("self")]    KwSelf,
    #[token("true")]    KwTrue,
    #[token("false")]   KwFalse,

    // Type keywords
    #[token("Int")]     TyInt,
    #[token("Float")]   TyFloat,
    #[token("Bool")]    TyBool,
    #[token("String")]  TyString,
    #[token("Unit")]    TyUnit,
    #[token("List")]    TyList,
    #[token("Option")]  TyOption,
    #[token("Inferred")]TyInferred,
    #[token("Agent")]   TyAgent,

    // Literals
    #[regex(r"-?[0-9]+", priority = 2)]     IntLit,
    #[regex(r"-?[0-9]+\.[0-9]+")]           FloatLit,
    #[regex(r#""([^"\\]|\\.)*""#)]          StringLit,

    // Identifiers
    #[regex(r"[a-zA-Z_][a-zA-Z0-9_]*")]    Ident,

    // Punctuation
    #[token("{")]   LBrace,
    #[token("}")]   RBrace,
    #[token("(")]   LParen,
    #[token(")")]   RParen,
    #[token("[")]   LBracket,
    #[token("]")]   RBracket,
    #[token(",")]   Comma,
    #[token(":")]   Colon,
    #[token(";")]   Semi,      // optional, not required
    #[token(".")]   Dot,
    #[token("->")]  Arrow,
    #[token("=")]   Eq,
    #[token("==")]  EqEq,
    #[token("!=")]  Ne,
    #[token("<")]   Lt,
    #[token(">")]   Gt,
    #[token("<=")]  Le,
    #[token(">=")]  Ge,
    #[token("+")]   Plus,
    #[token("-")]   Minus,
    #[token("*")]   Star,
    #[token("/")]   Slash,
    #[token("!")]   Bang,
    #[token("&&")]  And,
    #[token("||")]  Or,
    #[token("++")]  PlusPlus,  // string concat

    // Whitespace and comments (skip)
    #[regex(r"[ \t\r\n]+",    logos::skip)]
    #[regex(r"//[^\n]*",      logos::skip)]
    Whitespace,

    Error,
}
```

---

#### TASK-005 — Implement lexer public API
**Effort:** 2h | **Deps:** TASK-004

- `pub fn lex(source: &str) -> Result<Vec<(Token, Span)>, LexError>`
- Collect all tokens with their source spans
- Report all lex errors (don't stop at first)
- Add `LexError` type with miette integration

---

#### TASK-006 — Lexer tests
**Effort:** 2h | **Deps:** TASK-005

Write tests covering:
- All keywords tokenize correctly
- Integer, float, bool, string literals
- Identifiers vs keywords (e.g. `agent` vs `agent_name`)
- Comments are skipped
- Whitespace is skipped
- String literals with escape sequences
- Error tokens produce `LexError`
- Full agent declaration tokenizes correctly (integration test)

---

### Milestone 3: Parser
**Goal:** Token stream → typed AST.

---

#### TASK-007 — Define AST types
**Effort:** 3h | **Deps:** TASK-003

- Define all AST node types from §5.2 in `sage-parser/src/ast.rs`
- All nodes carry `Span`
- Derive `Debug`, `Clone`, `PartialEq`
- Write `Display` implementations for key nodes (useful in error messages)

---

#### TASK-008 — Parser: top-level structure
**Effort:** 3h | **Deps:** TASK-007

Using chumsky, implement parsers for:
- `program` — sequence of `agent_decl | fn_decl` ending with `run_stmt`
- `run_stmt` — `run IDENT`
- Report E008 if no `run` statement
- Report E009 if multiple `run` statements

---

#### TASK-009 — Parser: agent declarations
**Effort:** 4h | **Deps:** TASK-008

- `agent_decl` — `agent IDENT { agent_body }`
- `belief_decl` — `belief IDENT : type_expr`
- `handler_decl` — `on start { block }` | `on message(x: T) { block }` | `on stop { block }`
- `type_expr` — all types from §3.2 including generics `List<T>`, `Agent<T>`

---

#### TASK-010 — Parser: statements
**Effort:** 4h | **Deps:** TASK-009

- `let_stmt` with optional type annotation
- `assign_stmt`
- `return_stmt`
- `if_stmt` with optional `else`
- `for_stmt`
- `expr_stmt`

---

#### TASK-011 — Parser: expressions
**Effort:** 5h | **Deps:** TASK-010

- Binary expressions with correct precedence (standard C-like: `*` > `+` > `<` > `&&` > `||`)
- Unary expressions
- `infer(template)` and `infer(template -> T)`
- `spawn Agent { fields }` with field init list
- `await expr`
- `send(handle, msg)`
- `emit(value)`
- Function calls `name(args)`
- `self.field` access
- List literals `[a, b, c]`
- String literal with interpolation (parse `{ident}` patterns into `StringTemplate`)
- Parenthesized expressions

---

#### TASK-012 — Parser: function declarations
**Effort:** 2h | **Deps:** TASK-011

- `fn name(param: Type, ...) -> ReturnType { block }`
- Param list parsing
- Return type annotation

---

#### TASK-013 — Parser error recovery
**Effort:** 3h | **Deps:** TASK-012

- chumsky's `recovery` strategies for common errors
- Report multiple errors rather than stopping at first
- Error messages keyed to E001 code with useful spans

---

#### TASK-014 — Parser tests
**Effort:** 4h | **Deps:** TASK-013

Write tests covering:
- Parses the full research.sg example correctly
- All statement types
- All expression types
- Operator precedence (2+3*4 == 14)
- Type annotations on let bindings
- String interpolation in template
- Error cases: missing brace, missing colon, bad type

---

### Milestone 4: Name Resolution + Type Checker
**Goal:** AST + SymbolTable → TypedProgram, with compile-time error detection.

---

#### TASK-015 — Name resolver
**Effort:** 4h | **Deps:** TASK-014

- First pass: collect all agent and function names into global symbol table
- Second pass: resolve all identifier references
- Report E002 for undefined identifiers
- Report E010 for `self` used outside agent handler
- Build scope chain for blocks (let bindings shadow outer names)

---

#### TASK-016 — Type environment
**Effort:** 2h | **Deps:** TASK-015

- `TypeEnv` mapping `Ident → Type`
- Scope push/pop for block-level scoping
- Agent belief types stored per-agent
- Function signatures stored globally
- Prelude built-in function signatures pre-loaded

---

#### TASK-017 — Type checker: agents
**Effort:** 4h | **Deps:** TASK-016

- Check `on start`, `on message`, `on stop` handler bodies
- Introduce message parameter into scope for `on message`
- Infer emit type for each agent (scan all `emit` calls)
- Report E011 if multiple emit types found
- Check `self.field` access against belief declarations

---

#### TASK-018 — Type checker: expressions
**Effort:** 5h | **Deps:** TASK-017

- `infer(template)` → `Inferred<String>` (or annotated type)
- `spawn A { ... }` → `Agent<A>`: check all beliefs provided (E004/E005), types match (E003)
- `await expr` → emit type of target agent: check expr is `Agent<T>` (E006)
- `send(handle, msg)` → `Unit`: check handle is agent with message handler, msg type matches (E007)
- `emit(value)` → `Unit`
- Binary expressions: type check operands, infer result type
- List literals: all elements same type → `List<T>`
- Prelude function calls: check arity and argument types

---

#### TASK-019 — Type checker: statements
**Effort:** 3h | **Deps:** TASK-018

- `let x: T = e` — check e has type T (or infer T from e if no annotation)
- `assign x = e` — check x is in scope, e has same type as x
- `return e` — check e matches function return type
- `if e { } else { }` — e must be Bool
- `for x in e { }` — e must be `List<T>`, x has type T in body

---

#### TASK-020 — Type checker: functions
**Effort:** 2h | **Deps:** TASK-019

- Check return type matches declared type
- Check all code paths return (or return Unit)
- Check recursion (allowed — just needs to be in symbol table before body is checked)

---

#### TASK-021 — Entry agent validation
**Effort:** 1h | **Deps:** TASK-020

- Check that the agent named in `run` has no required beliefs (E012)
- Check `run` agent exists (E002)

---

#### TASK-022 — Type checker tests
**Effort:** 4h | **Deps:** TASK-021

Write tests covering all 12 error codes:
- Valid programs produce no errors
- Each error code triggered by appropriate invalid program
- Type inference on let without annotation
- Await type matches emit type
- Send type mismatch caught

---

### Milestone 5: Interpreter & Runtime
**Goal:** TypedProgram → execution, with real LLM calls and multi-agent concurrency.

---

#### TASK-023 — Value enum and runtime environment
**Effort:** 3h | **Deps:** TASK-022

- Define `Value` enum (§5.5)
- Implement `Display` for `Value`
- Define `Env` with push/pop scope and parent chain
- Agent context struct with beliefs and channel handles

---

#### TASK-024 — Prelude built-in functions
**Effort:** 2h | **Deps:** TASK-023

Implement all prelude functions from §3.4.5:
- `print`, `print_fmt`
- `len`, `push`, `join`
- `int_to_str`, `str_contains`
- `sleep_ms`

Each as a Rust async function operating on `Value` arguments.

---

#### TASK-025 — Expression evaluator
**Effort:** 5h | **Deps:** TASK-024

- `async fn eval_expr(expr: &Expr, env: &mut Env) -> Result<Value, RuntimeError>`
- All expression variants from §5.2
- String interpolation: substitute `{ident}` from env
- Binary and unary operations
- List construction
- Self-field access via agent context

---

#### TASK-026 — Statement evaluator
**Effort:** 3h | **Deps:** TASK-025

- `async fn eval_stmt(stmt: &Stmt, env: &mut Env) -> Result<StmtResult, RuntimeError>`
- `StmtResult` enum: `Continue`, `Return(Value)`, `Emit(Value)`
- All statement variants from §3.3

---

#### TASK-027 — Agent task spawning
**Effort:** 4h | **Deps:** TASK-026

- `async fn spawn_agent(decl: &AgentDecl, beliefs: HashMap<String, Value>) -> AgentHandle`
- Creates `tokio::spawn` task
- Sets up `mpsc` channel for messages (mailbox)
- Sets up `oneshot` channel for emit result
- Runs `on start` handler in the task
- Returns `AgentHandle` containing sender + await future

---

#### TASK-028 — Await and send implementation
**Effort:** 3h | **Deps:** TASK-027

- `await handle` → awaits the oneshot receiver, returns emitted value
- `send(handle, msg)` → sends on the mpsc sender
- Handle agent task completion and panic propagation to awaiter
- Test: spawn + await round-trip

---

#### TASK-029 — LLM backend
**Effort:** 3h | **Deps:** TASK-023

- `async fn llm_infer(prompt: String) -> Result<String, LlmError>`
- Read config from env vars (§4.4)
- Construct OpenAI-compatible chat completions request
- Parse response, extract content string
- Error handling: timeout, non-200 response, parse failure
- Retry once on timeout (simple retry, no exponential backoff in POC)

---

#### TASK-030 — Wire infer expression to LLM backend
**Effort:** 2h | **Deps:** TASK-025, TASK-029

- `Expr::Infer` evaluator calls `llm_infer` with rendered template
- Returns `Value::Inferred(Box::new(Value::String(response)))`
- Transparent unwrap when assigned to non-Inferred target

---

#### TASK-031 — Runtime entry point
**Effort:** 2h | **Deps:** TASK-028, TASK-030

- `async fn run_program(program: &TypedProgram) -> Result<(), RuntimeError>`
- Look up entry agent by name
- Spawn it with empty beliefs
- Await its completion
- Print runtime errors with miette formatting

---

#### TASK-032 — Minimal supervision (fail-fast)
**Effort:** 2h | **Deps:** TASK-031

- Catch panics in agent tasks via `tokio::task::catch_unwind` or `JoinHandle` error
- Print supervision event to stderr: `[SAGE SUPERVISOR] agent 'X' terminated unexpectedly`
- Propagate as `RuntimeError::AgentPanic` to awaiter

---

#### TASK-033 — Interpreter tests
**Effort:** 4h | **Deps:** TASK-032

- hello.sg runs and prints correctly
- two_agents.sg: spawn + await produces correct value
- for loop and list operations work correctly
- String interpolation works in regular strings
- LLM backend mocked for tests (inject mock via config or trait object)
- Mock returns deterministic responses for test assertions

---

### Milestone 6: CLI
**Goal:** `sage run file.sg` works end-to-end.

---

#### TASK-034 — CLI binary with clap
**Effort:** 2h | **Deps:** TASK-033

```
sage <COMMAND>

Commands:
  run    <FILE>    Run a .sg program
  check  <FILE>    Type-check without running
  lex    <FILE>    Print token stream (debug)
  parse  <FILE>    Print AST (debug)
```

- `sage run file.sg` — lex → parse → check → interpret
- `sage check file.sg` — lex → parse → check, print errors or "OK"
- `sage lex file.sg` — print tokens (debug aid)
- `sage parse file.sg` — print AST as pretty debug output
- All errors rendered via miette with source context

---

#### TASK-035 — Release binary and README
**Effort:** 2h | **Deps:** TASK-034

- `cargo build --release` produces single `sage` binary
- README.md with:
  - What Sage is (3 sentences)
  - Installation (`cargo install --path .`)
  - Quick start (hello.sg + infer.sg)
  - Environment variable config
  - Full research.sg demo with expected output
- Add syntax highlighting hint for GitHub (`.gitattributes`)

---

### Milestone 7: Examples and Demo
**Goal:** Four progressively complex example programs, research.sg running cleanly.

---

#### TASK-036 — hello.sg
**Effort:** 30m | **Deps:** TASK-035

```sage
agent Hello {
    on start {
        print("Hello from Sage!")
        emit("done")
    }
}

run Hello
```

Verifies: basic execution, no LLM needed.

---

#### TASK-037 — infer.sg
**Effort:** 30m | **Deps:** TASK-036

```sage
agent Greeter {
    belief name: String

    on start {
        let greeting: Inferred<String> = infer(
            "Write a warm, one-sentence greeting for someone named {self.name}"
        )
        print(greeting)
        emit(greeting)
    }
}

agent Main {
    on start {
        let g: Agent<Greeter> = spawn Greeter { name: "Pete" }
        let result: String = await g
        print("Agent said: " ++ result)
    }
}

run Main
```

Verifies: beliefs, LLM call, spawn + await.

---

#### TASK-038 — two_agents.sg
**Effort:** 1h | **Deps:** TASK-037

A writer agent and a critic agent. Writer generates text, Coordinator sends
it to Critic for feedback, Critic responds with a refined version.
Demonstrates: `send`, `on message`, bidirectional communication.

---

#### TASK-039 — research.sg (full demo)
**Effort:** 1h | **Deps:** TASK-038

The complete multi-agent research summarizer from §3.7. Three parallel
Researcher agents + one Synthesizer agent + Coordinator. This is the primary
demo for the GitHub README and any public announcement.

---

### Milestone 8: Polish
**Goal:** Production-quality first impression for the open-source launch.

---

#### TASK-040 — Error message polish
**Effort:** 2h | **Deps:** TASK-039

Review all 12 error codes. Each error must:
- Show the relevant source span
- Have a helpful "note" or "help" suggestion where possible
- Be tested in checker tests

---

#### TASK-041 — Compiler warning for unused beliefs
**Effort:** 1h | **Deps:** TASK-040

- Warn (not error) if a belief is declared but never accessed via `self.x`
- Warning format: `Warning[W001]: belief 'x' is declared but never used`
- Suppressable with `// sage: allow(unused_belief)` comment (POC: just implement the warning)

---

#### TASK-042 — `CONTRIBUTING.md` and issue templates
**Effort:** 1h | **Deps:** TASK-041

- `CONTRIBUTING.md`: build instructions, crate overview, how to add a test
- GitHub issue templates: bug report, feature request, language design proposal

---

## 9. Demo Program

The primary demo — `examples/research.sg` — is defined in §3.7. Here is the
expected terminal output when run against a real LLM:

```
$ SAGE_API_KEY=sk-... sage run examples/research.sg

[sage] spawning Coordinator
[sage] Coordinator spawning 3 Researcher agents...
[sage] Researcher(quantum computing) → infer started
[sage] Researcher(CRISPR gene editing) → infer started
[sage] Researcher(nuclear fusion energy) → infer started
[sage] Researcher(CRISPR gene editing) → emitted
[sage] Researcher(nuclear fusion energy) → emitted
[sage] Researcher(quantum computing) → emitted
[sage] Synthesizer → infer started
[sage] Synthesizer → emitted

=== Research Report ===
Quantum computing leverages quantum mechanical phenomena like superposition and
entanglement to solve problems intractable for classical computers, with current
systems achieving increasing qubit counts but still limited by decoherence.
CRISPR-Cas9 has revolutionized genetic engineering by enabling precise, low-cost
DNA editing, opening avenues in medicine, agriculture, and fundamental biology,
though ethical questions around germline editing remain contested. Nuclear fusion
promises virtually limitless clean energy by replicating stellar processes on Earth,
with recent breakthroughs at NIF and ITER suggesting commercial viability may
arrive within decades. Together, these technologies represent a convergence of
physical, biological, and computational frontiers that may fundamentally reshape
civilization in the coming century.
```

---

## 10. Success Criteria

The POC is complete when all of the following are true:

| Criterion | Verification |
|-----------|-------------|
| `cargo build --release` succeeds with zero warnings | CI |
| `cargo test` passes all tests | CI |
| `sage run examples/hello.sg` prints "Hello from Sage!" | Manual |
| `sage check examples/research.sg` reports "OK" | Manual |
| `sage run examples/research.sg` produces multi-agent LLM output | Manual |
| All 12 error codes produce correct miette output | Checker tests |
| `sage run` with bad syntax shows line/column and error code | Manual |
| Type mismatch in spawn field caught at compile time | Checker test |
| Three agents in research.sg run concurrently (not sequentially) | Timing check |
| README enables a new developer to run the demo in < 5 minutes | Peer review |

---

## 11. Out of Scope

The following are explicitly deferred to post-POC milestones and should not
be designed around or worked toward during the POC phase:

- **Algebraic effects system** — v0.2. The foundation for capability-based
  security and composable LLM handlers.
- **Session types** — v0.3. Typed multi-agent communication protocols.
- **Cranelift/LLVM codegen** — v0.4. Native binary compilation.
- **Package manager / module system** — v0.3.
- **LSP server** — v0.3.
- **Generics / parametric polymorphism** — v0.3.
- **Pattern matching** — v0.2.
- **Structs / records** — v0.2 (beliefs are the POC substitute).
- **Error type / Result<T,E>** — v0.2.
- **Budget supervision** — v0.2.
- **DSPy-style compiler optimization** — v0.5+.
- **Multi-file programs / imports** — v0.2.
- **WASM compilation target** — v0.4.

---

*End of RFC-0001*
