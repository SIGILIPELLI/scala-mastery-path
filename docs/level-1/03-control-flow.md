# 03 · Control Flow

## `if`/`else` is an expression

In Scala, `if`/`else` produces a value — you don't need a separate ternary
operator, and you often assign the result directly to a `val`:

```scala
val age = 20

val category =
  if age < 13 then "child"
  else if age < 20 then "teen"
  else "adult"

println(category)   // adult
```

```scala
// Old-style parentheses form still works and is common in Scala 2 code
val max =
  if (10 > 5) 10
  else 5

println(max)   // 10
```

Because `if`/`else` is an expression, both branches must produce compatible
types (or Scala widens to a common supertype) for the result to be useful as
a value.

## `while` loops

`while` exists but is used sparingly in idiomatic Scala — it's imperative
and needs a `var` to do anything useful:

```scala
var i = 0
while i < 5 do
  println(s"i = $i")
  i += 1
// i = 0
// i = 1
// i = 2
// i = 3
// i = 4
```

## `for` loops and `for` expressions

`for` in Scala is more powerful than a typical C-style loop — it iterates
over any collection, and with `yield` it becomes a **for-expression** that
produces a new collection (covered in depth in
[Level 2](../level-2/09-for-comprehensions.md)):

```scala
// Simple iteration -- a "for loop" used for side effects
for i <- 1 to 5 do
  println(i)
// 1
// 2
// 3
// 4
// 5

// Ranges: 1 to 5 is inclusive, 1 until 5 is exclusive
println((1 to 5).toList)     // List(1, 2, 3, 4, 5)
println((1 until 5).toList)  // List(1, 2, 3, 4)

// for-expression: yield produces a NEW collection instead of side effects
val doubled = for i <- 1 to 5 yield i * 2
println(doubled)             // Vector(2, 4, 6, 8, 10)

// Filtering inside a for
val evens = for i <- 1 to 10 if i % 2 == 0 yield i
println(evens)               // Vector(2, 4, 6, 8, 10)

// Nested iteration
val pairs = for
  x <- 1 to 2
  y <- 1 to 2
yield (x, y)
println(pairs)                // Vector((1,1), (1,2), (2,1), (2,2))
```

## `match` expressions (a preview)

Scala's `match` is like a much more powerful `switch`. It gets a full module
to itself ([Module 8](08-pattern-matching-intro.md)), but here's the basic
shape since it's genuinely everyday control flow:

```scala
val day = 3

val name = day match
  case 1 => "Monday"
  case 2 => "Tuesday"
  case 3 => "Wednesday"
  case _ => "Some other day"

println(name)   // Wednesday
```

## Prefer expressions over mutation

Idiomatic Scala favors deriving a new value over mutating an existing one.
Compare two ways of summing a range:

```scala
// Imperative style -- works, but not idiomatic Scala
var total = 0
for i <- 1 to 5 do
  total += i
println(total)   // 15

// Functional style -- no var, sum is a method on the range
val total2 = (1 to 5).sum
println(total2)  // 15
```

You'll see this pattern throughout the course: reach for collection methods
(`sum`, `map`, `filter`, `fold`, and friends — introduced in
[Module 5](05-collections-basics.md) and deepened in
[Level 2](../level-2/03-collections-deep-dive.md)) before reaching for a
`var` and a loop.

## Cheat sheet

| Construct | Purpose |
|-----------|---------|
| `if ... then ... else ...` | Conditional expression, returns a value |
| `while ... do ...` | Imperative loop, needs a `var` to be useful |
| `for i <- range do ...` | Loop over a collection for side effects |
| `for i <- range yield ...` | Build a new collection from a loop |
| `x match { case ... }` | Multi-way branching on a value's shape |

## Exercise

Write a for-expression that produces a list of the squares of all odd
numbers from 1 to 20. Then write a `match` expression that takes an `Int`
grade (0-100) and returns a letter grade string (`"A"`, `"B"`, `"C"`, `"D"`,
`"F"`) using range guards (`case n if n >= 90 => "A"`).
