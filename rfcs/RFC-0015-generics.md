# RFC-0015: Generics (Parametric Polymorphism)

- **Status:** Draft
- **Created:** 2026-03-16
- **Author:** Sage Contributors
- **Depends on:** RFC-0005 (User-Defined Types), RFC-0009 (First-Class Functions), RFC-0010 (Maps, Tuples, Result)

---

## Table of Contents

1. [Summary](#1-summary)
2. [Motivation](#2-motivation)
3. [Design Goals](#3-design-goals)
4. [Language Changes Summary](#4-language-changes-summary)
5. [Generic Functions](#5-generic-functions)
6. [Generic Records](#6-generic-records)
7. [Generic Enums](#7-generic-enums)
8. [Type Inference](#8-type-inference)
9. [Monomorphisation](#9-monomorphisation)
10. [Interaction with Existing Parameterised Types](#10-interaction-with-existing-parameterised-types)
11. [Checker Rules](#11-checker-rules)
12. [Codegen](#12-codegen)
13. [Standard Library Enablement](#13-standard-library-enablement)
14. [Grammar Changes](#14-grammar-changes)
15. [New Error Codes](#15-new-error-codes)
16. [Implementation Plan](#16-implementation-plan)
17. [Open Questions](#17-open-questions)

---

## 1. Summary

This RFC introduces **generics** (parametric polymorphism) to Sage, enabling:

- **Generic functions**: `fn map<T, U>(list: List<T>, f: Fn(T) -> U) -> List<U>`
- **Generic records**: `record Pair<A, B> { first: A, second: B }`
- **Generic enums**: `enum Tree<T> { Leaf(T), Node(Tree<T>, Tree<T>) }`

Generics are **monomorphised at compile time** (Rust-style), producing specialised code for each concrete type combination. Type parameters are **unconstrained** in this RFC — trait-style bounds are deferred to a future RFC.

This is a prerequisite for Sage v2.0, enabling the Commons standard library with functions like `map`, `filter`, `find`, `reduce`, and generic data structures like `Pair<A, B>`, `Either<L, R>`, and `Timestamped<T>`.

---

## 2. Motivation

### 2.1 The stdlib cannot be written without generics

The most impactful standard library functions operate on collections:

```sage
// Today: impossible to write
fn map(list: ???, f: ???) -> ???

// With generics: straightforward
fn map<T, U>(list: List<T>, f: Fn(T) -> U) -> List<U> {
    let result: List<U> = [];
    for item in list {
        result = push(result, f(item));
    }
    return result;
}
```

Without generics, every list operation must be written separately for `List<Int>`, `List<String>`, `List<ResearchResult>`, etc. — an impossible maintenance burden.

### 2.2 User-defined generic data structures

Real programs need reusable container types:

```sage
// A paginated API response — works for any item type
record Page<T> {
    items: List<T>
    total: Int
    page: Int
    page_size: Int
}

// A timestamped wrapper — works for any value type
record Timestamped<T> {
    value: T
    created_at: String
    updated_at: String
}

// A cache entry — works for any cached value
record CacheEntry<V> {
    value: V
    expires_at: Int
    hit_count: Int
}
```

Today, users must duplicate these structures for every concrete type, polluting the namespace and violating DRY.

### 2.3 Generic enums for sum types

The `Result<T, E>` and `Option<T>` types are currently special-cased builtins. With generics, users can define their own sum types:

```sage
// A binary tree
enum Tree<T> {
    Leaf(T)
    Node(Tree<T>, Tree<T>)
}

// A choice between two types
enum Either<L, R> {
    Left(L)
    Right(R)
}

// A value that may be loading, loaded, or failed
enum Loadable<T, E> {
    Loading
    Loaded(T)
    Failed(E)
}
```

### 2.4 The v2.0 gate

The v2.0 roadmap lists generics as prerequisite P-06. The Commons stdlib, tool declarations with typed results, and session types all depend on parametric polymorphism. This RFC unblocks all of them.

---

## 3. Design Goals

1. **Rust-style monomorphisation.** No runtime type information. Generic code is compiled to specialised code for each concrete type combination.

2. **Unconstrained type parameters.** In this RFC, type parameters have no bounds. A generic function can only perform operations that work on all types (assignment, passing to other generic functions, returning). Trait-style constraints are deferred.

3. **Inference where possible.** Type arguments can often be inferred from usage. Explicit turbofish syntax (`::<T>`) is available when inference fails.

4. **Backward compatibility.** Existing `List<T>`, `Option<T>`, `Map<K, V>`, and `Result<T, E>` continue to work exactly as before. They become examples of generic types rather than special cases.

5. **Simple mental model.** Generic code is "code with holes" — the type checker validates that the holes are filled consistently, then the codegen stamps out concrete versions.

6. **Error messages that help.** When type inference fails or type arguments don't match, the error message should point to the specific location and explain what types were expected vs provided.

---

## 4. Language Changes Summary

| Change | Description |
|--------|-------------|
| Type parameters on functions | `fn foo<T, U>(...) -> ...` |
| Type parameters on records | `record Foo<T> { ... }` |
| Type parameters on enums | `enum Foo<T> { ... }` |
| Explicit type application | `foo::<Int, String>(...)` |
| Type parameter in type position | `T` where `T` is a declared type parameter |

---

## 5. Generic Functions

### 5.1 Declaration syntax

Type parameters are declared in angle brackets after the function name:

```sage
fn identity<T>(x: T) -> T {
    return x;
}

fn swap<A, B>(pair: (A, B)) -> (B, A) {
    return (pair.1, pair.0);
}

fn map<T, U>(list: List<T>, f: Fn(T) -> U) -> List<U> {
    let result: List<U> = [];
    for item in list {
        result = push(result, f(item));
    }
    return result;
}
```

Type parameters are single uppercase identifiers by convention (`T`, `U`, `A`, `B`), but any valid identifier is accepted. Multi-letter names like `Item`, `Key`, `Value` are allowed.

### 5.2 Using type parameters

Within the function body, type parameters can be used anywhere a type is expected:

```sage
fn wrap_in_list<T>(value: T) -> List<T> {
    return [value];
}

fn make_pair<A, B>(first: A, second: B) -> (A, B) {
    return (first, second);
}
```

### 5.3 Calling generic functions

When calling a generic function, type arguments are usually inferred:

```sage
let x = identity(42);           // T inferred as Int
let y = identity("hello");      // T inferred as String

let nums = [1, 2, 3];
let doubled = map(nums, |n: Int| n * 2);  // T=Int, U=Int inferred
```

When inference fails or is ambiguous, explicit type arguments use turbofish syntax:

```sage
let empty: List<Int> = [];
let mapped = map::<Int, String>(empty, |n: Int| "{n}");
```

The turbofish (`::<...>`) appears immediately after the function name, before the argument list.

### 5.4 Constraints on generic function bodies

Because type parameters are unconstrained, the function body can only perform operations that work on all types:

**Allowed:**
- Assign `T` values to `T` variables
- Pass `T` values to other generic functions expecting `T`
- Return `T` values
- Store `T` in generic containers (`List<T>`, `Option<T>`, etc.)
- Use `T` in tuple or record fields

**Not allowed (in this RFC):**
- Call methods on `T` (no method syntax exists yet)
- Use `==` on `T` unless `T` is known to be comparable
- Use `+` or other operators on `T`
- Print `T` directly (requires a `Display` bound)

```sage
// Valid
fn first<T>(list: List<T>) -> Option<T> {
    if len(list) == 0 {
        return None;
    }
    return Some(list[0]);  // Assuming index access returns T
}

// Invalid — cannot use == on unconstrained T
fn contains<T>(list: List<T>, target: T) -> Bool {
    for item in list {
        if item == target {  // E100: cannot apply == to unconstrained type parameter T
            return true;
        }
    }
    return false;
}
```

The `contains` example requires a future RFC that introduces type constraints (e.g., `T: Eq`).

### 5.5 Generic functions in agents

Generic functions can be called from agent handlers:

```sage
fn double_all<T>(items: List<T>, f: Fn(T) -> T) -> List<T> {
    return map(items, |x: T| f(f(x)));
}

agent Processor {
    on start {
        let nums = [1, 2, 3];
        let result = double_all(nums, |n: Int| n + 1);
        print("{result}");  // [3, 4, 5]
        emit(0);
    }
}
```

### 5.6 Recursive generic functions

Generic functions can be recursive:

```sage
fn tree_size<T>(tree: Tree<T>) -> Int {
    match tree {
        Tree.Leaf(_) => 1
        Tree.Node(left, right) => tree_size(left) + tree_size(right) + 1
    }
}
```

The recursive call `tree_size(left)` uses the same type parameter `T`.

---

## 6. Generic Records

### 6.1 Declaration syntax

Type parameters are declared in angle brackets after the record name:

```sage
record Pair<A, B> {
    first: A
    second: B
}

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

### 6.2 Construction

Generic records are constructed by providing concrete type arguments (inferred or explicit):

```sage
// Type arguments inferred from field values
let pair = Pair { first: 42, second: "hello" };
// pair: Pair<Int, String>

// Explicit type annotation
let page: Page<ResearchResult> = Page {
    items: results
    total: 100
    page: 1
    page_size: 10
};

// Explicit turbofish on the type name
let empty_page = Page::<String> {
    items: []
    total: 0
    page: 1
    page_size: 10
};
```

### 6.3 Field access

Field access works identically to non-generic records:

```sage
let pair = Pair { first: 42, second: "hello" };
let n: Int = pair.first;
let s: String = pair.second;
```

### 6.4 Generic records as function parameters and returns

```sage
fn unwrap_timestamped<T>(ts: Timestamped<T>) -> T {
    return ts.value;
}

fn paginate<T>(items: List<T>, page: Int, page_size: Int) -> Page<T> {
    let start = (page - 1) * page_size;
    let page_items = take(drop(items, start), page_size);
    return Page {
        items: page_items
        total: len(items)
        page: page
        page_size: page_size
    };
}
```

### 6.5 Nested generic records

Generic records can contain other generic types:

```sage
record CacheEntry<K, V> {
    key: K
    value: V
    metadata: Timestamped<V>
}

// CacheEntry<String, ResearchResult> contains Timestamped<ResearchResult>
```

### 6.6 Generic records as agent state

Generic records can be used as agent state fields:

```sage
agent PaginatedFetcher {
    current_page: Page<String>

    on start {
        self.current_page = Page {
            items: []
            total: 0
            page: 1
            page_size: 20
        };
        // ...
    }
}
```

### 6.7 Generic records in `Inferred<T>`

Generic records can be used with `Inferred<T>` when all type arguments are concrete:

```sage
record ExtractedData<T> {
    data: T
    confidence: Float
    source: String
}

agent Extractor {
    on start {
        // T must be concrete for Inferred<T>
        let result: Inferred<ExtractedData<ResearchResult>> = try infer(
            "Extract research data from this text..."
        );
        emit(result);
    }
}
```

The JSON schema is generated for the concrete type `ExtractedData<ResearchResult>`.

---

## 7. Generic Enums

### 7.1 Declaration syntax

Type parameters are declared in angle brackets after the enum name:

```sage
enum Tree<T> {
    Leaf(T)
    Node(Tree<T>, Tree<T>)
}

enum Either<L, R> {
    Left(L)
    Right(R)
}

enum Loadable<T, E> {
    Loading
    Loaded(T)
    Failed(E)
}
```

Variants can carry payloads that use type parameters, or carry no payload.

### 7.2 Construction

```sage
let leaf: Tree<Int> = Tree.Leaf(42);
let node: Tree<Int> = Tree.Node(Tree.Leaf(1), Tree.Leaf(2));

let choice: Either<String, Int> = Either.Left("hello");
let other: Either<String, Int> = Either.Right(42);

let loading: Loadable<String, String> = Loadable.Loading;
let loaded: Loadable<String, String> = Loadable.Loaded("data");
let failed: Loadable<String, String> = Loadable.Failed("network error");
```

### 7.3 Pattern matching

Pattern matching on generic enums works the same as non-generic enums:

```sage
fn tree_sum(tree: Tree<Int>) -> Int {
    match tree {
        Tree.Leaf(n) => n
        Tree.Node(left, right) => tree_sum(left) + tree_sum(right)
    }
}

fn unwrap_either<L, R>(e: Either<L, R>, default_left: L, default_right: R) -> (L, R) {
    match e {
        Either.Left(l)  => (l, default_right)
        Either.Right(r) => (default_left, r)
    }
}
```

### 7.4 Recursive generic enums

Generic enums can be recursive (as shown with `Tree<T>`):

```sage
enum JsonValue {
    Null
    Bool(Bool)
    Number(Float)
    Str(String)
    Array(List<JsonValue>)
    Object(Map<String, JsonValue>)
}
```

Note: `JsonValue` is not generic itself, but uses `List<JsonValue>` and `Map<String, JsonValue>` recursively.

### 7.5 Generic enums in `Inferred<T>`

Generic enums work with `Inferred<T>` using the tagged-union JSON schema format:

```sage
let result: Inferred<Either<String, Int>> = try infer(
    "Return either a string description or an integer count..."
);
```

Schema:
```json
{
  "oneOf": [
    { "type": "object", "properties": { "Left":  { "type": "string" } }, "required": ["Left"]  },
    { "type": "object", "properties": { "Right": { "type": "integer" } }, "required": ["Right"] }
  ]
}
```

---

## 8. Type Inference

### 8.1 Inference algorithm

Type inference for generics uses a constraint-based unification algorithm:

1. **Collect constraints.** For each use of a type parameter, record what type it must equal based on context.

2. **Unify constraints.** Check that all constraints on a type parameter are consistent (unify to the same concrete type).

3. **Substitute.** Replace type parameters with their inferred concrete types.

```sage
fn identity<T>(x: T) -> T { return x; }

let y = identity(42);
// Constraint: T = Int (from argument 42)
// Result: T = Int, y: Int
```

### 8.2 Bidirectional inference

Type information flows both from arguments and from expected return type:

```sage
fn first<T>(list: List<T>) -> Option<T> { ... }

// Inference from argument
let x = first([1, 2, 3]);
// Constraint: List<T> = List<Int> => T = Int
// Result: x: Option<Int>

// Inference from expected type
let y: Option<String> = first([]);
// Constraint: Option<T> = Option<String> => T = String
// Also: List<T> = List<String>
// Result: T = String
```

### 8.3 Inference failure

When constraints are inconsistent or insufficient, inference fails:

```sage
// Inconsistent constraints
fn bad_example<T>(a: T, b: T) -> T { ... }
let z = bad_example(42, "hello");
// Constraint: T = Int (from 42), T = String (from "hello")
// E101: type parameter T has conflicting constraints: Int vs String

// Insufficient constraints
let empty: List<???> = [];
// E102: cannot infer type parameter; add explicit annotation
```

### 8.4 Turbofish for disambiguation

When inference fails, the programmer provides explicit type arguments:

```sage
// Empty list needs explicit type
let empty = []::<Int>;       // Not valid Sage syntax

// Or use type annotation
let empty: List<Int> = [];   // Preferred

// Turbofish on function call
let result = parse::<ResearchResult>(json_string);
```

### 8.5 Partial inference

Some type arguments may be inferred while others need to be explicit:

```sage
fn transform<T, U>(input: T, f: Fn(T) -> U) -> U { ... }

// T inferred from input, U inferred from closure return type
let result = transform(42, |n: Int| "{n}");
// T = Int, U = String
```

---

## 9. Monomorphisation

### 9.1 Overview

Sage uses **monomorphisation** — generic code is compiled to specialised versions for each concrete type combination used in the program. This is the same strategy as Rust.

```sage
fn identity<T>(x: T) -> T { return x; }

let a = identity(42);       // Generates identity_Int
let b = identity("hello");  // Generates identity_String
let c = identity(true);     // Generates identity_Bool
```

### 9.2 Generated Rust code

The codegen produces a separate Rust function for each monomorphised instantiation:

**Sage:**
```sage
fn identity<T>(x: T) -> T { return x; }

let a = identity(42);
let b = identity("hello");
```

**Generated Rust:**
```rust
fn identity_i64(x: i64) -> i64 { x }
fn identity_String(x: String) -> String { x }

let a = identity_i64(42);
let b = identity_String("hello".to_string());
```

### 9.3 Name mangling

Monomorphised function names use a deterministic mangling scheme:

```
{original_name}_{type_param_1}_{type_param_2}_...
```

Type names are mangled to valid Rust identifiers:
- `Int` → `i64`
- `Float` → `f64`
- `Bool` → `bool`
- `String` → `String`
- `List<T>` → `Vec_{T}`
- `Option<T>` → `Option_{T}`
- User-defined types → their name

Example: `map<Int, String>` → `map_i64_String`

### 9.4 Record and enum monomorphisation

Generic records and enums are also monomorphised:

**Sage:**
```sage
record Pair<A, B> {
    first: A
    second: B
}

let p = Pair { first: 42, second: "hello" };
```

**Generated Rust:**
```rust
#[derive(Debug, Clone, PartialEq, serde::Serialize, serde::Deserialize)]
struct Pair_i64_String {
    first: i64,
    second: String,
}

let p = Pair_i64_String { first: 42, second: "hello".to_string() };
```

### 9.5 Compilation phases

The monomorphisation pass runs after type checking and before Rust codegen:

```
Source → Parse → Type Check → Monomorphise → Codegen → Rust
```

The monomorphisation pass:
1. Collects all concrete instantiations of generic items
2. Generates a monomorphised version of each
3. Rewrites call sites to use the monomorphised names

### 9.6 Code size implications

Monomorphisation can increase binary size if the same generic function is instantiated with many different types. In practice, this is rarely a problem for Sage programs, and the runtime performance benefit (no vtables, no boxing) outweighs the size cost.

---

## 10. Interaction with Existing Parameterised Types

### 10.1 `List<T>`, `Option<T>`, `Map<K, V>`, `Result<T, E>`

These types are currently special-cased in the compiler. With this RFC, they become regular generic types — albeit still builtin (not user-defined).

The syntax and semantics are unchanged:

```sage
let nums: List<Int> = [1, 2, 3];
let maybe: Option<String> = Some("hello");
let cache: Map<String, Int> = {};
let outcome: Result<Data, String> = Ok(data);
```

### 10.2 Unification with user-defined generics

The type checker treats builtin generic types and user-defined generic types uniformly. A function that takes `List<T>` can take any `List<Foo>`:

```sage
record MyData { value: Int }

fn process<T>(items: List<T>) -> Int {
    return len(items);
}

let my_items: List<MyData> = [MyData { value: 1 }];
let count = process(my_items);  // T = MyData
```

### 10.3 `Inferred<T>` with generic types

`Inferred<GenericType<ConcreteArgs>>` works when all type arguments are concrete:

```sage
record Wrapper<T> { inner: T }

// Valid — T is concrete (String)
let result: Inferred<Wrapper<String>> = try infer("...");

// Invalid — T is a type parameter, not concrete
fn bad<T>() -> Inferred<Wrapper<T>> { ... }  // E103: Inferred<T> requires concrete type
```

---

## 11. Checker Rules

### 11.1 Type parameter scope

Type parameters are in scope within the item they are declared on:

- For functions: in the parameter list, return type, and body
- For records: in field types
- For enums: in variant payload types

```sage
fn foo<T>(x: T) -> T {
    let y: T = x;  // T in scope
    return y;
}

// T not in scope outside the function
let z: T = ...;  // E104: unknown type T
```

### 11.2 Type parameter shadowing

Type parameters cannot shadow each other or existing type names:

```sage
fn bad<T, T>(x: T) -> T { ... }  // E105: duplicate type parameter T

record Point { x: Int, y: Int }
fn also_bad<Point>(x: Point) -> Point { ... }  // E106: type parameter shadows type Point
```

### 11.3 Type argument count

The number of type arguments must match the number of type parameters:

```sage
record Pair<A, B> { first: A, second: B }

let p: Pair<Int> = ...;           // E107: Pair expects 2 type arguments, got 1
let q: Pair<Int, Int, Int> = ...; // E107: Pair expects 2 type arguments, got 3
```

### 11.4 Recursive type parameter usage

Type parameters can be used recursively in the definition:

```sage
enum Tree<T> {
    Leaf(T)
    Node(Tree<T>, Tree<T>)  // Tree<T> uses T
}
```

But infinite expansion is prevented by requiring at least one base case variant:

```sage
// Invalid — no base case, infinite type
enum Bad<T> {
    Only(Bad<T>)  // E108: recursive enum has no base case
}
```

### 11.5 Concrete type requirements

Some contexts require concrete types (no unresolved type parameters):

- `Inferred<T>` argument
- `@persistent` field type (for serialisation)
- Top-level `const` type

```sage
fn generic_infer<T>() -> T {
    let x: Inferred<T> = try infer("...");  // E103: Inferred requires concrete type
    return x;
}
```

---

## 12. Codegen

### 12.1 Monomorphisation pass

Before Rust codegen, a monomorphisation pass:

1. **Collects instantiations.** Walks the typed AST and records every concrete instantiation of a generic item.

2. **Generates specialised items.** For each instantiation, generates a copy of the item with type parameters substituted.

3. **Rewrites call sites.** Replaces generic calls with calls to the specialised versions.

### 12.2 Function codegen

**Sage:**
```sage
fn first<T>(list: List<T>) -> Option<T> {
    if len(list) == 0 {
        return None;
    }
    return Some(list[0]);
}

let x = first([1, 2, 3]);
let y = first(["a", "b"]);
```

**Generated Rust:**
```rust
fn first_i64(list: Vec<i64>) -> Option<i64> {
    if list.len() == 0 {
        return None;
    }
    return Some(list[0]);
}

fn first_String(list: Vec<String>) -> Option<String> {
    if list.is_empty() {
        return None;
    }
    return Some(list[0].clone());
}

let x = first_i64(vec![1, 2, 3]);
let y = first_String(vec!["a".to_string(), "b".to_string()]);
```

### 12.3 Record codegen

**Sage:**
```sage
record Pair<A, B> {
    first: A
    second: B
}

let p: Pair<Int, String> = Pair { first: 42, second: "hello" };
let q: Pair<Bool, Float> = Pair { first: true, second: 3.14 };
```

**Generated Rust:**
```rust
#[derive(Debug, Clone, PartialEq, serde::Serialize, serde::Deserialize)]
struct Pair_i64_String {
    first: i64,
    second: String,
}

#[derive(Debug, Clone, PartialEq, serde::Serialize, serde::Deserialize)]
struct Pair_bool_f64 {
    first: bool,
    second: f64,
}

let p = Pair_i64_String { first: 42, second: "hello".to_string() };
let q = Pair_bool_f64 { first: true, second: 3.14 };
```

### 12.4 Enum codegen

**Sage:**
```sage
enum Either<L, R> {
    Left(L)
    Right(R)
}

let e: Either<String, Int> = Either.Left("hello");
```

**Generated Rust:**
```rust
#[derive(Debug, Clone, PartialEq, serde::Serialize, serde::Deserialize)]
enum Either_String_i64 {
    Left(String),
    Right(i64),
}

let e = Either_String_i64::Left("hello".to_string());
```

### 12.5 JSON schema generation

For `Inferred<GenericType<...>>`, the JSON schema is generated for the concrete instantiation:

**Sage:**
```sage
record Wrapper<T> {
    value: T
    metadata: String
}

let result: Inferred<Wrapper<Int>> = try infer("...");
```

**Generated schema:**
```json
{
  "type": "object",
  "properties": {
    "value": { "type": "integer" },
    "metadata": { "type": "string" }
  },
  "required": ["value", "metadata"]
}
```

---

## 13. Standard Library Enablement

### 13.1 List operations

With generics, the Commons stdlib can define:

```sage
fn map<T, U>(list: List<T>, f: Fn(T) -> U) -> List<U>
fn filter<T>(list: List<T>, pred: Fn(T) -> Bool) -> List<T>
fn find<T>(list: List<T>, pred: Fn(T) -> Bool) -> Option<T>
fn reduce<T, U>(list: List<T>, init: U, f: Fn(U, T) -> U) -> U
fn flatten<T>(list: List<List<T>>) -> List<T>
fn zip<T, U>(a: List<T>, b: List<U>) -> List<(T, U)>
fn take<T>(list: List<T>, n: Int) -> List<T>
fn drop<T>(list: List<T>, n: Int) -> List<T>
fn reverse<T>(list: List<T>) -> List<T>
fn concat<T>(a: List<T>, b: List<T>) -> List<T>
fn any<T>(list: List<T>, pred: Fn(T) -> Bool) -> Bool
fn all<T>(list: List<T>, pred: Fn(T) -> Bool) -> Bool
fn count<T>(list: List<T>, pred: Fn(T) -> Bool) -> Int
fn first<T>(list: List<T>) -> Option<T>
fn last<T>(list: List<T>) -> Option<T>
fn nth<T>(list: List<T>, n: Int) -> Option<T>
```

### 13.2 Option operations

```sage
fn unwrap_or<T>(opt: Option<T>, default: T) -> T
fn map_option<T, U>(opt: Option<T>, f: Fn(T) -> U) -> Option<U>
fn and_then<T, U>(opt: Option<T>, f: Fn(T) -> Option<U>) -> Option<U>
fn or_else<T>(opt: Option<T>, f: Fn() -> Option<T>) -> Option<T>
fn is_some<T>(opt: Option<T>) -> Bool
fn is_none<T>(opt: Option<T>) -> Bool
```

### 13.3 Result operations

```sage
fn unwrap_or_else<T, E>(result: Result<T, E>, f: Fn(E) -> T) -> T
fn map_result<T, U, E>(result: Result<T, E>, f: Fn(T) -> U) -> Result<U, E>
fn map_err<T, E, F>(result: Result<T, E>, f: Fn(E) -> F) -> Result<T, F>
fn and_then_result<T, U, E>(result: Result<T, E>, f: Fn(T) -> Result<U, E>) -> Result<U, E>
fn is_ok<T, E>(result: Result<T, E>) -> Bool
fn is_err<T, E>(result: Result<T, E>) -> Bool
```

### 13.4 Map operations

```sage
fn map_entries<K, V, U>(m: Map<K, V>, f: Fn(K, V) -> U) -> List<U>
fn filter_map<K, V>(m: Map<K, V>, pred: Fn(K, V) -> Bool) -> Map<K, V>
fn merge<K, V>(a: Map<K, V>, b: Map<K, V>) -> Map<K, V>
```

### 13.5 Generic utility types

```sage
record Pair<A, B> {
    first: A
    second: B
}

record Triple<A, B, C> {
    first: A
    second: B
    third: C
}

enum Either<L, R> {
    Left(L)
    Right(R)
}

record Timestamped<T> {
    value: T
    timestamp: Int
}

record Tagged<T> {
    value: T
    tag: String
}
```

---

## 14. Grammar Changes

### 14.1 Type parameters

```ebnf
type_params     ::= "<" type_param ("," type_param)* ">"

type_param      ::= IDENT
```

### 14.2 Updated declarations

```ebnf
fn_decl         ::= "pub"? "fn" IDENT type_params? "(" param_list? ")"
                    ("->" type_expr)? ("fails")? block

record_decl     ::= "pub"? "record" IDENT type_params? "{" record_field* "}"

enum_decl       ::= "pub"? "enum" IDENT type_params? "{" enum_variant* "}"
```

### 14.3 Type expressions

```ebnf
type_expr       ::= primitive_type
                  | "List" "<" type_expr ">"
                  | "Option" "<" type_expr ">"
                  | "Map" "<" type_expr "," type_expr ">"
                  | "Result" "<" type_expr "," type_expr ">"
                  | "Inferred" "<" type_expr ">"
                  | "Agent" "<" IDENT ">"
                  | "Fn" "(" type_list? ")" "->" type_expr
                  | tuple_type
                  | named_type

named_type      ::= IDENT type_args?

type_args       ::= "<" type_expr ("," type_expr)* ">"

tuple_type      ::= "(" type_expr "," type_expr ("," type_expr)* ")"
```

### 14.4 Turbofish syntax

```ebnf
call_expr       ::= IDENT turbofish? "(" arg_list? ")"
                  | expr "." IDENT turbofish? "(" arg_list? ")"

turbofish       ::= "::" "<" type_expr ("," type_expr)* ">"

// Constructor with turbofish
record_expr     ::= IDENT turbofish? "{" field_init* "}"
```

---

## 15. New Error Codes

| Code | Name | Description |
|------|------|-------------|
| E100 | `UnconstrainedOperation` | Operation (e.g., `==`, `+`) applied to unconstrained type parameter |
| E101 | `ConflictingTypeConstraints` | Type parameter has conflicting inferred types |
| E102 | `CannotInferType` | Type parameter cannot be inferred; explicit annotation required |
| E103 | `ConcreteTypeRequired` | Context requires concrete type, not type parameter |
| E104 | `UnknownTypeParameter` | Type parameter used outside its scope |
| E105 | `DuplicateTypeParameter` | Type parameter declared twice in same item |
| E106 | `TypeParameterShadows` | Type parameter shadows existing type name |
| E107 | `TypeArgCountMismatch` | Wrong number of type arguments for generic type |
| E108 | `InfiniteType` | Recursive type has no base case |
| E109 | `TypeParameterNotConcrete` | Type parameter used where concrete type required |

---

## 16. Implementation Plan

### Phase 1 — Parser & AST (3–4 days)

- Add `type_params: Vec<Ident>` field to `FnDecl`, `RecordDecl`, `EnumDecl` AST nodes
- Parse type parameter declarations: `<T>`, `<A, B>`, `<T, U, V>`
- Parse type arguments in type expressions: `Pair<Int, String>`
- Parse turbofish syntax: `foo::<Int>(x)`
- Parser tests for all new syntax

### Phase 2 — Type System (5–7 days)

- Extend `TypeExpr` to handle type parameters as distinct from concrete types
- Add type parameter environment to checker context
- Implement type parameter scoping (enter/exit scope for generic items)
- Implement constraint collection: gather type equations from function calls
- Implement unification: solve type equations to infer type arguments
- Implement substitution: replace type parameters with concrete types
- Add all new error codes (E100–E109)
- Checker tests for type inference, error cases

### Phase 3 — Monomorphisation (4–5 days)

- Build call graph to find all concrete instantiations of generic items
- Implement instantiation collection pass
- Implement AST cloning with type substitution
- Implement name mangling for monomorphised items
- Implement call site rewriting to use monomorphised names
- Handle recursive generics correctly
- Monomorphisation tests

### Phase 4 — Codegen (3–4 days)

- Update function codegen to handle monomorphised functions
- Update record codegen to handle monomorphised records
- Update enum codegen to handle monomorphised enums
- Update JSON schema generation for generic types
- Codegen snapshot tests for generic items

### Phase 5 — Stdlib Foundations (3–4 days)

- Implement `map`, `filter`, `find`, `reduce` in the stdlib
- Implement `unwrap_or`, `map_option` for `Option<T>`
- Implement `unwrap_or_else`, `map_result` for `Result<T, E>`
- Add `Pair<A, B>`, `Either<L, R>` to stdlib
- Integration tests for stdlib functions

### Phase 6 — Polish (2–3 days)

- Improve error messages for type inference failures
- Add documentation for generic syntax
- Update language guide with generics chapter
- Performance testing (compile time impact of monomorphisation)

**Total estimated effort:** ~3 weeks

---

## 17. Open Questions

### 17.1 Type constraints / bounds

This RFC introduces unconstrained generics. A future RFC should add type constraints:

```sage
// Future syntax — not in this RFC
fn contains<T: Eq>(list: List<T>, target: T) -> Bool {
    for item in list {
        if item == target {  // Allowed because T: Eq
            return true;
        }
    }
    return false;
}
```

The constraint `T: Eq` guarantees that `==` can be used on `T`. Similar constraints would be needed for `Ord` (comparison), `Hash` (for map keys), `Display` (for printing), and `Serialize` (for `Inferred<T>` and `@persistent`).

This is a significant design space (Rust traits vs Haskell typeclasses vs Go interfaces) and deserves its own RFC.

### 17.2 Higher-kinded types

This RFC does not support higher-kinded types:

```sage
// Not supported
fn map_over<F<_>, T, U>(container: F<T>, f: Fn(T) -> U) -> F<U>
```

Higher-kinded types enable more abstract programming patterns (functors, monads) but add significant complexity. Deferred indefinitely.

### 17.3 Associated types

This RFC does not support associated types:

```sage
// Not supported
trait Iterator {
    type Item;
    fn next(self) -> Option<Self::Item>;
}
```

Associated types are useful for collections where the element type is determined by the collection type. Deferred to the constraints RFC.

### 17.4 Default type parameters

This RFC does not support default type parameters:

```sage
// Not supported
record Response<T, E = String> { ... }
```

Default type parameters reduce boilerplate when one type argument is "usually" a certain type. Could be added in a follow-up RFC.

### 17.5 Variance

This RFC does not explicitly address variance (covariance, contravariance). In practice, monomorphisation sidesteps most variance concerns because each instantiation is a distinct type. If subtyping is added in a future RFC, variance will need explicit treatment.

### 17.6 Compile-time impact

Monomorphisation can slow down compilation for programs with many generic instantiations. The implementation should track instantiation count and warn if it exceeds a threshold (e.g., 1000 instantiations of a single generic function). This is unlikely to be a problem for typical Sage programs.

### 17.7 Interaction with `infer` and LLMs

When an LLM generates code in a Sage agent, can it use generics? The generated code would need to be type-checked, which means the LLM would need to understand the type parameter constraints. This is an advanced use case — for now, LLM-generated code should use concrete types.

---

*The types line up. The parameters are filled. Ward has been waiting for this since the first `List<T>` was written.*

