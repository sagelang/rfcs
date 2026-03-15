# RFC-0006: Agent Message Passing

- **Status:** Implemented
- **Created:** 2026-03-13
- **Author:** Pete Pavlovski
- **Depends on:** RFC-0003 (Compile to Rust)

---

## Table of Contents

1. [Summary](#1-summary)
2. [Motivation](#2-motivation)
3. [Design Goals](#3-design-goals)
4. [Language Design](#4-language-design)
5. [Type System](#5-type-system)
6. [Runtime Semantics](#6-runtime-semantics)
7. [Compiler Changes](#7-compiler-changes)
8. [Codegen](#8-codegen)
9. [Implementation Plan](#9-implementation-plan)
10. [Examples](#10-examples)
11. [Open Questions](#11-open-questions)
12. [Alternatives Considered](#12-alternatives-considered)
13. [Out of Scope](#13-out-of-scope)

---

## 1. Summary

This RFC introduces **typed message passing** between agents. Agents can now declare the message type they accept, receive messages from their mailbox via `receive()`, and send messages to live agent handles via `send(handle, msg)`. The model is an actor model - each agent has a bounded mailbox and runs a concurrent event loop. Message types are declared with existing `enum` declarations and are first-class in the type system.

---

## 2. Motivation

The current agent model is request-response only:

```sage
let worker = spawn Worker { topic: "rust" }
let result = await worker   // blocks until Worker emits
```

This covers a useful subset of coordination patterns, but it has a hard ceiling. Real multi-agent systems need agents that:

- **Run indefinitely** and process a stream of work items
- **React to different message types** differently (task, ping, shutdown, status query)
- **Communicate laterally** - worker-to-worker, not just coordinator-to-worker
- **Decouple dispatch from collection** - fire tasks at an agent without blocking to collect results immediately

Without these, Sage programs devolve into sequential pipelines with parallel wrappers. The fork-join pattern (`spawn` + `await`) handles embarrassingly parallel work but cannot express:

- Worker pools
- Streaming pipelines where each stage processes messages as they arrive
- Supervision (a monitor agent watching others and restarting them on failure)
- Request-reply over a channel (send a message, include a callback handle, receive the reply later)

These patterns are central to the multi-agent systems Sage is designed for. This RFC adds the minimum primitives required to express them.

---

## 3. Design Goals

1. **Typed mailboxes.** An agent's accepted message type is statically known. Sending the wrong type is a compile error.
2. **Minimal new syntax.** The `receives` clause, `receive()` builtin, and `send(handle, msg)` builtin. Nothing else.
3. **Composable with existing primitives.** `spawn` and `await` continue to work unchanged. A long-lived agent still emits a final value that can be `await`ed.
4. **Match Rust's async model.** `receive()` is async - it suspends the agent task, not the thread. Mailboxes are bounded `tokio::sync::mpsc` channels.
5. **Readable in the actor idiom.** A `loop { let msg = receive(); match msg { ... } }` pattern should be idiomatic and obvious.

---

## 4. Language Design

### 4.1 Message Types

Message types are declared using Sage's existing `enum` syntax:

```sage
enum WorkerMsg {
    Task,
    Ping,
    Shutdown,
}
```

### 4.2 Accepting Messages

An agent declares the message type it accepts with a `receives` annotation:

```sage
agent Worker receives WorkerMsg {
    id: Int

    on start {
        loop {
            let msg: WorkerMsg = receive()
            match msg {
                Task => {
                    let result: Inferred<String> = infer("Process task")
                    print(result)
                }
                Ping => {
                    print("Worker {self.id} is alive")
                }
                Shutdown => {
                    break
                }
            }
        }
        emit(0)
    }
}
```

An agent without `receives` is a pure spawn/await agent - its behaviour is unchanged.

### 4.3 `receive()`

`receive()` is a built-in async function. It blocks the current agent until a message arrives in its mailbox, then returns it. The return type is inferred from the agent's declared `receives` type.

```sage
let msg: WorkerMsg = receive()
```

Calling `receive()` inside an agent that has no `receives` declaration is a compile error.

### 4.4 `send(handle, msg)`

`send` is a built-in that puts a message into a live agent's mailbox. It takes the agent handle (the value returned by `spawn`) and the message.

```sage
let w = spawn Worker { id: 1 }
send(w, Task)
send(w, Ping)
send(w, Shutdown)
```

`send` is fire-and-forget - it does not block waiting for the agent to process the message. If the mailbox is full (see section 6.3), `send` blocks until there is space.

### 4.5 `loop` and `break`

Long-lived agents need an indefinite loop. `loop` runs a block until `break` is reached. This is standard in most languages and requires no Sage-specific semantics. It is introduced here because its primary motivating use case is the `loop { receive() }` pattern.

```sage
loop {
    let msg = receive()
    match msg {
        Shutdown => { break }
        _ => { ... }
    }
}
```

---

## 5. Type System

### 5.1 Agent Handle Types

Agent handles are parameterised by the agent's output type:

```
Agent<TResult>
```

For agents with a `receives` clause, the message type is tracked separately in the type checker to validate `send` calls.

### 5.2 Checking `send`

At every `send(handle, msg)` call site, the checker verifies:

1. `handle` has type `Agent<_>` for an agent with a `receives` clause
2. `msg` has the type declared in that agent's `receives` clause

A mismatch is a type error.

### 5.3 Checking `receive`

`receive()` is only valid inside an agent body with a `receives MsgType` declaration. The return type is `MsgType`. Using `receive()` in an agent without a `receives` clause is an error.

### 5.4 Checking `loop`/`break`

The checker tracks an `in_loop` flag. `break` outside a `loop` is an error.

---

## 6. Runtime Semantics

### 6.1 Actor Model

Each agent is an independent Tokio task. When a `receives` clause is present, the agent is allocated a mailbox at spawn time - a bounded `mpsc` channel. The sender half is stored in the `AgentHandle`, returned to the spawner. The receiver half is stored in the `AgentContext`, accessible inside the agent's `on start` body.

### 6.2 Long-Lived Agents and `emit`

An agent that loops indefinitely still emits a final value when it exits - either via an explicit `emit(value)` call or when the loop ends and falls through. This is the value returned by `await`. If a caller never `await`s the handle, the result is simply dropped.

### 6.3 Mailbox Backpressure

Mailboxes are bounded. The default size is **128 messages**. When the mailbox is full, `send` suspends the sender's Tokio task until space opens. This provides natural backpressure rather than silent message loss.

### 6.4 No Ordering Guarantees Across Senders

Messages from a single sender to a single agent are delivered in order. Messages from multiple senders are interleaved in arrival order - no global ordering is guaranteed. This is consistent with `tokio::sync::mpsc` semantics and with actor systems generally.

---

## 7. Compiler Changes

### 7.1 Lexer (`sage-lexer`)

New tokens:

| Token | Keyword |
|---|---|
| `KwLoop` | `loop` |
| `KwBreak` | `break` |
| `KwReceives` | `receives` |
| `KwReceive` | `receive` |

### 7.2 Parser (`sage-parser`)

New/modified AST nodes:

- `Stmt::Loop { body: Block, span: Span }`
- `Stmt::Break { span: Span }`
- `Expr::Receive { span: Span }`
- `AgentDecl` gains `receives: Option<TypeExpr>`

### 7.3 Type Checker (`sage-checker`)

1. Track `in_loop: bool` flag for break validation
2. Store `receives_type: Option<Type>` in `AgentInfo`
3. Check `receive()` only valid in agents with `receives` clause
4. Check `send(handle, msg)` type matches agent's receives type

### 7.4 Codegen (`sage-codegen`)

- Generate `loop { }` and `break;` directly to Rust
- Generate `ctx.receive().await?` for `receive()`
- Generate `handle.send(msg).await?` for `send(handle, msg)`

---

## 8. Examples

### 8.1 Worker Pool

```sage
enum WorkerMsg {
    Task,
    Shutdown,
}

agent Worker receives WorkerMsg {
    id: Int

    on start {
        loop {
            let msg: WorkerMsg = receive()
            match msg {
                Task => {
                    let result: Inferred<String> = infer("Summarise something")
                    print("Worker {self.id}: {result}")
                }
                Shutdown => {
                    break
                }
            }
        }
        emit(0)
    }
}

agent Coordinator {
    on start {
        let w1 = spawn Worker { id: 1 }
        let w2 = spawn Worker { id: 2 }

        send(w1, Task)
        send(w2, Task)
        send(w1, Task)
        send(w2, Task)

        send(w1, Shutdown)
        send(w2, Shutdown)

        await w1
        await w2

        emit(0)
    }
}

run Coordinator
```

---

## 9. Implementation Plan

### Phase 1 - Lexer & Parser
- [x] Add new tokens: `loop`, `break`, `receives`, `receive`
- [x] Add `Stmt::Loop` and `Stmt::Break` to AST
- [x] Add `Expr::Receive` to AST
- [x] Extend `AgentDecl` with `receives` field
- [x] Implement parsing for all new constructs

### Phase 2 - Type Checker
- [x] Track `in_loop` flag for break validation
- [x] Store `receives_type` in `AgentInfo`
- [x] Check `receive()` context
- [x] Check `send()` type matching (deferred - requires handle type tracking)

### Phase 3 - Runtime
- [x] Update `AgentHandle` with message sender
- [x] Add `receive()` method to `AgentContext`
- [x] Add `receive_timeout()` method to `AgentContext`
- [x] Update `spawn_agent` for mailbox creation

### Phase 4 - Codegen
- [x] Generate `loop`/`break`
- [x] Generate `receive()` calls
- [x] Generate `send()` calls

---

## 10. Open Questions

**Q1: Should `AgentHandle` be a first-class type that can be stored in beliefs?**

The type annotation syntax would be `Agent<Int>`. This is already supported in the type system.

**Q2: Should `send` be fallible in Sage source code?**

Currently `send` silently errors if the mailbox is closed. Making it explicitly fallible would surface dropped-handle bugs at the language level. Deferred to a follow-up - error handling as a whole needs its own RFC.

---

## 11. Alternatives Considered

### 11.1 Untyped Mailboxes

Agents could accept `Any` messages and downcast at runtime. Rejected: this eliminates the primary safety guarantee (type mismatch on `send` is a compile error).

### 11.2 Channels as First-Class Values

Instead of agent-owned mailboxes, expose `Channel<T>` as a value the programmer creates explicitly. Considered but rejected for the POC - the agent-owns-mailbox model keeps the common case simple.

---

## 12. Out of Scope

- **Error handling** - `send` and `receive` failures need a proper `Result`/`Option` story
- **Broadcast channels** - one-to-many messaging
- **Agent supervision trees** - automatic restart of failed agents
- **Session types** - statically verified communication protocols

---

*Message passing turns Sage agents from parallel functions into persistent actors. Ward approves.*
