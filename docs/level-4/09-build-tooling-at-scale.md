# 09 · Build Tooling at Scale

Every project so far has been one `build.sbt` and one source tree. A large
codebase usually splits into multiple **modules** — separately compiled
sub-projects sharing one build — so teams can work on different parts
independently, dependencies stay explicit, and unrelated changes don't
force a full rebuild.

## A multi-module build

```
multimod/
├── build.sbt
├── core/
│   └── src/main/scala/Model.scala
└── api/
    └── src/main/scala/Api.scala
```

```scala title="build.sbt"
ThisBuild / scalaVersion := "3.3.1"
ThisBuild / organization := "com.example"

lazy val common = Seq(
  scalacOptions ++= Seq("-deprecation", "-feature")
)

lazy val core = (project in file("core"))
  .settings(common)
  .settings(name := "core")

lazy val api = (project in file("api"))
  .dependsOn(core)
  .settings(common)
  .settings(name := "api")

lazy val root = (project in file("."))
  .aggregate(core, api)
  .settings(name := "multimod-root")
```

```scala title="core/src/main/scala/Model.scala"
package com.example.core
case class User(id: Int, name: String)
```

```scala title="api/src/main/scala/Api.scala"
package com.example.api
import com.example.core.User

@main def run(): Unit =
  val u = User(1, "Ada")
  println(s"loaded user: $u")
```

```text title="sbt compile \"api/run\""
[info] compiling 1 Scala source to .../core/target/scala-3.3.1/classes ...
[info] compiling 1 Scala source to .../api/target/scala-3.3.1/classes ...
[info] running com.example.api.run
loaded user: User(1,Ada)
```

`api` compiled successfully against `core`'s `User` — and importantly,
compiling `api` didn't require touching or recompiling `core`'s source
files themselves, only reading `core`'s already-built classes. This is the
whole payoff at scale: `core` recompiles when *its* source changes,
`api` recompiles when *either* its own source or `core`'s output changes,
and unrelated modules stay untouched.

## The four pieces that matter

- **`ThisBuild / setting`** applies a setting to every project in the
  build (`scalaVersion`, `organization`) — set it once instead of
  repeating per module.
- **`.dependsOn(core)`** on `api` makes `core`'s public classes visible to
  `api`'s code and tells sbt to build `core` first whenever `api` needs
  building.
- **`.aggregate(core, api)`** on `root` means running a task (like `compile`
  or `test`) at the root runs it across *all* aggregated projects — `sbt
  test` at the root runs every module's tests in one command.
- **`common` settings** shared via `.settings(common)` avoid repeating the
  same `scalacOptions` (or dependency versions, testing setup, etc.) in
  every module's block.

### The trap: `dependsOn` vs. `aggregate` are not the same thing

`dependsOn` is a **compile-time** dependency: `api`'s code can actually
import and use `core`'s classes. `aggregate` is a **task-fanout**
relationship: running a command at the root also runs it in the aggregated
projects, but aggregated projects don't automatically see each other's
code. Forgetting `dependsOn` on `api` (even with both projects aggregated
at the root) produces `Not found: com.example.core` the moment `api`'s
code tries to `import` from `core` — aggregation alone doesn't wire up
compile-time visibility.

## Why split into modules at all

A single-module build recompiles the whole project whenever *anything*
changes. A `core`/`api` split (and larger real builds might have
`core`/`persistence`/`http`/`cli` etc.) means:

- A change to `api`-only code never triggers recompiling `core`.
- `core` can be published as a library and reused by a separate `api`
  project entirely, or by multiple internal services.
- Team ownership boundaries can map to module boundaries — one team owns
  `core`, another owns `api`, each with a clear, explicit dependency
  contract instead of an implicit "everything can see everything" flat
  source tree.

### The trap: over-modularizing small projects

Splitting a project with a few hundred lines total into five modules adds
sbt project-definition overhead (more `build.sbt` boilerplate, slower
initial project loading, more places to keep `scalacOptions` in sync) for
no real compilation-time or ownership benefit. Reach for multiple modules
once you actually observe a reason — a genuinely reusable core library, a
team boundary, or compile times that start to hurt in a single flat
project — not by default.

## Cheat sheet

| Need to... | Use |
|---|---|
| Share a setting across every module | `ThisBuild / settingName := value` |
| Let one module use another's code | `.dependsOn(otherProject)` |
| Run a task across every module from the root | `.aggregate(project1, project2, ...)` |
| Share settings (not just one value) across modules | a `lazy val common = Seq(...)` applied via `.settings(common)` |
| Decide whether to split into modules | reuse, team ownership, or measured compile-time pain — not by default |

## Exercise

Add a third module, `cli`, depending only on `api` (not `core` directly),
with a `@main` entry point that calls into `api`'s logic *and* imports
`core`'s `User` type directly. Confirm this compiles — `dependsOn`'s
classpath visibility is transitive, so `cli` sees `core`'s classes through
`api` without listing `core` itself. Add `cli` to the root's
`.aggregate(...)` list and confirm `sbt compile` at the root builds all
three in dependency order (`core` before `api` before `cli`). Then think
through why you might still want `cli` to declare `.dependsOn(core)`
*explicitly* even though it isn't required for compilation to succeed today
— what breaks later if `api` ever stops depending on `core`, and `cli` was
relying on that indirect path all along?
