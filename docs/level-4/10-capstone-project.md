# 10 · Capstone Project

This capstone combines nearly everything from Levels 3 and 4 into one
service: a task-tracking API with real user accounts. It uses HTTP routing
(Level 3), a SQLite-backed database with prepared statements and safe
resource-handling helpers (Level 3), password hashing and HMAC-signed
tokens (Level 4 security), and structured error responses (Level 4
production APIs) — all in one runnable `sbt` project.

## Project layout

```
task-tracker/
├── build.sbt
└── src/main/scala/Main.scala
```

```scala title="build.sbt"
scalaVersion := "3.3.1"
libraryDependencies += "org.xerial" % "sqlite-jdbc" % "3.44.1.0"
```

## The data layer: users and tasks

```scala
object Db:
  Class.forName("org.sqlite.JDBC")
  val conn: Connection = DriverManager.getConnection("jdbc:sqlite::memory:")
  conn.createStatement().executeUpdate(
    "CREATE TABLE users(id INTEGER PRIMARY KEY AUTOINCREMENT, username TEXT UNIQUE NOT NULL, password_hash TEXT NOT NULL)"
  )
  conn.createStatement().executeUpdate(
    "CREATE TABLE tasks(id INTEGER PRIMARY KEY AUTOINCREMENT, user_id INTEGER NOT NULL, title TEXT NOT NULL, done INTEGER NOT NULL DEFAULT 0)"
  )

  def createUser(username: String, password: String): Either[String, Int] =
    try
      Using.resource(conn.prepareStatement("INSERT INTO users(username, password_hash) VALUES (?, ?)")) { ps =>
        ps.setString(1, username)
        ps.setString(2, Passwords.hash(password))
        ps.executeUpdate()
      }
      Using.resource(conn.createStatement()) { st =>
        Using.resource(st.executeQuery("SELECT last_insert_rowid() AS id")) { rs =>
          rs.next(); Right(rs.getInt("id"))
        }
      }
    catch case e: Exception => Left(s"could not create user: ${e.getMessage}")

  def authenticate(username: String, password: String): Option[Int] =
    Using.resource(conn.prepareStatement("SELECT id, password_hash FROM users WHERE username = ?")) { ps =>
      ps.setString(1, username)
      Using.resource(ps.executeQuery()) { rs =>
        if rs.next() && Passwords.verify(password, rs.getString("password_hash")) then Some(rs.getInt("id"))
        else None
      }
    }
```

`createUser` returns `Left` (not a thrown exception) if the `UNIQUE`
constraint on `username` rejects a duplicate signup — the database's own
integrity check becomes a normal `Either` value the handler can respond to
with a `400`, the same "make failure a value" discipline from Level 2.
`authenticate` never leaks *why* login failed (wrong username vs. wrong
password get the same `None`) — a deliberate security choice: telling an
attacker "that username doesn't exist" versus "wrong password" makes
username enumeration trivial.

`addTask`/`tasksFor` (omitted here, same shape as Level 3's project)
scope every task query by `user_id`, so one user's tasks are never visible
in another's `GET /tasks`.

## Auth: signup, login, and per-request tokens

Reusing Level 4's `Passwords` (PBKDF2 + salt) and `Tokens` (HMAC-signed
payload) objects unchanged:

```scala
case ("POST", "/signup") =>
  val body = new String(exchange.getRequestBody.readAllBytes())
  val Array(username, password) = body.split(",", 2)
  Db.createUser(username, password) match
    case Right(id) => respond(exchange, 201, s"""{"id":$id,"username":"$username"}""")
    case Left(err) => respond(exchange, 400, s"""{"error":"$err"}""")

case ("POST", "/login") =>
  val body = new String(exchange.getRequestBody.readAllBytes())
  val Array(username, password) = body.split(",", 2)
  Db.authenticate(username, password) match
    case Some(id) => respond(exchange, 200, s"""{"token":"${Tokens.sign(s"userId=$id")}"}""")
    case None     => respond(exchange, 401, """{"error":"invalid credentials"}""")
```

Every protected route reads the token from the `Authorization` header,
verifies its signature, and extracts the user id from the payload — this
is the authentication *and* authorization gate from Level 4's security
module, applied at the HTTP layer:

```scala
private def userIdFromAuth(exchange: HttpExchange): Option[Int] =
  Option(exchange.getRequestHeaders.getFirst("Authorization"))
    .flatMap(Tokens.verify)
    .flatMap(payload => payload.split("=").lift(1))
    .map(_.toInt)

case ("POST", "/tasks") =>
  userIdFromAuth(exchange) match
    case None => respond(exchange, 401, """{"error":"unauthorized"}""")
    case Some(uid) =>
      val title = new String(exchange.getRequestBody.readAllBytes())
      val id = Db.addTask(uid, title)
      respond(exchange, 201, s"""{"id":$id,"title":"$title","done":false}""")
```

That `flatMap` chain reads as one sentence: get the header, *if present*
verify it, *if valid* extract the id, *if parseable* use it — any missing
step falls through to `None` and a `401`, without a single nested `if`.

## Running it

```text
$ sbt run
{"id":1,"username":"ada"}
{"token":"userId=1.Z5ywgvV_5HtQnHG8CjtMbXyWPJ-v-OvZnX3RYo4XMLI"}
{"id":1,"title":"Write capstone","done":false}
{"id":2,"title":"Ship it","done":false}
[{"id":1,"title":"Write capstone","done":false},{"id":2,"title":"Ship it","done":false}]
401
401
```

The flow: sign up `ada`, log in to get a token, create two tasks
authenticated with that token, list both back, then confirm a wrong
password on `/login` and a bogus token on `GET /tasks` both correctly
return `401` rather than leaking data or crashing.

## Every level, one file

| From... | Used as |
|---|---|
| Level 1 pattern matching | Every `match` on `(method, path)` and `Option`/`Either` results |
| Level 2 `Either`/`Option` | `createUser`'s `Either[String, Int]`, `authenticate`'s `Option[Int]` |
| Level 3 HTTP routing | `HttpHandler`/`HttpServer` request dispatch |
| Level 3 databases | `PreparedStatement`, `Using.resource`, `last_insert_rowid()` |
| Level 4 production APIs | `try`/`catch` turning failures into real HTTP responses, not dropped connections |
| Level 4 security | PBKDF2 password hashing, HMAC-signed tokens, constant-time comparison |
| Level 4 Scala 3 features | `enum`/case matching throughout, could be extended with `opaque type UserId` |

## Stretch goals

- Replace the comma-delimited request bodies (`"ada,hunter2"`) with real
  JSON parsing and a hand-written minimal decoder (or a small JSON library),
  and add the structured, multi-field validation errors from Level 4's
  production APIs module for signup (blank username, too-short password).
- Add token expiry (Level 4 security's exercise) so a token issued at login
  stops working after, say, 30 minutes, and confirm an expired token gets
  a `401` distinct from an invalid-signature `401`.
- Wrap the whole service in the Level 4 Docker module's multi-stage
  Dockerfile, and add the `/health`/`/health/ready` routes from the
  production APIs module (the latter running a trivial query against `Db`).
- Split the project into `core` (models + `Db`), `auth` (`Passwords` +
  `Tokens`), and `api` (`HttpHandler`s) modules using Level 4's build
  tooling module, with `api` depending on both `core` and `auth`.
- If you completed the Akka modules, sketch (even just as a comment block)
  how you'd replace the single mutable `Db.conn` with a `TaskStore` actor
  per user — what changes about concurrency safety once each user's tasks
  are owned by one actor instead of shared through one JDBC connection.
