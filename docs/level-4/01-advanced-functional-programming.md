# 01 · Advanced Functional Programming

Level 3 built type classes and monoids by hand (`Show`, `Monoid`,
`Functor`). **Cats** is the library that provides battle-tested versions of
these — plus `Monad`, `Validated`, and dozens more — so production code
doesn't reinvent them per project. This module introduces Cats' core
abstractions and `Validated`, the accumulating alternative to `Either`.

```scala title="build.sbt"
scalaVersion := "3.3.1"
libraryDependencies += "org.typelevel" %% "cats-core" % "2.10.0"
```

## Monad: the pattern behind `for`-comprehensions

A `Monad[F[_]]` is a `Functor` (module 04, Level 3) that also has
`flatMap` and `pure` (wrap a plain value). It's the exact shape `List`,
`Option`, `Either`, and `Future` all already have — Cats just names it and
provides one shared interface:

```scala
trait MyMonad[F[_]]:
  def pure[A](a: A): F[A]
  def flatMap[A, B](fa: F[A])(f: A => F[B]): F[B]

given MyMonad[Option] with
  def pure[A](a: A): Option[A] = Some(a)
  def flatMap[A, B](fa: Option[A])(f: A => Option[B]): Option[B] = fa.flatMap(f)

println(summon[MyMonad[Option]].flatMap(Some(5))(x => Some(x * 2)))   // Some(10)
```

Cats' real `cats.Monad[F[_]]` is this same shape, already instanced for
`Option`, `List`, `Either`, `Future`, and more — importing `cats.implicits._`
brings all of it (plus extension methods like `.mapN` below) into scope.

## `Either` chains stop at the first error — `Validated` doesn't

Level 2's `Either`-based validation (recall `makeUser` from the
Option/Either module) short-circuits: the first `Left` wins, and later
checks never even run. `cats.data.Validated` fixes exactly this — it
*accumulates* every failure instead of stopping at the first:

```scala
import cats.data.Validated
import cats.data.Validated.{Valid, Invalid}
import cats.implicits._

def validateName(name: String): Validated[List[String], String] =
  if name.isBlank then Invalid(List("name cannot be blank")) else Valid(name)

def validateAge(age: Int): Validated[List[String], Int] =
  if age < 0 then Invalid(List("age cannot be negative")) else Valid(age)

case class User(name: String, age: Int)

def makeUser(name: String, age: Int): Validated[List[String], User] =
  (validateName(name), validateAge(age)).mapN(User.apply)

println(makeUser("Ada", 30))   // Valid(User(Ada,30))
println(makeUser("", -5))      // Invalid(List(name cannot be blank, age cannot be negative))
```

`makeUser("", -5)` reports **both** problems — exactly the gap the
Level 2 module flagged as a known limitation of plain `Either`. The trick is
`.mapN`: it doesn't chain like `flatMap` (which requires each step to know
the previous succeeded); it runs every validation independently first, then
combines the results, collecting failures from a `List` (any type with a
`Semigroup` — module 04's monoid idea, minus the `empty` requirement) along
the way.

### The trap: `Validated` isn't a `Monad`, and that's the point

`Validated` deliberately has no `flatMap` in the way `Either` does — if it
did, step 2 would depend on step 1 succeeding, which is exactly the
short-circuiting behavior `Validated` exists to avoid. Reach for `Either`
(or its Cats-flavored `EitherT`) when later steps genuinely need earlier
results and stopping early on failure is correct; reach for `Validated`
when checks are independent and you want every error reported at once —
the classic case being form/request validation.

## `mapN`: combining independent effects

`.mapN` works on any tuple of the same effect type — not just `Validated`:

```scala
val opt = (Option(1), Option(2)).mapN(_ + _)
println(opt)   // Some(3)
```

If either `Option` were `None`, the whole result is `None` — `mapN`
combines results only when every input succeeded, same spirit as
`Future.sequence` from Level 3 but for a fixed, small number of
heterogeneous inputs rather than a list of the same type.

## Cats Effect: `IO` as a purer `Future`

`cats-effect`'s `IO[A]` describes an effectful computation without running
it immediately — unlike `Future`, which starts running the moment it's
created, an `IO` is inert until something runs it (typically `IOApp`'s
`run` at the very edge of your program). This separation ("describe the
computation" vs. "run the computation") is what makes `IO`-based code
easier to test and reason about — you can build up an entire program as a
value and only decide *when* it executes at one point:

```scala
import cats.effect.{IO, IOApp}

object Main extends IOApp.Simple:
  def run: IO[Unit] =
    for
      _ <- IO.println("starting")
      a <- IO(21)
      b <- IO(21)
      _ <- IO.println(s"sum: ${a + b}")
    yield ()
```

```text
starting
sum: 42
```

## Cheat sheet

| Concept | What it gives you |
|---|---|
| `cats.Monad[F[_]]` | Shared `pure`/`flatMap` interface across `Option`/`List`/`Either`/`Future`/etc. |
| `Validated[E, A]` | `Either`-like, but accumulates every error instead of stopping at the first |
| `.mapN` | Combine several independent effects (`Validated`, `Option`, ...) into one result |
| `IO[A]` | A description of an effect, run explicitly (usually via `IOApp`), unlike eager `Future` |
| Reach for `Either`/`EitherT` | When step *N* genuinely needs step *N-1*'s result |
| Reach for `Validated` | When checks are independent and you want every failure reported |

## Exercise

Extend `makeUser` with a third check, `validateEmail(email: String):
Validated[List[String], String]` (require it contains `"@"`), and combine
all three with `.mapN` into a `User(name, age, email)`. Confirm passing all
three invalid values reports all three error messages together. Then write
a small `IO`-based program using `IOApp.Simple` that reads two `IO[Int]`
values (simulate with `IO(42)` and `IO(8)`), computes their `.mapN(_ + _)`
using `cats.implicits._`, and prints the result — note `IO` also supports
`.mapN` the same way `Option` and `Validated` do, because it's a `Monad`
too.
