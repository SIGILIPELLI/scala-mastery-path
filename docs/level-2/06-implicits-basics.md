# 06 · Implicits Basics

"Implicits" is Scala's long-standing name for values and conversions the
compiler can supply automatically, without you writing them out at every
call site. Scala 3 gave the underlying mechanism clearer, dedicated
keywords — `given`/`using` for implicit values and parameters, `extension`
for adding methods to existing types — which is what this module teaches.
You'll still see the older `implicit val`/`implicit def`/`implicit class`
spellings in Scala 2 code and older library docs; they solve the same
problems.

## Implicit parameters: `given` and `using`

A `using` clause on a method marks a parameter the caller normally doesn't
pass explicitly — the compiler looks for a matching `given` value in scope
instead:

```scala
case class Config(prefix: String)

def logMessage(msg: String)(using config: Config): String =
  s"${config.prefix}$msg"

given defaultConfig: Config = Config("[app] ")

println(logMessage("started"))   // [app] started -- defaultConfig supplied automatically
```

Nothing at the call site mentions `Config` at all — the compiler found the
one `given Config` in scope and threaded it through. This is how a lot of
Scala library code passes cross-cutting context (an `ExecutionContext` for
`Future`, a JSON encoder, an `Ordering`) without every call needing to
repeat it.

## The trap: which `given` wins, and when it's decided

Implicit resolution is lexical scoping, same as any other name — but
`given`/`implicit` definitions behave like `def`s for ordering purposes, not
like `val`s: a local `given` is visible to code *earlier in the same block*,
not just code after it. This regularly surprises people coming from
ordinary variable semantics:

```scala
case class Config(prefix: String)
def logMessage(msg: String)(using config: Config): String = s"${config.prefix}$msg"

given topLevel: Config = Config("[top] ")

@main def run(): Unit =
  println(logMessage("first"))          // you might expect "[top] first"...
  given local: Config = Config("[local] ")
  println(logMessage("second"))

// Actual output:
// [local] first
// [local] second
```

Both calls use `local`, even though it's declared *after* the first
`println` — because within `run`'s body, `local` is in scope for the whole
block (like a method, not like a `val`, which would instead fail to compile
with "forward reference extends over definition" if you tried the same
trick with a plain `val`). The more specific, more local `given` also wins
over the top-level one whenever both are visible. The practical lesson:
don't assume implicit resolution respects the order code reads top-to-bottom
— it respects *scope*, and a local `given` shadows an outer one for the
entire enclosing block.

## Implicit conversions

A `given Conversion[A, B]` tells the compiler it may automatically convert
an `A` to a `B` wherever a `B` is expected. This is powerful and easy to
overuse — reach for it sparingly, since silent type conversions can make
code harder to reason about at a glance:

```scala
case class Meters(value: Double)
case class Feet(value: Double)

given Conversion[Meters, Feet] with
  def apply(m: Meters): Feet = Feet(m.value * 3.28084)

def needsFeet(f: Feet): String = s"${f.value} ft"

val m = Meters(10.0)
println(needsFeet(m))   // 32.8084 ft -- Meters silently converted to Feet
```

Prefer an explicit method (`m.toFeet`) for most conversions; reserve
implicit conversions for narrow, well-understood cases (unit wrappers,
adapting a third-party type to an interface you don't own) where the
conversion is unambiguous and safe every time it fires.

## Extension methods: adding methods to types you don't own

`extension` lets you add methods to an existing type — including types from
the standard library — without subclassing or wrapping it:

```scala
extension (s: String)
  def shout: String = s.toUpperCase + "!"
  def isPalindrome: Boolean = s == s.reverse

println("hello".shout)          // HELLO!
println("level".isPalindrome)   // true
println("scala".isPalindrome)   // false
```

An extension can also take its own parameters or even a by-name block,
which is how you'd build a small DSL-like helper:

```scala
extension (n: Int)
  def times(block: => Unit): Unit =
    var i = 0
    while i < n do
      block
      i += 1

3.times { print("hi ") }   // hi hi hi
println()
```

`3.times { ... }` reads like new syntax, but it's ordinary Scala: `3` is an
`Int`, `times` is an extension method on `Int` that happens to accept a
block. This is exactly the mechanism behind familiar-looking library
one-liners like `5.seconds` or `"text".isBlank`-style helpers you'll meet
in third-party code.

## Bringing it together: given-based dispatch

Combining `given`/`using` with a small trait gets you type-directed
behavior — different logic runs depending on the *type* involved, decided
at compile time by which `given` matches:

```scala
trait Show[A]:
  def show(a: A): String

given Show[Int] with
  def show(a: Int): String = s"Int($a)"

given Show[String] with
  def show(a: String): String = s"Str(\"$a\")"

def printIt[A](a: A)(using s: Show[A]): Unit =
  println(s.show(a))

printIt(42)        // Int(42)
printIt("hello")   // Str("hello")
```

This pattern — a trait describing a capability, plus one `given` instance
per type that has that capability — is called a **type class**, and it's
exactly how [Module 7 · Working with JSON](07-working-with-json.md)'s
encoders/decoders work under the hood. [Level 3](../level-3/06-type-classes.md)
covers type classes properly, including writing your own generic ones.

## Cheat sheet

| Scala 3 keyword | Old (Scala 2) name | Purpose |
|---|---|---|
| `given` (value) | `implicit val`/`implicit object` | Provide a value the compiler can inject |
| `using` (parameter) | `implicit` parameter | Mark a parameter to be filled in automatically |
| `given Conversion[A, B] with ...` | `implicit def` (A => B) | Automatic type conversion |
| `extension (x: T) def foo = ...` | `implicit class` | Add a method to an existing type |

## Exercise

Define `case class Distance(km: Double)`. Write an `extension` on `Distance`
adding `def miles: Double` (1 km ≈ 0.621371 miles). Separately, define a
`trait Describable[A]` with `def describe(a: A): String`, then write two
`given Describable[...]` instances — one for `Int` (e.g. `"the number N"`)
and one for your `Distance` (e.g. `"N km (M miles)"`, reusing your `miles`
extension) — and a generic `def report[A](a: A)(using d: Describable[A]):
Unit = println(d.describe(a))`. Call `report` with both an `Int` and a
`Distance` to confirm the right `given` is picked for each type.
