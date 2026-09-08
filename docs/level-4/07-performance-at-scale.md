# 07 · Performance at Scale

[Level 3's performance module](../level-3/08-performance-profiling.md)
measured single-threaded code. Production services live or die on
*throughput under concurrent load* — this module covers caching to avoid
redundant work, thread pool sizing (the resource that determines how much
concurrent work actually gets done), and where to point people at for JVM
tuning.

## Caching: don't recompute what you already know

```scala
import java.util.concurrent.ConcurrentHashMap

def expensiveCompute(n: Int): Int =
  Thread.sleep(5)   // simulate real work
  n * n

val cache = new ConcurrentHashMap[Int, Int]()
def cachedCompute(n: Int): Int =
  cache.computeIfAbsent(n, k => expensiveCompute(k))

(1 to 20).foreach(_ => cachedCompute(5))
// 20 calls, same key, cached: 5.5 ms   -- one real ~5ms compute, 19 cache hits
```

`ConcurrentHashMap.computeIfAbsent` is the thread-safe way to do
"check-then-compute-then-store" atomically — a plain `Map` with a manual
`if (!map.contains(k))` check has a race: two threads can both see the key
missing and both run `expensiveCompute` before either stores a result.
`computeIfAbsent` guarantees the computation runs (at most) once per key
even under concurrent access.

### The trap: unbounded caches are a memory leak with extra steps

The `cache` above never evicts anything — every distinct key ever seen
stays in memory forever. For a cache keyed by something unbounded (user
IDs, request parameters), this is a slow leak that looks fine in
development and pages someone in production weeks later. Production caches
need an eviction policy — a maximum size, a time-to-live, or both (Caffeine
is the standard JVM library for this) — from the start, not bolted on after
an incident.

## Thread pool sizing changes real throughput

```scala
import java.util.concurrent.Executors

val pool = Executors.newFixedThreadPool(8)
val futures = (1 to 40).map(i => pool.submit(() => expensiveCompute(i)))
futures.foreach(_.get())
// 40 tasks, 8 threads: 30.9 ms

val pool2 = Executors.newFixedThreadPool(40)
val futures2 = (1 to 40).map(i => pool2.submit(() => expensiveCompute(i)))
futures2.foreach(_.get())
// 40 tasks, 40 threads: 7.9 ms
```

With 8 threads, 40 five-millisecond tasks queue up in batches of 8 —
roughly `40/8 * 5ms = 25ms` minimum, matching the ~31ms measured. With 40
threads, every task starts immediately and they all finish in roughly one
task's duration. This isn't "more threads is always better" — it's
"the pool needs enough threads for the *actual* concurrent workload," which
depends on how many requests really run at once and how much of each
task's time is spent waiting (I/O) versus computing (CPU).

### The trap: CPU-bound work doesn't benefit past your core count

The example above simulates I/O-bound work (`Thread.sleep`), where threads
spend most of their time waiting and a large pool helps. Genuinely
CPU-bound work (real computation, not waiting) has no benefit from a
thread pool larger than the number of CPU cores — extra threads just
context-switch against each other for the same fixed CPU capacity. Size
I/O-bound pools generously (they're usually blocked, not competing for
CPU); size CPU-bound pools around `Runtime.getRuntime.availableProcessors`.

## JVM-level tuning

Beyond application code, the JVM itself has tunable parameters that matter
at scale — most commonly heap size and garbage collector choice:

```text
java -Xms512m -Xmx2g -XX:+UseG1GC -jar app.jar
```

`-Xms`/`-Xmx` set the initial/maximum heap; `-XX:+UseG1GC` selects the G1
garbage collector (the JVM's modern default for most workloads, tuned for
balancing latency and throughput). Diagnosing *whether* GC is actually your
bottleneck — versus thread contention, slow I/O, or a genuinely slow
algorithm — needs a profiler (Level 3 mentioned JMH for microbenchmarks;
async-profiler or Java Flight Recorder are the tools for profiling a whole
running service) rather than guessing at flags.

## How It Actually Works

An in-memory cache backed by a plain mutable `Map` is a memory leak
"with extra steps" for a garbage-collection reason: the JVM's GC reclaims
an object only when nothing reachable still references it, and a
never-evicted cache entry is, by definition, still referenced by the
cache's own map — so the GC correctly considers it live and never
collects it, no matter how stale or unused it's become. Bounded caches
(LRU, size- or time-based eviction) solve this by actively removing
entries themselves, making them unreachable and therefore GC-eligible,
rather than relying on any GC feature to notice "this entry hasn't been
touched in a while" — ordinary GC has no such notion.

Thread pool sizing changes real throughput because of a very concrete
JVM-level distinction: **CPU-bound** work keeps a core continuously busy
computing, so a thread pool larger than the number of physical/logical
cores just causes threads to context-switch for CPU time they can't all
get simultaneously (each context switch itself has real cost — saving and
restoring register state and cache lines) — throughput plateaus once
threads ≈ cores. **I/O-bound** work (blocking on a socket or disk read,
as in [Level 3's JDBC calls](../level-3/03-databases.md)) spends most of
its time with the thread parked, not consuming CPU, so a larger pool
lets more of that idle-waiting overlap productively; the right pool size
there scales with how much time each task spends blocked vs. computing,
not simply with core count.

JVM-level tuning (heap sizing, garbage collector choice like G1 vs. ZGC)
changes the trade-offs of the same underlying mechanism from [Level
3](../level-3/08-performance-profiling.md)'s JIT/warmup discussion: a
larger heap means the GC runs less often but each collection may pause
longer (more objects to scan/relocate), while newer low-pause collectors
like ZGC trade some throughput for keeping individual GC pauses in the
sub-millisecond to few-millisecond range regardless of heap size — the
right choice depends on whether your service cares more about maximum
throughput or predictable low-latency response times.

## Cheat sheet

| Need to... | Use |
|---|---|
| Avoid recomputing the same result | `ConcurrentHashMap.computeIfAbsent` (thread-safe check-then-compute) |
| Keep a cache from growing forever | an eviction policy (max size / TTL) — Caffeine in production |
| Size a thread pool for I/O-bound work | generous — threads spend most time waiting, not competing for CPU |
| Size a thread pool for CPU-bound work | around `availableProcessors()` — more threads won't help |
| Tune heap/GC | `-Xms`/`-Xmx`, `-XX:+UseG1GC`, verified with a profiler, not guessed |

## Exercise

Extend the `cachedCompute` example with a simple time-based eviction: store
`(value, insertedAtMillis)` pairs, and have a lookup that treats an entry
older than some TTL as a miss (recomputing and overwriting it). Then run
the thread-pool comparison above with three pool sizes (`2`, `8`, `40`)
against 40 *CPU-bound* tasks (replace `Thread.sleep` with a tight loop
computing something, e.g. summing to a few million) instead of I/O-bound
ones, and confirm in your printed results that throughput stops improving
once the pool size exceeds your machine's core count
(`Runtime.getRuntime.availableProcessors()`).
