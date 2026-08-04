# 09 · Generics

Generics let a class or function work with any type while still giving you
full compile-time type checking — no casts, no `Any` grab-bags. Kotlin's
generics build on the same erasure model as Java's, but add two things Java
doesn't have in this form: declaration-site variance (`in`/`out`) and
`reified` type parameters that survive into runtime for `inline` functions.

## Generic classes and functions

A type parameter in angle brackets (`<T>`) stands in for a real type,
supplied when the class is instantiated or the function is called — usually
inferred from the argument, so you rarely write it out explicitly.

```kotlin
class Box<T>(val item: T) {
    fun get(): T = item
}

fun <T> firstOrNull(list: List<T>): T? = if (list.isEmpty()) null else list[0]

fun main() {
    val intBox = Box(42)
    val stringBox = Box("hello")
    println(intBox.get())
    println(stringBox.get())

    println(firstOrNull(listOf(1, 2, 3)))
    println(firstOrNull(emptyList<String>()))
}
```

```text
42
hello
1
null
```

## Constraints: limiting what `T` can be

An upper bound (`T : SomeType`) restricts a type parameter to types that
extend/implement `SomeType`, which lets you call `SomeType`'s members on a
value of type `T`. `Comparable<T>` is the most common bound — it's how
`>` and `<` become available for a generic value.

```kotlin
fun <T : Comparable<T>> max(a: T, b: T): T = if (a > b) a else b

fun main() {
    println(max(3, 7))              // 7
    println(max("apple", "banana")) // banana
}
```

```text
7
banana
```

Without the `Comparable<T>` bound, `a > b` wouldn't compile — the compiler
has no idea whether an arbitrary `T` supports ordering at all.

## Declaration-site variance: `out` (covariance)

By default, `Box<Dog>` and `Box<Animal>` are unrelated types even if `Dog`
extends `Animal` — generics are *invariant* unless you say otherwise. Marking
a type parameter `out` says "this class only ever *produces* `T` values
(returns them, never accepts them as a parameter)," which lets the compiler
treat `Container<Dog>` as a subtype of `Container<Animal>`.

```kotlin
open class Animal(val name: String)
class Dog(name: String) : Animal(name)

class Container<out T>(val value: T)

fun printAnimal(container: Container<Animal>) {
    println("Contains: ${container.value.name}")
}

fun main() {
    val dogContainer: Container<Dog> = Container(Dog("Rex"))
    printAnimal(dogContainer)   // OK -- Container<Dog> is a subtype of Container<Animal>
}
```

```text
Contains: Rex
```

This is exactly why `List<out T>` lets you pass a `List<Dog>` anywhere a
`List<Animal>` is expected, but `MutableList<T>` (which also *accepts* `T`
via `add`) can't safely be `out` — accepting an `Animal` into what's really a
`MutableList<Dog>` would break type safety, so the compiler forbids marking
a type parameter `out` if it's ever used as a function *parameter* type.

## Declaration-site variance: `in` (contravariance)

`in` is the mirror image: it says "this class only ever *consumes* `T`
values (accepts them as parameters, never returns them)." That makes
`Feeder<Animal>` usable anywhere a `Feeder<Dog>` is needed — a feeder that
knows how to handle *any* animal can certainly handle a dog.

```kotlin
open class Animal(val name: String)
class Dog(name: String) : Animal(name)

interface Feeder<in T> {
    fun feed(animal: T): String
}

fun main() {
    val animalFeeder: Feeder<Animal> = object : Feeder<Animal> {
        override fun feed(animal: Animal) = "Feeding ${animal.name} some generic chow"
    }

    val dogFeeder: Feeder<Dog> = animalFeeder   // OK -- Feeder<in T> is contravariant
    println(dogFeeder.feed(Dog("Rex")))
}
```

```text
Feeding Rex some generic chow
```

`Comparator<in T>` in the standard library works the same way: a
`Comparator<Any>` can compare two `String`s just fine, so it's valid
anywhere a `Comparator<String>` is required.

| Variance | Keyword | Type can be used as | Typical use |
|---|---|---|---|
| Covariant | `out T` | A *return type* only | Producers — `List<out T>`, `Container<out T>` |
| Contravariant | `in T` | A *parameter type* only | Consumers — `Comparator<in T>`, `Feeder<in T>` |
| Invariant | `T` (neither) | Both | Anything that both stores and returns `T` — `MutableList<T>` |

## Star projection: `<*>` when you don't care

Sometimes you need to accept a generic type without caring what its type
argument actually is. `List<*>` means "a `List` of *something*" — you can
read elements out as `Any?`, but you can't add anything to it (the compiler
has no idea what type would be safe to insert).

```kotlin
fun printAllItems(list: List<*>) {
    for (item in list) {
        println(item)
    }
}

fun main() {
    printAllItems(listOf(1, 2, 3))
    printAllItems(listOf("a", "b", "c"))
}
```

```text
1
2
3
a
b
c
```

## Reified type parameters: `T` that survives at runtime

Normally, generic type information is erased at runtime — a compiled
`List<String>` and `List<Int>` look identical to the JVM, so you can't write
`if (value is T)` in an ordinary generic function (there's nothing left at
runtime to check against). Marking an `inline` function's type parameter
`reified` fixes this: the compiler substitutes the *actual* type at every
call site during inlining, so `T` behaves like a real, checkable type.

```kotlin
inline fun <reified T> List<*>.filterByType(): List<T> {
    val result = mutableListOf<T>()
    for (item in this) {
        if (item is T) result.add(item)   // only legal because T is reified
    }
    return result
}

fun main() {
    val mixed = listOf(1, "two", 3, "four", 5.0)
    val strings: List<String> = mixed.filterByType()
    val ints: List<Int> = mixed.filterByType()
    println(strings)
    println(ints)
}
```

```text
[two, four]
[1, 3]
```

`reified` only works on `inline` functions — the compiler needs to paste the
function's body into each call site to know what concrete type to substitute
for `T`. A plain (non-inline) generic function has no way to recover `T` at
runtime, so `value is T` there is a compile error, not just a bad idea.

!!! warning "`reified` is an `inline`-only trick, not a generics upgrade"
    You can't store a `reified` type parameter in a class (only in an
    `inline fun`), and you can't call an `inline reified` function through a
    non-inlined layer of indirection (like a function reference) and expect
    the substitution to still happen. It's a compile-time code-generation
    tool, not a change to how the JVM handles generics underneath.

## Cheat sheet

| Concept | Syntax | Meaning |
|---|---|---|
| Generic class/function | `class Box<T>(...)` / `fun <T> f(...)` | Works with any type, checked at compile time |
| Upper bound | `<T : Comparable<T>>` | Restricts `T`, unlocks that type's members |
| Covariance (producer) | `class C<out T>` | `C<Dog>` is a subtype of `C<Animal>` |
| Contravariance (consumer) | `class C<in T>` | `C<Animal>` is usable as `C<Dog>` |
| Star projection | `List<*>` | "A `List` of something," read-only access |
| Reified type parameter | `inline fun <reified T>` | `T` usable in `is T` checks at runtime |

## Exercise

Write a generic class `Stack<T>` with `push(item: T)`, `pop(): T?`, and
`peek(): T?`, backed by a `MutableList<T>` internally. Then write an inline
extension function `fun <reified T> Stack<*>.countInstancesOf(): Int` that
counts how many elements currently on the stack are of type `T` using an
`is T` check — this only compiles because `T` is `reified`. Test it by
pushing a mix of `Int`, `String`, and `Double` values onto a `Stack<Any>`
and confirming `countInstancesOf<Int>()`, `countInstancesOf<String>()`, and
`countInstancesOf<Double>()` each report the right count.
