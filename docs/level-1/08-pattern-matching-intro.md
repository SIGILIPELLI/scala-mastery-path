# 08 · Pattern Matching Intro

## Basic `match`

You've already seen the simplest form of `match` in
[Module 3](03-control-flow.md) — matching literal values:

```scala
def dayName(day: Int): String =
  day match
    case 1 => "Monday"
    case 2 => "Tuesday"
    case 3 => "Wednesday"
    case 4 => "Thursday"
    case 5 => "Friday"
    case 6 | 7 => "Weekend"     // multiple patterns, same result
    case _ => "Invalid day"

println(dayName(6))   // Weekend
println(dayName(9))   // Invalid day
```

`match` is exhaustiveness-friendly and generally preferred over long
`if`/`else if` chains once you have more than two or three branches.

## Matching on type

```scala
def describe(x: Any): String =
  x match
    case i: Int => s"an Int: $i"
    case s: String => s"a String of length ${s.length}"
    case b: Boolean => s"a Boolean: $b"
    case _ => "something else"

println(describe(42))          // an Int: 42
println(describe("hello"))     // a String of length 5
println(describe(true))        // a Boolean: true
println(describe(3.14))        // something else
```

## Destructuring case classes

This is where `match` really shines — you can match on a case class's
*shape* and bind its fields to names in one step:

```scala
case class Point(x: Int, y: Int)

def classify(p: Point): String =
  p match
    case Point(0, 0) => "origin"
    case Point(0, _) => "on the y-axis"
    case Point(_, 0) => "on the x-axis"
    case Point(x, y) if x == y => "on the diagonal"
    case Point(x, y) => s"a regular point at ($x, $y)"

println(classify(Point(0, 0)))   // origin
println(classify(Point(0, 5)))   // on the y-axis
println(classify(Point(5, 0)))   // on the x-axis
println(classify(Point(3, 3)))   // on the diagonal
println(classify(Point(2, 7)))   // a regular point at (2, 7)
```

The `if` clauses above (`case Point(x, y) if x == y => ...`) are called
**guards** — an extra condition checked only after the shape matches.

## Matching on lists

```scala
def describeList(xs: List[Int]): String =
  xs match
    case Nil => "empty list"
    case x :: Nil => s"single element: $x"
    case x :: y :: Nil => s"two elements: $x and $y"
    case first :: rest => s"starts with $first, ${rest.length} more after"

println(describeList(Nil))              // empty list
println(describeList(List(5)))          // single element: 5
println(describeList(List(1, 2)))       // two elements: 1 and 2
println(describeList(List(1, 2, 3, 4))) // starts with 1, 3 more after
```

`::` (pronounced "cons") is the same operator you used to prepend an element
to a list in [Module 5](05-collections-basics.md) — here it's used in
reverse, to *deconstruct* a list into its head and tail.

## Matching on tuples

```scala
def quadrant(point: (Int, Int)): String =
  point match
    case (0, 0) => "origin"
    case (x, y) if x > 0 && y > 0 => "quadrant I"
    case (x, y) if x < 0 && y > 0 => "quadrant II"
    case (x, y) if x < 0 && y < 0 => "quadrant III"
    case (x, y) if x > 0 && y < 0 => "quadrant IV"
    case _ => "on an axis"

println(quadrant((3, 4)))    // quadrant I
println(quadrant((-2, 5)))   // quadrant II
println(quadrant((0, 7)))    // on an axis
```

## `match` as an expression

Like `if`/`else`, `match` produces a value — you rarely need a `var` to
accumulate a result across branches:

```scala
val n = 7
val parity = n match
  case x if x % 2 == 0 => "even"
  case _ => "odd"

println(parity)   // odd
```

## Cheat sheet

| Pattern | Matches |
|---------|---------|
| `case 3 =>` | The literal value `3` |
| `case 3 \| 4 =>` | Either `3` or `4` |
| `case i: Int =>` | Any value of type `Int`, bound to `i` |
| `case Point(x, y) =>` | A `Point`, binding its fields to `x` and `y` |
| `case x :: rest =>` | A non-empty list, binding head and tail |
| `case (a, b) =>` | A 2-tuple, binding both elements |
| `case x if cond =>` | Adds a guard condition after the shape matches |
| `case _ =>` | Matches anything (catch-all) |

## Exercise

Define `case class Shape2(kind: String, a: Double, b: Double = 0.0)` used to
represent either a `"circle"` (radius in `a`) or a `"rectangle"` (width `a`,
height `b`). Write a function `area(shape: Shape2): Double` that pattern
matches on `shape.kind` to compute the correct area, throwing a
`MatchError`-friendly message via `case _ => throw new IllegalArgumentException(...)`
for unknown kinds. Test it with a few circles and rectangles.
