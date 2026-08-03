# 08 · sbt Deep Dive

Every project so far has used a single-project `build.sbt` with one or two
settings. This module covers the sbt features you'll actually reach for on
a real project: adding library dependencies properly, splitting a codebase
into multiple modules, adding plugins, and writing your own custom tasks.

## Anatomy of `build.sbt`

```scala
// build.sbt
ThisBuild / scalaVersion := "3.3.3"
ThisBuild / version := "0.1.0-SNAPSHOT"

lazy val root = (project in file("."))
  .settings(
    name := "myapp",
    libraryDependencies += "com.lihaoyi" %% "upickle" % "3.3.1"
  )
```

- `ThisBuild / setting` applies a setting to every project in the build,
  not just one — `scalaVersion` and `version` almost always belong here.
- `project in file(".")` defines a project whose sources live in the given
  directory (`.` for the build root).
- `%%` picks the library build matching your project's Scala *binary*
  version automatically (so `"com.lihaoyi" %% "upickle" % "3.3.1"` resolves
  to `upickle_3` against a Scala 3 project) — use plain `%` only for
  Scala-version-independent (pure Java) artifacts.
- `% Test` (seen in [Module 5](05-scalatest-testing.md)) scopes a dependency
  to `src/test/scala` only.

## Multi-module builds

A real application often splits into a shared "core" module and one or more
consumers (a CLI, a web server, a test harness) that depend on it. Each
module gets its own `project in file(...)` and its own `src/main/scala`:

```text
myapp/
├── build.sbt
├── project/
│   └── build.properties
├── core/
│   └── src/main/scala/Greeter.scala
└── cli/
    └── src/main/scala/Main.scala
```

```scala
// build.sbt
ThisBuild / scalaVersion := "3.3.3"

lazy val core = (project in file("core"))
  .settings(name := "myapp-core")

lazy val cli = (project in file("cli"))
  .dependsOn(core)                      // cli can use anything public in core
  .settings(
    name := "myapp-cli",
    libraryDependencies += "com.lihaoyi" %% "upickle" % "3.3.1"
  )

lazy val root = (project in file("."))
  .aggregate(core, cli)                 // running a task on root runs it on both
  .settings(name := "myapp")
```

```scala
// core/src/main/scala/Greeter.scala
package myapp.core
object Greeter:
  def greet(name: String): String = s"Hello, $name, from core!"
```

```scala
// cli/src/main/scala/Main.scala
import myapp.core.Greeter
@main def cliMain(): Unit = println(Greeter.greet("world"))
```

```bash
sbt "cli/run"
# Hello, world, from core!
```

`dependsOn` and `aggregate` solve two different problems that look similar
at first: `dependsOn(core)` gives `cli`'s code access to `core`'s classes
(a *compile-time* dependency); `aggregate(core, cli)` just means "running
`compile`/`test`/etc. on `root` should also run it on these projects" (a
*command fan-out*, with no code sharing implied). Forgetting `dependsOn`
and only using `aggregate` is a common mistake — the modules will build
independently but `cli` won't actually be able to import anything from
`core`.

`sbt "cli/run"` (with the module name prefixed) runs a task scoped to one
specific module — plain `sbt run` in a multi-module build asks sbt to
figure out which project has a runnable `main`, which is ambiguous once you
have more than one.

## Plugins

Plugins add new tasks and settings to sbt itself, declared in
`project/plugins.sbt` (a separate, meta-level build file — notice it's
*inside* the `project/` directory, not the project root). A very common one
is **sbt-assembly**, which bundles a module and all its dependencies into a
single runnable "fat jar":

```scala
// project/plugins.sbt
addSbtPlugin("com.eed3si9n" % "sbt-assembly" % "2.2.0")
```

```bash
sbt "cli/assembly"
# ...
# [info] Built: cli/target/scala-3.3.3/myapp-cli-assembly-0.1.0-SNAPSHOT.jar

java -jar cli/target/scala-3.3.3/myapp-cli-assembly-0.1.0-SNAPSHOT.jar
# Hello, world, from core!
```

That jar bundles `core`'s classes, `cli`'s classes, and upickle's classes
all together — it runs with plain `java -jar`, with no sbt or Scala
installation required on the machine that runs it. This is exactly how
you'd package the [Module 10 project](10-project-weather-cli.md) for
distribution.

## Custom tasks

A `taskKey` declares a brand-new sbt command with its own logic, defined in
`build.sbt` right alongside the project settings:

```scala
lazy val hello = taskKey[Unit]("prints a friendly greeting")

hello := {
  println("Hello from a custom sbt task!")
}
```

```bash
sbt hello
# Hello from a custom sbt task!
```

Tasks can depend on other tasks and settings using `.value` inside the
task body — for example, a task that runs after compilation and reports how
many source files were compiled, or one that shells out to a code
formatter before `compile` runs. This is how most sbt plugins are built
internally: a plugin is really just a packaged set of custom `taskKey`s and
`settingKey`s, plus logic wiring them into existing tasks like `compile` or
`test`.

## The trap: `run`/`test` scope confusion in multi-module builds

Once a build has more than one project, unscoped commands (`compile`,
`run`, `test`) apply to whichever project is currently the sbt *shell's*
active project (the `root` project, by default, unless you `project cli`d
into another one first) — **not** necessarily the one you meant. Two
reliable habits avoid the confusion entirely:

- Prefix the module explicitly: `sbt "cli/run"`, `sbt "core/test"`.
- Or switch the active project inside an interactive sbt shell
  (`sbt` then `project cli` then plain `run`), and switch back
  (`project root`) when you're done.

## Cheat sheet

| Concept | Syntax | Purpose |
|---|---|---|
| Cross-versioned dependency | `"org" %% "lib" % "1.0"` | Resolves the right Scala-version build automatically |
| Test-only dependency | `"org" %% "lib" % "1.0" % Test` | Available only under `src/test/scala` |
| Multi-module project | `lazy val x = (project in file("x"))` | One module, own source tree and settings |
| Code dependency between modules | `.dependsOn(other)` | Lets this module import the other's classes |
| Command fan-out across modules | `.aggregate(a, b)` | Running a task on the parent runs it on `a` and `b` too |
| Add a plugin | `project/plugins.sbt` → `addSbtPlugin(...)` | New sbt-wide tasks/settings |
| Custom task | `lazy val t = taskKey[Unit]("...")` + `t := { ... }` | Your own `sbt <name>` command |
| Run a specific module | `sbt "modulename/run"` | Avoids ambiguity in multi-module builds |

## Exercise

Turn a single-module project into two: a `model` module containing a
`case class Product(name: String, price: Double)` and a `def
applyDiscount(p: Product, pct: Double): Product`, and an `app` module that
`dependsOn(model)` and has a `@main` printing a discounted product. Add
`sbt-assembly` as a plugin and produce a runnable fat jar for `app` with
`sbt "app/assembly"`. Finally, add one custom `taskKey[Unit]` called
`banner` to the root build that prints your project's name and version
(read `name.value` and `version.value` inside the task body), and confirm
`sbt banner` prints it.
