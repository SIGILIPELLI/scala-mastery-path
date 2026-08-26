# 02 · Building APIs

Real Scala services usually expose HTTP endpoints, typically with a library
like **http4s** or **akka-http** built on top of `Future`/effect types. This
module builds the same request/response mental model with the JDK's own
`com.sun.net.httpserver` — no extra dependencies, fully runnable with plain
`scala Main.scala` — so you can see exactly what a routing library automates
before you reach for one.

## A minimal HTTP server

```scala
import com.sun.net.httpserver.{HttpServer, HttpExchange, HttpHandler}
import java.net.InetSocketAddress

class HelloHandler extends HttpHandler:
  def handle(exchange: HttpExchange): Unit =
    val query = Option(exchange.getRequestURI.getQuery).getOrElse("")
    val name = query.split("=").lift(1).getOrElse("World")
    val body = s"""{"message":"Hello, $name!"}"""
    exchange.getResponseHeaders.add("Content-Type", "application/json")
    exchange.sendResponseHeaders(200, body.length)
    val os = exchange.getResponseBody
    os.write(body.getBytes)
    os.close()

val server = HttpServer.create(new InetSocketAddress(8181), 0)
server.createContext("/hello", new HelloHandler)
server.setExecutor(null)
server.start()
```

Every handler follows the same shape: read something off the request
(`exchange.getRequestURI`, headers, body), decide a status code, write a
response body, and — critically — **close the response stream**. Forgetting
`os.close()` hangs the client, because it's still waiting for the server to
signal "no more bytes coming."

## Calling it back

```scala
import java.net.http.{HttpClient, HttpRequest, HttpResponse}
import java.net.URI

val client = HttpClient.newHttpClient()
val req = HttpRequest.newBuilder(URI.create("http://localhost:8181/hello?name=Ada")).build()
val resp = client.send(req, HttpResponse.BodyHandlers.ofString())
println(s"status: ${resp.statusCode()}")   // status: 200
println(s"body: ${resp.body()}")           // body: {"message":"Hello, Ada!"}
```

`HttpClient.send` is synchronous and blocks the calling thread until the
response arrives — fine here since it's just a demo hitting `localhost`.
`client.sendAsync(...)` returns a `Future`-like `CompletableFuture` for
non-blocking use in real code.

### The trap: one handler per path, matched by prefix

`createContext("/hello", handler)` matches `/hello` *and every path under
it* (`/hello/world`, `/hello/anything`) unless the handler itself checks
`exchange.getRequestURI.getPath` and rejects what it doesn't expect. Real
routing libraries like http4s give you declarative, exhaustive route
matching (`GET -> Root / "hello" / name`); rolling your own dispatch on
`HttpServer` means remembering to validate the path and method yourself:

```scala
class StrictHandler extends HttpHandler:
  def handle(exchange: HttpExchange): Unit =
    if exchange.getRequestMethod != "GET" then
      exchange.sendResponseHeaders(405, -1)
    else
      // ... handle GET
      exchange.sendResponseHeaders(200, 0)
```

## Routing by method and path

A tiny router is just a pattern match over method and path segments:

```scala
class Router extends HttpHandler:
  def handle(exchange: HttpExchange): Unit =
    val path = exchange.getRequestURI.getPath
    val method = exchange.getRequestMethod
    (method, path) match
      case ("GET", "/users") =>
        respond(exchange, 200, """[{"id":1,"name":"Ada"}]""")
      case ("GET", p) if p.startsWith("/users/") =>
        val id = p.stripPrefix("/users/")
        respond(exchange, 200, s"""{"id":$id,"name":"User $id"}""")
      case _ =>
        respond(exchange, 404, """{"error":"not found"}""")

  private def respond(exchange: HttpExchange, status: Int, body: String): Unit =
    exchange.getResponseHeaders.add("Content-Type", "application/json")
    exchange.sendResponseHeaders(status, body.length)
    val os = exchange.getResponseBody
    os.write(body.getBytes)
    os.close()
```

This is the same idea http4s expresses as `HttpRoutes.of[IO] { case GET ->
Root / "users" / IntVar(id) => ... }` — a pure function from request to
response, matched declaratively — just without the parser combinators or
the effect type wrapping it.

## Status codes as data

Model your API's outcomes as a small enum instead of scattering raw integers
through handler code — it documents every response your endpoint can give
and keeps the mapping to an HTTP status in one place:

```scala
enum ApiResult:
  case Ok(body: String)
  case NotFound(resource: String)
  case BadRequest(reason: String)

def toResponse(result: ApiResult): (Int, String) = result match
  case ApiResult.Ok(body)          => (200, body)
  case ApiResult.NotFound(res)     => (404, s"""{"error":"$res not found"}""")
  case ApiResult.BadRequest(why)   => (400, s"""{"error":"$why"}""")

println(toResponse(ApiResult.Ok("""{"id":1}""")))          // (200,{"id":1})
println(toResponse(ApiResult.NotFound("user")))            // (404,{"error":"user not found"})
```

## Cheat sheet

| Need to... | Use |
|---|---|
| Start a server on a port | `HttpServer.create(new InetSocketAddress(port), 0)` |
| Attach a handler to a path prefix | `server.createContext(path, handler)` |
| Read query params / path | `exchange.getRequestURI.getQuery` / `.getPath` |
| Send a response | `sendResponseHeaders(status, len)` then write + **close** `getResponseBody` |
| Make an HTTP call | `HttpClient.newHttpClient().send(request, BodyHandlers.ofString())` |
| Route by method + path declaratively | pattern match on `(method, path)`, or use http4s/akka-http in production |
| Model API outcomes as data | an `enum` mapped to `(status, body)` in one place |

## Exercise

Extend the `Router` above with a `POST /users` route: read the request body
with `new String(exchange.getRequestBody.readAllBytes())`, treat it as a
plain-text name, "store" it in a mutable `scala.collection.mutable.ListBuffer[String]`
declared outside the handler, and respond `201` with the new user's assigned
id. Add a `GET /users` route that lists everyone currently stored as a JSON
array. Start the server, use `HttpClient` to `POST` two names and then `GET
/users`, and print the final response body to confirm both are present.
