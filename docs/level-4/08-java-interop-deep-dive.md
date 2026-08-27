# 08 · Java Interop Deep Dive

Almost every real Kotlin codebase touches Java: an existing service being
migrated incrementally, a Java library with no Kotlin equivalent, or a
Kotlin library that needs to be usable from Java callers who've never
seen `suspend` or `data class`. Kotlin was designed for this — but the
seams show up in specific, learnable places. This module calls Java from
Kotlin and Kotlin from Java, compiling both directions for real.

## Calling Java from Kotlin

```java
// JavaLegacy.java
public class JavaLegacy {
    public String name;
    private int score;

    public JavaLegacy(String name, int score) {
        this.name = name;
        this.score = score;
    }

    public int getScore() { return score; }
    public void setScore(int score) { this.score = score; }

    public String maybeNull(boolean giveNull) {
        return giveNull ? null : "present";
    }
}
```

```kotlin
fun main() {
    val legacy = JavaLegacy("Ada", 90)
    // Java getter/setter pair -> Kotlin property syntax
    println("score via property: ${legacy.score}")
    legacy.score = 95
    println("score after set: ${legacy.score}")

    // Platform type: Kotlin can't know from bytecode alone
    // whether this return value is nullable
    val maybe: String? = legacy.maybeNull(true)
    println("maybe is null: ${maybe == null}")
}
```

```console
$ javac JavaLegacy.java
$ kotlinc Interop.kt -cp . -include-runtime -d interop.jar
$ java -cp interop.jar:. InteropKt
score via property: 90
score after set: 95
maybe is null: true
```

Two things happened automatically: `getScore()`/`setScore(int)` became
`legacy.score` (Kotlin recognizes the Java getter/setter naming
convention and synthesizes a property), and `maybeNull`'s return type
showed up as a **platform type** (`String!` in error messages) — neither
definitely nullable nor definitely non-null, because Java's type system
has no annotation-free way to express nullability. Kotlin trusts the
caller to declare the right type (`String?` here) rather than forcing a
choice; declaring it `String` instead would compile fine and then throw
`NullPointerException` at the `println` if `maybeNull` ever actually
returned null — the null-safety guarantee Kotlin gives for its own code
doesn't extend automatically across a Java boundary.

## Calling Kotlin from Java

```kotlin
class Greeter {
    companion object {
        @JvmStatic
        fun staticGreet(name: String): String = "Hello, $name (static)"
    }

    @JvmOverloads
    fun greet(name: String, punctuation: String = "!"): String = "Hi $name$punctuation"
}

@JvmName("createGreeter")
fun factory(): Greeter = Greeter()
```

```java
public class CallFromJava {
    public static void main(String[] args) {
        System.out.println(Greeter.staticGreet("Java"));
        Greeter g = KotlinLibKt.createGreeter();
        System.out.println(g.greet("World"));
        System.out.println(g.greet("World", "?"));
    }
}
```

```console
$ kotlinc KotlinLib.kt -include-runtime -d kotlinlib.jar
$ javac -cp kotlinlib.jar CallFromJava.java
$ java -cp kotlinlib.jar:. CallFromJava
Hello, Java (static)
Hi World!
Hi World?
```

Three annotations did the work: without `@JvmStatic`, a `companion
object` function compiles to an instance method on a synthetic
`Companion` object (`Greeter.Companion.staticGreet(...)`), awkward from
Java — `@JvmStatic` generates a real `static` method instead. Without
`@JvmOverloads`, a default parameter (`punctuation: String = "!"`)
compiles to *one* Java method requiring both arguments — Java has no
concept of default parameters — `@JvmOverloads` generates the overload
(`greet(String)` calling `greet(String, String)`) that lets Java callers
omit it. And top-level functions compile into a file class named
`<Filename>Kt` (`KotlinLib.kt` → `KotlinLibKt`) unless `@JvmName`
overrides it — `factory()` is called as `KotlinLibKt.createGreeter()`
from Java specifically because of the `@JvmName("createGreeter")`
annotation above it; without it, Java would call
`KotlinLibKt.factory()`.

## The platform type trap in more detail

Platform types are the single biggest interop hazard because they're
*invisible* at the call site — nothing in the Kotlin code marks
`legacy.maybeNull(...)` as different from a call to a Kotlin function
that's guaranteed non-null. The only signals are the IDE showing
`String!` in a hover tooltip, and the crash if the assumption is wrong.
Two defenses:

- **Annotate Java APIs you control** with `@Nullable`/`@NonNull` (JSR-305
  or JetBrains annotations) — Kotlin reads these and turns the platform
  type into a real `String?`/`String`, restoring compile-time checking.
- **For Java APIs you don't control** (a third-party library), wrap the
  boundary explicitly: a thin Kotlin adapter function that calls the
  Java method once and returns a properly nullable/non-null Kotlin type,
  so every other Kotlin caller gets real null-safety instead of a fresh
  platform type at every call site.

## Traps

- **`equals`/`hashCode` from a Java class are used as-is** — a Java
  class with a broken or absent `equals` override behaves the same way
  from Kotlin (reference equality via `==` maps to Java's `.equals()` in
  Kotlin, unlike Java where `==` is always reference equality) — Kotlin
  doesn't retroactively fix a Java class's identity semantics.
- **Java's checked exceptions are invisible to Kotlin.** Kotlin has no
  checked exceptions at all, so calling a Java method declared `throws
  IOException` compiles with no `try`/`catch` required — the exception
  can still be thrown at runtime; only the compiler's enforcement is
  gone, so relying on "the compiler would have told me" for Java-thrown
  checked exceptions doesn't work from Kotlin call sites.
- **Java collections crossing into Kotlin are mutable regardless of the
  Kotlin type used to reference them.** A Java method returning
  `List<String>` can be assigned to a Kotlin `val list: List<String>`
  (Kotlin's read-only interface), but the underlying object is still the
  same mutable `ArrayList` — anything else holding a reference to it
  (including the Java code that returned it) can still mutate it out
  from under the Kotlin caller, since `List` vs `MutableList` is a
  Kotlin-only compile-time distinction with no enforcement at the JVM
  bytecode level.
- **`@JvmField` vs. a Kotlin property matters for Java frameworks that
  need direct field access** (some serialization/reflection libraries
  expect a `public` field, not a getter) — a plain Kotlin `val` compiles
  to a private field plus a getter, which such a framework won't find
  unless `@JvmField` is added to expose the field itself.

## Cheat sheet

| Concern | Annotation/pattern |
|---|---|
| Companion function as real Java `static` | `@JvmStatic` |
| Default parameters usable from Java | `@JvmOverloads` |
| Rename the generated file-class or method | `@JvmName("...")` |
| Expose a raw field (not getter) to Java | `@JvmField` |
| Trust a Java API's nullability | `@Nullable`/`@NonNull` on the Java side |
| Unknown Java nullability at a call site | Platform type (`String!`) — wrap it |
| Java getter/setter pair | Kotlin property syntax (`obj.prop`) |

## Exercise

Add a second Java class, `JavaConfig`, with a public method
`public static String env(String key, String fallback)` that returns
`System.getenv(key)` or `fallback` if null. Call it from Kotlin as a
platform type, assign the result to an explicitly-typed `val port:
String` (non-nullable), and set an environment variable when running so
the non-null assumption holds; then remove the environment variable and
observe the `NullPointerException` at the assignment when the platform
type turns out to have been null after all.
