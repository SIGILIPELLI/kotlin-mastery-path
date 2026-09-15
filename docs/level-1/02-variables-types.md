---
description: "Variables & Types — Kotlin is statically typed like Java, but with type inference doing most of the work and a hard distinction between things that can…"
---

# 02 · Variables & Types

Kotlin is statically typed like Java, but with type inference doing most of
the work and a hard distinction between things that can and can't be
reassigned. Get comfortable with `val`/`var` and the basic types before
anything else.

## `val` vs `var`

- **`val`** — a read-only reference. Once assigned, it can't be reassigned
  (though if it points to a mutable object, that object's *contents* can
  still change). Prefer `val` by default.
- **`var`** — a mutable reference. Use it only when reassignment is actually
  needed.

```kotlin
fun main() {
    val name = "Alice"       // val -- cannot be reassigned
    var age = 30             // var -- can be reassigned

    age = 31                 // fine
    // name = "Bob"          // compile error: val cannot be reassigned

    println("$name is $age")
}
```

```text
Alice is 31
```

## Type inference

Kotlin infers the type from the initializer, so explicit type annotations are
usually optional — but you can always write them out.

```kotlin
fun main() {
    val inferred = 42           // inferred as Int
    val explicit: Int = 42      // same thing, spelled out
    val price: Double = 19.99   // annotation required if you want Double from an integer literal

    println(inferred)
    println(explicit)
    println(price)
}
```

## Basic types

| Type | Example | Notes |
|------|---------|-------|
| `Int` | `val x: Int = 42` | 32-bit signed integer |
| `Long` | `val x: Long = 42L` | 64-bit, note the `L` suffix |
| `Double` | `val x: Double = 3.14` | 64-bit floating point (default for decimals) |
| `Float` | `val x: Float = 3.14f` | 32-bit floating point, note the `f` suffix |
| `Boolean` | `val x: Boolean = true` | `true`/`false` only |
| `Char` | `val x: Char = 'K'` | single character, single quotes |
| `String` | `val x: String = "Kotlin"` | double quotes |

```kotlin
fun main() {
    val count: Int = 10
    val total: Long = 10_000_000_000L
    val pi: Double = 3.14159
    val ratio: Float = 0.5f
    val isReady: Boolean = true
    val grade: Char = 'A'
    val language: String = "Kotlin"

    println("$count $total $pi $ratio $isReady $grade $language")
}
```

```text
10 10000000000 3.14159 0.5 true A Kotlin
```

Notice the underscore in `10_000_000_000L` — Kotlin lets you use `_` as a
visual digit separator in numeric literals, purely for readability.

## Everything is an object

Unlike Java, Kotlin has no true "primitives" in the language itself — `Int`,
`Double`, `Boolean`, and friends are all types with methods, and the compiler
optimizes them down to JVM primitives where possible.

```kotlin
fun main() {
    val x = 5
    println(x.toString())        // "5" -- calling a method on an Int
    println((-5).absoluteValue)  // 5 -- extension property from kotlin.math
    println(x.coerceAtLeast(10)) // 10 -- another method available on Int
}
```

```text
5
5
10
```

## String templates

Instead of concatenation, Kotlin lets you embed expressions directly inside
string literals with `$` (simple variable) or `${ }` (any expression).

```kotlin
fun main() {
    val name = "Alice"
    val age = 30

    println("Name: $name")                       // simple variable
    println("Next year: ${age + 1}")              // expression
    println("Uppercase name: ${name.uppercase()}") // method call
}
```

```text
Name: Alice
Next year: 31
Uppercase name: ALICE
```

## Type conversion

Kotlin does not auto-widen numeric types the way Java does — conversions are
always explicit method calls.

```kotlin
fun main() {
    val i: Int = 42
    val l: Long = i.toLong()
    val d: Double = i.toDouble()
    val s: String = i.toString()
    val parsed: Int = "123".toInt()

    println("$l $d $s $parsed")

    // val bad: Long = i  // compile error -- no implicit widening in Kotlin
}
```

```text
42 42.0 42 123
```

## Constants

`const val` declares a compile-time constant — must be a top-level or
companion-object property, and must be a `String` or primitive type known at
compile time.

```kotlin
const val MAX_USERS = 100

fun main() {
    println("Limit: $MAX_USERS")
}
```

## How It Actually Works

`val` and `var` are a source-level distinction only — the JVM bytecode for a
local `val` and a local `var` of the same type is identical (a slot on the
stack frame, loaded/stored with `iload`/`istore` and friends). The
`val`-cannot-be-reassigned rule is enforced entirely by the Kotlin compiler
during the "immutability check" phase before code generation; there is no
`final` flag involved for locals the way there is for fields. For a `val`
declared as a class **property**, though, the compiler does emit a `final`
field plus a getter (and no setter), so at the class level immutability is
real, not just a compiler courtesy.

The basic types are more interesting under the hood: `Int`, `Double`,
`Boolean`, `Char`, etc. are not boxed objects by default. The compiler
represents them as the JVM's raw primitives (`int`, `double`, `boolean`,
`char`) whenever it can prove that's safe — e.g. a non-nullable `Int` local
compiles to a plain `int`. Kotlin only *boxes* a primitive into its wrapper
object (`java.lang.Integer`, etc.) when it must: when the value is nullable
(`Int?`), stored in a generic collection (`List<Int>` erases to
`List<Object>`, forcing boxing), or used through a type parameter. This is
why `Int?` in Kotlin and `int` are genuinely different at the bytecode level
— the `?` isn't cosmetic, it changes which JVM type gets emitted. `Long`'s
`L` suffix and `Float`'s `f` suffix aren't runtime markers either; they only
tell the compiler which literal-parsing rule to apply at compile time — by
the time bytecode exists, the value is just a `long` or `float` slot.

## Cheat sheet

| Concept | Syntax |
|---------|--------|
| Read-only variable | `val x = 10` |
| Mutable variable | `var x = 10` |
| Explicit type | `val x: Int = 10` |
| Long literal | `val x: Long = 10L` |
| Float literal | `val x: Float = 1.5f` |
| String template | `"$name is ${age + 1}"` |
| Convert types | `i.toLong()`, `i.toDouble()`, `"42".toInt()` |
| Compile-time constant | `const val NAME = value` |

## 🔀 See this in another language

- [Scala — Variables & Types](https://sigilipelli.github.io/scala-mastery-path/level-1/02-variables-types/)
- [PowerShell — Variables & Types](https://sigilipelli.github.io/powershell-mastery-path/level-1/02-variables-types/)
- [C++ — Variables, Types & Operators](https://sigilipelli.github.io/cpp-mastery-path/level-1/02-variables-types-operators/)

## Exercise

Write a program that declares a `var` holding a temperature in Celsius as a
`Double`, converts it to Fahrenheit using the formula `f = c * 9 / 5 + 32`,
and prints both values using a string template, e.g.
`25.0°C is 77.0°F`. Then reassign the Celsius `var` to a new value and print
the converted result again.
