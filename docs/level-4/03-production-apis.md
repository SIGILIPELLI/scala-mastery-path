# 03 · Production APIs

Level 3's REST API project made something that works. This module hardens
it toward something you'd actually run in production: a versioned URL
scheme, structured (not stringly-typed) validation errors, and basic
observability — a health check and per-request timing logs — building on
the same `HttpHandler` shape from Level 3.

## Versioning the API surface

Prefixing every route with a version (`/v1/...`) lets you introduce
breaking changes behind `/v2/...` later without touching existing clients:

```scala
server.createContext("/v1", new ApiV1Handler)
```

A handler that only ever answers under `/v1` makes the version an explicit
part of the contract, rather than something bolted on with a header nobody
remembers to check.

## Structured validation errors

Level 2's `Either`-based validation returned a single `String` error.
Production APIs should return **structured** errors — data a client can
parse and act on, not a message meant only for a human:

```scala
case class ValidationError(field: String, reason: String)

def validateCreateUser(name: String, age: String): Either[List[ValidationError], (String, Int)] =
  val nameErr = if name.isBlank then Some(ValidationError("name", "cannot be blank")) else None
  val ageParsed = age.toIntOption
  val ageErr = if ageParsed.isEmpty then Some(ValidationError("age", "must be a number")) else None
  val errs = List(nameErr, ageErr).flatten
  if errs.nonEmpty then Left(errs) else Right((name, ageParsed.get))
```

Note this collects **every** validation problem (both `name` and `age` if
both are bad) rather than stopping at the first — the same accumulating
shape as `Validated` from module 01, expressed here with a plain `List`
since only one field needs it. Reach for `Validated` itself once you have
more than a couple of independent checks to combine.

## Wiring validation into a handler

```scala
class ApiV1Handler extends HttpHandler:
  def handle(exchange: HttpExchange): Unit =
    val start = System.nanoTime()
    val path = exchange.getRequestURI.getPath
    val method = exchange.getRequestMethod
    try
      (method, path) match
        case ("GET", "/v1/health") =>
          respond(exchange, 200, """{"status":"ok"}""")
        case ("POST", "/v1/users") =>
          val body = new String(exchange.getRequestBody.readAllBytes())
          val parts = body.split(",", 2)
          val (name, age) = (parts.lift(0).getOrElse(""), parts.lift(1).getOrElse(""))
          validateCreateUser(name, age) match
            case Right((n, a)) => respond(exchange, 201, s"""{"name":"$n","age":$a}""")
            case Left(errs) =>
              val body2 = errs.map(e => s"""{"field":"${e.field}","reason":"${e.reason}"}""").mkString("[", ",", "]")
              respond(exchange, 400, s"""{"errors":$body2}""")
        case _ =>
          respond(exchange, 404, """{"error":"not found"}""")
    finally
      val ms = (System.nanoTime() - start) / 1e6
      println(s"[${java.time.Instant.now()}] $method $path -> ${ms}ms")
```

```text
GET /v1/health -> {"status":"ok"}
POST /v1/users (Ada,30) -> 201 {"name":"Ada","age":30}
POST /v1/users (,abc) -> 400 {"errors":[{"field":"name","reason":"cannot be blank"},{"field":"age","reason":"must be a number"}]}
```

Both problems with the second request — blank name *and* unparseable age —
show up together in one `400` response, letting a client fix both at once
instead of a slow back-and-forth of one-error-per-request.

## The health check

`GET /v1/health` is deliberately the simplest possible route: no database
call, no dependency, just "the process is alive and answering requests."
Load balancers and orchestrators (Kubernetes readiness/liveness probes,
covered more in module 06) poll this to decide whether to route traffic to
an instance — keeping it dependency-free means a slow database doesn't get
misreported as "the whole service is down."

### The trap: `finally` for observability, not just cleanup

The `try`/`finally` around the whole match isn't there to catch
exceptions here — it's there so the timing log runs **regardless of which
branch handled the request**, success or 404. Putting the log line inside
each `case` individually would work today but silently stops logging the
moment someone adds a new route and forgets to copy the log line into it.
Wrapping the cross-cutting concern (timing, and in a real service, request
logging/tracing) once around the whole handler is what keeps every route
observable by construction rather than by discipline.

## Cheat sheet

| Concern | Technique |
|---|---|
| Non-breaking API evolution | Version prefix (`/v1`, `/v2`) on every route |
| Machine-readable validation failures | A structured error type (`field` + `reason`), not a bare string |
| Reporting all problems at once | Accumulate errors in a `List`/`Validated`, don't short-circuit |
| "Is the process alive" | A dependency-free `/health` route |
| Guaranteed observability | Wrap logging/timing in `finally` around the whole handler, not per-branch |

## Exercise

Add a `field`-level `reason` for a `"name too long"` case (say, over 50
characters) to `validateCreateUser`, and confirm a too-long name combined
with a bad age still reports both errors. Then add a second health-style
route, `GET /v1/health/ready`, that *does* check a dependency (reuse
Level 3's in-memory SQLite `Db` and run a trivial `SELECT 1`), returning
`503` if the query throws — and explain in a comment why this route should
never be the one a load balancer uses to decide whether to route *new*
traffic to a *starting* instance versus one used to decide whether to keep
routing to an *already-serving* instance.
