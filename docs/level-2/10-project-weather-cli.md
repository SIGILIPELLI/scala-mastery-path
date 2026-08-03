# 10 · Project — Weather CLI

A command-line app that looks up a city, fetches its current weather from a
real public API, and prints a short report — combining everything from
Level 2: `Either`-based error handling ([Module 4](04-option-either.md)),
sealed-trait pattern matching ([Module 2](02-pattern-matching-advanced.md)),
collections ([Module 3](03-collections-deep-dive.md)), JSON parsing
([Module 7](07-working-with-json.md)), `for`-comprehensions
([Module 9](09-for-comprehensions.md)), and ScalaTest
([Module 5](05-scalatest-testing.md)).

## The API: Open-Meteo (free, no key required)

[Open-Meteo](https://open-meteo.com/) is a free weather API that needs no
signup, no API key, and no payment — ideal for a learning project. It's
actually two endpoints:

1. **Geocoding** — turn a city name into latitude/longitude:
   `https://geocoding-api.open-meteo.com/v1/search?name=London&count=1&language=en&format=json`
2. **Forecast** — turn coordinates into current conditions:
   `https://api.open-meteo.com/v1/forecast?latitude=51.5&longitude=-0.13&current=temperature_2m,weather_code,wind_speed_10m&timezone=auto`

The forecast response's `weather_code` is a
[WMO numeric weather code](https://open-meteo.com/en/docs) — an integer
like `0` (clear sky) or `61` (slight rain) that this project decodes with a
pattern match.

## Project layout

```text
weather-cli/
├── build.sbt
└── src/
    ├── main/
    │   └── scala/
    │       ├── Models.scala
    │       ├── Sky.scala
    │       ├── WeatherClient.scala
    │       ├── Report.scala
    │       └── Main.scala
    └── test/
        └── scala/
            └── SkySpec.scala
```

```scala
// build.sbt
ThisBuild / scalaVersion := "3.3.3"

lazy val root = (project in file("."))
  .settings(
    name := "weather-cli",
    libraryDependencies ++= Seq(
      "com.lihaoyi" %% "upickle" % "3.3.1",
      "org.scalatest" %% "scalatest" % "3.2.19" % Test
    )
  )
```

HTTP requests use `java.net.http.HttpClient`, built into the JDK since Java
11 — no extra networking library needed, keeping the dependency list to
just upickle ([Module 7](07-working-with-json.md)) and ScalaTest
([Module 5](05-scalatest-testing.md)).

## Models.scala — the API response shapes

```scala
// src/main/scala/Models.scala
package weathercli

import upickle.default._

case class GeoResult(name: String, latitude: Double, longitude: Double, country: String) derives ReadWriter
case class GeoResponse(results: List[GeoResult] = Nil) derives ReadWriter

case class CurrentWeather(temperature_2m: Double, weather_code: Int, wind_speed_10m: Double) derives ReadWriter
case class ForecastResponse(current: CurrentWeather) derives ReadWriter
```

Notice `results: List[GeoResult] = Nil` rather than the more "obvious"
`Option[List[GeoResult]]`. This is a direct lesson from
[Module 7](07-working-with-json.md#the-trap-option-doesnt-serialize-the-way-youd-guess):
upickle represents `Option[T]` as a JSON array of zero or one elements,
which does **not** match how a real-world API represents "no results" (it
simply omits the `results` key entirely when a city isn't found). Giving
the field a plain `List` type with a `Nil` default reads correctly either
way — a present array parses normally, and a missing key falls back to the
default — and then ordinary Scala (`.headOption`) turns "zero or more
results" into the `Option` you actually want to work with, right where you
want it instead of fighting the JSON library over it.

## Sky.scala — decoding weather codes with pattern matching

```scala
// src/main/scala/Sky.scala
package weathercli

sealed trait Sky
case object Clear extends Sky
case object PartlyCloudy extends Sky
case object Overcast extends Sky
case object Fog extends Sky
case object Drizzle extends Sky
case object Rain extends Sky
case object Snow extends Sky
case object Showers extends Sky
case object Thunderstorm extends Sky
case class UnknownSky(code: Int) extends Sky

object Sky:
  // Mapping from the WMO weather codes Open-Meteo returns.
  def fromCode(code: Int): Sky = code match
    case 0                           => Clear
    case 1 | 2                       => PartlyCloudy
    case 3                           => Overcast
    case 45 | 48                     => Fog
    case c if (51 to 57).contains(c) => Drizzle
    case c if (61 to 67).contains(c) => Rain
    case c if (71 to 77).contains(c) => Snow
    case c if (80 to 82).contains(c) => Showers
    case 85 | 86                     => Snow
    case c if (95 to 99).contains(c) => Thunderstorm
    case other                       => UnknownSky(other)

  def describe(sky: Sky): String = sky match
    case Clear         => "Clear sky"
    case PartlyCloudy  => "Partly cloudy"
    case Overcast      => "Overcast"
    case Fog           => "Foggy"
    case Drizzle       => "Drizzle"
    case Rain          => "Rain"
    case Snow          => "Snow"
    case Showers       => "Rain showers"
    case Thunderstorm  => "Thunderstorm"
    case UnknownSky(c) => s"Unknown conditions (code $c)"
```

`UnknownSky(code)` (rather than throwing or silently defaulting to `Clear`)
means an API response with a weather code this project doesn't recognize
still produces a sensible, honest report instead of crashing or lying about
the conditions.

## WeatherClient.scala — HTTP + JSON, wrapped in `Either`

```scala
// src/main/scala/WeatherClient.scala
package weathercli

import java.net.URI
import java.net.http.{HttpClient, HttpRequest, HttpResponse}
import upickle.default._
import scala.util.Try

object WeatherClient:
  private val client = HttpClient.newHttpClient()

  private def get(url: String): Either[String, String] =
    Try {
      val request = HttpRequest.newBuilder().uri(URI.create(url)).GET().build()
      client.send(request, HttpResponse.BodyHandlers.ofString())
    }.toEither match
      case Left(e) => Left(s"network error: ${e.getMessage}")
      case Right(response) =>
        if response.statusCode() == 200 then Right(response.body())
        else Left(s"HTTP ${response.statusCode()} from $url")

  def geocode(city: String): Either[String, GeoResult] =
    val encoded = java.net.URLEncoder.encode(city, "UTF-8")
    val url = s"https://geocoding-api.open-meteo.com/v1/search?name=$encoded&count=1&language=en&format=json"
    for
      body   <- get(url)
      parsed <- Try(read[GeoResponse](body)).toEither.left.map(e => s"could not parse geocoding response: ${e.getMessage}")
      first  <- parsed.results.headOption.toRight(s"no location found for \"$city\"")
    yield first

  def forecast(latitude: Double, longitude: Double): Either[String, CurrentWeather] =
    val url = s"https://api.open-meteo.com/v1/forecast?latitude=$latitude&longitude=$longitude&current=temperature_2m,weather_code,wind_speed_10m&timezone=auto"
    for
      body   <- get(url)
      parsed <- Try(read[ForecastResponse](body)).toEither.left.map(e => s"could not parse forecast response: ${e.getMessage}")
    yield parsed.current
```

Every failure mode — a network error, a non-200 response, malformed JSON,
or a city that doesn't exist — becomes a `Left(message)` rather than an
uncaught exception. `geocode` chains three fallible steps in one
`for`-comprehension: fetch, parse, then `.headOption.toRight(...)` to turn
"zero geocoding results" into a `Left` with a specific, useful message.

## Report.scala — formatting the result

```scala
// src/main/scala/Report.scala
package weathercli

object Report:
  def render(location: GeoResult, current: CurrentWeather): String =
    val sky = Sky.fromCode(current.weather_code)
    List(
      s"Weather for ${location.name}, ${location.country}",
      f"  ${Sky.describe(sky)}",
      f"  Temperature:  ${current.temperature_2m}%.1f C",
      f"  Wind speed:   ${current.wind_speed_10m}%.1f km/h"
    ).mkString("\n")
```

Keeping formatting in its own pure function (no network, no `println`)
is what makes it directly unit-testable in `SkySpec.scala` below, without
needing a live network call in the test suite.

## Main.scala — tying it together

```scala
// src/main/scala/Main.scala
package weathercli

@main def weatherCli(args: String*): Unit =
  args.toList match
    case Nil =>
      println("Usage: weather-cli <city name>")
    case cityParts =>
      val city = cityParts.mkString(" ")
      val result =
        for
          location <- WeatherClient.geocode(city)
          current  <- WeatherClient.forecast(location.latitude, location.longitude)
        yield Report.render(location, current)

      result match
        case Right(report) => println(report)
        case Left(err)      => println(s"Error: $err")
```

`cityParts.mkString(" ")` lets a multi-word city (`sbt "run New York"`)
reassemble correctly, since sbt/the OS splits arguments on spaces before
the program ever sees them.

## SkySpec.scala — testing the pure logic

```scala
// src/test/scala/SkySpec.scala
package weathercli

import org.scalatest.flatspec.AnyFlatSpec
import org.scalatest.matchers.should.Matchers

class SkySpec extends AnyFlatSpec with Matchers:

  "Sky.fromCode" should "map 0 to Clear" in {
    Sky.fromCode(0) shouldBe Clear
  }

  it should "map 61-67 to Rain" in {
    Sky.fromCode(61) shouldBe Rain
    Sky.fromCode(65) shouldBe Rain
    Sky.fromCode(67) shouldBe Rain
  }

  it should "map 95-99 to Thunderstorm" in {
    Sky.fromCode(95) shouldBe Thunderstorm
    Sky.fromCode(99) shouldBe Thunderstorm
  }

  it should "fall back to UnknownSky for an unrecognized code" in {
    Sky.fromCode(999) shouldBe UnknownSky(999)
  }

class ReportSpec extends AnyFlatSpec with Matchers:

  "Report.render" should "include the location and formatted readings" in {
    val location = GeoResult("London", 51.5, -0.12, "United Kingdom")
    val current = CurrentWeather(temperature_2m = 18.456, weather_code = 3, wind_speed_10m = 12.3)

    val output = Report.render(location, current)

    output should include("Weather for London, United Kingdom")
    output should include("Overcast")
    output should include("18.5 C")
  }
```

Notice what's deliberately *not* tested here: `WeatherClient.geocode` and
`.forecast` themselves, since they make real network calls. Keeping the
decoding (`Sky`) and formatting (`Report`) logic in small, pure functions
means the parts most worth unit-testing don't require mocking an HTTP
client at all — only the thin `WeatherClient` layer touches the network,
and it's kept as small as possible on purpose.

## Running it

```bash
sbt "run London"
# Weather for London, United Kingdom
#   Clear sky
#   Temperature:  27.6 C
#   Wind speed:   9.7 km/h

sbt "run Tokyo"
# Weather for Tokyo, Japan
#   Partly cloudy
#   Temperature:  23.9 C
#   Wind speed:   5.2 km/h

sbt "run zzzznotacityxyz"
# Error: no location found for "zzzznotacityxyz"

sbt run
# Usage: weather-cli <city name>

sbt test
# ...
# Tests: succeeded 6, failed 0, canceled 0, ignored 0, pending 0
# All tests passed.
```

(Temperature and conditions reflect real, live weather at the time you run
it — the exact numbers above will differ by the time you try it yourself.)

## Stretch goals

- Add a `--units imperial` flag that requests
  `&temperature_unit=fahrenheit&wind_speed_unit=mph` from the forecast
  endpoint instead of converting the Celsius/km-h values yourself.
- Cache the geocoding result for a city to a local JSON file (using
  [Module 7](07-working-with-json.md)'s `write`/`read`) so a second lookup
  of the same city skips the geocoding API call.
- Package the CLI as a standalone fat jar with the `sbt-assembly` plugin
  from [Module 8](08-sbt-deep-dive.md#plugins), so it runs anywhere with
  `java -jar` and no sbt installation required.
- Extend `Sky` with a "3-day outlook" by requesting `daily=weather_code,
  temperature_2m_max,temperature_2m_min` instead of `current=...`, and
  render one line per day using the same `Sky.describe` logic.

Completing this project means you're ready for **Level 3 · Advanced**.
