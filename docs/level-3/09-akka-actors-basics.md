# 09 · Akka Actors Basics

`Future` (module 01) models one asynchronous value. **Akka actors** model
asynchronous, stateful *objects* that process one message at a time, never
share mutable state with each other, and communicate only by sending
messages — a different, complementary answer to "how do I write concurrent
code without locks and race conditions."

```scala title="build.sbt"
scalaVersion := "3.3.1"
libraryDependencies += "com.typesafe.akka" %% "akka-actor-typed" % "2.8.5"
```

## The actor model in one sentence

An actor has: a mailbox (a queue of incoming messages), a behavior (what to
do with the next message), and the ability to send messages to other
actors, create new actors, or change its own behavior for the *next*
message. Crucially, an actor processes messages **one at a time, in order**
— there's no way for two messages to run concurrently against the same
actor's state, which is what eliminates the shared-mutable-state race
conditions that make raw multithreading hard.

## Defining and running a first actor

```scala
import akka.actor.typed.{ActorSystem, Behavior}
import akka.actor.typed.scaladsl.Behaviors

object Greeter:
  sealed trait Command
  case class Greet(name: String) extends Command

  def apply(): Behavior[Command] = Behaviors.receive { (ctx, msg) =>
    msg match
      case Greet(name) =>
        println(s"Hello, $name!")
        Behaviors.same
  }

val system = ActorSystem(Greeter(), "greeter-system")
system ! Greeter.Greet("Ada")
Thread.sleep(200)
system.terminate()
// Hello, Ada!
```

Every actor's protocol is a sealed trait of `Command`s it accepts — Akka
Typed (the modern API) enforces this at compile time, so you can't
accidentally send an actor a message its behavior doesn't handle. `!` (the
"tell" operator) sends a message and returns immediately — actor
communication is fire-and-forget by default, same spirit as
`onComplete` on a `Future`. `Behaviors.same` means "keep this exact
behavior for the next message" — the actor doesn't change how it responds.

### The trap: actors don't run synchronously

That `Thread.sleep(200)` after `system ! Greeter.Greet("Ada")` exists only
because this is a `main` method that would otherwise exit before the
message is processed on its own thread. `!` never blocks and never
guarantees the message has been handled by the time the next line runs —
production code reacts to actor results via further messages or the `ask`
pattern below, never by sleeping and hoping.

## Actors with internal state

An actor "changes behavior" by returning a *different* `Behavior` value
from its message handler — this is how actors hold state without mutable
`var`s, using recursion instead:

```scala
import akka.actor.typed.ActorRef

object Counter:
  sealed trait Command
  case object Increment extends Command
  case class GetCount(replyTo: ActorRef[Int]) extends Command

  def apply(): Behavior[Command] = counting(0)

  private def counting(count: Int): Behavior[Command] = Behaviors.receive { (ctx, msg) =>
    msg match
      case Increment =>
        counting(count + 1)          // returns a NEW behavior closing over count + 1
      case GetCount(replyTo) =>
        replyTo ! count
        Behaviors.same
  }
```

Each `Increment` doesn't mutate a field — it returns `counting(count + 1)`,
a fresh behavior value that closes over the incremented count and becomes
what handles the *next* message. There's no `var` anywhere, yet the actor
is stateful — this is the same "state as an immutable value passed
forward" idea as a recursive accumulator, just driven by incoming messages
instead of a loop.

## Getting an answer back: the `ask` pattern

`GetCount` above carries an `ActorRef[Int]` — a reference the actor can
reply to. Wiring that up manually gets verbose, so Akka provides `ask`,
which returns a `Future` for the reply:

```scala
import akka.actor.typed.scaladsl.AskPattern._
import akka.util.Timeout
import scala.concurrent.duration._
import scala.concurrent.Await

counterSystem ! Counter.Increment
counterSystem ! Counter.Increment
counterSystem ! Counter.Increment

given Timeout = 3.seconds
given akka.actor.typed.Scheduler = counterSystem.scheduler

val countFuture: Future[Int] = counterSystem.ask[Int](replyTo => Counter.GetCount(replyTo))
val count = Await.result(countFuture, 3.seconds)
println(s"count: $count")   // count: 3
```

`ask` needs an implicit `Timeout` (in case the actor never replies) and a
`Scheduler` in scope, and returns a `Future[Int]` that completes when the
actor sends its reply — bridging the actor world back into the `Future`
world from module 01 whenever you need one specific answer rather than
an ongoing stream of messages.

### The trap: `ask` everywhere defeats the point of actors

Reaching for `ask` (and blocking on the resulting `Future`) for every
interaction reintroduces synchronous, request/response thinking into a
system designed around asynchronous message flows — and each `ask` has
real overhead (a temporary actor to receive the reply, a timeout). Prefer
plain `!` (tell) and modeling the *response* as another message to a
known recipient wherever the caller doesn't need to block for a single
answer; reserve `ask` for edges of the actor system, like an HTTP handler
that genuinely needs a value before it can respond to its own caller.

## Cheat sheet

| Need to... | Use |
|---|---|
| Define an actor's message protocol | a sealed trait of case classes/objects extending `Command` |
| Define behavior | `Behaviors.receive { (ctx, msg) => ... }` returning a `Behavior[Command]` |
| Start an actor system | `ActorSystem(rootBehavior, "name")` |
| Send a message, don't wait | `actorRef ! message` |
| Keep the same behavior for the next message | `Behaviors.same` |
| Hold state without `var` | return a new behavior value closing over updated state |
| Get one reply as a `Future` | `actorRef.ask[Reply](replyTo => Message(replyTo))` (needs `Timeout` + `Scheduler`) |
| Shut everything down | `system.terminate()` |

## Exercise

Build a `Bank` actor with commands `Deposit(amount: Int)`, `Withdraw(amount:
Int, replyTo: ActorRef[Boolean])` (replies `false` and leaves the balance
unchanged if funds are insufficient), and `GetBalance(replyTo:
ActorRef[Int])`. Model the balance as recursive-behavior state, the same
way `Counter` does. Start an `ActorSystem`, deposit some amount, attempt an
over-limit withdrawal and confirm via `ask` it returns `false` with the
balance unchanged, then withdraw a valid amount and confirm the balance
via `ask`.
