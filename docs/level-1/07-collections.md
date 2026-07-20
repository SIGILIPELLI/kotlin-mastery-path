# 07 · Collections

Kotlin's standard library ships with rich, well-designed collection types out
of the box — `List`, `Set`, and `Map` — each with both read-only and mutable
variants. This module covers creating and using them; the powerful
functional operations (`map`, `filter`, `reduce`) get their own deep dive in
[Level 2](../level-2/05-collections-deep-dive.md).

## Lists

A `List` is an ordered collection that allows duplicates. `listOf(...)`
creates a read-only list; `mutableListOf(...)` creates one you can modify.

```kotlin
fun main() {
    val fruits: List<String> = listOf("apple", "banana", "cherry")
    println(fruits)          // [apple, banana, cherry]
    println(fruits[0])       // apple
    println(fruits.size)     // 3
    // fruits.add("date")    // compile error -- List has no add()

    val mutableFruits: MutableList<String> = mutableListOf("apple", "banana")
    mutableFruits.add("cherry")
    mutableFruits.remove("apple")
    println(mutableFruits)   // [banana, cherry]
}
```

```text
[apple, banana, cherry]
apple
3
[banana, cherry]
```

Note that `List` is read-only, not necessarily immutable — it simply doesn't
expose mutating methods. It's a view; whether the underlying data can change
elsewhere depends on what created it.

## Common list operations

```kotlin
fun main() {
    val numbers = listOf(5, 3, 8, 1, 9, 3)

    println(numbers.first())          // 5
    println(numbers.last())           // 3
    println(numbers.contains(8))      // true
    println(numbers.indexOf(8))       // 2
    println(numbers.sorted())         // [1, 3, 3, 5, 8, 9]
    println(numbers.reversed())       // [3, 9, 1, 8, 3, 5]
    println(numbers.max())            // 9
    println(numbers.min())            // 1
    println(numbers.sum())            // 29
    println(numbers.distinct())       // [5, 3, 8, 1, 9]
}
```

```text
5
3
true
2
[1, 3, 3, 5, 8, 9]
[3, 9, 1, 8, 3, 5]
9
1
29
[5, 3, 8, 1, 9]
```

## Sets

A `Set` is an unordered collection with no duplicate elements.

```kotlin
fun main() {
    val uniqueNumbers: Set<Int> = setOf(1, 2, 2, 3, 3, 3)
    println(uniqueNumbers)          // [1, 2, 3] -- duplicates collapsed

    val mutableSet: MutableSet<String> = mutableSetOf("a", "b")
    mutableSet.add("c")
    mutableSet.add("a")             // no-op -- already present
    println(mutableSet)             // [a, b, c]

    val setA = setOf(1, 2, 3)
    val setB = setOf(2, 3, 4)
    println(setA.union(setB))         // [1, 2, 3, 4]
    println(setA.intersect(setB))     // [2, 3]
    println(setA.subtract(setB))      // [1]
}
```

```text
[1, 2, 3]
[a, b, c]
[1, 2, 3, 4]
[2, 3]
[1]
```

## Maps

A `Map` stores key-value pairs, with each key appearing at most once.

```kotlin
fun main() {
    val ages: Map<String, Int> = mapOf("Alice" to 30, "Bob" to 25)
    println(ages)                 // {Alice=30, Bob=25}
    println(ages["Alice"])        // 30
    println(ages["Unknown"])      // null -- missing key returns null, not an exception
    println(ages.getOrDefault("Unknown", 0))   // 0
    println(ages.containsKey("Bob"))            // true

    val mutableAges: MutableMap<String, Int> = mutableMapOf()
    mutableAges["Carol"] = 28
    mutableAges["Dave"] = 35
    mutableAges["Carol"] = 29     // overwrites the previous value
    println(mutableAges)          // {Carol=29, Dave=35}
}
```

```text
{Alice=30, Bob=25}
30
null
0
true
{Carol=29, Dave=35}
```

The `to` in `"Alice" to 30` creates a `Pair` — `mapOf` just takes a `vararg`
of pairs.

## Iterating collections

```kotlin
fun main() {
    val fruits = listOf("apple", "banana", "cherry")
    for (fruit in fruits) {
        println(fruit)
    }

    val ages = mapOf("Alice" to 30, "Bob" to 25)
    for ((name, age) in ages) {
        println("$name is $age")
    }

    // forEach as an alternative to a for loop
    fruits.forEach { fruit -> println("Fruit: $fruit") }
}
```

```text
apple
banana
cherry
Alice is 30
Bob is 25
Fruit: apple
Fruit: banana
Fruit: cherry
```

## Arrays (briefly)

Kotlin also has `Array<T>` (and primitive-specialized versions like
`IntArray`), fixed-size and mutable in place. Lists are preferred for most
everyday code; arrays show up mainly for performance-sensitive code or
interop with Java APIs.

```kotlin
fun main() {
    val numbers = arrayOf(1, 2, 3)
    numbers[0] = 99
    println(numbers.joinToString())   // 99, 2, 3

    val ints = intArrayOf(1, 2, 3)    // specialized, avoids boxing
    println(ints.sum())               // 6
}
```

```text
99, 2, 3
6
```

## Cheat sheet

| Task | Syntax |
|------|--------|
| Read-only list | `listOf(1, 2, 3)` |
| Mutable list | `mutableListOf(1, 2, 3)` |
| Read-only set | `setOf(1, 2, 3)` |
| Read-only map | `mapOf("a" to 1, "b" to 2)` |
| Mutable map | `mutableMapOf<String, Int>()` |
| Access by index | `list[0]` |
| Access map value | `map["key"]` (returns nullable) |
| Add to mutable list | `list.add(value)` |
| Iterate map | `for ((k, v) in map)` |
| Combine key+value | `"key" to value` |

## Exercise

Write a program that builds a `MutableMap<String, MutableList<String>>`
representing students grouped by grade (e.g. `"A" to mutableListOf("Alice",
"Anna")`). Add at least three grades with a couple of students each. Then
print, for each grade in the map, the grade letter and the number of
students in it, followed by their names — sorted alphabetically within each
grade using `sorted()`.
