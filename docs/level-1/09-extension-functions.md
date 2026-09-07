# 09 · Extension Functions Intro

Extension functions let you add new functions to *existing* types — including
types you don't own, like `String` or `Int` from the standard library —
without inheritance or wrapper classes. This is one of the idioms that makes
Kotlin code read so differently from Java, and the standard library itself
(`map`, `filter`, and friends) is built almost entirely out of extensions.

## Declaring an extension function

```kotlin
fun String.shout(): String {
    return this.uppercase() + "!"
}

fun main() {
    val message = "hello"
    println(message.shout())   // HELLO!
}
```

```text
HELLO!
```

The type before the dot (`String`) is the **receiver type**. Inside the
function body, `this` refers to the specific `String` instance the function
was called on — same as inside a regular member function, even though
`String` itself wasn't modified or subclassed.

## Extensions on your own types

```kotlin
data class Rectangle(val width: Double, val height: Double)

fun Rectangle.area(): Double = width * height

fun Rectangle.isSquare(): Boolean = width == height

fun main() {
    val rect = Rectangle(4.0, 5.0)
    println(rect.area())        // 20.0
    println(rect.isSquare())    // false
}
```

```text
20.0
false
```

This is useful for adding capability to a `data class` you want to keep
minimal and focused purely on holding data, with behavior layered on
separately.

## Extension functions with parameters

Extension functions accept parameters exactly like regular functions.

```kotlin
fun Int.isDivisibleBy(divisor: Int): Boolean = this % divisor == 0

fun main() {
    println(10.isDivisibleBy(2))   // true
    println(10.isDivisibleBy(3))   // false
    println(15.isDivisibleBy(5))   // true
}
```

```text
true
false
true
```

## Extension properties

Extensions aren't limited to functions — you can add computed properties
too (they must be computed, since there's no backing field to store state on
a type you don't own).

```kotlin
val String.lastChar: Char
    get() = this[this.length - 1]

fun main() {
    println("Kotlin".lastChar)   // n
}
```

```text
n
```

## Extensions on nullable types

An extension can be declared on a nullable receiver type (`String?` instead
of `String`), letting it safely handle the `null` case itself rather than
forcing callers to write `?.` everywhere.

```kotlin
fun String?.orDefault(default: String): String {
    return this ?: default
}

fun main() {
    val name: String? = null
    println(name.orDefault("Anonymous"))   // Anonymous -- called directly, no ?. needed

    val realName: String? = "Alice"
    println(realName.orDefault("Anonymous"))   // Alice
}
```

```text
Anonymous
Alice
```

## How extension functions actually resolve (and their limits)

Extension functions are resolved **statically**, based on the declared type
at the call site — they are not truly added to the class, just syntactic
sugar for a regular function taking the receiver as its first argument. This
means:

- They cannot access private members of the class they extend.
- They cannot be overridden polymorphically the way member functions can.
- If a member function and an extension function have the same signature,
  the member function always wins.

```kotlin
fun main() {
    val obj: Any = "hello"
    // obj.shout()   // compile error even if shout() is defined for String --
                      // "obj" is statically typed as Any, not String, at this call site
}
```

## Practical example: a small utility library

```kotlin
fun List<Int>.average2(): Double {
    if (this.isEmpty()) return 0.0
    return this.sum().toDouble() / this.size
}

fun String.isValidEmail(): Boolean {
    return this.contains("@") && this.substringAfter("@").contains(".")
}

fun main() {
    val numbers = listOf(4, 8, 15, 16, 23, 42)
    println(numbers.average2())   // 18.0

    println("alice@example.com".isValidEmail())   // true
    println("not-an-email".isValidEmail())         // false
}
```

```text
18.0
true
false
```

## How It Actually Works

Extension functions are pure compiler illusion — the JVM has no concept of
"attaching" a method to a class you don't own, so Kotlin doesn't actually
modify `String` or `Rectangle` at all. `fun String.shout(): String` compiles
to an ordinary **static method** on the file's synthetic container class,
with the receiver passed as an invisible first parameter: effectively
`public static String shout(String $this$shout)`. Calling `message.shout()`
compiles to `ExtensionsKt.shout(message)` — a plain static call, not a
virtual method invocation (`invokevirtual`). You can confirm this with
`javap`: `String.class` itself never changes, and there's no `shout()`
listed on it.

This has a real, observable consequence: extension functions are resolved
**statically, by the declared compile-time type**, not dynamically by the
runtime type — the opposite of how member function overrides work. If you
declare `fun Animal.speak() = "..."` and `fun Dog.speak() = "Woof"`, and call
`speak()` on a variable declared as `Animal` but holding a `Dog` at runtime,
the `Animal` extension wins, because the compiler picks which static method
to call based on what it can see at the call site, before the program ever
runs — there is no vtable lookup the way there is for a genuinely overridden
member function. This is often the single most surprising thing about
extension functions to Java developers: they look like polymorphic methods
but behave like statically-dispatched free functions, because that's
literally what they compile to.

`this` inside an extension body refers to the receiver parameter that got
passed in — the compiler substitutes every `this` with a read of that hidden
first parameter, and unqualified property/function access resolves against
it exactly like member access would, purely as a syntactic convenience layered
over what is, underneath, an ordinary static utility function.

## Cheat sheet

| Concept | Syntax |
|---------|--------|
| Extension function | `fun ReceiverType.name(params): ReturnType { ... use this ... }` |
| Extension property (computed only) | `val ReceiverType.name: Type get() = ...` |
| Extension on nullable type | `fun String?.name(): Type { ... }` |
| Resolution | Static — based on declared type, member functions win over extensions |

## Exercise

Write an extension function `Int.factorial(): Long` that computes the
factorial of a non-negative integer (use `require` to reject negative
input). Write an extension property `List<Int>.median: Double` that returns
the median of a non-empty list (sort it first; average the two middle values
if the size is even). Test both with a few inputs.
