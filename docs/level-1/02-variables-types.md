# 02 · Variables & Types

## `val` vs `var`

Scala has two ways to declare a variable, and the difference matters a lot
more than it does in most languages:

```scala
val name = "Ada"        // immutable -- cannot be reassigned
var count = 0           // mutable -- can be reassigned

count = count + 1       // fine
// name = "Grace"       // compile error: reassignment to val
```

```scala
val x: Int = 10
// x = 20               // error: cannot assign to a val
println(x)              // 10
```

Idiomatic Scala strongly favors `val`. Immutability makes code easier to
reason about — a `val` bound to a value never changes underneath you, which
is especially important once you start passing data between functions and
threads (see [Level 3](../level-3/01-futures-concurrency.md)). Reach for
`var` only when you have a specific, local reason (like a loop counter you
genuinely need to mutate) — and even then, prefer a functional alternative
first (see [Module 3](03-control-flow.md)).

## Type inference

Scala infers types, so you rarely need to write them out — but you always
can, and it's often good practice for public APIs:

```scala
val age = 30              // inferred as Int
val price = 19.99         // inferred as Double
val label = "widget"      // inferred as String
val active = true         // inferred as Boolean

// Explicit types (same values, spelled out)
val ageExplicit: Int = 30
val priceExplicit: Double = 19.99
```

## Primitive-ish types

Scala's numeric and boolean types map closely to the JVM's primitives, but
they're full objects with methods you can call directly:

```scala
val i: Int = 42
val l: Long = 42L
val d: Double = 3.14
val f: Float = 3.14f
val b: Byte = 127
val s: Short = 1000
val bool: Boolean = true
val ch: Char = 'A'

println(i.toDouble)     // 42.0
println(d.toInt)        // 3 (truncates, doesn't round)
println((-5).abs)        // 5 -- calling a method directly on a literal
```

## Strings

```scala
val first = "Ada"
val last = "Lovelace"

// Concatenation
val full = first + " " + last
println(full)                          // Ada Lovelace

// String interpolation -- the idiomatic way
val greeting = s"Hello, $first $last!"
println(greeting)                      // Hello, Ada Lovelace!

// Interpolation with expressions
val a = 3
val b = 4
println(s"${a} + ${b} = ${a + b}")     // 3 + 4 = 7

// Multi-line strings
val poem =
  """Roses are red,
    |Violets are blue.""".stripMargin
println(poem)
```

There are three string interpolators: `s"..."` for basic interpolation,
`f"..."` for `printf`-style formatting, and `raw"..."` for strings where
escape sequences shouldn't be processed.

```scala
val pi = 3.14159
println(f"Pi is about $pi%.2f")        // Pi is about 3.14
```

## Type aliases and `Any`/`Nothing`

Scala's type hierarchy has `Any` at the top (every type is a subtype of it)
and `Nothing` at the bottom (no values exist of this type — it's used for
things like "this expression never returns," e.g. throwing an exception).

```scala
val anything: Any = 42
val alsoAnything: Any = "a string"

def fail(msg: String): Nothing =
  throw new RuntimeException(msg)
```

You won't reach for `Any` or `Nothing` often as a beginner, but recognizing
them in error messages or library signatures will save you confusion later.

## Cheat sheet

| Type | Example | Notes |
|------|---------|-------|
| `Int` | `42` | 32-bit signed integer, most common numeric type |
| `Long` | `42L` | 64-bit integer |
| `Double` | `3.14` | 64-bit floating point, default for decimals |
| `Float` | `3.14f` | 32-bit floating point |
| `Boolean` | `true` / `false` | |
| `Char` | `'A'` | single character, single quotes |
| `String` | `"hello"` | double quotes |
| `Unit` | — | "no meaningful value," like `void` |

## How It Actually Works

`val` and `var` are a source-level distinction only — the JVM bytecode
doesn't have an "immutable local variable" concept for method-local values;
a `val` inside a method just becomes a bytecode local slot that the compiler
guarantees (via type checking, not runtime enforcement) is only ever
stored to once. The safety is compile-time, not a runtime lock. Where it
*does* show up in bytecode is fields: a `val` member of a class compiles to
a `private final` field plus a public getter method, while a `var` member
compiles to a plain `private` field with both a getter and a setter — the
`final` modifier is the JVM-level guarantee the compiler is relying on.

`Int`, `Double`, `Boolean`, etc. look like objects (you can call `.abs` or
`.toDouble` on a literal), but the compiler erases that appearance whenever
it can: in most contexts an `Int` is compiled straight down to the JVM's
primitive `int`, and `i.toDouble` becomes a single `i2d` bytecode
instruction, not a method call on a heap object. Scala only "boxes" a
primitive into an actual object (`java.lang.Integer`, etc.) when it has to
— e.g. storing it in a generic collection like `List[Int]` before Scala's
specialization kicks in, or when code demands `Any`. This is why `Any` sits
at the top of the type hierarchy above both objects and value types like
`Int`: on the JVM, `Any` unifies things that are fundamentally different at
the bytecode level (object references vs. raw primitives), and the compiler
inserts the boxing/unboxing conversions for you wherever the two need to
mix.

String interpolation (`s"..."`) is a compile-time macro-like rewrite, not
string concatenation with extra syntax: `s"${a} + ${b} = ${a + b}"` is
rewritten by the compiler into a call to `StringContext(...).s(a, b, a + b)`,
which under the hood builds the result with a `StringBuilder` — the same
mechanism you'd get from manual `+` concatenation, just generated for you.

## Exercise

Declare a `val` for your name, a `val` for your birth year, and a `var` for a
running total starting at `0`. Use string interpolation to print a sentence
like `"Ada was born in 1815 and the running total is 0."` Then increment the
running total three times by different amounts and print it again — but try
to also write a version using only `val`s (hint: create a new `val` each
time instead of mutating).
