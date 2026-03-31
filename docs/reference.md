# Mu Language Syntax and Semantics Reference

*Version 0.1 (Draft)*

Mu is a general-purpose, multi-paradigm programming language with Python-like syntax and significant whitespace. It targets Python developers who need C-level performance without leaving the familiar syntax family. The language is expression-oriented where possible, supports gradual optimization through attributes, and provides explicit control over evaluation strategy, memory model, and error handling.

### Lexical Conventions

- **Indentation**: Significant whitespace (4 spaces recommended). Blocks are opened with a colon `:` and closed implicitly by dedent (or explicitly with optional `end`).
- **Inline vs Block Forms**:
  - `then` ::= inline expression form.
  - `:` ::= block form (statements end with newline or semicolon `;`).
- **end keyword**:
  - Optional for readability in multi-line blocks.
  - Required for one-line blocks that use semicolons: `if cond: stmt1; stmt2; end`.
- **Comments**: `#` to end-of-line (Python style).
- **String literals**: `"..."`, or interpolated `$"..."` (with `{expr}` inside).
- **Pipe operator**: `|>` (left-to-right function composition).

### Expressions vs Statements

Almost everything is an expression. Block forms (`:`) may contain multiple statements; the last expression becomes the value. Inline forms (`then`) are always single expressions.

### Control Flow & Branching Statements

All six branching constructs (`if`, `match`, `for`, `while`, `try`, `with`) follow the same pattern:

```mu
# Block form
keyword subject:
    body
    ...

# Inline expression form
keyword subject then expr
```

**if / else**

```mu
if x > 0 then x else -x

if x > 0:
    print("positive")
else:
    print("non-positive")
    end
```

**match / case / else** (exhaustive by default)

```mu
match e:
    case file.OpenError of { filename }:
        print($"Error opening {filename}")
    case file.ReadError of { filename }:
        print($"Error reading {filename}")
    else:
        print("Unknown error")
```

Inline version:

```mu
(match e                      # <-- no colon 
    case pattern then value   # linebreaks ingored within parantheses
    else fallback_value)
```

**for** (comprehensions or loops)

```mu
for x in list then x * 2          # <-- expression
for x in list:
    print(x)
```

**while**

```mu
while cond then do_something()
while cond:
    # ...
```

**try / except**

```mu
try divide(1, 0) except e then 0.0

try:
    risky()?
except e:
    print(e)
    end
```

**with** (resource management)

```mu
with f = file.open("file1")?, g = file.open("file2")?:
    f.read()? |> g.write()?
```

### Variable Declarations

Go-style syntax (no colon):

```mu
x int = 42
name str = "Mu"
y = 43 # type inference

# Multiple bindings (parentheses required for 2+)
two_cubed = let x = 2 then x * x * x
sum = let (x = 1, y = 2) then x + y
```

### Function Declarations (`fun`)

Two equivalent ways to declare parameters:

**Eager style** (traditional, all parameters required up-front):

```mu
fun add(a int, b int) -> int:
    return a + b
```

**Lazy / curried style** (parameters consumed one-by-one):

```mu
fun add int -> int -> int:
    param a
    param b
    return a + b
```

- `param name` binds the next argument and continues execution.
- Eager parameters must appear before lazy ones in mixed signatures.
- Both styles support the same call syntax thanks to **tuple concatenation**:
  - `(1)(2)(3) == (1, 2, 3)`
  - `add(1, 2) == add(1)(2)`
- Partial application is automatic: calling with fewer arguments returns a new function.

**Anonymous functions** use the same `fun` syntax inside expressions.

### Attributes & Definitions (`def`)

All top-level definitions are introduced with `def` followed by optional attributes, then the kind (`fun`, `struct`, `class`, `type`, etc.).

```mu
def fun add(a int, b int) -> int          # no attributes
def safe fun add(a int, b int) -> int     # single attribute
def (safe, pure, inline) fun add(a int, b int) -> int
def (collect(GC)) struct Node             # custom GC
```

**Available attributes** (non-exhaustive):

- `inline` - force inlining
- `rec` - recurcive function
- `pure` - no side effects, last expression is return value
- `safe` - Rust-style ownership / borrow checker
- `count` - reference counting
- `collect(GC)` - garbage collection (GC can be a reference to any collector)
- `comptime` - executed at compile time (used for metaprogramming and type generation)

Conflicts (e.g. `safe` + `collect`) are compile errors.

### Types

- **Basic syntax**: `varname type` (space-separated, Go style).
- **Function types**:
  - Eager: `(type, type) -> type`
  - Lazy/curried: `type -> type -> type`
- **Result / error types**: `type!` or `type!MyError`
- **Error sumtypes**: Automatically accumulated from every `?` in a function or `try` block.
- **Comptime type generation** (replaces traditional generics):

```mu
def comptime fun MyData(T type) -> type:
    struct Self:
        data T
    return Self

let my_data MyData(int) = { data = 42 }
```

### Error Handling

- `expr?` – propagate error (Rust/Swift `?` style).
- Errors are typed sumtypes; the compiler unions all possible errors from `?` sites.
- `try` / `except` can be expression or block and must unify return types (like `if`/`else`).
- Pattern matching on errors works with `match`, `if case`, or `let case`.

### Pattern Matching & Destructuring

- Full `match` / `case` / `else` (exhaustive unless `else` present).
- `if case Pattern = value:` (like Rust `if let`).
- `let case Pattern = value else fallback:`.
- Destructuring supports structs, tuples, variants, records, and wildcards (`_`).

### Memory Models & Pragmatism

Mu is **multi-pragmatic**: different functions, structs, or modules can use different memory strategies in the same program. Attributes control the model per definition. Boundary crossing between models follows FFI-like rules (automatic marshaling where possible, explicit escapes when needed).

### Additional Features

- **Pipelines**: `f.read()? |> g.write()?`
- **Interpolated strings**: `$"Error: {filename}"`
- **Comptime functions**: Full Mu language available at compile time for type construction, code generation, etc.
- **Tuple concatenation rule**: Powers clean currying and partial application.

### Summary of Design Philosophy

- **Readability first** – Python-like syntax with significant whitespace.
- **Performance on demand** – start with GC/reference counting, add `safe`/`inline`/`pure` where needed.
- **Explicit but ergonomic** – `?` for errors, attributes for memory, `then` vs `:` for evaluation style.
- **Unified concepts** – tuple concatenation, eager/lazy via type signature, comptime replacing generics.
- **Python-developer friendly** – gradual typing, familiar control flow, zero-FFI path from script to high-performance code.

This document captures the current state of the Mu design. The language is still evolving; future revisions will add modules, async/concurrency, operator overloading, virtual classes, and full Python interop.

Mu aims to be the language you wish Python compiled to—familiar syntax, C-level performance when you need it, and no second language or FFI layer required.
