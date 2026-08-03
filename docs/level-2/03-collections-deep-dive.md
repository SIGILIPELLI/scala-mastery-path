# 03 · Collections Deep Dive

[Level 1](../level-1/05-collections-basics.md) introduced `List`, `map`, and
`filter` as everyday tools. This module goes deeper into the fold/reduce
family (and where each one can bite you), lazy `view`s for processing large
sequences without building intermediate collections, and how to pick the
right collection type — `List`, `Vector`, `Set`, or `Map` — for the job
instead of defaulting to `List` everywhere.

## `map`, `filter`, `flatMap` — the basics, extended

```scala
val nums = List(1, 2, 3, 4, 5)

println(nums.map(_ * 2))                 // List(2, 4, 6, 8, 10)
println(nums.filter(_ % 2 == 0))         // List(2, 4)
println(nums.flatMap(n => List(n, -n)))  // List(1, -1, 2, -2, 3, -3, 4, -4, 5, -5)
```

`flatMap` is `map` followed by flattening one level — each input element can
expand into zero, one, or many output elements, which is exactly what you
need when a transformation itself returns a collection (or, as you'll see in
[Module 4](04-option-either.md), an `Option`).

## `fold` vs `reduce`: the difference that matters

Both combine a collection down to a single value with a binary operation,
but they differ in one important way: `fold` takes an explicit starting
value and an operation whose accumulator and element types don't even have
to be the same; `reduce` uses the collection's first element as the starting
point and requires the operation to combine same-typed values.

```scala
val nums = List(1, 2, 3, 4, 5)

val sum = nums.fold(0)(_ + _)
println(sum)   // 15

// fold can change type entirely -- the accumulator starts as a String
val trace = nums.foldLeft("start")((acc, n) => s"$acc-$n")
println(trace)   // start-1-2-3-4-5

val product = nums.reduce(_ * _)
println(product)   // 120
```

The trap: **`reduce` throws on an empty collection**, because there's no
first element to seed it with. `fold` never has this problem, because you
supplied the starting value yourself:

```scala
try
  List.empty[Int].reduce(_ + _)
catch
  case e: UnsupportedOperationException => println(s"caught: ${e.getMessage}")
  // caught: empty.reduceLeft

println(List.empty[Int].fold(0)(_ + _))   // 0 -- no exception, just the seed value
```

Prefer `fold` (with an explicit, meaningful starting value) over `reduce`
any time the input might be empty — which, in real code reading from files,
APIs, or user input, is more often than it looks.

`foldLeft`/`foldRight` are the direction-explicit versions of `fold` — use
them when the accumulator type differs from the element type (as `trace`
does above) or when evaluation order actually matters, such as building a
string or a reversed structure.

## `groupBy` and `partition`

```scala
val nums = List(1, 2, 3, 4, 5)

val grouped = nums.groupBy(n => if n % 2 == 0 then "even" else "odd")
println(grouped)   // HashMap(odd -> List(1, 3, 5), even -> List(2, 4))

val (evens, odds) = nums.partition(_ % 2 == 0)
println(evens)   // List(2, 4)
println(odds)    // List(1, 3, 5)
```

`groupBy` builds a `Map` keyed by whatever your function returns — useful
for bucketing (note the result is a `HashMap`, so don't rely on key order).
`partition` is the two-bucket special case (matches vs. non-matches) and
returns a plain tuple, so you can destructure it directly as shown.

Combining a fold with a tuple accumulator is a common way to compute several
running totals in a single pass, without groupBy's overhead of materializing
sublists:

```scala
val (evenSum, oddSum) = nums.foldLeft((0, 0)) { case ((e, o), n) =>
  if n % 2 == 0 then (e + n, o) else (e, o + n)
}
println(s"evenSum=$evenSum oddSum=$oddSum")   // evenSum=6 oddSum=9
```

## Lazy views: process without materializing

`map` and `filter` on a normal collection are **eager** — each call builds a
brand-new collection right away, even if you only need the first few
results. Calling `.view` switches to a **lazy** wrapper: transformations are
recorded but not run until you force the result (with `.toList`, `.force`,
etc.), and only as many elements as needed are ever computed.

```scala
var mapCalls = 0
var filterCalls = 0

val result = (1 to 1_000_000).view
  .map { x => mapCalls += 1; x * 2 }
  .filter { x => filterCalls += 1; x % 3 == 0 }
  .take(3)
  .toList

println(result)       // List(6, 12, 18)
println(mapCalls)     // 9  -- not 1,000,000
println(filterCalls)  // 9
```

Without `.view`, that pipeline would allocate two full million-element
intermediate collections before `take(3)` ever ran. With `.view`, Scala
pulls elements through the whole chain one at a time and stops the instant
`take(3)` is satisfied. Reach for `.view` on large sequences with multiple
chained transformations where you don't need every element — for small
collections it's not worth the extra layer of indirection.

## Choosing the right collection

| Type | Ordered? | Duplicates? | Typical use | Lookup by key/index |
|---|---|---|---|---|
| `List` | yes | yes | sequential processing, prepend-heavy | `O(n)` random access |
| `Vector` | yes | yes | general-purpose sequence, frequent random access/updates | `O(log n)`, effectively fast |
| `Set` | no (in general) | no | membership tests, deduplication | `O(1)`-ish `contains` |
| `Map` | no (in general) | no (unique keys) | key-value lookups | `O(1)`-ish `get`/`apply` |

```scala
val s: Set[Int] = Set(1, 2, 2, 3)
println(s)               // Set(1, 2, 3) -- duplicate 2 silently collapsed
println(s.contains(2))   // true -- much faster than nums.contains(2) on a large List

val m: Map[String, Int] = Map("a" -> 1, "b" -> 2)
println(m("a"))                 // 1 -- throws NoSuchElementException if missing
println(m.getOrElse("z", -1))   // -1 -- safe default instead of an exception
println(m + ("c" -> 3))         // Map(a -> 1, b -> 2, c -> 3) -- immutable, returns a new Map
```

The trap: reaching for `List` by habit and then calling `.contains` or index
lookups on it in a hot path. `List.contains` and `List(i)` are both `O(n)` —
walking the whole linked list — while the equivalent operations on `Set`
and `Vector` are effectively constant time. If you're checking membership
repeatedly, convert to a `Set` once; if you're indexing repeatedly, use a
`Vector`.

## Cheat sheet

| Method | Input → Output | Empty-safe? |
|---|---|---|
| `map(f)` | `List[A] → List[B]` | yes |
| `flatMap(f)` | `List[A] → List[B]` (flattened) | yes |
| `filter(p)` | `List[A] → List[A]` | yes |
| `fold(seed)(op)` | `List[A] → B` | yes (returns seed) |
| `reduce(op)` | `List[A] → A` | **no** — throws on empty |
| `groupBy(f)` | `List[A] → Map[K, List[A]]` | yes |
| `partition(p)` | `List[A] → (List[A], List[A])` | yes |
| `.view...toList` | lazy pipeline → eager result | yes |

## Exercise

Given `val words = List("scala", "is", "a", "great", "language", "for", "jvm", "developers")`,
use `groupBy` to bucket the words by length, then use `fold` (not `reduce`)
to compute the total character count across all words starting from `0`.
Separately, build a lazy `.view` pipeline over `(1 to 10_000_000)` that maps
each number to its square, filters to keep only squares ending in `5`, and
takes the first 5 — confirm (with a counter, like the example above) that
far fewer than 10 million squarings actually happen.
