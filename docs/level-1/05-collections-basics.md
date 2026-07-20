# 05 · Collections Basics

Scala's standard library ships a rich set of immutable-by-default
collections. This module covers the three you'll use constantly: `List`,
`Map`, and `Set`. All three examples below use their **immutable** versions
(the default when you just write `List`, `Map`, `Set` with no import) —
operations return a *new* collection rather than modifying the original.

## `List`

An ordered, immutable sequence.

```scala
val fruits = List("apple", "banana", "cherry")

println(fruits(0))          // apple -- indexing
println(fruits.head)        // apple -- first element
println(fruits.tail)        // List(banana, cherry) -- everything but the first
println(fruits.length)      // 3
println(fruits.isEmpty)     // false

// Adding elements creates a NEW list -- the original is untouched
val moreFruits = fruits :+ "date"       // append
val evenMore = "avocado" :: fruits      // prepend (cons operator)
println(moreFruits)   // List(apple, banana, cherry, date)
println(evenMore)     // List(avocado, apple, banana, cherry)
println(fruits)       // List(apple, banana, cherry) -- unchanged

// Common transformations
println(fruits.map(_.toUpperCase))        // List(APPLE, BANANA, CHERRY)
println(fruits.filter(_.length > 5))      // List(banana, cherry)
println(fruits.sorted)                    // List(apple, banana, cherry)
println(fruits.reverse)                   // List(cherry, banana, apple)
println(fruits.contains("banana"))        // true
```

Iterating:

```scala
for fruit <- fruits do
  println(s"I like $fruit")
// I like apple
// I like banana
// I like cherry
```

## `Map`

A collection of key-value pairs.

```scala
val ages = Map("Ada" -> 36, "Grace" -> 85, "Alan" -> 41)

println(ages("Ada"))              // 36
println(ages.getOrElse("Bob", 0)) // 0 -- safe lookup for a missing key
println(ages.contains("Grace"))   // true
println(ages.keys)                // Set(Ada, Grace, Alan) (order not guaranteed)
println(ages.values)              // collection of 36, 85, 41

// Adding/updating returns a new Map
val updated = ages + ("Bob" -> 29)
println(updated)   // Map(Ada -> 36, Grace -> 85, Alan -> 41, Bob -> 29)

// Iterating over entries
for (name, age) <- ages do
  println(s"$name is $age years old")
```

Looking up a missing key with `ages("Bob")` (no default) throws a
`NoSuchElementException` — prefer `getOrElse` or pattern matching on the
`Option` returned by `ages.get("Bob")` (more on `Option` in
[Level 2](../level-2/04-option-either.md)).

## `Set`

An unordered collection with no duplicate elements.

```scala
val colors = Set("red", "green", "blue")

println(colors.contains("red"))    // true
println(colors + "yellow")         // Set(red, green, blue, yellow) -- new set
println(colors - "green")          // Set(red, blue) -- new set, one removed

val moreColors = Set("blue", "purple")
println(colors union moreColors)         // Set(red, green, blue, purple)
println(colors intersect moreColors)     // Set(blue)
println(colors diff moreColors)          // Set(red, green)

// Adding a duplicate is a no-op
val stillThree = colors + "red"
println(stillThree.size)   // 3
```

## Tuples (a quick companion tool)

Tuples group a fixed number of values of possibly different types — handy
for returning multiple values without defining a class:

```scala
val point = (3, 4)
println(point._1)   // 3
println(point._2)   // 4

val (x, y) = point   // destructuring
println(s"x=$x, y=$y")   // x=3, y=4

def minMax(nums: List[Int]): (Int, Int) =
  (nums.min, nums.max)

val (lo, hi) = minMax(List(4, 1, 9, 2))
println(s"low=$lo, high=$hi")   // low=1, high=9
```

## Cheat sheet

| Collection | Ordered? | Duplicates? | Key operations |
|------------|----------|-------------|-----------------|
| `List[A]` | Yes | Yes | `head`, `tail`, `:+`, `::`, `map`, `filter` |
| `Map[K, V]` | No (by default) | Unique keys | `apply`, `getOrElse`, `get`, `+`, `-` |
| `Set[A]` | No | No | `contains`, `+`, `-`, `union`, `intersect`, `diff` |
| `(A, B)` tuple | fixed arity | — | `._1`, `._2`, destructuring |

## Exercise

Given `val words = List("scala", "is", "fun", "and", "concise")`, write
expressions that: (1) return only the words longer than 3 characters, (2)
return a `Map` from each word to its length, (3) return the `Set` of unique
first letters across all words. Print each result.
