# 04 · Functions

## Defining functions with `def`

```scala
def add(a: Int, b: Int): Int =
  a + b

println(add(2, 3))   // 5
```

The return type (`Int` here) can usually be inferred and omitted, but it's
good practice to write it explicitly on any function others will call — it
documents the contract and catches mistakes early:

```scala
def square(n: Int) = n * n   // return type inferred as Int
println(square(4))            // 16
```

A function body is just an expression — the last expression's value is
returned automatically. There's no `return` keyword needed (and using it
explicitly is considered poor style in idiomatic Scala):

```scala
def classify(n: Int): String =
  if n < 0 then "negative"
  else if n == 0 then "zero"
  else "positive"

println(classify(-5))   // negative
println(classify(0))    // zero
println(classify(7))    // positive
```

## Default and named parameters

```scala
def greet(name: String, greeting: String = "Hello"): String =
  s"$greeting, $name!"

println(greet("Ada"))                       // Hello, Ada!
println(greet("Ada", "Welcome"))            // Welcome, Ada!
println(greet(name = "Grace", greeting = "Hi"))  // Hi, Grace!
println(greet(greeting = "Hey", name = "Alan"))  // Hey, Alan! -- order doesn't matter with names
```

## Multiple parameter lists

Functions can take several parameter lists instead of one — useful for
readability and for enabling type inference across groups of arguments
(you'll see this heavily with higher-order functions):

```scala
def multiply(a: Int)(b: Int): Int = a * b

println(multiply(3)(4))   // 12

val timesFive = multiply(5)  // partial application -- fixes the first argument
println(timesFive(2))         // 10
println(timesFive(10))        // 50
```

## Functions as values

In Scala, functions are values just like `Int` or `String` — you can store
them in a `val`, pass them as arguments, and return them from other
functions. This is the foundation of Scala's functional style.

```scala
val double: Int => Int = n => n * 2
println(double(21))   // 42

// A function that takes another function as a parameter
def applyTwice(f: Int => Int, x: Int): Int =
  f(f(x))

println(applyTwice(double, 5))   // 20 -- double(double(5)) = double(10) = 20
```

## Anonymous functions (lambdas)

```scala
val square = (n: Int) => n * n
println(square(6))   // 36

// Shorthand placeholder syntax for simple cases
val nums = List(1, 2, 3, 4)
val doubled = nums.map(n => n * 2)
val doubledShort = nums.map(_ * 2)   // same thing, terser
println(doubled)        // List(2, 4, 6, 8)
println(doubledShort)   // List(2, 4, 6, 8)
```

## Recursion

Recursive functions are common in Scala since loops with mutable state are
discouraged. `@tailrec` asks the compiler to verify a recursive call is in
tail position (so it compiles to a loop, not stack-consuming recursion):

```scala
import scala.annotation.tailrec

@tailrec
def factorial(n: Int, acc: Long = 1): Long =
  if n <= 1 then acc
  else factorial(n - 1, n * acc)   // tail call -- last thing the function does

println(factorial(10))   // 3628800
```

If a function marked `@tailrec` isn't actually tail-recursive, the compiler
raises an error at compile time rather than letting you discover a
`StackOverflowError` later at runtime — a nice safety net.

## How It Actually Works

A plain `def` compiles to an ordinary JVM instance (or static) method — no
surprises there. The interesting mechanism is what happens when you turn a
method into a *value* with `val addFn = add` or write a lambda: Scala
compiles that into an instance of a `scala.FunctionN` interface (`Function1`
for one argument, `Function2` for two, and so on, up to 22 — the historic
reason Scala tuples and function arities top out at 22). Each `FunctionN`
trait defines an `apply` method, so `addFn(3, 4)` is really sugar for
`addFn.apply(3, 4)`. A lambda like `(x: Int) => x * 2` becomes, since Scala
2.12+, an `invokedynamic` call site backed by Java's `LambdaMetafactory` —
the JVM generates the implementing class at runtime the first time that
call site executes and caches it, rather than the compiler pre-generating a
named anonymous class for every lambda the way older Scala/Java did. This
is why lambdas are cheap but not entirely free: the first invocation of a
given lambda literal pays a one-time class-generation cost.

Recursion has no special bytecode support on the JVM (no built-in
tail-call instruction), so a recursive `def` normally consumes one stack
frame per call and can overflow on deep recursion. The Scala compiler
optimizes the specific case of **self-tail-recursion** — a call to the
function itself in tail position — by rewriting it into a `while`-style
loop with mutated local variables, eliminating the stack growth entirely.
`@tailrec` doesn't enable this optimization; it only asks the compiler to
*fail the build* if the rewrite ISN'T possible, catching accidental
non-tail recursion (e.g. `n * factorial(n - 1)`, where the multiplication
happens after the recursive call returns) at compile time instead of as a
`StackOverflowError` at runtime.

## Cheat sheet

| Syntax | Meaning |
|--------|---------|
| `def f(x: Int): Int = ...` | Named function with an explicit return type |
| `def f(x: Int = 5) = ...` | Default parameter value |
| `f(x = 1, y = 2)` | Named arguments |
| `def f(a: Int)(b: Int) = ...` | Multiple parameter lists (curried) |
| `val f: Int => Int = n => n * 2` | Function value with an explicit function type |
| `_ * 2` | Placeholder syntax for a simple one-argument lambda |
| `@tailrec` | Compiler-checked tail-call optimization |

## Exercise

Write a function `isPrime(n: Int): Boolean` that checks primality. Then
write a higher-order function `countMatching(nums: List[Int], predicate: Int => Boolean): Int`
that counts how many elements of a list satisfy a predicate, and use it with
`isPrime` to count the primes in `(1 to 50).toList`. Finally, write a
tail-recursive `sumList(nums: List[Int], acc: Int = 0): Int` that sums a list
without using a built-in `sum` method.
