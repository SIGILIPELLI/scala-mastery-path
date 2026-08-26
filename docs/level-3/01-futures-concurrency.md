# 01 · Futures & Concurrency

Everything so far has run on one thread, one step after another. Real
services need to do several things at once — call a database while calling
an API, handle a hundred requests concurrently instead of one at a time.
Scala's answer is `Future[A]`: a value that represents a computation that
may not have finished yet.

## `Future` and `ExecutionContext`

A `Future` needs somewhere to actually run its work — a thread pool called
an `ExecutionContext`. Almost every `Future`-returning call needs one in
scope, so it's usually imported once at the top of a file:

```scala
import scala.concurrent.{Future, Await}
import scala.concurrent.ExecutionContext.Implicits.global
import scala.concurrent.duration._
import scala.util.{Success, Failure}

def slowSquare(n: Int): Future[Int] = Future {
  Thread.sleep(100)
  n * n
}
```

Calling `slowSquare(5)` returns *immediately* with a `Future[Int]` — the
`Thread.sleep` runs on a background thread from `global`'s pool. The caller
doesn't block; it gets a placeholder for a value that will show up later.

## Reacting when it's done: `onComplete`

```scala
val f = slowSquare(5)
f.onComplete {
  case Success(v) => println(s"got $v")
  case Failure(e) => println(s"failed: $e")
}
Thread.sleep(200) // just so this demo doesn't exit before the callback fires
// got 25
```

`onComplete` registers a callback and returns immediately — it never blocks
the calling thread. In a real application you'd chain further work with
`map`/`flatMap` instead of sleeping to "wait" for a side-effecting callback;
the sleep above exists only so this script's output is deterministic.

## Blocking when you must: `Await.result`

Tests, `main` methods, and REPL sessions sometimes need an actual value
before moving on. `Await.result` blocks the calling thread until the
`Future` completes or a timeout elapses:

```scala
val f2 = slowSquare(3)
val result = Await.result(f2, 2.seconds)
println(s"awaited: $result")   // awaited: 9
```

### The trap: `Await` in production code

`Await.result` defeats the entire purpose of `Future` — it ties up a thread
doing nothing but waiting, which is exactly what async code exists to avoid.
Under load, blocking threads like this is how services exhaust their thread
pool and grind to a halt. Reserve `Await` for `main` entry points, tests,
and REPL exploration; production request-handling code should stay
`Future`-based end to end via `map`/`flatMap`/`for`.

## Composing Futures with `for`

Because `Future` has `map` and `flatMap`, `for`-comprehensions chain
several async steps into one that runs them and combines the results:

```scala
val combined = for
  a <- slowSquare(2)
  b <- slowSquare(4)
yield a + b

println(Await.result(combined, 2.seconds))   // 20
```

### The trap: sequential `for` vs. parallel start

That `for`-comprehension runs `slowSquare(2)` and *then* `slowSquare(4)` —
roughly 200ms total, not 100ms. Desugared, a `for` calls `.flatMap` on the
first `Future`, and the second `Future` isn't even created until the first
one's callback runs. To run independent work concurrently, start every
`Future` *before* the `for`-comprehension so both begin immediately:

```scala
val fa = slowSquare(2)   // starts now
val fb = slowSquare(4)   // starts now, concurrently with fa
val bothStarted = for
  a <- fa
  b <- fb
yield a + b
// ~100ms total, because fa and fb were already running in parallel
```

## Handling failure

A `Future` whose block throws doesn't crash the program — the exception is
captured as a `Failure` inside the `Future` itself:

```scala
val failing = Future { throw new RuntimeException("boom") }
try Await.result(failing, 2.seconds)
catch case e: RuntimeException => println(s"caught: ${e.getMessage}")
// caught: boom
```

Recover from a failed `Future` without unwrapping it manually using
`recover` (supply a fallback value) or `recoverWith` (supply a fallback
`Future`):

```scala
val recovered = failing.recover { case _: RuntimeException => -1 }
println(Await.result(recovered, 2.seconds))   // -1
```

## Running many Futures together: `Future.sequence`

Turning a `List[Future[A]]` into a `Future[List[A]]` — running everything
concurrently and collecting the results once they're *all* done — is common
enough to have a dedicated combinator:

```scala
val many: List[Future[Int]] = List(1, 2, 3).map(slowSquare)
val allDone: Future[List[Int]] = Future.sequence(many)
println(Await.result(allDone, 2.seconds))   // List(1, 4, 9)
```

All three `slowSquare` calls start at once, so this takes roughly 100ms
total rather than 300ms — the whole point of using `Future` in the first
place.

## Cheat sheet

| Need to... | Use |
|---|---|
| Run work asynchronously | `Future { ... }` (needs an `ExecutionContext` in scope) |
| React without blocking | `.onComplete { case Success(v) => ...; case Failure(e) => ... }` |
| Transform a future value | `.map(f)` |
| Chain futures that depend on each other | `.flatMap(f)` or a `for`-comprehension |
| Run independent futures concurrently | start them *before* the `for`, or use `Future.sequence` |
| Block for a result (tests/`main` only) | `Await.result(f, timeout)` |
| Recover from failure | `.recover { case e => fallback }` / `.recoverWith { case e => otherFuture }` |
| Combine a list of futures | `Future.sequence(listOfFutures)` |

## Exercise

Write `def fetchUser(id: Int): Future[String]` and `def fetchOrders(id: Int):
Future[List[String]]`, each simulating latency with `Thread.sleep` inside a
`Future` block. Write a third function `def userSummary(id: Int): Future[String]`
that runs both concurrently (start both `Future`s before combining them) and
combines the results into a single summary string via a `for`-comprehension.
Use `Await.result` to print the summary for a couple of ids. Then make
`fetchUser` fail for id `0` (throw inside the `Future` block) and use
`.recover` so `userSummary(0)` prints a friendly fallback message instead of
propagating the exception.
