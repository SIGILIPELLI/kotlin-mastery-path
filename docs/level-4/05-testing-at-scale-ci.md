# 05 · Testing at Scale & CI

A handful of tests in one file is easy to keep green. Hundreds of tests
across dozens of files, run on every push, need structure: clear
pass/fail reporting, tests organized by what they exercise, and a CI
pipeline that fails the build the moment something breaks. This module
builds a small banking domain, a JUnit5-shaped test suite for it (using a
minimal hand-rolled runner so every example stays self-contained and
runnable with only `kotlinc`), and a GitHub Actions workflow that runs it
on every push.

```kotlin
dependencies {
    testImplementation("org.junit.jupiter:junit-jupiter:5.10.2")
    testImplementation("io.kotest:kotest-assertions-core:5.9.0")
}
```

## The code under test

```kotlin
class InsufficientFundsException(message: String) : RuntimeException(message)

class Account(val id: String, initialBalance: Long) {
    var balance: Long = initialBalance
        private set

    fun deposit(amount: Long) {
        require(amount > 0) { "deposit amount must be positive" }
        balance += amount
    }

    fun withdraw(amount: Long) {
        require(amount > 0) { "withdraw amount must be positive" }
        if (amount > balance) throw InsufficientFundsException("balance $balance < requested $amount")
        balance -= amount
    }
}

class TransferService(private val fee: Long = 10) {
    fun transfer(from: Account, to: Account, amount: Long) {
        from.withdraw(amount + fee)
        to.deposit(amount)
    }
}
```

In a real project this is `src/main/kotlin/Account.kt`; tests live in the
parallel `src/test/kotlin/` source set that Gradle's `kotlin("jvm")`
plugin wires up automatically, and `./gradlew test` compiles and runs
only that tree.

## Structuring the tests

With real JUnit5, each of these would be a `@Test fun` inside a class,
using `assertEquals`/`assertThrows` from `org.junit.jupiter.api.Assertions`
or Kotest's fluent matchers. The shape below — named cases collected into
a list, each running independently and reporting its own pass/fail — is
exactly what those frameworks do under the hood, kept dependency-free so
it compiles with plain `kotlinc`.

```kotlin
class TestFailure(message: String) : AssertionError(message)

fun assertEquals(expected: Any?, actual: Any?, label: String) {
    if (expected != actual) throw TestFailure("$label: expected=$expected actual=$actual")
}

fun assertThrows(label: String, block: () -> Unit) {
    try {
        block()
        throw TestFailure("$label: expected an exception, none was thrown")
    } catch (e: TestFailure) {
        throw e
    } catch (e: Exception) {
        // expected
    }
}

data class TestCase(val name: String, val run: () -> Unit)

val tests = listOf(
    TestCase("deposit increases balance") {
        val acc = Account("A1", 100)
        acc.deposit(50)
        assertEquals(150L, acc.balance, "balance after deposit")
    },
    TestCase("withdraw decreases balance") {
        val acc = Account("A2", 100)
        acc.withdraw(30)
        assertEquals(70L, acc.balance, "balance after withdraw")
    },
    TestCase("withdraw beyond balance throws") {
        val acc = Account("A3", 50)
        assertThrows("overdraft") { acc.withdraw(51) }
    },
    TestCase("transfer moves amount and deducts fee from sender") {
        val a = Account("A4", 1000)
        val b = Account("A5", 0)
        TransferService(fee = 10).transfer(a, b, 200)
        assertEquals(790L, a.balance, "sender balance")
        assertEquals(200L, b.balance, "receiver balance")
    },
    TestCase("transfer fails loudly when sender lacks funds for amount+fee") {
        val a = Account("A6", 100)
        val b = Account("A7", 0)
        assertThrows("insufficient for fee") { TransferService(fee = 10).transfer(a, b, 100) }
    },
    // Deliberately wrong, to show what a real CI failure looks like below.
    TestCase("intentionally wrong expectation") {
        val acc = Account("A8", 10)
        acc.deposit(5)
        assertEquals(999L, acc.balance, "balance after deposit (bad expectation)")
    },
)
```

## Running the suite and reporting results

```kotlin
fun main() {
    var passed = 0
    var failed = 0
    for (t in tests) {
        try {
            t.run()
            println("PASS  ${t.name}")
            passed++
        } catch (e: Throwable) {
            println("FAIL  ${t.name} -- ${e.message}")
            failed++
        }
    }
    println("---")
    println("$passed passed, $failed failed, ${tests.size} total")
    if (failed > 0) kotlin.system.exitProcess(1)
}
```

```text
PASS  deposit increases balance
PASS  withdraw decreases balance
PASS  withdraw beyond balance throws
PASS  transfer moves amount and deducts fee from sender
PASS  transfer fails loudly when sender lacks funds for amount+fee
FAIL  intentionally wrong expectation -- balance after deposit (bad expectation): expected=999 actual=15
---
5 passed, 1 failed, 6 total
```

The process also exits with code `1` when any test fails — this is the
detail that makes CI actually work: `./gradlew test` (or this compiled
jar) failing means the *shell command* failed, which is what GitHub
Actions checks to mark a workflow run red. A test runner that always
prints a summary but exits `0` regardless would never fail a build no
matter how many tests failed.

## Wiring it into GitHub Actions

```yaml
# .github/workflows/ci.yml
name: CI
on:
  push:
    branches: [main]
  pull_request:

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: '21'
      - name: Run tests
        run: ./gradlew test --no-daemon
      - name: Upload test report
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: test-results
          path: build/test-results/test/
```

`if: always()` on the upload step is what makes failed-run debugging
possible — without it, the artifact upload is skipped the moment `test`
fails (the default), so the very run you need the report for is the one
that never uploads it.

## Kotlin-specific traps

- **A test that only exercises the "happy path" through a `sealed
  class`/`when` misses the exhaustiveness guarantee entirely.** The
  compiler enforces you *handle* every subtype in a `when`; it does not
  enforce you *test* every subtype. Structure test cases per subtype
  deliberately (as this module tests both the success and
  `InsufficientFundsException` paths of `withdraw`).
- **`@BeforeEach` (JUnit5) creates a fresh instance per test *unless*
  `@TestInstance(Lifecycle.PER_CLASS)` is used** — Kotlin test classes
  default to JUnit's per-method instantiation, so shared `var` state
  declared as a class property resets between tests unless you opt into
  per-class lifecycle explicitly, which surprises developers coming from
  languages where classes are more commonly reused across tests.
- **Extension functions used only in tests still need a real receiver.**
  A test helper like `fun Account.creditForTest(amt: Long)` compiles fine
  but silently does nothing useful if called on a mock/fake that doesn't
  share the real class's invariants — prefer testing through the public
  API (`deposit`/`withdraw`) over reaching around it.
- **Coroutine code needs `kotlinx-coroutines-test` and `runTest`, not
  `runBlocking`, in tests** — `runTest` uses a virtual clock, so a
  `delay(10_000)` inside code under test completes instantly instead of
  making the test suite actually wait ten seconds; forgetting this
  dependency and using `runBlocking` for coroutine tests is a common
  source of a CI suite that "works" but takes minutes longer than it
  should.
- **Flaky tests fail differently on CI than locally** because CI runners
  are typically slower and more resource-constrained — a test relying on
  wall-clock timing (`Thread.sleep` races) rather than deterministic
  synchronization (channels, `join()`, `runTest`'s virtual time) is the
  most common source of a suite that's green on a laptop and red in
  Actions.

## Cheat sheet

| Concern | Approach |
|---|---|
| Assert equality | `assertEquals`/Kotest matchers, not `println` + eyeballing |
| Assert an exception is thrown | `assertThrows<T> { }` |
| Coroutine tests | `runTest` from `kotlinx-coroutines-test`, not `runBlocking` |
| Fail the build on failing tests | Ensure the test process exits non-zero |
| CI trigger | `on: push`/`pull_request` in `.github/workflows/*.yml` |
| Debugging a red CI run | `if: always()` on an artifact-upload step for reports |

## Exercise

Add a seventh `TestCase` that verifies `deposit` throws when given a
negative amount (`acc.deposit(-5)`), using `assertThrows`. Run the suite
again and confirm the total goes from `6` to `7` while the pass count for
that new case is correct — then fix the deliberately-wrong test from this
module (change its expectation to `15L`) and confirm the suite reports
`0 failed`.
