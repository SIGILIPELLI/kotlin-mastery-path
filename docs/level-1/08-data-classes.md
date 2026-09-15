---
description: "Data Classes — data class is one of Kotlin's most useful idioms: a single keyword that generates equals(), hashCode(), toString(), and a copy() function…"
---

# 08 · Data Classes

`data class` is one of Kotlin's most useful idioms: a single keyword that
generates `equals()`, `hashCode()`, `toString()`, and a `copy()` function for
a class whose main job is to hold data. This eliminates a huge amount of the
boilerplate that plain Java classes require.

## Declaring a data class

```kotlin
data class Point(val x: Int, val y: Int)

fun main() {
    val p1 = Point(1, 2)
    println(p1)                 // Point(x=1, y=2) -- toString() generated automatically
}
```

```text
Point(x=1, y=2)
```

Compare that `println` output to a regular class, where you'd see something
like `Point@1b6d3586` unless you wrote `toString()` yourself.

## Generated `equals()` and `hashCode()`

Data classes compare by **value** — two instances with the same property
values are equal — unlike regular classes, which compare by reference.

```kotlin
data class Point(val x: Int, val y: Int)

class RegularPoint(val x: Int, val y: Int)

fun main() {
    val p1 = Point(1, 2)
    val p2 = Point(1, 2)
    println(p1 == p2)           // true -- structural equality

    val r1 = RegularPoint(1, 2)
    val r2 = RegularPoint(1, 2)
    println(r1 == r2)           // false -- reference equality (different objects)

    // hashCode() is consistent with equals(), so data classes work correctly as map keys
    val counts = mutableMapOf<Point, Int>()
    counts[Point(1, 2)] = 10
    println(counts[Point(1, 2)])   // 10 -- found by value, not by reference
}
```

```text
true
false
10
```

## `copy()` — modifying one field at a time

`copy()` creates a new instance with the same values, letting you override
just the fields you want to change — essential for working with immutable
(`val`-only) data classes.

```kotlin
data class User(val name: String, val email: String, val age: Int)

fun main() {
    val user = User("Alice", "alice@example.com", 30)

    val birthdayUser = user.copy(age = 31)   // everything else stays the same
    println(user)            // User(name=Alice, email=alice@example.com, age=30)
    println(birthdayUser)    // User(name=Alice, email=alice@example.com, age=31)
}
```

```text
User(name=Alice, email=alice@example.com, age=30)
User(name=Alice, email=alice@example.com, age=31)
```

## Destructuring declarations

Data classes automatically get `component1()`, `component2()`, etc.
(matching constructor parameter order), which enables destructuring into
separate variables in one line.

```kotlin
data class Point(val x: Int, val y: Int)

fun main() {
    val point = Point(3, 4)
    val (x, y) = point   // destructuring -- calls component1()/component2() under the hood
    println("x=$x, y=$y")

    // Very common with maps in a for loop, also destructuring-based
    val ages = mapOf("Alice" to 30, "Bob" to 25)
    for ((name, age) in ages) {
        println("$name: $age")
    }
}
```

```text
x=3, y=4
Alice: 30
Bob: 25
```

## Data classes with default values and validation

Data classes support everything regular classes do, including default
parameter values and `init` block validation.

```kotlin
data class Product(
    val name: String,
    val price: Double,
    val quantity: Int = 1
) {
    init {
        require(price >= 0) { "price cannot be negative" }
        require(quantity >= 0) { "quantity cannot be negative" }
    }

    val total: Double
        get() = price * quantity
}

fun main() {
    val item = Product("Widget", 9.99, 3)
    println(item)          // Product(name=Widget, price=9.99, quantity=3)
    println(item.total)    // 29.97

    val single = Product("Gadget", 19.99)   // quantity defaults to 1
    println(single)
}
```

```text
Product(name=Widget, price=9.99, quantity=3)
29.97
Product(name=Gadget, price=19.99, quantity=1)
```

## How It Actually Works

The `data` modifier is a compile-time instruction to the compiler: "generate
these standard methods for me based on the primary-constructor properties."
`equals()`, `hashCode()`, `toString()`, `copy()`, and `componentN()` are all
ordinary methods written into the `.class` file exactly as if you'd typed
them by hand — nothing about them is a special JVM feature or reflective
trick. You can prove it with `javap -p Product.class` and see plain methods
like `public boolean equals(Object)`.

The generated `equals()` compares every property listed in the primary
constructor (not properties declared in the class body) using `==`, which
for reference types means calling their own `.equals()` — so `Product`'s
`equals` calls `String.equals` on `name`, boxed `Double.equals` on `price`,
and boxed `Integer.equals` on `quantity`. `hashCode()` is generated to be
consistent with that: it combines each property's `hashCode()` using the
classic `31 * result + property.hashCode()` recipe, the same convention
`Objects.hash()` and Java's IDE-generated hashCode use — which matters
because it's what makes `HashMap`/`HashSet` correctly treat two data-class
instances with equal fields as the same key.

`copy()` is not reflective either — the compiler generates a real method
that calls the primary constructor with each parameter defaulted to
`this.propertyName`, and any arguments you pass to `copy()` override just
those defaults. That's why `copy()` can only touch primary-constructor
properties: there's no generic "clone with these fields changed" mechanism,
just a generated call to the same constructor you already have.

`componentN()` functions (`component1()`, `component2()`, ...) are what
destructuring (`val (a, b) = instance`) actually compiles to — `val (x, y) =
point` becomes `val x = point.component1(); val y = point.component2()`
under the hood, resolved purely by position and by the `componentN` naming
convention, which is also why destructuring works on `Pair`, `Map.Entry`,
and any class you manually add `componentN()` functions to, data class or
not.

## When to use a data class vs. a regular class

| Use a data class when... | Use a regular class when... |
|---------------------------|-------------------------------|
| The type mainly holds data | The type has significant behavior/identity beyond its data |
| You want structural equality (`==` by value) | You want reference equality or custom equality rules |
| You'd benefit from `copy()` for updates | Instances represent a mutable, evolving entity (e.g. a live connection) |
| A readable `toString()` is helpful for logging/debugging | You need full control over `toString()`/`equals()` |

## Cheat sheet

| Feature | Generated automatically for a `data class` |
|---------|----------------------------------------------|
| `toString()` | `ClassName(prop1=val1, prop2=val2)` |
| `equals()` / `hashCode()` | Structural (value-based) comparison |
| `copy()` | `instance.copy(prop = newValue)` |
| `componentN()` | Enables `val (a, b) = instance` destructuring |

## 🔀 See this in another language

- [Scala — Pattern Matching Intro](https://sigilipelli.github.io/scala-mastery-path/level-1/08-pattern-matching-intro/)
- [PowerShell — Error Handling Basics](https://sigilipelli.github.io/powershell-mastery-path/level-1/08-error-handling-basics/)
- [C++ — References & Pointers](https://sigilipelli.github.io/cpp-mastery-path/level-1/08-references-pointers/)

## Exercise

Define a data class `Movie(val title: String, val director: String, val year:
Int, val rating: Double = 0.0)`. Create a list of several movies. Use `copy()`
to create an updated version of one movie with a new `rating`. Then write a
function that takes the list, destructures each movie in a loop, and prints
`"<title> (<year>) by <director> -- rating: <rating>"` for each one.
