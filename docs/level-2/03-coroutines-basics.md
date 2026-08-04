# 03 · Coroutines Basics

Coroutines are Kotlin's answer to asynchronous, non-blocking code:
lightweight "tasks" that can suspend and resume without blocking an
operating-system thread. Where a blocked thread just sits there burning a
stack and a scheduler slot, a *suspended* coroutine gives the thread back
entirely — you can run hundreds of thousands of coroutines on a handful of
threads. This module covers the building blocks; [Module 10](10-project-weather-cli.md)
puts them to work fetching real data over the network.

Coroutines live in the `kotlinx.coroutines` library (not the standard
library proper), added to a project via Gradle — see
[Module 8](08-gradle-basics.md) for how that dependency is declared.

## `suspend` functions

A function marked `suspend` can pause its execution at certain points
(usually while waiting on I/O) and resume later, without blocking the
thread it started on. You can only call a `suspend` function from inside
another `suspend` function or a coroutine.

```kotlin
import kotlinx.coroutines.*

suspend fun fetchGreeting(): String {
    delay(100)              // suspends -- does NOT block the thread, unlike Thread.sleep
    return "Hello from a coroutine"
}

fun main() = runBlocking {   // bridges the non-suspend `main` into coroutine world
    val greeting = fetchGreeting()
    println(greeting)
}
```

```text
Hello from a coroutine
```

`runBlocking` is mainly for entry points like `main` or tests — it blocks
the current thread until the coroutines inside it finish. Real application
code launches coroutines from an existing scope rather than starting a new
blocking one.

## `launch`: fire-and-forget coroutines

`launch` starts a new coroutine that runs concurrently and returns a `Job`
you can use to wait for it or cancel it. It doesn't return a value.

```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking {
    println("Start")

    val job = launch {
        delay(200)
        println("Task from launch")
    }

    println("End of main (task is still running)")
    job.join()   // wait for the launched coroutine to finish
    println("Task finished")
}
```

```text
Start
End of main (task is still running)
Task from launch
Task finished
```

Notice the order: `launch` doesn't block — "End of main" prints before the
delayed coroutine finishes. `job.join()` is what actually waits for it.

## `async`/`await`: concurrent results

`async` is like `launch` but returns a `Deferred<T>` — a promise of a
future value you retrieve with `.await()`. Launching several `async` blocks
before awaiting any of them runs them *concurrently*, not sequentially.

```kotlin
import kotlinx.coroutines.*
import kotlin.system.measureTimeMillis

suspend fun fetchUser(): String {
    delay(500)
    return "Alice"
}

suspend fun fetchOrders(): Int {
    delay(500)
    return 3
}

fun main() = runBlocking {
    val time = measureTimeMillis {
        val userDeferred = async { fetchUser() }     // starts immediately
        val ordersDeferred = async { fetchOrders() }  // also starts immediately, in parallel

        val user = userDeferred.await()
        val orders = ordersDeferred.await()
        println("$user has $orders orders")
    }
    println("Took about ${time}ms")   // ~500ms, not ~1000ms -- they ran concurrently
}
```

```text
Alice has 3 orders
Took about 512ms
```

If you called `fetchUser()` and `fetchOrders()` with plain sequential
`suspend` calls (no `async`), the total time would be roughly the *sum* of
both delays (~1000ms) instead of running side by side.

## Structured concurrency: `coroutineScope`

`coroutineScope` creates a scope that waits for *all* coroutines launched
inside it to finish before it returns — including if one of them throws,
in which case the others are cancelled automatically. This is "structured
concurrency": child coroutines can never outlive the scope that created
them, so you can't accidentally leak a background task.

```kotlin
import kotlinx.coroutines.*

suspend fun loadDashboard(): String = coroutineScope {
    val profile = async { delay(100); "Profile" }
    val feed = async { delay(150); "Feed" }
    "${profile.await()} + ${feed.await()} loaded"   // coroutineScope returns once both finish
}

fun main() = runBlocking {
    println(loadDashboard())
}
```

```text
Profile + Feed loaded
```

## Cancellation is cooperative

Cancelling a `Job` doesn't forcibly kill a thread — it sets a flag that
*well-behaved* coroutine code checks. Suspending functions like `delay()`
check it automatically and throw `CancellationException`, but tight loops
of plain computation (no suspension points) won't notice a cancellation at
all unless you check `isActive` or call `yield()` yourself.

```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking {
    val job = launch(Dispatchers.Default) {
        var i = 0
        while (isActive) {          // cooperative check -- exits promptly once cancelled
            i++
        }
        println("Cancelled cleanly after $i iterations")
    }
    delay(50)
    job.cancelAndJoin()
    println("Done")
}
```

```text
Cancelled cleanly after 32686553 iterations   // exact count varies every run
Done
```

!!! warning "A `while (true)` loop with no suspension point ignores cancellation"
    Replace `while (isActive)` above with `while (true)` and `job.cancelAndJoin()`
    will hang forever — the loop never calls anything that checks for
    cancellation, so the "cancelled" flag is simply never noticed. Always
    either check `isActive`/`ensureActive()` in CPU-bound loops, or call a
    suspending function periodically (even `yield()`) so cancellation has a
    chance to take effect.

Cleanup code (closing a file, releasing a lock) inside a `finally` block
still runs on cancellation — but if that cleanup itself needs to suspend,
wrap it in `withContext(NonCancellable)`, since a *cancelled* coroutine
can't normally call more suspending functions:

```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking {
    val job = launch {
        try {
            delay(1000)
        } finally {
            withContext(NonCancellable) {
                delay(50)   // would otherwise throw immediately -- already cancelling
                println("Cleanup finished")
            }
        }
    }
    delay(100)
    job.cancelAndJoin()
}
```

```text
Cleanup finished
```

## Dispatchers: which thread(s) run the coroutine

A `Dispatcher` decides what thread pool a coroutine runs on. You rarely
need to think about this until you're mixing CPU work and I/O.

| Dispatcher | Use for |
|---|---|
| `Dispatchers.Default` | CPU-bound work (sorting, parsing, computation) |
| `Dispatchers.IO` | Blocking I/O (network calls, file access, JDBC) |
| `Dispatchers.Main` | Updating UI (Android/desktop UI frameworks only) |
| none specified in `runBlocking` | Runs on the calling thread |

```kotlin
import kotlinx.coroutines.*

suspend fun readConfig(): String = withContext(Dispatchers.IO) {
    // Pretend this is a blocking file read -- withContext moves it off
    // whatever dispatcher called readConfig(), and back when it's done.
    "config loaded"
}

fun main() = runBlocking {
    println(readConfig())
}
```

```text
config loaded
```

## Cheat sheet

| Concept | Purpose |
|---|---|
| `suspend fun` | A function that can pause without blocking a thread |
| `runBlocking { }` | Bridge blocking code (like `main`) into coroutine world |
| `launch { }` | Start a coroutine, get a `Job`, no return value |
| `async { }` / `.await()` | Start a coroutine that produces a value, run concurrently |
| `coroutineScope { }` | Structured concurrency -- waits for all children, propagates failure |
| `delay(ms)` | Non-blocking suspend for a duration |
| `isActive` / `ensureActive()` | Cooperative cancellation check in a loop |
| `withContext(NonCancellable)` | Run cleanup that must suspend, even while cancelling |

## Exercise

Write a `suspend fun fetchTemperature(city: String): Double` that `delay`s
for a random amount of time (e.g. `Random.nextLong(100, 400)`) before
returning a fake temperature. Then write a `main` (using `runBlocking`)
that uses `async` to fetch the temperature for three different cities
concurrently, awaits all three, and prints them along with the total
elapsed time using `measureTimeMillis` — confirm the total is close to the
*slowest* single fetch, not the sum of all three.
