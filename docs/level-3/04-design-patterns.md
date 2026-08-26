# 04 · Design Patterns in Kotlin

Classic Gang-of-Four patterns exist to work around limitations of older,
more verbose OOP languages. Kotlin's language features — `object`,
data/sealed classes, function types, extension functions — collapse many
of them into a few lines, or make them unnecessary entirely. This module
walks through the patterns that still earn their keep in Kotlin, and how
idiomatic Kotlin expresses them.

## Singleton: just use `object`

Java's Singleton pattern needs a private constructor, a static field, and
careful double-checked locking for thread safety. Kotlin's `object`
declaration *is* a thread-safe singleton, built into the language.

```kotlin
object ConfigRegistry {
    private val values = mutableMapOf<String, String>()
    fun set(key: String, value: String) { values[key] = value }
    fun get(key: String) = values[key]
}

fun main() {
    ConfigRegistry.set("env", "prod")
    println("Singleton: env=${ConfigRegistry.get("env")}, same instance=${ConfigRegistry === ConfigRegistry}")
}
```

```text
Singleton: env=prod, same instance=true
```

## Builder: fluent setters + `apply`/`also`

Kotlin's named/default arguments remove most of the *original* need for
Builder (avoiding giant constructor calls), but a fluent, chainable builder
is still useful for building up a request or config object step by step.

```kotlin
class HttpRequestBuilder {
    var url: String = ""
    var method: String = "GET"
    private val headers = mutableMapOf<String, String>()
    fun header(name: String, value: String) = apply { headers[name] = value }
    fun build(): String = "$method $url headers=$headers"
}

fun main() {
    val request = HttpRequestBuilder()
        .also { it.url = "https://api.example.com/tasks"; it.method = "POST" }
        .header("Authorization", "Bearer token")
        .header("Content-Type", "application/json")
        .build()
    println("Builder: $request")
}
```

```text
Builder: POST https://api.example.com/tasks headers={Authorization=Bearer token, Content-Type=application/json}
```

`apply` returns the receiver (`this`), so `header(...)` can keep chaining —
that's the whole trick behind fluent builders in Kotlin: any function that
returns `this` (directly, or via `apply`) supports chaining.

## Strategy: function types instead of an interface hierarchy

The Strategy pattern (interchangeable algorithms behind one interface) is
usually just a function type in Kotlin. A `fun interface` (a **SAM**,
single-abstract-method interface) lets callers pass either a named
strategy object or an inline lambda.

```kotlin
fun interface PricingStrategy {
    fun price(base: Double): Double
}

val regularPricing = PricingStrategy { base -> base }
val discountPricing = PricingStrategy { base -> base * 0.9 }

fun checkout(base: Double, strategy: PricingStrategy) = strategy.price(base)

fun main() {
    println("Strategy regular: ${checkout(100.0, regularPricing)}")
    println("Strategy discount: ${checkout(100.0, discountPricing)}")
    println("Strategy inline lambda: ${checkout(100.0) { it * 0.5 }}")
}
```

```text
Strategy regular: 100.0
Strategy discount: 90.0
Strategy inline lambda: 50.0
```

`checkout(100.0) { it * 0.5 }` works because Kotlin SAM-converts the
trailing lambda into a `PricingStrategy` automatically — no explicit
`PricingStrategy { }` wrapper needed at the call site.

## Observer and State: sealed classes + listener lists

`sealed class` hierarchies make the State pattern's core promise — the
compiler knows every possible state — literal and enforced, via exhaustive
`when`. Observer is just a `MutableList` of function-type listeners; no
`Observer`/`Observable` interfaces required.

```kotlin
sealed class OrderState {
    data object Pending : OrderState()
    data object Shipped : OrderState()
    data class Cancelled(val reason: String) : OrderState()
}

fun describe(state: OrderState): String = when (state) {
    is OrderState.Pending -> "waiting to ship"
    is OrderState.Shipped -> "on its way"
    is OrderState.Cancelled -> "cancelled: ${state.reason}"
}

class OrderTracker {
    private val listeners = mutableListOf<(OrderState) -> Unit>()
    var state: OrderState = OrderState.Pending
        set(value) {
            field = value
            listeners.forEach { it(value) }
        }
    fun onChange(listener: (OrderState) -> Unit) = listeners.add(listener)
}

fun main() {
    val tracker = OrderTracker()
    tracker.onChange { println("Listener A: ${describe(it)}") }
    tracker.onChange { println("Listener B notified, state=$it") }

    tracker.state = OrderState.Shipped
    tracker.state = OrderState.Cancelled("customer request")

    val states = listOf(OrderState.Pending, OrderState.Shipped, OrderState.Cancelled("out of stock"))
    states.forEach { println(describe(it)) }
}
```

```text
Listener A: on its way
Listener B notified, state=Shipped
Listener A: cancelled: customer request
Listener B notified, state=Cancelled(reason=customer request)
waiting to ship
on its way
cancelled: out of stock
```

`state` is a custom property setter, not a plain `var` — every assignment
runs through `set(value)`, which is where the listener notification lives.
This is the whole "Observable property" idea, with zero framework code.

## Kotlin-specific traps

- **`when` on a sealed class is only exhaustive without an `else`
  branch if every subtype is covered.** Adding a new subtype later and
  forgetting to update a `when` is caught by the compiler *only* if there's
  no catch-all `else` — resist the urge to add `else -> ...` "just in
  case," since it silently swallows new states.
- **`data object` (Kotlin 1.9+) vs. plain `object` for sealed leaves.**
  `data object` gives you a sensible `toString()`; a plain nested `object`
  prints its mangled class name, which is a common surprise when logging.
- **`apply` vs. `also` vs. `with`/`let`.** `apply`/`also` return the
  receiver (`this`/`it`) — good for builder chains and side effects.
  `let`/`with`/`run` can return something else — good for
  transformations. Using the wrong one breaks a chain silently by
  returning `Unit` instead of the object.
- **A `fun interface` is not the same as a type alias for a function
  type.** `fun interface Strategy { fun run(): Int }` gives you SAM
  conversion *and* a nominal type you can extend/implement normally;
  `typealias Strategy = () -> Int` is purely structural — pick based on
  whether you need named implementations later.
- **Singleton `object`s are eagerly initialized on first access, not at
  class-load time** — usually irrelevant, but it means side effects in an
  `object`'s initializer run lazily, at a time determined by whoever
  references it first.

## Cheat sheet

| Pattern | Idiomatic Kotlin |
|---|---|
| Singleton | `object Name { ... }` |
| Builder | Chainable methods returning `apply { }`, or named/default args |
| Strategy | `fun interface` + lambda, or plain function type parameter |
| Observer | `MutableList<(T) -> Unit>` + custom property setter |
| State | `sealed class`/`sealed interface` + exhaustive `when` |
| Factory | A top-level or companion `fun create(...): Type` |
| Decorator | Extension functions, or delegation via `by` |

## Exercise

Model a traffic light as a sealed class (`Red`, `Yellow`, `Green`) with a
`next()` extension function that returns the correct following state
(`Red -> Green -> Yellow -> Red`). Add an Observer-style `TrafficLight`
class that holds the current state, exposes `advance()` to move to the
next state and notify listeners, and register two listeners: one that
prints the state name, one that prints how many total transitions have
happened so far.
