# Mu Language Syntax and Semantics Reference

*Version 0.1 (Draft)*

Mu is a general-purpose, multi-paradigm programming language with Python-like syntax and significant whitespace. It targets Python developers who need C-level performance without leaving the familiar syntax family. The language is expression-oriented where possible, supports gradual optimization through attributes, and provides explicit control over evaluation strategy, memory model, and error handling.

---

## Lexical Conventions

- **Indentation**: Significant whitespace (4 spaces recommended). Blocks are opened with a colon `:` and closed implicitly by dedent, or explicitly with `end`.
- **Inline vs block forms**:
  - `then` — inline single-expression form.
  - `:` — block form (statements end with newline or semicolon `;`).
- **`end` keyword**:
  - Optional for readability in multi-line blocks.
  - Required to close multiple nested blocks at once: `if cond: stmt1; stmt2; end`.
- **Comments**: `#` to end-of-line.
- **String literals**: `"..."` plain, or `$"..."` for interpolation with `{expr}` inside.
- **Pipe operator**: `|>` for left-to-right function composition.

---

## Expressions vs Statements

Almost everything is an expression. Block forms (`:`) may contain multiple statements; the last expression is the value. Inline forms (`then`) are always a single expression.

---

## Variable Declarations

There are four declaration forms. Type annotations use `: type`; when omitted, the type is inferred from the value.

| Form | Type | Value | Mutability |
|---|---|---|---|
| `name: type` | explicit | deferred | immutable |
| `mu name: type` | explicit | deferred | mutable |
| `name = value` | inferred | immediate | immutable* |
| `mu name = value` | inferred | immediate | mutable |

*`name = value` is a mutation instead of a new binding if `name` is a `mu` variable already in scope (see Scoping below).

```mu
# Compile-time constant — cannot be shadowed in any scope
const Pi = 3.14159

# Immutable with explicit type, value assigned later
latebinding_num: int
latebinding_num = 5

# Immutable with explicit type and immediate value
immutable_num: int = 4

# Immutable with inferred type
greeting = "hello"

# Mutable with explicit type
mu mutable_num: int = 6
mutable_num = 7
mutable_num = 8

# Mutable with inferred type
mu count = 0
count = count + 1
```

`mu` variables are mutable in value but not in type — reassigning a different type is a compile error.

### `const` vs Immutable Bindings

`const` values are unshadowable: the same name resolves to the same value in every scope. Immutable bindings declared without `const` can be shadowed by a new binding of the same name in an inner scope.

### Scoping and `inherit`

Bare assignment (`name = value`) always creates a local binding unless `name` is a `mu` variable already in the current scope. To mutate a `mu` variable from an enclosing scope, declare it with `inherit`:

```mu
mu count = 0

def increment():
    inherit count       # explicitly captures the outer mu variable
    count = count + 1

increment()             # count is now 1
```

Without `inherit`, `count = count + 1` inside the function would create a new local binding and leave the outer `count` unchanged. Any function containing an `inherit` statement is immediately recognisable as one with side effects on external state.

---

## Functions (`def`)

Functions are defined with `def`. The return type follows `->`.

```mu
def my_func(a: int, b: int) -> int:
    return a + b
```

### Lazy / Curried Parameters (`param`)

A function can declare lazy parameters using `param` in the body. `param` works like `yield` in reverse: instead of producing a value, it consumes the next argument and suspends until it arrives. A function with lazy parameters returns a new function with one fewer parameter each time it is partially applied.

```mu
def my_lazy_func int -> int -> int:
    param a, b
    return a + b

my_lazy_func(1, 2)          # => 3  (called eagerly)
my_lazy_func(1)(2)          # => 3  (called via currying)
add_one = my_lazy_func(1)   # => function int -> int
```

The type signature `int -> int -> int` describes the curried form. Eager parameters (declared in the argument list) must appear before lazy ones. A zero-argument lazy function may be called without parentheses:

```mu
def my_getter_func int:     # takes no arguments, returns int
    return mutable_num

my_getter_func              # valid call
my_getter_func()            # also valid
```

### Tuple Concatenation

Adjacent argument lists are concatenated, so eager and curried calls are equivalent:

```
(1)(2)(3) == (1, 2, 3)
my_func(1)(2) == my_func(1, 2)
```

### Anonymous Functions

Anonymous functions use the same `def` syntax inside expressions:

```mu
double = def(x: int) -> int then x * 2
```

---

## Attributes (`@`)

Attributes annotate a definition with behavioural or optimisation hints. They are written with `@` above the definition or as a single attribute inline:

```mu
@pure
def add(a: int, b: int) -> int:
    return a + b

@(safe, inline)
def fast_add(a: int, b: int) -> int:
    return a + b
```

**Available attributes** (non-exhaustive):

- `inline` — force inlining.
- `rec` — recursive function (required for self-reference in some contexts).
- `pure` — no side effects; last expression is the return value.
- `safe` — Rust-style ownership and borrow checking.
- `count` — reference counting memory model.
- `collect(GC)` — garbage collection; `GC` may be a reference to any collector.
- `comptime` — executed at compile time (used for metaprogramming and type generation).
- `override` — marks a method as intentionally overriding a parent definition.

Conflicting attributes (e.g. `safe` + `collect`) are compile errors.

---

## Types

### Structs

Structs are plain data containers. They cannot extend other structs or classes.

```mu
def MyStruct struct:
    x: int
    y: int
```

Methods and trait implementations are added separately with `impl`:

```mu
impl MyStruct:
    def init(x: int, y: int) -> Self:
        return MyStruct { x=x, y=y }
```

### Enums

Enums define a closed set of variants. Variants may carry data or an inline struct.

```mu
def MyEnum enum:
    case First
    case Second(int)
    case Third struct:
        val: int
```

### Errors

Errors are similar to enums but used for error handling. See "Error handling" down below for my details.

```mu
def MyError error:
    case OutOfBounds
    case DivideByZero(int)
```

### Virtual Types (`virt`)

A `virt` is an abstract interface — a named contract with no data. It is equivalent to a trait or interface in other languages.

```mu
def MyVirtual virt:
    def speak(self) -> void
```

### Implementing Virtuals (`impl`)

`impl Type(Virt):` follows "type extends type" order, mirroring Python's `class Name(Super)` pattern.

```mu
impl MyStruct(MyVirtual):
    def speak(self):
        print($"I am a MyStruct {{ x={self.x}, y={self.y} }}")

impl MyEnum(MyVirtual):
    def speak(self):
        match self:
            case First:
                print("I am a MyEnum of First")
            case Second(x):
                print($"I am a MyEnum of Second({x})")
            case Third { val }:
                print($"I am a MyEnum of Third {{ val={val} }}")
```

### Classes

A `class` is shorthand for a struct with bundled method definitions. Unlike structs, classes can inherit from other classes and implement virtuals.

```mu
class MyClass(MyVirtual):
    val: int

    def init() -> Self:
        return MyClass { val = 0 }

    def speak(self):
        print($"I am a MyClass {{ val={self.val} }}")
```

### Inheritance and Visibility

All members of a class are public by default. When subclassing, inherited members become private to the subclass unless explicitly redeclared with `inherit`. This encourages separating public and private data into distinct types rather than using access modifiers.

```mu
class MyOtherClass(MyClass):
    inherit val     # redeclared — remains public in MyOtherClass
    other: int

    def init() -> Self:
        return MyOtherClass {
            val = 0,
            other = 1,
        }

    @override
    def speak(self):
        print($"I am a MyOtherClass {{ val={self.val}, other={self.other} }}")
```

A subclass cannot accidentally expose or clash with a private inherited member because types only see members that have been explicitly declared within them. This mirrors the convention used for imports.

---

## Type Annotations

- **Basic**: `name: type`
- **Function (eager)**: `(type, type) -> type`
- **Function (lazy/curried)**: `type -> type -> type`
- **Result / error**: `type!` or `type!MyError`
- **Inferred**: omit the annotation entirely

### Comptime Generics

Traditional generics are replaced by compile-time functions that produce types:

```mu
@comptime
def MyData(T: type) -> type:
    def Self struct:
        data: T
    return Self

my_data: MyData(int) = { data = 42 }
```

---

## Control Flow

All six branching constructs (`if`, `match`, `for`, `while`, `try`, `with`) share the same block / inline pattern:

```mu
# Block form
keyword subject:
    body

# Inline expression form
keyword subject then expr
```

### if / else

```mu
if x > 0 then x else -x

if x > 0:
    print("positive")
else:
    print("non-positive")
```

### match / case / else

Exhaustive by default; `else` provides a fallback.

```mu
match self:
    case First:
        print("First")
    case Second(x):
        print($"Second({x})")
    case Third { val }:
        print($"Third {{ val={val} }}")

# Inline form (no colon; line breaks ignored inside parentheses)
(match e
    case file.OpenError { filename } then $"Open error: {filename}"
    else "Unknown error")
```

### for

```mu
new_list = [for x in list then x * 2]

for x in list:
    print(x)
```

### while

```mu
while cond:
    body
```

### try / except

```mu
try divide(1, 0) except _ then 0.0

try:
    risky()?
except e:
    print(e)
```

### with

```mu
with f = file.open("a.txt")?, g = file.open("b.txt")?:
    f.read()? |> g.write()?
```

### Closing Multiple Blocks with `end`

```mu
try:
    if file1.endswith(".txt") and file2.endswith(".txt"):
        with f = file.open(file1)?, g = file.open(file2)?:
            f.read()? |> g.write()?
    end     # closes both `if` and `with`
    print("Success")
except e:
    print($"Error: {str(e)}")
```

---

## Error Handling

- `expr?` — propagate an error upward (Rust/Swift style).
- Errors are typed sum types; the compiler unions all possible error types from every `?` site in a function.
- `try` / `except` can be an expression or a block and must unify return types (like `if` / `else`).
- Pattern matching on errors works with `match`, `if Pattern of x = value:`, or `let Pattern of x = value else:`.

---

## Pattern Matching and Destructuring

- Full `match` / `case` / `else` (exhaustive unless `else` is present).
- `if case Pattern(x) = value:` — like Rust's `if let`.
- Destructuring supports structs, tuples, enum variants, and wildcards (`_`).

---

## Memory Models

Mu is multi-paradigm: different functions, structs, or modules can use different memory strategies in the same program. The model is controlled per-definition via attributes. Boundary crossing between models follows FFI-like rules — automatic marshalling where possible, explicit escapes otherwise.

---

## Additional Features

- **Pipelines**: `f.read()? |> g.write()?`
- **Interpolated strings**: `$"value is {expr}"`
- **Comptime functions**: the full Mu language is available at compile time for type construction, code generation, and metaprogramming.
- **Tuple concatenation**: powers clean currying and partial application — `f(1)(2) == f(1, 2)`.

---

## Design Philosophy

- **Readability first** — Python-like syntax with significant whitespace.
- **Performance on demand** — start with GC or reference counting; add `safe`, `inline`, or `pure` where needed.
- **Explicit but ergonomic** — `?` for errors, attributes for memory models, `then` vs `:` for evaluation style.
- **Unified concepts** — `inherit` for both scope capture and member visibility; `def` for all top-level definitions; tuple concatenation powering currying.
- **Python-developer friendly** — gradual typing, familiar control flow, no second language or FFI layer required.

---

*This document captures the current state of the Mu design. The language is still evolving; future revisions will cover modules, async/concurrency, operator overloading, and full Python interop.*
