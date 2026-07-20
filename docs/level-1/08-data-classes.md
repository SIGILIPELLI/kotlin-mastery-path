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

## Exercise

Define a data class `Movie(val title: String, val director: String, val year:
Int, val rating: Double = 0.0)`. Create a list of several movies. Use `copy()`
to create an updated version of one movie with a new `rating`. Then write a
function that takes the list, destructures each movie in a loop, and prints
`"<title> (<year>) by <director> -- rating: <rating>"` for each one.
