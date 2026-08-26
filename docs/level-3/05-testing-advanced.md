# 05 · Testing Advanced

[Level 2](../level-2/05-scalatest-testing.md) covered writing individual
example-based tests with ScalaTest. This module adds two techniques for
when hand-picked examples aren't enough: **property-based testing** with
ScalaCheck, which generates hundreds of inputs automatically, and
**mocking** dependencies with Mockito so you can test one class in
isolation from the collaborators it calls.

```scala title="build.sbt"
scalaVersion := "3.3.1"
libraryDependencies += "org.scalatest" %% "scalatest" % "3.2.19" % Test
libraryDependencies += "org.scalatestplus" %% "scalacheck-1-17" % "3.2.18.0" % Test
libraryDependencies += "org.mockito" % "mockito-core" % "5.11.0" % Test
```

## Property-based testing with ScalaCheck

Example-based tests check specific inputs you thought of. A **property**
describes something that should hold for *every* input, and ScalaCheck
generates a wide range of random inputs (including edge cases like empty
strings) to try to falsify it:

```scala
// src/main/scala/Lib.scala
def reverseString(s: String): String = s.reverse
```

```scala
// src/test/scala/LibSpec.scala
import org.scalatest.funsuite.AnyFunSuite
import org.scalatestplus.scalacheck.ScalaCheckPropertyChecks
import org.scalacheck.Gen

class LibSpec extends AnyFunSuite with ScalaCheckPropertyChecks:
  test("reversing twice returns the original string") {
    forAll(Gen.alphaStr) { s =>
      assert(reverseString(reverseString(s)) == s)
    }
  }
```

```text title="sbt test"
[info] LibSpec:
[info] - reversing twice returns the original string
[info] Tests: succeeded 2, failed 0, canceled 0, ignored 0, pending 0
```

`forAll(Gen.alphaStr)` runs the block 100 times by default with different
generated strings each time. `Gen` has building blocks for most shapes:
`Gen.alphaStr`, `Gen.choose(1, 100)`, `Gen.listOf(Gen.posNum[Int])`, and
`Gen.oneOf(a, b, c)` for picking among fixed alternatives. You describe
*what kind* of input to generate; ScalaCheck handles picking specific
values (including nasty ones like empty lists or `Int.MinValue`).

### The trap: properties that are trivially true

`forAll(Gen.alphaStr) { s => assert(s == s) }` passes 100/100 runs and
tells you nothing. A useful property encodes an actual invariant of your
code — "reversing twice is a no-op," "sorting is idempotent," "encoding
then decoding returns the original" — not a tautology. If you can't state
what should always be true about a function beyond "it doesn't crash,"
example-based tests are probably the better fit for it.

## Mocking with Mockito

Testing `OrderCalculator` shouldn't require a real price database — only a
*stand-in* for `PriceService` that returns canned answers, so the test
exercises `OrderCalculator`'s logic in isolation:

```scala
// src/main/scala/Lib.scala
trait PriceService:
  def priceOf(sku: String): Double

class OrderCalculator(prices: PriceService):
  def total(skus: List[String]): Double = skus.map(prices.priceOf).sum
```

```scala
import org.mockito.Mockito._

test("OrderCalculator sums mocked prices") {
  val mockPrices = mock(classOf[PriceService])
  when(mockPrices.priceOf("A")).thenReturn(10.0)
  when(mockPrices.priceOf("B")).thenReturn(5.0)

  val calc = new OrderCalculator(mockPrices)
  assert(calc.total(List("A", "B", "A")) == 25.0)

  verify(mockPrices, times(2)).priceOf("A")   // called twice, as expected
}
```

`mock(classOf[PriceService])` creates a fake implementation where every
method returns a default (`0.0`, `null`, etc.) unless you stub it with
`when(...).thenReturn(...)`. `verify` then asserts something about *how*
the mock was called — here, that `priceOf("A")` was invoked exactly twice,
matching the two `"A"`s in the input list.

### The trap: mocking everything, testing nothing

Mocks are for **dependencies your test doesn't care about** — a database, a
network client, a clock. Mocking the class under test itself, or mocking so
much of a collaborator's behavior that the test just re-describes the
implementation (`when(mock.compute(1)).thenReturn(2)` in a test literally
titled "compute doubles its input"), produces a test that passes even after
the real logic breaks, because it's only checking that the mock does what
you told it to do. Prefer a real, cheap dependency (an in-memory list, the
in-memory SQLite from the previous module) over a mock whenever one is
available — mock what's genuinely expensive or non-deterministic (network
calls, wall-clock time, external services) and nothing else.

## Combining both

Properties and mocks aren't mutually exclusive — you can generate random
inputs for a class that itself depends on a mock:

```scala
test("total is always non-negative for non-negative prices") {
  forAll(Gen.listOf(Gen.oneOf("A", "B"))) { skus =>
    val mockPrices = mock(classOf[PriceService])
    when(mockPrices.priceOf("A")).thenReturn(10.0)
    when(mockPrices.priceOf("B")).thenReturn(5.0)
    val calc = new OrderCalculator(mockPrices)
    assert(calc.total(skus) >= 0.0)
  }
}
```

## Cheat sheet

| Need to... | Use |
|---|---|
| Generate random test inputs | `Gen.alphaStr`, `Gen.choose(lo, hi)`, `Gen.listOf(gen)`, `Gen.oneOf(...)` |
| Run a block against many generated inputs | `forAll(gen) { x => assert(...) }` (mix in `ScalaCheckPropertyChecks`) |
| Create a fake dependency | `mock(classOf[Trait])` |
| Make it return canned values | `when(mock.method(args)).thenReturn(value)` |
| Assert how a mock was called | `verify(mock, times(n)).method(args)` |
| Decide mock vs. real | mock expensive/non-deterministic externals; use a real cheap object otherwise |

## Exercise

Write a property for `OrderCalculator.total` stating that adding one more
SKU to the list never decreases the total, given non-negative mocked
prices — generate a `List[String]` and one extra SKU with ScalaCheck, mock
prices for all SKUs involved, and assert `calc.total(skus :+ extra) >=
calc.total(skus)`. Then write a mock-based test for a `NotificationService`
trait with a `send(msg: String): Boolean` method: stub it to fail on the
first call and succeed on a retry, and verify a `ReliableSender` class you
write calls `send` twice when the first attempt returns `false`.
