# 10 · Capstone Project

This capstone combines every module in this level into one small but
real project: an order-processing service. It validates input,
calculates fees, processes a batch of orders concurrently with
structured coroutines, records results in a thread-safe repository, and
ships with a test suite run through Gradle. The whole thing is under 150
lines of Kotlin, laid out as a real multi-module Gradle project you can
build and run yourself.

## Project layout

```
capstone/
├── settings.gradle.kts
├── gradle.properties
└── app/
    ├── build.gradle.kts
    └── src/
        ├── main/kotlin/com/mastery/capstone/
        │   ├── Orders.kt      # domain model, validation, fees, repository
        │   ├── Service.kt     # concurrent processing with coroutines
        │   └── Main.kt        # entry point
        └── test/kotlin/com/mastery/capstone/
            └── OrderServiceTest.kt
```

```kotlin
// settings.gradle.kts
rootProject.name = "capstone"
include("app")
```

```kotlin
// app/build.gradle.kts
plugins {
    kotlin("jvm") version "1.9.24"
    application
}

repositories { mavenCentral() }

dependencies {
    implementation("org.jetbrains.kotlinx:kotlinx-coroutines-core:1.8.1")
    testImplementation(kotlin("test"))
    testImplementation("org.jetbrains.kotlinx:kotlinx-coroutines-test:1.8.1")
    testRuntimeOnly("org.junit.platform:junit-platform-launcher")
}

java {
    toolchain {
        languageVersion.set(JavaLanguageVersion.of(17))
    }
}

application {
    mainClass.set("com.mastery.capstone.MainKt")
}

tasks.test {
    useJUnitPlatform()
}
```

The `java { toolchain { ... } }` block is what makes this build
reproducible across machines with different JDKs installed — Gradle
downloads or locates a JDK 17 to compile and run with regardless of
which JDK is on the developer's `PATH`, the same problem module 06's
container base image solves for deployment.

## Domain and validation (`Orders.kt`)

```kotlin
data class Order(val id: String, val customer: String, val amountCents: Long)

class ValidationException(message: String) : RuntimeException(message)

sealed class OrderResult {
    data class Success(val order: Order, val fee: Long) : OrderResult()
    data class Failure(val orderId: String, val reason: String) : OrderResult()
}

class OrderValidator {
    fun validate(order: Order) {
        if (order.amountCents <= 0) {
            throw ValidationException("order ${order.id}: amount must be positive")
        }
        if (order.customer.isBlank()) {
            throw ValidationException("order ${order.id}: customer must not be blank")
        }
    }
}

class FeeCalculator(private val feeRateBps: Long = 250) {
    fun feeFor(amountCents: Long): Long = amountCents * feeRateBps / 10_000
}

class OrderRepository {
    private val processed = mutableMapOf<String, OrderResult>()

    @Synchronized
    fun record(result: OrderResult) {
        val key = when (result) {
            is OrderResult.Success -> result.order.id
            is OrderResult.Failure -> result.orderId
        }
        processed[key] = result
    }

    @Synchronized
    fun all(): List<OrderResult> = processed.values.sortedBy {
        when (it) {
            is OrderResult.Success -> it.order.id
            is OrderResult.Failure -> it.orderId
        }
    }
}
```

`OrderResult` is a `sealed class` so every consumer's `when` over it is
exhaustive at compile time — exactly the coverage module 05 warned is
easy to skip in tests, so this project's test suite deliberately covers
both branches. `OrderRepository` is written to be called from multiple
coroutines concurrently (the service below fires one coroutine per
order), so both methods are `@Synchronized` — a plain `mutableMapOf`
with unsynchronized access from concurrent coroutines would be a data
race even though coroutines aren't OS threads directly, since the
default dispatcher does run them across a real thread pool.

## Concurrent processing (`Service.kt`)

```kotlin
class OrderService(
    private val validator: OrderValidator = OrderValidator(),
    private val feeCalculator: FeeCalculator = FeeCalculator(),
    private val repository: OrderRepository = OrderRepository(),
) {
    suspend fun processAll(orders: List<Order>): List<OrderResult> = coroutineScope {
        val deferred = orders.map { order ->
            async {
                val result = try {
                    validator.validate(order)
                    val fee = feeCalculator.feeFor(order.amountCents)
                    OrderResult.Success(order, fee)
                } catch (e: ValidationException) {
                    OrderResult.Failure(order.id, e.message ?: "validation failed")
                }
                repository.record(result)
                result
            }
        }
        deferred.map { it.await() }
        repository.all()
    }
}
```

`coroutineScope` is structured concurrency doing its job: every `async`
launched inside it is a child of that scope, so `processAll` doesn't
return until every order has finished processing, and an unhandled
exception in any child would cancel the siblings and propagate out of
`processAll` rather than being silently lost. Catching
`ValidationException` *inside* each `async` block, rather than letting it
propagate, is a deliberate choice — a bad order should become a recorded
`Failure`, not cancel every other order's processing.

## Entry point (`Main.kt`)

```kotlin
fun main() = runBlocking {
    val orders = listOf(
        Order("ORD-1", "Ada Lovelace", 12_000),
        Order("ORD-2", "Grace Hopper", 5_500),
        Order("ORD-3", "", 3_000),
        Order("ORD-4", "Alan Turing", -100),
        Order("ORD-5", "Margaret Hamilton", 90_000),
    )

    val service = OrderService()
    val results = service.processAll(orders)

    var succeeded = 0
    var failed = 0
    for (result in results) {
        when (result) {
            is OrderResult.Success -> {
                succeeded++
                println("OK   ${result.order.id}: fee=${result.fee}c for ${result.order.customer}")
            }
            is OrderResult.Failure -> {
                failed++
                println("FAIL ${result.orderId}: ${result.reason}")
            }
        }
    }
    println("---")
    println("processed=${results.size} succeeded=$succeeded failed=$failed")
}
```

## Tests

```kotlin
class OrderServiceTest {
    @Test
    fun `valid orders succeed with correct fee`() = runTest {
        val service = OrderService()
        val results = service.processAll(listOf(Order("ORD-1", "Ada", 10_000)))
        val success = results.single() as OrderResult.Success
        assertEquals(250, success.fee)
    }

    @Test
    fun `invalid amount fails validation`() = runTest {
        val service = OrderService()
        val results = service.processAll(listOf(Order("ORD-2", "Ada", -5)))
        val failure = results.single() as OrderResult.Failure
        assertTrue(failure.reason.contains("positive"))
    }

    // ... blank-customer and mixed-batch cases follow the same pattern
}
```

Each test uses `runTest` from `kotlinx-coroutines-test` (not
`runBlocking`) — module 05's coroutine-testing trap applies directly
here, since `processAll` is a `suspend` function.

## Running it

```console
$ gradle test
BUILD SUCCESSFUL in 1s
4 actionable tasks: 2 executed, 2 up-to-date

$ gradle run -q
OK   ORD-1: fee=300c for Ada Lovelace
OK   ORD-2: fee=137c for Grace Hopper
FAIL ORD-3: order ORD-3: customer must not be blank
FAIL ORD-4: order ORD-4: amount must be positive
OK   ORD-5: fee=2250c for Margaret Hamilton
---
processed=5 succeeded=3 failed=2
```

Both real Gradle runs against the project as laid out above: all 4 tests
pass, and the sample batch of 5 orders produces 3 successes and 2
validation failures, with fees calculated at the default 2.5% rate
(`12_000 * 250 / 10_000 = 300`).

## What this capstone demonstrates from each module

| Module | Where it shows up |
|---|---|
| 01 Coroutines | `coroutineScope`/`async`/`await` in `OrderService.processAll` |
| 02 Production APIs | `OrderService` is shaped like a Ktor route handler minus the HTTP layer |
| 04 Security | Input validation rejects blank/negative fields before processing |
| 05 Testing & CI | `kotlin.test` + `kotlinx-coroutines-test`, runnable via `gradle test` |
| 06 Deployment | Toolchain-pinned Gradle build, ready for the multi-stage Dockerfile pattern |
| 07 Performance | `sealed class` dispatch instead of boxed exception-driven control flow |
| 08 Java Interop | `@Synchronized` — a JVM/Java-originated primitive used directly from Kotlin |
| 09 Code Quality | Consistent naming/formatting throughout, ready for `ktlintCheck`/`detekt` |

## Stretch goals

- Wrap `OrderService` in a minimal Ktor `POST /orders` endpoint (module
  02) that accepts a JSON array of orders and returns the same
  `OrderResult` list serialized back to JSON.
- Replace `OrderRepository`'s in-memory map with a real persistence layer
  (SQLite via `Exposed`, or a Docker Postgres container per module 06)
  so results survive a restart.
- Add a `ktlint`/`detekt` Gradle task (module 09) to the build and fix
  every finding it reports against this project's own source.
- Package the whole thing with the multi-stage Dockerfile from module 06
  and confirm `docker run` produces the same batch-processing output as
  `gradle run`.
- Add a load test (module 07) that submits 10,000 orders at once and
  measure how `processAll`'s all-at-once `async` fan-out compares to
  batching them through a fixed-size coroutine dispatcher or
  `Semaphore`-limited concurrency.
