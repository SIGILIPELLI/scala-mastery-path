# 07 · Working with JSON

The [Level 1 project](../level-1/10-project-todo-app.md) stored tasks in a
hand-rolled pipe-delimited text format specifically to avoid needing JSON
before it was covered. This module fills that gap using
[upickle](https://com-lihaoyi.github.io/upickle/), a small, dependency-light
JSON library for Scala. (Its sibling library [circe](https://circe.github.io/circe/)
is the other library you'll see often in production Scala codebases — same
ideas, more type-class-heavy API — but upickle's `derives` syntax is the
gentler on-ramp.)

## Adding upickle to a project

```scala
// build.sbt
libraryDependencies += "com.lihaoyi" %% "upickle" % "3.3.1"
```

## Deriving JSON for a case class

upickle can generate a `ReadWriter` for any case class automatically with
Scala 3's `derives` clause — no manual encoder/decoder boilerplate:

```scala
import upickle.default._

case class Address(city: String, zip: String) derives ReadWriter
case class Person(name: String, age: Int, address: Option[Address]) derives ReadWriter

val p = Person("Ada", 30, Some(Address("London", "E1")))

val json = write(p)
println(json)
// {"name":"Ada","age":30,"address":[{"city":"London","zip":"E1"}]}

val parsedBack = read[Person](json)
println(parsedBack)              // Person(Ada,30,Some(Address(London,E1)))
println(parsedBack == p)         // true -- case class equality makes round-tripping easy to verify
```

`write` serializes any type with a `ReadWriter` in scope to a JSON string;
`read[T]` parses a JSON string back into `T`, given the exact type
annotation (upickle needs `read[Person]`, not just `read`, since the target
type can't be inferred from a `String`).

## The trap: `Option` doesn't serialize the way you'd guess

It's natural to assume `Some(x)` becomes `x` or `null` and `None` becomes a
missing key or `null`. upickle actually represents `Option[T]` as a
**JSON array of zero or one elements** — `[]` for `None`, `[value]` for
`Some(value)`:

```scala
val noAddress = Person("Bob", 25, None)
println(write(noAddress))
// {"name":"Bob","age":25,"address":[]}

val roundTripped = read[Person](write(noAddress))
println(roundTripped)   // Person(Bob,25,None) -- round-trips correctly...

// ...but if you're consuming JSON from a NON-Scala API (a real HTTP
// endpoint, not one you serialized with upickle yourself), it almost
// certainly represents "no address" as either a missing "address" key or
// "address": null -- neither of which upickle's default Option encoding
// expects. Model fields coming from external APIs as plain (non-Option)
// types with sensible defaults, or write a custom ReadWriter, rather than
// assuming Option "just works" against arbitrary JSON.
```

This is the single most common surprise when picking up upickle: it's
perfectly consistent *for JSON upickle produced itself*, but it is not the
"idiomatic JSON" convention (missing key / `null`) most external APIs use
for optional fields.

## Pretty-printing

```scala
println(write(p, indent = 2))
// {
//   "name": "Ada",
//   "age": 30,
//   "address": [
//     {
//       "city": "London",
//       "zip": "E1"
//     }
//   ]
// }
```

## Parsing JSON you don't have a case class for

Sometimes you just need to pull a couple of fields out of a response
without modeling the whole shape. `ujson.read` parses into a generic,
dynamically-navigable `ujson.Value` tree:

```scala
val raw = """{"name":"Cleo","age":5,"tags":["cat","fluffy"]}"""
val parsed = ujson.read(raw)

println(parsed("name").str)                        // Cleo
println(parsed("age").num)                          // 5.0 -- JSON numbers are Double by default
println(parsed("tags").arr.map(_.str).mkString(","))// cat,fluffy
```

`.str`, `.num`, `.arr`, `.obj`, and `.bool` unwrap a `ujson.Value` to the
Scala type you expect — and throw if the value is actually a different
JSON type, which is the generic-tree equivalent of `Option.get`: fine for a
quick script, risky against JSON you don't fully control. For anything
long-lived, model it as a case class with `derives ReadWriter` instead so
the compiler checks the shape for you.

## Handling malformed or unexpected JSON

Reading into a typed case class throws when the input doesn't match —
combine it with `Try` (from [Module 4](04-option-either.md)) to turn a
parse failure into an `Option`/`Either` instead of an uncaught exception:

```scala
import scala.util.Try

def parsePerson(json: String): Either[String, Person] =
  Try(read[Person](json)).toEither.left.map(_ => s"invalid person JSON: $json")

println(parsePerson("""{"name":"Ada","age":30,"address":[]}"""))
// Right(Person(Ada,30,None))

println(parsePerson("""{"name":"NoAge"}"""))
// Left(invalid person JSON: {"name":"NoAge"})
```

Wrapping every external `read[T]` call this way is worth the small amount of
ceremony — it turns "a malformed response crashes the program" into "a
malformed response is a value the caller has to explicitly handle," exactly
the philosophy Module 4 built up.

## Cheat sheet

| Task | Code |
|---|---|
| Derive JSON support for a case class | `case class Foo(...) derives ReadWriter` |
| Serialize to a JSON string | `write(value)` |
| Pretty-print | `write(value, indent = 2)` |
| Parse into a known type | `read[Foo](jsonString)` |
| Parse into a generic tree | `ujson.read(jsonString)` |
| Navigate a generic tree | `.obj`, `.arr`, `.str`, `.num`, `.bool`, `parsed("key")` |
| Handle parse failure as data | `Try(read[Foo](s)).toEither` |

## Exercise

Define `case class Book(title: String, author: String, year: Int, tags: List[String]) derives ReadWriter`.
Create a `List[Book]` with at least three entries, serialize the whole list
with `write` (hint: `write(books)` works directly on a `List[Book]` since
upickle can derive collection support once the element type has a
`ReadWriter`), then parse it back with `read[List[Book]]` and confirm it
equals the original list. Then take a hand-written JSON string missing the
`year` field and confirm `read[Book]` throws — wrap that call in a `Try`
and print a friendly error message instead of letting the exception
propagate.
