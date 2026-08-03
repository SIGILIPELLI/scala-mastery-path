# 01 · OOP Deep Dive

[Level 1](../level-1/09-traits-basics.md) introduced traits as mixable
contracts and gave you a first taste of stacking traits with `super`. This
module goes deeper into what actually happens when several traits are mixed
into one class — the order Scala resolves calls in, the `abstract override`
pattern that makes stackable traits work, self-types for declaring "this
trait needs a particular collaborator," and a closer look at how case
classes and companion objects combine to build safe, well-modeled types.

## Linearization: what order do mixed-in traits run in?

When a class mixes in multiple traits, Scala builds a single linear order
(the *linearization*) that determines both which implementation "wins" and
what `super` refers to inside each trait. The rule that matters day to day:
**the rightmost trait in the `with` chain is the first one whose override
runs**, and each trait's `super` call forwards to the *next* trait moving
leftward — not to a fixed parent.

```scala
trait Greeter:
  def name: String
  def greet(): String = s"Hello, $name"

trait Loud extends Greeter:
  override def greet(): String = super.greet().toUpperCase + "!!!"

trait Polite extends Greeter:
  override def greet(): String = super.greet() + ", nice to meet you"

class Person(val name: String) extends Greeter with Loud with Polite
class Person2(val name: String) extends Greeter with Polite with Loud

println(Person("Ada").greet())
// HELLO, ADA!!!, nice to meet you
// Polite runs first (rightmost), calls super -> Loud, which calls super -> Greeter

println(Person2("Ada").greet())
// HELLO, ADA, NICE TO MEET YOU!!!
// Loud runs first (rightmost), calls super -> Polite, which calls super -> Greeter
```

The two classes mix in exactly the same traits, only in a different order,
and get completely different behavior. This trips people up constantly:
**mixin order is not cosmetic — it changes which code runs first and what
each `super` call sees.**

## `abstract override`: the pattern that makes stacking work

Notice `Loud` and `Polite` above only work as overrides *of* `Greeter`
because `Greeter.greet()` has a real implementation to fall back on. If a
trait wants to call `super.someMethod()` on a method that might not be
implemented yet in any concrete class it's eventually mixed into, it must
mark the override `abstract override` — a promise that "some trait or class
further left in the chain will supply a real implementation before this
runs."

```scala
trait Counter:
  private var count = 0
  def increment(): Int =
    count += 1
    count

trait LoggingCounter extends Counter:
  abstract override def increment(): Int =
    val result = super.increment()
    println(s"[log] incremented to $result")
    result

class RealCounter extends Counter with LoggingCounter

val rc = RealCounter()
rc.increment()   // [log] incremented to 1
rc.increment()   // [log] incremented to 2
```

`LoggingCounter` adds logging *around* whatever `increment()` it's stacked
on top of, without knowing or caring what that implementation is. This is
the "stackable trait" pattern: each trait in the chain contributes one
slice of behavior, and `with` lets you compose them like middleware.

## Self-types: declaring a required collaborator

A **self-type** lets a trait say "I can only be mixed into something that
also has type `X`," without extending `X` itself. It's how you express a
dependency between traits that shouldn't be tied together by inheritance.

```scala
trait HasEngine:
  def horsepower: Int

trait Drivable:
  self: HasEngine =>                       // Drivable requires a HasEngine
  def drive(): String = s"Driving with $horsepower hp"

class Car(val horsepower: Int) extends HasEngine with Drivable

println(Car(300).drive())   // Driving with 300 hp
```

`Drivable` can reference `horsepower` even though it doesn't define it,
because the self-type annotation guarantees anything using `Drivable` also
provides `HasEngine`. Try mixing `Drivable` into a class that *doesn't*
extend `HasEngine` and it won't compile — the self-type is checked at
compile time, not just documented in a comment.

## Sealed hierarchies of case classes

Level 1 covered `sealed trait` with `case object`s for enum-like values.
The same `sealed` mechanism works with `case class`es that carry data,
which is the more common shape for real domain models:

```scala
sealed trait Shape
case class Circle(radius: Double) extends Shape
case class Rectangle(width: Double, height: Double) extends Shape
case class Triangle(base: Double, height: Double) extends Shape

def area(s: Shape): Double = s match
  case Circle(r)        => Math.PI * r * r
  case Rectangle(w, h)  => w * h
  case Triangle(b, h)   => 0.5 * b * h
  // no case _ needed -- the compiler can verify all three subtypes are handled

val shapes: List[Shape] = List(Circle(2.0), Rectangle(3.0, 4.0), Triangle(5.0, 2.0))
shapes.foreach(s => println(f"${area(s)}%.2f"))
// 12.57
// 12.00
// 5.00
```

This `sealed trait` + `case class` combination is Scala's version of an
**algebraic data type (ADT)** — it's the backbone of idiomatic Scala
modeling, and you'll lean on it constantly in
[Module 2 · Advanced Pattern Matching](02-pattern-matching-advanced.md) and
[Module 4 · Option/Either](04-option-either.md).

## Companion objects as smart constructors

[Level 1](../level-1/06-classes-objects.md) showed a companion object's
`apply` replacing `new`. A step further: make the primary constructor
`private` and route all construction through a companion method that
validates the input and reports failure as data instead of an exception.

```scala
class Account private (val id: String, val balance: Double)

object Account:
  def open(id: String, initialDeposit: Double): Either[String, Account] =
    if initialDeposit < 0 then Left("Initial deposit cannot be negative")
    else if id.isBlank then Left("Account id cannot be blank")
    else Right(new Account(id, initialDeposit))

println(Account.open("acc-1", 100.0))   // Right(Account@...)
println(Account.open("acc-2", -5.0))    // Left(Initial deposit cannot be negative)
```

Because the constructor is `private`, the *only* way to get an `Account`
from outside this file is `Account.open`, and that method can never return
an invalid one. This "smart constructor" pattern shows up constantly once
you've met `Either` properly in Module 4 — it's the idiomatic Scala
alternative to throwing in a constructor.

## Cheat sheet

| Concept | What it means |
|---|---|
| Linearization | The order Scala flattens a `with` chain into for method dispatch |
| Rightmost trait wins | In `extends A with B with C`, `C`'s override is the entry point |
| `super` in a trait | Forwards to the *next* trait leftward in the linearization, not a fixed parent |
| `abstract override` | Marks an override that requires a real implementation supplied further down the chain |
| Self-type (`self: T =>`) | Declares "this trait can only be used where `T` is also present," without extending `T` |
| `sealed trait` + `case class`es | An algebraic data type — exhaustiveness-checked, data-carrying hierarchy |
| Private constructor + companion factory | "Smart constructor" — the only path to a valid instance |

## Exercise

Model a small permissions system: `sealed trait Role` with `case object Admin`,
`case object Editor`, and `case object Viewer`. Then build a stackable trait
pair around a `trait Repository` with a single method
`def save(item: String): String` (base implementation just returns
`s"saved: $item"`). Write `trait Auditing extends Repository` using
`abstract override` to prepend `"[audit] "` to whatever the wrapped
`save` returns, and `trait Validating extends Repository` using
`abstract override` to reject (`throw new IllegalArgumentException`) any
blank item before delegating to `super.save`. Mix both into one class in two
different orders and observe how the order changes whether validation or
auditing "sees" the call first.
