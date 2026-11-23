---
title: "Kotlin Cheat Sheet: Quick Reference, Beginner to Advanced"
description: "A free Kotlin cheat sheet: 36 quick-reference tables from every level of the Kotlin Mastery Path, each linked to its full lesson."
---

# Kotlin Cheat Sheet

Quick-reference tables from **36 lessons** of the free Kotlin Mastery Path, in course order from beginner to advanced. Each section links to the full lesson with explanations and runnable examples.

## Level 1 · Entry

### [01 · Setup &amp; First Program](level-1/01-setup.md)

| Task | Command |
|------|---------|
| Check JDK version | `java -version` |
| Check Kotlin compiler version | `kotlinc -version` |
| Compile to a runnable jar | `kotlinc File.kt -include-runtime -d app.jar` |
| Run a jar | `java -jar app.jar` |
| Run a `.kt` file directly | `kotlin File.kt` |
| Start the REPL | `kotlinc` |
| Exit the REPL | `:quit` |

### [02 · Variables &amp; Types](level-1/02-variables-types.md)

| Concept | Syntax |
|---------|--------|
| Read-only variable | `val x = 10` |
| Mutable variable | `var x = 10` |
| Explicit type | `val x: Int = 10` |
| Long literal | `val x: Long = 10L` |
| Float literal | `val x: Float = 1.5f` |
| String template | `"$name is ${age + 1}"` |
| Convert types | `i.toLong()`, `i.toDouble()`, `"42".toInt()` |
| Compile-time constant | `const val NAME = value` |

### [03 · Control Flow](level-1/03-control-flow.md)

| Concept | Syntax |
|---------|--------|
| `if` as expression | `val x = if (cond) a else b` |
| `when` (switch replacement) | `when (x) { 1 -> ...; else -> ... }` |
| `when` with no subject | `when { cond1 -> ...; else -> ... }` |
| `when` with type check | `when (x) { is String -> ...; }` |
| Inclusive range | `1..5` |
| Exclusive range | `1 until 5` |
| Reverse range | `5 downTo 1` |
| Step | `1..10 step 2` |
| Loop with index | `for ((i, v) in list.withIndex())` |
| Labeled continue | `continue@outer` |

### [04 · Functions](level-1/04-functions.md)

| Concept | Syntax |
|---------|--------|
| Basic function | `fun name(param: Type): ReturnType { ... }` |
| Single-expression function | `fun name(param: Type) = expression` |
| Default argument | `fun f(x: Int = 10)` |
| Named argument call | `f(x = 5, y = 10)` |
| Varargs | `fun f(vararg nums: Int)` |
| Spread operator | `f(*array)` |
| Function reference | `::functionName` |
| Function type | `(Int) -> Int` |

### [05 · Classes &amp; Objects Basics](level-1/05-classes-objects.md)

| Concept | Syntax |
|---------|--------|
| Class with constructor properties | `class Person(val name: String, var age: Int)` |
| Computed property | `val area: Double get() = width * height` |
| Init block | `init { ... }` |
| Secondary constructor | `constructor(...) : this(...)` |
| Validate arguments | `require(condition) { "message" }` |
| Private member | `private var x` |
| Singleton | `object Name { ... }` |

### [06 · Null Safety Basics](level-1/06-null-safety.md)

| Operator | Purpose | Example |
|----------|---------|---------|
| `?` | Declare a nullable type | `var x: String? = null` |
| `?.` | Safe call — no-op if receiver is null | `x?.length` |
| `?:` | Elvis — default if left side is null | `x?.length ?: 0` |
| `?:` + `return`/`throw` | Guard clause | `val v = x ?: return` |
| `!!` | Force non-null, throws if actually null | `x!!.length` |
| `as?` | Safe cast — null instead of exception | `obj as? String` |
| smart cast | Compiler narrows type after a null check | `if (x != null) x.length` |

### [07 · Collections](level-1/07-collections.md)

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

### [08 · Data Classes](level-1/08-data-classes.md)

| Feature | Generated automatically for a `data class` |
|---------|----------------------------------------------|
| `toString()` | `ClassName(prop1=val1, prop2=val2)` |
| `equals()` / `hashCode()` | Structural (value-based) comparison |
| `copy()` | `instance.copy(prop = newValue)` |
| `componentN()` | Enables `val (a, b) = instance` destructuring |

### [09 · Extension Functions Intro](level-1/09-extension-functions.md)

| Concept | Syntax |
|---------|--------|
| Extension function | `fun ReceiverType.name(params): ReturnType { ... use this ... }` |
| Extension property (computed only) | `val ReceiverType.name: Type get() = ...` |
| Extension on nullable type | `fun String?.name(): Type { ... }` |
| Resolution | Static — based on declared type, member functions win over extensions |

## Level 2 · Intermediate

### [01 · OOP Deep Dive](level-2/01-oop-deep-dive.md)

| Concept | Keyword | Key trait |
|---|---|---|
| Capability contract | `interface` | Multiple per class, default methods, no state |
| Shared base with state | `abstract class` | Single per class, can hold fields |
| Fixed set of subtypes | `sealed class` / `sealed interface` | Exhaustive `when`, no `else` needed |
| Singleton | `object` | Exactly one instance, lazily created |
| Static-like members | `companion object` | Attached to the class, not an instance |
| Opt-in overriding | `open` / `override` | Classes/members are final unless marked `open` |

### [02 · Lambdas &amp; Higher-Order Functions](level-2/02-lambdas-higher-order-functions.md)

| Syntax | Meaning |
|---|---|
| `(Int) -> String` | Function type: takes an `Int`, returns a `String` |
| `{ x -> x * 2 }` | Lambda literal with named parameter |
| `{ it * 2 }` | Lambda with implicit single parameter |
| `f(x) { ... }` | Trailing lambda -- last function-type param moves outside `()` |
| `::functionName` | Reference to a top-level or member function |
| `inline fun f(block: () -> Unit)` | Lambda body copied inline -- no allocation, allows non-local `return` |

### [03 · Coroutines Basics](level-2/03-coroutines-basics.md)

| Concept | Purpose |
|---|---|
| `suspend fun` | A function that can pause without blocking a thread |
| `runBlocking { }` | Bridge blocking code (like `main`) into coroutine world |
| `launch { }` | Start a coroutine, get a `Job`, no return value |
| `async { }` / `.await()` | Start a coroutine that produces a value, run concurrently |
| `coroutineScope { }` | Structured concurrency -- waits for all children, propagates failure |
| `delay(ms)` | Non-blocking suspend for a duration |
| `isActive` / `ensureActive()` | Cooperative cancellation check in a loop |
| `withContext(NonCancellable)` | Run cleanup that must suspend, even while cancelling |

### [04 · Null Safety Advanced](level-2/04-null-safety-advanced.md)

| Situation | Tool |
|---|---|
| Calling unannotated Java code | Assign to an explicit `T?` immediately, don't trust the platform type |
| Non-null property set later by external code | `lateinit var` |
| Expensive value computed once, lazily | `val x by lazy { ... }` |
| Nullable elements in a non-null collection | `List<T?>` |
| Possibly-absent collection | `List<T>?` |
| Smart cast fails on a `var`/property | Copy to a local `val` first, then null-check |

### [05 · Collections Deep Dive](level-2/05-collections-deep-dive.md)

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

### [06 · Testing with JUnit/Kotest](level-2/06-testing-junit-kotest.md)

| Need | JUnit 5 | Kotest |
|---|---|---|
| Mark a test | `@Test` | A string literal or `test("...")` block |
| Run before each test | `@BeforeEach` | `beforeTest { }` |
| Assert equality | `assertEquals(expected, actual)` | `actual shouldBe expected` |
| Assert an exception | `assertThrows(Type::class.java) { }` | `shouldThrow<Type> { }` |
| Parameterize | `@ParameterizedTest` + `@ValueSource` | `withData(...)` / table tests |
| Test a `suspend fun` | `runTest { }` (from `kotlinx-coroutines-test`) | same `runTest { }` works inside a spec |

### [07 · Working with JSON](level-2/07-working-with-json.md)

| Task | Syntax |
|---|---|
| Mark a type serializable | `@Serializable data class Foo(...)` |
| Object to JSON string | `Json.encodeToString(value)` |
| JSON string to object | `Json.decodeFromString<Foo>(jsonString)` |
| Rename a field for the JSON key | `@SerialName("json_key") val ktName: Type` |
| Optional field with a default | `val field: Type = defaultValue` |
| Nullable field | `val field: Type? = null` |
| Ignore unmodeled JSON fields | `Json { ignoreUnknownKeys = true }` |
| Enum as a JSON string | `@Serializable enum class Foo { A, B }` |

### [08 · Gradle Basics](level-2/08-gradle-basics.md)

| Task | Command / syntax |
|---|---|
| Build everything (compile + test + package) | `./gradlew build` |
| Run just the tests | `./gradlew test` |
| Run the application | `./gradlew run` |
| Clean build outputs | `./gradlew clean` |
| List available tasks | `./gradlew tasks` |
| Add a runtime dependency | `implementation("group:artifact:version")` |
| Add a test-only dependency | `testImplementation("group:artifact:version")` |
| Use JUnit 5 | `tasks.test { useJUnitPlatform() }` |
| Reference a sibling module | `implementation(project(":module-name"))` |

### [09 · Generics](level-2/09-generics.md)

| Concept | Syntax | Meaning |
|---|---|---|
| Generic class/function | `class Box<T>(...)` / `fun <T> f(...)` | Works with any type, checked at compile time |
| Upper bound | `<T : Comparable<T>>` | Restricts `T`, unlocks that type's members |
| Covariance (producer) | `class C<out T>` | `C<Dog>` is a subtype of `C<Animal>` |
| Contravariance (consumer) | `class C<in T>` | `C<Animal>` is usable as `C<Dog>` |
| Star projection | `List<*>` | "A `List` of something," read-only access |
| Reified type parameter | `inline fun <reified T>` | `T` usable in `is T` checks at runtime |

## Level 3 · Advanced

### [01 · Advanced Coroutines (Flow)](level-3/01-advanced-coroutines-flow.md)

| Concept | Type | Key trait |
|---|---|---|
| `flow { emit(x) }` | Cold `Flow<T>` | Runs per collector, from scratch |
| `.map`, `.filter`, `.onEach` | Operators | Suspend-aware, chainable |
| `.catch { }` | Operator | Catches only upstream exceptions |
| `MutableStateFlow(initial)` | Hot, `StateFlow<T>` | Has `.value`, conflates equal values |
| `MutableSharedFlow(replay = n)` | Hot, `SharedFlow<T>` | No current value, replays last `n` |
| `.collect { }` | Terminal operator | Suspends until flow completes (or forever for hot flows) |
| `.toList()`, `.first()`, `.count()` | Terminal operators | Convenience collectors |

### [02 · Building APIs with Ktor](level-3/02-building-apis-ktor.md)

| Concept | Kotlin/Ktor construct |
|---|---|
| Define a route | `get("/path") { }`, `post("/path") { }` in a `routing { }` block |
| Read path param | `call.parameters["id"]` (nullable `String`) |
| Read JSON body | `call.receive<T>()` |
| Send JSON response | `call.respond(value)` (needs `ContentNegotiation` + `json()`) |
| Custom status + body | `call.respond(HttpStatusCode.Created, value)` |
| Map exceptions to responses | `install(StatusPages) { exception<E> { call, e -> ... } }` |
| Call your own/another API | `HttpClient(CIO) { install(ContentNegotiation) { json() } }` |

### [03 · Databases (Exposed/SQLite)](level-3/03-databases-exposed-sqlite.md)

| Concept | Exposed construct |
|---|---|
| Define schema | `object X : IntIdTable() { val col = varchar(...) }` |
| Create tables | `SchemaUtils.create(X)` inside a transaction |
| Wrap DB work | `transaction { ... }` |
| DAO entity | `class E(id) : IntEntity(id) { companion object : IntEntityClass<E>(X) }` |
| Insert (DAO) | `E.new { field = value }` |
| Query (DAO) | `E.find { X.col eq value }`, `E.all()` |
| Query (DSL) | `X.selectAll()`, `X.select(cols).where { }` |
| Update | mutate a fetched entity's `var` property |
| Delete | `entity.delete()` |

### [04 · Design Patterns in Kotlin](level-3/04-design-patterns.md)

| Pattern | Idiomatic Kotlin |
|---|---|
| Singleton | `object Name { ... }` |
| Builder | Chainable methods returning `apply { }`, or named/default args |
| Strategy | `fun interface` + lambda, or plain function type parameter |
| Observer | `MutableList<(T) -> Unit>` + custom property setter |
| State | `sealed class`/`sealed interface` + exhaustive `when` |
| Factory | A top-level or companion `fun create(...): Type` |
| Decorator | Extension functions, or delegation via `by` |

### [05 · Kotlin Multiplatform Basics](level-3/05-multiplatform-basics.md)

| Concept | Purpose |
|---|---|
| `commonMain` | Code shared across every target |
| `jvmMain`, `jsMain`, `iosMain`, ... | Platform-specific code and `actual` implementations |
| `expect class`/`expect fun` | Declares an API `commonMain` needs, without an implementation |
| `actual class`/`actual fun` | Supplies the platform-specific implementation |
| `jvm()`, `js(IR) { browser() }`, `iosArm64()` | Declares which targets to build |
| `kotlinx-coroutines-core`, `kotlinx-serialization`, `kotlinx-datetime` | Multiplatform libraries usable directly from `commonMain` |

### [06 · Testing Advanced](level-3/06-testing-advanced.md)

| Need | Tool |
|---|---|
| Fake an interface | `mockk<T>()` |
| Stub a non-suspend function | `every { mock.fn() } returns value` |
| Stub a suspend function | `coEvery { mock.fn() } returns value` |
| Verify a call happened | `verify { mock.fn() }` / `coVerify { }` |
| Run suspend test code | `= runTest { ... }` |
| Assert an exception is thrown | `assertFailsWith<E> { ... }` |
| Run one test over many inputs | `@ParameterizedTest` + `@CsvSource`/`@ValueSource` |

### [07 · DSLs in Kotlin](level-3/07-dsls-in-kotlin.md)

| Concept | Syntax | Purpose |
|---|---|---|
| Lambda with receiver | `block: Receiver.() -> Unit` | `this`/implicit receiver inside the lambda |
| Type-safe builder | `fun receiver(block: T.() -> Unit): T` | The core DSL-building pattern |
| Infix function | `infix fun T.name(x: X)` | Operator-like call syntax: `a name b` |
| Operator overload | `operator fun T.plus(x: X)` | Reuse `+`, `-`, `[]`, etc. on custom types |
| `@DslMarker` | Annotation on an annotation, applied to builder classes | Restricts implicit receivers to the nearest scope |

### [08 · Android Basics](level-3/08-android-basics.md)

| Concept | API |
|---|---|
| Single screen | `class X : ComponentActivity()` |
| Lifecycle callbacks | `onCreate`, `onStart`, `onResume`, `onPause`, `onStop`, `onDestroy` |
| Survives rotation | `class X : ViewModel()` |
| Observable state | `MutableStateFlow` + `StateFlow` (or Compose `mutableStateOf`) |
| Declarative UI | `@Composable fun Screen() { ... }` |
| Bridge Flow to Compose | `val x by flow.collectAsState()` |
| Side effects in Compose | `LaunchedEffect(key) { ... }` |

### [09 · Performance &amp; Profiling](level-3/09-performance-profiling.md)

| Concern | Fix | When it matters |
|---|---|---|
| Long operator chains | `.asSequence()` before the chain | 3+ chained operations, large collections |
| Numeric collections | `IntArray`/`LongArray`/`DoubleArray` | Hot loops, large numeric datasets |
| Hot higher-order functions | `inline fun` | Called extremely frequently, small body |
| Non-local return needed | Only works in `inline` functions | Early-exit from inside a lambda |
| "Is this actually slow?" | `measureTimeMillis`/`measureNanoTime` + warmup, or a real profiler | Before optimizing anything |

## Level 4 · Master

### [01 · Advanced Coroutines &amp; Structured Concurrency](level-4/01-advanced-coroutines-structured-concurrency.md)

| Concept | API | Failure behavior |
|---|---|---|
| Structured child failure propagates | `coroutineScope { launch { } }` | One failure cancels all siblings |
| Isolated child failures | `supervisorScope { launch { } }` | Failures don't cancel siblings |
| Timeout, throws on expiry | `withTimeout(ms) { }` | `TimeoutCancellationException` |
| Timeout, returns null on expiry | `withTimeoutOrNull(ms) { }` | `null` |
| Unstructured (avoid in app code) | `GlobalScope.launch { }` | No parent, no automatic cancellation |
| Handle otherwise-uncaught failures | `CoroutineExceptionHandler` | Installed on a `supervisorScope`/root scope |

### [02 · Production APIs with Ktor](level-4/02-production-apis-ktor.md)

| Concern | Plugin/API |
|---|---|
| Verify a bearer JWT | `install(Authentication) { jwt("name") { verifier(...); validate { } } }` |
| Protect routes | `authenticate("name") { get("/x") { ... } }` |
| Access the verified token | `call.principal<JWTPrincipal>()` |
| Validate request bodies | `install(RequestValidation) { validate<T> { } }` |
| Map validation failures | `exception<RequestValidationException> { }` in `StatusPages` |
| Throttle a route group | `install(RateLimit) { register(...) }` + `rateLimit(name) { }` |
| Structured request logs | `install(CallLogging) { level = Level.INFO }` |

### [03 · Multiplatform at Scale](level-4/03-multiplatform-at-scale.md)

| Concern | Approach |
|---|---|
| Multiplatform artifact naming | Drop the `-jvm`/`-android` suffix in `commonMain` deps |
| Networking across platforms | `expect fun engine(): HttpClientEngineFactory<*>` |
| Storage across platforms | `expect class Store` wrapping each platform's native API |
| Apple-specific shared code | An intermediate `iosMain` source set above per-arch ones |
| Verify true portability | Actually add and build a second target, don't assume |

### [04 · Security Best Practices](level-4/04-security-best-practices.md)

| Concern | Do | Don't |
|---|---|---|
| Store a password | `BCrypt.withDefaults().hashToString(cost, pw)` | Plaintext, or MD5/SHA-256 alone |
| Generate a token/secret | `SecureRandom` | `Random`/`kotlin.random.Random` |
| Build a query with user input | Exposed DSL (`eq`, `like`), or JDBC `PreparedStatement` | String concatenation into SQL |
| Compare a secret/HMAC | `MessageDigest.isEqual(a, b)` | `==` / `.contentEquals()` |
| Log request data | Mask/exclude sensitive fields | Log raw bodies unconditionally |

### [05 · Testing at Scale &amp; CI](level-4/05-testing-at-scale-ci.md)

| Concern | Approach |
|---|---|
| Assert equality | `assertEquals`/Kotest matchers, not `println` + eyeballing |
| Assert an exception is thrown | `assertThrows<T> { }` |
| Coroutine tests | `runTest` from `kotlinx-coroutines-test`, not `runBlocking` |
| Fail the build on failing tests | Ensure the test process exits non-zero |
| CI trigger | `on: push`/`pull_request` in `.github/workflows/*.yml` |
| Debugging a red CI run | `if: always()` on an artifact-upload step for reports |

### [06 · Deployment (Docker)](level-4/06-deployment-docker.md)

| Concern | Approach |
|---|---|
| Small final image | Multi-stage build, `COPY --from=build` |
| Base image for running | `eclipse-temurin:21-jre-alpine` (JRE, not JDK) |
| Exclude files from build context | `.dockerignore` |
| Cap JVM heap to container limit | `-XX:MaxRAMPercentage=75.0` |
| Map container port to host | `docker run -p HOST:CONTAINER` |
| Don't run as root | `USER appuser` in the Dockerfile |
| Fat jar with all dependencies | `shadowJar`/`-include-runtime` |

### [07 · Performance at Scale](level-4/07-performance-at-scale.md)

| Concern | Approach |
|---|---|
| Avoid boxing in hot numeric loops | Primitive arrays/loops, not `ArrayList<Long>` |
| Large collection + early exit | `asSequence()` before `.first()`/`.take()` |
| Small collection, consumes everything | Plain `List` chain — `Sequence` overhead not worth it |
| Type-safe wrapper, zero allocation | `value class` |
| Measure before optimizing | `measureNanoTime`, or JMH for rigor |
| Repeated string building in a loop | `StringBuilder`, not `+=` |

### [08 · Java Interop Deep Dive](level-4/08-java-interop-deep-dive.md)

| Concern | Annotation/pattern |
|---|---|
| Companion function as real Java `static` | `@JvmStatic` |
| Default parameters usable from Java | `@JvmOverloads` |
| Rename the generated file-class or method | `@JvmName("...")` |
| Expose a raw field (not getter) to Java | `@JvmField` |
| Trust a Java API's nullability | `@Nullable`/`@NonNull` on the Java side |
| Unknown Java nullability at a call site | Platform type (`String!`) — wrap it |
| Java getter/setter pair | Kotlin property syntax (`obj.prop`) |

### [09 · Code Quality (detekt/ktlint)](level-4/09-code-quality.md)

| Concern | Tool/command |
|---|---|
| Style/formatting check | `ktlint <file>` or `./gradlew ktlintCheck` |
| Auto-fix formatting | `ktlint -F <file>` |
| Static analysis (complexity, smells) | `detekt` or `./gradlew detekt` |
| Grandfather existing violations | detekt `baseline.xml` |
| Tune rule thresholds | `detekt.yml` |
| Fail CI on violations | Both tasks exit non-zero by default |
| Catch issues before PR | Pre-commit hook or IDE plugin |

---

Go deeper with the full [Kotlin Mastery Path](index.md) curriculum or test yourself with the [interview questions](interview-questions.md).
