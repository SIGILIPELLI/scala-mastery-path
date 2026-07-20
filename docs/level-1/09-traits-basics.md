# 09 · Traits Basics

## What a trait is

A `trait` defines a contract — behavior and/or state that a class can mix
in. It's similar to a Java interface, but richer: traits can hold concrete
method implementations *and* fields, not just abstract signatures.

```scala
trait Greetable:
  def name: String                       // abstract -- no body, must be provided
  def greet(): String = s"Hello, $name!" // concrete -- provided for free

class Person(val name: String) extends Greetable

val p = Person("Ada")
println(p.greet())   // Hello, Ada!
```

`Person` only had to supply `name` — it got `greet()` for free from the
trait.

## Mixing in multiple traits

Unlike classes (Scala only allows single inheritance via `extends`), a class
can mix in as many traits as it needs using `with`:

```scala
trait Flyable:
  def fly(): String = "Flying!"

trait Swimmable:
  def swim(): String = "Swimming!"

class Duck extends Flyable with Swimmable

val duck = Duck()
println(duck.fly())    // Flying!
println(duck.swim())   // Swimming!
```

## Abstract members

Traits can declare abstract `def`s, `val`s, and `var`s that implementing
classes must fill in:

```scala
trait Shape:
  def area: Double
  def name: String
  def describe(): String = s"$name has area $area"

class Square(side: Double) extends Shape:
  def area: Double = side * side
  def name: String = "Square"

class Circle(radius: Double) extends Shape:
  def area: Double = Math.PI * radius * radius
  def name: String = "Circle"

val shapes: List[Shape] = List(Square(4), Circle(3))
for shape <- shapes do
  println(shape.describe())
// Square has area 16.0
// Circle has area 28.274333882308138
```

This is polymorphism in action: `describe()` is written once against the
trait, but works for every current and future `Shape` implementation.

## Traits vs. abstract classes

Scala also has `abstract class`, which looks similar but has one key
restriction: a class can `extend` only *one* class (abstract or not), while
it can mix in *many* traits.

```scala
abstract class Animal(val name: String):
  def sound(): String
  def describe(): String = s"$name says ${sound()}"

class Dog(name: String) extends Animal(name):
  def sound(): String = "Woof!"

println(Dog("Rex").describe())   // Rex says Woof!
```

| Aspect | `trait` | `abstract class` |
|--------|---------|-------------------|
| Instantiable directly? | No | No |
| Constructor parameters | Not until Scala 3 (limited) | Yes, freely |
| Can a class combine several? | Yes, via multiple `with` | No — only one `extends` |
| Use when | Defining reusable, composable capability | Modeling a single, closely related class hierarchy with shared constructor logic |

## Sealed traits

Marking a trait `sealed` means every class that extends it must live in the
*same file* — the compiler then knows the complete list of subtypes, which
lets it check whether a `match` covers every case:

```scala
sealed trait Direction
case object North extends Direction
case object South extends Direction
case object East extends Direction
case object West extends Direction

def opposite(d: Direction): Direction =
  d match
    case North => South
    case South => North
    case East => West
    case West => East
    // no "case _" needed -- the compiler knows these four are the only options

println(opposite(North))   // South
```

If you later add a fifth `case object` and forget to update `opposite`, the
compiler warns you that the `match` is no longer exhaustive — a powerful
safety net that plain classes and traits don't give you.

## Stacking traits with `override`

When multiple mixed-in traits define the same method, Scala resolves the
call through a linearization order (right to left in the `with` chain); each
trait's `super` call refers to the *next* trait in that order, not
necessarily a fixed parent:

```scala
trait Logger:
  def log(msg: String): String = s"[LOG] $msg"

trait TimestampedLogger extends Logger:
  override def log(msg: String): String = s"[${java.time.LocalTime.now()}] " + super.log(msg)

class App extends TimestampedLogger

println(App().log("started"))   // [12:34:56.789] [LOG] started
```

This "stackable trait" pattern is used throughout Scala library code; for
Level 1 it's enough to recognize the shape when you see it.

## Cheat sheet

| Syntax | Meaning |
|--------|---------|
| `trait Foo` | Defines a mixable contract |
| `class Bar extends Foo` | `Bar` implements/inherits `Foo` |
| `class Bar extends Foo with Baz` | `Bar` mixes in both `Foo` and `Baz` |
| `sealed trait Foo` | All subtypes must be declared in this file — enables exhaustive `match` |
| `override def x = ...` | Provide/replace a member from a parent trait or class |

## Exercise

Define a `sealed trait PaymentMethod` with `case object Cash`,
`case object Card`, and `case object Crypto`. Write a function
`processingFee(method: PaymentMethod): Double` that pattern matches to
return a different fee for each (e.g. `0.0`, `0.02`, `0.05`). Separately,
define traits `Discountable` (with a `discountRate: Double` and a concrete
`applyDiscount(price: Double): Double`) and `Taxable` (with a `taxRate: Double`
and a concrete `applyTax(price: Double): Double`), then create a
`case class Product(name: String, price: Double, discountRate: Double, taxRate: Double)`
that mixes in both, and compute a final price by chaining
`applyDiscount` then `applyTax`.
