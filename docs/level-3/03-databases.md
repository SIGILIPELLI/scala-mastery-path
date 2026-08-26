# 03 · Databases

Scala talks to relational databases the same way Java does: JDBC. Higher-level
libraries like Slick or Doobie wrap JDBC in functional, type-safe query
APIs, but they all bottom out in the same `Connection`/`Statement`/`ResultSet`
primitives this module covers directly — using an in-memory SQLite database
via the `org.xerial:sqlite-jdbc` driver, so every example here runs with
`sbt run` and no external database server.

## A minimal sbt project

```scala title="build.sbt"
scalaVersion := "3.3.1"
libraryDependencies += "org.xerial" % "sqlite-jdbc" % "3.44.1.0"
```

## Connecting and running statements

```scala
import java.sql.{DriverManager, Connection}

Class.forName("org.sqlite.JDBC")
val conn: Connection = DriverManager.getConnection("jdbc:sqlite::memory:")
val stmt = conn.createStatement()
stmt.executeUpdate("CREATE TABLE users(id INTEGER PRIMARY KEY, name TEXT)")
stmt.executeUpdate("INSERT INTO users(name) VALUES ('Ada')")
stmt.executeUpdate("INSERT INTO users(name) VALUES ('Grace')")
```

`executeUpdate` is for statements that change data (`CREATE`, `INSERT`,
`UPDATE`, `DELETE`) and returns a row count, not a result set.

## Reading rows: `ResultSet`

```scala
val rs = stmt.executeQuery("SELECT id, name FROM users ORDER BY id")
while rs.next() do
  println(s"${rs.getInt("id")}: ${rs.getString("name")}")
conn.close()
// 1: Ada
// 2: Grace
```

`ResultSet` is a mutable cursor: `.next()` advances it and returns `false`
once there are no more rows, and `.getInt`/`.getString`/etc. read columns
from the *current* row by name or 1-based index. It's a very Java-shaped
API — nothing here is functional yet.

### The trap: SQL injection via string concatenation

Building queries by concatenating user input into SQL text is the classic
way to accidentally let a caller run arbitrary SQL:

```scala
val userInput = "Ada'; DROP TABLE users; --"
// NEVER do this:
// stmt.executeQuery(s"SELECT * FROM users WHERE name = '$userInput'")
```

`PreparedStatement` fixes this by treating parameters as *data*, never as
SQL syntax, regardless of what they contain:

```scala
val safe = conn.prepareStatement("SELECT id, name FROM users WHERE name = ?")
safe.setString(1, userInput)
val rs2 = safe.executeQuery()
println(rs2.next())   // false -- no row matches that literal (harmless) string
```

Always use `PreparedStatement` with `?` placeholders for any query built
from external input — it's not just safer, the database can also cache and
reuse the query plan across calls with different parameters.

## Wrapping JDBC in something safer

Raw JDBC leaks resources easily if an exception is thrown mid-query. A
small `using`-style helper (Scala 3's `Using` from `scala.util`) closes
resources automatically, similar to try-with-resources in Java:

```scala
import scala.util.Using

def findUser(conn: Connection, id: Int): Option[String] =
  Using.resource(conn.prepareStatement("SELECT name FROM users WHERE id = ?")) { ps =>
    ps.setInt(1, id)
    Using.resource(ps.executeQuery()) { rs =>
      if rs.next() then Some(rs.getString("name")) else None
    }
  }

println(findUser(conn, 1))   // Some(Ada)
println(findUser(conn, 99))  // None
```

`Using.resource` guarantees `.close()` runs even if the block throws —
returning a proper `Option` instead of a raw, still-open `ResultSet` also
makes the function's contract ("might not find one") visible in its type.

### The trap: connections aren't free

Every `DriverManager.getConnection` call opens a real socket/file handle to
the database. Opening one per query and never pooling them is a common
source of "too many connections" errors under load. Production code uses a
connection pool (HikariCP is the standard choice) and borrows/returns
connections from it rather than opening new ones per request.

## Transactions

Multiple statements that must all succeed or all fail together need a
transaction — turn off auto-commit, run the statements, then commit (or
roll back on failure):

```scala
conn.setAutoCommit(false)
try
  stmt.executeUpdate("INSERT INTO users(name) VALUES ('Alan')")
  stmt.executeUpdate("INSERT INTO users(name) VALUES ('Barbara')")
  conn.commit()
  println("transaction committed")
catch
  case e: Exception =>
    conn.rollback()
    println(s"rolled back: ${e.getMessage}")
finally
  conn.setAutoCommit(true)
// transaction committed
```

## Cheat sheet

| Need to... | Use |
|---|---|
| Load a JDBC driver and connect | `Class.forName(driver)`, `DriverManager.getConnection(url)` |
| Run `CREATE`/`INSERT`/`UPDATE`/`DELETE` | `Statement.executeUpdate(sql)` |
| Run `SELECT` | `Statement.executeQuery(sql)` → `ResultSet` |
| Iterate rows | `while rs.next() do ...`, read with `rs.getX("col")` |
| Safely embed parameters | `PreparedStatement` with `?` placeholders — never string concat |
| Guarantee cleanup | `Using.resource(resource) { r => ... }` |
| Group statements atomically | `setAutoCommit(false)`, then `commit()`/`rollback()` |

## Exercise

Extend the schema with an `orders` table (`id INTEGER PRIMARY KEY, user_id
INTEGER, amount REAL`) referencing `users`. Write `def totalSpent(conn:
Connection, userId: Int): Double` using a `PreparedStatement` with a `SUM(amount)`
query, returning `0.0` if the user has no orders (handle the `NULL` sum
result from `SUM` over zero rows). Insert a couple of orders for one user
and none for another, then print `totalSpent` for both to confirm the
zero-orders case works. Wrap both inserts for one user in a transaction and
verify a rollback (triggered by a deliberate exception between the two
inserts) leaves neither order in the table.
