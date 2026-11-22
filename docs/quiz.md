---
title: "Final Quiz"
description: "Test what you have learned across all four levels of Scala Mastery Path with a 19-question final quiz."
---

# Final Quiz

19 questions drawn from all four levels of this course. You need **70% (14 out of 19)** to pass —
you can retake it as many times as you like. Your best result is saved in this browser.

<div id="mp-quiz-root"></div>

<script type="application/json" id="mp-quiz-data">
[
  {
    "question": "What does this print?\n```scala\n@main def m() =\n  val xs = List(1, 2, 3, 4)\n  println(xs.foldLeft(\"\")((acc, x) => acc + x))\n  println(xs.foldRight(\"\")((x, acc) => acc + x))\n  println(xs.map(_ + 1).filter(_ % 2 == 0).sum)\n```\n",
    "options": [
      "1234\n4321\n6",
      "4321\n1234\n6",
      "1234\n4321\n10",
      "1234\n1234\n6"
    ],
    "correctIndex": 0,
    "explanation": "foldLeft processes left to right; foldRight starts from the right. Verified by compiling and running it."
  },
  {
    "question": "What does this print?\n```scala\nsealed trait Shape\ncase class Circle(r: Double) extends Shape\ncase class Sq(s: Double) extends Shape\ndef area(s: Shape) = s match\n  case Circle(r) => 3 * r * r\n  case Sq(x) => x * x\n@main def m() =\n  println(List(Circle(1), Sq(2)).map(area))\n  println(Circle(1) == Circle(1.0))\n```\n",
    "options": [
      "List(3, 4)\ntrue",
      "List(3.0, 4.0)\ntrue",
      "List(Circle(1.0), Sq(2.0))\ntrue",
      "List(3.0, 4.0)\nfalse"
    ],
    "correctIndex": 1,
    "explanation": "Case classes compare structurally; the match on a sealed trait is checked for exhaustiveness. Verified by compiling and running it."
  },
  {
    "question": "What does this print?\n```scala\n@main def m() =\n  val r = scala.util.Try(\"12x\".toInt)\n  println(r.isFailure)\n  println(r.getOrElse(0))\n  println(scala.util.Try(10 / 2).toOption)\n  println(\"12\".toIntOption.map(_ * 2))\n```\n",
    "options": [
      "true\n0\nSome(5)\nSome(24)",
      "true\nNone\nSome(5)\nSome(24)",
      "false\n0\nSome(5)\nSome(24)",
      "true\n0\n5\nSome(24)"
    ],
    "correctIndex": 0,
    "explanation": "Try captures exceptions as values; toIntOption avoids them altogether. Verified by compiling and running it."
  },
  {
    "question": "What does this print?\n```scala\ndef counter(): () => Int =\n  var n = 0\n  () => { n += 1; n }\n@main def m() =\n  val c = counter()\n  c(); c()\n  println(c())\n  val d = counter()\n  println(d())\n```\n",
    "options": [
      "3\n1",
      "3\n3",
      "3\n4",
      "1\n1"
    ],
    "correctIndex": 0,
    "explanation": "Each call to counter creates its own captured variable. Verified by compiling and running it."
  },
  {
    "question": "What does this print?\n```scala\n@main def m() =\n  val words = List(\"apple\", \"bob\", \"cat\", \"apple\", \"bob\", \"apple\")\n  val freq = words.groupMapReduce(identity)(_ => 1)(_ + _)\n  println(freq.toList.sortBy((w, n) => (-n, w)))\n  println(words.distinct.size)\n```\n",
    "options": [
      "List((apple,3), (bob,2), (cat,1))\n3",
      "List((apple,3), (bob,2), (cat,1))\n6",
      "List((apple,3), (bob,2))\n3",
      "List((cat,1), (bob,2), (apple,3))\n3"
    ],
    "correctIndex": 0,
    "explanation": "groupMapReduce groups, maps and reduces in one pass. Verified by compiling and running it."
  },
  {
    "question": "Why make domain types sealed ADTs?",
    "options": [
      "The compiler checks that matches are exhaustive, making illegal states unrepresentable",
      "Smaller",
      "Required",
      "Faster"
    ],
    "correctIndex": 0,
    "explanation": "New variants surface as warnings."
  },
  {
    "question": "When should you use Either rather than throwing an exception?",
    "options": [
      "Always throw",
      "Never",
      "For bugs",
      "For expected, recoverable failures that callers must handle"
    ],
    "correctIndex": 3,
    "explanation": "Exceptions are for the exceptional."
  },
  {
    "question": "What does Future.traverse do?",
    "options": [
      "Blocks",
      "Sorts",
      "Applies a function returning futures to each element and combines the results into one future",
      "Cancels"
    ],
    "correctIndex": 2,
    "explanation": "Parallel when the futures start eagerly."
  },
  {
    "question": "Why avoid Await.result inside request handlers?",
    "options": [
      "It blocks threads and can exhaust the pool",
      "Syntax",
      "Law",
      "Colour"
    ],
    "correctIndex": 0,
    "explanation": "Compose futures instead."
  },
  {
    "question": "What is the idiomatic way to share state between actors?",
    "options": [
      "Locks",
      "Send immutable messages; do not share mutable state",
      "Globals",
      "Shared var"
    ],
    "correctIndex": 1,
    "explanation": "The actor model."
  },
  {
    "question": "What does a type class like Ordering[A] allow?",
    "options": [
      "Behaviour for a type chosen by an implicit/given instance, even for types you do not own",
      "Reflection",
      "Macros",
      "Inheritance"
    ],
    "correctIndex": 0,
    "explanation": "Ad hoc polymorphism."
  },
  {
    "question": "What is the purpose of opaque types in Scala 3?",
    "options": [
      "Macros",
      "Hiding code",
      "Speed",
      "Distinct, zero-cost wrapper types such as UserId over Long"
    ],
    "correctIndex": 3,
    "explanation": "Avoid mixing IDs."
  },
  {
    "question": "What is the point of idempotency for message handlers?",
    "options": [
      "At-least-once delivery means a message may be processed twice",
      "Order",
      "Speed",
      "Law"
    ],
    "correctIndex": 0,
    "explanation": "Use unique keys."
  },
  {
    "question": "What does a circuit breaker protect?",
    "options": [
      "Logs",
      "Your service and a failing dependency by temporarily stopping calls",
      "Wires",
      "Caches"
    ],
    "correctIndex": 1,
    "explanation": "Fail fast."
  },
  {
    "question": "How should secrets reach a Scala service in production?",
    "options": [
      "Through environment or a secret manager, never committed",
      "In logs",
      "In code",
      "In git"
    ],
    "correctIndex": 0,
    "explanation": "Rotate."
  },
  {
    "question": "Why use a multi-stage Docker build?",
    "options": [
      "Style",
      "Law",
      "Speed",
      "Compile with sbt in one stage, ship only the jar on a slim JRE image"
    ],
    "correctIndex": 3,
    "explanation": "Smaller images."
  },
  {
    "question": "What does sbt %% do?",
    "options": [
      "Override",
      "Both",
      "Percent",
      "Appends the Scala binary version to the artifact name"
    ],
    "correctIndex": 3,
    "explanation": "Cross-versioned artifacts."
  },
  {
    "question": "What evidence shows a Scala service is production-ready?",
    "options": [
      "Compiles",
      "CI with tests and scans, load testing, metrics, logs and a runbook",
      "Few lines",
      "Works locally"
    ],
    "correctIndex": 1,
    "explanation": "Evidence over claims."
  },
  {
    "question": "Which collection suits many appends and random reads?",
    "options": [
      "List",
      "Stream",
      "Array only",
      "Vector"
    ],
    "correctIndex": 3,
    "explanation": "List excels at prepend."
  }
]
</script>

## Coding challenges

Try these on your own computer, then compare your output. Nothing you write is sent anywhere.

### An inventory with typed errors

Implement apply and parse for a small inventory: commands update an immutable Map, failures are values (Either) and the fold keeps the last good stock when a command fails. This is ordinary Scala 3 code you run on your own machine.

**Starter code**

```scala
enum Cmd:
  case Add(sku: String, qty: Int)
  case Remove(sku: String, qty: Int)

enum InvError:
  case BadQty(n: Int)
  case Unknown(sku: String)
  case Insufficient(sku: String, have: Int, want: Int)

import Cmd.*, InvError.*

type Stock = Map[String, Int]

// TODO: def apply(stock: Stock, cmd: Cmd): Either[InvError, Stock]
//   Add / Remove with qty <= 0 -> Left(BadQty(qty)); Remove of a missing sku -> Left(Unknown(sku));
//   Remove of more than available -> Left(Insufficient(sku, have, want)); otherwise Right(updated stock)
// TODO: def parse(line: String): Option[Cmd]   ("add apple 10" / "remove apple 3"; anything else, including non-numeric qty, is None)

@main def run() =
  val lines = List("add apple 10", "add pear 4", "remove apple 3", "remove kiwi 1", "remove pear 9", "add fig 0", "oops", "add apple 5")
  val (finalStock, errors) = lines.foldLeft((Map.empty: Stock, List.empty[String])) { case ((st, errs), line) =>
    parse(line) match
      case None => (st, errs :+ s"unparsable: $line")
      case Some(c) => apply(st, c) match
        case Right(s2) => (s2, errs)
        case Left(e) => (st, errs :+ e.toString)
  }
  println(finalStock.toList.sortBy(_._1))
  errors.foreach(println)
  println(s"errors=${errors.size}")
```

**Expected output**

```text
List((apple,12), (pear,4))
Unknown(kiwi)
Insufficient(pear,4,9)
BadQty(0)
unparsable: oops
errors=4
```

<details><summary>Show solution and explanation</summary>

<pre><code>enum Cmd:
  case Add(sku: String, qty: Int)
  case Remove(sku: String, qty: Int)

enum InvError:
  case BadQty(n: Int)
  case Unknown(sku: String)
  case Insufficient(sku: String, have: Int, want: Int)

import Cmd.*, InvError.*

type Stock = Map[String, Int]

def apply(stock: Stock, cmd: Cmd): Either[InvError, Stock] = cmd match
  case Add(sku, q) if q &lt;= 0 =&gt; Left(BadQty(q))
  case Add(sku, q) =&gt; Right(stock.updated(sku, stock.getOrElse(sku, 0) + q))
  case Remove(sku, q) if q &lt;= 0 =&gt; Left(BadQty(q))
  case Remove(sku, q) =&gt;
    stock.get(sku) match
      case None =&gt; Left(Unknown(sku))
      case Some(h) if h &lt; q =&gt; Left(Insufficient(sku, h, q))
      case Some(h) =&gt; Right(stock.updated(sku, h - q))

def parse(line: String): Option[Cmd] = line.split(&quot; &quot;).toList match
  case &quot;add&quot; :: sku :: q :: Nil =&gt; q.toIntOption.map(Add(sku, _))
  case &quot;remove&quot; :: sku :: q :: Nil =&gt; q.toIntOption.map(Remove(sku, _))
  case _ =&gt; None

@main def run() =
  val lines = List(&quot;add apple 10&quot;, &quot;add pear 4&quot;, &quot;remove apple 3&quot;, &quot;remove kiwi 1&quot;, &quot;remove pear 9&quot;, &quot;add fig 0&quot;, &quot;oops&quot;, &quot;add apple 5&quot;)
  val (finalStock, errors) = lines.foldLeft((Map.empty: Stock, List.empty[String])) { case ((st, errs), line) =&gt;
    parse(line) match
      case None =&gt; (st, errs :+ s&quot;unparsable: $line&quot;)
      case Some(c) =&gt; apply(st, c) match
        case Right(s2) =&gt; (s2, errs)
        case Left(e) =&gt; (st, errs :+ e.toString)
  }
  println(finalStock.toList.sortBy(_._1))
  errors.foreach(println)
  println(s&quot;errors=${errors.size}&quot;)
</code></pre>

<p>Each command returns Either, so errors never throw. The foldLeft threads (stock, errors) through, keeping the previous stock when a command is rejected. Verified by compiling and running with Scala 3.3.1.</p>
</details>

### Parallel word count with retries

Count words in several documents concurrently with Futures, retrying a flaky step with recoverWith, then merge the maps. Futures run on the global execution context; Await is used only at the program edge so the result can be printed.

**Starter code**

```scala
import scala.concurrent.{Await, Future}
import scala.concurrent.ExecutionContext.Implicits.global
import scala.concurrent.duration.*

val docs = Vector(
  "the quick brown fox",
  "the lazy dog and the fox",
  "quick quick slow",
  "a dog a fox a cat"
)

// TODO: def countWords(doc: String): Map[String, Int]
// TODO: def merge(a: Map[String, Int], b: Map[String, Int]): Map[String, Int]   (sum counts)
// TODO: def withRetry(doc: String, max: Int): Future[Map[String, Int]]   (retry flaky(doc, n) with n = 0, 1, ... up to max)
def flaky(doc: String, attempts: Int): Future[Map[String, Int]] =
  Future { if doc.startsWith("quick") && attempts < 2 then throw RuntimeException("busy") else countWords(doc) }

@main def run() =
  val total = Await.result(Future.traverse(docs)(d => withRetry(d, 3)).map(_.reduce(merge)), 5.seconds)
  println(total.toList.sortBy((w, n) => (-n, w)).take(4))
  println(total.values.sum)
  val failing = Await.result(withRetry("quick fail", 1).failed.map(_.getMessage), 2.seconds)
  println(failing)
```

**Expected output**

```text
List((a,3), (fox,3), (quick,3), (the,3))
19
busy
```

<details><summary>Show solution and explanation</summary>

<pre><code>import scala.concurrent.{Await, Future}
import scala.concurrent.ExecutionContext.Implicits.global
import scala.concurrent.duration.*

val docs = Vector(
  &quot;the quick brown fox&quot;,
  &quot;the lazy dog and the fox&quot;,
  &quot;quick quick slow&quot;,
  &quot;a dog a fox a cat&quot;
)

def countWords(doc: String): Map[String, Int] =
  doc.split(&quot; &quot;).toList.groupBy(identity).view.mapValues(_.size).toMap

def merge(a: Map[String, Int], b: Map[String, Int]): Map[String, Int] =
  b.foldLeft(a) { case (acc, (w, n)) =&gt; acc.updated(w, acc.getOrElse(w, 0) + n) }

def flaky(doc: String, attempts: Int): Future[Map[String, Int]] =
  Future { if doc.startsWith(&quot;quick&quot;) &amp;&amp; attempts &lt; 2 then throw RuntimeException(&quot;busy&quot;) else countWords(doc) }

def withRetry(doc: String, max: Int): Future[Map[String, Int]] =
  def go(n: Int): Future[Map[String, Int]] =
    flaky(doc, n).recoverWith { case _ if n &lt; max =&gt; go(n + 1) }
  go(0)

@main def run() =
  val total = Await.result(Future.traverse(docs)(d =&gt; withRetry(d, 3)).map(_.reduce(merge)), 5.seconds)
  println(total.toList.sortBy((w, n) =&gt; (-n, w)).take(4))
  println(total.values.sum)
  val failing = Await.result(withRetry(&quot;quick fail&quot;, 1).failed.map(_.getMessage), 2.seconds)
  println(failing)
</code></pre>

<p>Future.traverse starts all documents, recoverWith retries a failing future up to max times and merge sums counts. A document that keeps failing surfaces its exception as a failed future. Verified by compiling and running with Scala 3.3.1.</p>
</details>
