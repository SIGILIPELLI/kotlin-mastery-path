# 05 · Kotlin Multiplatform Basics

Everything so far has targeted the JVM. **Kotlin Multiplatform (KMP)**
lets you share one Kotlin codebase across JVM, JS, Native (iOS, desktop),
and more — sharing business logic, networking, and data models while
platform-specific code (UI, platform APIs) stays separate. This module
covers the `expect`/`actual` mechanism and how a multiplatform Gradle
project is structured, targeting only the JVM here so every example
actually compiles and runs; the same source layout scales to other targets
without changing `commonMain`.

## Project layout: `commonMain` and platform source sets

A KMP module's `build.gradle.kts` declares targets and gives each one its
own source set:

```kotlin
plugins {
    kotlin("multiplatform") version "2.0.20"
}
repositories { mavenCentral() }
kotlin {
    jvmToolchain(17)
    jvm()
    // js(IR) { browser() }       // would add a JS target
    // iosArm64()                  // would add an iOS target
    sourceSets {
        val commonMain by getting
        val jvmMain by getting
    }
}
```

Code lives in `src/commonMain/kotlin` (shared across every target) and
`src/jvmMain/kotlin` (JVM-only). Adding `js(IR) { browser() }` above would
add `src/jsMain/kotlin` alongside it, with the *same* `commonMain` shared
into both.

## `expect`/`actual`: one declaration, many implementations

`commonMain` can *declare* something it needs without knowing how each
platform provides it, using `expect`. Each platform source set supplies the
real implementation with a matching `actual`.

```kotlin
// src/commonMain/kotlin/Common.kt
expect class PlatformInfo() {
    fun name(): String
}

// commonMain code depends only on the expect declaration -- it compiles
// once and would work unmodified against a jsMain/iosMain actual too.
fun greet(info: PlatformInfo): String = "Hello from ${info.name()}"

expect fun currentTimeMillis(): Long
```

```kotlin
// src/jvmMain/kotlin/Jvm.kt
actual class PlatformInfo actual constructor() {
    actual fun name(): String = "JVM ${System.getProperty("java.version")}"
}

actual fun currentTimeMillis(): Long = System.currentTimeMillis()

fun main() {
    val info = PlatformInfo()
    println(greet(info))
    val t1 = currentTimeMillis()
    Thread.sleep(5)
    val t2 = currentTimeMillis()
    println("Elapsed at least: ${t2 - t1 >= 0}")
}
```

```text
Hello from JVM 17.0.5
Elapsed at least: true
```

`greet()` is defined once, in `commonMain`, and never mentions the JVM —
it only knows about the `expect class PlatformInfo`. A hypothetical
`iosMain` `actual class PlatformInfo` (using `NSProcessInfo` instead of
`System.getProperty`) would let this exact same `greet()` function run
unmodified on iOS. `expect`/`actual` classes are still a Beta feature (as
of Kotlin 2.0) — the compiler warns about it, which is expected and safe
to suppress with `-Xexpect-actual-classes` in real projects; plain `expect
fun` (like `currentTimeMillis`) is stable and warning-free.

## Kotlin-specific traps

- **`expect`/`actual` signatures must match exactly** — same parameter
  types, same nullability, same default values (defaults must live on the
  `expect` side only). A tiny mismatch fails to compile with an error
  pointing at the `actual` side, which can be confusing if you didn't
  write the `expect` recently.
- **Not everything needs `expect`/`actual`.** If an API is already
  available identically on every target (most of the Kotlin standard
  library, `kotlinx.coroutines`, `kotlinx.serialization`), just use it
  directly in `commonMain` — reach for `expect`/`actual` only for things
  that are genuinely platform-specific (file I/O, current time, platform
  identification, native APIs).
- **Common code can't reference JVM-only libraries.** Importing
  `java.io.File` or `java.util.Date` inside `src/commonMain` is a compile
  error the moment you add a second target — these are JVM-only. Use
  multiplatform equivalents (`kotlinx-datetime`, `okio`) or `expect`/`actual`
  wrappers instead.
- **Each target needs its own toolchain to actually build.** JS needs
  Node, Native/iOS needs Kotlin/Native (and Xcode, for iOS specifically) —
  a project that "builds" with only `jvm()` configured proves nothing
  about whether the `commonMain` code is genuinely platform-agnostic until
  a second target is added and built for real.
- **Gradle's `getting`/`by getting` source-set delegate syntax is being
  phased out** in favor of `named("jvmMain")` style APIs — expect
  deprecation warnings on current Kotlin Multiplatform Gradle plugin
  versions; they're warnings, not errors, for now.

## Cheat sheet

| Concept | Purpose |
|---|---|
| `commonMain` | Code shared across every target |
| `jvmMain`, `jsMain`, `iosMain`, ... | Platform-specific code and `actual` implementations |
| `expect class`/`expect fun` | Declares an API `commonMain` needs, without an implementation |
| `actual class`/`actual fun` | Supplies the platform-specific implementation |
| `jvm()`, `js(IR) { browser() }`, `iosArm64()` | Declares which targets to build |
| `kotlinx-coroutines-core`, `kotlinx-serialization`, `kotlinx-datetime` | Multiplatform libraries usable directly from `commonMain` |

## Exercise

Add a second `expect`/`actual` pair for reading an environment-style
key-value setting: `expect fun getSetting(key: String): String?`, with a
JVM `actual` backed by `System.getenv(key)`. Write a `commonMain` function
`describeEnvironment()` that reads a `"USER"` (or similar) setting via
`getSetting` and falls back to `"unknown"` if it's null, then call it from
`jvmMain`'s `main()` and print the result.
