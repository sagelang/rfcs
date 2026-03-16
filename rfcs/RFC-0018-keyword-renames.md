# RFC-0018: Keyword Renames (Divine, Oracle, Summon, Yield)

- **Status:** Draft
- **Created:** 2026-03-16
- **Author:** Pete Pavlovski
- **Amends:** RFC-0005 (User-Defined Types), RFC-0006 (Message Passing)

---

## Summary

Rename four keywords/built-ins to better reflect Sage's thematic identity:

| Current | Proposed | Purpose |
|---------|----------|---------|
| `infer(...)` | `divine(...)` | LLM inference call |
| `Inferred<T>` | `Oracle<T>` | Structured LLM output type |
| `spawn` | `summon` | Create an agent |
| `emit` | `yield` | Produce a value from an agent |

---

## Motivation

Sage is a language built around agents and LLM inference. The current keywords (`infer`, `spawn`, `emit`) are functional but generic — they could belong to any language.

The proposed names embrace Sage's identity:

- **`divine`** — To divine is to seek hidden knowledge through supernatural means. Perfect for querying an LLM.
- **`Oracle<T>`** — An oracle delivers prophecies in structured form. The type that wraps divined knowledge.
- **`summon`** — Agents aren't mere processes; they're autonomous entities called into existence.
- **`yield`** — A sage yields wisdom. Cleaner than "emit" and consistent with generator semantics in other languages.

This gives Sage a distinctive voice. Code reads less like "call the API, spawn a thread" and more like "divine the answer, summon the agent."

---

## Changes

### `infer(...)` → `divine(...)`

```sage
// Before
let answer = infer("What is the capital of France?")

// After
let answer = divine("What is the capital of France?")
```

### `Inferred<T>` → `Oracle<T>`

```sage
// Before
fn get_user(prompt: String) -> Inferred<User> {
    infer(prompt)
}

// After
fn get_user(prompt: String) -> Oracle<User> {
    divine(prompt)
}
```

### `spawn` → `summon`

```sage
// Before
let agent = spawn MyAgent { name: "Alice" }

// After
let agent = summon MyAgent { name: "Alice" }
```

### `emit` → `yield`

```sage
// Before
on message(request: Query) {
    let result = process(request)
    emit result
}

// After
on message(request: Query) {
    let result = process(request)
    yield result
}
```

---

## Affected Components

1. **sage-parser** — Update lexer tokens and AST node names
2. **sage-checker** — Update type names (`Inferred` → `Oracle`)
3. **sage-codegen** — Update keyword mappings in code generation
4. **sage-runtime** — Update any runtime type names if exposed
5. **sage-sense** — Update LSP completions and hover info
6. **Documentation** — Update all examples in book, README, and RFCs

---

## Backwards Compatibility

This is a breaking change. Old keywords will not be recognised after the update.

Given Sage is pre-1.0, no deprecation period is provided. Users should update their code to use the new keywords.

---

## Implementation

1. Update lexer in `sage-parser/src/lexer.rs` (or equivalent)
2. Update AST types in `sage-parser/src/ast.rs`
3. Update type definitions in `sage-checker/src/types.rs`
4. Update codegen mappings in `sage-codegen/src/generator.rs`
5. Update all test fixtures and example projects
6. Update documentation

---

## Open Questions

None.
