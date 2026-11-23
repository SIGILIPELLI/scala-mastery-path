---
title: "Scala Cheat Sheet: Quick Reference, Beginner to Advanced"
description: "A free Scala cheat sheet: 35 quick-reference tables from every level of the Scala Mastery Path, each linked to its full lesson."
---

# Scala Cheat Sheet

Quick-reference tables from **35 lessons** of the free Scala Mastery Path, in course order from beginner to advanced. Each section links to the full lesson with explanations and runnable examples.

## Level 1 · Entry

### [02 · Variables &amp; Types](level-1/02-variables-types.md)

| Type | Example | Notes |
|------|---------|-------|
| `Int` | `42` | 32-bit signed integer, most common numeric type |
| `Long` | `42L` | 64-bit integer |
| `Double` | `3.14` | 64-bit floating point, default for decimals |
| `Float` | `3.14f` | 32-bit floating point |
| `Boolean` | `true` / `false` | |
| `Char` | `'A'` | single character, single quotes |
| `String` | `"hello"` | double quotes |
| `Unit` | — | "no meaningful value," like `void` |

### [03 · Control Flow](level-1/03-control-flow.md)

| Construct | Purpose |
|-----------|---------|
| `if ... then ... else ...` | Conditional expression, returns a value |
| `while ... do ...` | Imperative loop, needs a `var` to be useful |
| `for i <- range do ...` | Loop over a collection for side effects |
| `for i <- range yield ...` | Build a new collection from a loop |
| `x match { case ... }` | Multi-way branching on a value's shape |

### [04 · Functions](level-1/04-functions.md)

| Syntax | Meaning |
|--------|---------|
| `def f(x: Int): Int = ...` | Named function with an explicit return type |
| `def f(x: Int = 5) = ...` | Default parameter value |
| `f(x = 1, y = 2)` | Named arguments |
| `def f(a: Int)(b: Int) = ...` | Multiple parameter lists (curried) |
| `val f: Int => Int = n => n * 2` | Function value with an explicit function type |
| `_ * 2` | Placeholder syntax for a simple one-argument lambda |
| `@tailrec` | Compiler-checked tail-call optimization |

### [05 · Collections Basics](level-1/05-collections-basics.md)

| Collection | Ordered? | Duplicates? | Key operations |
|------------|----------|-------------|-----------------|
| `List[A]` | Yes | Yes | `head`, `tail`, `:+`, `::`, `map`, `filter` |
| `Map[K, V]` | No (by default) | Unique keys | `apply`, `getOrElse`, `get`, `+`, `-` |
| `Set[A]` | No | No | `contains`, `+`, `-`, `union`, `intersect`, `diff` |
| `(A, B)` tuple | fixed arity | — | `._1`, `._2`, destructuring |

### [06 · Classes &amp; Objects Basics](level-1/06-classes-objects.md)

| Concept | Syntax | Purpose |
|---------|--------|---------|
| Class | `class Foo(val x: Int)` | Blueprint for objects, with constructor params as fields |
| Auxiliary constructor | `def this(...) = this(...)` | Alternate ways to build an instance |
| Singleton | `object Foo` | Exactly one instance, `static`-like members |
| Companion object | `object Foo` next to `class Foo` | Factory methods, constants tied to the class |
| `apply` | `def apply(...) = ...` | Lets `Foo(...)` work without `new` |
| `private` | `private val x = ...` | Restricts visibility to the class (+ companion) |

### [08 · Pattern Matching Intro](level-1/08-pattern-matching-intro.md)

| Pattern | Matches |
|---------|---------|
| `case 3 =>` | The literal value `3` |
| `case 3 \| 4 =>` | Either `3` or `4` |
| `case i: Int =>` | Any value of type `Int`, bound to `i` |
| `case Point(x, y) =>` | A `Point`, binding its fields to `x` and `y` |
| `case x :: rest =>` | A non-empty list, binding head and tail |
| `case (a, b) =>` | A 2-tuple, binding both elements |
| `case x if cond =>` | Adds a guard condition after the shape matches |
| `case _ =>` | Matches anything (catch-all) |

### [09 · Traits Basics](level-1/09-traits-basics.md)

| Syntax | Meaning |
|--------|---------|
| `trait Foo` | Defines a mixable contract |
| `class Bar extends Foo` | `Bar` implements/inherits `Foo` |
| `class Bar extends Foo with Baz` | `Bar` mixes in both `Foo` and `Baz` |
| `sealed trait Foo` | All subtypes must be declared in this file — enables exhaustive `match` |
| `override def x = ...` | Provide/replace a member from a parent trait or class |

## Level 2 · Intermediate

### [01 · OOP Deep Dive](level-2/01-oop-deep-dive.md)

| Concept | What it means |
|---|---|
| Linearization | The order Scala flattens a `with` chain into for method dispatch |
| Rightmost trait wins | In `extends A with B with C`, `C`'s override is the entry point |
| `super` in a trait | Forwards to the *next* trait leftward in the linearization, not a fixed parent |
| `abstract override` | Marks an override that requires a real implementation supplied further down the chain |
| Self-type (`self: T =>`) | Declares "this trait can only be used where `T` is also present," without extending `T` |
| `sealed trait` + `case class`es | An algebraic data type — exhaustiveness-checked, data-carrying hierarchy |
| Private constructor + companion factory | "Smart constructor" — the only path to a valid instance |

### [02 · Pattern Matching Advanced](level-2/02-pattern-matching-advanced.md)

| Pattern | What it needs | Binds |
|---|---|---|
| `case Even() =>` | `unapply` returning `Boolean` | nothing |
| `case AsPair(a, b) =>` | `unapply` returning `Option[(A, B)]` | `a`, `b` |
| `case List(a, b, rest*) =>` | `unapplySeq` returning `Option[Seq[T]]` | `a`, `b`, `rest` |
| `case Outer(Inner(x)) =>` | nested case class shapes | `x`, several levels deep |
| `case x @ Pattern =>` | any pattern | `x` bound to the whole matched value |
| `case _ if cond =>` | a guard | whatever the pattern before it binds |

### [03 · Collections Deep Dive](level-2/03-collections-deep-dive.md)

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

### [04 · Option/Either for Error Handling](level-2/04-option-either.md)

| Type | Success case | Failure case | Carries a reason? |
|---|---|---|---|
| `Option[A]` | `Some(value)` | `None` | no |
| `Either[L, R]` | `Right(value)` | `Left(error)` | yes |
| `Try[A]` | `Success(value)` | `Failure(exception)` | yes (an exception) |

| Need to... | Use |
|---|---|
| Provide a fallback value | `.getOrElse(default)` |
| Transform only if present/successful | `.map(f)` |
| Chain calls that themselves return `Option`/`Either` | `.flatMap(f)` or a `for`-comprehension |
| Convert between the three | `.toOption`, `.toEither` |
| Drop `None`s from a `List[Option[A]]` | `.flatten` |
| Run code that might throw, capture the exception | `Try(...)` |

### [05 · Testing with ScalaTest](level-2/05-scalatest-testing.md)

| Style | Test declared as | Reads like |
|---|---|---|
| `AnyFunSuite` | `test("description") { ... }` | JUnit-style, plain description |
| `AnyFlatSpec` | `"subject" should "behavior" in { ... }` | a sentence |

| Matcher | Checks |
|---|---|
| `x shouldBe y` | equality (also works for `Some`/`None`/booleans) |
| `x should equal(y)` | equality, allows custom equality overrides |
| `xs should contain(y)` | collection membership |
| `xs should have length n` / `have size n` | collection size |
| `xs shouldBe sorted` | ordering |
| `opt shouldBe defined` / `shouldBe empty` | `Option` state |
| `an [Ex] should be thrownBy expr` | exception assertion (`AnyFlatSpec`) |
| `assertThrows[Ex] { expr }` | exception assertion (any style) |

### [06 · Implicits Basics](level-2/06-implicits-basics.md)

| Scala 3 keyword | Old (Scala 2) name | Purpose |
|---|---|---|
| `given` (value) | `implicit val`/`implicit object` | Provide a value the compiler can inject |
| `using` (parameter) | `implicit` parameter | Mark a parameter to be filled in automatically |
| `given Conversion[A, B] with ...` | `implicit def` (A => B) | Automatic type conversion |
| `extension (x: T) def foo = ...` | `implicit class` | Add a method to an existing type |

### [07 · Working with JSON](level-2/07-working-with-json.md)

| Task | Code |
|---|---|
| Derive JSON support for a case class | `case class Foo(...) derives ReadWriter` |
| Serialize to a JSON string | `write(value)` |
| Pretty-print | `write(value, indent = 2)` |
| Parse into a known type | `read[Foo](level-2/jsonString)` |
| Parse into a generic tree | `ujson.read(jsonString)` |
| Navigate a generic tree | `.obj`, `.arr`, `.str`, `.num`, `.bool`, `parsed("key")` |
| Handle parse failure as data | `Try(read[Foo](level-2/s)).toEither` |

### [08 · sbt Deep Dive](level-2/08-sbt-deep-dive.md)

| Concept | Syntax | Purpose |
|---|---|---|
| Cross-versioned dependency | `"org" %% "lib" % "1.0"` | Resolves the right Scala-version build automatically |
| Test-only dependency | `"org" %% "lib" % "1.0" % Test` | Available only under `src/test/scala` |
| Multi-module project | `lazy val x = (project in file("x"))` | One module, own source tree and settings |
| Code dependency between modules | `.dependsOn(other)` | Lets this module import the other's classes |
| Command fan-out across modules | `.aggregate(a, b)` | Running a task on the parent runs it on `a` and `b` too |
| Add a plugin | `project/plugins.sbt` → `addSbtPlugin(...)` | New sbt-wide tasks/settings |
| Custom task | `lazy val t = taskKey[Unit](level-2/"...")` + `t := { ... }` | Your own `sbt <name>` command |
| Run a specific module | `sbt "modulename/run"` | Avoids ambiguity in multi-module builds |

### [09 · For-Comprehensions](level-2/09-for-comprehensions.md)

| `for` syntax | Desugars to |
|---|---|
| `for x <- xs yield f(x)` (last/only generator) | `xs.map(f)` |
| `for x <- xs; y <- ys yield ...` | `xs.flatMap(x => ys.map(y => ...))` |
| `for x <- xs if cond yield f(x)` | `xs.withFilter(cond).map(f)` |
| `for x <- xs do sideEffect(x)` (no `yield`) | `xs.foreach(sideEffect)` |

## Level 3 · Advanced

### [01 · Futures &amp; Concurrency](level-3/01-futures-concurrency.md)

| Need to... | Use |
|---|---|
| Run work asynchronously | `Future { ... }` (needs an `ExecutionContext` in scope) |
| React without blocking | `.onComplete { case Success(v) => ...; case Failure(e) => ... }` |
| Transform a future value | `.map(f)` |
| Chain futures that depend on each other | `.flatMap(f)` or a `for`-comprehension |
| Run independent futures concurrently | start them *before* the `for`, or use `Future.sequence` |
| Block for a result (tests/`main` only) | `Await.result(f, timeout)` |
| Recover from failure | `.recover { case e => fallback }` / `.recoverWith { case e => otherFuture }` |
| Combine a list of futures | `Future.sequence(listOfFutures)` |

### [02 · Building APIs](level-3/02-building-apis.md)

| Need to... | Use |
|---|---|
| Start a server on a port | `HttpServer.create(new InetSocketAddress(port), 0)` |
| Attach a handler to a path prefix | `server.createContext(path, handler)` |
| Read query params / path | `exchange.getRequestURI.getQuery` / `.getPath` |
| Send a response | `sendResponseHeaders(status, len)` then write + **close** `getResponseBody` |
| Make an HTTP call | `HttpClient.newHttpClient().send(request, BodyHandlers.ofString())` |
| Route by method + path declaratively | pattern match on `(method, path)`, or use http4s/akka-http in production |
| Model API outcomes as data | an `enum` mapped to `(status, body)` in one place |

### [03 · Databases](level-3/03-databases.md)

| Need to... | Use |
|---|---|
| Load a JDBC driver and connect | `Class.forName(driver)`, `DriverManager.getConnection(url)` |
| Run `CREATE`/`INSERT`/`UPDATE`/`DELETE` | `Statement.executeUpdate(sql)` |
| Run `SELECT` | `Statement.executeQuery(sql)` → `ResultSet` |
| Iterate rows | `while rs.next() do ...`, read with `rs.getX("col")` |
| Safely embed parameters | `PreparedStatement` with `?` placeholders — never string concat |
| Guarantee cleanup | `Using.resource(resource) { r => ... }` |
| Group statements atomically | `setAutoCommit(false)`, then `commit()`/`rollback()` |

### [04 · Functional Programming Patterns](level-3/04-functional-patterns.md)

| Pattern | Shape | Standard-library examples |
|---|---|---|
| Monoid | `combine(A, A): A` + `empty: A`, with `combine(empty, x) == x` | `Int` (+/0), `String` (++/`""`), `List` (++/`Nil`) |
| Functor | `map[A, B](level-3/F[A])(A => B): F[B]`, preserving shape | `List`, `Option`, `Either[E, _]`, `Future` |
| Generic code over either | `given`/`using` instances + a type parameter bound like `F[_]: Functor` | `combineAll`, `double` above |

### [05 · Testing Advanced](level-3/05-testing-advanced.md)

| Need to... | Use |
|---|---|
| Generate random test inputs | `Gen.alphaStr`, `Gen.choose(lo, hi)`, `Gen.listOf(gen)`, `Gen.oneOf(...)` |
| Run a block against many generated inputs | `forAll(gen) { x => assert(...) }` (mix in `ScalaCheckPropertyChecks`) |
| Create a fake dependency | `mock(classOf[Trait])` |
| Make it return canned values | `when(mock.method(args)).thenReturn(value)` |
| Assert how a mock was called | `verify(mock, times(n)).method(args)` |
| Decide mock vs. real | mock expensive/non-deterministic externals; use a real cheap object otherwise |

### [06 · Type Classes](level-3/06-type-classes.md)

| Need to... | Use |
|---|---|
| Define a capability as a type class | `trait TypeClass[A]` with the operations |
| Provide an implementation for a type | `given TypeClass[SomeType] with ...` |
| Summon an instance by name | `summon[TypeClass[A]]` or a custom `apply` in the companion |
| Write code generic over "has this capability" | `def f[A: TypeClass](level-3/...)` (context bound) |
| Give instances a natural method-call syntax | `extension [A](a: A)(using tc: TypeClass[A]) def op = ...` |
| Add capability to a type you don't own | define a `given` for it — no inheritance required |

### [07 · Higher-Kinded Types Intro](level-3/07-higher-kinded-types.md)

| Concept | Meaning |
|---|---|
| Kind `*` | An ordinary, complete type (`Int`, `String`, `List[Int]`) |
| Kind `* -> *` | A type constructor needing one type argument (`List`, `Option`, `Box`) |
| `F[_]` | Scala's syntax for a type parameter of kind `* -> *` |
| `[F[_]: TypeClass]` | Context bound on a higher-kinded parameter |
| When to use it | You have ≥2 concrete `F[_]`s needing the same generic logic |

### [08 · Performance &amp; Profiling](level-3/08-performance-profiling.md)

| Need to... | Use |
|---|---|
| Time code without lying to yourself | warm up first, average several measured runs |
| Compare two implementations | run both through the same `benchmark` helper |
| Build a collection in a loop | `ListBuffer` (or a single-pass `.map`), not repeated `List :+` |
| Get statistically trustworthy JVM benchmarks | JMH via `sbt-jmh`, forked JVMs + warm-up + measurement phases |
| Avoid dead-code-elimination skewing results | JMH's `Blackhole`, or ensure the result is actually used/printed |

### [09 · Akka Actors Basics](level-3/09-akka-actors-basics.md)

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

### [10 · Project — REST API + Database Service](level-3/10-project-rest-api-db.md)

| Piece | Role |
|---|---|
| `Db` object | Owns the JDBC `Connection`, exposes `insert`/`all` as plain Scala functions |
| `last_insert_rowid()` | SQLite's portable way to get an auto-increment id (its driver skips `getGeneratedKeys`) |
| `TasksHandler` | Routes by `(method, path)`, delegates to `Db`, always sends *some* response |
| `try`/`catch` around `handleInner` | Turns handler exceptions into a real `500` instead of a dropped connection |
| `Executors.newFixedThreadPool` | Lets blocking JDBC calls run concurrently across requests |

## Level 4 · Master

### [01 · Advanced Functional Programming](level-4/01-advanced-functional-programming.md)

| Concept | What it gives you |
|---|---|
| `cats.Monad[F[_]]` | Shared `pure`/`flatMap` interface across `Option`/`List`/`Either`/`Future`/etc. |
| `Validated[E, A]` | `Either`-like, but accumulates every error instead of stopping at the first |
| `.mapN` | Combine several independent effects (`Validated`, `Option`, ...) into one result |
| `IO[A]` | A description of an effect, run explicitly (usually via `IOApp`), unlike eager `Future` |
| Reach for `Either`/`EitherT` | When step *N* genuinely needs step *N-1*'s result |
| Reach for `Validated` | When checks are independent and you want every failure reported |

### [02 · Distributed Systems with Akka](level-4/02-distributed-systems-akka.md)

| Need to... | Use |
|---|---|
| Enable clustering for an `ActorSystem` | `akka.actor.provider = cluster` in config |
| Join a cluster | `Cluster(system).manager ! Join(address)` |
| Check this node's status | `cluster.selfMember.status` |
| See all known members | `cluster.state.members` |
| React to membership changes | subscribe to cluster events, don't poll |
| Distribute many stateful actors across nodes | Cluster Sharding (built on the mechanism above) |

### [03 · Production APIs](level-4/03-production-apis.md)

| Concern | Technique |
|---|---|
| Non-breaking API evolution | Version prefix (`/v1`, `/v2`) on every route |
| Machine-readable validation failures | A structured error type (`field` + `reason`), not a bare string |
| Reporting all problems at once | Accumulate errors in a `List`/`Validated`, don't short-circuit |
| "Is the process alive" | A dependency-free `/health` route |
| Guaranteed observability | Wrap logging/timing in `finally` around the whole handler, not per-branch |

### [04 · Security Best Practices](level-4/04-security-best-practices.md)

| Need to... | Use |
|---|---|
| Store a password safely | PBKDF2 (or bcrypt/argon2 in production) with a per-user random salt |
| Compare secrets/hashes | `MessageDigest.isEqual`, never `==` |
| Issue a stateless auth token | HMAC-signed payload (the mechanism behind JWT) |
| Verify a token wasn't tampered with | recompute the signature and compare (constant-time) |
| Gate an action by role/permission | check the verified payload's claims explicitly — authn ≠ authz |

### [05 · Testing at Scale &amp; CI](level-4/05-testing-at-scale-ci.md)

| Need to... | Use |
|---|---|
| Mark a test as slow/integration | `object SlowTest extends Tag("SlowTest")`, pass it to `test(name, SlowTest)` |
| Run everything except tagged tests | `sbt "testOnly * -- -l SlowTest"` |
| Run only a tagged group | `sbt "testOnly * -- -n SlowTest"` |
| Parallelize tests within one suite | mix in `ParallelTestExecution` (only if tests own their state) |
| Guarantee every push is tested | a CI workflow that runs `sbt test` on a clean checkout |
| Fail fast in CI | run a quick, untagged-only step before the full suite |

### [06 · Deployment (Docker)](level-4/06-deployment-docker.md)

| Need to... | Use |
|---|---|
| Bundle app + dependencies into one JAR | `sbt-assembly` plugin, `sbt assembly` |
| Configure per-environment without rebuilding | read from environment variables (`sys.env`), never hardcode |
| Keep the final image small | a JRE-only (not JDK) base image for the run stage |
| Build without requiring local `sbt` | a multi-stage Dockerfile with a build stage and a run stage |
| Avoid leaking host build artifacts into the image | a `.dockerignore` excluding `target/`, `.git/` |

### [07 · Performance at Scale](level-4/07-performance-at-scale.md)

| Need to... | Use |
|---|---|
| Avoid recomputing the same result | `ConcurrentHashMap.computeIfAbsent` (thread-safe check-then-compute) |
| Keep a cache from growing forever | an eviction policy (max size / TTL) — Caffeine in production |
| Size a thread pool for I/O-bound work | generous — threads spend most time waiting, not competing for CPU |
| Size a thread pool for CPU-bound work | around `availableProcessors()` — more threads won't help |
| Tune heap/GC | `-Xms`/`-Xmx`, `-XX:+UseG1GC`, verified with a profiler, not guessed |

### [08 · Scala 3 Features](level-4/08-scala-3-features.md)

| Feature | Use it for |
|---|---|
| `enum` with data-carrying cases | An ADT (sealed hierarchy) without per-case `case class` boilerplate |
| `opaque type A = B` | A distinct compile-time type over `B`, zero runtime cost |
| `A \| B` (union type) | "One of several valid, unrelated input shapes," no error bias |
| `Either[L, R]` | "One of two outcomes," with `L` conventionally meaning failure |
| `extension (x: T) def method = ...` | Add a method to a type you don't own, no type class needed |

### [09 · Build Tooling at Scale](level-4/09-build-tooling-at-scale.md)

| Need to... | Use |
|---|---|
| Share a setting across every module | `ThisBuild / settingName := value` |
| Let one module use another's code | `.dependsOn(otherProject)` |
| Run a task across every module from the root | `.aggregate(project1, project2, ...)` |
| Share settings (not just one value) across modules | a `lazy val common = Seq(...)` applied via `.settings(common)` |
| Decide whether to split into modules | reuse, team ownership, or measured compile-time pain — not by default |

---

Go deeper with the full [Scala Mastery Path](index.md) curriculum or test yourself with the [interview questions](interview-questions.md).
