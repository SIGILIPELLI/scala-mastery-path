# 10 · Project — REST API + Database Service

This project combines everything from Level 3: an HTTP server (module 02),
a real database via JDBC (module 03), and careful resource handling —
into one small, complete task-tracking service. It's a single `sbt`
project you can run end to end.

## Project layout

```
tasks-service/
├── build.sbt
└── src/main/scala/Main.scala
```

```scala title="build.sbt"
scalaVersion := "3.3.1"
libraryDependencies += "org.xerial" % "sqlite-jdbc" % "3.44.1.0"
```

## The database layer

An in-memory SQLite database (module 03's approach) backs the service — no
external database server needed to run this project:

```scala
import java.sql.{DriverManager, Connection}
import scala.util.Using

object Db:
  Class.forName("org.sqlite.JDBC")
  val conn: Connection = DriverManager.getConnection("jdbc:sqlite::memory:")
  conn.createStatement().executeUpdate(
    "CREATE TABLE tasks(id INTEGER PRIMARY KEY AUTOINCREMENT, title TEXT NOT NULL, done INTEGER NOT NULL DEFAULT 0)"
  )

  def insert(title: String): Int =
    Using.resource(conn.prepareStatement("INSERT INTO tasks(title) VALUES (?)")) { ps =>
      ps.setString(1, title)
      ps.executeUpdate()
    }
    // SQLite's JDBC driver doesn't implement getGeneratedKeys — last_insert_rowid() is the portable trap to know about
    Using.resource(conn.createStatement()) { st =>
      Using.resource(st.executeQuery("SELECT last_insert_rowid() AS id")) { rs =>
        rs.next()
        rs.getInt("id")
      }
    }

  def all(): List[(Int, String, Boolean)] =
    Using.resource(conn.createStatement()) { st =>
      Using.resource(st.executeQuery("SELECT id, title, done FROM tasks ORDER BY id")) { rs =>
        val buf = scala.collection.mutable.ListBuffer.empty[(Int, String, Boolean)]
        while rs.next() do buf += ((rs.getInt("id"), rs.getString("title"), rs.getInt("done") != 0))
        buf.toList
      }
    }
```

### The trap that actually showed up building this: `getGeneratedKeys`

The first version of `insert` used `Statement.RETURN_GENERATED_KEYS` and
`ps.getGeneratedKeys` — the standard JDBC way to get an auto-increment id
back — and it compiled fine but *threw at runtime*:

```
java.sql.SQLFeatureNotSupportedException: not implemented by SQLite JDBC driver
```

Not every JDBC driver implements every optional part of the spec.
SQLite's driver doesn't support `getGeneratedKeys`, so the portable fix is
a follow-up query for `SELECT last_insert_rowid()` — the version above.
This is exactly the kind of gap that only shows up when you actually run
the code against the real driver, not when you read the JDBC docs.

## The HTTP layer

Same `HttpHandler` routing shape as module 02, wired to `Db`:

```scala
import com.sun.net.httpserver.{HttpServer, HttpExchange, HttpHandler}
import java.net.InetSocketAddress

class TasksHandler extends HttpHandler:
  def handle(exchange: HttpExchange): Unit =
    try handleInner(exchange)
    catch case e: Throwable =>
      e.printStackTrace()
      respond(exchange, 500, s"""{"error":"${e.getMessage}"}""")

  private def handleInner(exchange: HttpExchange): Unit =
    (exchange.getRequestMethod, exchange.getRequestURI.getPath) match
      case ("GET", "/tasks") =>
        val body = Db.all().map((id, title, done) =>
          s"""{"id":$id,"title":"$title","done":$done}"""
        ).mkString("[", ",", "]")
        respond(exchange, 200, body)
      case ("POST", "/tasks") =>
        val title = new String(exchange.getRequestBody.readAllBytes())
        val id = Db.insert(title)
        respond(exchange, 201, s"""{"id":$id,"title":"$title","done":false}""")
      case _ =>
        respond(exchange, 404, """{"error":"not found"}""")

  private def respond(exchange: HttpExchange, status: Int, body: String): Unit =
    exchange.getResponseHeaders.add("Content-Type", "application/json")
    exchange.sendResponseHeaders(status, body.length)
    val os = exchange.getResponseBody
    os.write(body.getBytes)
    os.close()
```

Wrapping the real routing logic (`handleInner`) in a `try`/`catch` inside
`handle` matters here for a subtle reason: `com.sun.net.httpserver`'s
default behavior when a handler throws is to **abruptly close the
connection** rather than send any response — a client sees a bare
`IOException: HTTP/1.1 header parser received no bytes` with no indication
what went wrong server-side. Catching and turning failures into a real
`500` response (with the exception logged server-side) is the difference
between a debuggable failure and a mysterious connection drop — this is
also exactly the JDBC feature gap above, before the fix, manifesting as a
broken HTTP response instead of a clear stack trace.

## Wiring it together

```scala
@main def run(): Unit =
  val server = HttpServer.create(new InetSocketAddress(8282), 0)
  server.createContext("/tasks", new TasksHandler)
  server.setExecutor(java.util.concurrent.Executors.newFixedThreadPool(4))
  server.start()
```

A real, non-null `Executor` matters once a handler can block on I/O like
this one does (JDBC calls are blocking) — the JDK's null-executor default
runs requests sequentially, which is fine for a demo `main` but would
serialize every request in a real service.

## Running it

```text
$ sbt run
{"id":1,"title":"Write report","done":false}
{"id":2,"title":"Review PR","done":false}
status: 200
body: [{"id":1,"title":"Write report","done":false},{"id":2,"title":"Review PR","done":false}]
```

Two `POST /tasks` calls each return the newly created row with its assigned
id, and the subsequent `GET /tasks` lists both — confirming the id
assignment, insert, and read-back all round-trip correctly through the
fixed `Db.insert`.

## Cheat sheet

| Piece | Role |
|---|---|
| `Db` object | Owns the JDBC `Connection`, exposes `insert`/`all` as plain Scala functions |
| `last_insert_rowid()` | SQLite's portable way to get an auto-increment id (its driver skips `getGeneratedKeys`) |
| `TasksHandler` | Routes by `(method, path)`, delegates to `Db`, always sends *some* response |
| `try`/`catch` around `handleInner` | Turns handler exceptions into a real `500` instead of a dropped connection |
| `Executors.newFixedThreadPool` | Lets blocking JDBC calls run concurrently across requests |

## Stretch goals

- Add `PUT /tasks/{id}` to mark a task done, and `DELETE /tasks/{id}` to
  remove one — both need path-segment parsing (`path.stripPrefix("/tasks/")`)
  and a `PreparedStatement` `UPDATE`/`DELETE` guarded by `?` parameters.
- Replace the hand-rolled JSON string building with a minimal JSON encoder
  function that escapes quotes/backslashes in `title` — the current version
  would produce broken JSON for a task titled `Say "hi"`.
- Swap the in-memory SQLite database for a file-backed one
  (`jdbc:sqlite:tasks.db`) and confirm tasks survive a restart of the
  program.
- Add a `HikariCP` connection pool in front of `DriverManager.getConnection`
  and compare behavior under several concurrent clients hitting the server
  at once with a simple loop of `HttpClient` calls from multiple threads.
