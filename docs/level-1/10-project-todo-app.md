# 10 · Project — CLI To-Do App

A small end-to-end project combining everything from Level 1: `val`/`var`,
control flow, functions, collections, case classes, pattern matching, and
traits.

## What you'll build

A command-line to-do list that:

- Adds tasks
- Lists tasks (with done/pending status)
- Marks tasks done
- Deletes tasks
- Persists everything to a plain text file between runs

We use a simple pipe-delimited text file for storage rather than JSON, since
a proper JSON library is introduced in
[Level 2 · Working with JSON](../level-2/07-working-with-json.md). Parsing a
one-line-per-task format by hand is a nice, self-contained exercise in
string processing.

## Project layout

```text
todo-app/
├── build.sbt
└── src/
    └── main/
        └── scala/
            ├── Task.scala
            ├── TaskStorage.scala
            └── Main.scala
```

```scala
// build.sbt
ThisBuild / scalaVersion := "3.3.3"

lazy val root = (project in file("."))
  .settings(
    name := "todo-app"
  )
```

## Task.scala — the data model

```scala
// src/main/scala/Task.scala
case class Task(description: String, done: Boolean = false):
  def toLine: String =
    s"${if done then "1" else "0"}|$description"

object Task:
  def fromLine(line: String): Option[Task] =
    line.split("\\|", 2) match
      case Array(doneFlag, description) =>
        Some(Task(description, doneFlag == "1"))
      case _ =>
        None
```

`Task.fromLine` returns `Option[Task]` — `None` for a malformed line instead
of crashing. `Option` gets a full treatment in
[Level 2](../level-2/04-option-either.md); for now, read `Some(x)` as "a
value is present" and `None` as "nothing here," and note that `fromLine`
lets the caller decide what to do about missing/bad data rather than
throwing.

## TaskStorage.scala — persistence layer

```scala
// src/main/scala/TaskStorage.scala
import scala.io.Source
import java.io.PrintWriter
import java.io.File

class TaskStorage(path: String = "tasks.txt"):
  def load(): List[Task] =
    val file = File(path)
    if !file.exists() then List.empty
    else
      val source = Source.fromFile(file)
      try
        source.getLines()
          .map(Task.fromLine)
          .collect { case Some(task) => task }   // drop any malformed lines
          .toList
      finally
        source.close()

  def save(tasks: List[Task]): Unit =
    val writer = PrintWriter(File(path))
    try
      tasks.foreach(task => writer.println(task.toLine))
    finally
      writer.close()
```

`collect { case Some(task) => task }` combines filtering and unwrapping in
one pass — it keeps only the `Some` results and extracts their contents,
skipping any `None`s from malformed lines entirely.

## Main.scala — CLI logic

```scala
// src/main/scala/Main.scala
def addTask(tasks: List[Task], description: String): List[Task] =
  tasks :+ Task(description)

def listTasks(tasks: List[Task]): Unit =
  if tasks.isEmpty then println("No tasks yet.")
  else
    tasks.zipWithIndex.foreach { case (task, i) =>
      val status = if task.done then "x" else " "
      println(s"[$status] ${i + 1}. ${task.description}")
    }

def completeTask(tasks: List[Task], index: Int): List[Task] =
  if index < 1 || index > tasks.length then
    println(s"No task with number $index")
    tasks
  else
    tasks.zipWithIndex.map { case (task, i) =>
      if i == index - 1 then task.copy(done = true) else task
    }

def deleteTask(tasks: List[Task], index: Int): List[Task] =
  if index < 1 || index > tasks.length then
    println(s"No task with number $index")
    tasks
  else
    println(s"Deleted: ${tasks(index - 1).description}")
    tasks.zipWithIndex.filter { case (_, i) => i != index - 1 }.map(_._1)

@main def todo(args: String*): Unit =
  val storage = TaskStorage()
  val tasks = storage.load()

  args.toList match
    case Nil =>
      println("Usage: todo [add <text> | list | done <n> | delete <n>]")

    case "add" :: rest if rest.nonEmpty =>
      val updated = addTask(tasks, rest.mkString(" "))
      storage.save(updated)
      println(s"Added: ${rest.mkString(" ")}")

    case "add" :: Nil =>
      println("Usage: todo add <text>")

    case "list" :: _ =>
      listTasks(tasks)

    case "done" :: nStr :: _ =>
      val updated = completeTask(tasks, nStr.toIntOption.getOrElse(0))
      storage.save(updated)

    case "delete" :: nStr :: _ =>
      val updated = deleteTask(tasks, nStr.toIntOption.getOrElse(0))
      storage.save(updated)

    case command :: _ =>
      println(s"Unknown command: $command")
```

The `match` on `args.toList` is doing a lot of work here: `"add" :: rest`
destructures the list into its first element and everything after, `nStr.toIntOption`
safely parses a string to an `Int` (returning `None` instead of throwing on
bad input), and the final `case command :: _` catches anything unrecognized.

## Running it

```bash
sbt run add "Write Level 1 exercises"
sbt run add "Review Level 2 outline"
sbt run list
# [ ] 1. Write Level 1 exercises
# [ ] 2. Review Level 2 outline

sbt run done 1
sbt run list
# [x] 1. Write Level 1 exercises
# [ ] 2. Review Level 2 outline
```

(Passing arguments through `sbt run` requires quoting the whole command as
one line, e.g. `sbt "run add \"Write Level 1 exercises\""`, or building a
standalone jar with `sbt package` and running it with `scala` directly — sbt
argument-passing quirks are covered in
[Level 2 · sbt Deep Dive](../level-2/08-sbt-deep-dive.md).)

## Stretch goals

- Add a `priority` field (`Low`/`Medium`/`High`) modeled as a `sealed trait`
  (see [Module 9](09-traits-basics.md)) and sort the list by it.
- Add a `dueDate` field using `java.time.LocalDate` and highlight overdue
  tasks when listing.
- Write a basic test for `TaskStorage` by hand (round-trip a list of tasks
  through `save`/`load` and assert equality) — you'll formalize this
  properly with ScalaTest in
  [Level 2 · Testing with ScalaTest](../level-2/05-scalatest-testing.md).

Completing this project means you're ready for **Level 2 · Intermediate**.
