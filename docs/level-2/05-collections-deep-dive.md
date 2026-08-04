# 05 · Collections Deep Dive

[Level 1](../level-1/07-collections.md) covered `List`, `Set`, and `Map`
themselves. This module is about the functional operations that make
Kotlin collections so pleasant to work with — chains of `map`/`filter`/
`reduce` that replace hand-written loops — plus **sequences**, which change
*when* those operations run and can matter a lot for performance.

## `map`, `filter`, and chaining

`map` transforms each element; `filter` keeps only elements matching a
predicate. Both return a new collection, so they chain naturally.

```kotlin
data class Product(val name: String, val price: Double, val inStock: Boolean)

fun main() {
    val products = listOf(
        Product("Widget", 9.99, true),
        Product("Gadget", 24.99, false),
        Product("Gizmo", 14.50, true),
        Product("Doohickey", 5.00, true)
    )

    val affordableInStockNames = products
        .filter { it.inStock }
        .filter { it.price < 20.0 }
        .map { it.name }

    println(affordableInStockNames)   // [Widget, Gizmo, Doohickey]
}
```

```text
[Widget, Gizmo, Doohickey]
```

## `reduce` and `fold`

Both combine every element into a single value, but `fold` takes an
explicit starting value and `reduce` uses the first element as its
starting point.

```kotlin
fun main() {
    val prices = listOf(9.99, 24.99, 14.50, 5.00)

    val total = prices.fold(0.0) { acc, price -> acc + price }
    println(total)   // 54.48

    val maxPrice = prices.reduce { acc, price -> if (price > acc) price else acc }
    println(maxPrice)   // 24.99

    // reduce throws on an empty collection -- there's no "first element" to start from
    val empty = emptyList<Double>()
    // empty.reduce { acc, x -> acc + x }   // throws UnsupportedOperationException
    println(empty.fold(0.0) { acc, x -> acc + x })   // 0.0 -- fold handles empty fine
}
```

```text
54.48
24.99
0.0
```

`fold` is the safer default: it always has a well-defined answer for an
empty collection, since you supply the starting value yourself.

## `groupBy`, `associateBy`, and `partition`

These reshape a flat list into a map or a pair of lists — extremely common
for reporting and bucketing.

```kotlin
data class Employee(val name: String, val department: String, val salary: Int)

fun main() {
    val employees = listOf(
        Employee("Alice", "Engineering", 95000),
        Employee("Bob", "Sales", 60000),
        Employee("Carol", "Engineering", 105000),
        Employee("Dave", "Sales", 65000)
    )

    val byDept = employees.groupBy { it.department }
    println(byDept.keys)                       // [Engineering, Sales]
    println(byDept["Engineering"]?.map { it.name })   // [Alice, Carol]

    val byName = employees.associateBy { it.name }
    println(byName["Bob"]?.salary)              // 60000

    val (highEarners, others) = employees.partition { it.salary > 70000 }
    println(highEarners.map { it.name })        // [Alice, Carol]
    println(others.map { it.name })              // [Bob, Dave]
}
```

```text
[Engineering, Sales]
[Alice, Carol]
60000
[Alice, Carol]
[Bob, Dave]
```

## `flatMap`: flattening nested collections

`map` alone would leave you with a list of lists; `flatMap` merges them
into one flat list.

```kotlin
data class Order(val id: Int, val items: List<String>)

fun main() {
    val orders = listOf(
        Order(1, listOf("apple", "banana")),
        Order(2, listOf("cherry")),
        Order(3, listOf("date", "elderberry"))
    )

    val allItems = orders.flatMap { it.items }
    println(allItems)   // [apple, banana, cherry, date, elderberry]

    // compare to plain map, which would give a List<List<String>>:
    println(orders.map { it.items })   // [[apple, banana], [cherry], [date, elderberry]]
}
```

```text
[apple, banana, cherry, date, elderberry]
[[apple, banana], [cherry], [date, elderberry]]
```

## Sorting with `sortedBy` and comparators

```kotlin
data class Person(val name: String, val age: Int)

fun main() {
    val people = listOf(Person("Bob", 25), Person("Alice", 30), Person("Carol", 25))

    println(people.sortedBy { it.age })                 // youngest first
    println(people.sortedByDescending { it.age })        // oldest first

    // sort by age, then by name for ties -- compareBy supports multiple keys
    val sorted = people.sortedWith(compareBy({ it.age }, { it.name }))
    println(sorted)
}
```

```text
[Person(name=Bob, age=25), Person(name=Carol, age=25), Person(name=Alice, age=30)]
[Person(name=Alice, age=30), Person(name=Bob, age=25), Person(name=Carol, age=25)]
[Person(name=Bob, age=25), Person(name=Carol, age=25), Person(name=Alice, age=30)]
```

## Sequences: lazy evaluation

Every operation on a regular `List` (`map`, `filter`, ...) runs *eagerly*
and builds a brand-new intermediate list right away. Chain several of them
and you allocate one throwaway list per step. `asSequence()` switches to
**lazy** evaluation: nothing runs until you ask for a final result (like
`.toList()`, `.first()`, or `.sum()`), and elements flow through the whole
chain one at a time instead of one full pass per operation.

```kotlin
fun main() {
    val numbers = (1..1_000_000).toList()

    // Eager: filter builds a full intermediate list of ~500,000 elements,
    // THEN map builds another full list from that, THEN first() looks at it.
    val eagerResult = numbers.filter { it % 2 == 0 }.map { it * it }.first()

    // Lazy: each number flows through filter -> map one at a time; evaluation
    // stops the instant the first match is found -- no full intermediate lists.
    val lazyResult = numbers.asSequence()
        .filter { it % 2 == 0 }
        .map { it * it }
        .first()

    println(eagerResult)   // 4
    println(lazyResult)    // 4 -- same answer, far less allocation for large inputs
}
```

```text
4
4
```

!!! warning "Sequences aren't automatically faster for everything"
    For small collections, or when you need to fully process every element
    anyway (e.g. `.map { }.toList()` over 10 items), the eager `List`
    version is simpler and just as fast — sequence setup has its own small
    overhead. Reach for `asSequence()` when you have a *long chain* of
    operations over a *large* collection, especially if an early operation
    (like `first()`, `find()`, or `take()`) can short-circuit before
    processing everything.

## Cheat sheet

| Operation | Purpose |
|---|---|
| `map { }` | Transform each element |
| `filter { }` | Keep elements matching a predicate |
| `fold(initial) { acc, x -> }` | Combine into one value, safe on empty collections |
| `reduce { acc, x -> }` | Combine into one value, using the first element as the start (throws on empty) |
| `groupBy { }` | Bucket elements into a `Map<Key, List<Element>>` |
| `associateBy { }` | Build a `Map<Key, Element>` (last one wins on duplicate keys) |
| `partition { }` | Split into a `Pair` of (matching, non-matching) lists |
| `flatMap { }` | Map then flatten one level of nesting |
| `sortedBy { }` / `sortedWith(compareBy(...))` | Sort by one or more keys |
| `asSequence()` | Switch to lazy, single-pass evaluation for a chain |

## Exercise

Given a `List<Order>` where `data class Order(val customer: String, val
total: Double, val isPaid: Boolean)`, write code that: groups orders by
`customer` using `groupBy`, then for each customer computes their total
*paid* revenue using `filter` + `fold` (or `sumOf`), and finally prints
customers sorted by revenue descending. Then rewrite the pipeline using
`asSequence()` and confirm you get the same result.
