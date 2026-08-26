# 01 · Advanced Coroutines & Structured Concurrency

[Level 3's Flow module](../level-3/01-advanced-coroutines-flow.md) covered
streams of values over time. This module covers the *scoping* rules that
make coroutines safe to use at scale: how a parent coroutine's lifetime
bounds its children's, how one child's failure propagates, and how to
opt out of that propagation with `supervisorScope` when it's the wrong
default — plus timeouts, essential for production code that talks to
anything over a network.

## Structured concurrency: `coroutineScope` and failure propagation

`coroutineScope { }` doesn't return until every coroutine launched inside
it (directly or nested) completes. If any child throws, the scope cancels
**every other child** and rethrows — there's no way to "leak" a
still-running coroutine out of a `coroutineScope` block, which is the core
guarantee structured concurrency provides.

```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking {
    println("Parent scope starts")
    try {
        coroutineScope {
            launch {
                delay(50)
                println("Child A done")
            }
            launch {
                delay(20)
                throw RuntimeException("Child B failed")
            }
            launch {
                delay(200)
                println("Child C should be cancelled before this prints")
            }
        }
    } catch (e: Exception) {
        println("Caught in parent: ${e.message}")
    }
    println("Parent scope ends")
}
```

```text
Parent scope starts
Caught in parent: Child B failed
Parent scope ends
```

Neither "Child A done" nor "Child C should be cancelled..." ever prints.
Child B fails at 20ms, which immediately cancels Child A (still waiting
at 50ms) and Child C (waiting at 200ms) — the exception only surfaces
*after* every sibling has actually finished cancelling, then propagates to
the caller of `coroutineScope`.

## `supervisorScope`: opting out of propagation

Sometimes one failing task genuinely shouldn't cancel its siblings — think
of independent background jobs (send an analytics event, refresh a cache)
where one failing is unfortunate but not fatal to the others.
`supervisorScope` changes the failure-propagation rule: a child's failure
does *not* cancel its siblings, though it's still reported (to the
`CoroutineExceptionHandler`, or — with none installed — the thread's
default handler).

```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking {
    println("-- supervisorScope: sibling failure isolated --")
    supervisorScope {
        val jobA = launch {
            delay(20)
            throw RuntimeException("A failed")
        }
        val jobB = launch {
            delay(50)
            println("B still completed")
        }
        jobA.join()
        jobB.join()
    }
}
```

```text
-- supervisorScope: sibling failure isolated --
Exception in thread "main" java.lang.RuntimeException: A failed
	at L4_01bKt$main$1$1$jobA$1.invokeSuspend(l4_01b.kt:8)
	...
B still completed
```

"B still completed" prints despite A's exception — exactly the isolation
`supervisorScope` provides. Notice the stack trace is printed but the
*process does not crash*: an uncaught exception from a `launch`ed
coroutine with no installed `CoroutineExceptionHandler` goes to the
thread's default uncaught-exception handler, which for a JVM `main` thread
just logs it. This is easy to miss in real code — a silently-logged
failure with no handler is a common source of "why didn't anything happen"
bugs; install a `CoroutineExceptionHandler` on the supervisor scope in
real code rather than relying on the default.

## Timeouts: `withTimeout` and `withTimeoutOrNull`

Any coroutine waiting on I/O needs a timeout — an unresponsive network
call shouldn't hang a coroutine (and its resources) forever.
`withTimeout` throws `TimeoutCancellationException` if the block doesn't
finish in time; `withTimeoutOrNull` returns `null` instead, which is often
more convenient when a timeout is an expected, handled outcome rather than
an error.

```kotlin
try {
    withTimeout(100) {
        delay(500)
        println("This never prints")
    }
} catch (e: TimeoutCancellationException) {
    println("Timed out: ${e.message}")
}

val result = withTimeoutOrNull(100) {
    delay(500)
    "unreachable"
}
println("Result: $result")
```

```text
Timed out: Timed out waiting for 100 ms
Result: null
```

`TimeoutCancellationException` is itself a subtype of
`CancellationException` — the same mechanism coroutine cancellation always
uses. This matters for a common bug: a bare `catch (e: Exception)` around
coroutine code accidentally swallows cancellation/timeout signals too,
which can leave a coroutine hierarchy in an inconsistent state. Catch
specific exception types, or re-throw `CancellationException` after
logging.

## Kotlin-specific traps

- **Catching `CancellationException` (directly, or via a broad `catch
  (e: Exception)`) breaks cooperative cancellation.** Cancellation in
  Kotlin coroutines works by throwing `CancellationException` at suspend
  points — swallowing it without rethrowing leaves the coroutine "alive"
  logically while its parent thinks it's cancelled.
- **`GlobalScope.launch` opts out of structured concurrency entirely** —
  it has no parent to be cancelled by, which means leaked, unbounded
  coroutines. It's rarely the right call outside of genuinely
  fire-and-forget top-level work; prefer a scope tied to a real lifecycle
  (a `ViewModel`'s `viewModelScope`, a request's own `coroutineScope`).
- **`supervisorScope` only changes propagation for *direct children* of
  that scope.** A `launch` nested two levels deep still cancels its own
  parent normally — supervision doesn't apply transitively through
  ordinary nested scopes, only at the level where `supervisorScope` (or a
  `SupervisorJob`) is actually used.
- **`withTimeout` cancels via the same suspend-point mechanism as
  `Job.cancel()`.** Code that never actually suspends (a tight
  non-suspending CPU loop) inside a `withTimeout` block will **not** be
  interrupted — cancellation is cooperative, not preemptive, so CPU-bound
  work needs to call `yield()` or check `isActive` periodically.
- **`launch` vs. `async`'s exception handling differs.** An `async`'s
  exception is stored and only thrown when you call `.await()` — forgetting
  to call `.await()` on a failed `async` silently discards the exception
  (though it's still reported to the parent's exception propagation
  machinery unless the scope is a `SupervisorJob`).

## Cheat sheet

| Concept | API | Failure behavior |
|---|---|---|
| Structured child failure propagates | `coroutineScope { launch { } }` | One failure cancels all siblings |
| Isolated child failures | `supervisorScope { launch { } }` | Failures don't cancel siblings |
| Timeout, throws on expiry | `withTimeout(ms) { }` | `TimeoutCancellationException` |
| Timeout, returns null on expiry | `withTimeoutOrNull(ms) { }` | `null` |
| Unstructured (avoid in app code) | `GlobalScope.launch { }` | No parent, no automatic cancellation |
| Handle otherwise-uncaught failures | `CoroutineExceptionHandler` | Installed on a `supervisorScope`/root scope |

## Exercise

Write a function `fetchAll(urls: List<String>): List<Result<String>>` that
launches one coroutine per URL inside a `supervisorScope` (simulate
fetching with `delay` + either a fake result string or a thrown exception
for "bad" URLs), collecting each result as `Result.success` or
`Result.failure` rather than letting one failure cancel the others. Wrap
the whole thing in `withTimeoutOrNull(1000)` so the entire batch gives up
after a second, and print how many succeeded vs. failed.
