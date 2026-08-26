# 06 · Testing Advanced

[Level 2's testing module](../level-2/06-testing-junit-kotest.md) covered
plain JUnit 5 and Kotest fundamentals. Real services need more: mocking
dependencies you don't want to hit for real (payment gateways, external
APIs), testing `suspend` functions without real delays, and running the
same test logic against many inputs. This module covers **MockK** (a
Kotlin-native mocking library), `kotlinx-coroutines-test`, and
`@ParameterizedTest`.

```kotlin
dependencies {
    testImplementation(kotlin("test"))
    testImplementation("org.junit.jupiter:junit-jupiter:5.10.2")
    testImplementation("io.mockk:mockk:1.13.11")
    testImplementation("org.jetbrains.kotlinx:kotlinx-coroutines-test:1.8.1")
    implementation("org.jetbrains.kotlinx:kotlinx-coroutines-core:1.8.1")
}
```

## The code under test

```kotlin
interface PaymentGateway {
    suspend fun charge(cents: Int): Boolean
}

class OrderService(private val gateway: PaymentGateway) {
    suspend fun placeOrder(amountCents: Int): String {
        if (amountCents <= 0) throw IllegalArgumentException("amount must be positive")
        return if (gateway.charge(amountCents)) "confirmed" else "declined"
    }
}

fun classify(n: Int): String = when {
    n < 0 -> "negative"
    n == 0 -> "zero"
    else -> "positive"
}
```

`OrderService` depends on a `PaymentGateway` interface rather than a
concrete implementation specifically so tests can substitute a fake —
this is the whole reason to prefer constructor-injected interfaces over
`object`-style singletons for anything that talks to the outside world.

## Mocking with MockK, and testing coroutines with `runTest`

MockK's `mockk<T>()` creates a fake implementation; `coEvery { }`
stubs a `suspend` function's return value (`every { }` for non-suspend
ones). `runTest` (from `kotlinx-coroutines-test`) runs a test body as a
coroutine and — crucially — auto-advances virtual time, so a real
`delay(5000)` inside the code under test doesn't actually make the test
wait five seconds.

```kotlin
import io.mockk.coEvery
import io.mockk.mockk
import kotlinx.coroutines.test.runTest
import org.junit.jupiter.api.Test
import kotlin.test.assertEquals
import kotlin.test.assertFailsWith

class OrderServiceTest {

    @Test
    fun `charge succeeds returns confirmed`() = runTest {
        val gateway = mockk<PaymentGateway>()
        coEvery { gateway.charge(500) } returns true
        val service = OrderService(gateway)

        val result = service.placeOrder(500)

        assertEquals("confirmed", result)
    }

    @Test
    fun `charge declined returns declined`() = runTest {
        val gateway = mockk<PaymentGateway>()
        coEvery { gateway.charge(any()) } returns false
        val service = OrderService(gateway)

        assertEquals("declined", service.placeOrder(100))
    }

    @Test
    fun `non-positive amount throws before touching the gateway`() = runTest {
        val gateway = mockk<PaymentGateway>()
        val service = OrderService(gateway)

        assertFailsWith<IllegalArgumentException> { service.placeOrder(0) }
    }
}
```

Note the third test never stubs `gateway.charge` at all — because the
`IllegalArgumentException` is thrown before the gateway is touched, MockK
never needs a stub for it. If `placeOrder`'s validation were removed, this
test would fail with a MockK "no answer found" error rather than silently
passing, which is a nice guardrail: it proves the short-circuit actually
happens.

## Parameterized tests

`@ParameterizedTest` with `@CsvSource` runs one test method against many
inputs, reporting each as a separate result.

```kotlin
import org.junit.jupiter.params.ParameterizedTest
import org.junit.jupiter.params.provider.CsvSource
import kotlin.test.assertEquals

class ClassifyTest {
    @ParameterizedTest(name = "classify({0}) = {1}")
    @CsvSource("-5,negative", "0,zero", "7,positive")
    fun `classify handles all three ranges`(input: Int, expected: String) {
        assertEquals(expected, classify(input))
    }
}
```

Running the whole suite (`./gradlew test`) with `testLogging { events(...)
}` configured to print results:

```text
OrderServiceTest > charge succeeds returns confirmed() PASSED

OrderServiceTest > charge declined returns declined() PASSED

OrderServiceTest > non-positive amount throws before touching the gateway() PASSED

OrderServiceTest > classify handles all three ranges(int, String) > classify(-5) = negative PASSED

OrderServiceTest > classify handles all three ranges(int, String) > classify(0) = zero PASSED

OrderServiceTest > classify handles all three ranges(int, String) > classify(7) = positive PASSED

BUILD SUCCESSFUL
```

## Kotlin-specific traps

- **`every { }` vs. `coEvery { }`.** Stubbing a `suspend` function with
  plain `every { }` fails to compile (or, with some MockK versions,
  compiles but never matches) — `suspend` functions always need the `co`-
  prefixed MockK variants (`coEvery`, `coVerify`).
- **`runTest` isn't just "run this in `runBlocking`."** It uses a virtual
  clock, so `delay()` calls inside the code under test complete instantly
  in test time — a test with a real 5-second `delay` finishes in
  milliseconds. Don't be alarmed if your test timing assertions need to
  account for this.
- **MockK mocks are strict by default about unstubbed calls.** Calling a
  method you never stubbed with `every`/`coEvery` throws at test time
  rather than silently returning `null` — this is a feature (catches
  tests that don't actually test what you think), but surprises people
  used to more permissive mocking frameworks.
- **`assertFailsWith<T>` needs the exact (or a supertype) exception
  type.** `assertFailsWith<IllegalArgumentException> { ... }` won't match
  if the code actually throws a *subtype-unrelated* exception like
  `IllegalStateException` — read the failure message carefully, it names
  the exception that was actually thrown.
- **`@CsvSource` values are always parsed as strings first**, then
  converted to the parameter's declared type — a malformed row (wrong
  column count, non-numeric value for an `Int` parameter) fails at
  runtime with a `CsvSource`-specific error, not a compile error.

## Cheat sheet

| Need | Tool |
|---|---|
| Fake an interface | `mockk<T>()` |
| Stub a non-suspend function | `every { mock.fn() } returns value` |
| Stub a suspend function | `coEvery { mock.fn() } returns value` |
| Verify a call happened | `verify { mock.fn() }` / `coVerify { }` |
| Run suspend test code | `= runTest { ... }` |
| Assert an exception is thrown | `assertFailsWith<E> { ... }` |
| Run one test over many inputs | `@ParameterizedTest` + `@CsvSource`/`@ValueSource` |

## Exercise

Add a `NotificationService` interface with a `suspend fun notify(orderId:
String): Unit` method, inject it into `OrderService`, and call it only
when an order is `"confirmed"`. Write a MockK test using `coVerify` to
assert `notify` was called exactly once on success, and a second test
using `coVerify(exactly = 0)` to assert it's never called when the charge
is declined.
