# 06 · Classes & Objects Basics

## Defining a class

```scala
class Person(val name: String, var age: Int):
  def greet(): String =
    s"Hi, I'm $name and I'm $age years old."

val ada = Person("Ada", 28)
println(ada.name)          // Ada
println(ada.greet())       // Hi, I'm Ada and I'm 28 years old.

ada.age = 29                // allowed -- age was declared with var
println(ada.age)            // 29
// ada.name = "Grace"       // error -- name was declared with val
```

The constructor parameters go directly in the class signature. Prefixing a
parameter with `val` (or `var`) both stores it as a field *and* generates a
public accessor — this is far more concise than Java's constructor + private
field + getter boilerplate.

## Methods and fields

```scala
class Rectangle(val width: Double, val height: Double):
  def area: Double = width * height          // no-parens method -- reads like a field
  def perimeter(): Double = 2 * (width + height)

  def scaled(factor: Double): Rectangle =
    Rectangle(width * factor, height * factor)

val r = Rectangle(3.0, 4.0)
println(r.area)             // 12.0 -- called without parentheses
println(r.perimeter())      // 14.0
println(r.scaled(2.0).area) // 48.0
```

Notice `scaled` returns a brand-new `Rectangle` rather than mutating `r` —
this mirrors the immutable-by-default style from earlier modules and carries
through the rest of the course.

## Auxiliary constructors

```scala
class Point(val x: Double, val y: Double):
  def this() = this(0.0, 0.0)   // auxiliary constructor -- must call the primary one

  def describe: String = s"($x, $y)"

val origin = Point()
val custom = Point(3.0, 4.0)
println(origin.describe)   // (0.0, 0.0)
println(custom.describe)   // (3.0, 4.0)
```

## `object` — singletons

An `object` declares a class with exactly one instance, created lazily on
first use. It's Scala's replacement for `static` members in Java:

```scala
object MathUtils:
  val Pi = 3.14159
  def square(n: Double): Double = n * n

println(MathUtils.Pi)          // 3.14159
println(MathUtils.square(5))   // 25.0
```

## Companion objects

An `object` with the *same name* as a class, declared in the same file, is
that class's **companion object**. It's the idiomatic place for factory
methods and constants related to the class:

```scala
class Circle private (val radius: Double):
  def area: Double = Circle.Pi * radius * radius

object Circle:
  val Pi = 3.14159

  // Factory method -- lets callers write Circle(radius) instead of new Circle(radius)
  def apply(radius: Double): Circle =
    if radius < 0 then throw new IllegalArgumentException("radius must be >= 0")
    else new Circle(radius)

val c = Circle(2.0)          // calls Circle.apply -- no "new" needed
println(c.area)              // 12.56636
```

`Circle`'s primary constructor is marked `private`, so the *only* way to
build one from outside the file is through the companion's `apply` — a
common pattern for validating inputs at construction time.

## `apply` — why `Circle(2.0)` works

Any object (or class) with an `apply` method can be "called" like a
function — Scala rewrites `Circle(2.0)` to `Circle.apply(2.0)` automatically.
This is the same mechanism that lets you write `List(1, 2, 3)` instead of
`List.apply(1, 2, 3)`.

## Visibility modifiers

```scala
class BankAccount(private var balance: Double):
  def deposit(amount: Double): Unit =
    balance += amount

  def currentBalance: Double = balance   // controlled read access

val account = BankAccount(100.0)
account.deposit(50.0)
println(account.currentBalance)   // 150.0
// account.balance -- error: balance is private
```

`private` restricts a member to the class itself (and, in Scala, its
companion object too). There's no field to accidentally expose — callers
only ever see the methods you choose to publish.

## Cheat sheet

| Concept | Syntax | Purpose |
|---------|--------|---------|
| Class | `class Foo(val x: Int)` | Blueprint for objects, with constructor params as fields |
| Auxiliary constructor | `def this(...) = this(...)` | Alternate ways to build an instance |
| Singleton | `object Foo` | Exactly one instance, `static`-like members |
| Companion object | `object Foo` next to `class Foo` | Factory methods, constants tied to the class |
| `apply` | `def apply(...) = ...` | Lets `Foo(...)` work without `new` |
| `private` | `private val x = ...` | Restricts visibility to the class (+ companion) |

## Exercise

Write a class `Temperature(val celsius: Double)` with a method `toFahrenheit: Double`.
Give it a companion object with two factory methods: `fromCelsius(c: Double): Temperature`
and `fromFahrenheit(f: Double): Temperature` (the latter converts to Celsius
internally). Add a no-arg auxiliary constructor that defaults to `0.0`
Celsius. Test all three ways of constructing a `Temperature` and print their
Fahrenheit values.
