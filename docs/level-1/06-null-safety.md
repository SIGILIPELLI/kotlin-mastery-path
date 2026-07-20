# 06 · Null Safety Basics

Kotlin's type system distinguishes nullable from non-nullable types at
compile time, which eliminates most `NullPointerException`s before the
program even runs. This is one of Kotlin's most-loved features coming from
Java — get comfortable with `?`, `!!`, `?:`, and safe calls now, since
they're everywhere in idiomatic code.

## Nullable vs. non-nullable types

By default, a Kotlin type cannot hold `null`. To allow it, add `?` after the
type.

```kotlin
fun main() {
    var name: String = "Alice"
    // name = null              // compile error -- String can't be null

    var nickname: String? = "Al"   // String? -- nullable
    nickname = null                 // fine
    println(nickname)
}
```

```text
null
```

The compiler tracks this distinction everywhere: a `String` parameter is
guaranteed non-null by the time your function body runs, so you never need
defensive null checks for it.

## The safe call operator `?.`

Calling a method or property directly on a nullable type is a compile error.
The safe call operator `?.` calls it only if the receiver isn't null,
producing `null` otherwise.

```kotlin
fun main() {
    val nickname: String? = null

    // println(nickname.length)   // compile error -- nickname might be null
    println(nickname?.length)     // null -- safe call short-circuits to null

    val realNickname: String? = "Al"
    println(realNickname?.length) // 2
}
```

```text
null
2
```

Safe calls chain naturally, short-circuiting to `null` at the first `null`
link:

```kotlin
class Address(val city: String?)
class Person(val address: Address?)

fun main() {
    val person1: Person? = Person(Address("Boston"))
    val person2: Person? = Person(null)
    val person3: Person? = null

    println(person1?.address?.city)   // Boston
    println(person2?.address?.city)   // null
    println(person3?.address?.city)   // null
}
```

```text
Boston
null
null
```

## The Elvis operator `?:`

`?:` supplies a default value when the expression on its left is `null` —
named for how the operator looks like a smiley face when rotated.

```kotlin
fun main() {
    val nickname: String? = null

    val displayName = nickname ?: "Anonymous"
    println(displayName)   // Anonymous

    val length = nickname?.length ?: 0
    println(length)         // 0 -- combines safe call + default in one line
}
```

```text
Anonymous
0
```

`?:` can also be used to `return` or `throw` early, which is a very common
Kotlin idiom for guard clauses:

```kotlin
fun printUpper(text: String?) {
    val nonNull = text ?: return   // exit the function early if text is null
    println(nonNull.uppercase())
}

fun main() {
    printUpper("hello")   // HELLO
    printUpper(null)       // prints nothing, function returns immediately
}
```

```text
HELLO
```

## The not-null assertion `!!`

`!!` converts a nullable type to non-null by force — if the value actually
is `null`, it throws a `NullPointerException` immediately. Use it sparingly,
only when you are certain (from context the compiler can't see) that the
value cannot be null.

```kotlin
fun main() {
    val nickname: String? = "Al"
    val length: Int = nickname!!.length   // "I know this isn't null"
    println(length)

    val missing: String? = null
    // println(missing!!.length)   // throws NullPointerException at runtime
}
```

```text
2
```

!!! warning "Avoid `!!` where possible"
    Reaching for `!!` defeats the purpose of Kotlin's null safety — it's a
    promise to the compiler that can still blow up at runtime. Prefer `?.`,
    `?:`, or an explicit `if (x != null)` check instead. `!!` is best reserved
    for truly exceptional situations (e.g. a bug you *want* to crash loudly
    on) rather than routine null handling.

## Smart casts with explicit null checks

Once the compiler proves a nullable variable isn't `null` (e.g. inside an
`if` block), it lets you use it as if it were non-null without `?.` or `!!`.

```kotlin
fun describe(nickname: String?) {
    if (nickname != null) {
        // Inside this block, nickname is smart-cast to String (non-null)
        println("Nickname has ${nickname.length} characters")
    } else {
        println("No nickname set")
    }
}

fun main() {
    describe("Al")
    describe(null)
}
```

```text
Nickname has 2 characters
No nickname set
```

## Safe casts with `as?`

`as` throws a `ClassCastException` on a bad cast; `as?` returns `null`
instead.

```kotlin
fun main() {
    val obj: Any = "hello"

    val asString: String? = obj as? String   // "hello" -- succeeds
    val asInt: Int? = obj as? Int             // null -- fails safely, no exception

    println(asString)
    println(asInt)
}
```

```text
hello
null
```

## Cheat sheet

| Operator | Purpose | Example |
|----------|---------|---------|
| `?` | Declare a nullable type | `var x: String? = null` |
| `?.` | Safe call — no-op if receiver is null | `x?.length` |
| `?:` | Elvis — default if left side is null | `x?.length ?: 0` |
| `?:` + `return`/`throw` | Guard clause | `val v = x ?: return` |
| `!!` | Force non-null, throws if actually null | `x!!.length` |
| `as?` | Safe cast — null instead of exception | `obj as? String` |
| smart cast | Compiler narrows type after a null check | `if (x != null) x.length` |

## Exercise

Write a function `safeDivide(a: Int, b: Int): Int?` that returns `null` if
`b` is zero, otherwise the integer division result. Then write a function
`describeDivision(a: Int, b: Int): String` that calls `safeDivide` and uses
the Elvis operator to produce `"Result: N"` on success or
`"Cannot divide by zero"` on `null`. Test it with a few values including
`b = 0`.
