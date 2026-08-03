# 02 · Pattern Matching Advanced

[Level 1](../level-1/08-pattern-matching-intro.md) covered matching on
literals, types, case classes, lists and tuples. This module explains *how*
`case Point(x, y) =>` actually works under the hood — through a mechanism
called an extractor — and then uses that to write your own custom patterns,
match nested structures several levels deep, and lean on sealed hierarchies
so the compiler proves your matches are exhaustive.

## Extractor objects: what `unapply` actually is

Every pattern like `case Point(x, y) =>` is powered by a method named
`unapply` on `Point`'s companion object. Case classes get one generated for
free, but you can write your own on a plain `object` to make **any**
condition usable as a pattern — not just "is this a certain shape of data."

An `unapply` that just tests a condition returns `Boolean`:

```scala
object Even:
  def unapply(n: Int): Boolean = n % 2 == 0

def describe(n: Int): String = n match
  case Even() => s"$n is even"
  case _      => s"$n is odd"

println(describe(4))   // 4 is even
println(describe(7))   // 7 is odd
```

An `unapply` that also wants to **bind values** returns `Option[T]` (or a
tuple inside the `Option` for multiple bindings) — `None` means "doesn't
match," `Some(...)` means "matches, here are the extracted parts":

```scala
object AsPair:
  def unapply(s: String): Option[(String, String)] =
    s.split("@", 2) match
      case Array(user, domain) => Some((user, domain))
      case _                   => None

def splitEmail(s: String): String = s match
  case AsPair(user, domain) => s"user=$user domain=$domain"
  case _                    => "not an email"

println(splitEmail("ada@example.com"))   // user=ada domain=example.com
println(splitEmail("not-an-email"))      // not an email
```

`case AsPair(user, domain) =>` reads exactly like matching a case class, but
`AsPair` is a plain `object` — the pattern is entirely custom logic. This is
how libraries hand you nice, readable patterns (`case r"..."` regex
extractors, `case NonEmptyList(head, tail) =>`, etc.) without exposing you to
their internal representation.

There's also `unapplySeq`, for patterns that extract a variable-length
sequence (returning `Option[Seq[T]]`), used by list/varargs-style patterns
like `case List(a, b, rest*) =>`.

## Sealed hierarchies and exhaustiveness

[Module 1](01-oop-deep-dive.md) modeled shapes as a `sealed trait` with
several `case class`es. The payoff for `sealed` shows up specifically at
`match` time: the compiler knows the *complete* list of subtypes, so it can
verify every one is handled and warn you (as an error, under `-Werror`, or a
compiler warning otherwise) if you add a new case and forget a match
somewhere:

```scala
sealed trait Json
case class JNum(value: Double) extends Json
case class JStr(value: String) extends Json
case class JArr(items: List[Json]) extends Json
case object JNull extends Json

def render(j: Json): String = j match
  case JNum(n)        => n.toString
  case JStr(s)        => s"\"$s\""
  case JArr(Nil)      => "[]"
  case JArr(items)    => "[" + items.map(render).mkString(",") + "]"
  case JNull          => "null"
  // every subtype of Json is covered -- no case _ needed

val doc: Json = JArr(List(JNum(1), JStr("hi"), JNull, JArr(List(JNum(2)))))
println(render(doc))   // [1.0,"hi",null,[2.0]]
```

This is the single biggest reason to prefer `sealed trait` + `case class`
over an open class hierarchy for anything you plan to pattern match on: the
compiler becomes a safety net that catches missing cases the moment you add
a new variant, rather than at runtime via a `MatchError`.

## Nested pattern matching

Patterns compose — you can match several levels deep in one `case`, which is
usually clearer than chaining separate `match` expressions or option
lookups:

```scala
case class Address(city: String, zip: String)
case class Person(name: String, address: Option[Address])

def cityOf(p: Person): String = p match
  case Person(_, Some(Address(city, _))) => city
  case Person(name, None)                => s"$name has no address"

println(cityOf(Person("Ada", Some(Address("London", "E1")))))   // London
println(cityOf(Person("Bob", None)))                             // Bob has no address
```

`Person(_, Some(Address(city, _)))` destructures the `Person`, then the
`Option`, then the `Address` — all in a single pattern, binding only the
piece you actually need (`city`).

## `@` bindings: matching *and* naming the whole thing

Sometimes you want to match part of a structure but still keep a handle on
the whole value. The `name @ pattern` syntax binds `name` to whatever the
pattern matches, while still checking the pattern's shape:

```scala
def classifyPair(p: (Int, Int)): String = p match
  case (a, b) if a == b => "equal"
  case (a @ 0, _)       => s"first is zero (bound as $a)"
  case (a, b) if a > b  => "descending"
  case _                => "ascending"

println(classifyPair((3, 3)))   // equal
println(classifyPair((0, 5)))   // first is zero (bound as 0)
println(classifyPair((5, 2)))   // descending
println(classifyPair((2, 5)))   // ascending
```

## The trap: guards silently break exhaustiveness checking

A `case _ if someCondition =>` guard means the compiler can no longer prove
that case handles "everything else" — from the compiler's point of view, a
guarded case might not fire, so it still expects a plan for what happens if
every guard fails. Forgetting a final unguarded catch-all after a chain of
guarded cases is one of the most common sources of a runtime `MatchError` in
otherwise "exhaustive-looking" Scala code:

```scala
def sign(n: Int): String = n match
  case x if x > 0 => "positive"
  case x if x < 0 => "negative"
  // MatchError at runtime for n == 0 -- the compiler can't tell the guards
  // above are jointly exhaustive, so it won't warn you, and there's no
  // catch-all here to save you
```

Always give a guard chain a final unguarded `case _ =>`, exactly as you
would for matching on an open (non-sealed) type.

## Cheat sheet

| Pattern | What it needs | Binds |
|---|---|---|
| `case Even() =>` | `unapply` returning `Boolean` | nothing |
| `case AsPair(a, b) =>` | `unapply` returning `Option[(A, B)]` | `a`, `b` |
| `case List(a, b, rest*) =>` | `unapplySeq` returning `Option[Seq[T]]` | `a`, `b`, `rest` |
| `case Outer(Inner(x)) =>` | nested case class shapes | `x`, several levels deep |
| `case x @ Pattern =>` | any pattern | `x` bound to the whole matched value |
| `case _ if cond =>` | a guard | whatever the pattern before it binds |

## Exercise

Write an extractor `object Prime` with `def unapply(n: Int): Boolean` that
returns whether `n` is a prime number. Separately, model
`sealed trait Command` with `case class Move(dx: Int, dy: Int) extends Command`,
`case class Say(msg: String) extends Command`, and `case object Quit extends
Command`. Write `def run(commands: List[Command]): Unit` that pattern
matches on each command (print `"Moving by ($dx, $dy)"`, print the message
for `Say`, and print `"Bye!"` and stop processing further commands for
`Quit` — hint: use recursion or `takeWhile`/`span` rather than a mutable
loop). Test both with a handful of values, including at least one prime
checked via your `Prime` extractor inside a `match` on an `Int`.
