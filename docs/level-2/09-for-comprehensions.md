# 09 · For-Comprehensions

You've used `for x <- list do ...` for a plain loop since Level 1, and
[Module 4](04-option-either.md) used `for`-comprehensions to chain
`Option`/`Either` validations without spelling out `flatMap` by hand. This
module explains exactly what a `for`-comprehension desugars into — because
once you know that, "why doesn't this compile" and "why did this run twice"
stop being mysteries.

## It's not a loop — it's sugar for `map`/`flatMap`

A `for`-comprehension with a `yield` is rewritten by the compiler into calls
to `map` and `flatMap` (and `withFilter`, covered below) on whatever type
you're iterating over. This one:

```scala
def half(n: Int): Option[Int] =
  if n % 2 == 0 then Some(n / 2) else None

def parseInt(s: String): Option[Int] = s.toIntOption

val result =
  for
    a <- parseInt("20")
    b <- half(a)
    c <- half(b)
  yield a + b + c

println(result)   // Some(35)  -- 20 + 10 + 5
```

is exactly equivalent to:

```scala
val desugared =
  parseInt("20").flatMap(a => half(a).flatMap(b => half(b).map(c => a + b + c)))

println(desugared)          // Some(35)
println(result == desugared) // true
```

Every generator (`<-`) except the *last* one becomes a `flatMap`; the last
one becomes a `map` (since it's the one that produces the final `yield`ed
value, and only the outermost result needs wrapping). This is why a
`for`-comprehension works on `Option`, `Either`, `List`, `Future`, and any
other type you define yourself — as long as it has `map` and `flatMap` with
compatible signatures, `for` syntax works on it, no special support needed.

## Short-circuiting falls out of `flatMap`, not special `for` logic

Because the desugaring is just nested `flatMap` calls, "stop at the first
`None`/`Left`" isn't a feature of `for` — it's simply how `Option.flatMap`
and `Either.flatMap` already behave (a `flatMap` on `None`/`Left` returns
`None`/`Left` without ever calling the function you passed):

```scala
val result2 =
  for
    a <- parseInt("7")   // Some(7) -- parses fine
    b <- half(a)         // 7 is odd -> None
    c <- half(b)         // never runs -- flatMap on None short-circuits
  yield a + b + c

println(result2)   // None
```

```scala
def validatePositive(n: Int): Either[String, Int] =
  if n > 0 then Right(n) else Left(s"$n is not positive")

val eitherResult =
  for
    x <- validatePositive(5)
    y <- validatePositive(3)
  yield x * y
println(eitherResult)   // Right(15)

val eitherFail =
  for
    x <- validatePositive(5)
    y <- validatePositive(-3)
  yield x * y
println(eitherFail)   // Left(-3 is not positive)
```

## `for` over collections: it's a cartesian product, not nested loops in disguise

Applied to two `List`s, the same `flatMap`/`map` desugaring produces every
combination of elements — worth seeing explicitly, since it's easy to
picture `for` over two lists as "zip them together" when it actually
produces their full cross product:

```scala
val pairs =
  for
    x <- List(1, 2, 3)
    y <- List("a", "b")
  yield (x, y)

println(pairs)
// List((1,a), (1,b), (2,a), (2,b), (3,a), (3,b))
```

## Guards: the `if` inside a `for` desugars to `withFilter`

An `if` condition inside a `for`-comprehension (not the block after
`yield`) becomes a call to `withFilter` (a lazy relative of `filter` used
specifically to support `for`) before the next `map`/`flatMap` in the
chain:

```scala
val evensDoubled =
  for
    x <- List(1, 2, 3, 4, 5, 6)
    if x % 2 == 0
  yield x * 2

println(evensDoubled)   // List(4, 8, 12)
```

## `for` without `yield`: pure side effects

Drop `yield` entirely and the comprehension desugars to `foreach` instead of
`map`/`flatMap` — it runs for its side effects and produces `Unit`, not a
new collection:

```scala
for
  x <- List(1, 2, 3)
do println(s"side effect: $x")
// side effect: 1
// side effect: 2
// side effect: 3
```

This is exactly the form you've been using since Level 1's `for x <- xs do
...` loops — now you know it's `foreach` under the hood, not special loop
syntax.

## The trap: mixing incompatible container types in one `for`

Because `for` desugars to `flatMap`/`map` calls on the *specific* type of
each generator, every generator in one comprehension has to be a type whose
`flatMap` can plausibly chain with the others — in practice, this almost
always means "all generators must be the same kind of container" (all
`Option`, or all `List`, or all `Future`, etc.):

```scala
// This will NOT compile:
// for
//   a <- Some(1)
//   b <- List(1, 2)
// yield (a, b)
//
// Option's flatMap expects a function returning Option[_]; List(1, 2) isn't
// an Option, so there's no way to desugar this into a type-correct
// flatMap call. The fix is almost always: convert one side to match the
// other explicitly (e.g. a.toList) rather than mixing container types.
```

If you ever see a `for`-comprehension refuse to compile with a confusing
type-mismatch error, the first thing to check is whether every generator
really is the same kind of container — the error message rarely says that
directly.

## Cheat sheet

| `for` syntax | Desugars to |
|---|---|
| `for x <- xs yield f(x)` (last/only generator) | `xs.map(f)` |
| `for x <- xs; y <- ys yield ...` | `xs.flatMap(x => ys.map(y => ...))` |
| `for x <- xs if cond yield f(x)` | `xs.withFilter(cond).map(f)` |
| `for x <- xs do sideEffect(x)` (no `yield`) | `xs.foreach(sideEffect)` |

## Exercise

Write `def safeDivide(a: Int, b: Int): Option[Int] = if b == 0 then None else
Some(a / b)`. Using a `for`-comprehension over three calls to
`safeDivide`, compute `((100 / a) / b) / c` for some `a`, `b`, `c` of your
choosing, short-circuiting to `None` if any denominator is zero. Then
manually rewrite your `for`-comprehension as nested `flatMap`/`map` calls
(no `for` syntax at all) and confirm — by comparing the two results with
`==` — that they produce identical output for at least one input that
succeeds and one that hits a `None` partway through.
