---
title: "Kotlin Interview Questions and Answers (180 Questions)"
description: "180 Kotlin interview questions with answers and explanations, from beginner to advanced, organised by level. Free, from the Kotlin Mastery Path."
---

# Kotlin Interview Questions and Answers

**180 questions** with the correct answer and a short explanation, organised from beginner (Level 1) to advanced (Level 4). Every question comes from a lesson in the free Kotlin Mastery Path, so you can open the lesson to go deeper.

!!! tip "Practise, don't just read"
    Each lesson also ends with an interactive quiz, and the [final quiz](quiz.md) tests all four levels. Try to answer each question below before reading the answer.

## Level 1 · Entry

### [01 · Setup &amp; First Program](level-1/01-setup.md)

**Q1.** What is Kotlin?

**Answer:** A modern statically typed language for the JVM (and native/JS) fully interoperable with Java

Concise and null-safe.

**Q2.** What is the entry point?

**Answer:** fun main()

Top-level function.

**Q3.** Which tool compiles Kotlin?

**Answer:** kotlinc (or Gradle's Kotlin plugin)

Targets JVM bytecode.

**Q4.** What does Kotlin run on by default?

**Answer:** The JVM

Also native and JS targets.

### [02 · Variables &amp; Types](level-1/02-variables-types.md)

**Q5.** What does this print?

```kotlin
fun main() {
val a = 7
var b = 2.5
b += a
println("$a $b ${a / 2} ${a / 2.0}")
}
```

**Answer:** 7 9.5 3 3.5

val is read-only, var is mutable; integer division truncates. Verified by compiling and running it.

**Q6.** What does this print?

```kotlin
fun main() {
val x: Long = 3
val y = x.toInt() + 1
println("$y ${"abc".length} ${'a' + 1}")
}
```

**Answer:** 4 3 b

Explicit conversions; Char + Int gives Char. Verified by compiling and running it.

**Q7.** What does type inference mean?

**Answer:** The compiler deduces types from initializers

val n = 5 is Int.

**Q8.** Are numeric types implicitly widened?

**Answer:** No: convert explicitly (toLong())

Strict.

### [03 · Control Flow](level-1/03-control-flow.md)

**Q9.** What does this print?

```kotlin
fun main() {
val n = 3
val s = when (n) { 1 -> "one"; in 2..4 -> "few"; else -> "many" }
var t = 0
for (i in 1..10 step 3) t += i
println("$s $t")
}
```

**Answer:** few 22

when is an expression; ranges and step. Verified by compiling and running it.

**Q10.** What does this print?

```kotlin
fun main() {
val r = if (5 > 3) "yes" else "no"
var i = 0
while (i < 3) i++
println("$r $i ${(1..5).sum()} ${(5 downTo 1).toList()}")
}
```

**Answer:** yes 3 15 [5, 4, 3, 2, 1]

if is an expression. Verified by compiling and running it.

**Q11.** What does `..` create?

**Answer:** An inclusive range

..&lt; is exclusive.

**Q12.** Is there a ternary operator?

**Answer:** No: use if/else as an expression

Elvis is for null.

### [04 · Functions](level-1/04-functions.md)

**Q13.** What does this print?

```kotlin
fun add(a: Int, b: Int): Int { return a + b }
fun greet(name: String, greeting: String = "Hello") = "$greeting, $name!"
fun sq(x: Int) = x * x
fun main() {
println("${add(2, 3)} ${greet("Ada")} ${greet("Bo", "Hi")} ${sq(4)}")
}
```

**Answer:** 5 Hello, Ada! Hi, Bo! 16

Default and expression-body functions. Verified by compiling and running it.

**Q14.** What does this print?

```kotlin
fun describe(times: Int, name: String) = name.repeat(times)
fun main() {
println(describe(name = "x", times = 2))
}
```

**Answer:** xx

Named arguments. Verified by compiling and running it.

**Q15.** What does Unit mean?

**Answer:** The return type of functions returning nothing useful

Like void.

**Q16.** What's vararg?

**Answer:** A parameter accepting any number of arguments

fun f(vararg n: Int).

### [05 · Classes &amp; Objects Basics](level-1/05-classes-objects.md)

**Q17.** What does this print?

```kotlin
class Person(val name: String, var age: Int) { val isAdult get() = age >= 18; fun birthday() { age++ } }
fun main() {
val p = Person("Ada", 36)
p.birthday()
println("${p.name} ${p.age} ${p.isAdult}")
}
```

**Answer:** Ada 37 true

Primary constructor properties and computed property. Verified by compiling and running it.

**Q18.** What does this print?

```kotlin
open class Animal { open fun speak() = "..." }
class Dog : Animal() { override fun speak() = "Woof" }
fun main() {
val a = Animal()
val d: Animal = Dog()
println("${a.speak()} ${d.speak()}")
}
```

**Answer:** ... Woof

Classes are final by default; open and override are explicit. Verified by compiling and running it.

**Q19.** Are Kotlin classes open for inheritance by default?

**Answer:** No: they're final unless marked open

Safer design.

**Q20.** What does `init` do?

**Answer:** Runs initialization code when an instance is created

Primary constructor blocks.

### [06 · Null Safety Basics](level-1/06-null-safety.md)

**Q21.** What does this print?

```kotlin
fun main() {
val s: String? = null
val t: String? = "hi"
println("${s?.length} ${s ?: "none"} ${t!!.length} ${t?.uppercase()}")
}
```

**Answer:** null none 2 HI

Safe call, Elvis and non-null assertion. Verified by compiling and running it.

**Q22.** What does this print?

```kotlin
fun main() {
val x: Any = "text"
if (x is String) println(x.length)
val n: Int? = "12".toIntOrNull()
val m: Int? = "x".toIntOrNull()
println("$n $m")
}
```

**Answer:** 4
12 null

Smart casts and safe conversions. Verified by compiling and running it.

**Q23.** What does `String?` mean?

**Answer:** A String that may be null

Null safety is in the type.

**Q24.** When does `!!` throw?

**Answer:** When the value is null (NullPointerException)

Avoid it.

### [07 · Collections](level-1/07-collections.md)

**Q25.** What does this print?

```kotlin
fun main() {
val l = mutableListOf(3, 1, 2)
l.add(9); l.sort()
val m = mapOf("a" to 1, "b" to 2)
val s = setOf(1, 2, 2, 3)
println("$l ${m["b"]} ${s.size} ${l.filter { it > 1 }.map { it * 2 }}")
}
```

**Answer:** [1, 2, 3, 9] 2 3 [4, 6, 18]

Collections and functional operators. Verified by compiling and running it.

**Q26.** What does this print?

```kotlin
fun main() {
val xs = listOf("apple", "bob", "cat", "avocado")
println(xs.groupBy { it.first() }.mapValues { it.value.size })
println(xs.partition { it.length > 3 })
println(xs.sumOf { it.length })
}
```

**Answer:** {a=2, b=1, c=1}
([apple, avocado], [bob, cat])
18

groupBy, partition and sumOf. Verified by compiling and running it.

**Q27.** What's the difference between List and MutableList?

**Answer:** List is read-only; MutableList can be modified

Read-only view.

**Q28.** Are collection operations like map eager or lazy?

**Answer:** Eager on collections; use asSequence() for lazy

Sequences.

### [08 · Data Classes](level-1/08-data-classes.md)

**Q29.** What does this print?

```kotlin
data class User(val name: String, val age: Int)
fun main() {
val a = User("ada", 30)
val b = a.copy(age = 31)
val (n, ag) = b
println("$a $b ${a == User("ada", 30)} $n $ag")
}
```

**Answer:** User(name=ada, age=30) User(name=ada, age=31) true ada 31

Data classes generate equals, toString, copy and destructuring. Verified by compiling and running it.

**Q30.** What do data classes generate?

**Answer:** equals, hashCode, toString, copy and componentN

Boilerplate-free.

**Q31.** Can data classes be inherited?

**Answer:** They can't be open, abstract, sealed or inner

Use interfaces.

**Q32.** What requires at least one primary constructor parameter?

**Answer:** A data class

val/var parameters.

### [09 · Extension Functions Intro](level-1/09-extension-functions.md)

**Q33.** What does this print?

```kotlin
fun String.capitalizeWord() = replaceFirstChar { it.uppercase() }
fun <T> List<T>.secondOrNull(): T? = if (size >= 2) this[1] else null
fun main() {
println("kotlin".capitalizeWord() + " " + listOf(1, 2, 3).secondOrNull() + " " + emptyList<Int>().secondOrNull())
}
```

**Answer:** Kotlin 2 null

Extension functions add members without inheritance. Verified by compiling and running it.

**Q34.** How are extensions resolved?

**Answer:** Statically, by the declared type

They don't modify the class.

**Q35.** Can an extension access private members?

**Answer:** No, only public API

Not part of the class.

**Q36.** What does an extension property need?

**Answer:** A custom getter (no backing field)

val String.last get() = ...

### [10 · Project — CLI To-Do App](level-1/10-project-todo-app.md)

**Q37.** What does the todo app combine?

**Answer:** Classes, collections, data classes, null safety and file persistence

Level 1 project.

**Q38.** How persist todos simply?

**Answer:** Serialize to a text/JSON file and read on startup

File I/O.

**Q39.** How parse commands safely?

**Answer:** Use toIntOrNull and handle invalid input

Null safety.

**Q40.** How structure the code?

**Answer:** Model data classes, a repository and a small UI loop

Separation.

## Level 2 · Intermediate

### [01 · OOP Deep Dive](level-2/01-oop-deep-dive.md)

**Q41.** What does this print?

```kotlin
sealed interface Shape
class Circle(val r: Double) : Shape
class Square(val s: Double) : Shape
fun area(s: Shape): Double = when (s) { is Circle -> 3.0 * s.r * s.r; is Square -> s.s * s.s }
class Counter private constructor() { companion object { private var n = 0; fun created(): Int { n++; n++; return n } } }
fun main() {
val s: Shape = Circle(1.0)
val q: Shape = Square(2.0)
println("${area(s)} ${area(q)} ${Counter.created()}")
}
```

**Answer:** 3.0 4.0 2

Sealed hierarchies enable exhaustive when; companion objects hold statics. Verified by compiling and running it.

**Q42.** What does `object` declare?

**Answer:** A singleton

Also companion.

**Q43.** What is `sealed` for?

**Answer:** Restricting subclasses so `when` can be exhaustive

Closed hierarchies.

**Q44.** How does Kotlin handle multiple inheritance?

**Answer:** Classes extend one class but can implement many interfaces

Interfaces can hold implementations.

### [02 · Lambdas &amp; Higher-Order Functions](level-2/02-lambdas-higher-order-functions.md)

**Q45.** What does this print?

```kotlin
fun apply2(x: Int, f: (Int) -> Int) = f(x)
fun main() {
val inc = { x: Int -> x + 1 }
val twice: (Int, (Int) -> Int) -> Int = { x, f -> f(f(x)) }
println("${inc(1)} ${twice(3, inc)} ${listOf(1,2,3).fold(0) { a, b -> a + b }} ${apply2(5) { it * it }}")
}
```

**Answer:** 2 5 6 25

Lambdas and higher-order functions. Verified by compiling and running it.

**Q46.** What does this print?

```kotlin
fun main() {
val sb = StringBuilder().apply { append("a"); append("b") }
val n = "hi".let { it.length }
println("$sb $n ${with(sb) { length }} ${sb.also { it.append("c") }}")
}
```

**Answer:** ab 2 2 abc

Scope functions apply, let, with, also. Verified by compiling and running it.

**Q47.** What is `it`?

**Answer:** The implicit single parameter name in a lambda

Readable short lambdas.

**Q48.** What does inline do for higher-order functions?

**Answer:** Inlines the function and lambda to avoid allocation

Performance.

### [03 · Coroutines Basics](level-2/03-coroutines-basics.md)

**Q49.** What are coroutines?

**Answer:** Lightweight concurrent computations that can suspend without blocking threads

kotlinx.coroutines.

**Q50.** What does `suspend` mean?

**Answer:** The function can pause and resume without blocking a thread

Only callable from coroutines.

**Q51.** What does launch return?

**Answer:** A Job (fire and forget)

async returns Deferred.

**Q52.** What starts a blocking bridge in main?

**Answer:** runBlocking

Tests/main.

### [04 · Null Safety Advanced](level-2/04-null-safety-advanced.md)

**Q53.** What does this print?

```kotlin
fun readName(): String? = null
fun main() {
val name: String? = readName()
val len = name?.length ?: -1
val list: List<String?> = listOf("a", null, "b")
println("$len ${list.filterNotNull()} ${list.mapNotNull { it?.uppercase() }}")
}
```

**Answer:** -1 [a, b] [A, B]

Elvis and filterNotNull. Verified by compiling and running it.

**Q54.** What's a platform type?

**Answer:** A Java type with unknown nullability (T!)

Interop caution.

**Q55.** What does lateinit allow?

**Answer:** A non-null var initialized later (throws if read early)

Injection.

**Q56.** What does `by lazy` do?

**Answer:** Initializes a val on first access, thread-safe by default

Delegated property.

### [05 · Collections Deep Dive](level-2/05-collections-deep-dive.md)

**Q57.** What does this print?

```kotlin
fun main() {
val xs = (1..10).toList()
println(xs.chunked(4))
println(xs.windowed(3, step = 4))
println(xs.zip(xs.drop(1)).take(2))
println(xs.asSequence().filter { it % 2 == 0 }.map { it * it }.first())
}
```

**Answer:** [[1, 2, 3, 4], [5, 6, 7, 8], [9, 10]]
[[1, 2, 3], [5, 6, 7]]
[(1, 2), (2, 3)]
4

Collection helpers and lazy sequences. Verified by compiling and running it.

**Q58.** What do sequences avoid?

**Answer:** Intermediate collections (lazy, element-at-a-time)

Large pipelines.

**Q59.** What does groupingBy().eachCount() do?

**Answer:** Counts items per group

Frequency map.

**Q60.** What's flatMap for?

**Answer:** Mapping to collections and flattening the result

Nested lists.

### [06 · Testing with JUnit/Kotest](level-2/06-testing-junit-kotest.md)

**Q61.** Which frameworks test Kotlin?

**Answer:** JUnit 5 and Kotest (plus MockK for mocking)

Idiomatic DSLs.

**Q62.** What's MockK?

**Answer:** A Kotlin-native mocking library

Handles final classes.

**Q63.** What does runTest do?

**Answer:** Runs suspending code in tests with virtual time

kotlinx-coroutines-test.

**Q64.** What makes tests readable in Kotest?

**Answer:** Spec styles like StringSpec/BehaviorSpec

Descriptive.

### [07 · Working with JSON](level-2/07-working-with-json.md)

**Q65.** Which library serializes JSON idiomatically?

**Answer:** kotlinx.serialization (or Jackson/Moshi)

@Serializable.

**Q66.** What does @Serializable require?

**Answer:** The serialization plugin and annotated classes

Compile-time.

**Q67.** How handle unknown JSON keys?

**Answer:** ignoreUnknownKeys = true in the Json config

Robust parsing.

**Q68.** How model optional fields?

**Answer:** Nullable types with defaults

Compatibility.

### [08 · Gradle Basics](level-2/08-gradle-basics.md)

**Q69.** What does build.gradle.kts use?

**Answer:** The Kotlin DSL (type-safe build scripts)

IDE support.

**Q70.** What does the kotlin plugin do?

**Answer:** Compiles Kotlin sources and configures tasks

Plugins.

**Q71.** What does ./gradlew test do?

**Answer:** Runs tests using the project's Gradle wrapper

Reproducible.

**Q72.** What is dependency scope implementation?

**Answer:** A dependency not exposed to consumers' compile classpath

Encapsulation.

### [09 · Generics](level-2/09-generics.md)

**Q73.** What does this print?

```kotlin
fun <T : Comparable<T>> maxOf2(a: T, b: T) = if (a >= b) a else b
class Box<T>(val value: T) { fun <R> map(f: (T) -> R) = Box(f(value)) }
fun main() {
println("${maxOf2(3, 7)} ${maxOf2("b", "a")} ${Box(5).map { it * 2 }.value}")
}
```

**Answer:** 7 b 10

Bounded generics. Verified by compiling and running it.

**Q74.** What does `out T` mean?

**Answer:** Covariance: the type is only produced (List&lt;out T&gt;)

Declaration-site variance.

**Q75.** What does `in T` mean?

**Answer:** Contravariance: the type is only consumed

Comparable&lt;in T&gt;.

**Q76.** What's type erasure's effect?

**Answer:** Generic types aren't available at runtime (use reified with inline)

reified T.

### [10 · Project — Weather CLI](level-2/10-project-weather-cli.md)

**Q77.** What does the weather CLI project combine?

**Answer:** HTTP client, JSON parsing, null safety, coroutines and tests

Level 2 project.

**Q78.** How test the HTTP layer?

**Answer:** Fake the client interface or use a mock server

Deterministic.

**Q79.** How handle failures?

**Answer:** Return a sealed Result type with error cases

Explicit.

**Q80.** Where store API keys?

**Answer:** Environment variables, not source

Security.

## Level 3 · Advanced

### [01 · Advanced Coroutines (Flow)](level-3/01-advanced-coroutines-flow.md)

**Q81.** What is Flow?

**Answer:** A cold asynchronous stream of values

kotlinx.coroutines.

**Q82.** What does a cold flow do?

**Answer:** Starts producing when collected

Each collector re-runs it.

**Q83.** StateFlow vs SharedFlow?

**Answer:** StateFlow holds the latest value; SharedFlow broadcasts events with configurable replay

UI state vs events.

**Q84.** How handle backpressure?

**Answer:** Operators like buffer, conflate and collectLatest

Slow collectors.

### [02 · Building APIs with Ktor](level-3/02-building-apis-ktor.md)

**Q85.** What is Ktor?

**Answer:** A Kotlin framework for asynchronous HTTP servers and clients

Coroutine-based.

**Q86.** How are Ktor routes defined?

**Answer:** In the routing { get("/x") { ... } } DSL

Type-safe.

**Q87.** What are plugins in Ktor?

**Answer:** Installable features like ContentNegotiation and CallLogging

install(...).

**Q88.** How return JSON?

**Answer:** call.respond(obj) with ContentNegotiation and serialization

Auto.

### [03 · Databases (Exposed/SQLite)](level-3/03-databases-exposed-sqlite.md)

**Q89.** What is Exposed?

**Answer:** A Kotlin SQL library with a DSL and DAO API

JetBrains.

**Q90.** How avoid SQL injection in Exposed?

**Answer:** Use the DSL/parameters rather than string concatenation

Parameterized.

**Q91.** What does transaction { } do?

**Answer:** Runs database operations atomically

Commit/rollback.

**Q92.** What's SQLite good for?

**Answer:** Embedded, file-based databases for small apps and tests

Zero setup.

### [04 · Design Patterns in Kotlin](level-3/04-design-patterns.md)

**Q93.** Which pattern replaces a Java Singleton?

**Answer:** object declaration

Language support.

**Q94.** What replaces many Builder uses?

**Answer:** Named/default arguments and apply blocks

Concise.

**Q95.** What does `by` do for classes?

**Answer:** Delegates interface implementation to another object

Decorator.

**Q96.** What's a sealed class's use in state machines?

**Answer:** Exhaustive when over states

Safe.

### [05 · Kotlin Multiplatform Basics](level-3/05-multiplatform-basics.md)

**Q97.** What is Kotlin Multiplatform?

**Answer:** Sharing Kotlin code across JVM, Android, iOS, JS and native

Share logic.

**Q98.** What are expect/actual declarations?

**Answer:** Platform-specific implementations of a common API

Per target.

**Q99.** What do you typically share?

**Answer:** Business logic, networking and data models

UI may be native.

**Q100.** What source sets exist?

**Answer:** commonMain plus per-platform main sets

Gradle.

### [06 · Testing Advanced](level-3/06-testing-advanced.md)

**Q101.** What does MockK's coEvery do?

**Answer:** Stubs suspending functions

Coroutines.

**Q102.** What does Turbine test?

**Answer:** Flows by awaiting emitted items

Flow testing.

**Q103.** What's property-based testing?

**Answer:** Checking invariants over generated inputs (Kotest property)

More coverage.

**Q104.** What should integration tests use?

**Answer:** Real dependencies in containers (Testcontainers)

Realism.

### [07 · DSLs in Kotlin](level-3/07-dsls-in-kotlin.md)

**Q105.** What does this DSL build?

```kotlin
class Page { val sb = StringBuilder()
 fun h1(t: String) { sb.append("<h1>$t</h1>") }
 fun ul(block: Ul.() -> Unit) { sb.append("<ul>"); sb.append(Ul().apply(block).sb); sb.append("</ul>") } }
class Ul { val sb = StringBuilder(); fun li(t: String) { sb.append("<li>$t</li>") } }
fun page(block: Page.() -> Unit) = Page().apply(block).sb.toString()
fun main() {
val html = page { h1("Hi"); ul { li("a"); li("b") } }
println(html)
}
```

**Answer:** &lt;h1&gt;Hi&lt;/h1&gt;&lt;ul&gt;&lt;li&gt;a&lt;/li&gt;&lt;li&gt;b&lt;/li&gt;&lt;/ul&gt;

Lambdas with receivers create type-safe builders. Verified by compiling and running it.

**Q106.** What makes DSLs possible?

**Answer:** Lambdas with receivers, extension functions and infix/operator functions

Gradle Kotlin DSL.

**Q107.** What does @DslMarker do?

**Answer:** Prevents accidental access to outer receivers

Scope control.

**Q108.** When write a DSL?

**Answer:** For readable configuration or construction APIs

Don't overdo it.

### [08 · Android Basics](level-3/08-android-basics.md)

**Q109.** What's Jetpack Compose?

**Answer:** Android's declarative UI toolkit written in Kotlin

@Composable.

**Q110.** What is a ViewModel for?

**Answer:** Holding UI state that survives configuration changes

Lifecycle.

**Q111.** What does `remember` do?

**Answer:** Keeps a value across recompositions

Compose state.

**Q112.** Which scope runs coroutines in a ViewModel?

**Answer:** viewModelScope

Lifecycle-aware.

### [09 · Performance &amp; Profiling](level-3/09-performance-profiling.md)

**Q113.** First step in optimizing Kotlin?

**Answer:** Profile (JFR/profilers) before changing code

Evidence.

**Q114.** What reduces allocation in lambdas?

**Answer:** inline functions and primitive-specialized types

Avoid boxing.

**Q115.** What's a value class?

**Answer:** A wrapper type erased at runtime to avoid allocation

@JvmInline.

**Q116.** What causes hidden allocations?

**Answer:** Boxing, iterators and capturing lambdas in hot paths

Check bytecode.

### [10 · Project — REST API + Database Service](level-3/10-project-rest-api-db.md)

**Q117.** What does the REST API + DB project combine?

**Answer:** Ktor routes, Exposed/SQLite, serialization and tests

Level 3 project.

**Q118.** How validate request bodies?

**Answer:** Check fields and return 400 with errors

Safe APIs.

**Q119.** How test Ktor routes?

**Answer:** testApplication with an in-memory database

Fast.

**Q120.** How structure it?

**Answer:** Routes, services and repositories

Layers.

## Level 4 · Master

### [01 · Advanced Coroutines &amp; Structured Concurrency](level-4/01-advanced-coroutines-structured-concurrency.md)

**Q121.** What is structured concurrency?

**Answer:** Coroutines are scoped to a parent so lifetimes, errors and cancellation propagate

coroutineScope.

**Q122.** What happens when a child coroutine fails?

**Answer:** The parent is cancelled (unless supervisorScope)

Fail fast.

**Q123.** Why avoid GlobalScope?

**Answer:** Coroutines outlive their owner and leak

Use scoped.

**Q124.** How make cancellation cooperative?

**Answer:** Check isActive or call suspending functions

Suspension points.

### [02 · Production APIs with Ktor](level-4/02-production-apis-ktor.md)

**Q125.** What do production Ktor services need?

**Answer:** Config, logging, metrics, health checks, graceful shutdown

Operations.

**Q126.** How handle errors uniformly?

**Answer:** StatusPages plugin mapping exceptions to responses

Consistency.

**Q127.** How secure endpoints?

**Answer:** Authentication plugin (JWT/OAuth) and authorization checks

Defense.

**Q128.** How configure per environment?

**Answer:** application.conf/env variables

12-factor.

### [03 · Multiplatform at Scale](level-4/03-multiplatform-at-scale.md)

**Q129.** What helps share code at scale in KMP?

**Answer:** Clear module boundaries and a small expect/actual surface

Maintainability.

**Q130.** What's a KMP pitfall?

**Answer:** Leaking platform types into common code

Abstract.

**Q131.** How test shared code?

**Answer:** commonTest plus platform tests for actuals

Coverage.

**Q132.** How publish libraries?

**Answer:** Maven publications per target with Gradle

Distribution.

### [04 · Security Best Practices](level-4/04-security-best-practices.md)

**Q133.** How avoid SQL injection in Kotlin?

**Answer:** Parameterized queries/DSL

Never string-build SQL.

**Q134.** How store passwords?

**Answer:** Slow salted hashes (bcrypt/argon2)

Never reversible.

**Q135.** What does dependency scanning find?

**Answer:** Known vulnerable libraries

OWASP check.

**Q136.** Where keep secrets?

**Answer:** Environment/secret manager, never in code

Rotate.

### [05 · Testing at Scale &amp; CI](level-4/05-testing-at-scale-ci.md)

**Q137.** What should CI run for Kotlin?

**Answer:** ktlint/detekt, tests and build

Gate merges.

**Q138.** What's detekt?

**Answer:** Static analysis for Kotlin code smells

Quality.

**Q139.** What's ktlint?

**Answer:** A linter/formatter enforcing the Kotlin style

Consistent.

**Q140.** How speed Gradle in CI?

**Answer:** Build cache and dependency caching

Efficiency.

### [06 · Deployment (Docker)](level-4/06-deployment-docker.md)

**Q141.** Why multi-stage Docker builds?

**Answer:** Build with the JDK and run on a slim JRE

Smaller images.

**Q142.** What does the Ktor Gradle plugin provide?

**Answer:** A runnable fat jar task

buildFatJar.

**Q143.** Why non-root containers?

**Answer:** Limit damage if compromised

Least privilege.

**Q144.** What's a health endpoint for?

**Answer:** Liveness/readiness probes

Orchestration.

### [07 · Performance at Scale](level-4/07-performance-at-scale.md)

**Q145.** What first when tuning at scale?

**Answer:** Measure with profiling and metrics

Evidence.

**Q146.** What improves throughput in Ktor?

**Answer:** Non-blocking I/O, connection pools and avoiding blocking calls on event loops

Dispatchers.

**Q147.** Which dispatcher suits blocking IO?

**Answer:** Dispatchers.IO

Isolate blocking.

**Q148.** What does conflate do on flows?

**Answer:** Drops intermediate values for slow collectors

Latest only.

### [08 · Java Interop Deep Dive](level-4/08-java-interop-deep-dive.md)

**Q149.** How does Kotlin expose top-level functions to Java?

**Answer:** As static methods in a FileNameKt class (or @file:JvmName)

Interop.

**Q150.** What does @JvmStatic do?

**Answer:** Generates a static method for companion/object members

Java callers.

**Q151.** What does @JvmOverloads do?

**Answer:** Generates overloads for default parameters

Java lacks defaults.

**Q152.** What do Java callers see for Kotlin nullability?

**Answer:** Annotations and Intrinsics checks

@Nullable/@NotNull.

### [09 · Code Quality (detekt/ktlint)](level-4/09-code-quality.md)

**Q153.** What practices improve code quality?

**Answer:** Small functions, immutability, tests, linting and code review

Idiomatic Kotlin.

**Q154.** Why prefer val and immutable collections?

**Answer:** Fewer bugs from shared mutable state

Safer.

**Q155.** What does `require` do?

**Answer:** Validates arguments, throwing IllegalArgumentException

Preconditions.

**Q156.** What's a code smell in Kotlin?

**Answer:** Overusing !! and platform types

Null safety lost.

### [10 · Capstone Project](level-4/10-capstone-project.md)

**Q157.** What does the capstone combine?

**Answer:** Ktor API, persistence, coroutines, tests, CI and Docker

Production service.

**Q158.** How prove readiness?

**Answer:** Tests, load checks, monitoring and rollback

Operate.

**Q159.** What documentation helps?

**Answer:** README, architecture notes and runbooks

Onboarding.

**Q160.** What sustains the project?

**Answer:** CI, dependency updates and ownership

Maintenance.

## Mixed questions across all levels

**Q161.** What does this print?

```kotlin
fun main() {
val a = 7
var b = 2.5
b += a
println("$b ${a / 2} ${a / 2.0}")
}
```

**Answer:** 9.5 3 3.5

val is read-only; integer division truncates. Verified by compiling and running it.

**Q162.** What does this print?

```kotlin
fun main() {
val s: String? = null
println("${s?.length} ${s ?: "none"} ${"hi".length}")
}
```

**Answer:** null none 2

Safe call and Elvis. Verified by compiling and running it.

**Q163.** What does this print?

```kotlin
data class User(val name: String, val age: Int)
fun main() {
val u = User("ada", 30)
println("${u.copy(age = 31)} ${u == User("ada", 30)}")
}
```

**Answer:** User(name=ada, age=31) true

Data classes generate equals, toString and copy. Verified by compiling and running it.

**Q164.** What does this print?

```kotlin
fun main() {
val n = 3
println(when (n) { 1 -> "one"; in 2..4 -> "few"; else -> "many" })
}
```

**Answer:** few

when is an expression with ranges. Verified by compiling and running it.

**Q165.** What does this print?

```kotlin
fun String.cap() = replaceFirstChar { it.uppercase() }
fun <T> List<T>.second(): T = this[1]
fun main() {
println("kotlin".cap() + " " + listOf(1,2,3).second())
}
```

**Answer:** Kotlin 2

Extension functions add members without inheritance. Verified by compiling and running it.

**Q166.** What does this print?

```kotlin
fun main() {
val xs = listOf("apple", "bob", "cat", "avocado")
println(xs.groupBy { it.first() }.mapValues { it.value.size })
}
```

**Answer:** {a=2, b=1, c=1}

groupBy and mapValues. Verified by compiling and running it.

**Q167.** What does this print?

```kotlin
fun main() {
val xs = (1..10).toList()
println(xs.asSequence().filter { it % 2 == 0 }.map { it * it }.first())
}
```

**Answer:** 4

Sequences are lazy and stop at the first match. Verified by compiling and running it.

**Q168.** What does this print?

```kotlin
sealed interface Shape
class Circle(val r: Double) : Shape
class Square(val s: Double) : Shape
fun area(s: Shape): Double = when (s) { is Circle -> 3.0 * s.r * s.r; is Square -> s.s * s.s }
fun main() {
val s: Shape = Square(2.0)
println(area(s))
}
```

**Answer:** 4.0

Exhaustive when over a sealed type. Verified by compiling and running it.

**Q169.** Why are Kotlin classes final by default?

**Answer:** Safer design: inheritance must be opted into with open

Effective Java advice built in.

**Q170.** When does `!!` throw?

**Answer:** When the value is null (NullPointerException); prefer safe calls and Elvis

Avoid it.

**Q171.** What does `suspend` mean?

**Answer:** The function can pause without blocking a thread and may only be called from a coroutine or another suspend function

Coroutines.

**Q172.** A coroutine started with GlobalScope outlives its screen and leaks. What's the fix?

**Answer:** Use a structured scope tied to the lifecycle (viewModelScope/coroutineScope)

Structured concurrency.

**Q173.** A child coroutine fails. What happens to its parent scope by default?

**Answer:** The parent and siblings are cancelled (use supervisorScope to isolate)

Fail-fast propagation.

**Q174.** Which dispatcher should blocking file/DB calls use?

**Answer:** Dispatchers.IO

Keep the event loop free.

**Q175.** A Flow collector is slower than the producer. Which operator helps?

**Answer:** buffer or conflate (or collectLatest)

Backpressure.

**Q176.** A StateFlow vs SharedFlow for UI state?

**Answer:** StateFlow: it always holds the latest value

State vs events.

**Q177.** A Ktor endpoint throws and clients see a stack trace. What do you install?

**Answer:** StatusPages to map exceptions to safe, consistent responses

Error handling.

**Q178.** A SQL query is built with string templates from user input. What's wrong?

**Answer:** SQL injection; use parameterized queries/the Exposed DSL

Never concatenate.

**Q179.** Kotlin code called from Java needs default parameters as overloads. Which annotation?

**Answer:** @JvmOverloads

Interop.

**Q180.** What do Detekt and ktlint do in CI?

**Answer:** Static analysis and style enforcement that fail the build on violations

Quality gates.

## Practical coding questions

### A todo repository with a sealed Result

Implement TaskRepo.add (reject blank titles, trim, assign ids), complete (copy the task with done = true, or an error for unknown ids) and summary using partition. Results are a sealed Result type handled with an exhaustive when.

Starter code:

```
sealed interface Result<out T> {
    data class Ok<T>(val value: T) : Result<T>
    data class Err(val message: String) : Result<Nothing>
}

data class Task(val id: Int, val title: String, val done: Boolean = false)

class TaskRepo {
    private val tasks = mutableListOf<Task>()
    private var nextId = 1

    fun add(title: String): Result<Task> {
        // blank titles -> Result.Err("title must not be blank"); otherwise trim, assign nextId++, store, Result.Ok(task)
        return Result.Err("todo")
    }

    fun complete(id: Int): Result<Task> {
        // unknown id -> Result.Err("no task with id N"); otherwise replace the task with copy(done = true)
        return Result.Err("todo")
    }

    fun summary(): String {
        // "open=[titles] done=[titles]" using partition
        return ""
    }
}

fun describe(r: Result<Task>): String = when (r) {
    is Result.Ok -> "ok: ${r.value}"
    is Result.Err -> "error: ${r.message}"
}

fun main() {
    val repo = TaskRepo()
    println(describe(repo.add("write quiz")))
    println(describe(repo.add("   ")))
    println(describe(repo.add("publish")))
    println(describe(repo.complete(1)))
    println(describe(repo.complete(9)))
    println(repo.summary())
}
```

Expected output:

```text
ok: Task(id=1, title=write quiz, done=false)
error: title must not be blank
ok: Task(id=2, title=publish, done=false)
ok: Task(id=1, title=write quiz, done=true)
error: no task with id 9
open=[publish] done=[write quiz]
```

??? success "Show a solution"

    ```
    sealed interface Result<out T> {
        data class Ok<T>(val value: T) : Result<T>
        data class Err(val message: String) : Result<Nothing>
    }

    data class Task(val id: Int, val title: String, val done: Boolean = false)

    class TaskRepo {
        private val tasks = mutableListOf<Task>()
        private var nextId = 1

        fun add(title: String): Result<Task> {
            if (title.isBlank()) return Result.Err("title must not be blank")
            val t = Task(nextId++, title.trim())
            tasks += t
            return Result.Ok(t)
        }

        fun complete(id: Int): Result<Task> {
            val i = tasks.indexOfFirst { it.id == id }
            if (i < 0) return Result.Err("no task with id $id")
            tasks[i] = tasks[i].copy(done = true)
            return Result.Ok(tasks[i])
        }

        fun summary(): String {
            val (done, open) = tasks.partition { it.done }
            return "open=${open.map { it.title }} done=${done.map { it.title }}"
        }
    }

    fun describe(r: Result<Task>): String = when (r) {
        is Result.Ok -> "ok: ${r.value}"
        is Result.Err -> "error: ${r.message}"
    }

    fun main() {
        val repo = TaskRepo()
        println(describe(repo.add("write quiz")))
        println(describe(repo.add("   ")))
        println(describe(repo.add("publish")))
        println(describe(repo.complete(1)))
        println(describe(repo.complete(9)))
        println(repo.summary())
    }
    ```

    A sealed Result makes failure explicit and forces callers to handle every case, data classes give copy for immutable updates, and partition splits done/open in one pass. Verified by compiling and running the solution with kotlinc.

### Extensions, grouping and pagination

Write the extension pageOf(page, size) with argument checks, and topSpenders(orders, n) that totals each customer's spend, sorts highest first (ties by name) and formats name=12.34.

Starter code:

```
fun <T> List<T>.pageOf(page: Int, size: Int): List<T> {
    // require page >= 1 and size >= 1 (message "page and size must be >= 1"); return that page of the list
    return emptyList()
}

data class Order(val customer: String, val item: String, val qty: Int, val price: Double)

fun Order.total() = qty * price

fun topSpenders(orders: List<Order>, n: Int): List<String> {
    // total per customer, highest first (ties by name), top n, formatted "name=12.34"
    return emptyList()
}

fun main() {
    val orders = listOf(
        Order("ada", "pen", 3, 1.5), Order("bo", "book", 1, 12.0), Order("ada", "book", 1, 12.0),
        Order("cy", "pen", 10, 1.5), Order("bo", "pen", 2, 1.5),
    )
    println(topSpenders(orders, 2))
    println((1..10).toList().pageOf(2, 4))
    println((1..10).toList().pageOf(3, 4))
    println((1..10).toList().pageOf(4, 4))
    try { listOf(1).pageOf(0, 5) } catch (e: IllegalArgumentException) { println("error: ${e.message}") }
    println(orders.map { it.total() }.average())
}
```

Expected output:

```text
[ada=16.50, bo=15.00]
[5, 6, 7, 8]
[9, 10]
[]
error: page and size must be >= 1
9.3
```

??? success "Show a solution"

    ```
    fun <T> List<T>.pageOf(page: Int, size: Int): List<T> {
        require(page >= 1 && size >= 1) { "page and size must be >= 1" }
        return drop((page - 1) * size).take(size)
    }

    data class Order(val customer: String, val item: String, val qty: Int, val price: Double)

    fun Order.total() = qty * price

    fun topSpenders(orders: List<Order>, n: Int): List<String> =
        orders.groupBy { it.customer }
            .mapValues { (_, os) -> os.sumOf { it.total() } }
            .entries.sortedWith(compareByDescending<Map.Entry<String, Double>> { it.value }.thenBy { it.key })
            .take(n)
            .map { "%s=%.2f".format(it.key, it.value) }

    fun main() {
        val orders = listOf(
            Order("ada", "pen", 3, 1.5), Order("bo", "book", 1, 12.0), Order("ada", "book", 1, 12.0),
            Order("cy", "pen", 10, 1.5), Order("bo", "pen", 2, 1.5),
        )
        println(topSpenders(orders, 2))
        println((1..10).toList().pageOf(2, 4))
        println((1..10).toList().pageOf(3, 4))
        println((1..10).toList().pageOf(4, 4))
        try { listOf(1).pageOf(0, 5) } catch (e: IllegalArgumentException) { println("error: ${e.message}") }
        println(orders.map { it.total() }.average())
    }
    ```

    Extension functions keep helpers close to the types they extend, require() validates preconditions, and groupBy/mapValues/sortedWith express the report declaratively. Verified by compiling and running the solution with kotlinc.

---

More free practice: the [Kotlin Mastery Path](index.md) curriculum, the [Interview Prep Pack](https://learnmasterypath.com/interview-pack/) and the [free resume ATS check](https://learnmasterypath.com/resume/).
