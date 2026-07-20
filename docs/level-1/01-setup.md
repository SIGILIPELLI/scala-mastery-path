# 01 · Setup & First Program

## Install the JDK and sbt

Scala runs on the Java Virtual Machine (JVM), so you need a JDK installed
first, then Scala's own tooling on top. Install a recent JDK (17 or 21 LTS)
and [sbt](https://www.scala-sbt.org/) (the standard Scala build tool):

```bash
# macOS (Homebrew)
brew install openjdk@21
brew install sbt

# Ubuntu/Debian
sudo apt install openjdk-21-jdk
echo "deb https://repo.scala-sbt.org/scalasbt/debian all main" | sudo tee /etc/apt/sources.list.d/sbt.list
sudo apt update && sudo apt install sbt

# Windows: use the installer from https://adoptium.net for the JDK,
# then https://www.scala-sbt.org/download.html for sbt
```

Verify the install:

```bash
java --version
# openjdk 21.0.x

sbt --version
# sbt script version: 1.10.x
```

Most Scala installations (via `sbt`, [Coursier](https://get-coursier.io/), or
an IDE) also give you the `scala` command for running the REPL and scripts
directly.

## The Scala REPL

The REPL (Read-Eval-Print Loop) is the fastest way to try things out. Launch
it by typing `scala` in your terminal:

```bash
scala
# Welcome to Scala 3.x
# scala>
```

```scala
scala> val greeting = "Hello, Scala!"
// val greeting: String = Hello, Scala!

scala> println(greeting)
// Hello, Scala!

scala> 2 + 2
// val res0: Int = 4

scala> :quit
```

`val res0` is the REPL auto-naming an unnamed expression's result so you can
refer to it later (`res0`, `res1`, ...). This is a REPL-only convenience, not
something you write in real source files.

## Your first program with sbt

Real projects use sbt to manage dependencies, compilation, and running.
Create a new project directory with this layout:

```
hello-scala/
├── build.sbt
└── src/
    └── main/
        └── scala/
            └── Hello.scala
```

```scala
// build.sbt
ThisBuild / scalaVersion := "3.3.3"

lazy val root = (project in file("."))
  .settings(
    name := "hello-scala"
  )
```

```scala
// src/main/scala/Hello.scala
@main def hello(): Unit =
  println("Hello, world!")
```

Run it with sbt:

```bash
cd hello-scala
sbt run
# ...
# Hello, world!
```

`@main` marks a top-level method as a program entry point (Scala 3) — no
wrapping class or `object` boilerplate required, unlike Java's
`public static void main`.

## Single-file scripts (quick experiments)

For quick one-off scripts without a full sbt project, use the `scala-cli`
tool (bundled with recent Scala installs) or plain `scala` on a `.scala`
file:

```bash
scala-cli Hello.scala
# Hello, world!
```

This compiles and runs in one step — handy for small scripts, but real
projects still use sbt (see [Module 9 of Level 2](../level-2/08-sbt-deep-dive.md)
for sbt in depth).

## Anatomy of the program

| Piece | Meaning |
|-------|---------|
| `@main def hello(): Unit = ...` | Declares `hello` as a runnable entry point. `Unit` means "returns nothing useful," Scala's equivalent of `void`. |
| `println(...)` | Prints text followed by a newline to standard output. |
| `build.sbt` | Declares the project's Scala version, name, and dependencies. |
| `src/main/scala/` | Where sbt looks for your `.scala` source files by convention. |

## Choosing an IDE

**IntelliJ IDEA** with the Scala plugin is the most common choice — solid
refactoring, debugging, and sbt integration. **VS Code** with the "Metals"
extension (a Scala language server) is a strong free/lightweight alternative.
Either works well for everything in this course.

## Exercise

Create a new sbt project named `greeter`. Add a `@main` entry point that
prints a greeting for three different names, one per line. Run it with
`sbt run`, then also try running the same logic directly in the `scala` REPL
by pasting the `println` lines in one at a time.
