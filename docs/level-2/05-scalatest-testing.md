# 05 · Testing with ScalaTest

Every function you've written so far has been "tested" by eyeballing
`println` output — fine for a lesson, unworkable for a real project. This
module introduces [ScalaTest](https://www.scalatest.org/), the most widely
used testing library in the Scala ecosystem, and shows the two styles
you'll see most often in real codebases: `AnyFunSuite` (xUnit-flavored,
JUnit-like) and `AnyFlatSpec` (BDD-flavored, reads like a sentence).

## Adding ScalaTest to a project

ScalaTest is a library dependency, added in `build.sbt`
(see [Module 8 · sbt Deep Dive](08-sbt-deep-dive.md) for a full tour of
`build.sbt`):

```scala
// build.sbt
libraryDependencies += "org.scalatest" %% "scalatest" % "3.2.19" % Test
```

The `% Test` at the end scopes the dependency to the `Test` configuration
only — it's available in `src/test/scala`, but never bundled into the
program you actually ship. Test files themselves live under
`src/test/scala`, mirroring the `src/main/scala` layout from the
[Level 1 project](../level-1/10-project-todo-app.md).

## `AnyFunSuite`: xUnit style

`AnyFunSuite` names each test with a plain string and a block of assertions
— the closest match to JUnit if you've used it before:

```scala
// src/test/scala/MathUtilsSuite.scala
import org.scalatest.funsuite.AnyFunSuite

object MathUtils:
  def add(a: Int, b: Int): Int = a + b
  def divide(a: Int, b: Int): Int =
    if b == 0 then throw new ArithmeticException("/ by zero") else a / b

class MathUtilsSuite extends AnyFunSuite:

  test("add combines two numbers") {
    assert(MathUtils.add(2, 3) == 5)
  }

  test("add handles negative numbers") {
    assert(MathUtils.add(-2, 5) == 3)
  }

  test("divide throws on zero denominator") {
    assertThrows[ArithmeticException] {
      MathUtils.divide(10, 0)
    }
  }
```

Run it with `sbt test` (or `sbt "testOnly MathUtilsSuite"` for just this
one). Each `test("...")` block is an independent unit — one failing
`assert` fails that test but doesn't stop the others from running.

## `AnyFlatSpec`: BDD style

`AnyFlatSpec` reads almost like an English sentence — `"X" should "do Y" in
{ ... }` — which tends to produce more descriptive failure output for
larger suites, at the cost of slightly more ceremony:

```scala
import org.scalatest.flatspec.AnyFlatSpec
import org.scalatest.matchers.should.Matchers

class MathUtilsSpec extends AnyFlatSpec with Matchers:

  "add" should "combine two positive numbers" in {
    MathUtils.add(2, 3) shouldBe 5
  }

  "add" should "handle negative numbers" in {
    MathUtils.add(-2, 5) shouldBe 3
  }

  "divide" should "throw ArithmeticException on zero" in {
    an [ArithmeticException] should be thrownBy MathUtils.divide(10, 0)
  }
```

Mixing in `Matchers` unlocks the `shouldBe`/`should` DSL used above — it's
what makes `AnyFlatSpec` read naturally, and you can mix it into
`AnyFunSuite` too if you like the matcher syntax but prefer the simpler
`test("...")` block structure.

## Matchers: readable assertions

`Matchers` provides many small, readable checks beyond plain equality:

```scala
class MatcherExamplesSpec extends AnyFlatSpec with Matchers:

  "a matcher" should "check equality, collections, and options" in {
    val nums = List(1, 2, 3)

    5 shouldBe 5
    5 should equal(5)
    nums should contain(2)
    nums should have length 3
    nums shouldBe sorted
    Some(4) shouldBe defined
    None shouldBe empty
    "hello" should startWith("he")
    "hello" should endWith("lo")
  }
```

## The trap: testing only the happy path

It's tempting to write one passing test per function and call it done. The
tests that actually catch bugs are the edge cases: empty input, zero,
negative numbers, `None`/`Left` branches. Since Module 4 modeled failure
as `Option`/`Either` rather than exceptions, testing those branches is just
another assertion, not a special case:

```scala
def half(n: Int): Option[Int] =
  if n % 2 == 0 then Some(n / 2) else None

class HalfSpec extends AnyFlatSpec with Matchers:

  "half" should "divide even numbers" in {
    half(10) shouldBe Some(5)
  }

  "half" should "return None for odd numbers" in {
    half(7) shouldBe None
  }
```

A test suite that never exercises the `None`/`Left`/failure path is only
half testing the function — usually the more bug-prone half, since it's the
branch developers reread the least.

## Fixtures: setup shared across tests

For state that needs to be reset before every test (a fresh instance, a
temp file, a mutable collection), mix in `BeforeAndAfterEach` and override
`beforeEach`:

```scala
import org.scalatest.BeforeAndAfterEach
import scala.collection.mutable.ListBuffer

class BufferSpec extends AnyFlatSpec with Matchers with BeforeAndAfterEach:

  val buffer = ListBuffer.empty[Int]

  override def beforeEach(): Unit =
    buffer.clear()
    buffer += 1
    buffer += 2

  "the buffer" should "start with two seeded elements" in {
    buffer should have size 2
  }

  "the buffer" should "allow appending" in {
    buffer += 3
    buffer should have size 3
  }
```

Each test sees a freshly reset `buffer`, so append side effects in one test
never leak into another — a common source of order-dependent, flaky test
suites when fixtures are shared without a reset.

## How It Actually Works

ScalaTest's `AnyFunSuite` and `AnyFlatSpec` look like two different DSLs,
but both compile down to the same underlying model: each `test("...") {
... }` or `"it" should "..." in { ... }` block is registered, at
*construction time* (when the test class's constructor runs), into an
internal list of test cases held by the suite instance — the string
becomes a key, and the block becomes a closure (compiled to a `Function0`,
see [Module 4's functions-as-values](../level-1/04-functions.md)) stored
against it. Running the suite doesn't re-read your source; it just
iterates that already-built list and invokes each closure, catching any
thrown exception (including a failed `assert` or matcher, which work by
throwing a `TestFailedException`) to report pass/fail without one test's
exception aborting the whole run.

Matchers like `result should be(5)` and `list should contain(3)` are
ordinary method calls chained together using Scala's ability to define
methods with symbolic or word-like names and to omit dots/parens in
infix position: `should` is a method on an implicit wrapper class around
`result`, taking `be(5)` — itself a small "matcher" object — as its
argument. There's no keyword `should` in the language; it's a library
convention built entirely from implicit conversions (see [Module
6](06-implicits-basics.md)) and Scala's flexible method-call syntax, which
is also why writing your own custom matcher is "just" implementing the
right trait rather than hooking into compiler internals.

Fixtures that recreate state per test (rather than sharing one mutable
instance across tests) matter because sbt normally runs all tests in a
suite against **one shared instance** of that suite class by default —
if a `var` were declared directly in the class body instead of freshly set
up per test via a fixture method, test order could leak state between
tests non-deterministically, which is precisely the bug fixtures are
designed to make structurally impossible.

## Cheat sheet

| Style | Test declared as | Reads like |
|---|---|---|
| `AnyFunSuite` | `test("description") { ... }` | JUnit-style, plain description |
| `AnyFlatSpec` | `"subject" should "behavior" in { ... }` | a sentence |

| Matcher | Checks |
|---|---|
| `x shouldBe y` | equality (also works for `Some`/`None`/booleans) |
| `x should equal(y)` | equality, allows custom equality overrides |
| `xs should contain(y)` | collection membership |
| `xs should have length n` / `have size n` | collection size |
| `xs shouldBe sorted` | ordering |
| `opt shouldBe defined` / `shouldBe empty` | `Option` state |
| `an [Ex] should be thrownBy expr` | exception assertion (`AnyFlatSpec`) |
| `assertThrows[Ex] { expr }` | exception assertion (any style) |

## Exercise

Write a small `Stack[Int]` class (backed by a `List`, with `push`, `pop:
Option[Int]` returning `None` on an empty stack, and `peek: Option[Int]`)
in `src/main/scala`. Then write `StackSpec` using `AnyFlatSpec with
Matchers` in `src/test/scala` covering at least: pushing then popping
returns the last-pushed value, popping an empty stack returns `None`,
and `peek` doesn't remove the element. Run the suite with `sbt test` and
confirm all tests pass.
