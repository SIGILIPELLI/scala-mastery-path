# 08 · Scala 3 Features

You've used `given`/`using` (Level 2, formalized in Level 3's type
classes), `enum` for ADTs, and `extension` methods throughout this course
already — they're Scala 3's biggest departures from Scala 2, and you've
been writing idiomatic Scala 3 the whole time. This module rounds out the
set with two features not yet covered: **opaque types** and **union
types** — plus a proper look at enums with data.

## Enums beyond simple cases

Level 1's pattern matching used simple enums. Scala 3 enums can carry data
per case, making them a concise way to define an algebraic data type that
in Scala 2 needed a `sealed trait` plus several `case class`es:

```scala
enum Direction:
  case North, South, East, West
  def opposite: Direction = this match
    case North => South
    case South => North
    case East => West
    case West => East

println(Direction.North.opposite)   // South

enum Shape:
  case Circle(radius: Double)
  case Rectangle(w: Double, h: Double)

def area(s: Shape): Double = s match
  case Shape.Circle(r)       => math.Pi * r * r
  case Shape.Rectangle(w, h) => w * h

println(area(Shape.Circle(2.0)))       // 12.566370614359172
println(area(Shape.Rectangle(3, 4)))   // 12.0
```

An `enum` can even mix methods (like `opposite`) with data-carrying cases
(like `Circle`/`Rectangle`) in the same declaration — this is the same
exhaustiveness-checked ADT shape you've relied on since Level 1's pattern
matching, just with `case class`-per-variant boilerplate collapsed into one
`enum` block.

## Opaque types: zero-cost type safety

Passing raw `Int`s around for things like user ids, order ids, and
quantities makes it easy to accidentally swap two arguments of the same
underlying type — the compiler can't catch `chargeUser(orderId, userId)`
called with the arguments reversed if both are just `Int`. An **opaque
type** gives you a distinct type at compile time with *zero* runtime
overhead — it compiles down to the underlying type, no wrapper object
allocated:

```scala
opaque type UserId = Int
object UserId:
  def apply(raw: Int): UserId = raw
  extension (id: UserId) def raw: Int = id

val uid = UserId(42)
println(uid.raw)   // 42
// uid: Int  -- would NOT compile; UserId and Int are distinct types outside this file
```

Outside the file that declares `opaque type UserId = Int`, `UserId` and
`Int` are completely distinct, unrelated types — you cannot pass a plain
`Int` where a `UserId` is expected, or vice versa, without going through
`UserId.apply`/`.raw`. Inside the declaring file (and only there), the
compiler still knows `UserId` *is* `Int` underneath, which is what lets
`.raw` and `apply` be trivial identity functions with no boxing.

### The trap: a `case class` wrapper isn't free

Before opaque types, the usual way to get this same safety was
`case class UserId(raw: Int)` — which works, but allocates a real object
on the heap for every `UserId`, wrapping the `Int` it holds. In a hot path
creating millions of ids, that allocation is measurable. `opaque type`
gives you the identical compile-time safety with none of the runtime cost
— reach for it specifically when you want a distinct type purely as a
compile-time distinction, not extra behavior (for extra methods and data,
a `case class` remains the right tool).

## Union types: "this or that," without a common supertype

Scala's had `Either[L, R]` since Level 2 for "one of two things, with a
label for which." A **union type** (`A | B`) expresses the same "one of
these types" idea directly in the type signature, without wrapping in
`Left`/`Right`:

```scala
def processOrder(status: Int | String): String = status match
  case n: Int    => s"code $n"
  case s: String => s"status $s"

println(processOrder(404))        // code 404
println(processOrder("timeout"))  // status timeout
```

`Int | String` accepts *either* type directly at the call site — no
`Left(404)`/`Right("timeout")` wrapping needed — and the `match` on
concrete types (`case n: Int`, `case s: String`) is exhaustively checked by
the compiler against the declared union.

### The trap: union types vs. `Either` — pick based on symmetry

`Either[L, R]` carries a clear asymmetry (by convention, `Left` = failure,
`Right` = success) and comes with `map`/`flatMap` biased toward `Right` —
useful exactly when one side represents an error to propagate. A union
type like `Int | String` has no such bias — both sides are just "valid
inputs," with no special short-circuiting behavior. Reach for `Either` when
one side means "stop, something went wrong"; reach for a union type when
several genuinely different-but-valid input shapes need to flow through the
same function, like `processOrder` above accepting either a numeric or
string status.

## Extension methods, revisited

Module 06 of Level 3 used `extension` to add `.show` to arbitrary types via
a type class. The same syntax works directly on a concrete type without
any type class machinery at all — this is Scala 3's replacement for Scala
2's implicit-class pattern for "add a method to a type I don't own":

```scala
case class Point(x: Int, y: Int)

extension (p: Point)
  def +(other: Point): Point = Point(p.x + other.x, p.y + other.y)

println(Point(1, 2) + Point(3, 4))   // Point(4,6)
```

## Cheat sheet

| Feature | Use it for |
|---|---|
| `enum` with data-carrying cases | An ADT (sealed hierarchy) without per-case `case class` boilerplate |
| `opaque type A = B` | A distinct compile-time type over `B`, zero runtime cost |
| `A \| B` (union type) | "One of several valid, unrelated input shapes," no error bias |
| `Either[L, R]` | "One of two outcomes," with `L` conventionally meaning failure |
| `extension (x: T) def method = ...` | Add a method to a type you don't own, no type class needed |

## Exercise

Define `opaque type Meters = Double` and `opaque type Feet = Double` with
`apply`/`.value` extensions for each, plus a `toFeet(m: Meters): Feet`
conversion function — confirm the compiler rejects passing a `Feet` value
where a `Meters` is expected even though both are `Double` underneath.
Then write `def parseConfig(value: String | Int | Boolean): String`
handling all three cases in a `match`, and an `enum ConfigError` with cases
`Missing(key: String)` and `Invalid(key: String, reason: String)` used as
the error type in an `Either[ConfigError, String]`-returning lookup
function.
