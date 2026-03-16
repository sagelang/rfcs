# RFC-0019: Syntax Highlighting Improvements

- **Status:** Draft
- **Created:** 2026-03-16
- **Author:** Pete Pavlovski
- **Affects:** tree-sitter-sage, sage-vscode, sage-zed

---

## Summary

Overhaul Sage's syntax highlighting to fix bugs, expand the color palette, and align with industry best practices. This includes grammar fixes for missing constructs and comprehensive highlight queries.

---

## Motivation

Current syntax highlighting has several problems:

### Bugs

1. **`const` declarations aren't parsed** — The grammar has no `const_declaration` rule, causing strings in constants to not highlight and keywords inside strings (like `and`) to incorrectly highlight as keywords
2. **`spawn` not highlighted** — Missing from grammar as a distinct construct

### Limited palette

The current highlighting uses only three colors:
- Purple: keywords
- Green: strings
- Grey: everything else

This makes code visually flat and hard to scan. Types, functions, constants, and variables all blend together.

### Missing semantic categories

- Built-in types (`String`, `Int`, `Bool`, etc.) not distinguished
- Function calls not highlighted
- `self` not highlighted as a keyword
- String interpolation expressions not distinguished from string content
- Constant names not highlighted differently from variables

---

## Design

### Semantic token categories

Following industry standards (VS Code, JetBrains, tree-sitter conventions), Sage should use these highlight groups:

| Category | Scope | Example | Suggested Color |
|----------|-------|---------|-----------------|
| `@keyword` | Language keywords | `fn`, `let`, `agent`, `if` | Purple/Magenta |
| `@keyword.control` | Control flow | `if`, `else`, `for`, `match`, `return` | Purple/Magenta |
| `@keyword.operator` | Operator keywords | `and`, `or`, `try` | Purple/Magenta |
| `@type` | Type names | `String`, `Int`, `MyRecord` | Cyan/Teal |
| `@type.builtin` | Built-in types | `String`, `Int`, `Bool`, `Float` | Cyan/Teal (bold) |
| `@function` | Function definitions | `fn foo()` | Blue |
| `@function.call` | Function calls | `foo()` | Blue |
| `@function.builtin` | Built-in functions | `divine`, `print`, `len` | Blue (italic) |
| `@variable` | Variables | `let x` | White/Default |
| `@variable.parameter` | Parameters | `fn foo(x: Int)` | Orange/Peach |
| `@variable.builtin` | Built-in variables | `self` | Orange (italic) |
| `@constant` | Constants | `const FOO` | Orange/Gold |
| `@constant.builtin` | Built-in constants | `true`, `false`, `none` | Orange/Gold |
| `@string` | String literals | `"hello"` | Green |
| `@string.escape` | Escape sequences | `\n`, `\"` | Cyan |
| `@string.special` | Interpolation delimiters | `{` `}` in `"{x}"` | Cyan |
| `@number` | Numeric literals | `42`, `3.14` | Light Orange/Peach |
| `@operator` | Operators | `+`, `-`, `==` | White/Default |
| `@punctuation.bracket` | Brackets | `(`, `)`, `{`, `}` | White/Default |
| `@punctuation.delimiter` | Delimiters | `,`, `:`, `;` | White/Default |
| `@comment` | Comments | `// ...` | Grey (italic) |
| `@property` | Fields/properties | `record.field` | Light Blue |
| `@label` | Agent/type names in declarations | `agent Foo` | Yellow/Gold |

### Color palette

A modern, readable palette with good contrast:

```
Keywords:        #C678DD (purple/magenta)
Types:           #56B6C2 (cyan)
Functions:       #61AFEF (blue)
Strings:         #98C379 (green)
Numbers:         #D19A66 (orange)
Constants:       #E5C07B (gold)
Parameters:      #E06C75 (coral/red)
Properties:      #ABB2BF (light grey)
Comments:        #5C6370 (dark grey, italic)
Operators:       #ABB2BF (light grey)
Escape/Special:  #56B6C2 (cyan)
```

This is based on the One Dark theme, which is widely regarded as readable and aesthetically pleasing.

---

## Grammar changes

### Add `const_declaration`

```javascript
// In grammar.js _definition choice:
$.const_declaration,

// New rule:
const_declaration: $ => seq(
  optional('pub'),
  'const',
  field('name', $.identifier),
  ':',
  field('type', $._type),
  '=',
  $._expression,
  optional(';'),
),
```

### Add `spawn_expression`

```javascript
// In _expression choice:
$.spawn_expression,

// New rule:
spawn_expression: $ => seq(
  'spawn',
  $.identifier,
  optional($.record_body),
),
```

### Fix string parsing for `and`/`or`

Ensure the word boundary for keywords doesn't match inside strings. The current grammar should handle this, but the `const` bug causes cascading failures.

---

## Highlight query changes

Update `queries/highlights.scm`:

```scheme
; Keywords
"const" @keyword
"spawn" @keyword
"self" @variable.builtin

; Control flow (more specific)
["if" "else" "for" "while" "match" "return" "break" "continue"] @keyword.control
["try" "await"] @keyword.control

; Operators as keywords
["and" "or"] @keyword.operator

; Built-in types
((identifier) @type.builtin
 (#match? @type.builtin "^(String|Int|Float|Bool|Unit|List|Map|Option|Result|Oracle)$"))

; Constant declarations
(const_declaration
  name: (identifier) @constant)

; Function calls
(call_expression
  function: (identifier) @function.call)

; Method calls
(method_call_expression
  method: (identifier) @function.call)

; Built-in functions
((identifier) @function.builtin
 (#match? @function.builtin "^(divine|print|len|push|pop|keys|values|some|none|ok|err)$"))

; Self
((identifier) @variable.builtin
 (#eq? @variable.builtin "self"))

; String interpolation
(interpolated_string
  "{" @string.special
  "}" @string.special)

; Spawn expression
(spawn_expression
  "spawn" @keyword
  (identifier) @type)
```

---

## Affected files

| Repository | File | Changes |
|------------|------|---------|
| tree-sitter-sage | `grammar.js` | Add `const_declaration`, `spawn_expression` |
| tree-sitter-sage | `queries/highlights.scm` | Comprehensive highlight queries |
| sage-zed | `languages/sage/highlights.scm` | Copy of tree-sitter queries |
| sage-vscode | `syntaxes/sage.tmLanguage.json` | Update TextMate grammar to match |

---

## Implementation

1. Update `grammar.js` with missing rules (`const`, `spawn`)
2. Regenerate tree-sitter parser (`tree-sitter generate`)
3. Update `highlights.scm` with comprehensive queries
4. Test in Zed and VS Code with sample files
5. Copy highlights to sage-zed extension
6. Update VS Code TextMate grammar to match semantics
7. Consider shipping a recommended theme or theme snippet

---

## Testing

Validate with a test file covering all constructs:

```sage
const TOPIC: String = "test with and keyword inside";
const COUNT: Int = 42;

record User {
    name: String,
    age: Int,
}

agent Greeter {
    belief greeting: String = "Hello"

    on start {
        let user = User { name: "Alice", age: 30 };
        let msg = divine("{self.greeting}, {user.name}!");
        emit(msg);
    }
}

fn greet(name: String) -> String {
    if name == "World" {
        return "Hello, World!";
    }
    return "Hi, " ++ name;
}

run spawn Greeter {}
```

All elements should highlight distinctly and correctly.

---

## Open Questions

1. **Theme shipping** — Should we ship a recommended Sage theme, or rely on editor defaults?
2. **Semantic tokens** — Should sage-sense (LSP) provide semantic tokens for even richer highlighting?
