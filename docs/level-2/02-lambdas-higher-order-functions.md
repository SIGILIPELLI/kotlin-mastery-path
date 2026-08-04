# 02 · Lambdas & Higher-Order Functions

A **lambda** is a function literal — a block of code you can pass around as
a value. A **higher-order function** is any function that takes another
function as a parameter or returns one. Together these are the foundation
of Kotlin's functional style, and they're what makes the collection
operations in [Module 5](05-collections-deep-dive.md) (`map`, `filter`,
`reduce`) so compact.

## Lambda syntax and function types

A lambda is written `{ parameters -> body }`. Its type is a **function
type**, written `(ParamTypes) -> ReturnType`.

```kotlin
fun main() {
    val square: (Int) -> Int = { x -> x * x }
    println(square(5))          // 25

    val add: (Int, Int) -> Int = { a, b -> a + b }
    println(add(2, 3))           // 5

    val sayHi: () -> Unit = { println("Hi!") }
    sayHi()
}
```

```text
25
5
Hi!
```

The compiler can infer the lambda's parameter types from context, so you
rarely need to spell out `(Int) -> Int` explicitly when passing a lambda
directly to a function that expects one — Kotlin figures it out from the
parameter's declared type.

## Higher-order functions: taking a function as a parameter

A function is "higher-order" once one of its parameters is itself a
function type.

```kotlin
fun applyTwice(x: Int, operation: (Int) -> Int): Int {
    return operation(operation(x))
}

fun main() {
    val result = applyTwice(3) { n -> n * 2 }   // trailing lambda syntax
    println(result)   // 12 -- (3*2)*2
}
```

```text
12
```

When the last parameter of a function is a function type, Kotlin lets you
move the lambda *outside* the parentheses — that's the "trailing lambda"
syntax used above (`applyTwice(3) { n -> n * 2 }` instead of
`applyTwice(3, { n -> n * 2 })`). If the lambda is the *only* argument, the
parentheses can be dropped entirely.

### The implicit `it`

When a lambda takes exactly one parameter and you don't name it, Kotlin
lets you refer to it as `it`:

```kotlin
fun describe(numbers: List<Int>, predicate: (Int) -> Boolean): List<Int> =
    numbers.filter(predicate)

fun main() {
    val numbers = listOf(1, 2, 3, 4, 5, 6)
    println(describe(numbers) { it % 2 == 0 })   // [2, 4, 6] -- `it` is each number
}
```

```text
[2, 4, 6]
```

`it` is convenient for short lambdas but hurts readability once the lambda
does anything non-trivial — name the parameter explicitly (`{ n -> ... }`)
as soon as the body is more than one simple expression.

## Functions that return functions

A function can also *return* a function type, which is how you build
configurable behavior — a function factory.

```kotlin
fun multiplier(factor: Int): (Int) -> Int {
    return { number -> number * factor }
}

fun main() {
    val double = multiplier(2)
    val triple = multiplier(3)

    println(double(5))   // 10
    println(triple(5))   // 15
}
```

```text
10
15
```

## Closures: capturing variables

A lambda "closes over" (captures) variables from its surrounding scope,
including mutable ones — unlike Java's lambdas, which can only capture
effectively-final variables.

```kotlin
fun counter(): () -> Int {
    var count = 0
    return {
        count++          // mutating a captured `var` -- allowed in Kotlin
        count
    }
}

fun main() {
    val next = counter()
    println(next())   // 1
    println(next())   // 2
    println(next())   // 3 -- `count` persists between calls, tied to this closure instance
}
```

```text
1
2
3
```

!!! warning "Each closure captures its own copy of the enclosing state"
    Calling `counter()` again creates a brand-new `count` starting at 0 —
    closures are independent per call. A classic trap is capturing a loop
    variable expecting a fresh value per iteration:

    ```kotlin
    fun main() {
        val actions = mutableListOf<() -> Unit>()
        for (i in 1..3) {
            actions.add { println(i) }   // captures `i` by value at each iteration in Kotlin's for-loop
        }
        actions.forEach { it() }   // 1, 2, 3 -- Kotlin's for-loop gives each iteration its own `i`
    }
    ```

    Kotlin's `for (i in range)` binds a fresh `i` each iteration, so this
    prints `1, 2, 3` as expected. The equivalent trap in some other
    languages (a shared mutable loop variable) doesn't apply here — but
    if you capture a `var` you mutate *after* the loop, every closure
    still sees the final value, since they all reference the same variable.

## Function references

Instead of wrapping an existing function in a lambda, reference it directly
with `::`.

```kotlin
fun isEven(n: Int) = n % 2 == 0

fun main() {
    val numbers = listOf(1, 2, 3, 4, 5, 6)
    val words = listOf("", "hi", "", "kotlin")

    // Equivalent ways to pass the same behavior:
    println(numbers.filter { isEven(it) })   // lambda wrapping the function
    println(numbers.filter(::isEven))         // function reference -- more direct

    // Member references (Type::method) work the same way for instance methods:
    println(words.filter(String::isNotEmpty))
}
```

```text
[2, 4, 6]
[2, 4, 6]
[hi, kotlin]
```

## Why `inline` matters

Every lambda is, under the hood, an object implementing a function
interface — passing one normally means allocating an object and an
indirect call. For small, hot higher-order functions (like the custom ones
above), Kotlin lets you mark the function `inline`: the compiler copies the
function's *and* the lambda's code directly into the call site, eliminating
both the object allocation and the call overhead.

```kotlin
inline fun measureAndRun(label: String, block: () -> Unit) {
    val start = System.nanoTime()
    block()
    val elapsedMs = (System.nanoTime() - start) / 1_000_000.0
    println("$label took ${elapsedMs}ms")
}

fun main() {
    measureAndRun("sum") {
        var total = 0L                     // Long -- the sum overflows Int past ~46,000 terms
        for (i in 1..1_000_000) total += i
        println("Total: $total")
    }
}
```

```text
Total: 500000500000
sum took 3.1ms   // exact timing varies per run
```

`inline` is also what makes `return` from inside a lambda passed to that
function work like a normal early return from the enclosing function (a
"non-local return") — without `inline`, that `return` wouldn't be legal.
Almost all of Kotlin's standard library higher-order functions (`filter`,
`map`, `let`, `run`, `also`, `apply`) are `inline` for exactly this reason.

## Cheat sheet

| Syntax | Meaning |
|---|---|
| `(Int) -> String` | Function type: takes an `Int`, returns a `String` |
| `{ x -> x * 2 }` | Lambda literal with named parameter |
| `{ it * 2 }` | Lambda with implicit single parameter |
| `f(x) { ... }` | Trailing lambda -- last function-type param moves outside `()` |
| `::functionName` | Reference to a top-level or member function |
| `inline fun f(block: () -> Unit)` | Lambda body copied inline -- no allocation, allows non-local `return` |

## Exercise

Write a higher-order function `retry(times: Int, action: () -> Boolean):
Boolean` that calls `action()` up to `times` times, stopping as soon as it
returns `true` (simulating retrying a flaky operation), and returns whether
it ever succeeded. Test it with an action that uses a captured `var` counter
to fail the first two times and succeed on the third.
