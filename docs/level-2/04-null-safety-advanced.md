---
description: "Null Safety Advanced — Level 1 covered ?, ?., ?:, and !! — the everyday tools. This module covers the trickier corners: what happens when nullability…"
---

# 04 · Null Safety Advanced

[Level 1](../level-1/06-null-safety.md) covered `?`, `?.`, `?:`, and `!!` —
the everyday tools. This module covers the trickier corners: what happens
when nullability meets Java interop, how `lateinit` lets you defer
initialization safely, nullable *generic* types, and why the compiler
sometimes "forgets" a null check you just did.

## Platform types: when Kotlin can't see nullability

Java has no `?` — a `String` in Java might or might not be null, and the
compiler can't tell you which. When Kotlin calls Java code, it can't
assume either `String` or `String?`, so it uses a **platform type**,
written internally as `String!`. Kotlin lets you treat a platform type as
either nullable or non-null — but if you treat it as non-null and it turns
out to actually be null, you get a runtime `NullPointerException` with
*zero compile-time warning*, exactly like plain Java.

```java
// LegacyUserLookup.java -- imagine this is an old Java library with no
// @Nullable/@NotNull annotations, so Kotlin can't know its real nullability.
public class LegacyUserLookup {
    public static String findEmail(String userId) {
        return userId.equals("known") ? "user@example.com" : null;
    }
}
```

```kotlin
fun main() {
    val email = LegacyUserLookup.findEmail("unknown")   // platform type String!
    println(email.length)   // compiles fine -- but throws NullPointerException at runtime!
}
```

```text
Exception in thread "main" java.lang.NullPointerException
```

!!! warning "Treat unannotated Java return values as nullable"
    The safe move when calling Java code without nullability annotations
    is to immediately assign the result to an explicitly nullable Kotlin
    type (`val email: String? = LegacyUserLookup.findEmail(id)`), forcing
    yourself to handle the `null` case with `?.`/`?:` right away instead of
    hoping it's never null.

## `lateinit`: deferred initialization without nullability

Sometimes a `var` genuinely can't have a value at construction time (it's
set later by a framework, a DI container, or a setup function) but you
don't want to make it nullable and litter your code with `!!` or `?.`.
`lateinit` lets a non-null `var` skip initialization until later.

```kotlin
class ReportGenerator {
    lateinit var title: String   // non-null String, but not set yet

    fun configure(reportTitle: String) {
        title = reportTitle
    }

    fun render(): String {
        // isInitialized lets you check safely before first use
        if (!this::title.isInitialized) {
            return "Error: title not configured"
        }
        return "Report: $title"
    }
}

fun main() {
    val report = ReportGenerator()
    println(report.render())          // Error: title not configured

    report.configure("Q3 Sales")
    println(report.render())          // Report: Q3 Sales
}
```

```text
Error: title not configured
Report: Q3 Sales
```

`lateinit` only works on `var` properties of non-null, non-primitive types
(no `Int`, `Boolean`, etc. — use a nullable type with a default for those).
Accessing a `lateinit` property before it's set throws
`UninitializedPropertyAccessException`, not `NullPointerException` — a
clearer signal that *initialization order* is the bug, not a stray null.

### `lateinit` vs. `by lazy`

Both defer a value, but for different situations:

| | `lateinit var` | `val x by lazy { ... }` |
|---|---|---|
| Mutability | Mutable (`var`) | Read-only (`val`) |
| Who sets it | External code, any time | Computed once, on first access |
| Initializer | None at the declaration | A lambda you provide |
| Thread safety | None built in | Synchronized by default |
| Typical use | Android `onCreate`/DI-injected fields | Expensive values computed on demand |

## Nullable generics: `T?` inside vs. outside

`List<Int?>` and `List<Int>?` look similar but mean very different things —
mixing them up is a common source of confusion.

```kotlin
fun main() {
    val listOfNullableInts: List<Int?> = listOf(1, null, 3)   // the LIST always exists;
    println(listOfNullableInts)                                 // some ELEMENTS may be null

    val nullableList: List<Int>? = null   // the LIST itself might not exist;
    println(nullableList?.size ?: "no list at all")   // elements, if present, are never null

    // Summing a List<Int?> needs to handle per-element nulls:
    val total = listOfNullableInts.filterNotNull().sum()
    println(total)   // 4 -- (1 + 3), null was dropped
}
```

```text
[1, null, 3]
no list at all
4
```

Generic type parameters are non-null by default too — `class Box<T>` means
`T` can't be `Any?` unless you say so with `class Box<T : Any?>` (or just
use `T?` at the point you need a nullable value of that type):

```kotlin
class Box<T : Any>(val value: T)   // T is constrained to non-null types

fun main() {
    val box = Box("hello")     // fine
    // val bad = Box<String?>(null)   // compile error -- String? doesn't satisfy T : Any
    println(box.value)
}
```

```text
hello
```

## Trap: smart-cast invalidation

The compiler smart-casts a nullable `val` to non-null after a null check —
but only when it can *prove* nothing else could have changed the value in
between. A `var` property (even a local one captured by a lambda, or a
`var` on a class with a custom getter) can fail this proof, since Kotlin
can't guarantee another thread or a later line didn't reset it.

```kotlin
class Config {
    var name: String? = "default"   // a mutable property, not a local val
}

fun printLength(config: Config) {
    if (config.name != null) {
        // println(config.name.length)   // compile error! `name` is a `var`
        // property -- Kotlin can't prove it's still non-null by the time
        // this line runs (another thread, or a custom getter, could change it)
        println(config.name!!.length)     // workaround: `!!`, or copy to a local val first
    }
}

fun printLengthFixed(config: Config) {
    val name = config.name          // copy into a local val
    if (name != null) {
        println(name.length)         // smart-casts fine -- a local `val` can't change
    }
}

fun main() {
    val config = Config()
    printLength(config)
    printLengthFixed(config)
}
```

```text
7
7
```

The fix is almost always the same: copy the nullable `var`/property into a
local `val` first, then null-check *that*. Local `val`s are the one thing
the compiler can always fully reason about.

## How It Actually Works

Platform types (`String!`) are a fiction that exists **only inside the
compiler's type-checking phase** — by the time bytecode is generated,
`String!` erases to the exact same `java.lang.String` as `String` or
`String?`. What's actually happening is that the compiler has no static
information to run its nullability analysis on (no `@Nullable`/`@NotNull`
annotation to read), so it disables the compile-time null-check entirely for
that value and lets you use it as either type without a warning. This is why
`email.length` in the example above compiles clean — the compiler isn't
"guessing non-null," it's simply not checking at all, exactly as javac
wouldn't. The runtime NPE you get isn't a Kotlin-specific exception either;
it's a bare `java.lang.NullPointerException`, thrown by the JVM the instant
`invokevirtual length()` is attempted on a null reference — the same failure
mode Java itself has, because at that point there is no Kotlin null-safety
machinery left to intercept it.

`lateinit var title: String` compiles to a plain, ordinary (non-final)
`String` field with no wrapper, no `Optional`, and critically **no
initialization** in the constructor — the JVM defaults an uninitialized
reference field to `null` automatically, same as Java, so `lateinit` is
really "let the JVM's normal zero-value default stand in temporarily,
but forbid me from reading it while it's still null." The compiler enforces
that promise by inserting a null-check (`Intrinsics.checkNotNullField` /
equivalent) into the generated getter for that property, so reading an
unset `lateinit var` throws
`UninitializedPropertyAccessException` (not a generic NPE — a distinct,
more diagnostic exception type made specifically for this case). `this::title
.isInitialized` compiles to a call into Kotlin reflection
(`kotlin-reflect`, or an intrinsic check when reflection isn't on the
classpath) that inspects the backing field directly to see if it's still
null, bypassing the getter's own throwing check.

## Cheat sheet

| Situation | Tool |
|---|---|
| Calling unannotated Java code | Assign to an explicit `T?` immediately, don't trust the platform type |
| Non-null property set later by external code | `lateinit var` |
| Expensive value computed once, lazily | `val x by lazy { ... }` |
| Nullable elements in a non-null collection | `List<T?>` |
| Possibly-absent collection | `List<T>?` |
| Smart cast fails on a `var`/property | Copy to a local `val` first, then null-check |

## Exercise

Write a class `UserSession` with a `lateinit var token: String`. Add a
function `isLoggedIn(): Boolean` that uses `::token.isInitialized` to check
whether the session has a token without throwing. Then write a
`requireToken(): String` that returns the token or throws a clear
`IllegalStateException` (via `check()`) if it isn't initialized yet. Test
both functions before and after calling a `login(newToken: String)` method
that sets `token`.
