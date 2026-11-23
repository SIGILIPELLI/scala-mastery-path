---
title: "Scala Interview Questions and Answers (179 Questions)"
description: "179 Scala interview questions with answers and explanations, from beginner to advanced, organised by level. Free, from the Scala Mastery Path."
---

# Scala Interview Questions and Answers

**179 questions** with the correct answer and a short explanation, organised from beginner (Level 1) to advanced (Level 4). Every question comes from a lesson in the free Scala Mastery Path, so you can open the lesson to go deeper.

!!! tip "Practise, don't just read"
    Each lesson also ends with an interactive quiz, and the [final quiz](quiz.md) tests all four levels. Try to answer each question below before reading the answer.

## Level 1 · Entry

### [01 · Setup &amp; First Program](level-1/01-setup.md)

**Q1.** What does the JVM give Scala programs?

**Answer:** A portable runtime with garbage collection and access to the Java library ecosystem

Scala compiles to JVM bytecode (also JS and native targets exist).

**Q2.** What does sbt do?

**Answer:** Builds Scala projects: compiles, manages dependencies, runs tests and packages

build.sbt defines the project.

**Q3.** What does @main do in Scala 3?

**Answer:** Marks a top-level method as a program entry point

No object with main(args) needed.

**Q4.** What is the Scala REPL for?

**Answer:** Trying expressions interactively

Great for exploring the standard library.

### [02 · Variables &amp; Types](level-1/02-variables-types.md)

**Q5.** What does this print?

```scala
@main def m() =
  val x = 5
  var y = 1
  y += x
  val s: String = s"x=$x y=$y"
  lazy val boom = { println("computed"); 42 }
  println(s)
  println(boom + boom)
```

**Answer:** x=5 y=6
computed
84

lazy val is evaluated once on first use. Verified by compiling and running it.

**Q6.** What happens if you reassign a val?

**Answer:** Compile error

Use var for mutable bindings, though immutability is preferred.

**Q7.** What does this print?

```scala
@main def m() =
  println(7 / 2)
  println(7 / 2.0)
  println(7 % 3)
  println("ab" * 3)
  println((1 to 4).sum)
```

**Answer:** 3
3.5
1
ababab
10

Integer division truncates; String * repeats. Verified by compiling and running it.

**Q8.** What is Scala's top type?

**Answer:** Any (with AnyVal and AnyRef below it)

Nothing is the bottom type.

### [03 · Control Flow](level-1/03-control-flow.md)

**Q9.** What does this print?

```scala
@main def m() =
  val n = 15
  val r = if n % 15 == 0 then "FizzBuzz" else if n % 3 == 0 then "Fizz" else n.toString
  println(r)
  val xs = for i <- 1 to 5 if i % 2 == 1 yield i * i
  println(xs)
  var i = 0
  while i < 3 do i += 1
  println(i)
```

**Answer:** FizzBuzz
Vector(1, 9, 25)
3

if is an expression; a for-yield over a Range gives an IndexedSeq (Vector). Verified by compiling and running it.

**Q10.** What does for ... yield return?

**Answer:** A new collection of the yielded values

Without yield it is only for side effects.

**Q11.** What is the type of the expression println("hi")?

**Answer:** Unit

Unit has the single value ().

**Q12.** What does the match expression return?

**Answer:** The value of the chosen case

It is an expression.

### [04 · Functions](level-1/04-functions.md)

**Q13.** What does this print?

```scala
def add(a: Int, b: Int = 10): Int = a + b
def sum(xs: Int*): Int = xs.sum
def curried(a: Int)(b: Int) = a * b
@main def m() =
  println(add(1))
  println(add(b = 1, a = 5))
  println(sum(1, 2, 3))
  println(curried(3)(4))
  val f = (x: Int) => x + 1
  println(List(1, 2).map(f))
```

**Answer:** 11
6
6
12
List(2, 3)

Default and named arguments, varargs and curried parameter lists. Verified by compiling and running it.

**Q14.** What does a method return if it has no explicit return?

**Answer:** The value of its last expression

Avoid return in Scala.

**Q15.** What is a higher-order function?

**Answer:** A function that takes or returns functions

map and filter are examples.

**Q16.** What does def f(x: =&gt; Int) mean?

**Answer:** x is a by-name parameter evaluated each time it is used

Evaluated lazily at use.

### [05 · Collections Basics](level-1/05-collections-basics.md)

**Q17.** What does this print?

```scala
@main def m() =
  val xs = List(3, 1, 2)
  println(xs.sorted)
  println(xs.map(_ * 2).filter(_ > 2))
  println(xs.foldLeft(0)(_ + _))
  println(xs :+ 4)
  println(0 :: xs)
  val m = Map("a" -> 1, "b" -> 2)
  println(m.getOrElse("z", 0))
  println(m + ("c" -> 3))
  println(xs.zipWithIndex)
```

**Answer:** List(1, 2, 3)
List(6, 4)
6
List(3, 1, 2, 4)
List(0, 3, 1, 2)
0
Map(a -&gt; 1, b -&gt; 2, c -&gt; 3)
List((3,0), (1,1), (2,2))

Immutable collections return new collections; :: prepends, :+ appends. Verified by compiling and running it.

**Q18.** What is the difference between List and Vector?

**Answer:** List is a linked list (fast prepend); Vector has fast random access and updates

Choose by access pattern.

**Q19.** What does the immutable Map return when you add a pair?

**Answer:** A new Map; the original is unchanged

Use mutable.Map explicitly if you need mutation.

**Q20.** What does flatMap do on a list?

**Answer:** Maps each element to a collection and flattens the results

The basis of for-comprehensions.

### [06 · Classes &amp; Objects Basics](level-1/06-classes-objects.md)

**Q21.** What does this print?

```scala
class Counter(private var n: Int = 0):
  def inc(): Unit = n += 1
  def value: Int = n
object Counter:
  def apply(start: Int): Counter = new Counter(start)
@main def m() =
  val c = Counter(5)
  c.inc(); c.inc()
  println(c.value)
  println(Counter(0).value)
```

**Answer:** 7
0

The companion's apply works like a factory; the private var is hidden. Verified by compiling and running it.

**Q22.** What is a companion object?

**Answer:** An object with the same name as a class in the same file, holding static-like members and factories

Scala has no static keyword.

**Q23.** What does object Foo define?

**Answer:** A singleton instance

Created lazily on first use.

**Q24.** What is the primary constructor?

**Answer:** The class parameter list in the class header

Auxiliary constructors use this(...).

### [07 · Case Classes](level-1/07-case-classes.md)

**Q25.** What does this print?

```scala
case class Point(x: Int, y: Int)
@main def m() =
  val p = Point(1, 2)
  val q = p.copy(y = 5)
  println(p)
  println(q)
  println(p == Point(1, 2))
  val Point(a, b) = q
  println(a + b)
  println(p.hashCode == Point(1, 2).hashCode)
```

**Answer:** Point(1,2)
Point(1,5)
true
6
true

Case classes give structural equality, copy, toString and extractors. Verified by compiling and running it.

**Q26.** Do you need new to create a case class instance?

**Answer:** No, the generated apply method is used

Point(1, 2).

**Q27.** Why are case class fields immutable by default?

**Answer:** Encourages safe, shareable values; use copy to derive changes

Mutable fields are possible with var but discouraged.

**Q28.** What other features come free with a case class?

**Answer:** equals, hashCode, toString and pattern-matching support

A concise data holder.

### [08 · Pattern Matching Intro](level-1/08-pattern-matching-intro.md)

**Q29.** What does this print?

```scala
def describe(x: Any): String = x match
  case 0 => "zero"
  case n: Int if n < 0 => "neg"
  case n: Int => s"int $n"
  case s: String => s"str ${s.length}"
  case (a, b) => s"pair $a,$b"
  case _ => "other"
@main def m() =
  println(List(0, -3, 7, "hey", (1, 2), 2.5).map(describe))
```

**Answer:** List(zero, neg, int 7, str 3, pair 1,2, other)

Cases are tried top to bottom; guards add conditions. Verified by compiling and running it.

**Q30.** What happens if a match on a sealed trait misses a case?

**Answer:** The compiler warns that the match may not be exhaustive

Seal ADTs for exhaustiveness checking.

**Q31.** What does case _ do?

**Answer:** Matches anything (wildcard)

Usually last.

**Q32.** What does case x @ Some(_) do?

**Answer:** Binds the whole matched value to x while matching the pattern

Pattern binders.

### [09 · Traits Basics](level-1/09-traits-basics.md)

**Q33.** What does this print?

```scala
trait Greeter:
  def name: String
  def greet: String = s"Hello, $name"
trait Loud extends Greeter:
  override def greet: String = super.greet.toUpperCase + "!"
class Bob extends Greeter:
  def name = "Bob"
class LoudBob extends Bob, Loud
@main def m() =
  println(Bob().greet)
  println(LoudBob().greet)
```

**Answer:** Hello, Bob
HELLO, BOB!

Traits mix in behaviour; linearisation puts Loud's override first. Verified by compiling and running it.

**Q34.** What is the difference between a trait and an abstract class?

**Answer:** A class can mix in many traits but extend only one class; traits can't take constructor args in Scala 2

Scala 3 traits can take parameters.

**Q35.** What does sealed do?

**Answer:** Restricts subclasses to the same file so matches can be checked exhaustively

Great for ADTs.

**Q36.** How do you model 'a payment is Card, Cash or Transfer'?

**Answer:** A sealed trait with case class or case object variants (an ADT)

Make illegal states unrepresentable.

### [10 · Project — CLI To-Do App](level-1/10-project-todo-app.md)

**Q37.** What is a good structure for the todo app?

**Answer:** Immutable case classes and pure functions for logic, with IO at the edges

Easy to test.

**Q38.** How do you update an immutable todo in a list?

**Answer:** Create a changed copy with copy and rebuild the list (map)

todos.map(t =&gt; if t.id == id then t.copy(done = true) else t).

**Q39.** How would you persist the todos?

**Answer:** Serialise to a file (for example JSON) at the edges

Keep persistence separate from logic.

**Q40.** Why write tests for the pure functions first?

**Answer:** No setup needed, so they are fast and reliable

Functional core, imperative shell.

## Level 2 · Intermediate

### [01 · OOP Deep Dive](level-2/01-oop-deep-dive.md)

**Q41.** What does this print?

```scala
trait Shape { def area: Double }
case class Circle(r: Double) extends Shape { def area = 3 * r * r }
case class Rect(w: Double, h: Double) extends Shape { def area = w * h }
abstract class Animal(val name: String):
  def sound: String
  override def toString = s"$name says $sound"
class Dog extends Animal("Rex"):
  def sound = "woof"
@main def m() =
  println(List(Circle(1), Rect(2, 3)).map(_.area).sum)
  println(Dog())
```

**Answer:** 9.0
Rex says woof

Polymorphism through the Shape trait; toString overrides in the abstract class. Verified by compiling and running it.

**Q42.** What does override require in Scala?

**Answer:** The override keyword when replacing a concrete member

Catches accidental overrides.

**Q43.** What does final on a class do?

**Answer:** Prevents subclassing

Use for value types.

**Q44.** What is the difference between val and def in a trait member?

**Answer:** val is computed once; def is computed on each call

Beware initialisation order with val.

### [02 · Pattern Matching Advanced](level-2/02-pattern-matching-advanced.md)

**Q45.** What does this print?

```scala
enum Expr:
  case Num(n: Int)
  case Add(a: Expr, b: Expr)
  case Mul(a: Expr, b: Expr)
import Expr.*
def eval(e: Expr): Int = e match
  case Num(n) => n
  case Add(a, b) => eval(a) + eval(b)
  case Mul(a, b) => eval(a) * eval(b)
@main def m() =
  println(eval(Add(Num(2), Mul(Num(3), Num(4)))))
  val xs = List(1, 2, 3, 4)
  xs match
    case h :: t => println(s"$h | $t")
    case Nil => println("empty")
```

**Answer:** 14
1 | List(2, 3, 4)

Enums define ADTs; :: destructures a list. Verified by compiling and running it.

**Q46.** What do extractors (unapply) allow?

**Answer:** Custom patterns in match expressions

Case classes get them for free.

**Q47.** What does a pattern guard (if ...) do?

**Answer:** Adds a boolean condition to a case

The case matches only if the guard holds.

**Q48.** What does the pipe in case 1 | 2 do?

**Answer:** Matches either alternative

Alternative patterns.

### [03 · Collections Deep Dive](level-2/03-collections-deep-dive.md)

**Q49.** What does this print?

```scala
@main def m() =
  val xs = List(1, 2, 3, 4, 5, 6)
  println(xs.groupBy(_ % 3).toList.sortBy(_._1))
  println(xs.partition(_ % 2 == 0))
  println(xs.sliding(3).map(_.sum).toList)
  println(xs.takeWhile(_ < 4) ++ xs.dropWhile(_ < 6))
  println((xs.reduce(_ max _), xs.scan(0)(_ + _)))
```

**Answer:** List((0,List(3, 6)), (1,List(1, 4)), (2,List(2, 5)))
(List(2, 4, 6),List(1, 3, 5))
List(6, 9, 12, 15)
List(1, 2, 3, 6)
(6,List(0, 1, 3, 6, 10, 15, 21))

groupBy returns a Map; sorting its entries gives stable output. Verified by compiling and running it.

**Q50.** What is the cost of appending to a List in a loop?

**Answer:** O(n) each time; build with prepend and reverse, or use a builder or Vector

Choose the right collection.

**Q51.** What does view do on a collection?

**Answer:** Makes transformations lazy so intermediate collections are not built

Useful for long chains.

**Q52.** What does .iterator give you?

**Answer:** A single-pass lazy traversal

Consumed once.

### [04 · Option/Either for Error Handling](level-2/04-option-either.md)

**Q53.** What does this print?

```scala
def parse(s: String): Option[Int] = s.toIntOption
def div(a: Int, b: Int): Either[String, Int] = if b == 0 then Left("div by zero") else Right(a / b)
@main def m() =
  println(parse("42").map(_ + 1))
  println(parse("x").getOrElse(-1))
  println(for a <- parse("10"); b <- parse("2") yield a / b)
  println(div(1, 0))
  println(div(8, 2).map(_ * 3))
  println(div(1, 0).left.map(_.toUpperCase))
  println(List("1", "x", "3").flatMap(parse))
```

**Answer:** Some(43)
-1
Some(5)
Left(div by zero)
Right(12)
Left(DIV BY ZERO)
List(1, 3)

Option and Either replace nulls and exceptions with values; flatMap drops Nones. Verified by compiling and running it.

**Q54.** Why prefer Option over null?

**Answer:** The type forces callers to handle absence

No NullPointerException from forgetting.

**Q55.** By convention, which side of Either holds the error?

**Answer:** Left (Right is the success)

Right-biased.

**Q56.** What does Try do?

**Answer:** Wraps code that may throw into Success or Failure

Converts exceptions into values.

### [05 · Testing with ScalaTest](level-2/05-scalatest-testing.md)

**Q57.** What does a ScalaTest FlatSpec/AnyFunSuite test look like?

**Answer:** A named block with assertions such as assert(a == b) or a shouldBe b

Run with sbt test.

**Q58.** What does sbt testOnly do?

**Answer:** Runs only matching test suites

Quick feedback.

**Q59.** Why prefer pure functions for testing?

**Answer:** No mocks or setup; same input gives same output

Easier reasoning.

**Q60.** What is a property-based test (ScalaCheck)?

**Answer:** A test that checks a property over many generated inputs

Failures shrink to a minimal case.

### [06 · Implicits Basics](level-2/06-implicits-basics.md)

**Q61.** What is an implicit/given value used for?

**Answer:** Supplying context or type class instances automatically at call sites

In Scala 3: given and using.

**Q62.** Which Scala 3 keyword replaces the implicit parameter list?

**Answer:** using

def f(using ord: Ordering[Int]).

**Q63.** What is a risk of overusing implicits?

**Answer:** Code that is hard to follow because behaviour comes from invisible imports

Keep them to type classes and context.

**Q64.** What does an extension method do?

**Answer:** Adds a method to an existing type without modifying it

extension (s: String) def shout = ...

### [07 · Working with JSON](level-2/07-working-with-json.md)

**Q65.** Which library style is common for JSON in Scala?

**Answer:** Libraries such as circe or play-json with typed encoders/decoders

Derive codecs from case classes.

**Q66.** What does decoding JSON into a case class give you?

**Answer:** Type-safe access and errors for missing or malformed fields

Return Either with errors.

**Q67.** Why not pattern match on raw JSON strings?

**Answer:** JSON is structured; use a parser to avoid fragile hacks

Parse, then validate.

**Q68.** What should you do with unexpected extra fields?

**Answer:** Decide explicitly: ignore for forward compatibility or reject for strict APIs

Document the choice.

### [08 · sbt Deep Dive](level-2/08-sbt-deep-dive.md)

**Q69.** What does libraryDependencies += ... do in sbt?

**Answer:** Adds a dependency to the project

Use %% for Scala-versioned artifacts.

**Q70.** What does %% mean in an sbt dependency?

**Answer:** Append the Scala binary version to the artifact name

Cross-built libraries.

**Q71.** What are sbt tasks and settings?

**Answer:** Settings hold values; tasks compute things like compile

Defined in build.sbt.

**Q72.** What does sbt assembly (plugin) produce?

**Answer:** A single fat jar with dependencies

For deployment.

### [09 · For-Comprehensions](level-2/09-for-comprehensions.md)

**Q73.** What does this print?

```scala
@main def m() =
  val r = for
    a <- List(1, 2)
    b <- List("x", "y")
    if a != 2 || b != "y"
  yield s"$a$b"
  println(r)
  val o = for
    x <- Some(3)
    y <- None: Option[Int]
  yield x + y
  println(o)
```

**Answer:** List(1x, 1y, 2x)
None

for-comprehensions desugar to flatMap, map and withFilter; one None short-circuits the whole thing. Verified by compiling and running it.

**Q74.** What does a for-comprehension desugar to?

**Answer:** flatMap, map and withFilter calls

Any type with those methods works.

**Q75.** Can you mix Option and List in one for?

**Answer:** Not directly; the generator types must be compatible (convert with toList)

Types must line up.

**Q76.** Why use for-comprehension instead of nested flatMap?

**Answer:** Readability

Same behaviour.

### [10 · Project — Weather CLI](level-2/10-project-weather-cli.md)

**Q77.** How do you test a weather CLI that calls an API?

**Answer:** Separate parsing from fetching and test parsing with sample JSON

Deterministic tests.

**Q78.** Where should errors from the API surface?

**Answer:** As Either or exceptions translated into a clear user message

Exit with non-zero.

**Q79.** How can you model temperature units safely?

**Answer:** An enum or sealed trait for units and a case class for the reading

Types prevent mix-ups.

**Q80.** Why cache responses briefly?

**Answer:** Reduce calls and latency

Respect API limits.

## Level 3 · Advanced

### [01 · Futures &amp; Concurrency](level-3/01-futures-concurrency.md)

**Q81.** What does this print?

```scala
import scala.concurrent.{Future, Await}
import scala.concurrent.ExecutionContext.Implicits.global
import scala.concurrent.duration.*
@main def m() =
  val f1 = Future { 20 }
  val f2 = Future { 22 }
  val sum = for a <- f1; b <- f2 yield a + b
  println(Await.result(sum, 2.seconds))
  val bad = Future { 1 / 0 }.recover { case _: ArithmeticException => -1 }
  println(Await.result(bad, 2.seconds))
```

**Answer:** 42
-1

Futures compose with for; recover turns a failure into a value. Verified by compiling and running it.

**Q82.** What is an ExecutionContext?

**Answer:** The thread pool on which futures run

Provide one explicitly.

**Q83.** Why avoid Await.result in production code?

**Answer:** It blocks a thread; compose futures instead

Use it at the edge or in tests.

**Q84.** What does Future.sequence do?

**Answer:** Turns a collection of futures into one future of a collection

Fails if any fails.

### [02 · Building APIs](level-3/02-building-apis.md)

**Q85.** What does an HTTP API framework such as http4s or Play provide?

**Answer:** Routing, request/response handling and JSON integration

Pick one suited to your style.

**Q86.** Which status code means 'created'?

**Answer:** 201

Return Location.

**Q87.** Why define request and response case classes?

**Answer:** Typed contracts and automatic codecs

Validate input.

**Q88.** How should errors be returned?

**Answer:** With the right status and a consistent JSON error body

Don't leak internals.

### [03 · Databases](level-3/03-databases.md)

**Q89.** What does Slick or Doobie provide?

**Answer:** Typed database access for Scala

Slick is FRM; Doobie is functional JDBC.

**Q90.** Why use connection pools?

**Answer:** Opening connections is expensive and limited

HikariCP is common.

**Q91.** How do you prevent SQL injection?

**Answer:** Parameterised queries or typed query builders

Never concatenate input.

**Q92.** What should DB calls return in a functional design?

**Answer:** Effects or Futures, with errors as values at the boundary

Keep IO explicit.

### [04 · Functional Programming Patterns](level-3/04-functional-patterns.md)

**Q93.** What is referential transparency?

**Answer:** An expression can be replaced by its value without changing behaviour

Enables local reasoning.

**Q94.** What is a monad (informally)?

**Answer:** A type with flatMap and unit that sequences computations with context

Option, Either, List and Future behave this way.

**Q95.** What does Functor's map guarantee?

**Answer:** Transforms contents while preserving structure

Laws: identity and composition.

**Q96.** Why push side effects to the edge?

**Answer:** A pure core is easier to test and reason about

Functional core, imperative shell.

### [05 · Testing Advanced](level-3/05-testing-advanced.md)

**Q97.** What is a mock used for?

**Answer:** Replacing a collaborator to control and verify interactions

Prefer fakes for simple cases.

**Q98.** Why use test containers for integration tests?

**Answer:** Real dependencies such as databases in disposable containers

Closer to production.

**Q99.** What is a flaky test?

**Answer:** One that sometimes fails without code changes

Fix root causes: time, ordering, shared state.

**Q100.** What does a test pyramid recommend?

**Answer:** Many unit tests, fewer integration and few end-to-end

Balance speed and confidence.

### [06 · Type Classes](level-3/06-type-classes.md)

**Q101.** What does this print?

```scala
trait Show[A]:
  def show(a: A): String
given Show[Int] with
  def show(a: Int) = s"Int($a)"
given Show[String] with
  def show(a: String) = s"Str($a)"
given [A](using s: Show[A]): Show[List[A]] with
  def show(xs: List[A]) = xs.map(s.show).mkString("[", ",", "]")
def render[A](a: A)(using s: Show[A]) = s.show(a)
@main def m() =
  println(render(5))
  println(render("hi"))
  println(render(List(1, 2)))
```

**Answer:** Int(5)
Str(hi)
[Int(1),Int(2)]

The compiler selects the right given by type, including derived instances for List. Verified by compiling and running it.

**Q102.** What is a type class?

**Answer:** A set of operations defined for a type separately from the type itself

Ad hoc polymorphism.

**Q103.** What advantage do type classes have over inheritance?

**Answer:** You can add behaviour to types you do not own

For example Ordering for any type.

**Q104.** What does Ordering[T] enable?

**Answer:** Sorting and comparison for T

sorted uses it implicitly.

### [07 · Higher-Kinded Types Intro](level-3/07-higher-kinded-types.md)

**Q105.** What is a higher-kinded type?

**Answer:** A type that takes a type constructor as a parameter, such as F[_]

Functor[F[_]].

**Q106.** Why are HKTs useful?

**Answer:** Abstracting over containers like Option, List and Future

Write generic code once.

**Q107.** What is the kind of Option?

**Answer:** * -&gt; * (it takes one type)

Option[Int] is a proper type.

**Q108.** What risk comes with heavy abstraction?

**Answer:** Hard-to-read code and slow onboarding; use it where it pays off

Balance.

### [08 · Performance &amp; Profiling](level-3/08-performance-profiling.md)

**Q109.** What is the first step in performance work?

**Answer:** Measure with a profiler or JMH before changing code

Find the bottleneck.

**Q110.** What is JMH?

**Answer:** A JVM microbenchmark harness that handles warm-up and dead-code elimination

Avoid naive timing loops.

**Q111.** Why can boxing hurt performance?

**Answer:** Primitive values wrapped in objects cause allocation and indirection

Specialisation and arrays help.

**Q112.** What does tail recursion with @tailrec guarantee?

**Answer:** The compiler turns it into a loop or errors

Avoids stack overflow.

### [09 · Akka Actors Basics](level-3/09-akka-actors-basics.md)

**Q113.** What does an actor encapsulate?

**Answer:** State and behaviour, communicating only by asynchronous messages

No shared mutable state.

**Q114.** What is a mailbox?

**Answer:** The queue of messages waiting for an actor

Processed one at a time.

**Q115.** What is 'let it crash' supervision?

**Answer:** A parent actor restarts or stops failing children by policy

Fault isolation.

**Q116.** What is the cost of messages to actors?

**Answer:** Asynchrony: results arrive later, so design for it

Use ask carefully.

### [10 · Project — REST API + Database Service](level-3/10-project-rest-api-db.md)

**Q117.** In the REST API project, where should SQL live?

**Answer:** In a repository layer separate from routes

Easier to test.

**Q118.** How do you test routes without a server?

**Answer:** Call the routes with a request in-memory

Fast tests.

**Q119.** How should migrations be managed?

**Answer:** Versioned migration scripts (Flyway or similar)

Reproducible schema.

**Q120.** What should the service log?

**Answer:** Structured request and error logs with correlation IDs

Help debugging.

## Level 4 · Master

### [01 · Advanced Functional Programming](level-4/01-advanced-functional-programming.md)

**Q121.** What is the Free monad idea?

**Answer:** Describing a program as data and interpreting it separately

Separate description from execution.

**Q122.** What does tagless final do?

**Answer:** Writes code abstractly over F[_] and picks an effect type at the end

Common with cats-effect.

**Q123.** What is a Resource in cats-effect?

**Answer:** A safely acquired and released value (bracket pattern)

Cleanup even on error.

**Q124.** Why use IO-style effects?

**Answer:** Referential transparency and explicit control of side effects and concurrency

Compose descriptions of effects.

### [02 · Distributed Systems with Akka](level-4/02-distributed-systems-akka.md)

**Q125.** What is the CAP theorem's message?

**Answer:** During a partition, you must choose between consistency and availability

Design accordingly.

**Q126.** What is idempotency in messaging?

**Answer:** Processing the same message twice has the same effect as once

At-least-once delivery needs it.

**Q127.** What does a circuit breaker do?

**Answer:** Stops calling a failing dependency for a while to let it recover

Protect resources.

**Q128.** What is event sourcing?

**Answer:** Storing changes as an append-only log of events, deriving state by replay

Auditable.

### [03 · Production APIs](level-4/03-production-apis.md)

**Q129.** What belongs in a production API checklist?

**Answer:** Validation, auth, rate limiting, timeouts, metrics and structured logs

Beyond happy paths.

**Q130.** Why set timeouts everywhere?

**Answer:** A hung dependency would otherwise block resources

Also retries with backoff.

**Q131.** What is a health endpoint for?

**Answer:** Orchestrators need to know whether to route traffic or restart

Liveness vs readiness.

**Q132.** What is graceful shutdown?

**Answer:** Finish in-flight requests on SIGTERM before exiting

Avoid dropped requests.

### [04 · Security Best Practices](level-4/04-security-best-practices.md)

**Q133.** How should secrets be handled?

**Answer:** Injected via environment or a secret manager, never committed

Rotate them.

**Q134.** What stops SQL injection?

**Answer:** Parameterised queries

Use bind variables.

**Q135.** Why hash passwords with bcrypt/argon2?

**Answer:** Slow, salted hashes resist brute force

Never reversible encryption.

**Q136.** Why run dependency vulnerability scans?

**Answer:** Libraries carry known CVEs

Automate in CI.

### [05 · Testing at Scale &amp; CI](level-4/05-testing-at-scale-ci.md)

**Q137.** What speeds up Scala CI builds?

**Answer:** Caching dependencies and incremental compilation

Keep feedback fast.

**Q138.** What does cross-building mean?

**Answer:** Compiling and testing against several Scala versions

Libraries need it.

**Q139.** What coverage should you chase?

**Answer:** Meaningful tests of behaviour, not a number for its own sake

Coverage is a signal.

**Q140.** What should you do with failing flaky tests in CI?

**Answer:** Quarantine and fix the cause

Track them.

### [06 · Deployment (Docker)](level-4/06-deployment-docker.md)

**Q141.** Why multi-stage Docker builds for Scala?

**Answer:** Build with sbt in one stage and ship only the jar on a small JRE image

Smaller images.

**Q142.** Why set JVM memory flags in containers?

**Answer:** The JVM must respect container memory limits

Use container-aware defaults.

**Q143.** What does sbt-native-packager do?

**Answer:** Builds Docker images and packages

Convenience.

**Q144.** Why run as non-root?

**Answer:** Limits damage if compromised

Least privilege.

### [07 · Performance at Scale](level-4/07-performance-at-scale.md)

**Q145.** What should you do before tuning at scale?

**Answer:** Profile production-like load to find hot spots

Measure first.

**Q146.** What causes GC pauses?

**Answer:** Large allocation rates and heaps; tune collectors and reduce garbage

G1 or ZGC.

**Q147.** What is backpressure?

**Answer:** Signalling producers to slow down when consumers can't keep up

Streams provide it.

**Q148.** Why prefer bounded queues?

**Answer:** Unbounded queues hide overload until memory runs out

Fail fast.

### [08 · Scala 3 Features](level-4/08-scala-3-features.md)

**Q149.** What does this Scala 3 snippet print?

```scala
enum Color(val hex: String):
  case Red extends Color("#f00")
  case Green extends Color("#0f0")
opaque type UserId = Long
object UserId:
  def apply(n: Long): UserId = n
  extension (u: UserId) def value: Long = u
@main def m() =
  println(Color.values.map(_.hex).mkString(","))
  println(UserId(7).value + 1)
  val xs = List(1, 2, 3)
  println(xs.map(x => x * 2).sum)
  println(if xs.nonEmpty then "yes" else "no")
```

**Answer:** #f00,#0f0
8
12
yes

Scala 3 enums can carry parameters; opaque types give zero-cost distinct types. Verified by compiling and running it.

**Q150.** What does the indentation-based syntax in Scala 3 replace?

**Answer:** Many curly braces (they remain optional)

Use consistent indentation.

**Q151.** What are union types?

**Answer:** A type like Int | String meaning either

Replace some Either uses.

**Q152.** What is the purpose of given/using?

**Answer:** A clearer replacement for implicits

Context abstractions.

### [09 · Build Tooling at Scale](level-4/09-build-tooling-at-scale.md)

**Q153.** What is a multi-module sbt build for?

**Answer:** Splitting code into modules with separate dependencies

core, api, cli.

**Q154.** Why share common settings in one place?

**Answer:** Consistency across modules

ThisBuild settings.

**Q155.** What does a lockfile or dependency pinning give?

**Answer:** Reproducible builds

Avoid surprise upgrades.

**Q156.** What is incremental compilation?

**Answer:** Recompiling only changed files and dependents

Zinc does this.

### [10 · Capstone Project](level-4/10-capstone-project.md)

**Q157.** What is the capstone's goal?

**Answer:** A production-style Scala service with tests, CI, security, observability and a container image

Show readiness.

**Q158.** What belongs in the README?

**Answer:** Setup, run, test, deploy and design decisions

Reviewers start there.

**Q159.** What evidence proves readiness?

**Answer:** CI passing, tests, security scan, load test and runbook

Evidence.

**Q160.** Why add metrics and structured logs?

**Answer:** To see failures and latency in production

Observability.

## Mixed questions across all levels

**Q161.** What does this print?

```scala
@main def m() =
  val xs = List(1, 2, 3, 4)
  println(xs.foldLeft("")((acc, x) => acc + x))
  println(xs.foldRight("")((x, acc) => acc + x))
  println(xs.map(_ + 1).filter(_ % 2 == 0).sum)
```

**Answer:** 1234
4321
6

foldLeft processes left to right; foldRight starts from the right. Verified by compiling and running it.

**Q162.** What does this print?

```scala
sealed trait Shape
case class Circle(r: Double) extends Shape
case class Sq(s: Double) extends Shape
def area(s: Shape) = s match
  case Circle(r) => 3 * r * r
  case Sq(x) => x * x
@main def m() =
  println(List(Circle(1), Sq(2)).map(area))
  println(Circle(1) == Circle(1.0))
```

**Answer:** List(3.0, 4.0)
true

Case classes compare structurally; the match on a sealed trait is checked for exhaustiveness. Verified by compiling and running it.

**Q163.** What does this print?

```scala
@main def m() =
  val r = scala.util.Try("12x".toInt)
  println(r.isFailure)
  println(r.getOrElse(0))
  println(scala.util.Try(10 / 2).toOption)
  println("12".toIntOption.map(_ * 2))
```

**Answer:** true
0
Some(5)
Some(24)

Try captures exceptions as values; toIntOption avoids them altogether. Verified by compiling and running it.

**Q164.** What does this print?

```scala
def counter(): () => Int =
  var n = 0
  () => { n += 1; n }
@main def m() =
  val c = counter()
  c(); c()
  println(c())
  val d = counter()
  println(d())
```

**Answer:** 3
1

Each call to counter creates its own captured variable. Verified by compiling and running it.

**Q165.** What does this print?

```scala
@main def m() =
  val words = List("apple", "bob", "cat", "apple", "bob", "apple")
  val freq = words.groupMapReduce(identity)(_ => 1)(_ + _)
  println(freq.toList.sortBy((w, n) => (-n, w)))
  println(words.distinct.size)
```

**Answer:** List((apple,3), (bob,2), (cat,1))
3

groupMapReduce groups, maps and reduces in one pass. Verified by compiling and running it.

**Q166.** Why make domain types sealed ADTs?

**Answer:** The compiler checks that matches are exhaustive, making illegal states unrepresentable

New variants surface as warnings.

**Q167.** When should you use Either rather than throwing an exception?

**Answer:** For expected, recoverable failures that callers must handle

Exceptions are for the exceptional.

**Q168.** What does Future.traverse do?

**Answer:** Applies a function returning futures to each element and combines the results into one future

Parallel when the futures start eagerly.

**Q169.** Why avoid Await.result inside request handlers?

**Answer:** It blocks threads and can exhaust the pool

Compose futures instead.

**Q170.** What is the idiomatic way to share state between actors?

**Answer:** Send immutable messages; do not share mutable state

The actor model.

**Q171.** What does a type class like Ordering[A] allow?

**Answer:** Behaviour for a type chosen by an implicit/given instance, even for types you do not own

Ad hoc polymorphism.

**Q172.** What is the purpose of opaque types in Scala 3?

**Answer:** Distinct, zero-cost wrapper types such as UserId over Long

Avoid mixing IDs.

**Q173.** What is the point of idempotency for message handlers?

**Answer:** At-least-once delivery means a message may be processed twice

Use unique keys.

**Q174.** What does a circuit breaker protect?

**Answer:** Your service and a failing dependency by temporarily stopping calls

Fail fast.

**Q175.** How should secrets reach a Scala service in production?

**Answer:** Through environment or a secret manager, never committed

Rotate.

**Q176.** Why use a multi-stage Docker build?

**Answer:** Compile with sbt in one stage, ship only the jar on a slim JRE image

Smaller images.

**Q177.** What does sbt %% do?

**Answer:** Appends the Scala binary version to the artifact name

Cross-versioned artifacts.

**Q178.** What evidence shows a Scala service is production-ready?

**Answer:** CI with tests and scans, load testing, metrics, logs and a runbook

Evidence over claims.

**Q179.** Which collection suits many appends and random reads?

**Answer:** Vector

List excels at prepend.

## Practical coding questions

### An inventory with typed errors

Implement apply and parse for a small inventory: commands update an immutable Map, failures are values (Either) and the fold keeps the last good stock when a command fails. This is ordinary Scala 3 code you run on your own machine.

Starter code:

```
enum Cmd:
  case Add(sku: String, qty: Int)
  case Remove(sku: String, qty: Int)

enum InvError:
  case BadQty(n: Int)
  case Unknown(sku: String)
  case Insufficient(sku: String, have: Int, want: Int)

import Cmd.*, InvError.*

type Stock = Map[String, Int]

// TODO: def apply(stock: Stock, cmd: Cmd): Either[InvError, Stock]
//   Add / Remove with qty <= 0 -> Left(BadQty(qty)); Remove of a missing sku -> Left(Unknown(sku));
//   Remove of more than available -> Left(Insufficient(sku, have, want)); otherwise Right(updated stock)
// TODO: def parse(line: String): Option[Cmd]   ("add apple 10" / "remove apple 3"; anything else, including non-numeric qty, is None)

@main def run() =
  val lines = List("add apple 10", "add pear 4", "remove apple 3", "remove kiwi 1", "remove pear 9", "add fig 0", "oops", "add apple 5")
  val (finalStock, errors) = lines.foldLeft((Map.empty: Stock, List.empty[String])) { case ((st, errs), line) =>
    parse(line) match
      case None => (st, errs :+ s"unparsable: $line")
      case Some(c) => apply(st, c) match
        case Right(s2) => (s2, errs)
        case Left(e) => (st, errs :+ e.toString)
  }
  println(finalStock.toList.sortBy(_._1))
  errors.foreach(println)
  println(s"errors=${errors.size}")
```

Expected output:

```text
List((apple,12), (pear,4))
Unknown(kiwi)
Insufficient(pear,4,9)
BadQty(0)
unparsable: oops
errors=4
```

??? success "Show a solution"

    ```
    enum Cmd:
      case Add(sku: String, qty: Int)
      case Remove(sku: String, qty: Int)

    enum InvError:
      case BadQty(n: Int)
      case Unknown(sku: String)
      case Insufficient(sku: String, have: Int, want: Int)

    import Cmd.*, InvError.*

    type Stock = Map[String, Int]

    def apply(stock: Stock, cmd: Cmd): Either[InvError, Stock] = cmd match
      case Add(sku, q) if q <= 0 => Left(BadQty(q))
      case Add(sku, q) => Right(stock.updated(sku, stock.getOrElse(sku, 0) + q))
      case Remove(sku, q) if q <= 0 => Left(BadQty(q))
      case Remove(sku, q) =>
        stock.get(sku) match
          case None => Left(Unknown(sku))
          case Some(h) if h < q => Left(Insufficient(sku, h, q))
          case Some(h) => Right(stock.updated(sku, h - q))

    def parse(line: String): Option[Cmd] = line.split(" ").toList match
      case "add" :: sku :: q :: Nil => q.toIntOption.map(Add(sku, _))
      case "remove" :: sku :: q :: Nil => q.toIntOption.map(Remove(sku, _))
      case _ => None

    @main def run() =
      val lines = List("add apple 10", "add pear 4", "remove apple 3", "remove kiwi 1", "remove pear 9", "add fig 0", "oops", "add apple 5")
      val (finalStock, errors) = lines.foldLeft((Map.empty: Stock, List.empty[String])) { case ((st, errs), line) =>
        parse(line) match
          case None => (st, errs :+ s"unparsable: $line")
          case Some(c) => apply(st, c) match
            case Right(s2) => (s2, errs)
            case Left(e) => (st, errs :+ e.toString)
      }
      println(finalStock.toList.sortBy(_._1))
      errors.foreach(println)
      println(s"errors=${errors.size}")
    ```

    Each command returns Either, so errors never throw. The foldLeft threads (stock, errors) through, keeping the previous stock when a command is rejected. Verified by compiling and running with Scala 3.3.1.

### Parallel word count with retries

Count words in several documents concurrently with Futures, retrying a flaky step with recoverWith, then merge the maps. Futures run on the global execution context; Await is used only at the program edge so the result can be printed.

Starter code:

```
import scala.concurrent.{Await, Future}
import scala.concurrent.ExecutionContext.Implicits.global
import scala.concurrent.duration.*

val docs = Vector(
  "the quick brown fox",
  "the lazy dog and the fox",
  "quick quick slow",
  "a dog a fox a cat"
)

// TODO: def countWords(doc: String): Map[String, Int]
// TODO: def merge(a: Map[String, Int], b: Map[String, Int]): Map[String, Int]   (sum counts)
// TODO: def withRetry(doc: String, max: Int): Future[Map[String, Int]]   (retry flaky(doc, n) with n = 0, 1, ... up to max)
def flaky(doc: String, attempts: Int): Future[Map[String, Int]] =
  Future { if doc.startsWith("quick") && attempts < 2 then throw RuntimeException("busy") else countWords(doc) }

@main def run() =
  val total = Await.result(Future.traverse(docs)(d => withRetry(d, 3)).map(_.reduce(merge)), 5.seconds)
  println(total.toList.sortBy((w, n) => (-n, w)).take(4))
  println(total.values.sum)
  val failing = Await.result(withRetry("quick fail", 1).failed.map(_.getMessage), 2.seconds)
  println(failing)
```

Expected output:

```text
List((a,3), (fox,3), (quick,3), (the,3))
19
busy
```

??? success "Show a solution"

    ```
    import scala.concurrent.{Await, Future}
    import scala.concurrent.ExecutionContext.Implicits.global
    import scala.concurrent.duration.*

    val docs = Vector(
      "the quick brown fox",
      "the lazy dog and the fox",
      "quick quick slow",
      "a dog a fox a cat"
    )

    def countWords(doc: String): Map[String, Int] =
      doc.split(" ").toList.groupBy(identity).view.mapValues(_.size).toMap

    def merge(a: Map[String, Int], b: Map[String, Int]): Map[String, Int] =
      b.foldLeft(a) { case (acc, (w, n)) => acc.updated(w, acc.getOrElse(w, 0) + n) }

    def flaky(doc: String, attempts: Int): Future[Map[String, Int]] =
      Future { if doc.startsWith("quick") && attempts < 2 then throw RuntimeException("busy") else countWords(doc) }

    def withRetry(doc: String, max: Int): Future[Map[String, Int]] =
      def go(n: Int): Future[Map[String, Int]] =
        flaky(doc, n).recoverWith { case _ if n < max => go(n + 1) }
      go(0)

    @main def run() =
      val total = Await.result(Future.traverse(docs)(d => withRetry(d, 3)).map(_.reduce(merge)), 5.seconds)
      println(total.toList.sortBy((w, n) => (-n, w)).take(4))
      println(total.values.sum)
      val failing = Await.result(withRetry("quick fail", 1).failed.map(_.getMessage), 2.seconds)
      println(failing)
    ```

    Future.traverse starts all documents, recoverWith retries a failing future up to max times and merge sums counts. A document that keeps failing surfaces its exception as a failed future. Verified by compiling and running with Scala 3.3.1.

---

More free practice: the [Scala Mastery Path](index.md) curriculum, the [Interview Prep Pack](https://learnmasterypath.com/interview-pack/) and the [free resume ATS check](https://learnmasterypath.com/resume/).
