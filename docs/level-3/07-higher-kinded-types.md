# 07 · Higher-Kinded Types Intro

You've already written `F[_]` several times — in the `Functor[F[_]]` type
class from module 04 and the generic `printAll[A: Show]` from module 06.
This module slows down on exactly what `F[_]` means, why it's called
"higher-kinded," and where it stops being useful.

## Kinds: types of types

An ordinary type like `Int` or `String` has **kind** `*` (pronounced
"type") — it's a complete, concrete type you can have a value of. `List`
by itself is *not* a complete type — you can't have a value of type
`List` — it needs a type argument first: `List[Int]` is a complete type,
but `List` alone is a **type constructor**, something that takes a type and
produces a type. Its kind is written `* -> *`.

```scala
val x: Int = 5            // Int has kind *
val xs: List[Int] = List(1, 2, 3)   // List[Int] has kind *, List alone has kind * -> *
```

A **higher-kinded type** is a type parameter that is itself a type
constructor rather than a concrete type — written `F[_]` in Scala, meaning
"some type constructor that takes one type parameter." This is exactly what
lets `Functor[F[_]]` be generic over `List`, `Option`, `Future`, or any
other single-parameter container, rather than being written separately for
each one.

## Writing code parameterized over the container itself

```scala
trait Container[F[_]]:
  def pure[A](a: A): F[A]
  def map[A, B](fa: F[A])(f: A => B): F[B]

given Container[List] with
  def pure[A](a: A): List[A] = List(a)
  def map[A, B](fa: List[A])(f: A => B): List[B] = fa.map(f)

given Container[Option] with
  def pure[A](a: A): Option[A] = Some(a)
  def map[A, B](fa: Option[A])(f: A => B): Option[B] = fa.map(f)

def wrapAndDouble[F[_]](x: Int)(using c: Container[F]): F[Int] =
  c.map(c.pure(x))(_ * 2)

println(wrapAndDouble[List](5))     // List(10)
println(wrapAndDouble[Option](5))   // Some(10)
```

Notice `F[_]` appears in `Container`'s definition and again in
`wrapAndDouble`'s type parameter list — `F` is never given a concrete type
argument in either signature. It's abstracted over the *shape* "a container
of one thing," and the concrete shape (`List`, `Option`, your own type) is
only decided at the call site via `[List]`/`[Option]`.

### Your own type constructor works the same way

```scala
case class Box[A](value: A)

given Container[Box] with
  def pure[A](a: A): Box[A] = Box(a)
  def map[A, B](fa: Box[A])(f: A => B): Box[B] = Box(f(fa.value))

println(wrapAndDouble[Box](5))   // Box(10)
```

`wrapAndDouble` didn't change at all — `Box` slots into the exact same
higher-kinded parameter as `List` and `Option` because it also has kind
`* -> *` (one type parameter) and a lawful `Container` instance.

## `Sized[F[_]]`: extension methods on higher-kinded parameters

Type classes over `F[_]` can also add extension methods usable inside
generic functions, exactly like the ordinary type classes from module 06:

```scala
trait Sized[F[_]]:
  extension [A](fa: F[A]) def size: Int

given Sized[List] with
  extension [A](fa: List[A]) def size: Int = fa.length

def describe[F[_]: Sized, A](fa: F[A]): String = s"size=${fa.size}"

println(describe(List(1, 2, 3)))   // size=3
```

`[F[_]: Sized, A]` combines a context bound on a higher-kinded parameter
`F[_]` with an ordinary type parameter `A` — the same context-bound sugar
from module 06, just applied one level up.

### The trap: kind mismatches read as cryptic errors

Passing `Int` (kind `*`) where an `F[_]` (kind `* -> *`) is expected — or
vice versa — produces an error like `Int takes no type parameters, expected:
one`. This is the compiler enforcing kinds the same way it enforces types;
the fix is almost always that you've mixed up a concrete type with a type
constructor, e.g. writing `Container[Int]` when you meant `Container[List]`
applied to `Int` elsewhere.

## When higher-kinded abstraction stops paying for itself

Not every function needs to be generic over `F[_]`. If you only ever call
`wrapAndDouble` with `List`, writing it concretely (`def wrapAndDouble(x:
Int): List[Int] = List(x * 2)`) is simpler to read and just as correct.
Reach for `F[_]` abstraction when you actually have *multiple* concrete
containers that need the same logic (as here, `List`/`Option`/`Box`) — this
is the same "don't abstract until you have two real cases" judgment call
you already make with ordinary generics.

## Cheat sheet

| Concept | Meaning |
|---|---|
| Kind `*` | An ordinary, complete type (`Int`, `String`, `List[Int]`) |
| Kind `* -> *` | A type constructor needing one type argument (`List`, `Option`, `Box`) |
| `F[_]` | Scala's syntax for a type parameter of kind `* -> *` |
| `[F[_]: TypeClass]` | Context bound on a higher-kinded parameter |
| When to use it | You have ≥2 concrete `F[_]`s needing the same generic logic |

## Exercise

Write a type class `Emptyable[F[_]]` with `def isEmpty[A](fa: F[A]):
Boolean` and `given` instances for `List` and `Option`. Write `def
firstNonEmpty[F[_]: Emptyable, A](xs: List[F[A]]): Option[F[A]]` that
returns the first element of `xs` for which `isEmpty` is `false` (or `None`
if all are empty), using only the type class — no pattern matching on
`List`/`Option` directly inside the generic function. Test it against a
`List[List[Int]]` containing some empty lists and a `List[Option[Int]]`
containing some `None`s.
