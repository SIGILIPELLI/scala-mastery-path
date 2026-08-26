# 06 · Type Classes

`Monoid` and `Functor` from the previous module are both examples of a
broader design pattern called a **type class**: define a trait describing a
capability, provide `given` instances for the types that have it, and write
functions generic over "any type with this capability" — without touching
the original type's definition at all. This module names the pattern
explicitly and works through building one from scratch.

## The problem type classes solve

Suppose you want a `show` operation — a custom, controllable
`toString`-like string representation — for `Int`, `Boolean`, and your own
`Point` class. You can't add methods to `Int` or `Boolean` (you don't own
them), and even for `Point`, baking `show` in as a member method means
every type needing it must be modified individually, with no shared
contract a generic function could rely on.

## Defining a type class

A type class is just a trait parameterized by the type it describes:

```scala
trait Show[A]:
  def show(a: A): String

object Show:
  def apply[A](using s: Show[A]): Show[A] = s   // summon helper

  given Show[Int] with
    def show(a: Int): String = a.toString

  given Show[Boolean] with
    def show(a: Boolean): String = if a then "yes" else "no"
```

`Show.apply` (often called a "summoner") lets you write `Show[Int]` to pull
the `given Show[Int]` instance out of implicit scope by name — useful inside
generic functions.

## Writing generic code against the type class

```scala
def printAll[A: Show](xs: List[A]): Unit =
  xs.foreach(x => println(Show[A].show(x)))

printAll(List(1, 2, 3))
// 1
// 2
// 3

printAll(List(true, false))
// yes
// no
```

`[A: Show]` is a **context bound** — sugar for `[A](using Show[A])`.
`printAll` never mentions `Int` or `Boolean`; it only needs *some* `Show[A]`
to exist, supplied automatically by the compiler at each call site.

## Adding an instance for your own type — no inheritance needed

This is the pattern's real payoff: giving `Point` a `Show` instance doesn't
require `Point` to extend anything or know `Show` exists:

```scala
case class Point(x: Int, y: Int)

given Show[Point] with
  def show(p: Point): String = s"(${p.x}, ${p.y})"

printAll(List(Point(0, 0), Point(3, 4)))
// (0, 0)
// (3, 4)
```

Compare this to putting `show` on `Point` directly (intrusive, and
impossible for types you don't control like `Int`) or to an
overloaded-methods approach (`show(x: Int)`, `show(x: Boolean)`, ... — no
shared abstraction a generic function like `printAll` could target).

## Extension methods for a nicer call site

`Show[A].show(x)` works but reads backwards. An `extension` combined with a
`using` clause gives instances a natural `x.show` syntax:

```scala
extension [A](a: A)(using s: Show[A])
  def show: String = s.show(a)

println(Point(1, 2).show)   // (1, 2)
```

Now any type with a `given Show[_]` instance gets `.show` for free — this
is exactly how the standard library gives you `xs.sorted` (via
`Ordering[A]`) and `xs.sum` (via `Numeric[A]`) on generic collections.

### The trap: ambiguous or missing instances

If two `given Show[Int]` instances are in scope at once, the compiler
reports an ambiguity error at the *call site* using them — not where the
duplicate was defined — which can be confusing in a large codebase with
instances scattered across files. And if no instance exists for a type you
call `printAll` on, you get a compile error ("no given instance of type
`Show[X]` was found") rather than a runtime failure — this is a **feature**
of type classes (missing behavior is caught at compile time), but the error
message is easy to misread as "context bound syntax is broken" when the
real issue is simply a missing `given`.

## Cheat sheet

| Need to... | Use |
|---|---|
| Define a capability as a type class | `trait TypeClass[A]` with the operations |
| Provide an implementation for a type | `given TypeClass[SomeType] with ...` |
| Summon an instance by name | `summon[TypeClass[A]]` or a custom `apply` in the companion |
| Write code generic over "has this capability" | `def f[A: TypeClass](...)` (context bound) |
| Give instances a natural method-call syntax | `extension [A](a: A)(using tc: TypeClass[A]) def op = ...` |
| Add capability to a type you don't own | define a `given` for it — no inheritance required |

## Exercise

Define a `Ord[A]` type class with `def compare(x: A, y: A): Int` (negative
if `x < y`, zero if equal, positive if `x > y`), provide `given` instances
for `Int` and `String`, and write `def maxOf[A: Ord](xs: List[A]): A` that
finds the largest element using only the type class (no `.max` from the
standard library). Add an extension method `def isGreaterThan[A](other:
A)(using ord: Ord[A]): Boolean` and use it to compare two `Point`s by
distance from the origin (you'll need a `given Ord[Point]` computing
`x*x + y*y` for each side). Confirm `maxOf` works on both `List[Int]` and
`List[String]`.
