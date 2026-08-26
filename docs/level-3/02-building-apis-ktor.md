# 02 · Building APIs with Ktor

[Module 8 of Level 2](../level-2/08-gradle-basics.md) covered Gradle, and
`suspend` functions from [Level 2's coroutines module](../level-2/03-coroutines-basics.md)
are what make Ktor's request handlers non-blocking under the hood. Ktor is
JetBrains's Kotlin-first web framework — this module builds a small JSON
REST API with it: routing, request/response serialization, and structured
error handling.

Ktor needs a real Gradle project (it pulls in Netty, serialization, and
several Ktor modules), so every example below is one `main()` from a
project with this `build.gradle.kts`:

```kotlin
plugins {
    kotlin("jvm") version "2.0.20"
    kotlin("plugin.serialization") version "2.0.20"
    application
}
repositories { mavenCentral() }
val ktorVersion = "2.3.12"
dependencies {
    implementation("io.ktor:ktor-server-core-jvm:$ktorVersion")
    implementation("io.ktor:ktor-server-netty-jvm:$ktorVersion")
    implementation("io.ktor:ktor-server-content-negotiation-jvm:$ktorVersion")
    implementation("io.ktor:ktor-serialization-kotlinx-json-jvm:$ktorVersion")
    implementation("io.ktor:ktor-server-status-pages-jvm:$ktorVersion")
    implementation("io.ktor:ktor-client-core-jvm:$ktorVersion")
    implementation("io.ktor:ktor-client-cio-jvm:$ktorVersion")
    implementation("io.ktor:ktor-client-content-negotiation-jvm:$ktorVersion")
}
application { mainClass.set("MainKt") }
```

The `kotlin("plugin.serialization")` plugin is easy to forget — without it,
`@Serializable` classes compile fine but fail at runtime with
"Serializer for class 'X' is not found."

## Defining routes and responding with JSON

Ktor routing reads like a small DSL: HTTP verbs map to functions, and
`call.respond` serializes any `@Serializable` object using whatever
`ContentNegotiation` converter you installed.

```kotlin
import io.ktor.server.application.*
import io.ktor.server.engine.*
import io.ktor.server.netty.*
import io.ktor.server.routing.*
import io.ktor.server.response.*
import io.ktor.server.request.*
import io.ktor.server.plugins.contentnegotiation.*
import io.ktor.serialization.kotlinx.json.*
import io.ktor.http.*
import kotlinx.serialization.Serializable

@Serializable
data class Task(val id: Int, val title: String, val done: Boolean = false)

@Serializable
data class TaskList(val tasks: List<Task>)

object TaskRepo {
    private val tasks = mutableMapOf(
        1 to Task(1, "Write Ktor module", done = false),
        2 to Task(2, "Ship it", done = false)
    )
    private var nextId = 3
    fun all() = tasks.values.toList()
    fun get(id: Int) = tasks[id] ?: throw TaskNotFoundException(id)
    fun add(title: String): Task {
        val t = Task(nextId++, title)
        tasks[t.id] = t
        return t
    }
    fun complete(id: Int): Task {
        val t = get(id)
        val updated = t.copy(done = true)
        tasks[id] = updated
        return updated
    }
}

class TaskNotFoundException(id: Int) : Exception("Task $id not found")

fun main() {
    embeddedServer(Netty, port = 8080) {
        install(ContentNegotiation) { json() }
        routing {
            get("/tasks") { call.respond(TaskList(TaskRepo.all())) }
            get("/tasks/{id}") {
                val id = call.parameters["id"]!!.toInt()
                call.respond(TaskRepo.get(id))
            }
            post("/tasks") {
                val title = call.receive<Map<String, String>>()["title"] ?: "Untitled"
                call.respond(HttpStatusCode.Created, TaskRepo.add(title))
            }
            post("/tasks/{id}/complete") {
                val id = call.parameters["id"]!!.toInt()
                call.respond(TaskRepo.complete(id))
            }
        }
    }.start(wait = true)
}
```

A list response (`TaskList`) is wrapped in a small data class rather than
returned as a bare `List<Task>` directly — `kotlinx.serialization` needs a
concrete, statically-known type to pick a serializer for, and a bare
generic `List<Task>` return type from a lambda erases to `List<*>` at the
call site, which fails to serialize at runtime.

## Calling it with Ktor's HTTP client, and structured errors

Adding `install(StatusPages)` turns thrown exceptions into proper HTTP
responses instead of raw 500s with a stack trace. Here the whole thing —
server startup, four real requests via `HttpClient`, and shutdown — runs in
one program so the output is fully reproducible:

```kotlin
import io.ktor.server.plugins.statuspages.*
import io.ktor.client.*
import io.ktor.client.engine.cio.*
import io.ktor.client.request.*
import io.ktor.client.statement.*
import io.ktor.client.call.*
import io.ktor.client.plugins.contentnegotiation.ContentNegotiation as ClientContentNegotiation
import kotlinx.coroutines.*

fun main() {
    val server = embeddedServer(Netty, port = 8080) {
        install(ContentNegotiation) { json() }
        install(StatusPages) {
            exception<TaskNotFoundException> { call, cause ->
                call.respond(HttpStatusCode.NotFound, mapOf("error" to cause.message))
            }
        }
        routing { /* same routes as above */ }
    }.start(wait = false)

    runBlocking {
        val client = HttpClient(CIO) { install(ClientContentNegotiation) { json() } }
        delay(300) // give Netty a moment to bind

        println("GET /tasks -> ${client.get("http://127.0.0.1:8080/tasks").body<TaskList>()}")

        val created = client.post("http://127.0.0.1:8080/tasks") {
            contentType(ContentType.Application.Json)
            setBody("""{"title":"Test the endpoints"}""")
        }.body<Task>()
        println("POST /tasks -> $created")

        val completed = client.post("http://127.0.0.1:8080/tasks/1/complete").body<Task>()
        println("POST /tasks/1/complete -> $completed")

        val missing = client.get("http://127.0.0.1:8080/tasks/999")
        println("GET /tasks/999 -> status=${missing.status}")
        client.close()
    }
    server.stop(200, 200)
}
```

```text
GET /tasks -> TaskList(tasks=[Task(id=1, title=Write Ktor module, done=false), Task(id=2, title=Ship it, done=false)])
POST /tasks -> Task(id=3, title=Test the endpoints, done=false)
POST /tasks/1/complete -> Task(id=1, title=Write Ktor module, done=true)
GET /tasks/999 -> status=404 Not Found
```

`StatusPages` intercepts the `TaskNotFoundException` thrown by
`TaskRepo.get` and turns it into a clean 404 with a JSON body, rather than
letting it propagate as an unhandled 500.

## Kotlin-specific traps

- **Forgetting `kotlin("plugin.serialization")`.** `@Serializable`
  compiles without it (it's just an annotation at that point), then fails
  at runtime the first time something tries to serialize.
- **Returning a bare generic collection from `respond`.** Wrap it in a
  small `@Serializable` holder class, or use `respond<List<Task>>(...)`
  with the explicit type argument so the reified type isn't erased.
- **Routing lambdas are extension functions on `PipelineContext`.** `call`
  inside `get("/x") { ... }` is available because the lambda receiver is
  the call context — this reads like ordinary code but is
  receiver-scoped, so accidentally nesting handlers or extracting a
  lambda to a top-level function loses access to `call` unless you pass
  it explicitly.
- **`embeddedServer(...).start(wait = true)` blocks forever** — fine for a
  real service's `main`, but in a test/demo harness use
  `wait = false` and call `.stop()` yourself, as above.
- **Path parameters are always `String?`.** `call.parameters["id"]!!.toInt()`
  throws two different exceptions depending on which part fails (`NPE` for
  a missing param, `NumberFormatException` for a non-numeric one) — worth
  handling both explicitly in real code rather than a blind `!!`.

## Cheat sheet

| Concept | Kotlin/Ktor construct |
|---|---|
| Define a route | `get("/path") { }`, `post("/path") { }` in a `routing { }` block |
| Read path param | `call.parameters["id"]` (nullable `String`) |
| Read JSON body | `call.receive<T>()` |
| Send JSON response | `call.respond(value)` (needs `ContentNegotiation` + `json()`) |
| Custom status + body | `call.respond(HttpStatusCode.Created, value)` |
| Map exceptions to responses | `install(StatusPages) { exception<E> { call, e -> ... } }` |
| Call your own/another API | `HttpClient(CIO) { install(ContentNegotiation) { json() } }` |

## Exercise

Extend the task API with a `DELETE /tasks/{id}` route that removes a task
(throwing `TaskNotFoundException` if it doesn't exist, so `StatusPages`
turns it into a 404), and a `GET /tasks?done=true` query-parameter filter
that returns only completed or only incomplete tasks depending on the
value. Write a small client block that exercises both new behaviors and
prints the results.
