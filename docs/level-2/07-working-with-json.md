# 07 · Working with JSON

Almost every real Kotlin project talks to something over JSON — a REST
API, a config file, a message queue payload. `kotlinx.serialization` is
the JetBrains-maintained library for this: it works by generating
serialization code *at compile time* via a compiler plugin, so there's no
reflection, no runtime surprises, and it fails to compile (rather than
fails at runtime) if your model doesn't match what you're asking it to do.
[Module 10](10-project-weather-cli.md) uses exactly this to parse a real
weather API's response.

Setting it up needs two things in Gradle (see [Module 8](08-gradle-basics.md)):
the `kotlinx-serialization` Kotlin compiler plugin, and the
`kotlinx-serialization-json` library dependency.

## `@Serializable` data classes

Mark a data class `@Serializable` and `Json.encodeToString`/
`decodeFromString` handle the conversion both ways.

```kotlin
import kotlinx.serialization.Serializable
import kotlinx.serialization.encodeToString
import kotlinx.serialization.json.Json

@Serializable
data class Product(val name: String, val price: Double, val inStock: Boolean)

fun main() {
    val product = Product("Widget", 9.99, true)

    val json = Json.encodeToString(product)
    println(json)

    val parsed = Json.decodeFromString<Product>(json)
    println(parsed)
    println(parsed == product)   // true -- data class structural equality
}
```

```text
{"name":"Widget","price":9.99,"inStock":true}
Product(name=Widget, price=9.99, inStock=true)
true
```

## Nested objects and lists

Serialization works recursively — a `@Serializable` class containing other
`@Serializable` classes or lists of them just works.

```kotlin
import kotlinx.serialization.Serializable
import kotlinx.serialization.encodeToString
import kotlinx.serialization.json.Json

@Serializable
data class Address(val city: String, val zip: String)

@Serializable
data class Customer(val name: String, val address: Address, val orderIds: List<Int>)

fun main() {
    val customer = Customer("Alice", Address("Boston", "02101"), listOf(101, 102, 103))
    val json = Json.encodeToString(customer)
    println(json)

    val parsed = Json.decodeFromString<Customer>(json)
    println(parsed.address.city)     // Boston
    println(parsed.orderIds.sum())   // 306
}
```

```text
{"name":"Alice","address":{"city":"Boston","zip":"02101"},"orderIds":[101,102,103]}
Boston
306
```

## Optional and nullable fields

A property with a default value doesn't need to appear in the incoming
JSON at all; a nullable property can be present but `null`.

```kotlin
import kotlinx.serialization.Serializable
import kotlinx.serialization.json.Json

@Serializable
data class UserProfile(
    val username: String,
    val bio: String? = null,       // may be missing OR explicitly null in the JSON
    val loginCount: Int = 0        // may be missing -- defaults to 0
)

fun main() {
    val fullJson = """{"username": "alice", "bio": "Kotlin fan", "loginCount": 42}"""
    val minimalJson = """{"username": "bob"}"""

    println(Json.decodeFromString<UserProfile>(fullJson))
    println(Json.decodeFromString<UserProfile>(minimalJson))
}
```

```text
UserProfile(username=alice, bio=Kotlin fan, loginCount=42)
UserProfile(username=bob, bio=null, loginCount=0)
```

## Matching real-world JSON field names with `@SerialName`

APIs frequently use `snake_case` while idiomatic Kotlin uses `camelCase`.
`@SerialName` maps a property to whatever the JSON actually calls it,
without forcing your Kotlin code to look like the wire format.

```kotlin
import kotlinx.serialization.Serializable
import kotlinx.serialization.SerialName
import kotlinx.serialization.json.Json

@Serializable
data class WeatherReading(
    @SerialName("temperature_2m") val temperature: Double,
    @SerialName("wind_speed_10m") val windSpeed: Double
)

fun main() {
    val apiResponse = """{"temperature_2m": 19.4, "wind_speed_10m": 11.9}"""
    val reading = Json.decodeFromString<WeatherReading>(apiResponse)
    println("${reading.temperature}°C, wind ${reading.windSpeed} km/h")
}
```

```text
19.4°C, wind 11.9 km/h
```

## Handling fields you don't care about

By default, `kotlinx.serialization` throws if the JSON contains a key your
data class doesn't declare — a real API response almost always has more
fields than you need. Configure a `Json` instance with
`ignoreUnknownKeys = true` to skip them instead of failing.

```kotlin
import kotlinx.serialization.Serializable
import kotlinx.serialization.json.Json

@Serializable
data class Coordinates(val latitude: Double, val longitude: Double)

private val lenientJson = Json { ignoreUnknownKeys = true }

fun main() {
    // Real API responses include lots of fields we don't model here
    val apiResponse = """
        {"latitude": 51.51, "longitude": -0.13, "elevation": 16.0, "timezone": "GMT"}
    """.trimIndent()

    val coords = lenientJson.decodeFromString<Coordinates>(apiResponse)
    println(coords)   // Coordinates(latitude=51.51, longitude=-0.13) -- extra fields ignored
}
```

```text
Coordinates(latitude=51.51, longitude=-0.13)
```

!!! warning "The default `Json` is strict on purpose"
    Without `ignoreUnknownKeys = true`, an unexpected field throws
    `SerializationException` rather than silently dropping data. That's a
    deliberate safety net — if your model is missing a field that
    *actually matters*, you find out immediately instead of quietly losing
    information. Reach for the lenient config only once you've confirmed
    which fields you genuinely don't need.

## Enums in JSON

A `@Serializable enum class` (de)serializes to its constant name as a
plain JSON string, no extra configuration needed.

```kotlin
import kotlinx.serialization.Serializable
import kotlinx.serialization.encodeToString
import kotlinx.serialization.json.Json

@Serializable
enum class Priority { LOW, MEDIUM, HIGH }

@Serializable
data class Ticket(val title: String, val priority: Priority)

fun main() {
    val ticket = Ticket("Fix login bug", Priority.HIGH)
    println(Json.encodeToString(ticket))

    val parsed = Json.decodeFromString<Ticket>("""{"title":"Fix login bug","priority":"HIGH"}""")
    println(parsed.priority)
}
```

```text
{"title":"Fix login bug","priority":"HIGH"}
HIGH
```

## Cheat sheet

| Task | Syntax |
|---|---|
| Mark a type serializable | `@Serializable data class Foo(...)` |
| Object to JSON string | `Json.encodeToString(value)` |
| JSON string to object | `Json.decodeFromString<Foo>(jsonString)` |
| Rename a field for the JSON key | `@SerialName("json_key") val ktName: Type` |
| Optional field with a default | `val field: Type = defaultValue` |
| Nullable field | `val field: Type? = null` |
| Ignore unmodeled JSON fields | `Json { ignoreUnknownKeys = true }` |
| Enum as a JSON string | `@Serializable enum class Foo { A, B }` |

## Exercise

Model a `@Serializable data class Book(val title: String, @SerialName("author_name")
val authorName: String, val year: Int, val genres: List<String> = emptyList())`.
Write a JSON string representing a book with a genres list, decode it, and
print the result. Then write a *second* JSON string that omits `genres`
entirely and confirm it still decodes successfully with an empty list. Use
a lenient `Json { ignoreUnknownKeys = true }` instance and add an extra
unmodeled field (like `"isbn": "..."`) to confirm it's ignored rather than
throwing.
