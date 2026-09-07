# 05 · Classes & Objects Basics

Kotlin classes are more compact than Java's — constructor parameters and
properties are often declared in one line. This module covers the essentials:
defining a class, properties, constructors, and visibility.

## A basic class

```kotlin
class Person(val name: String, var age: Int) {
    fun greet(): String {
        return "Hi, I'm $name and I'm $age years old."
    }
}

fun main() {
    val person = Person("Alice", 30)
    println(person.greet())
    person.age = 31              // "age" is a var, so it can change
    println(person.age)
    // person.name = "Bob"       // compile error -- "name" is a val
}
```

```text
Hi, I'm Alice and I'm 30 years old.
31
```

This single line, `class Person(val name: String, var age: Int)`, replaces
what would be a constructor plus fields plus getters/setters in Java — the
**primary constructor** and its properties are declared together.

## Properties with custom logic

Properties can have custom `get()`/`set()` bodies instead of just storing a
value directly.

```kotlin
class Rectangle(val width: Double, val height: Double) {
    val area: Double
        get() = width * height   // computed every time it's accessed, not stored

    val isSquare: Boolean
        get() = width == height
}

fun main() {
    val rect = Rectangle(4.0, 5.0)
    println(rect.area)       // 20.0
    println(rect.isSquare)   // false

    val square = Rectangle(3.0, 3.0)
    println(square.isSquare) // true
}
```

```text
20.0
false
true
```

## Secondary constructors and `init` blocks

The primary constructor can be supplemented with an `init` block (runs when
the object is created) and secondary constructors for alternate ways to build
an instance.

```kotlin
class Book(val title: String, val author: String, val pages: Int) {

    init {
        require(pages > 0) { "pages must be positive" }
        println("Created: $title")
    }

    // Secondary constructor -- delegates to the primary with "this(...)"
    constructor(title: String, author: String) : this(title, author, 0)
}

fun main() {
    val book1 = Book("Kotlin in Action", "Dmitry Jemerov", 360)
    val book2 = Book("Untitled Draft", "Anonymous")   // uses secondary constructor
    println("${book1.title}: ${book1.pages} pages")
    println("${book2.title}: ${book2.pages} pages")
}
```

```text
Created: Kotlin in Action
Created: Untitled Draft
Kotlin in Action: 360 pages
Untitled Draft: 0 pages
```

`require(...)` throws an `IllegalArgumentException` with the given message if
the condition is false — a common, concise way to validate constructor
arguments.

## Member functions and `this`

```kotlin
class BankAccount(val owner: String, private var balance: Double) {

    fun deposit(amount: Double) {
        require(amount > 0) { "deposit amount must be positive" }
        this.balance += amount   // "this" is optional here, but clarifies intent
    }

    fun withdraw(amount: Double): Boolean {
        if (amount > balance) return false
        balance -= amount
        return true
    }

    fun getBalance(): Double = balance
}

fun main() {
    val account = BankAccount("Alice", 100.0)
    account.deposit(50.0)
    println(account.getBalance())        // 150.0
    println(account.withdraw(200.0))     // false -- insufficient funds
    println(account.withdraw(100.0))     // true
    println(account.getBalance())        // 50.0
}
```

```text
150.0
false
true
50.0
```

## Visibility modifiers

| Modifier | Visible from |
|----------|---------------|
| `public` (default) | Everywhere |
| `private` | Only within the same class/file |
| `protected` | The class and its subclasses |
| `internal` | Anywhere in the same module |

```kotlin
class Counter {
    private var count = 0   // hidden from outside code

    fun increment() {
        count++
    }

    fun current(): Int = count
}

fun main() {
    val counter = Counter()
    counter.increment()
    counter.increment()
    println(counter.current())   // 2
    // counter.count += 1        // compile error -- "count" is private
}
```

```text
2
```

## Objects: `object` for singletons

Kotlin has built-in singleton support via the `object` keyword — no manual
"private constructor + static instance" boilerplate needed.

```kotlin
object AppConfig {
    var appName = "MyApp"
    var version = "1.0.0"

    fun describe(): String = "$appName v$version"
}

fun main() {
    println(AppConfig.describe())
    AppConfig.version = "1.1.0"
    println(AppConfig.describe())
}
```

```text
MyApp v1.0.0
MyApp v1.1.0
```

## How It Actually Works

`class Person(val name: String, var age: Int)` expands, at compile time, into
exactly what you'd hand-write in Java: a `private final String name` field, a
`private int age` field, a constructor that assigns both, a public
`getName()`, and a public `getAge()`/`setAge(int)` pair (`var` gets a setter,
`val` doesn't). Kotlin properties are always backed by this
field-plus-accessors shape at the bytecode level — `person.age = 31` in
Kotlin source literally compiles to an `invokevirtual` call to
`setAge(31)`, not a direct field write, which is why you can later add
validation logic inside `set(value)` without touching any call site.

A computed property like `area` with only a custom `get()` compiles to a
getter method containing that expression's bytecode and **no backing field
at all** — `javap` on `Rectangle.class` shows no `area` field, only a
`getArea()` method, confirming it's recomputed on every access rather than
cached.

`init` blocks and the primary constructor are merged by the compiler into a
single actual JVM constructor: property initializers and `init { ... }`
blocks run in the exact textual order they appear in the class body, all
inside that one generated `<init>` method. A secondary constructor that
delegates via `: this(...)` compiles to a call to that primary `<init>`
method as its very first bytecode instruction (the JVM requires every
constructor to invoke another constructor or `Object`'s before doing
anything else) — so `init` blocks and property initializers always run
first, then the secondary constructor's own body.

`object AppConfig` is compiled to a normal class (`AppConfig.class`) with a
`private` constructor and one `public static final AppConfig INSTANCE`
field, eagerly initialized in a static initializer block (`<clinit>`) the
first time the class is loaded by the JVM's classloader — thread-safe by
construction, since class loading itself is synchronized by the JVM spec.
`AppConfig.describe()` from Kotlin is really `AppConfig.INSTANCE.describe()`
under the hood; the compiler just hides the `.INSTANCE` for you.

## Cheat sheet

| Concept | Syntax |
|---------|--------|
| Class with constructor properties | `class Person(val name: String, var age: Int)` |
| Computed property | `val area: Double get() = width * height` |
| Init block | `init { ... }` |
| Secondary constructor | `constructor(...) : this(...)` |
| Validate arguments | `require(condition) { "message" }` |
| Private member | `private var x` |
| Singleton | `object Name { ... }` |

## Exercise

Write a class `Circle(val radius: Double)` with a computed property `area`
(`Math.PI * radius * radius`) and a computed property `circumference`
(`2 * Math.PI * radius`). Add an `init` block that uses `require` to reject a
non-positive radius. Then write a class `CircleCatalog` (as an `object`
singleton) with a mutable list of circles, a function `addCircle(radius:
Double)`, and a function `totalArea()` that sums the area of every circle
added so far.
