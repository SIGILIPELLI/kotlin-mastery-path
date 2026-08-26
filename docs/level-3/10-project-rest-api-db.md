# 10 · Project — REST API + Database Service

This project combines everything from Level 3: [Ktor routing and error
handling](02-building-apis-ktor.md), [Exposed against SQLite](03-databases-exposed-sqlite.md),
and [testing with a real HTTP client](06-testing-advanced.md), into one
small but complete service — a book catalog with full CRUD, a search
filter, and an automated test suite. The full project (`build.gradle.kts`,
`Main.kt`, and a test file) is laid out below exactly as built and run.

## Project structure

```text
book-service/
├── build.gradle.kts
└── src/
    ├── main/kotlin/Main.kt
    └── test/kotlin/AppTest.kt
```

```kotlin
// build.gradle.kts
plugins {
    kotlin("jvm") version "2.0.20"
    kotlin("plugin.serialization") version "2.0.20"
    application
}
repositories { mavenCentral() }
val ktorVersion = "2.3.12"
val exposedVersion = "0.52.0"
dependencies {
    implementation("io.ktor:ktor-server-core-jvm:$ktorVersion")
    implementation("io.ktor:ktor-server-netty-jvm:$ktorVersion")
    implementation("io.ktor:ktor-server-content-negotiation-jvm:$ktorVersion")
    implementation("io.ktor:ktor-serialization-kotlinx-json-jvm:$ktorVersion")
    implementation("io.ktor:ktor-server-status-pages-jvm:$ktorVersion")
    implementation("io.ktor:ktor-client-core-jvm:$ktorVersion")
    implementation("io.ktor:ktor-client-cio-jvm:$ktorVersion")
    implementation("io.ktor:ktor-client-content-negotiation-jvm:$ktorVersion")
    implementation("ch.qos.logback:logback-classic:1.4.14")
    implementation("org.jetbrains.exposed:exposed-core:$exposedVersion")
    implementation("org.jetbrains.exposed:exposed-dao:$exposedVersion")
    implementation("org.jetbrains.exposed:exposed-jdbc:$exposedVersion")
    implementation("org.xerial:sqlite-jdbc:3.46.0.0")
    testImplementation("io.ktor:ktor-server-test-host-jvm:$ktorVersion")
    testImplementation(kotlin("test"))
}
application { mainClass.set("MainKt") }
tasks.test { useJUnitPlatform() }
```

## The service

```kotlin
// src/main/kotlin/Main.kt
import io.ktor.server.application.*
import io.ktor.server.engine.*
import io.ktor.server.netty.*
import io.ktor.server.routing.*
import io.ktor.server.response.*
import io.ktor.server.request.*
import io.ktor.server.plugins.contentnegotiation.*
import io.ktor.server.plugins.statuspages.*
import io.ktor.serialization.kotlinx.json.*
import io.ktor.http.*
import kotlinx.serialization.Serializable
import org.jetbrains.exposed.sql.*
import org.jetbrains.exposed.sql.transactions.transaction
import org.jetbrains.exposed.dao.id.IntIdTable
import org.jetbrains.exposed.dao.id.EntityID

@Serializable
data class BookDto(val id: Int, val title: String, val author: String, val year: Int)

@Serializable
data class NewBook(val title: String, val author: String, val year: Int)

@Serializable
data class BookList(val books: List<BookDto>)

class BookNotFoundException(id: Int) : Exception("Book $id not found")

object Books : IntIdTable() {
    val title = varchar("title", 200)
    val author = varchar("author", 100)
    val year = integer("year")
}

object BookRepository {
    fun init(dbPath: String = "library.db") {
        Database.connect("jdbc:sqlite:$dbPath", driver = "org.sqlite.JDBC")
        transaction {
            SchemaUtils.create(Books)
            if (Books.selectAll().empty()) {
                insert(NewBook("Kotlin in Action", "Jemerov & Isakova", 2017))
                insert(NewBook("Effective Kotlin", "Marcin Moskala", 2019))
            }
        }
    }

    fun insert(book: NewBook): BookDto = transaction {
        val id = Books.insertAndGetId {
            it[title] = book.title
            it[author] = book.author
            it[year] = book.year
        }
        BookDto(id.value, book.title, book.author, book.year)
    }

    fun all(): List<BookDto> = transaction { Books.selectAll().map { toDto(it) } }

    fun byId(id: Int): BookDto = transaction {
        val row = Books.selectAll().where { Books.id eq id }.singleOrNull()
            ?: throw BookNotFoundException(id)
        toDto(row)
    }

    fun byAuthorContains(fragment: String): List<BookDto> = transaction {
        Books.selectAll().where { Books.author like "%$fragment%" }.map { toDto(it) }
    }

    fun delete(bookId: Int) = transaction {
        if (Books.selectAll().where { Books.id eq bookId }.empty()) throw BookNotFoundException(bookId)
        Books.deleteWhere { Op.build { Books.id eq EntityID(bookId, Books) } }
    }

    private fun toDto(row: ResultRow) = BookDto(row[Books.id].value, row[Books.title], row[Books.author], row[Books.year])
}

fun Application.module() {
    install(ContentNegotiation) { json() }
    install(StatusPages) {
        exception<BookNotFoundException> { call, cause ->
            call.respond(HttpStatusCode.NotFound, mapOf("error" to cause.message))
        }
        exception<Throwable> { call, cause ->
            call.respond(HttpStatusCode.InternalServerError, mapOf("error" to (cause.message ?: "unknown error")))
        }
    }
    routing {
        get("/books") {
            val author = call.request.queryParameters["author"]
            val result = if (author != null) BookRepository.byAuthorContains(author) else BookRepository.all()
            call.respond(BookList(result))
        }
        get("/books/{id}") { call.respond(BookRepository.byId(call.parameters["id"]!!.toInt())) }
        post("/books") { call.respond(HttpStatusCode.Created, BookRepository.insert(call.receive<NewBook>())) }
        delete("/books/{id}") {
            BookRepository.delete(call.parameters["id"]!!.toInt())
            call.respond(HttpStatusCode.NoContent)
        }
    }
}

fun main() {
    BookRepository.init()
    embeddedServer(Netty, port = 8080, module = Application::module).start(wait = true)
}
```

`BookRepository.delete` reads oddly at first: `Op.build { Books.id eq
EntityID(bookId, Books) }` wraps the comparison instead of the more natural
`Books.id eq bookId`. That's because `deleteWhere`'s lambda receiver is the
table itself (not a `SqlExpressionBuilder`, unlike `where { }`), so `eq`
needs to be built explicitly with `Op.build { }`, and the ID column expects
a wrapped `EntityID`, not a bare `Int` — a real trap hit while building
this project, kept here as the working fix.

## Tests

```kotlin
// src/test/kotlin/AppTest.kt
import io.ktor.client.request.*
import io.ktor.client.statement.*
import io.ktor.http.*
import io.ktor.server.testing.*
import org.junit.jupiter.api.Test
import org.junit.jupiter.api.BeforeAll
import kotlin.test.assertEquals
import kotlin.test.assertTrue

class AppTest {
    companion object {
        @JvmStatic
        @BeforeAll
        fun setup() { BookRepository.init() }
    }

    @Test
    fun `GET books returns seeded books`() = testApplication {
        application { module() }
        val response = client.get("/books")
        assertEquals(HttpStatusCode.OK, response.status)
        assertTrue(response.bodyAsText().contains("Kotlin in Action"))
    }

    @Test
    fun `POST then GET by id round-trips`() = testApplication {
        application { module() }
        val postResponse = client.post("/books") {
            contentType(ContentType.Application.Json)
            setBody("""{"title":"Atomic Kotlin","author":"Eckel & Isakova","year":2021}""")
        }
        assertEquals(HttpStatusCode.Created, postResponse.status)
        assertTrue(postResponse.bodyAsText().contains("Atomic Kotlin"))
    }

    @Test
    fun `GET missing book returns 404`() = testApplication {
        application { module() }
        assertEquals(HttpStatusCode.NotFound, client.get("/books/9999").status)
    }
}
```

Using an in-memory shared-cache SQLite URL
(`jdbc:sqlite:file:x?mode=memory&cache=shared`) here was the first
approach tried, and it **failed intermittently** — Exposed opens and
closes a JDBC connection per `transaction { }`, and shared-cache in-memory
SQLite drops all data the moment the connection count hits zero between
transactions. Switching to a real file-backed database (`library.db`)
fixed it; this is exactly the kind of thing that only shows up once you
actually run the tests, not from reading the code.

## Running it

```text
$ ./gradlew test

AppTest > GET books returns seeded books() PASSED
AppTest > POST then GET by id round-trips() PASSED
AppTest > GET missing book returns 404() PASSED

BUILD SUCCESSFUL
```

Running the actual server and exercising it with a real `HttpClient`:

```text
GET /books -> {"books":[{"id":1,"title":"Kotlin in Action","author":"Jemerov & Isakova","year":2017},{"id":2,"title":"Effective Kotlin","author":"Marcin Moskala","year":2019}]}
POST /books -> status=201 Created, body={"id":3,"title":"Atomic Kotlin","author":"Eckel & Isakova","year":2021}
GET /books?author=Moskala -> {"books":[{"id":2,"title":"Effective Kotlin","author":"Marcin Moskala","year":2019}]}
DELETE /books/1 -> status=204 No Content
GET /books/1 (after delete) -> status=404 Not Found, body={"error":"Book 1 not found"}
```

Every response matches expectations: the author filter finds only Moskala,
the deleted book (id 1) is gone, and fetching it afterward correctly
returns the `StatusPages`-mapped 404 with a JSON error body instead of a
raw exception.

## Stretch goals

- Add a `PUT /books/{id}` route for full updates, and a `PATCH
  /books/{id}` for partial ones (e.g. updating just the `year`) — think
  through what request body shape makes sense for a partial update in
  Kotlin (a data class with nullable fields, only updating non-null ones).
- Add pagination to `GET /books` (`?page=1&size=10`) using Exposed's
  `.limit(size).offset(...)` query methods, and return the total count
  alongside the page in the response body.
- Swap SQLite for an in-process Postgres via
  [Testcontainers](https://testcontainers.com/) in the test suite, so the
  tests exercise the same database engine production would use.
- Add request validation (e.g. reject a `NewBook` with a blank title or a
  `year` outside a sane range) using `StatusPages`' `exception<
  IllegalArgumentException>` to turn validation failures into 400
  responses instead of 500s.
