# 04 · Option/Either for Error Handling

You've already seen `Option` in passing — the
[Level 1 project](../level-1/10-project-todo-app.md) used it to represent a
line that might fail to parse. This module treats it properly, alongside
`Either` and `Try`: three types that let you model "this might not work" as
*data* you can transform and pattern match on, instead of relying on
exceptions or `null`.

## `Option`: modeling absence

`Option[A]` is either `Some(value)` or `None` — nothing else, and no `null`
in sight. It's the standard replacement for "this might not have a value."

```scala
def parseInt(s: String): Option[Int] = s.toIntOption

println(parseInt("42"))     // Some(42)
println(parseInt("nope"))   // None
```

`Option` supports the same transformation methods as collections, because
it's really "a list of at most one element":

```scala
println(parseInt("42").map(_ * 2))     // Some(84)
println(parseInt("nope").map(_ * 2))   // None -- map on None is a no-op
```

### The trap: `.get` overuse

`Option` has a `.get` method that returns the value directly — and throws
`NoSuchElementException` if called on `None`. Reaching for `.get` defeats
the entire point of `Option`, reintroducing exactly the runtime crash it
exists to prevent:

```scala
val maybe: Option[Int] = None
try
  maybe.get
catch
  case e: NoSuchElementException => println(s"caught: ${e.getMessage}")
  // caught: None.get
```

Use `.getOrElse(default)` for a safe fallback, `.map`/`.flatMap` to
transform without unwrapping, or pattern matching / a `for` loop to act only
when a value is present:

```scala
println(maybe.getOrElse(-1))   // -1

def half(n: Int): Option[Int] =
  if n % 2 == 0 then Some(n / 2) else None

println(parseInt("6").flatMap(half))   // Some(3)
println(parseInt("7").flatMap(half))   // None -- 7 isn't even, half never runs

for n <- parseInt("10") do println(s"got $n")   // got 10 -- runs only if Some
```

`flatMap` is essential here: `parseInt("6").map(half)` would give you
`Some(Some(3))` — a nested `Option` — because `half` itself returns an
`Option`. `flatMap` flattens that one extra layer, exactly like it does for
`List[List[A]]`.

## `Either`: modeling failure with a reason

`Option` tells you *whether* something worked, but not *why* it failed.
`Either[L, R]` fixes that: by convention, `Left` carries the failure/error
and `Right` carries the success value ("Right is right," i.e. correct).

```scala
def safeDivide(a: Int, b: Int): Either[String, Int] =
  if b == 0 then Left("division by zero") else Right(a / b)

println(safeDivide(10, 2))   // Right(5)
println(safeDivide(10, 0))   // Left(division by zero)
```

`Either` is right-biased: `map`, `flatMap`, and `for`-comprehensions all
operate on the `Right` case and pass a `Left` straight through untouched —
which is exactly the "stop on first error" behavior you usually want.

```scala
println(safeDivide(10, 0).map(_ * 100))   // Left(division by zero) -- map skipped
println(safeDivide(10, 0).left.map(_.toUpperCase))   // Left(DIVISION BY ZERO)

safeDivide(10, 0) match
  case Right(v)  => println(s"ok: $v")
  case Left(err) => println(s"error: $err")
// error: division by zero
```

### Chaining validations with `Either` and `for`

This is where `Either` earns its keep — validating several fields and
stopping at the first failure, with a message that says what went wrong:

```scala
case class User(name: String, age: Int)

def validateName(name: String): Either[String, String] =
  if name.isBlank then Left("name cannot be blank") else Right(name)

def validateAge(age: Int): Either[String, Int] =
  if age < 0 then Left("age cannot be negative")
  else if age > 150 then Left("age unrealistically high")
  else Right(age)

def makeUser(name: String, age: Int): Either[String, User] =
  for
    validName <- validateName(name)
    validAge  <- validateAge(age)
  yield User(validName, validAge)

println(makeUser("Ada", 30))   // Right(User(Ada,30))
println(makeUser("", 30))      // Left(name cannot be blank)
println(makeUser("Bob", -5))   // Left(age cannot be negative)
```

Note that `makeUser("", -5)` would still only report the *first* problem
(the blank name) — a plain `Either` chain short-circuits at the first
`Left` rather than collecting every error. If you need "report all the
problems at once," that's what a validation-accumulating type
(`cats.data.Validated` and similar, outside the standard library) is for —
worth knowing the limitation exists even though it's beyond this course.

## `Try`: wrapping code that throws

Some code — especially calls into Java libraries — throws exceptions rather
than returning `Option`/`Either`. `Try` wraps a computation and turns a
thrown exception into a `Failure` value instead of letting it propagate:

```scala
import scala.util.{Try, Success, Failure}

def riskyParse(s: String): Try[Int] = Try(s.toInt)   // s.toInt throws on bad input

riskyParse("42") match
  case Success(v) => println(s"parsed: $v")            // parsed: 42
  case Failure(e) => println(s"failed: ${e.getClass.getSimpleName}")

riskyParse("nope") match
  case Success(v) => println(s"parsed: $v")
  case Failure(e) => println(s"failed: ${e.getClass.getSimpleName}")
  // failed: NumberFormatException
```

`Try` converts to the other two when you need to: `.toOption` collapses
`Failure` to `None` (discarding the exception detail), and `.toEither` keeps
the exception as the `Left` value:

```scala
println(riskyParse("nope").toOption)   // None
println(riskyParse("nope").toEither)
// Left(java.lang.NumberFormatException: For input string: "nope")
```

## Cleaning up a collection of `Option`s

A common pattern: you've mapped a parsing function across a list and ended
up with a `List[Option[A]]`. `.flatten` drops every `None` and unwraps every
`Some` in one call:

```scala
val opts = List(Some(1), None, Some(3))
println(opts.flatten)       // List(1, 3)
println(opts.flatten.sum)   // 4
```

## Cheat sheet

| Type | Success case | Failure case | Carries a reason? |
|---|---|---|---|
| `Option[A]` | `Some(value)` | `None` | no |
| `Either[L, R]` | `Right(value)` | `Left(error)` | yes |
| `Try[A]` | `Success(value)` | `Failure(exception)` | yes (an exception) |

| Need to... | Use |
|---|---|
| Provide a fallback value | `.getOrElse(default)` |
| Transform only if present/successful | `.map(f)` |
| Chain calls that themselves return `Option`/`Either` | `.flatMap(f)` or a `for`-comprehension |
| Convert between the three | `.toOption`, `.toEither` |
| Drop `None`s from a `List[Option[A]]` | `.flatten` |
| Run code that might throw, capture the exception | `Try(...)` |

## Exercise

Write `def parsePositiveEven(s: String): Either[String, Int]` that parses a
string to an `Int` (failing with `"not a number"` if it doesn't parse),
then checks it's positive (`"must be positive"`), then checks it's even
(`"must be even"`) — chain these with a `for`-comprehension over `Either`,
short-circuiting at the first failure. Test it against `"8"`, `"-4"`,
`"7"`, and `"abc"`, printing each result. Then write a second function that
takes a `List[String]` and returns only the values that parse successfully,
using `.flatMap` with your Level-1-style `Option`-returning helper or by
converting your `Either` results with `.toOption` and `.flatten`.
