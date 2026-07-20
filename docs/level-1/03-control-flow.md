# 03 · Control Flow

Kotlin's `if` and loops look familiar coming from Java, but `if` is an
*expression* that can produce a value, and `when` — Kotlin's replacement for
`switch` — is one of the language's signature idioms. Master `when` early;
you'll use it constantly.

## `if` as an expression

In Kotlin, `if`/`else` can be used anywhere an expression is expected,
replacing the ternary operator that Kotlin deliberately omits.

```kotlin
fun main() {
    val age = 20

    // Statement form -- familiar
    if (age >= 18) {
        println("Adult")
    } else {
        println("Minor")
    }

    // Expression form -- assigns directly, no ternary operator needed
    val status = if (age >= 18) "Adult" else "Minor"
    println(status)
}
```

```text
Adult
Adult
```

## `when` — Kotlin's `switch`, and much more

`when` checks a value against multiple branches. Unlike Java's `switch`, it
requires no `break` (no fallthrough by default), can match ranges and types,
and can be used as an expression.

```kotlin
fun main() {
    val day = 3

    // Statement form
    when (day) {
        1 -> println("Monday")
        2 -> println("Tuesday")
        3 -> println("Wednesday")
        else -> println("Some other day")
    }

    // Expression form -- must be exhaustive (else required unless all cases covered)
    val dayName = when (day) {
        1 -> "Monday"
        2 -> "Tuesday"
        3 -> "Wednesday"
        4, 5 -> "Almost the weekend"   // multiple values, comma-separated
        else -> "Unknown"
    }
    println(dayName)
}
```

```text
Wednesday
Wednesday
```

### `when` with ranges and conditions

```kotlin
fun main() {
    val score = 82

    val grade = when (score) {
        in 90..100 -> "A"
        in 80..89 -> "B"
        in 70..79 -> "C"
        else -> "F"
    }
    println(grade)   // B

    // No subject at all -- each branch is its own boolean condition
    val temperature = 15
    val description = when {
        temperature < 0 -> "Freezing"
        temperature in 0..15 -> "Cold"
        temperature in 16..25 -> "Mild"
        else -> "Hot"
    }
    println(description)   // Cold
}
```

### `when` with types (`is`)

```kotlin
fun describe(value: Any): String = when (value) {
    is Int -> "An integer: $value"
    is String -> "A string of length ${value.length}"
    is Boolean -> "A boolean: $value"
    else -> "Something else"
}

fun main() {
    println(describe(42))
    println(describe("hello"))
    println(describe(true))
}
```

```text
An integer: 42
A string of length 5
A boolean: true
```

Note that inside the `is String ->` branch, `value` is automatically
**smart-cast** to `String` — no explicit cast needed, unlike Java's older
`instanceof` + cast pattern.

## `for` loops

Kotlin's `for` iterates over anything "iterable" — ranges, collections,
strings — there is no classic C-style `for (int i = 0; ...)`.

```kotlin
fun main() {
    // Range, inclusive of both ends
    for (i in 1..5) {
        print("$i ")
    }
    println()

    // Range, exclusive of the end
    for (i in 1 until 5) {
        print("$i ")
    }
    println()

    // Step and downTo
    for (i in 10 downTo 0 step 2) {
        print("$i ")
    }
    println()

    // Over a collection
    val fruits = listOf("apple", "banana", "cherry")
    for (fruit in fruits) {
        print("$fruit ")
    }
    println()

    // With index
    for ((index, fruit) in fruits.withIndex()) {
        println("$index: $fruit")
    }
}
```

```text
1 2 3 4 5
1 2 3 4
10 8 6 4 2 0
apple banana cherry
0: apple
1: banana
2: cherry
```

## `while` and `do-while`

```kotlin
fun main() {
    var count = 3
    while (count > 0) {
        println("Countdown: $count")
        count--
    }

    var input = 0
    do {
        println("Runs at least once, input=$input")
        input++
    } while (input < 2)
}
```

```text
Countdown: 3
Countdown: 2
Countdown: 1
Runs at least once, input=0
Runs at least once, input=1
```

## `break`, `continue`, and labels

```kotlin
fun main() {
    outer@ for (i in 1..3) {
        for (j in 1..3) {
            if (j == 2) continue@outer   // skip to next i, not just next j
            println("i=$i j=$j")
        }
    }
}
```

```text
i=1 j=1
i=2 j=1
i=3 j=1
```

Labels (`outer@`) let `break`/`continue` target an outer loop directly —
useful for nested loops without extra flag variables.

## Cheat sheet

| Concept | Syntax |
|---------|--------|
| `if` as expression | `val x = if (cond) a else b` |
| `when` (switch replacement) | `when (x) { 1 -> ...; else -> ... }` |
| `when` with no subject | `when { cond1 -> ...; else -> ... }` |
| `when` with type check | `when (x) { is String -> ...; }` |
| Inclusive range | `1..5` |
| Exclusive range | `1 until 5` |
| Reverse range | `5 downTo 1` |
| Step | `1..10 step 2` |
| Loop with index | `for ((i, v) in list.withIndex())` |
| Labeled continue | `continue@outer` |

## Exercise

Write a program that loops from 1 to 30 and, for each number, prints
`"Fizz"` if divisible by 3, `"Buzz"` if divisible by 5, `"FizzBuzz"` if
divisible by both, or the number itself otherwise — implement the branching
with a single `when` expression (no chained `if`/`else if`). Then write a
second version using ranges to categorize each number 1-30 as `"low"` (1-10),
`"mid"` (11-20), or `"high"` (21-30) using `when` with `in` range checks.
