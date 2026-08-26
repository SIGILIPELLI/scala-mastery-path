# 04 · Functional Programming Patterns

You've been using `map`, `flatMap`, and `for`-comprehensions since Level 1
on `List`, `Option`, and `Either`. This module names the pattern behind why
all three support the same operations — and introduces the two most common
functional abstractions you'll meet in real Scala code: **monoids** and
**functors**.

## Monoid: "combinable, with an identity"

A monoid is any type with a way to combine two values into one, plus an
"empty" value that doesn't change anything when combined. `Int` addition
(`combine = +`, `empty = 0`), `String` concatenation (`combine = +`, `empty
= ""`), and `List` concatenation (`combine = ++`, `empty = Nil`) are all
monoids — the same shape shows up everywhere once you look for it.

```scala
trait Monoid[A]:
  def combine(x: A, y: A): A
  def empty: A

object Monoid:
  given intAdd: Monoid[Int] with
    def combine(x: Int, y: Int): Int = x + y
    def empty: Int = 0

  given stringConcat: Monoid[String] with
    def combine(x: String, y: String): String = x + y
    def empty: String = ""

  def combineAll[A](xs: List[A])(using m: Monoid[A]): A =
    xs.foldLeft(m.empty)(m.combine)

println(Monoid.combineAll(List(1, 2, 3, 4)))     // 10
println(Monoid.combineAll(List("a", "b", "c")))  // abc
```

`combineAll` is written once and works for *any* type with a `Monoid`
instance — the `using` parameter (Level 2's implicits, formalized) supplies
the right `combine`/`empty` pair for whatever `A` is at the call site. This
is the same trick behind `.sum` on numeric collections in the standard
library.

### The trap: `empty` must be a true identity

A `Monoid` instance is only lawful if `combine(empty, x) == x` and
`combine(x, empty) == x` for every `x`. A "monoid" for integer
multiplication with `empty = 0` would be wrong — `combine(0, x)` is always
`0`, not `x`. The identity for multiplication is `1`. Getting this wrong
doesn't cause a compile error; it silently breaks any code that assumes the
law holds (like folding an empty list and expecting a no-op result).

## Functor: "mappable"

A functor is any type constructor `F[_]` with a `map` that transforms the
value(s) inside without changing the "shape" — `List[A] => List[B]` stays a
list of the same length, `Option[A] => Option[B]` stays `Some`/`None` the
same way, a `Future[A] => Future[B]` is still exactly one eventual value.

```scala
trait Functor[F[_]]:
  def map[A, B](fa: F[A])(f: A => B): F[B]

given Functor[List] with
  def map[A, B](fa: List[A])(f: A => B): List[B] = fa.map(f)

given Functor[Option] with
  def map[A, B](fa: Option[A])(f: A => B): Option[B] = fa.map(f)

def double[F[_]: Functor](fa: F[Int]): F[Int] =
  summon[Functor[F]].map(fa)(_ * 2)

println(double(List(1, 2, 3)))               // List(2, 4, 6)
println(double(Some(5): Option[Int]))        // Some(10)
```

`double` doesn't know or care whether it's working on a `List` or an
`Option` — it only relies on the `Functor` contract, i.e. "there's a `map`."
That `F[_]: Functor` syntax is a *context bound*: shorthand for `F[_]`
plus an implicit `Functor[F]` parameter, same mechanism as `using`.

### The trap: not every `F[_]` is a functor

A type constructor only qualifies as a functor if `map` satisfies its own
laws — `map(fa)(identity) == fa`, and mapping with `f andThen g` gives the
same result as mapping with `f` then mapping with `g`. `Set[A]` looks like
it should be a functor (it has `.map`), but if `f` isn't injective (e.g.
`(_: Int) % 2`), the resulting set can have *fewer* elements than the
original — the "shape" (size) isn't preserved, which many algorithms
written generically over `Functor` silently assume.

## Why the standard library already looks like this

Scala's own `for`-comprehensions desugar to `map`/`flatMap`/`withFilter`
calls precisely because so many types — `List`, `Option`, `Either`,
`Future`, and any custom type you define with those three methods — share
this functor/monad shape. Recognizing "this is just a functor" or "this is
just a monoid" is what lets you write one generic function (`combineAll`,
`double`) instead of one per concrete type, which is the entire appeal of
libraries like Cats that formalize dozens of these patterns (covered in
Level 4).

## Cheat sheet

| Pattern | Shape | Standard-library examples |
|---|---|---|
| Monoid | `combine(A, A): A` + `empty: A`, with `combine(empty, x) == x` | `Int` (+/0), `String` (++/`""`), `List` (++/`Nil`) |
| Functor | `map[A, B](F[A])(A => B): F[B]`, preserving shape | `List`, `Option`, `Either[E, _]`, `Future` |
| Generic code over either | `given`/`using` instances + a type parameter bound like `F[_]: Functor` | `combineAll`, `double` above |

## Exercise

Define a `Monoid[List[Int]]` instance where `combine` is `++` and `empty`
is `Nil`, then use `Monoid.combineAll` on a `List[List[Int]]` to flatten it
into one list. Separately, write a `Semigroup`-flavored `Monoid[Int]` for
`max` (combine is `math.max`, and pick a sensible `empty` — think about
what value never changes a max) and use it via `combineAll` on a list of
scores. Finally, add a third `given Functor[F]` instance for `Either[String,
_]` and confirm `double` works on `Right(21): Either[String, Int]` but
returns the `Left` unchanged when given `Left("bad")`.
