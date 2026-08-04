# 10 · Project — Weather CLI

The Level 1 capstone was a single `.kt` file compiled straight with
`kotlinc`. This one is a real Gradle project: multiple source files, two
external dependencies, a test suite, and live network calls. It pulls
together nearly everything in Level 2 —
[coroutines](03-coroutines-basics.md) for concurrent HTTP,
[advanced null safety](04-null-safety-advanced.md) for API fields that may
simply not be there, [collections](05-collections-deep-dive.md) for
reshaping the response, [sealed types](01-oop-deep-dive.md) for modelling
per-city outcomes, [kotlinx.serialization](07-working-with-json.md) for
parsing, [Gradle](08-gradle-basics.md) for building, and
[tests](06-testing-junit-kotest.md) for the parts that don't need a
network.

## What you'll build

A `weather` command that takes one or more city names and prints current
conditions plus a three-day outlook for each:

- Resolves each city name to coordinates via Open-Meteo's geocoding API
- Fetches the forecast for those coordinates
- Runs every city's lookup **concurrently**, not one after another
- Handles "city doesn't exist" and "network blew up" as ordinary results
  rather than crashes
- Reports which of the requested cities is warmest right now

## The API

[Open-Meteo](https://open-meteo.com) is free for non-commercial use and
needs **no API key** — which is exactly why it's a good teaching target.
Two endpoints are involved.

**Geocoding**, to turn a name into coordinates:

```text
https://geocoding-api.open-meteo.com/v1/search?name=Hyderabad&count=1&language=en&format=json
```

```json
{"results":[{"id":1269843,"name":"Hyderabad","latitude":17.38405,
  "longitude":78.45636,"elevation":515.0,"country_code":"IN",
  "timezone":"Asia/Kolkata","population":6993262,"country":"India",
  "admin1":"Telangana","admin2":"Hyderabad District"}],
 "generationtime_ms":0.386}
```

**Forecast**, to get the weather at those coordinates:

```text
https://api.open-meteo.com/v1/forecast?latitude=51.5&longitude=-0.13
  &current=temperature_2m,wind_speed_10m,weather_code&timezone=auto
```

```json
{"latitude":51.51,"longitude":-0.13,"timezone":"Europe/London",
 "current":{"time":"2026-08-04T15:15","interval":900,
            "temperature_2m":27.7,"wind_speed_10m":21.2,"weather_code":3}}
```

Two details shape the whole design. First, when nothing matches a search
the `results` key is **absent entirely** — not an empty list — so the
Kotlin field has to be `List<Place>?` with a default. Second, the daily
forecast comes back as *parallel arrays* (`time`, `temperature_2m_max`,
`temperature_2m_min`, `weather_code`), not a list of day objects, so
something has to zip them back together.

## Project layout

```text
weather-cli/
    build.gradle.kts
    settings.gradle.kts
    src/
        main/kotlin/com/example/weather/
            Models.kt        -- @Serializable API shapes + the sealed result type
            Format.kt        -- pure functions: codes, dates, rendering
            WeatherApi.kt    -- suspend functions that do the HTTP
            Main.kt          -- argument parsing + concurrent fetch
        test/kotlin/com/example/weather/
            WeatherTest.kt
```

## `build.gradle.kts`

```kotlin
plugins {
    kotlin("jvm") version "2.4.10"
    kotlin("plugin.serialization") version "2.4.10"   // the serialization compiler plugin
    application
}

version = "1.0.0"

repositories {
    mavenCentral()
}

dependencies {
    implementation("org.jetbrains.kotlinx:kotlinx-coroutines-core:1.11.0")
    implementation("org.jetbrains.kotlinx:kotlinx-serialization-json:1.9.0")
    testImplementation(kotlin("test"))
}

application {
    mainClass.set("com.example.weather.MainKt")
}

tasks.test {
    useJUnitPlatform()
    testLogging {
        events("passed", "failed")   // print one line per test
    }
}
```

```kotlin
// settings.gradle.kts
rootProject.name = "weather-cli"
```

There's no HTTP library dependency — the JDK's built-in
`java.net.http.HttpClient` is enough.

## `Models.kt` — the shapes

```kotlin
package com.example.weather

import kotlinx.serialization.SerialName
import kotlinx.serialization.Serializable

/** One match from the geocoding API. `country`/`admin1` are often absent. */
@Serializable
data class Place(
    val name: String,
    val latitude: Double,
    val longitude: Double,
    val country: String? = null,
    val admin1: String? = null,
) {
    /** "London, England, United Kingdom" -- skipping whichever parts are null. */
    val label: String
        get() = listOfNotNull(name, admin1, country).joinToString(", ")
}

/** `results` is missing entirely -- not an empty list -- when nothing matches. */
@Serializable
data class GeocodingResponse(val results: List<Place>? = null)

@Serializable
data class CurrentWeather(
    val time: String,
    @SerialName("temperature_2m") val temperature: Double,
    @SerialName("wind_speed_10m") val windSpeed: Double,
    @SerialName("weather_code") val weatherCode: Int,
)

/** The API returns parallel arrays, one entry per day, not a list of objects. */
@Serializable
data class DailyBlock(
    val time: List<String>,
    @SerialName("temperature_2m_max") val maxTemps: List<Double>,
    @SerialName("temperature_2m_min") val minTemps: List<Double>,
    @SerialName("weather_code") val weatherCodes: List<Int>,
)

@Serializable
data class ForecastResponse(
    val timezone: String,
    val current: CurrentWeather,
    val daily: DailyBlock,
)

/** One day, after the parallel arrays have been zipped back together. */
data class DayForecast(
    val date: String,
    val minTemp: Double,
    val maxTemp: Double,
    val weatherCode: Int,
)

/** What the CLI actually prints: a place plus its weather. */
data class Report(val place: Place, val forecast: ForecastResponse)

/**
 * A per-city outcome. Sealed, so the `when` in `render()` is exhaustive and
 * the compiler will flag any new case we forget to handle.
 */
sealed interface CityResult {
    data class Success(val report: Report) : CityResult
    data class NotFound(val query: String) : CityResult
    data class Failed(val query: String, val reason: String) : CityResult
}
```

`@SerialName` maps snake_case JSON keys onto idiomatic Kotlin names, so
nothing downstream has to say `temperature_2m`. `listOfNotNull(...)` in
`label` is the null-safety idiom doing real work: a city with no `admin1`
just prints one fewer comma-separated part, with no `if` ladder.

## `Format.kt` — pure functions

Everything here is deterministic and network-free, which is what makes the
test suite easy.

```kotlin
package com.example.weather

import java.time.LocalDate
import java.time.format.DateTimeFormatter
import java.util.Locale

/** WMO weather interpretation codes -- the subset Open-Meteo actually emits. */
private val WEATHER_CODES: Map<Int, String> = mapOf(
    0 to "Clear sky", 1 to "Mainly clear", 2 to "Partly cloudy", 3 to "Overcast",
    45 to "Fog", 48 to "Depositing rime fog",
    51 to "Light drizzle", 53 to "Moderate drizzle", 55 to "Dense drizzle",
    61 to "Slight rain", 63 to "Moderate rain", 65 to "Heavy rain",
    71 to "Slight snow", 73 to "Moderate snow", 75 to "Heavy snow",
    80 to "Rain showers", 81 to "Moderate rain showers", 82 to "Violent rain showers",
    95 to "Thunderstorm", 96 to "Thunderstorm with hail",
)

fun describeWeatherCode(code: Int): String = WEATHER_CODES[code] ?: "Unknown ($code)"

private val DAY_FORMAT: DateTimeFormatter =
    DateTimeFormatter.ofPattern("EEE dd MMM", Locale.ENGLISH)

fun formatDate(isoDate: String): String =
    runCatching { LocalDate.parse(isoDate).format(DAY_FORMAT) }.getOrDefault(isoDate)

/**
 * Zip the API's four parallel arrays back into one object per day, stopping at
 * the shortest array so a truncated response can't blow up with an index error.
 */
fun DailyBlock.toDays(): List<DayForecast> {
    val count = minOf(time.size, minTemps.size, maxTemps.size, weatherCodes.size)
    return (0 until count).map { i ->
        DayForecast(time[i], minTemps[i], maxTemps[i], weatherCodes[i])
    }
}

fun Double.asTemp(): String = "%.1f°C".format(Locale.ENGLISH, this)

fun Report.render(): String = buildString {
    appendLine("${place.label}  (${forecast.timezone})")
    appendLine(
        "  Now: ${forecast.current.temperature.asTemp()}, " +
            "${describeWeatherCode(forecast.current.weatherCode)}, " +
            "wind %.1f km/h".format(Locale.ENGLISH, forecast.current.windSpeed)
    )
    forecast.daily.toDays().forEach { day ->
        appendLine(
            "  ${formatDate(day.date)}  " +
                "${day.minTemp.asTemp()} - ${day.maxTemp.asTemp()}  " +
                describeWeatherCode(day.weatherCode)
        )
    }
}.trimEnd()

/** Exhaustive over the sealed CityResult -- no `else` branch needed. */
fun CityResult.render(): String = when (this) {
    is CityResult.Success -> report.render()
    is CityResult.NotFound -> "$query: no matching place found"
    is CityResult.Failed -> "$query: lookup failed ($reason)"
}

/** Warmest city among the successful lookups, or null if none succeeded. */
fun List<CityResult>.warmest(): Report? =
    filterIsInstance<CityResult.Success>()
        .map { it.report }
        .maxByOrNull { it.forecast.current.temperature }
```

`warmest()` is a three-line pipeline that would be a loop with a
`bestSoFar` variable in most languages: `filterIsInstance` narrows the
sealed type *and* casts in one step, and `maxByOrNull` returns `null`
rather than throwing on an empty list, which the caller then handles with
an Elvis.

## `WeatherApi.kt` — the coroutine layer

```kotlin
package com.example.weather

import java.net.URI
import java.net.URLEncoder
import java.net.http.HttpClient
import java.net.http.HttpRequest
import java.net.http.HttpResponse
import java.nio.charset.StandardCharsets
import java.time.Duration
import kotlinx.coroutines.Dispatchers
import kotlinx.coroutines.withContext
import kotlinx.serialization.json.Json

class WeatherApi(
    private val client: HttpClient = HttpClient.newBuilder()
        .connectTimeout(Duration.ofSeconds(10))
        .build(),
) {
    // Open-Meteo sends far more fields than we model, so be lenient.
    private val json = Json { ignoreUnknownKeys = true }

    /** Blocking HTTP, moved off the caller's thread onto the IO dispatcher. */
    private suspend fun get(url: String): String = withContext(Dispatchers.IO) {
        val request = HttpRequest.newBuilder(URI.create(url))
            .timeout(Duration.ofSeconds(15))
            .header("Accept", "application/json")
            .GET()
            .build()
        val response = client.send(request, HttpResponse.BodyHandlers.ofString())
        if (response.statusCode() != 200) {
            throw WeatherApiException("HTTP ${response.statusCode()} from ${URI.create(url).host}")
        }
        response.body()
    }

    /** Returns null when the API has no match for [city] -- not an exception. */
    suspend fun geocode(city: String): Place? {
        val encoded = URLEncoder.encode(city, StandardCharsets.UTF_8)
        val body = get("$GEOCODING_URL?name=$encoded&count=1&language=en&format=json")
        return json.decodeFromString<GeocodingResponse>(body).results?.firstOrNull()
    }

    suspend fun forecast(place: Place, days: Int = 3): ForecastResponse {
        val body = get(
            buildString {
                append(FORECAST_URL)
                append("?latitude=${place.latitude}&longitude=${place.longitude}")
                append("&current=temperature_2m,wind_speed_10m,weather_code")
                append("&daily=temperature_2m_max,temperature_2m_min,weather_code")
                append("&forecast_days=$days&timezone=auto")
            }
        )
        return json.decodeFromString<ForecastResponse>(body)
    }

    /** Geocode then forecast -- the two calls a single city needs. */
    suspend fun lookup(city: String, days: Int = 3): CityResult =
        try {
            val place = geocode(city)
            if (place == null) {
                CityResult.NotFound(city)
            } else {
                CityResult.Success(Report(place, forecast(place, days)))
            }
        } catch (e: Exception) {
            CityResult.Failed(city, e.message ?: e::class.simpleName ?: "unknown error")
        }

    companion object {
        const val GEOCODING_URL = "https://geocoding-api.open-meteo.com/v1/search"
        const val FORECAST_URL = "https://api.open-meteo.com/v1/forecast"
    }
}

class WeatherApiException(message: String) : Exception(message)
```

`HttpClient.send()` blocks the calling thread, so it's wrapped in
`withContext(Dispatchers.IO)` — the pattern from
[Module 3](03-coroutines-basics.md) for making a blocking API safe to call
from a coroutine. Note the *two* different failure styles: a city that
doesn't exist is a normal `null`/`NotFound` value, while a broken network
is an exception, caught once in `lookup` and converted into a `Failed`
result. Nothing above this layer ever needs a `try`/`catch`.

## `Main.kt` — fetching concurrently

```kotlin
package com.example.weather

import kotlin.system.exitProcess
import kotlinx.coroutines.async
import kotlinx.coroutines.awaitAll
import kotlinx.coroutines.coroutineScope
import kotlinx.coroutines.runBlocking

private const val USAGE = "Usage: weather <city> [city ...]"

/** Fetch every city concurrently instead of one after another. */
suspend fun fetchAll(api: WeatherApi, cities: List<String>): List<CityResult> =
    coroutineScope {
        cities.map { city -> async { api.lookup(city) } }.awaitAll()
    }

fun main(args: Array<String>) = runBlocking {
    val cities = args.filter { it.isNotBlank() }
    if (cities.isEmpty()) {
        println(USAGE)
        exitProcess(1)
    }

    val results = fetchAll(WeatherApi(), cities)
    results.forEach { println(it.render()) }

    if (cities.size > 1) {
        val warmest = results.warmest()
        println(
            if (warmest == null) "No city could be resolved."
            else "Warmest right now: ${warmest.place.name} " +
                "at ${warmest.forecast.current.temperature.asTemp()}"
        )
    }

    if (results.any { it !is CityResult.Success }) exitProcess(2)
}
```

`fetchAll` is the heart of it: `map { async { ... } }` starts every city's
lookup immediately, and `awaitAll()` waits for all of them. Three cities
take about as long as the slowest one, not the sum of all three — but
`awaitAll` still returns results **in argument order**, so output stays
predictable. Because it's a `coroutineScope`, a crash in one lookup would
cancel the rest instead of leaking a runaway coroutine.

## `WeatherTest.kt` — testing without the network

The pure functions and the parsing are all testable offline; only
`WeatherApi.get()` touches the internet.

```kotlin
package com.example.weather

import kotlin.test.Test
import kotlin.test.assertEquals
import kotlin.test.assertNull
import kotlin.test.assertTrue
import kotlinx.serialization.json.Json

class WeatherTest {

    private val json = Json { ignoreUnknownKeys = true }

    @Test
    fun `known weather codes map to descriptions`() {
        assertEquals("Clear sky", describeWeatherCode(0))
        assertEquals("Overcast", describeWeatherCode(3))
        assertEquals("Thunderstorm", describeWeatherCode(95))
    }

    @Test
    fun `unknown weather codes degrade gracefully`() {
        assertEquals("Unknown (999)", describeWeatherCode(999))
    }

    @Test
    fun `parallel daily arrays zip into one object per day`() {
        val block = DailyBlock(
            time = listOf("2026-08-04", "2026-08-05"),
            maxTemps = listOf(29.4, 24.4),
            minTemps = listOf(20.0, 18.3),
            weatherCodes = listOf(3, 61),
        )
        val days = block.toDays()
        assertEquals(2, days.size)
        assertEquals(DayForecast("2026-08-04", 20.0, 29.4, 3), days[0])
        assertEquals(61, days[1].weatherCode)
    }

    @Test
    fun `a truncated daily block stops at the shortest array`() {
        val block = DailyBlock(
            time = listOf("2026-08-04", "2026-08-05"),
            maxTemps = listOf(29.4),          // one value short
            minTemps = listOf(20.0, 18.3),
            weatherCodes = listOf(3, 61),
        )
        assertEquals(1, block.toDays().size)   // no IndexOutOfBoundsException
    }

    @Test
    fun `place label skips null parts`() {
        assertEquals(
            "London, England, United Kingdom",
            Place("London", 51.5, -0.13, country = "United Kingdom", admin1 = "England").label,
        )
        assertEquals("Atlantis", Place("Atlantis", 0.0, 0.0).label)
    }

    @Test
    fun `geocoding response with no results decodes to null`() {
        val body = """{"generationtime_ms":0.6}"""
        val decoded = json.decodeFromString<GeocodingResponse>(body)
        assertNull(decoded.results?.firstOrNull())
    }

    @Test
    fun `forecast json decodes into the model`() {
        val body = """
            {"latitude":51.51,"longitude":-0.13,"elevation":16.0,"timezone":"Europe/London",
             "current":{"time":"2026-08-04T10:30","interval":900,"temperature_2m":25.3,
                        "wind_speed_10m":8.6,"weather_code":2},
             "daily":{"time":["2026-08-04"],"temperature_2m_max":[29.4],
                      "temperature_2m_min":[20.0],"weather_code":[3]}}
        """.trimIndent()
        val forecast = json.decodeFromString<ForecastResponse>(body)
        assertEquals(25.3, forecast.current.temperature)
        assertEquals("Partly cloudy", describeWeatherCode(forecast.current.weatherCode))
        assertEquals(1, forecast.daily.toDays().size)
    }

    @Test
    fun `failed and not-found results render without throwing`() {
        assertEquals("Narnia: no matching place found", CityResult.NotFound("Narnia").render())
        assertTrue(CityResult.Failed("Paris", "timeout").render().contains("timeout"))
    }

    @Test
    fun `warmest picks the highest current temperature and ignores failures`() {
        val results = listOf(
            CityResult.Success(sampleReport("Oslo", 14.2)),
            CityResult.NotFound("Narnia"),
            CityResult.Success(sampleReport("Cairo", 36.8)),
        )
        assertEquals("Cairo", results.warmest()?.place?.name)
        assertNull(listOf<CityResult>(CityResult.NotFound("Narnia")).warmest())
    }

    @Test
    fun `dates format as English day labels`() {
        assertEquals("Tue 04 Aug", formatDate("2026-08-04"))
        assertEquals("not-a-date", formatDate("not-a-date"))   // falls back to the raw string
    }

    private fun sampleReport(city: String, temp: Double) = Report(
        place = Place(city, 0.0, 0.0),
        forecast = ForecastResponse(
            timezone = "UTC",
            current = CurrentWeather("2026-08-04T10:30", temp, 5.0, 0),
            daily = DailyBlock(listOf("2026-08-04"), listOf(temp), listOf(temp - 5), listOf(0)),
        ),
    )
}
```

That extra `"interval":900` field in the test JSON is deliberate: it isn't
in `CurrentWeather`, and the test proves `ignoreUnknownKeys = true` copes
with the real API sending more than you modelled.

## Running it

```bash
./gradlew test
```

```text
> Task :test
WeatherTest > forecast json decodes into the model() PASSED
WeatherTest > warmest picks the highest current temperature and ignores failures() PASSED
WeatherTest > known weather codes map to descriptions() PASSED
WeatherTest > a truncated daily block stops at the shortest array() PASSED
WeatherTest > parallel daily arrays zip into one object per day() PASSED
WeatherTest > dates format as English day labels() PASSED
WeatherTest > failed and not-found results render without throwing() PASSED
WeatherTest > unknown weather codes degrade gracefully() PASSED
WeatherTest > geocoding response with no results decodes to null() PASSED
WeatherTest > place label skips null parts() PASSED

BUILD SUCCESSFUL in 3s
```

`installDist` builds a launcher script, which is nicer than `./gradlew run`
for a CLI that returns meaningful exit codes:

```bash
./gradlew installDist
./build/install/weather-cli/bin/weather-cli London Hyderabad Reykjavik
```

```text
London, England, United Kingdom  (Europe/London)
  Now: 27.6°C, Overcast, wind 21.2 km/h
  Tue 04 Aug  20.0°C - 29.7°C  Overcast
  Wed 05 Aug  18.2°C - 24.3°C  Overcast
  Thu 06 Aug  14.9°C - 22.1°C  Overcast
Hyderabad, Telangana, India  (Asia/Kolkata)
  Now: 25.5°C, Light drizzle, wind 6.0 km/h
  Tue 04 Aug  23.2°C - 28.8°C  Thunderstorm
  Wed 05 Aug  22.7°C - 29.9°C  Dense drizzle
  Thu 06 Aug  23.6°C - 29.7°C  Moderate drizzle
Reykjavik, Capital Region, Iceland  (Atlantic/Reykjavik)
  Now: 13.2°C, Mainly clear, wind 11.9 km/h
  Tue 04 Aug  10.1°C - 13.4°C  Overcast
  Wed 05 Aug  10.4°C - 13.0°C  Overcast
  Thu 06 Aug  10.0°C - 11.7°C  Slight rain
Warmest right now: London at 27.6°C
```

An unresolvable city degrades instead of crashing — the other cities still
print, and the process exits `2`:

```text
$ ./build/install/weather-cli/bin/weather-cli Zzqqxyz Oslo
Zzqqxyz: no matching place found
Oslo, Oslo, Norway  (Europe/Oslo)
  Now: 23.3°C, Overcast, wind 15.1 km/h
  Tue 04 Aug  14.1°C - 23.6°C  Dense drizzle
  Wed 05 Aug  14.7°C - 18.3°C  Heavy rain
  Thu 06 Aug  16.7°C - 20.1°C  Slight rain
Warmest right now: Oslo at 23.3°C

$ echo $?
2
```

!!! tip "Try a name you think is fictional"
    `Narnia` resolves — there's a real place by that name in Bangladesh.
    Geocoders match far more strings than you'd expect, which is exactly
    why `NotFound` has to be a normal, printable result rather than an
    edge case you hope never happens.

## Where each Level 2 idea shows up

| Module | In this project |
|---|---|
| 01 · OOP | `sealed interface CityResult` with an exhaustive `when` in `render()` |
| 02 · Lambdas | `map`/`filter`/`forEach` pipelines; `runCatching { }.getOrDefault(...)` |
| 03 · Coroutines | `suspend` HTTP, `withContext(Dispatchers.IO)`, `async`/`awaitAll` in a `coroutineScope` |
| 04 · Null safety | `List<Place>?`, `?.firstOrNull()`, `listOfNotNull`, Elvis fallbacks |
| 05 · Collections | `filterIsInstance`, `maxByOrNull`, zipping the parallel daily arrays |
| 06 · Testing | Offline tests over the pure functions and JSON decoding |
| 07 · JSON | `@Serializable`, `@SerialName`, `ignoreUnknownKeys` |
| 08 · Gradle | Two plugins, two dependencies, `application`, `installDist` |

## Stretch goals

- **Cache lookups.** Geocoding results never change; store them in a
  `Map<String, Place>` on disk as JSON so repeat runs skip a request.
- **Add units.** A `--fahrenheit` flag — Open-Meteo accepts
  `&temperature_unit=fahrenheit`, but doing the conversion in `asTemp()`
  keeps the API layer unchanged.
- **Make it testable without the network.** Extract an interface for
  `get(url)`, pass a fake implementation returning canned JSON, and test
  `lookup()` end to end with `runTest` from `kotlinx-coroutines-test`.
- **Add a timeout per city.** Wrap each `async` body in
  `withTimeoutOrNull(5_000)` and turn a `null` into
  `CityResult.Failed(city, "timed out")`.
- **Hourly view.** Request `&hourly=temperature_2m` and use
  `chunked`/`windowed` from [Module 5](05-collections-deep-dive.md) to
  print a compact sparkline of the next 24 hours.
- **Generic response envelope.** Rework the decoding through a
  `Result<T>`-style wrapper using [Module 9's](09-generics.md) generics so
  every endpoint shares one error-handling path.

Finishing this means you've built a real multi-file, dependency-managed,
tested Kotlin application against a live API — you're ready for
**Level 3 · Advanced**.
