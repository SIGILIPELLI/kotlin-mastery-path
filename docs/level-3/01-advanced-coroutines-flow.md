# 01 · Advanced Coroutines (Flow)

[Level 2](../level-2/03-coroutines-basics.md) covered `suspend` functions,
`launch`, and `async` — coroutines that produce a *single* value (or none).
`Flow` is Kotlin's answer for a *stream* of values over time: think of it
as the asynchronous, suspend-aware cousin of `Sequence`. This module covers
cold flows, the operator toolkit, and the two "hot" flow types —
`StateFlow` and `SharedFlow` — that model state and events shared across
collectors.

Flow lives in `kotlinx.coroutines.flow`, part of the same
`kotlinx-coroutines-core` dependency from Level 2.

## Cold flows: nothing runs until you collect

A `Flow` built with the `flow { }` builder is **cold**: the block inside it
doesn't execute at all until a collector calls `collect`. Every new
collector re-runs the block from scratch.

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*

fun simpleFlow(): Flow<Int> = flow {
    for (i in 1..3) {
        delay(50)
        println("Emitting $i")
        emit(i)
    }
}

fun main() = runBlocking {
    println("Calling simpleFlow()...")
    val flow = simpleFlow()
    println("Calling collect...")
    flow.collect { value -> println("Collected $value") }
}
```

```text
Calling simpleFlow()...
Calling collect...
Emitting 1
Collected 1
Emitting 2
Collected 2
Emitting 3
Collected 3
```

Notice "Calling simpleFlow()..." and "Calling collect..." print *before*
anything is emitted — building a `Flow` value does no work. Also notice the
emit/collect pairs interleave: each `emit` suspends the producer until the
downstream collector's lambda finishes, so this is a fully synchronous
handoff, not a buffered queue (that's what `buffer()` is for, see below).

## The operator toolkit

Flow ships the same shape of operators as `Sequence` — `map`, `filter`,
`onEach`, `take` — but every one of them is `suspend`-aware, so lambdas can
call `delay` or other suspend functions freely.

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*

fun main() = runBlocking {
    val result = (1..5).asFlow()
        .filter { it % 2 == 1 }
        .map { it * it }
        .toList()
    println(result)

    (1..3).asFlow()
        .onEach { delay(10) }
        .catch { e -> println("Caught: $e") }
        .collect { println("Value: $it") }

    val risky = flow {
        emit(1)
        emit(2)
        throw RuntimeException("boom")
    }
    risky
        .catch { e -> println("Handled: ${e.message}") }
        .collect { println("Got $it") }
}
```

```text
[1, 9, 25]
Value: 1
Value: 2
Value: 3
Got 1
Got 2
Handled: boom
```

`catch` only catches exceptions thrown **upstream** of it in the chain —
it's an operator, not a `try/catch` around the whole pipeline. Put it right
before `collect` to catch everything above it, including exceptions from
the flow builder itself. An exception thrown *inside* the `collect { }`
lambda is not caught by an upstream `catch` — wrap that in a regular
`try/catch` if needed.

## `StateFlow` and `SharedFlow`: hot flows

Cold flows model "a recipe for producing values." `StateFlow` and
`SharedFlow` are **hot**: they exist and hold/broadcast values whether or
not anyone is collecting, which makes them a good fit for shared UI state
or event buses.

`StateFlow` always has a current value (`.value`), always replays exactly
that one value to new collectors, and — critically — **skips emitting a
value equal to the current one** (conflation).

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*

fun main() = runBlocking {
    val state = MutableStateFlow(0)
    println("Initial: ${state.value}")

    val job = launch {
        state.collect { println("Collector saw: $it") }
    }
    delay(10)
    state.value = 1
    delay(10)
    state.value = 2
    delay(10)
    state.value = 2 // duplicate -- StateFlow drops it, no re-emit
    delay(10)
    job.cancel()

    val shared = MutableSharedFlow<String>(replay = 1)
    shared.emit("before subscribe")
    val subJob = launch {
        shared.collect { println("Sub got: $it") }
    }
    delay(10)
    shared.emit("after subscribe")
    delay(10)
    subJob.cancel()
}
```

```text
Initial: 0
Collector saw: 0
Collector saw: 1
Collector saw: 2
Sub got: before subscribe
Sub got: after subscribe
```

Note the second `state.value = 2` produces **no** extra "Collector saw: 2"
line — that's conflation in action. `SharedFlow` has no built-in "current
value" concept; `replay` controls how many *past* emissions a new
collector receives, and with `replay = 1` the value emitted before anyone
subscribed was still delivered once a collector showed up.

## Kotlin-specific traps

- **`catch` placement matters.** `catch` only sees exceptions from
  operators *above* it. `flow.map { risky() }.catch { }.filter { }` will
  not catch an exception thrown inside `filter`.
- **`StateFlow` needs an initial value and never completes.** Unlike a cold
  flow, `collect` on a `StateFlow`/`SharedFlow` suspends forever (until
  cancelled) — there's no natural "end of stream."
- **Collecting a cold flow twice re-runs it twice.** If a flow wraps a
  network call, `.toList()` followed by `.count()` on the same flow value
  makes the request twice. Cache the result (e.g. with `.toList()` once)
  if you need to reuse it.
- **`flowOn` changes the upstream context, not downstream.** `flow{...}.flowOn(Dispatchers.IO).map{...}` runs the builder on IO but `map` still runs in the collector's context — a common point of confusion when reasoning about which dispatcher code executes on.
- **`launchIn` needs a scope.** `flow.onEach{...}.launchIn(scope)` is the idiomatic way to start collecting without wrapping everything in `launch { flow.collect {} }`, but forgetting the scope argument is a common compile error.

## How It Actually Works

A `Flow<T>` is, at its core, a single-method interface:
`suspend fun collect(collector: FlowCollector<T>)`. `flow { ... }` doesn't
run anything — it just wraps your lambda body into an object implementing
that interface, and `emit(value)` inside it is itself a `suspend` function
call that invokes the downstream collector's own `suspend fun
emit(value: T)` directly. This is why building `simpleFlow()` prints
nothing: you've only constructed an object holding a reference to a lambda,
and `collect { }` is the first thing that actually calls `invoke()` on it.
Because `emit` calls straight into the collector's suspend function rather
than pushing into a queue, the "producer suspends until the collector's
lambda finishes" behavior isn't a design choice bolted on top — it's a
direct consequence of `emit` being an ordinary (if compiler-transformed
into a state machine) suspending call: control literally doesn't return to
the `for` loop in `flow { }` until the collector lambda's own generated
continuation resumes it.

Each operator (`.map`, `.filter`, `.onEach`) is implemented as a new `Flow`
that wraps the upstream one, so a chain like `.filter { }.map { }` is really
nested `Flow` objects — the outer one, when collected, collects the flow
underneath it and, in its own `emit` implementation, applies the
transformation before forwarding the value onward, exactly mirroring the
lazy `Sequence` wrapper-chain mechanics from the collections module, except
each `emit`/collect step is a `suspend` call instead of a plain function
call, which is what lets `.onEach { delay(10) }` genuinely pause between
values without blocking a thread.

`.catch { }` works by having the generated collector wrap the **upstream**
`collect` call in a `try`/`catch` and route any exception thrown during
upstream emission into the catch lambda instead of propagating it further —
but it deliberately does **not** catch exceptions thrown by the *downstream*
collector's own `collect { }` block, only ones from operators above it in
the chain. That asymmetry is enforced by exactly where in the generated
code the `try` block's boundaries sit, and it's why `.catch` must appear
before `.collect` in the chain to have any effect on an exception thrown
inside the upstream flow.

## Cheat sheet

| Concept | Type | Key trait |
|---|---|---|
| `flow { emit(x) }` | Cold `Flow<T>` | Runs per collector, from scratch |
| `.map`, `.filter`, `.onEach` | Operators | Suspend-aware, chainable |
| `.catch { }` | Operator | Catches only upstream exceptions |
| `MutableStateFlow(initial)` | Hot, `StateFlow<T>` | Has `.value`, conflates equal values |
| `MutableSharedFlow(replay = n)` | Hot, `SharedFlow<T>` | No current value, replays last `n` |
| `.collect { }` | Terminal operator | Suspends until flow completes (or forever for hot flows) |
| `.toList()`, `.first()`, `.count()` | Terminal operators | Convenience collectors |

## Exercise

Write a cold `Flow<Int>` that emits the first 10 Fibonacci numbers (with a
short `delay` between each, to simulate work). Using operators, produce:

1. The sum of the even Fibonacci numbers in that sequence.
2. A `StateFlow<Int>` that always holds the *running total* seen so far,
   updated as the flow is collected (hint: launch a collector that updates
   a `MutableStateFlow`).

Print the running total after each update, and print the final even-sum at
the end.
