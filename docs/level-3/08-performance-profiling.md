# 08 · Performance & Profiling

Correct code isn't always fast code, and guessing where the slow part is
usually wrong. This module covers measuring performance properly —
including why naive hand-rolled timing lies to you — and points at JMH, the
standard tool for trustworthy JVM microbenchmarks.

## Why `System.nanoTime` around one run isn't enough

The JVM doesn't run at full speed immediately: the JIT compiler needs to
see a method called many times before it compiles it to fast native code,
and garbage collection pauses can land in the middle of any single
measurement. A one-shot timing captures the JVM warming up, not steady-state
performance. The fix is a warm-up phase followed by averaging several
measured runs:

```scala
def benchmark[A](label: String, warmup: Int = 5, runs: Int = 5)(block: => A): A =
  var result: A = null.asInstanceOf[A]
  for _ <- 1 to warmup do result = block          // let the JIT warm up, discard timings
  val times = for _ <- 1 to runs yield
    val start = System.nanoTime()
    result = block
    System.nanoTime() - start
  val avgMs = times.sum.toDouble / times.size / 1e6
  println(f"$label%-20s avg: $avgMs%.3f ms")
  result
```

`block: => A` is a by-name parameter — the code isn't evaluated when passed
in, only each time `block` is referenced inside `benchmark`, which is what
lets one function body run the same work repeatedly.

## Comparing two implementations

```scala
def sumWithFor(n: Int): Long =
  var total = 0L
  var i = 0
  while i < n do
    total += i
    i += 1
  total

def sumWithFold(n: Int): Long =
  (0 until n).foldLeft(0L)(_ + _)

val n = 5_000_000
benchmark("while loop") { sumWithFor(n) }
benchmark("foldLeft") { sumWithFold(n) }
```

```text
while loop           avg: 1.920 ms
foldLeft             avg: 24.771 ms
```

The `while` loop is roughly 10x faster here — `foldLeft` over a `Range`
allocates a closure invocation per element and can't always be inlined the
way a hand-written loop can. This doesn't mean "never use `foldLeft`" — it
means measure before assuming idiomatic-looking code is fast enough for a
genuine hot path, and reach for imperative loops only where profiling shows
it actually matters.

## A classic Scala performance trap: `List` append

```scala
benchmark("List :+ append (bad)", warmup = 1, runs = 1) {
  var acc = List.empty[Int]
  for i <- 1 to 5000 do acc = acc :+ i
  acc.length
}
benchmark("ListBuffer append (good)", warmup = 1, runs = 1) {
  val buf = scala.collection.mutable.ListBuffer.empty[Int]
  for i <- 1 to 5000 do buf += i
  buf.length
}
```

```text
List :+ append (bad)      avg: 149.287 ms
ListBuffer append (good)  avg: 0.298 ms
```

That's a ~500x difference for building the same 5,000-element sequence.
`List` is a singly-linked list optimized for prepending (`::`) at the head
— appending at the tail (`:+`) has to walk and rebuild the *entire* list
every single call, making a loop of `n` appends `O(n²)` overall.
`ListBuffer` is a mutable buffer designed for exactly this — appending is
`O(1)` — and `.toList` converts it back once you're done building.

### The trap: reaching for the wrong collection by habit

`List` is the default collection Scala tutorials reach for, which makes
"loop and append to a `List`" an easy habit to fall into. Whenever you're
building a collection incrementally in a loop, prefer `ListBuffer` (or a
`for`-comprehension / `.map` that builds the whole thing in one pass) and
only convert to an immutable `List` at the end — never accumulate with
repeated `:+` on an immutable `List`.

## Profiling beyond hand-rolled timing: JMH

Hand-rolled benchmarks like the ones above are fine for a quick A/B check,
but they're still vulnerable to dead-code elimination (the JIT can
sometimes discover a result is never used and skip computing it entirely)
and don't control for JVM startup, forking, or statistical noise the way a
dedicated tool does. **JMH** (Java Microbenchmark Harness), used via the
`sbt-jmh` plugin, is the standard for trustworthy JVM/Scala benchmarks — it
forks a fresh JVM per benchmark, runs proper warm-up and measurement
iterations, and reports results with error bars:

```scala
// with sbt-jmh, a benchmark is a plain annotated method:
import org.openjdk.jmh.annotations._

@State(Scope.Benchmark)
class SumBenchmark:
  val n = 5_000_000

  @Benchmark
  def whileLoop(): Long = sumWithFor(n)

  @Benchmark
  def foldLeft(): Long = sumWithFold(n)
```

Running `sbt "Jmh/run -i 5 -wi 5 -f1"` executes both in isolated forked
JVMs and prints throughput/latency with confidence intervals — reach for
this once a hand-rolled comparison suggests something worth measuring
rigorously, especially before changing performance-sensitive production
code based on a microbenchmark.

## Cheat sheet

| Need to... | Use |
|---|---|
| Time code without lying to yourself | warm up first, average several measured runs |
| Compare two implementations | run both through the same `benchmark` helper |
| Build a collection in a loop | `ListBuffer` (or a single-pass `.map`), not repeated `List :+` |
| Get statistically trustworthy JVM benchmarks | JMH via `sbt-jmh`, forked JVMs + warm-up + measurement phases |
| Avoid dead-code-elimination skewing results | JMH's `Blackhole`, or ensure the result is actually used/printed |

## Exercise

Extend the `benchmark` helper to also report the *minimum* and *maximum*
of the measured runs, not just the average — this reveals outliers a raw
average can hide. Then benchmark three ways of checking whether a `List[Int]`
of 100,000 elements contains a given value: `.contains`, converting to a
`Set` once and checking membership, and a hand-rolled recursive search.
Report all three with min/avg/max and explain in a comment which one you'd
actually pick for a function called once per request versus one called in
a tight loop.
