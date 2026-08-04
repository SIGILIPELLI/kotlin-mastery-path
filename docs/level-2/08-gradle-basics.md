# 08 · Gradle Basics

Every module so far has run a single `.kt` file straight through `kotlinc`.
Real projects need more: managing dependencies (like the
`kotlinx-coroutines` and `kotlinx-serialization` libraries used in
[Module 3](03-coroutines-basics.md) and [Module 7](07-working-with-json.md)),
compiling multiple source files together, running tests, and packaging a
runnable artifact. **Gradle** is the build tool that does all of this for
the vast majority of Kotlin projects, configured through `build.gradle.kts`
— a build script that's itself just Kotlin.

## Project layout

A standard Gradle Kotlin project follows this structure, matching what
[Module 10's](10-project-weather-cli.md) capstone uses:

```text
weather-cli/
    build.gradle.kts       -- what to build and what it depends on
    settings.gradle.kts    -- project name, module list
    gradlew, gradlew.bat   -- the "Gradle wrapper" -- see below
    src/
        main/kotlin/       -- your application code
        test/kotlin/       -- your test code
```

## The Gradle wrapper

You almost never install Gradle globally. Instead, projects ship a
**wrapper** (`gradlew`/`gradlew.bat` plus a small `gradle/wrapper/`
directory) that downloads the exact Gradle version the project was built
with, the first time it runs. This means everyone on a team — and CI —
builds with an identical Gradle version without installing anything by
hand.

```bash
./gradlew build     # on macOS/Linux
gradlew.bat build    # on Windows
```

A fresh project gets its wrapper via `gradle wrapper` (using a
system-installed Gradle just once) or, more commonly, by generating the
project with `gradle init` or an IDE's "New Project" flow, which sets up
the wrapper automatically.

## A minimal `build.gradle.kts`

```kotlin
// build.gradle.kts
plugins {
    kotlin("jvm") version "2.4.10"   // compiles Kotlin for the JVM
    application                       // adds `run` and packaging tasks
}

version = "1.0.0"

repositories {
    mavenCentral()   // where dependencies get downloaded from
}

dependencies {
    implementation("org.jetbrains.kotlinx:kotlinx-coroutines-core:1.11.0")
    testImplementation(kotlin("test"))
}

application {
    mainClass.set("com.example.MainKt")
}

tasks.test {
    useJUnitPlatform()   // run tests with the JUnit 5 platform
}
```

`settings.gradle.kts` just names the project (and, in a multi-module
build, lists the modules):

```kotlin
// settings.gradle.kts
rootProject.name = "weather-cli"
```

## Dependency configurations

The `dependencies { }` block uses different keywords depending on *where*
a dependency is needed:

| Configuration | Meaning |
|---|---|
| `implementation(...)` | Needed to compile and run your code; not exposed to other modules that depend on yours |
| `api(...)` | Like `implementation`, but *is* exposed to consumers of your module (library authors only) |
| `testImplementation(...)` | Only needed for compiling/running tests (JUnit, Kotest) |
| `runtimeOnly(...)` | Needed only at runtime, not for compiling (e.g. a JDBC driver) |

Most application code only ever needs `implementation` and
`testImplementation` — reach for `api` only when you're publishing a
library that re-exposes a dependency's types in its own public API.

## Common tasks

Gradle organizes work into **tasks** — the plugins you apply (`kotlin("jvm")`,
`application`) register a standard set for you.

```bash
./gradlew build    # compile, run tests, and package -- the full pipeline
./gradlew test     # just run tests
./gradlew run      # run the application (needs the `application` plugin)
./gradlew clean    # delete build outputs
./gradlew tasks    # list every task available in this project
```

```text
$ ./gradlew run

> Task :run
Fetching weather for London...
Temperature: 19.4°C, Wind: 11.9 km/h

BUILD SUCCESSFUL in 2s
2 actionable tasks: 2 executed
```

## Custom tasks

You can define your own tasks directly in `build.gradle.kts` when the
built-in ones aren't enough.

```kotlin
tasks.register("printVersion") {
    doLast {
        println("Building version ${project.version}")
    }
}
```

```bash
./gradlew printVersion
```

```text
> Task :printVersion
Building version 1.0.0
```

## Multi-module projects

As a codebase grows, splitting it into modules (e.g. a `core` library and
a `cli` that depends on it) keeps build times down — Gradle only rebuilds
the modules whose inputs actually changed. Each module gets its own
`build.gradle.kts`; the root `settings.gradle.kts` lists them all.

```kotlin
// settings.gradle.kts
rootProject.name = "weather-app"
include("core", "cli")
```

```kotlin
// cli/build.gradle.kts
dependencies {
    implementation(project(":core"))   // depend on the sibling module, not a remote artifact
}
```

## Version catalogs (keeping dependency versions in one place)

Larger projects centralize dependency versions in
`gradle/libs.versions.toml` instead of hardcoding version strings in every
module's `build.gradle.kts` — one file to bump when a library releases a
new version.

```toml
# gradle/libs.versions.toml
[versions]
coroutines = "1.11.0"

[libraries]
kotlinx-coroutines-core = { module = "org.jetbrains.kotlinx:kotlinx-coroutines-core", version.ref = "coroutines" }
```

```kotlin
// build.gradle.kts
dependencies {
    implementation(libs.kotlinx.coroutines.core)
}
```

## Cheat sheet

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

## Exercise

Starting from the minimal `build.gradle.kts` above, add a
`testImplementation` dependency on `io.kotest:kotest-runner-junit5:<version>`
(from [Module 6](06-testing-junit-kotest.md)) alongside the existing JUnit
setup, and add a custom task named `hello` that prints a greeting including
`project.name`. Run `./gradlew tasks` to confirm your new task shows up,
then run it with `./gradlew hello`.
