# 03 · Multiplatform at Scale

[Level 3's multiplatform module](../level-3/05-multiplatform-basics.md)
introduced `expect`/`actual` with a single JVM target. A real
multiplatform app shares much more than one function: networking, DTOs,
business logic, and often local storage, across Android, iOS, desktop, and
web. This module shares a networking client and a key-value store across
`commonMain`, using multiplatform Ktor and per-platform storage — again
built against only the JVM target so it actually runs, with notes on how
each piece extends to more targets.

```kotlin
kotlin {
    jvmToolchain(17)
    jvm { withJava() }
    // iosArm64(); iosSimulatorArm64()  -- would add iOS, sharing commonMain
    sourceSets {
        val commonMain by getting {
            dependencies {
                implementation("org.jetbrains.kotlinx:kotlinx-coroutines-core:1.8.1")
                implementation("io.ktor:ktor-client-core:2.3.12")
                implementation("io.ktor:ktor-client-content-negotiation:2.3.12")
                implementation("io.ktor:ktor-serialization-kotlinx-json:2.3.12")
                implementation("org.jetbrains.kotlinx:kotlinx-serialization-json:1.6.3")
            }
        }
        val jvmMain by getting {
            dependencies { implementation("io.ktor:ktor-client-cio:2.3.12") }
            // iosMain would instead add ktor-client-darwin here
        }
    }
}
```

Notice `ktor-client-core` (no `-jvm` suffix) in `commonMain` — that's the
multiplatform artifact coordinate; Gradle resolves it to the right
platform-specific jar per source set automatically.

## Sharing a networking client via `expect`/`actual` engines

Ktor's `HttpClient` takes an *engine* — CIO, OkHttp, Darwin (iOS), Js — as
its first argument. `commonMain` code can stay engine-agnostic by asking
for the engine through an `expect` function.

```kotlin
// src/commonMain/kotlin/Common.kt
import io.ktor.client.*
import io.ktor.client.engine.*
import io.ktor.client.plugins.contentnegotiation.*
import io.ktor.serialization.kotlinx.json.*
import kotlinx.serialization.Serializable

@Serializable
data class Repo(val name: String, val stargazers_count: Int)

expect fun httpClientEngine(): HttpClientEngineFactory<*>

class GitHubClient {
    private val client = HttpClient(httpClientEngine()) {
        install(ContentNegotiation) { json() }
    }

    suspend fun topKotlinRepoNames(sampleData: List<Repo>): List<String> =
        sampleData.sortedByDescending { it.stargazers_count }.take(3).map { it.name }

    fun close() = client.close()
}
```

```kotlin
// src/jvmMain/kotlin/Jvm.kt
import io.ktor.client.engine.*
import io.ktor.client.engine.cio.*

actual fun httpClientEngine(): HttpClientEngineFactory<*> = CIO
// An iosMain equivalent would be: actual fun httpClientEngine() = Darwin
```

`GitHubClient` itself — connection setup, JSON config, and the actual
business logic in `topKotlinRepoNames` — is 100% shared. Only the one-line
engine choice is platform-specific, which is the entire point: the
*amount* of `expect`/`actual` code in a well-structured multiplatform app
should be small relative to the shared logic surrounding it.

## Sharing storage the same way

Persisting a value looks completely different per platform (`SharedPreferences`
on Android, `NSUserDefaults` on iOS, `java.util.prefs.Preferences` here on
plain JVM) — another natural `expect`/`actual` boundary.

```kotlin
// commonMain
expect class PlatformStore() {
    fun save(key: String, value: String)
    fun load(key: String): String?
}

// jvmMain
import java.util.prefs.Preferences

actual class PlatformStore actual constructor() {
    private val prefs = Preferences.userRoot().node("kmp-demo")
    actual fun save(key: String, value: String) { prefs.put(key, value) }
    actual fun load(key: String): String? = prefs.get(key, null)
}
```

Running both together:

```text
Top 3 by stars: [ktor, kotlinx.coroutines, compose-multiplatform]
Loaded back: jvm-demo
```

(`GitHubClient.topKotlinRepoNames` sorts sample data rather than hitting
the real GitHub API, to keep this example deterministic and runnable
offline — swapping in a real `client.get(...)` call is a one-line change
once you have network access and an API you want to hit.)

## Kotlin-specific traps

- **Multiplatform Ktor artifact coordinates drop the `-jvm` suffix.**
  `commonMain` uses `io.ktor:ktor-client-core`; a JVM-only module uses
  `io.ktor:ktor-client-core-jvm`. Copy-pasting a dependency line from a
  JVM-only tutorial into a KMP `commonMain` block is a common,
  hard-to-notice mistake (it often still resolves, to the wrong artifact
  metadata) that manifests as bizarre "expected multiplatform" errors.
- **Each engine needs its own dependency, chosen per platform.** CIO and
  OkHttp work on JVM/Android, Darwin only on Apple targets, Js only for
  JS — forgetting to add the right engine dependency to a given platform
  source set fails at compile time with a missing `actual`, or at link
  time for Native targets.
- **A "shared" business logic function that secretly calls a JVM-only
  API compiles fine with a single target** and only breaks the moment a
  second target (iOS, JS) is added — this is why testing multiplatform
  code means actually building for more than one target, not just
  trusting a single-target green build.
- **`expect`/`actual` interfaces need matching *default parameter*
  handling carefully** — defaults belong on the `expect` declaration only;
  putting them on the `actual` side too is a compile error ("actual
  function cannot have default argument values").
- **Intermediate source sets (e.g. a shared `iosMain` above
  `iosArm64Main`/`iosSimulatorArm64Main`) let you share Apple-specific but
  cross-target code** without duplicating it — this hierarchical source
  set structure is exactly how a real large-scale KMP project (Compose
  Multiplatform apps, JetBrains's own libraries) avoids repeating
  `actual` implementations per Apple architecture.

## Cheat sheet

| Concern | Approach |
|---|---|
| Multiplatform artifact naming | Drop the `-jvm`/`-android` suffix in `commonMain` deps |
| Networking across platforms | `expect fun engine(): HttpClientEngineFactory<*>` |
| Storage across platforms | `expect class Store` wrapping each platform's native API |
| Apple-specific shared code | An intermediate `iosMain` source set above per-arch ones |
| Verify true portability | Actually add and build a second target, don't assume |

## Exercise

Add a second `expect`/`actual` pair, `expect fun platformName(): String`,
with the JVM `actual` returning `"JVM " + System.getProperty("os.name")`.
Use it inside `commonMain` to build a `UserAgent` string
(`"MyApp/1.0 (${platformName()})"`) that `GitHubClient` could attach as a
header on real requests, and print it from `jvmMain`'s `main()`.
