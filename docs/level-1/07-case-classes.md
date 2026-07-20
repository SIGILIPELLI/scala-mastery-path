# 07 · Case Classes

## What a case class gives you for free

A `case class` is Scala's go-to tool for modeling data. Compare it to the
`class` from the previous module:

```scala
case class Point(x: Double, y: Double)

val p1 = Point(3.0, 4.0)     // no "new" needed
val p2 = Point(3.0, 4.0)

println(p1)              // Point(3.0,4.0) -- toString for free
println(p1 == p2)         // true -- structural equality for free
println(p1.x)             // 3.0 -- fields are public vals automatically
```

Compare that to what an ordinary `class` requires to get the same behavior:
you'd have to hand-write `toString`, `equals`, and `hashCode` yourself. A
`case class` generates all of it from the constructor parameter list alone.

## What exactly you get

| Feature | Example |
|---------|---------|
| Fields are `val` by default | `p1.x` works, no explicit `val` needed in the parameter list |
| Structural `equals`/`hashCode` | `Point(1,2) == Point(1,2)` is `true` |
| Readable `toString` | `println(p1)` shows `Point(3.0,4.0)`, not a memory address |
| No `new` required | `Point(3.0, 4.0)` — a companion `apply` is generated automatically |
| `copy` method | make a modified duplicate without touching the original |
| Pattern-matching support | usable directly in `match` (see [Module 8](08-pattern-matching-intro.md)) |

## `copy` — the killer feature for immutable data

```scala
case class Employee(name: String, title: String, salary: Double)

val alice = Employee("Alice", "Engineer", 95000.0)

// Promote Alice -- produces a NEW Employee, alice is untouched
val promoted = alice.copy(title = "Senior Engineer", salary = 110000.0)

println(alice)      // Employee(Alice,Engineer,95000.0)
println(promoted)   // Employee(Alice,Senior Engineer,110000.0)
```

`copy` takes named arguments for whichever fields should change and reuses
the rest — this is the idiomatic way to "update" immutable data throughout
Scala code.

## Nesting case classes

```scala
case class Address(street: String, city: String, zip: String)
case class Customer(name: String, address: Address)

val cust = Customer("Bob", Address("1 Main St", "Springfield", "00000"))

println(cust.address.city)   // Springfield

// Update a nested field -- copy the outer AND the inner value
val moved = cust.copy(address = cust.address.copy(city = "Shelbyville"))
println(moved.address.city)   // Shelbyville
println(cust.address.city)    // Springfield -- original untouched
```

Notice both `copy` calls use named arguments (`city = "Shelbyville"`) — the
usual syntax for supplying arguments by name anywhere in Scala.

## Structural equality vs. reference equality

```scala
class RegularPoint(val x: Int, val y: Int)
val rp1 = RegularPoint(1, 2)
val rp2 = RegularPoint(1, 2)
println(rp1 == rp2)   // false -- plain classes compare by reference (identity)

case class CasePoint(x: Int, y: Int)
val cp1 = CasePoint(1, 2)
val cp2 = CasePoint(1, 2)
println(cp1 == cp2)   // true -- case classes compare by value (structure)
```

This is exactly why case classes are the default choice for data: two
`CasePoint`s with the same coordinates are considered equal, which is almost
always what you actually want when modeling values (as opposed to modeling
identity, like a database connection or a mutable account).

## Case classes with methods

Case classes aren't limited to plain data — you can add methods just like a
regular class:

```scala
case class Circle(radius: Double):
  def area: Double = Math.PI * radius * radius
  def circumference: Double = 2 * Math.PI * radius

val c = Circle(5.0)
println(f"${c.area}%.2f")             // 78.54
println(f"${c.circumference}%.2f")    // 31.42
```

## Case objects

For a value type that has no data — just one meaningful instance, like an
enum-style tag — use `case object` instead:

```scala
sealed trait TrafficLight
case object Red extends TrafficLight
case object Yellow extends TrafficLight
case object Green extends TrafficLight

val current: TrafficLight = Green
println(current)   // Green
```

(`sealed trait` is previewed here and covered properly in
[Module 9](09-traits-basics.md); Scala 3's `enum` — often a cleaner fit for
this exact pattern — appears in [Level 4](../level-4/08-scala-3-features.md).)

## Exercise

Define `case class Book(title: String, author: String, year: Int, read: Boolean = false)`.
Create a `List[Book]` of at least four books, at least two marked `read = true`.
Using `copy`, produce a new list where every unread book has been marked as
read (hint: `books.map(b => if !b.read then b.copy(read = true) else b)`).
Print both the original and updated lists to confirm the originals are
unchanged.
