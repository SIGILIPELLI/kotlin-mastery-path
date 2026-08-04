# 01 · OOP Deep Dive

Level 1 covered classes, objects, and data classes at a basic level. Real
Kotlin codebases lean much harder on a handful of OOP idioms that don't exist
in the same form in Java: interfaces with real implementations, `sealed`
hierarchies that the compiler can exhaustively check, and `object` for
singletons. This module goes deep on those, plus the inheritance rules and
traps that show up when combining them with data classes.

## Interfaces with implementation

Kotlin interfaces aren't just method signatures — they can have method
bodies and even properties (as long as the property has no backing field).

```kotlin
interface Greeter {
    val greeting: String              // abstract property -- must be overridden
    fun greet(name: String): String = "$greeting, $name!"   // default implementation
}

class EnglishGreeter : Greeter {
    override val greeting = "Hello"
    // greet() is inherited as-is -- no need to override it
}

class PirateGreeter : Greeter {
    override val greeting = "Ahoy"
    override fun greet(name: String) = "${greeting.uppercase()}, $name, ye scallywag!"
}

fun main() {
    println(EnglishGreeter().greet("Alice"))   // Hello, Alice!
    println(PirateGreeter().greet("Bob"))       // AHOY, Bob, ye scallywag!
}
```

```text
Hello, Alice!
AHOY, Bob, ye scallywag!
```

A class can implement multiple interfaces. If two of them provide the same
default method, Kotlin *forces* you to resolve the ambiguity explicitly with
`super<InterfaceName>` — it won't silently pick one for you.

```kotlin
interface A { fun greet() = println("Hello from A") }
interface B { fun greet() = println("Hello from B") }

class C : A, B {
    override fun greet() {
        super<A>.greet()   // explicitly call A's version
        super<B>.greet()   // and B's version
        println("...and hello from C")
    }
}

fun main() = C().greet()
```

```text
Hello from A
Hello from B
...and hello from C
```

## Abstract classes vs. interfaces

Both let you define a contract with optional shared implementation, so when
do you reach for which?

| | `interface` | `abstract class` |
|---|---|---|
| Multiple inheritance | A class can implement many | A class can extend only one |
| Constructor / state | No constructor, no backing-field state | Full constructor, can hold real state |
| Default methods | Yes | Yes |
| Use when | Defining a capability/role (`Comparable`, `Greeter`) | Modeling a shared base with real fields and invariants |

## Inheritance: `open`, `override`, and visibility

Classes and members are `final` (non-inheritable/non-overridable) by
default in Kotlin — the opposite of Java. You must opt in with `open`.

```kotlin
open class Animal(val name: String) {
    open fun sound(): String = "..."
    fun describe(): String = "$name says ${sound()}"   // not open -- can't be overridden
}

class Dog(name: String) : Animal(name) {
    override fun sound(): String = "Woof"
}

class Cat(name: String) : Animal(name) {
    override fun sound(): String = "Meow"
}

fun main() {
    val animals: List<Animal> = listOf(Dog("Rex"), Cat("Whiskers"))
    for (animal in animals) {
        println(animal.describe())   // dynamic dispatch -- calls the right sound()
    }
}
```

```text
Rex says Woof
Whiskers says Meow
```

`protected` members are visible to subclasses but not to outside code —
useful for internal hooks a subclass needs but callers shouldn't touch:

```kotlin
open class Account(private val owner: String) {
    protected var balance: Double = 0.0

    fun deposit(amount: Double) {
        balance += amount
    }
}

class LoggingAccount(owner: String) : Account(owner) {
    fun depositAndReport(amount: Double) {
        deposit(amount)
        println("Balance is now $balance")   // OK -- protected is visible to subclasses
    }
}
```

## Sealed classes and interfaces

A `sealed` type restricts its subtypes to those declared in the same
package/module — the compiler knows the *complete* set of possibilities,
so a `when` over a sealed type doesn't need an `else` branch to be
exhaustive. This is Kotlin's answer to modeling "one of a fixed set of
cases" more safely than an enum (each case can carry different data).

```kotlin
sealed class NetworkResult
data class Success(val body: String) : NetworkResult()
data class Failure(val code: Int, val message: String) : NetworkResult()
object Loading : NetworkResult()

fun describe(result: NetworkResult): String = when (result) {
    is Success -> "Got: ${result.body}"
    is Failure -> "Error ${result.code}: ${result.message}"
    Loading -> "Still loading..."
    // no `else` needed -- the compiler knows these are the only three subtypes
}

fun main() {
    println(describe(Success("hello")))
    println(describe(Failure(404, "Not Found")))
    println(describe(Loading))
}
```

```text
Got: hello
Error 404: Not Found
Still loading...
```

!!! warning "Exhaustiveness is a feature, not a formality"
    If you later add a fourth subtype of `NetworkResult`, every `when` that
    matches on it *without* an `else` will fail to compile until you handle
    the new case. That's deliberate — it turns "I forgot to handle a new
    state" from a runtime bug into a compile error. Adding a stray `else ->`
    "just in case" throws this safety net away.

A `sealed interface` works the same way and is handy when your cases don't
share a common base implementation:

```kotlin
sealed interface Shape
data class Circle(val radius: Double) : Shape
data class Rectangle(val width: Double, val height: Double) : Shape

fun area(shape: Shape): Double = when (shape) {
    is Circle -> Math.PI * shape.radius * shape.radius
    is Rectangle -> shape.width * shape.height
}
```

## `object` declarations: singletons for free

`object` declares a class *and* its single instance at the same time — no
`getInstance()` boilerplate, no double-checked locking.

```kotlin
object AppConfig {
    var environment: String = "development"
    val version: String = "1.0.0"
}

fun main() {
    AppConfig.environment = "production"
    println("${AppConfig.version} running in ${AppConfig.environment}")
    // There is only ever one AppConfig -- accessing it anywhere gives the same instance
}
```

```text
1.0.0 running in production
```

`companion object` attaches object-like (static-ish) members to a class,
commonly used for factory functions:

```kotlin
class User private constructor(val name: String, val email: String) {
    companion object {
        fun fromEmail(email: String): User {
            val name = email.substringBefore("@").replaceFirstChar { it.uppercase() }
            return User(name, email)
        }
    }
}

fun main() {
    val user = User.fromEmail("alice@example.com")
    println("${user.name} <${user.email}>")   // Alice <alice@example.com>
}
```

```text
Alice <alice@example.com>
```

## Trap: data classes and inheritance don't mix well

A `data class` cannot extend another `data class`, and if it extends a
regular open class, the generated `equals()`/`hashCode()`/`toString()` only
consider properties declared in the data class's *own* primary constructor
— parent-class properties are silently excluded.

```kotlin
open class Entity(val id: Int)

data class Product(val name: String) : Entity(1)

fun main() {
    val p1 = Product("Widget")
    val p2 = Product("Widget")
    println(p1 == p2)   // true -- only `name` is compared, `id` from Entity is ignored!
}
```

```text
true
```

This surprises people who expect `id` to matter. If identity fields live in
a base class, prefer composition (a regular property) over inheritance for
data classes, or override `equals()`/`hashCode()` yourself.

## Cheat sheet

| Concept | Keyword | Key trait |
|---|---|---|
| Capability contract | `interface` | Multiple per class, default methods, no state |
| Shared base with state | `abstract class` | Single per class, can hold fields |
| Fixed set of subtypes | `sealed class` / `sealed interface` | Exhaustive `when`, no `else` needed |
| Singleton | `object` | Exactly one instance, lazily created |
| Static-like members | `companion object` | Attached to the class, not an instance |
| Opt-in overriding | `open` / `override` | Classes/members are final unless marked `open` |

## Exercise

Model a `sealed class PaymentMethod` with three subtypes: `data class
CreditCard(val last4: String)`, `data class PayPal(val email: String)`, and
`object Cash`. Write a function `processPayment(method: PaymentMethod):
String` that uses an exhaustive `when` (no `else`) to return a description
for each case. Then add a fourth subtype and watch every `when` that
handled `PaymentMethod` exhaustively fail to compile until you add a branch
for it.
