# 06 · Testing with JUnit/Kotest

Untested Kotlin code is a liability the moment more than one person touches
it. This module covers the two most common testing setups in the Kotlin
ecosystem: **JUnit 5**, the JVM-wide standard (and what Kotlin projects
default to), and **Kotest**, a Kotlin-native framework with a more
expressive, idiomatic style. Both are wired into a project via
[Gradle](08-gradle-basics.md), and [Module 10](10-project-weather-cli.md)
uses JUnit to test the capstone project.

## JUnit 5 basics

A JUnit test is a function annotated `@Test` inside a class. Assertions
come from `org.junit.jupiter.api.Assertions` (or Kotlin-friendlier
top-level equivalents).

```kotlin
import org.junit.jupiter.api.Test
import org.junit.jupiter.api.Assertions.assertEquals
import org.junit.jupiter.api.Assertions.assertTrue

class Calculator {
    fun add(a: Int, b: Int) = a + b
    fun isEven(n: Int) = n % 2 == 0
}

class CalculatorTest {
    private val calculator = Calculator()

    @Test
    fun `add returns the sum of two numbers`() {
        assertEquals(5, calculator.add(2, 3))
    }

    @Test
    fun `isEven correctly identifies even numbers`() {
        assertTrue(calculator.isEven(4))
    }
}
```

Kotlin lets you write test method names as backtick-quoted strings with
spaces (`` `add returns the sum of two numbers`() ``) — test reports show
the readable sentence instead of a cramped `camelCase` name.

## Setup and teardown with `@BeforeEach`/`@AfterEach`

Shared setup that needs to run fresh before every test goes in a
`@BeforeEach` method — this avoids one test's leftover state leaking into
another.

```kotlin
import org.junit.jupiter.api.BeforeEach
import org.junit.jupiter.api.Test
import org.junit.jupiter.api.Assertions.assertEquals

class ShoppingCart {
    private val items = mutableListOf<String>()
    fun add(item: String) = items.add(item)
    fun itemCount() = items.size
}

class ShoppingCartTest {
    private lateinit var cart: ShoppingCart

    @BeforeEach
    fun setUp() {
        cart = ShoppingCart()   // a brand-new cart before every single test method
    }

    @Test
    fun `new cart starts empty`() {
        assertEquals(0, cart.itemCount())
    }

    @Test
    fun `adding an item increases the count`() {
        cart.add("apple")
        assertEquals(1, cart.itemCount())
    }
}
```

## Testing exceptions

`assertThrows` runs a block and asserts it throws the expected exception
type, then gives you the caught exception to make further assertions on.

```kotlin
import org.junit.jupiter.api.Test
import org.junit.jupiter.api.Assertions.assertThrows
import org.junit.jupiter.api.Assertions.assertEquals

class Account(private var balance: Int) {
    fun withdraw(amount: Int) {
        check(amount <= balance) { "Insufficient funds" }
        balance -= amount
    }
}

class AccountTest {
    @Test
    fun `withdrawing more than the balance throws`() {
        val account = Account(50)
        val exception = assertThrows(IllegalStateException::class.java) {
            account.withdraw(100)
        }
        assertEquals("Insufficient funds", exception.message)
    }
}
```

## Parameterized tests

Instead of copy-pasting near-identical test methods, `@ParameterizedTest`
runs the same test body once per supplied value.

```kotlin
import org.junit.jupiter.params.ParameterizedTest
import org.junit.jupiter.params.provider.ValueSource
import org.junit.jupiter.api.Assertions.assertTrue

class PositivityTest {
    @ParameterizedTest
    @ValueSource(ints = [1, 2, 100, 9999])
    fun `these numbers are all positive`(value: Int) {
        assertTrue(value > 0)
    }
}
```

## Kotest: a Kotlin-native alternative

Kotest reads more like natural language and comes with a large library of
`shouldBe`-style matchers. `StringSpec` is its simplest style — each test
is just a string description plus a lambda.

```kotlin
import io.kotest.core.spec.style.StringSpec
import io.kotest.matchers.shouldBe
import io.kotest.matchers.collections.shouldContain

class CalculatorSpec : StringSpec({
    "adding two numbers should return their sum" {
        val calculator = Calculator()
        calculator.add(2, 3) shouldBe 5
    }

    "a list of fruits should contain banana" {
        val fruits = listOf("apple", "banana", "cherry")
        fruits shouldContain "banana"
    }
})
```

`FunSpec` is closer to plain JUnit-style naming if you prefer `test(...)`
blocks over string-literal test names:

```kotlin
import io.kotest.core.spec.style.FunSpec
import io.kotest.matchers.shouldBe
import io.kotest.assertions.throwables.shouldThrow

class AccountSpec : FunSpec({
    test("withdrawing more than the balance throws") {
        val account = Account(50)
        shouldThrow<IllegalStateException> {
            account.withdraw(100)
        }
    }

    test("add returns the sum") {
        Calculator().add(2, 3) shouldBe 5
    }
})
```

## Testing suspend functions

A `suspend` function can't be called directly from a normal `@Test`
method. `kotlinx-coroutines-test`'s `runTest` gives you a coroutine scope
built for tests — it also runs any `delay()` calls instantly instead of
actually waiting, so tests using [Module 3's](03-coroutines-basics.md)
coroutines stay fast.

```kotlin
import kotlinx.coroutines.test.runTest
import kotlinx.coroutines.delay
import org.junit.jupiter.api.Test
import org.junit.jupiter.api.Assertions.assertEquals

suspend fun fetchAnswer(): Int {
    delay(1000)   // in a real test, this doesn't actually wait a full second
    return 42
}

class AsyncTest {
    @Test
    fun `fetchAnswer eventually returns 42`() = runTest {
        val answer = fetchAnswer()
        assertEquals(42, answer)
    }
}
```

## JUnit vs. Kotest

| | JUnit 5 | Kotest |
|---|---|---|
| Style | Annotated methods (`@Test`) | Spec classes (`StringSpec`, `FunSpec`, ...) |
| Assertions | `assertEquals(expected, actual)` | `actual shouldBe expected` |
| Ecosystem | JVM-wide standard, huge tooling support | Kotlin-only, very expressive matchers |
| Parameterized tests | `@ParameterizedTest` + `@ValueSource`/`@CsvSource` | Built-in data-driven testing (`forAll`, table tests) |
| Good default when | Working across Java+Kotlin, or team already knows JUnit | Kotlin-only codebase wanting more readable specs |

## Cheat sheet

| Need | JUnit 5 | Kotest |
|---|---|---|
| Mark a test | `@Test` | A string literal or `test("...")` block |
| Run before each test | `@BeforeEach` | `beforeTest { }` |
| Assert equality | `assertEquals(expected, actual)` | `actual shouldBe expected` |
| Assert an exception | `assertThrows(Type::class.java) { }` | `shouldThrow<Type> { }` |
| Parameterize | `@ParameterizedTest` + `@ValueSource` | `withData(...)` / table tests |
| Test a `suspend fun` | `runTest { }` (from `kotlinx-coroutines-test`) | same `runTest { }` works inside a spec |

## Exercise

Write a small `PasswordValidator` class with a function `isValid(password:
String): Boolean` that requires at least 8 characters and at least one
digit. Then write a JUnit 5 test class with: one test for a valid password,
one for a too-short password, one for a password with no digit, and a
`@ParameterizedTest` with `@ValueSource` covering several more valid
passwords at once.
