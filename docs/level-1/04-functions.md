# 04 · Functions

Functions are top-level citizens in Kotlin — they don't need to live inside a
class the way they must in Java. This module covers declaring functions,
default and named arguments, and the compact single-expression syntax you'll
see everywhere in idiomatic Kotlin code.

## Basic function syntax

```kotlin
fun greet(name: String): String {
    return "Hello, $name!"
}

fun main() {
    println(greet("Alice"))
}
```

```text
Hello, Alice!
```

## Single-expression functions

When a function body is a single expression, you can drop the braces,
`return`, and often the return-type annotation too (it's inferred).

```kotlin
// Full form
fun add(a: Int, b: Int): Int {
    return a + b
}

// Single-expression form -- equivalent, much shorter
fun addShort(a: Int, b: Int): Int = a + b

// Return type inferred from the expression
fun square(n: Int) = n * n

fun main() {
    println(add(2, 3))
    println(addShort(2, 3))
    println(square(5))
}
```

```text
5
5
25
```

## Default arguments

Parameters can have default values, so callers only need to supply what
differs from the default — no method-overloading juggling required.

```kotlin
fun buildGreeting(name: String, greeting: String = "Hello", punctuation: String = "!"): String {
    return "$greeting, $name$punctuation"
}

fun main() {
    println(buildGreeting("Alice"))                          // uses both defaults
    println(buildGreeting("Bob", "Hi"))                       // overrides greeting
    println(buildGreeting("Carol", punctuation = "?"))        // named arg, skips greeting
}
```

```text
Hello, Alice!
Hi, Bob!
Hello, Carol?
```

## Named arguments

Any argument can be passed by name, in any order — especially useful with
several parameters of the same type, where position alone is error-prone.

```kotlin
fun createUser(name: String, age: Int, isAdmin: Boolean = false) {
    println("$name, age $age, admin=$isAdmin")
}

fun main() {
    createUser(name = "Dana", age = 28, isAdmin = true)
    createUser(age = 35, name = "Evan")   // order doesn't matter with named args
}
```

```text
Dana, age 28, admin=true
Evan, age 35, admin=false
```

## Varargs

`vararg` lets a function accept a variable number of arguments of the same
type, collected into an array inside the function.

```kotlin
fun sum(vararg numbers: Int): Int {
    var total = 0
    for (n in numbers) total += n
    return total
}

fun main() {
    println(sum(1, 2, 3))
    println(sum(1, 2, 3, 4, 5))

    val values = intArrayOf(10, 20, 30)
    println(sum(*values))   // spread operator -- unpacks an array into vararg
}
```

```text
6
15
60
```

## Functions as first-class values

A function can be assigned to a variable or passed as an argument using a
function reference (`::name`) or a lambda — the basis for Kotlin's
higher-order function style, covered in depth in
[Level 2](../level-2/02-lambdas-higher-order-functions.md).

```kotlin
fun double(n: Int) = n * 2

fun main() {
    val operation: (Int) -> Int = ::double
    println(operation(5))   // 10

    val lambda: (Int) -> Int = { n -> n * 3 }
    println(lambda(5))      // 15
}
```

```text
10
15
```

## Local functions

Functions can be nested inside other functions, useful for small helpers
that only make sense in one place.

```kotlin
fun processOrder(quantity: Int, price: Double): Double {
    fun applyDiscount(amount: Double): Double =
        if (quantity > 10) amount * 0.9 else amount

    val subtotal = quantity * price
    return applyDiscount(subtotal)
}

fun main() {
    println(processOrder(15, 2.0))   // 27.0 -- discount applied
    println(processOrder(5, 2.0))    // 10.0 -- no discount
}
```

```text
27.0
10.0
```

## Cheat sheet

| Concept | Syntax |
|---------|--------|
| Basic function | `fun name(param: Type): ReturnType { ... }` |
| Single-expression function | `fun name(param: Type) = expression` |
| Default argument | `fun f(x: Int = 10)` |
| Named argument call | `f(x = 5, y = 10)` |
| Varargs | `fun f(vararg nums: Int)` |
| Spread operator | `f(*array)` |
| Function reference | `::functionName` |
| Function type | `(Int) -> Int` |

## Exercise

Write a single-expression function `isPrime(n: Int): Boolean` that checks
whether `n` is a prime number. Then write a function `describeNumbers` with
a `vararg numbers: Int` parameter and a default `label: String = "Numbers"`
parameter that prints the label followed by each number and whether it's
prime, using `isPrime`. Call it once with a few numbers and once more with
named `label` argument.
