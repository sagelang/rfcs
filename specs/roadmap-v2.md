# Sage v2.0 — The Steward Architecture
## Unified Language Specification

**Version:** 2.0.0-draft
**Author:** Sage Contributors
**Status:** Draft
**Depends on:** All v0.x and v1.x RFCs (RFC-0001 through RFC-0010)

---

## Table of Contents

1. [Vision & Motivation](#1-vision--motivation)
2. [Prerequisites — What Must Land Before v2.0](#2-prerequisites--what-must-land-before-v20)
3. [New Language Features](#3-new-language-features)
   - 3.1 [Persistent Beliefs](#31-persistent-beliefs)
   - 3.2 [Tool Declarations](#32-tool-declarations)
   - 3.3 [Supervision Trees](#33-supervision-trees)
   - 3.4 [Session Types](#34-session-types)
   - 3.5 [Algebraic Effects](#35-algebraic-effects)
   - 3.6 [Generics](#36-generics)
   - 3.7 [Payload-Carrying Enum Variants](#37-payload-carrying-enum-variants)
   - 3.8 [Agent Lifecycle Hooks](#38-agent-lifecycle-hooks)
   - 3.9 [Observability Primitives](#39-observability-primitives)
4. [Type System Changes](#4-type-system-changes)
5. [Runtime Semantics](#5-runtime-semantics)
6. [Standard Library — the Commons](#6-standard-library--the-commons)
7. [Grammar Changes](#7-grammar-changes)
8. [Compiler Architecture Changes](#8-compiler-architecture-changes)
9. [New Error Codes](#9-new-error-codes)
10. [The Steward Pattern](#10-the-steward-pattern)
11. [Reference Programs](#11-reference-programs)
    - 11.1 [Full-Stack Web App Steward](#111-full-stack-web-app-steward)
    - 11.2 [Background Service Daemon](#112-background-service-daemon)
12. [Migration from v1.x](#12-migration-from-v1x)
13. [Implementation Roadmap](#13-implementation-roadmap)
14. [Open Questions](#14-open-questions)

---

## 1. Vision & Motivation

Sage v1.x proved the core thesis: agents, beliefs, and LLM inference as first-class language constructs produce programs that are simpler, safer, and more readable than equivalent Python framework code. The research pipeline, the debate program, the code reviewer — all demonstrated that multi-agent orchestration belongs at the language level.

v2.0 pursues a deeper thesis: **agents as stewards of long-lived systems**.

In v1.x, agents are ephemeral. You spawn them, they do work, they emit a result, they die. This is the right model for tasks. But the most valuable systems in software are not tasks — they are ongoing processes. A database does not run once and exit. An API server does not emit a result and terminate. A background job processor does not complete. These are **stewards**: agents that own a domain, maintain it over time, react to change, and coordinate with other stewards.

v2.0 is the specification for Sage as a language capable of expressing steward systems. The headline application is a full-stack web application expressed as three coordinating steward agents — `DatabaseSteward`, `APISteward`, and `FrontendSteward` — each maintaining its domain autonomously, each notifying the others when its world changes, and each backed by the intelligence of an LLM for the decisions that cannot be encoded as rules.

This is not infrastructure-as-code. It is **infrastructure-as-intent**: you declare what you want each steward to maintain, and the language runtime ensures it is maintained.

### What makes this different from frameworks

Every major framework that attempts this today (LangChain, AutoGen, CrewAI) reaches the same ceiling: the coordination logic is expressed in the host language (Python), which means the framework cannot reason about it. When an agent crashes, the framework restarts it blindly. When two agents need to communicate, the protocol is a string. When a tool call fails, the error propagates as an exception.

Sage v2.0 addresses this at the language level:

- Agent crashes are handled by typed supervision trees — the restart strategy is declared, not bolted on
- Inter-agent protocols are session-typed — the compiler verifies that agents communicate in valid sequences
- Tool calls are typed declarations — the compiler knows what tools an agent uses and can reason about capabilities
- Persistent beliefs are checkpointed — an agent that restarts recovers its exact state

The result is a steward system where the **compiler** is the primary safety mechanism, not the programmer's vigilance.

---

## 2. Prerequisites — What Must Land Before v2.0

v2.0 builds on the complete v1.x foundation. The following features must be fully implemented and stable before v2.0 work begins. Each is an explicit dependency of one or more v2.0 features.

### P-01: RFC-0006 — Agent Message Passing (✓ Implemented)
Typed `receives` clauses, `receive()`, `send()`, `loop`, and `break`. Required by: supervision trees, session types, the steward pattern.

### P-02: RFC-0007 — Error Handling (✓ Implemented)
`try infer`, `on error` handlers, `ErrorKind`, error propagation semantics. Required by: supervision trees (restart-on-error), tool declarations (fallible tools), persistent beliefs (recovery after failed checkpoint).

### P-03: RFC-0005 — User-Defined Types (✓ Implemented)
Records, enums, `match`, `const`, `Inferred<T>` for structured LLM output. Required by: generics, payload-carrying variants, tool declarations (typed return values), observability (structured event records).

### P-04: RFC-0010 — Testing Framework
`_test.sg` files, `mock infer`, assertion builtins, `@serial`. Required by: all v2.0 features need testable semantics; steward programs in particular require mockable tools and persistence.

### P-05: Payload-Carrying Enum Variants (v1.x RFC, pre-v2.0)
Variants that carry data: `Failed(String)`, `Success(T)`. Required by: session types (protocol messages carry payloads), supervision trees (restart strategies carry context), the steward pattern's inter-agent events.

### P-06: Generics / Parametric Polymorphism (v1.x RFC, pre-v2.0)
`fn map<T, U>(list: List<T>, f: Fn(T) -> U) -> List<U>`. Required by: the Commons stdlib (a useful stdlib cannot be written without generics), tool declarations (typed tool results must be generic over the return type), session types (protocol parameterisation).

### P-07: Expression-Level String Interpolation (v1.x RFC, pre-v2.0)
`"{a + b}"`, not just `"{identifier}"`. Required by: complex `infer` templates in steward programs, tool prompt construction.

### P-08: `Option<T>` and `Result<T, E>` as Standard Types (v1.x RFC, pre-v2.0)
`Option<T>` for nullable values, `Result<T, E>` for typed errors. `match` on both. Required by: tool declarations (tools that may not find a result), persistent belief recovery (checkpoint may not exist), the Commons stdlib.

---

## 3. New Language Features

### 3.1 Persistent Beliefs

#### 3.1.1 Motivation

v1.x agents die when their `on start` handler completes. All belief state is discarded. This is correct for task agents but wrong for steward agents, which must maintain state across restarts, crashes, and process exits.

Persistent beliefs are agent fields that are automatically checkpointed to durable storage. When a persistent agent restarts (due to a crash, a process restart, or a supervision-triggered restart), it recovers its persistent fields from the last checkpoint before `on start` runs.

#### 3.1.2 Syntax

```sage
agent DatabaseSteward {
    @persistent connection_string: String
    @persistent schema_version: Int
    @persistent migration_log: List<String>

    // Non-persistent — recomputed on restart
    active_connections: Int

    on start {
        // schema_version and migration_log are already populated
        // from the last checkpoint, or zero-valued on first run
        print("DatabaseSteward starting at schema v{self.schema_version}");
        emit(0);
    }
}
```

The `@persistent` annotation on a field declaration marks it for checkpointing.

Non-annotated fields are initialised to their zero value on every start (0 for Int, `""` for String, `[]` for List, etc.).

#### 3.1.3 Checkpoint semantics

Checkpointing happens:
- After every assignment to a `@persistent` field (`self.schema_version = 5`)
- After every `on message` handler completes
- Before `emit` is called (final state checkpoint)
- Explicitly via `checkpoint()` builtin

Checkpoints are atomic. A crash during a checkpoint does not corrupt the stored state — the previous checkpoint is retained.

#### 3.1.4 Storage backend

The checkpoint backend is configured in `sage.toml`:

```toml
[persistence]
backend = "sqlite"          # sqlite | postgres | file
path = ".sage/checkpoints"  # for sqlite and file backends
url = ""                    # for postgres backend
```

The default is `sqlite` with a local `.sage/checkpoints.db` file. For production steward programs, `postgres` is recommended.

Each agent instance has a unique checkpoint key derived from its name and its spawn-time beliefs. Two `DatabaseSteward` agents with different `connection_string` values have independent checkpoint namespaces.

#### 3.1.5 First-run detection

```sage
agent APISteward {
    @persistent routes_generated: Bool

    on start {
        if !self.routes_generated {
            // First run — generate everything from scratch
            let routes = generate_initial_routes();
            self.routes_generated = true;
        }
        // Subsequent runs — state is already loaded
    }
}
```

#### 3.1.6 Type checker rules

| Rule | Error Code |
|---|---|
| `@persistent` on a non-agent field (e.g. inside a record) | E050 |
| `@persistent` field type is not serialisable (function types, Agent handles) | E051 |
| Reading a `@persistent` field outside an agent body | E052 |
| `checkpoint()` called outside an agent body | E053 |

#### 3.1.7 Grammar

```ebnf
agent_field     ::= annotation* IDENT ":" type_expr
annotation      ::= "@" IDENT
```

`@persistent` is the only valid annotation on agent fields in this RFC. Future annotations (`@encrypted`, `@ttl(n)`) are reserved.

#### 3.1.8 Codegen

`@persistent` fields generate a checkpoint read at agent initialisation and a checkpoint write on every assignment. The generated Rust uses `serde` for serialisation and the configured backend via the `sage-persistence` crate (new crate introduced in v2.0).

---

### 3.2 Tool Declarations

#### 3.2.1 Motivation

Sage agents currently interact with the world through two mechanisms: `infer` (LLM calls) and `send`/`receive` (inter-agent messages). Real steward programs need to interact with databases, HTTP APIs, filesystems, shell commands, and external services. These interactions are currently impossible to express in Sage.

Tool declarations introduce typed, named capabilities that agents can use. Tools are the bridge between Sage agents and the outside world.

#### 3.2.2 Syntax

Tools are declared at the top level with the `tool` keyword:

```sage
tool Database {
    fn query(sql: String) -> List<Row>
    fn execute(sql: String) -> Int          // returns rows affected
    fn transaction(sql: List<String>) -> Bool
}

tool Http {
    fn get(url: String) -> HttpResponse
    fn post(url: String, body: String) -> HttpResponse
    fn put(url: String, body: String) -> HttpResponse
    fn delete(url: String) -> HttpResponse
}

tool FileSystem {
    fn read(path: String) -> String
    fn write(path: String, content: String) -> Unit
    fn exists(path: String) -> Bool
    fn list(path: String) -> List<String>
    fn delete(path: String) -> Unit
}

tool Shell {
    fn run(command: String) -> ShellResult
}
```

#### 3.2.3 Using tools in agents

Agents declare which tools they use with a `uses` clause:

```sage
agent DatabaseSteward uses Database {
    @persistent schema_version: Int
    @persistent migration_log: List<String>

    on start {
        let current = try Database.query("SELECT version FROM schema_meta");
        print("Schema at version {self.schema_version}");
        emit(0);
    }
}
```

The `uses` clause is a capability declaration. An agent that does not declare `uses Database` cannot call `Database.*` methods. Attempting to do so is a compile error (E054). This is the capability model: an agent's permitted interactions with the world are visible in its declaration.

Multiple tools:

```sage
agent APISteward uses Database, Http, FileSystem {
    // ...
}
```

#### 3.2.4 All tool calls are fallible

Tool calls can fail (network errors, SQL errors, filesystem permission errors). All tool call return types are implicitly `Result<T, ToolError>`. The `try` keyword unwraps the result and propagates errors to the agent's `on error` handler:

```sage
on start {
    // try propagates ToolError to on error if the query fails
    let rows = try Database.query("SELECT * FROM users");

    // explicit error handling
    let result = Database.query("SELECT * FROM users");
    match result {
        Ok(rows) => { /* use rows */ }
        Err(e) => { print("Query failed: " ++ e.message); }
    }
}
```

#### 3.2.5 Built-in tool types

The following record types are provided by the Commons stdlib for built-in tool results:

```sage
record Row {
    columns: List<String>
    values: List<String>
}

record HttpResponse {
    status: Int
    body: String
    headers: List<String>
}

record ShellResult {
    exit_code: Int
    stdout: String
    stderr: String
}
```

#### 3.2.6 Tool implementation

Tools are implemented in Rust in the `sage-tools` crate (new in v2.0). The compiler generates tool call dispatch into the appropriate Rust function. Tool implementations are not user-extensible in this RFC — the set of built-in tools is fixed. Custom tool support (via MCP or user-defined Rust implementations) is deferred to a future RFC.

#### 3.2.7 Tool configuration

Tool connection parameters are configured in `sage.toml`:

```toml
[tools.database]
driver = "postgres"
url = "postgres://localhost/myapp"
pool_size = 10

[tools.http]
timeout_ms = 5000
user_agent = "sage-agent/2.0"

[tools.filesystem]
root = "/var/sage/data"   # all paths are relative to root
```

#### 3.2.8 Type checker rules

| Rule | Error Code |
|---|---|
| Tool method called in agent without `uses` clause | E054 |
| Tool method called with wrong argument types | E003 (existing type mismatch) |
| Tool declared but not used in agent | W003 (warning) |
| Unknown tool in `uses` clause | E055 |
| `uses` clause on non-agent declaration | E056 |

#### 3.2.9 Grammar

```ebnf
tool_decl       ::= "tool" IDENT "{" tool_method* "}"
tool_method     ::= "fn" IDENT "(" param_list? ")" "->" type_expr

agent_decl      ::= "pub"? "agent" IDENT
                    ("receives" type_expr)?
                    ("uses" tool_list)?
                    "{" agent_field* handler* "}"

tool_list       ::= IDENT ("," IDENT)*

tool_call_expr  ::= IDENT "." IDENT "(" arg_list? ")"
```

---

### 3.3 Supervision Trees

#### 3.3.1 Motivation

v1.x agents fail fatally — a crashed agent's awaiter receives an error, and the error propagates. For steward programs that must run indefinitely, fatal failure is unacceptable. A `DatabaseSteward` that crashes because of a transient connection error should restart, not bring down the whole program.

Supervision trees, inspired by Erlang/OTP, provide a declarative mechanism for specifying how agents should be restarted on failure, and what should happen to dependent agents when a steward goes down.

#### 3.3.2 Supervisor agents

A supervisor agent is a special agent that manages the lifecycle of a set of child agents. It is declared with the `supervisor` keyword:

```sage
supervisor AppSupervisor {
    strategy: OneForOne

    children {
        DatabaseSteward {
            restart: Permanent
            connection_string: "postgres://localhost/myapp"
            schema_version: 0
            migration_log: []
        }

        APISteward {
            restart: Permanent
            port: 8080
            routes_generated: false
        }

        FrontendSteward {
            restart: Permanent
            build_version: ""
        }
    }
}
```

#### 3.3.3 Restart strategies

**Agent-level restart annotation:**

| Strategy | Behaviour |
|---|---|
| `Permanent` | Always restart, regardless of exit reason |
| `Transient` | Restart only if the agent exited with an error |
| `Temporary` | Never restart — let the agent die |

**Supervisor-level strategy:**

| Strategy | Behaviour |
|---|---|
| `OneForOne` | When a child fails, restart only that child |
| `OneForAll` | When a child fails, restart all children |
| `RestForOne` | When a child fails, restart it and all children declared after it |

#### 3.3.4 Restart limits

A supervisor has a restart intensity limit to prevent restart storms:

```toml
[supervisor]
max_restarts = 5
within_seconds = 60
```

If a child restarts more than `max_restarts` times within `within_seconds`, the supervisor itself terminates with an error. If the supervisor has a parent supervisor (nested supervision), that parent applies its own strategy.

#### 3.3.5 Supervised agent state on restart

When a `Permanent` or `Transient` agent restarts:
- `@persistent` fields are loaded from the last checkpoint
- Non-persistent fields are zero-valued
- `on start` runs again from the beginning

The agent effectively resumes from its last stable checkpoint. This is the key integration between persistent beliefs and supervision: together they provide crash recovery with state.

#### 3.3.6 `run` statement changes

In v2.0, `run` can name either an agent or a supervisor:

```sage
run AppSupervisor;
```

If a supervisor is named, the runtime spawns the supervisor, which then spawns its children according to the declared strategy. The program exits when the top-level supervisor terminates.

#### 3.3.7 Type checker rules

| Rule | Error Code |
|---|---|
| `supervisor` declaration with no children | E060 |
| Child agent in supervisor that does not exist | E061 |
| Child agent in supervisor missing required (non-persistent) beliefs | E062 |
| `Permanent` restart on an agent without `@persistent` fields | W004 (warning — will restart but lose all state) |
| Nested supervisor depth exceeds 8 | E063 (practical limit to prevent pathological trees) |

#### 3.3.8 Grammar

```ebnf
supervisor_decl ::= "supervisor" IDENT "{" strategy_decl children_decl "}"

strategy_decl   ::= "strategy" ":" supervisor_strategy

supervisor_strategy ::= "OneForOne" | "OneForAll" | "RestForOne"

children_decl   ::= "children" "{" child_spec* "}"

child_spec      ::= IDENT "{" restart_decl field_init* "}"

restart_decl    ::= "restart" ":" restart_strategy

restart_strategy ::= "Permanent" | "Transient" | "Temporary"
```

---

### 3.4 Session Types

#### 3.4.1 Motivation

v1.x message passing is typed at the message level: the compiler knows that `send(worker, WorkerMsg.Task)` is valid because `Worker receives WorkerMsg`. But it does not know anything about the *sequence* of messages. You can send `Shutdown` before `Task`. You can send `Task` after an agent has shut down. The protocol exists only in the programmer's head.

Session types make communication protocols explicit and statically verified. The compiler rejects programs that send messages in the wrong order, send to a terminated agent, or fail to complete a protocol.

#### 3.4.2 Protocol declarations

```sage
protocol DebateRound {
    // Coordinator sends Argument to each debater, then receives Response
    Coordinator -> Debater: Argument
    Debater -> Coordinator: Response
}

protocol SchemaSync {
    // DatabaseSteward notifies APISteward of changes
    DatabaseSteward -> APISteward: SchemaChanged
    APISteward -> DatabaseSteward: Acknowledged
}

protocol WorkerLifecycle {
    // Coordinator can send tasks and then must shut down
    Coordinator -> Worker: Task
    Coordinator -> Worker: Task    // repetition allowed
    Coordinator -> Worker: Shutdown
    Worker -> Coordinator: Done
}
```

#### 3.4.3 Using protocols

Agents declare which protocols they participate in via the `follows` clause:

```sage
agent APISteward uses Database, Http
                 follows SchemaSync as APISteward {
    // ...

    on message(msg: SchemaChanged) {
        // Compiler knows: after receiving SchemaChanged,
        // we must send Acknowledged to the sender
        let updated = regenerate_affected_routes(msg);
        reply(Acknowledged);
    }
}
```

`reply(msg)` sends a message back to the most recent sender, as required by the protocol. Failing to call `reply` when the protocol requires it is a compile error (E070).

#### 3.4.4 Protocol violations

The checker statically verifies:
- All required sends in a protocol are made
- Sends happen in declared order
- No message is sent after the protocol has terminated
- No message is sent to an agent whose protocol role has ended

```sage
agent Coordinator {
    on start {
        let w = spawn Worker {};
        send(w, WorkerMsg.Shutdown);
        send(w, WorkerMsg.Task);   // E071: send after Shutdown in WorkerLifecycle
    }
}
```

#### 3.4.5 Type checker rules

| Rule | Error Code |
|---|---|
| Protocol step violated (wrong message type or order) | E070 |
| Message sent after protocol terminated | E071 |
| Agent declares `follows` for unknown protocol | E072 |
| Agent's message type incompatible with protocol step | E073 |
| Protocol declared but no agent follows it | W005 (warning) |

#### 3.4.6 Grammar

```ebnf
protocol_decl   ::= "protocol" IDENT "{" protocol_step* "}"

protocol_step   ::= IDENT "->" IDENT ":" type_expr

agent_decl      ::= "pub"? "agent" IDENT
                    ("receives" type_expr)?
                    ("uses" tool_list)?
                    ("follows" protocol_list)?
                    "{" agent_field* handler* "}"

protocol_list   ::= protocol_role ("," protocol_role)*

protocol_role   ::= IDENT "as" IDENT
```

---

### 3.5 Algebraic Effects

#### 3.5.1 Motivation

Currently, `infer` is a single global operation. Every agent in a program uses the same LLM backend, the same model, the same API key. There is no way to give different agents different LLM configurations, to intercept and log all LLM calls in a program, or to replace the LLM backend with a mock without modifying agent code.

Algebraic effects provide a principled mechanism for this. An effect is a typed operation that an agent *requests* — it does not perform the operation itself. The runtime *handles* the effect, and the handler can be swapped without changing the agent.

#### 3.5.2 Built-in effects

In v2.0, the only built-in effect is `Infer` — the LLM call. All other effects (database, HTTP, filesystem) are handled through tool declarations (§3.2), not the effect system. The effect system exists primarily to make `infer` configurable and mockable at the program level.

#### 3.5.3 Effect handlers

```sage
// Default handler — uses the configured LLM backend
handler DefaultLLM handles Infer {
    model: "gpt-4o"
    temperature: 0.7
    max_tokens: 1024
}

// A faster, cheaper handler for less critical agents
handler FastLLM handles Infer {
    model: "gpt-4o-mini"
    temperature: 0.3
    max_tokens: 256
}

// A handler for testing — returns a fixed response
handler MockLLM handles Infer {
    response: "Mocked LLM response."
}
```

#### 3.5.4 Assigning handlers to agents

Effect handlers are assigned in the supervisor or at the `run` site:

```sage
supervisor AppSupervisor {
    strategy: OneForOne

    children {
        DatabaseSteward {
            restart: Permanent
            handler Infer: DefaultLLM
            // beliefs...
        }

        MonitorAgent {
            restart: Permanent
            handler Infer: FastLLM   // cheaper model for monitoring
            // beliefs...
        }
    }
}
```

#### 3.5.5 Why this matters for steward programs

Different stewards have different LLM requirements. A `DatabaseSteward` deciding whether to apply a migration might need GPT-4o quality reasoning. A `MonitorAgent` checking if a metric is anomalous can use a cheaper, faster model. An `AlertAgent` composing a notification message needs a different temperature than a `PlannerAgent` designing a schema.

Without algebraic effects, all of these use the same model. With effects, each agent's LLM configuration is a declaration, not a runtime concern.

#### 3.5.6 User-defined effects

User-defined effects are out of scope for v2.0. The effect system is introduced specifically for `Infer` configurability. Generalisation to arbitrary user-defined effects is deferred to v3.0.

#### 3.5.7 Grammar

```ebnf
handler_decl    ::= "handler" IDENT "handles" IDENT "{" handler_field* "}"

handler_field   ::= IDENT ":" literal

child_spec      ::= IDENT "{" restart_decl handler_assign* field_init* "}"

handler_assign  ::= "handler" IDENT ":" IDENT
```

---

### 3.6 Generics

#### 3.6.1 Motivation

A useful stdlib requires generics. `List<T>` is already parameterised, but functions over lists, `Option<T>`, `Result<T, E>`, and typed channel primitives all require parametric polymorphism.

#### 3.6.2 Syntax

Generic functions:

```sage
fn map<T, U>(list: List<T>, f: Fn(T) -> U) -> List<U> {
    let result: List<U> = [];
    for item in list {
        result = push(result, f(item));
    }
    return result;
}

fn filter<T>(list: List<T>, pred: Fn(T) -> Bool) -> List<T> {
    let result: List<T> = [];
    for item in list {
        if pred(item) {
            result = push(result, item);
        }
    }
    return result;
}

fn find<T>(list: List<T>, pred: Fn(T) -> Bool) -> Option<T> {
    for item in list {
        if pred(item) {
            return Some(item);
        }
    }
    return None;
}
```

Generic records:

```sage
record Page<T> {
    items: List<T>
    total: Int
    page: Int
    page_size: Int
}

record Timestamped<T> {
    value: T
    created_at: String
    updated_at: String
}
```

#### 3.6.3 Constraints

Sage v2.0 generics are monomorphised at compile time (Rust-style). No runtime type information is required. Type parameters are unconstrained in v2.0 — trait-style constraints are deferred to v3.0. The type checker enforces consistent usage within a generic definition.

#### 3.6.4 Grammar

```ebnf
fn_decl         ::= "fn" IDENT type_params? "(" param_list? ")" "->" type_expr block

type_params     ::= "<" IDENT ("," IDENT)* ">"

record_decl     ::= "record" IDENT type_params? "{" record_field* "}"

type_expr       ::= ... | IDENT "<" type_expr ("," type_expr)* ">"
```

---

### 3.7 Payload-Carrying Enum Variants

#### 3.7.1 Syntax

```sage
enum AgentEvent {
    SchemaChanged(SchemaChange)
    RouteAdded(RouteSpec)
    BuildCompleted(BuildResult)
    Error(String)
    Shutdown
}

enum SchemaChange {
    AddColumn(ColumnSpec)
    DropColumn(String)
    AddTable(TableSpec)
    DropTable(String)
    AddIndex(IndexSpec)
}
```

#### 3.7.2 Match with payload binding

```sage
match event {
    AgentEvent.SchemaChanged(change) => {
        apply_schema_change(change);
    }
    AgentEvent.Error(msg) => {
        print("Error: " ++ msg);
    }
    AgentEvent.Shutdown => {
        break;
    }
    _ => {}
}
```

#### 3.7.3 Type checker rules

| Rule | Error Code |
|---|---|
| Pattern binding type mismatch | E003 |
| Variant constructed with wrong payload type | E003 |
| Variant constructed without required payload | E080 |
| Payload bound in match arm but variant has no payload | E081 |

---

### 3.8 Agent Lifecycle Hooks

#### 3.8.1 Full lifecycle

v2.0 introduces the complete agent lifecycle with all hooks:

```sage
agent DatabaseSteward uses Database {
    @persistent schema_version: Int

    on waking {
        // Runs before on start, after persistent state is loaded.
        // Use for: validating recovered state, opening connections,
        // registering with a service registry.
        print("Recovered schema at version {self.schema_version}");
    }

    on start {
        // Main agent logic — runs every time the agent starts/restarts.
    }

    on pause {
        // Runs when the supervisor signals a graceful pause.
        // Finish in-flight work, flush buffers, release locks.
    }

    on resume {
        // Runs when the agent is unpaused by the supervisor.
    }

    on resting {
        // Runs after emit, before process exit.
        // Use for: closing connections, flushing logs,
        // deregistering from service registry.
        print("DatabaseSteward resting.");
    }
}
```

#### 3.8.2 Lifecycle sequence

```
Process start / Supervisor restart
         │
         ▼
  ┌─────────────┐
  │  on waking  │  — persistent state loaded, connections opened
  └──────┬──────┘
         │
         ▼
  ┌─────────────┐
  │  on start   │  — main logic
  └──────┬──────┘
         │
    ┌────┴─────┐
    │          │
    ▼          ▼
┌────────┐  ┌───────────┐
│on pause│  │on message │  — concurrent with on start
└────┬───┘  └───────────┘
     │
     ▼
┌──────────┐
│on resume │
└────┬─────┘
     │
     ▼
  ┌──────┐
  │ emit │  — agent has produced its result
  └──┬───┘
     │
     ▼
┌────────────┐
│ on resting │  — cleanup
└────────────┘
```

#### 3.8.3 Type checker rules

| Rule | Error Code |
|---|---|
| `on waking` used in agent without `@persistent` fields | W006 (warning) |
| `emit` called inside `on waking`, `on pause`, `on resume`, or `on resting` | E090 |
| `receive()` called inside `on waking` or `on resting` | E091 |

---

### 3.9 Observability Primitives

#### 3.9.1 Motivation

Production steward programs need visibility. When an agent crashes, you need to know what it was doing. When an `infer` call takes 8 seconds, you need to know which agent called it. When a migration fails, you need the full context.

v2.0 introduces structured observability as a first-class language feature.

#### 3.9.2 `trace` statement

```sage
on start {
    trace "applying migration {migration.name}";
    let result = try Database.execute(migration.sql);
    trace "migration applied, {result} rows affected";
}
```

`trace` emits a structured event to the configured observability backend with the current agent name, handler, timestamp, and formatted message.

#### 3.9.3 `span` blocks

```sage
on start {
    span "schema reconciliation" {
        let current = try Database.query("SELECT version FROM schema_meta");
        let target = determine_target_version();
        apply_migrations(current, target);
    }
    // span ends here — duration is recorded automatically
}
```

`span` groups a block of work under a named span for distributed tracing. Nested spans create a trace tree.

#### 3.9.4 Automatic events

The runtime emits automatic observability events for:
- Agent spawn, restart, emit, crash
- `infer` call start, end, token count, latency
- Tool call start, end, success/failure
- Supervisor restart decision
- Checkpoint write (for persistent agents)

#### 3.9.5 Observability configuration

```toml
[observability]
backend = "stdout"      # stdout | otlp | file
level = "info"          # debug | info | warn | error
otlp_endpoint = ""      # for otlp backend
```

`otlp` (OpenTelemetry Protocol) enables integration with Grafana, Jaeger, Honeycomb, and any OpenTelemetry-compatible backend.

---

## 4. Type System Changes

### 4.1 New built-in types

| Type | Description |
|---|---|
| `Option<T>` | A value that may or may not exist. Variants: `Some(T)`, `None` |
| `Result<T, E>` | A value that may be an error. Variants: `Ok(T)`, `Err(E)` |
| `ToolError` | Error from a tool call. Fields: `message: String`, `code: Int`, `tool: String` |
| `AgentHandle<T>` | Handle to a running agent that will emit `T`. Alias for v1.x `Agent<T>` |
| `Channel<T>` | Typed communication channel (introduced with session types) |

### 4.2 Serialisable type constraint

`@persistent` fields and `Inferred<T>` require their types to be *serialisable*. A type is serialisable if it is:
- A primitive (`Int`, `Float`, `Bool`, `String`)
- A `List<T>` where `T` is serialisable
- An `Option<T>` where `T` is serialisable
- A record where all fields are serialisable
- An enum (with or without payloads, where all payload types are serialisable)

Function types, `AgentHandle<T>`, and `Channel<T>` are not serialisable.

### 4.3 Capability types

The `uses` clause introduces a capability type system. The checker tracks which tools each agent has access to and rejects tool calls that exceed the declared capability set. This is a simple form of capability-based security — you cannot accidentally use a tool you did not declare.

### 4.4 Effect types

Every `infer` call carries an implicit effect type: `Infer`. The checker tracks effect usage through function calls — if a function calls `infer`, its effective type includes the `Infer` effect. In v2.0 this is used only to ensure that `infer` calls inside agents are covered by an effect handler assignment. In v3.0 the effect system will be generalised.

---

## 5. Runtime Semantics

### 5.1 Steward process model

A v2.0 Sage program with a supervisor entry point runs as follows:

1. The runtime initialises the persistence backend (connects to SQLite/Postgres, opens checkpoint file)
2. The runtime initialises the observability backend
3. The top-level supervisor is spawned
4. The supervisor spawns its children according to the declared order
5. Each child runs `on waking` (loading persistent state), then `on start`
6. The supervisor monitors children. On child exit:
   - If exit is clean (`emit` called): apply restart strategy (Permanent restarts, Transient does not)
   - If exit is error: apply restart strategy, check restart intensity
7. The program exits when the top-level supervisor terminates

### 5.2 Checkpoint protocol

Checkpointing uses a write-ahead log (WAL) pattern:
1. Serialise the new field value
2. Write to WAL entry: `{agent_key, field_name, serialised_value, timestamp}`
3. On confirmed WAL write: update the checkpoint record
4. On startup: read checkpoint record, verify WAL for any uncommitted entries, apply or discard

This ensures that a crash during a checkpoint write never leaves state in an inconsistent condition.

### 5.3 Tool call execution

Tool calls are dispatched to the `sage-tools` crate via a trait object. The calling agent's Tokio task suspends during the tool call. Other agents continue to run concurrently. Timeouts are applied per tool call (configured in `sage.toml`). On timeout, the tool call returns `Err(ToolError { message: "timeout", ... })`.

### 5.4 Concurrent agent safety

All persistent belief writes are serialised per agent — no two writes to the same agent's checkpoint can interleave. Concurrent agents with different checkpoint namespaces write independently. Tool calls from concurrent agents are dispatched concurrently (the tool implementations are responsible for their own thread safety, e.g. using a connection pool for database tools).

---

## 6. Standard Library — the Commons

The Commons is the Sage standard library. v2.0 ships the first meaningful version.

### 6.1 List operations

```sage
fn map<T, U>(list: List<T>, f: Fn(T) -> U) -> List<U>
fn filter<T>(list: List<T>, pred: Fn(T) -> Bool) -> List<T>
fn find<T>(list: List<T>, pred: Fn(T) -> Bool) -> Option<T>
fn reduce<T, U>(list: List<T>, init: U, f: Fn(U, T) -> U) -> U
fn flatten<T>(list: List<List<T>>) -> List<T>
fn zip<T, U>(a: List<T>, b: List<U>) -> List<Pair<T, U>>
fn sort_by<T>(list: List<T>, key: Fn(T) -> Int) -> List<T>
fn unique<T>(list: List<T>) -> List<T>
fn take<T>(list: List<T>, n: Int) -> List<T>
fn drop<T>(list: List<T>, n: Int) -> List<T>
fn any<T>(list: List<T>, pred: Fn(T) -> Bool) -> Bool
fn all<T>(list: List<T>, pred: Fn(T) -> Bool) -> Bool
fn count<T>(list: List<T>, pred: Fn(T) -> Bool) -> Int
```

### 6.2 String operations

```sage
fn split(s: String, delimiter: String) -> List<String>
fn trim(s: String) -> String
fn upper(s: String) -> String
fn lower(s: String) -> String
fn replace(s: String, from: String, to: String) -> String
fn parse_int(s: String) -> Option<Int>
fn parse_float(s: String) -> Option<Float>
fn repeat(s: String, n: Int) -> String
fn pad_left(s: String, width: Int, char: String) -> String
fn pad_right(s: String, width: Int, char: String) -> String
```

### 6.3 Option operations

```sage
fn unwrap_or<T>(opt: Option<T>, default: T) -> T
fn map_option<T, U>(opt: Option<T>, f: Fn(T) -> U) -> Option<U>
fn is_some<T>(opt: Option<T>) -> Bool
fn is_none<T>(opt: Option<T>) -> Bool
```

### 6.4 Result operations

```sage
fn unwrap_or_else<T, E>(result: Result<T, E>, f: Fn(E) -> T) -> T
fn map_result<T, U, E>(result: Result<T, E>, f: Fn(T) -> U) -> Result<U, E>
fn is_ok<T, E>(result: Result<T, E>) -> Bool
fn is_err<T, E>(result: Result<T, E>) -> Bool
```

### 6.5 Time operations

```sage
fn now_unix() -> Int                    // Unix timestamp in seconds
fn now_iso() -> String                  // ISO 8601 string
fn sleep_ms(ms: Int) -> Unit
fn format_duration(ms: Int) -> String   // "1h 23m 4s"
```

### 6.6 JSON operations

```sage
fn to_json<T>(value: T) -> String       // T must be serialisable
fn from_json<T>(s: String) -> Result<T, String>
```

---

## 7. Grammar Changes

The complete v2.0 grammar adds the following top-level productions to the v1.x grammar:

```ebnf
program         ::= top_level*

top_level       ::= agent_decl
                  | supervisor_decl
                  | fn_decl
                  | record_decl
                  | enum_decl
                  | tool_decl
                  | protocol_decl
                  | handler_decl
                  | const_decl
                  | run_stmt

(* Agent declaration — updated *)
agent_decl      ::= "pub"? "agent" IDENT type_params?
                    ("receives" type_expr)?
                    ("uses" tool_list)?
                    ("follows" protocol_list)?
                    "{" agent_field* handler_decl* "}"

agent_field     ::= annotation* IDENT ":" type_expr

annotation      ::= "@" IDENT

handler_decl    ::= "on" event_kind block

event_kind      ::= "waking"
                  | "start"
                  | "pause"
                  | "resume"
                  | "resting"
                  | "message" "(" IDENT ":" type_expr ")"
                  | "error" "(" IDENT ")"

(* Supervisor declaration *)
supervisor_decl ::= "supervisor" IDENT "{" strategy_clause children_clause "}"

strategy_clause ::= "strategy" ":" supervisor_strategy

supervisor_strategy ::= "OneForOne" | "OneForAll" | "RestForOne"

children_clause ::= "children" "{" child_spec* "}"

child_spec      ::= IDENT "{" restart_clause handler_assign* field_init* "}"

restart_clause  ::= "restart" ":" restart_strategy

restart_strategy ::= "Permanent" | "Transient" | "Temporary"

handler_assign  ::= "handler" IDENT ":" IDENT

(* Tool declaration *)
tool_decl       ::= "tool" IDENT "{" tool_method* "}"

tool_method     ::= "fn" IDENT "(" param_list? ")" "->" type_expr

(* Protocol declaration *)
protocol_decl   ::= "protocol" IDENT "{" protocol_step* "}"

protocol_step   ::= IDENT "->" IDENT ":" type_expr

(* Effect handler declaration *)
handler_decl    ::= "handler" IDENT "handles" IDENT "{" handler_field* "}"

handler_field   ::= IDENT ":" literal

(* Statements — new *)
stmt            ::= ...
                  | trace_stmt
                  | span_stmt
                  | checkpoint_stmt

trace_stmt      ::= "trace" STRING_LIT ";"

span_stmt       ::= "span" STRING_LIT block

checkpoint_stmt ::= "checkpoint" "(" ")" ";"

(* Expressions — new *)
expr            ::= ...
                  | tool_call_expr
                  | reply_expr

tool_call_expr  ::= IDENT "." IDENT "(" arg_list? ")"

reply_expr      ::= "reply" "(" expr ")"

(* Types — new *)
type_expr       ::= ...
                  | "Option" "<" type_expr ">"
                  | "Result" "<" type_expr "," type_expr ">"
                  | "Channel" "<" type_expr ">"
                  | "ToolError"
```

---

## 8. Compiler Architecture Changes

### 8.1 New crates

| Crate | Responsibility |
|---|---|
| `sage-persistence` | Checkpoint read/write, WAL, backend adapters (SQLite, Postgres, file) |
| `sage-tools` | Built-in tool implementations (Database, Http, FileSystem, Shell) |
| `sage-observe` | Observability event emission, span tracking, OTLP export |
| `sage-supervisor` | Supervisor runtime: child lifecycle management, restart logic, intensity tracking |
| `sage-session` | Session type runtime enforcement (complements compile-time checking) |

### 8.2 Updated compiler pipeline

```
source.sg
    │
    ▼
┌─────────┐
│  Lexer  │  Extended with new keywords: persistent, tool, supervisor,
└────┬────┘  protocol, handler, uses, follows, waking, resting,
     │       pause, resume, trace, span, checkpoint, reply
     ▼
┌─────────┐
│ Parser  │  Extended with new top-level and statement productions
└────┬────┘
     │
     ▼
┌──────────────┐
│ Name Resolver│  Resolves tool names, protocol names, handler names
└──────┬───────┘
       │
       ▼
┌──────────────────┐
│ Capability Checker│  NEW: verifies uses clauses, tool call permissions
└──────┬───────────┘
       │
       ▼
┌──────────────┐
│ Type Checker │  Extended: generics, Option/Result, payload variants,
└──────┬───────┘  serialisability, effect types
       │
       ▼
┌──────────────────┐
│ Session Checker  │  NEW: protocol step verification, send/receive ordering
└──────┬───────────┘
       │
       ▼
┌────────────────────┐
│ Persistence Checker│  NEW: @persistent field serialisability,
└──────┬─────────────┘  checkpoint() call validity
       │
       ▼
┌─────────────┐
│   Codegen   │  Extended: supervisor scaffolding, tool dispatch,
└──────┬──────┘  checkpoint calls, observability instrumentation
       │
       ▼
   .rs files → rustc → native binary
```

### 8.3 Codegen changes

The generated Rust for a v2.0 steward program includes:
- A supervisor task (`sage-supervisor`) wrapping the top-level supervisor
- Per-agent checkpoint initialisation from `sage-persistence`
- Tool dispatch traits resolved via `sage-tools`
- Observability instrumentation injected at every `trace`, `span`, `infer`, and tool call site
- Session type runtime state machine (one per `follows` declaration)

---

## 9. New Error Codes

### Persistent beliefs (E050–E059)

| Code | Message |
|---|---|
| E050 | `@persistent` annotation only valid on agent fields |
| E051 | `@persistent` field type is not serialisable |
| E052 | `@persistent` field accessed outside agent body |
| E053 | `checkpoint()` called outside agent body |

### Tool declarations (E054–E059)

| Code | Message |
|---|---|
| E054 | tool method called without `uses` declaration |
| E055 | unknown tool in `uses` clause |
| E056 | `uses` clause on non-agent declaration |
| E057 | tool method does not exist on declared tool |
| E058 | tool call argument type mismatch |

### Supervision trees (E060–E069)

| Code | Message |
|---|---|
| E060 | supervisor has no children |
| E061 | child agent in supervisor does not exist |
| E062 | child agent missing required beliefs in supervisor spec |
| E063 | supervisor nesting depth exceeds 8 |

### Session types (E070–E079)

| Code | Message |
|---|---|
| E070 | protocol step violated — missing required reply |
| E071 | message sent after protocol terminated |
| E072 | unknown protocol in `follows` clause |
| E073 | agent message type incompatible with protocol step |

### Payload variants (E080–E089)

| Code | Message |
|---|---|
| E080 | variant constructed without required payload |
| E081 | payload bound in match arm but variant has no payload |

### Lifecycle hooks (E090–E099)

| Code | Message |
|---|---|
| E090 | `emit` called in non-emittable lifecycle hook |
| E091 | `receive()` called in `on waking` or `on resting` |

### Warnings

| Code | Message |
|---|---|
| W003 | tool declared in `uses` but never called |
| W004 | `Permanent` restart on agent with no `@persistent` fields |
| W005 | protocol declared but no agent follows it |
| W006 | `on waking` in agent with no `@persistent` fields |

---

## 10. The Steward Pattern

### 10.1 Definition

A **steward** is a `Permanent`-restart agent with:
- One or more `@persistent` fields representing its domain state
- Tool declarations matching its domain (`DatabaseSteward uses Database`, etc.)
- A `receives` clause for an event enum that notifies it of changes from other stewards
- Protocol declarations for the coordination contracts with neighbouring stewards
- An `on waking` hook for connection initialisation
- An `on resting` hook for connection cleanup

### 10.2 The three-steward web app architecture

```
┌────────────────────────────────────────────────┐
│                  AppSupervisor                  │
│         strategy: RestForOne                    │
│                                                 │
│  ┌─────────────────────────────────────────┐   │
│  │ DatabaseSteward  restart: Permanent      │   │
│  │   @persistent schema_version: Int        │   │
│  │   @persistent migration_log: List<String>│   │
│  │   uses: Database                         │   │
│  │   follows: SchemaSync as DatabaseSteward │   │
│  └───────────────────┬─────────────────────┘   │
│                      │ SchemaChanged             │
│                      ▼                           │
│  ┌─────────────────────────────────────────┐   │
│  │ APISteward       restart: Permanent      │   │
│  │   @persistent routes_generated: Bool     │   │
│  │   @persistent route_version: Int         │   │
│  │   uses: Database, Http, FileSystem       │   │
│  │   follows: SchemaSync as APISteward      │   │
│  │   follows: ApiSync as APISteward         │   │
│  └───────────────────┬─────────────────────┘   │
│                      │ RouteChanged              │
│                      ▼                           │
│  ┌─────────────────────────────────────────┐   │
│  │ FrontendSteward  restart: Permanent      │   │
│  │   @persistent build_version: String      │   │
│  │   @persistent component_map: List<String>│   │
│  │   uses: FileSystem, Shell                │   │
│  │   follows: ApiSync as FrontendSteward    │   │
│  └─────────────────────────────────────────┘   │
└────────────────────────────────────────────────┘
```

`RestForOne` is the right supervisor strategy here: if the `DatabaseSteward` crashes, the `APISteward` and `FrontendSteward` should also restart (they may be serving stale schema data). If only the `APISteward` crashes, the `DatabaseSteward` continues — the schema is fine.

### 10.3 Change propagation

The key pattern in the steward architecture is **declarative change propagation**:

1. `DatabaseSteward` detects a schema change (via `infer` reasoning about the current vs desired schema, or via a direct instruction from an external operator)
2. `DatabaseSteward` applies the migration via `Database.execute`
3. `DatabaseSteward` increments `self.schema_version` (checkpointed immediately)
4. `DatabaseSteward` sends `SchemaChanged(change)` to `APISteward`
5. `APISteward` receives `SchemaChanged`, regenerates affected routes via `infer`
6. `APISteward` increments `self.route_version` (checkpointed)
7. `APISteward` sends `RouteChanged(change)` to `FrontendSteward`
8. `FrontendSteward` receives `RouteChanged`, regenerates affected UI components
9. `FrontendSteward` runs `Shell.run("npm run build")`
10. `FrontendSteward` updates `self.build_version` (checkpointed)

At any point in this chain, a crash is safe: the checkpointed state tells the restarted agent exactly where it left off, and the supervision tree ensures the downstream stewards are also restarted to re-receive the change.

---

## 11. Reference Programs

### 11.1 Full-Stack Web App Steward

```sage
// examples/webapp_steward.sg
// A three-steward web application.
// Run: sage run examples/webapp_steward.sg

// ─── Types ──────────────────────────────────────────────────────────────────

record ColumnSpec {
    name: String
    data_type: String
    nullable: Bool
}

record TableSpec {
    name: String
    columns: List<ColumnSpec>
}

record RouteSpec {
    method: String
    path: String
    handler_description: String
}

record BuildResult {
    success: Bool
    output: String
    duration_ms: Int
}

enum SchemaChange {
    AddTable(TableSpec)
    DropTable(String)
    AddColumn(ColumnSpec)
    DropColumn(String)
}

enum ApiChange {
    RouteAdded(RouteSpec)
    RouteRemoved(String)
    RouteModified(RouteSpec)
}

// ─── Protocols ───────────────────────────────────────────────────────────────

protocol SchemaSync {
    DatabaseSteward -> APISteward: SchemaChange
    APISteward -> DatabaseSteward: Bool
}

protocol ApiSync {
    APISteward -> FrontendSteward: ApiChange
    FrontendSteward -> APISteward: Bool
}

// ─── Tools ───────────────────────────────────────────────────────────────────

tool Database {
    fn query(sql: String) -> List<Row>
    fn execute(sql: String) -> Int
    fn transaction(statements: List<String>) -> Bool
}

tool FileSystem {
    fn read(path: String) -> String
    fn write(path: String, content: String) -> Unit
    fn exists(path: String) -> Bool
}

tool Shell {
    fn run(command: String) -> ShellResult
}

// ─── DatabaseSteward ─────────────────────────────────────────────────────────

agent DatabaseSteward
    uses Database
    follows SchemaSync as DatabaseSteward
    receives SchemaChange {

    @persistent schema_version: Int
    @persistent migration_log: List<String>

    on waking {
        trace "DatabaseSteward waking at schema version {self.schema_version}";
    }

    on start {
        trace "DatabaseSteward reconciling schema";

        span "schema reconciliation" {
            let desired: Inferred<List<TableSpec>> = try infer(
                "Given our application is a task management tool, describe the ideal " ++
                "database schema as a list of tables with columns. Current version: {self.schema_version}."
            );

            let current_tables = try Database.query(
                "SELECT table_name FROM information_schema.tables WHERE table_schema = 'public'"
            );

            let migration: Inferred<List<String>> = try infer(
                "Write the SQL migration statements needed to go from the current " ++
                "schema to the desired schema. Current tables: {current_tables}. " ++
                "Desired schema: {desired}. Return only executable SQL statements."
            );

            let success = try Database.transaction(migration);

            if success {
                self.schema_version = self.schema_version + 1;
                self.migration_log = push(self.migration_log,
                    "v{self.schema_version}: applied at {now_iso()}"
                );
                trace "Schema migrated to version {self.schema_version}";
            }
        }

        // Now run indefinitely, handling schema change requests
        loop {
            let change: SchemaChange = receive();

            span "applying schema change" {
                match change {
                    SchemaChange.AddTable(spec) => {
                        let sql: Inferred<String> = try infer(
                            "Write a CREATE TABLE SQL statement for: {spec}"
                        );
                        try Database.execute(sql);
                        self.schema_version = self.schema_version + 1;
                        trace "Table added: {spec.name}";
                    }
                    SchemaChange.DropTable(name) => {
                        try Database.execute("DROP TABLE IF EXISTS {name}");
                        self.schema_version = self.schema_version + 1;
                        trace "Table dropped: {name}";
                    }
                    _ => {
                        trace "Unhandled schema change type";
                    }
                }
            }

            reply(true);
        }

        emit(0);
    }

    on error(e) {
        trace "DatabaseSteward error: {e.message}";
        emit(1);
    }

    on resting {
        trace "DatabaseSteward resting. Final schema version: {self.schema_version}";
    }
}

// ─── APISteward ──────────────────────────────────────────────────────────────

agent APISteward
    uses Database, FileSystem
    follows SchemaSync as APISteward
    follows ApiSync as APISteward
    receives SchemaChange {

    @persistent routes_generated: Bool
    @persistent route_version: Int

    on waking {
        trace "APISteward waking. Routes generated: {self.routes_generated}";
    }

    on start {
        if !self.routes_generated {
            span "initial route generation" {
                let schema = try Database.query(
                    "SELECT table_name, column_name, data_type " ++
                    "FROM information_schema.columns WHERE table_schema = 'public'"
                );

                let routes: Inferred<List<RouteSpec>> = try infer(
                    "Given this database schema: {schema}, design a RESTful API " ++
                    "with standard CRUD routes for each table. Return route specifications."
                );

                let api_code: Inferred<String> = try infer(
                    "Write a complete Express.js API implementation for these routes: {routes}. " ++
                    "Use async/await, include error handling, and connect to PostgreSQL."
                );

                try FileSystem.write("src/api/routes.js", api_code);
                self.routes_generated = true;
                self.route_version = 1;
                trace "Initial routes generated: {len(routes)} routes";
            }
        }

        loop {
            let change: SchemaChange = receive();

            span "regenerating routes for schema change" {
                let affected: Inferred<List<RouteSpec>> = try infer(
                    "Given this schema change: {change}, which API routes need to " ++
                    "be added, removed, or modified? Return the affected route specifications."
                );

                let current_api = try FileSystem.read("src/api/routes.js");

                let updated_api: Inferred<String> = try infer(
                    "Update this API code: {current_api} to reflect these route changes: {affected}. " ++
                    "Return the complete updated file."
                );

                try FileSystem.write("src/api/routes.js", updated_api);
                self.route_version = self.route_version + 1;
                trace "Routes updated to version {self.route_version}";
            }

            reply(true);
        }

        emit(0);
    }

    on error(e) {
        trace "APISteward error: {e.message}";
        emit(1);
    }

    on resting {
        trace "APISteward resting. Final route version: {self.route_version}";
    }
}

// ─── FrontendSteward ─────────────────────────────────────────────────────────

agent FrontendSteward
    uses FileSystem, Shell
    follows ApiSync as FrontendSteward
    receives ApiChange {

    @persistent build_version: String
    @persistent component_map: List<String>

    on waking {
        trace "FrontendSteward waking. Build: {self.build_version}";
    }

    on start {
        if self.build_version == "" {
            span "initial frontend generation" {
                let api_spec = try FileSystem.read("src/api/routes.js");

                let components: Inferred<List<String>> = try infer(
                    "Given this API: {api_spec}, list the React components needed " ++
                    "for a complete task management UI. Return component names only."
                );

                for component in components {
                    let code: Inferred<String> = try infer(
                        "Write a complete React component named {component} that " ++
                        "integrates with the task management API. Use hooks, handle " ++
                        "loading and error states, and use Tailwind CSS for styling."
                    );

                    try FileSystem.write("src/components/{component}.jsx", code);
                    self.component_map = push(self.component_map, component);
                }

                let build = try Shell.run("npm run build");
                if build.exit_code == 0 {
                    self.build_version = now_iso();
                    trace "Initial build complete: {self.build_version}";
                }
            }
        }

        loop {
            let change: ApiChange = receive();

            span "regenerating frontend for api change" {
                match change {
                    ApiChange.RouteAdded(spec) => {
                        let component: Inferred<String> = try infer(
                            "Write a React component that uses the new API route: {spec}. " ++
                            "Use hooks and Tailwind CSS."
                        );
                        let name: Inferred<String> = try infer(
                            "What is a good React component name for a component " ++
                            "that handles the route: {spec.path}? Return just the name."
                        );
                        try FileSystem.write("src/components/{name}.jsx", component);
                        self.component_map = push(self.component_map, name);
                    }
                    ApiChange.RouteModified(spec) => {
                        let affected: Inferred<List<String>> = try infer(
                            "Given the component list {self.component_map} and the modified " ++
                            "route {spec}, which components need to be updated?"
                        );
                        for comp in affected {
                            let current = try FileSystem.read("src/components/{comp}.jsx");
                            let updated: Inferred<String> = try infer(
                                "Update this component {current} to use the modified route {spec}."
                            );
                            try FileSystem.write("src/components/{comp}.jsx", updated);
                        }
                    }
                    _ => {}
                }

                let build = try Shell.run("npm run build");
                if build.exit_code == 0 {
                    self.build_version = now_iso();
                    trace "Rebuild complete: {self.build_version}";
                }

                reply(true);
            }
        }

        emit(0);
    }

    on error(e) {
        trace "FrontendSteward error: {e.message}";
        emit(1);
    }

    on resting {
        trace "FrontendSteward resting. Final build: {self.build_version}";
    }
}

// ─── Supervisor ───────────────────────────────────────────────────────────────

supervisor AppSupervisor {
    strategy: RestForOne

    children {
        DatabaseSteward {
            restart: Permanent
            handler Infer: DefaultLLM
            schema_version: 0
            migration_log: []
        }

        APISteward {
            restart: Permanent
            handler Infer: DefaultLLM
            routes_generated: false
            route_version: 0
        }

        FrontendSteward {
            restart: Permanent
            handler Infer: DefaultLLM
            build_version: ""
            component_map: []
        }
    }
}

run AppSupervisor;
```

---

### 11.2 Background Service Daemon

A daemon that monitors a PostgreSQL database for slow queries, analyses them with an LLM, suggests and optionally applies indexes, and sends a daily digest.

```sage
// examples/db_guardian.sg
// A background daemon that monitors and optimises a PostgreSQL database.
// Run: sage run examples/db_guardian.sg

// ─── Types ───────────────────────────────────────────────────────────────────

record SlowQuery {
    query: String
    avg_ms: Int
    calls: Int
    table_names: List<String>
}

record IndexSuggestion {
    table: String
    columns: List<String>
    reasoning: String
    estimated_improvement: String
    sql: String
}

record DigestEntry {
    timestamp: String
    slow_queries_found: Int
    indexes_suggested: Int
    indexes_applied: Int
    summary: String
}

// ─── Tools ───────────────────────────────────────────────────────────────────

tool Database {
    fn query(sql: String) -> List<Row>
    fn execute(sql: String) -> Int
}

tool Http {
    fn post(url: String, body: String) -> HttpResponse
}

// ─── Agents ──────────────────────────────────────────────────────────────────

agent QueryMonitor uses Database {
    @persistent last_check: String
    @persistent slow_query_log: List<String>

    check_interval_ms: Int

    on waking {
        trace "QueryMonitor waking. Last check: {self.last_check}";
    }

    on start {
        loop {
            span "slow query check" {
                let slow_queries = try Database.query(
                    "SELECT query, mean_exec_time, calls " ++
                    "FROM pg_stat_statements " ++
                    "WHERE mean_exec_time > 500 " ++
                    "ORDER BY mean_exec_time DESC LIMIT 20"
                );

                if len(slow_queries) > 0 {
                    trace "Found {len(slow_queries)} slow queries";
                    self.last_check = now_iso();
                    checkpoint();
                }
            }

            sleep_ms(self.check_interval_ms);
        }

        emit(0);
    }

    on error(e) {
        trace "QueryMonitor error: {e.message}";
        sleep_ms(5000);
        emit(1);
    }
}

agent IndexAdvisor uses Database {
    @persistent suggestions_log: List<String>
    @persistent applied_count: Int

    auto_apply: Bool

    on waking {
        trace "IndexAdvisor waking. Applied {self.applied_count} indexes so far";
    }

    on start {
        loop {
            sleep_ms(300000);  // Check every 5 minutes

            span "index analysis" {
                let slow_queries = try Database.query(
                    "SELECT query, mean_exec_time, calls " ++
                    "FROM pg_stat_statements " ++
                    "WHERE mean_exec_time > 500 " ++
                    "ORDER BY mean_exec_time DESC LIMIT 10"
                );

                if len(slow_queries) > 0 {
                    let suggestions: Inferred<List<IndexSuggestion>> = try infer(
                        "Analyse these slow PostgreSQL queries and suggest indexes " ++
                        "that would improve performance. For each suggestion, provide " ++
                        "the table name, column names, reasoning, estimated improvement, " ++
                        "and the exact CREATE INDEX SQL statement.\n\nSlow queries:\n{slow_queries}"
                    );

                    for suggestion in suggestions {
                        trace "Index suggestion: {suggestion.table} ({suggestion.estimated_improvement})";

                        let entry = "{now_iso()}: {suggestion.table} — {suggestion.reasoning}";
                        self.suggestions_log = push(self.suggestions_log, entry);

                        if self.auto_apply {
                            span "applying index" {
                                let result = try Database.execute(suggestion.sql);
                                self.applied_count = self.applied_count + 1;
                                trace "Index applied on {suggestion.table}";
                            }
                        }
                    }
                }
            }
        }

        emit(0);
    }

    on error(e) {
        trace "IndexAdvisor error: {e.message}";
        emit(1);
    }
}

agent DigestSender uses Database, Http {
    @persistent digest_log: List<DigestEntry>
    @persistent last_digest_sent: String

    webhook_url: String

    on waking {
        trace "DigestSender waking. Last digest: {self.last_digest_sent}";
    }

    on start {
        loop {
            sleep_ms(86400000);  // Every 24 hours

            span "sending daily digest" {
                let stats = try Database.query(
                    "SELECT count(*) as slow_count, avg(mean_exec_time) as avg_ms " ++
                    "FROM pg_stat_statements WHERE mean_exec_time > 500"
                );

                let summary: Inferred<String> = try infer(
                    "Write a concise one-paragraph digest for a database administrator " ++
                    "summarising the database performance over the last 24 hours. " ++
                    "Stats: {stats}. " ++
                    "Historical digests for context: {self.digest_log}. " ++
                    "Note any trends or improvements."
                );

                let entry = DigestEntry {
                    timestamp: now_iso()
                    slow_queries_found: 0
                    indexes_suggested: 0
                    indexes_applied: 0
                    summary: summary
                };

                self.digest_log = push(self.digest_log, entry);
                self.last_digest_sent = now_iso();

                let payload = to_json(entry);
                let response = try Http.post(self.webhook_url, payload);

                if response.status == 200 {
                    trace "Daily digest sent successfully";
                }
            }
        }

        emit(0);
    }

    on error(e) {
        trace "DigestSender error: {e.message}";
        emit(1);
    }
}

// ─── Supervisor ───────────────────────────────────────────────────────────────

supervisor DbGuardian {
    strategy: OneForOne

    children {
        QueryMonitor {
            restart: Permanent
            handler Infer: DefaultLLM
            last_check: ""
            slow_query_log: []
            check_interval_ms: 60000
        }

        IndexAdvisor {
            restart: Permanent
            handler Infer: DefaultLLM
            suggestions_log: []
            applied_count: 0
            auto_apply: false
        }

        DigestSender {
            restart: Permanent
            handler Infer: DefaultLLM
            digest_log: []
            last_digest_sent: ""
            webhook_url: "https://hooks.slack.com/services/YOUR/WEBHOOK/URL"
        }
    }
}

run DbGuardian;
```

---

## 12. Migration from v1.x

### 12.1 Breaking changes

| Change | Migration |
|---|---|
| `run AgentName` still works but `run SupervisorName` is new | No migration needed for single-agent programs |
| `Agent<T>` is now also spellable as `AgentHandle<T>` | Both work; `Agent<T>` is not removed |
| `on start` / `on stop` still valid | `on resting` is the v2.0 preferred name for `on stop`; `on stop` remains as an alias |

### 12.2 Non-breaking additions

All v1.x programs compile under v2.0 without modification. The new features are purely additive: new keywords, new annotations, new top-level declarations. Existing programs do not use them and are unaffected.

### 12.3 Upgrade path

A typical v1.x program becomes a v2.0 steward program in three steps:

1. Add `@persistent` to fields that should survive restart
2. Add `uses ToolName` and replace direct computation with tool calls
3. Wrap in a supervisor for automatic restart handling

---

## 13. Implementation Roadmap

### Phase 1 — Foundation (pre-v2.0, v1.x milestones)

- [ ] P-04: RFC-0010 Testing Framework
- [ ] P-05: Payload-carrying enum variants
- [ ] P-06: Generics
- [ ] P-07: Expression-level string interpolation
- [ ] P-08: `Option<T>` and `Result<T, E>` as standard types

**Gate:** All v1.x prerequisites complete and stable before Phase 2 begins.

### Phase 2 — Core v2.0 (12–16 weeks)

- [ ] Tool declarations and `sage-tools` crate (4 weeks)
  - `Database`, `Http`, `FileSystem`, `Shell` implementations
  - `uses` clause parsing and capability checking
  - `sage.toml` tool configuration
- [ ] Persistent beliefs and `sage-persistence` crate (4 weeks)
  - `@persistent` annotation
  - SQLite and Postgres backends
  - WAL-based checkpoint protocol
  - `on waking` lifecycle hook
- [ ] Supervision trees and `sage-supervisor` crate (3 weeks)
  - `supervisor` declaration
  - `OneForOne`, `OneForAll`, `RestForOne` strategies
  - Restart intensity limiting
  - Integration with persistent belief recovery
- [ ] Generics in codegen (2 weeks)
  - Monomorphisation pass
  - Commons stdlib first implementation
- [ ] Payload-carrying variants in codegen (1 week)

### Phase 3 — Protocol & Effects (8–10 weeks)

- [ ] Session types and `sage-session` crate (5 weeks)
  - `protocol` declarations
  - `follows` clause
  - Protocol step checker
  - `reply()` builtin
  - Runtime enforcement
- [ ] Algebraic effects for `Infer` (3 weeks)
  - `handler` declarations
  - Effect handler assignment in supervisor children
  - Per-agent LLM configuration

### Phase 4 — Observability & Polish (4–6 weeks)

- [ ] Observability primitives and `sage-observe` crate (3 weeks)
  - `trace` statement
  - `span` blocks
  - Automatic runtime events
  - OTLP export
- [ ] Full lifecycle hooks (1 week)
  - `on pause`, `on resume`
- [ ] `sage test` integration with tools and persistence (2 weeks)
  - Mock tool support in `_test.sg` files
  - In-memory persistence backend for tests

### Phase 5 — Validation (4 weeks)

- [ ] Implement `examples/webapp_steward.sg` end-to-end
- [ ] Implement `examples/db_guardian.sg` end-to-end
- [ ] Full checker test suite for all new error codes
- [ ] Documentation: v2.0 guide sections for each new feature
- [ ] Performance baseline: steward program startup time, checkpoint latency

**Total estimated effort:** ~32–36 weeks from Phase 2 start (assuming v1.x prerequisites complete)

---

## 14. Open Questions

### 14.1 Custom tool implementations

The v2.0 tool system is closed — only `Database`, `Http`, `FileSystem`, and `Shell` are supported. User-defined tools (wrapping arbitrary Rust code or MCP servers) are the obvious next step. The interface is the `tool` declaration syntax, which is already designed for extensibility. The open question is whether user tools are compiled-in (requiring a build step) or loaded at runtime (requiring a plugin system). Deferred to v2.1.

### 14.2 Distributed stewards

All stewards in v2.0 run in a single process. A natural extension is multi-process or multi-machine steward deployment: `DatabaseSteward` on a database server, `APISteward` on an API server. This requires a distributed messaging layer (replacing Tokio mpsc channels with something like NATS or Redis Streams) and a distributed checkpoint store. The language surface would be minimal — `spawn RemoteAgent { ... }` — but the runtime changes are substantial. Deferred to v3.0.

### 14.3 Operator interface

Running `sage run examples/webapp_steward.sg` starts the program, but there is currently no way to interact with a running steward — no CLI to inspect checkpoint state, no way to trigger a schema change manually, no way to read the trace log in real time. A `sage inspect` command and a `sage send` command (to inject messages into a running steward's mailbox) would make the steward architecture operational rather than purely autonomous. Deferred to v2.1.

### 14.4 Schema migration safety

The `DatabaseSteward` in §11.1 uses LLM-generated SQL for migrations. This is powerful but dangerous — a hallucinated `DROP TABLE` would be catastrophic. A dry-run mode (`sage run --dry-run`), a confirmation step for destructive operations, and a rollback mechanism are all needed before this pattern is safe for production use. These are operational concerns that interact with the language (dry-run needs a way to signal to agents that they are in simulation mode) and need their own RFC.

### 14.5 Belief encryption

`@persistent` beliefs are stored in plaintext. Beliefs that contain secrets (API keys, tokens, sensitive user data) need to be encrypted at rest. `@encrypted` as a second annotation on persistent fields, with key management via environment variable or a secrets backend, is the obvious design. Deferred — the key management story needs careful design to avoid being worse than the problem it solves.

---

*The owl watches the whole system. The moose judges it. The stewards maintain it.*
*Ward has been waiting for this since v0.1.0.*
