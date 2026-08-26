# 08 · Android Basics

!!! warning "About this module's verification"
    Every other module in this course has its code blocks actually
    compiled and run (kotlinc, or a real Gradle/Ktor/Exposed project) and
    the printed output pasted in verbatim. **Android code cannot be
    compiled or run that way** — it needs the Android Gradle Plugin, the
    Android SDK's platform jars, and either a real device or an emulator
    to execute an `Activity` or a Composable. That toolchain isn't
    available in this environment beyond the bare SDK components, with no
    running emulator. The Android-framework-specific code below (the
    `Activity`, `ViewModel extends`, and `@Composable` snippets) has been
    written carefully and reviewed line by line against current Android
    API signatures, but **it has not been compiled or executed** — treat
    it as a well-reviewed reference, not a verified-working example, and
    expect to fix minor issues (a missing import, a Gradle plugin version
    mismatch) when you actually build it in Android Studio. The one piece
    of *plain Kotlin* logic in this module (the counter state class) has
    been compiled and run normally, as noted where it appears.

Android apps are ordinary JVM (or Kotlin/Native, for parts of Compose
Multiplatform) programs built on top of the Android framework — Kotlin is
Google's recommended language for Android since 2019. This module covers
the shape of an Android app: activities, the view lifecycle, `ViewModel`
for surviving configuration changes, and a first look at Jetpack Compose,
Android's modern declarative UI toolkit.

## Project shape

An Android module needs the Android Gradle Plugin and a `compileSdk`:

```kotlin
// app/build.gradle.kts
plugins {
    id("com.android.application")
    kotlin("android")
}
android {
    namespace = "com.example.counter"
    compileSdk = 34
    defaultConfig {
        applicationId = "com.example.counter"
        minSdk = 24
        targetSdk = 34
    }
}
dependencies {
    implementation("androidx.core:core-ktx:1.13.1")
    implementation("androidx.lifecycle:lifecycle-viewmodel-ktx:2.8.4")
    implementation("androidx.activity:activity-compose:1.9.1")
    implementation(platform("androidx.compose:compose-bom:2024.06.00"))
    implementation("androidx.compose.material3:material3")
}
```

## Activities and the lifecycle

An `Activity` is a single screen. Android calls fixed lifecycle methods as
the screen is created, shown, hidden, and destroyed — critical for
releasing resources (camera, location, network listeners) at the right
time.

```kotlin
import android.os.Bundle
import androidx.activity.ComponentActivity
import android.util.Log

class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        Log.d("MainActivity", "onCreate")
    }

    override fun onStart() {
        super.onStart()
        Log.d("MainActivity", "onStart -- screen becoming visible")
    }

    override fun onResume() {
        super.onResume()
        Log.d("MainActivity", "onResume -- screen interactive")
    }

    override fun onPause() {
        super.onPause()
        Log.d("MainActivity", "onPause -- release camera/sensors here")
    }

    override fun onDestroy() {
        super.onDestroy()
        Log.d("MainActivity", "onDestroy")
    }
}
```

`onCreate` runs once per Activity *instance* — and a screen rotation
**destroys and recreates** the Activity by default, running `onCreate`
again from scratch. That's the whole reason `ViewModel` exists (next
section): plain fields on an `Activity` don't survive rotation, but a
`ViewModel` does.

## `ViewModel`: state that survives configuration changes

The state-holding logic itself is plain Kotlin and doesn't need the
Android SDK to write or test — only the `class X : ViewModel()` wrapper is
Android-specific.

```kotlin
// Pure Kotlin logic -- this part IS compiled and run below, no Android SDK needed.
class CounterState {
    var count: Int = 0
        private set

    fun increment() { count++ }
    fun reset() { count = 0 }
}

fun main() {
    val state = CounterState()
    repeat(3) { state.increment() }
    println("Count after 3 increments: ${state.count}")
    state.reset()
    println("Count after reset: ${state.count}")
}
```

```text
Count after 3 increments: 3
Count after reset: 0
```

```kotlin
// Android-specific wrapper (reviewed, not executed) -- same logic, but
// surviving Activity re-creation by living in a ViewModel instead.
import androidx.lifecycle.ViewModel
import kotlinx.coroutines.flow.MutableStateFlow
import kotlinx.coroutines.flow.StateFlow
import kotlinx.coroutines.flow.asStateFlow

class CounterViewModel : ViewModel() {
    private val _count = MutableStateFlow(0)
    val count: StateFlow<Int> = _count.asStateFlow()

    fun increment() { _count.value++ }
    fun reset() { _count.value = 0 }
}
```

`ViewModel` instances are retained by the `ViewModelStore` across
configuration changes (rotation, dark-mode toggle) and are only actually
destroyed when the Activity finishes for good — that's the mechanism, and
it works whether the state inside is a plain `var` or, as shown here, a
`StateFlow` for observing changes from the UI.

## Jetpack Compose: declarative UI

Compose replaces XML layouts with functions annotated `@Composable` that
describe UI as a function of state. Recomposition (Compose re-running a
composable) happens automatically when observed state (like a `StateFlow`
collected via `collectAsState()`) changes.

```kotlin
import androidx.compose.runtime.Composable
import androidx.compose.runtime.collectAsState
import androidx.compose.runtime.getValue
import androidx.compose.foundation.layout.Column
import androidx.compose.material3.Button
import androidx.compose.material3.Text

@Composable
fun CounterScreen(viewModel: CounterViewModel) {
    val count by viewModel.count.collectAsState()

    Column {
        Text(text = "Count: $count")
        Button(onClick = { viewModel.increment() }) {
            Text("Increment")
        }
        Button(onClick = { viewModel.reset() }) {
            Text("Reset")
        }
    }
}
```

`by viewModel.count.collectAsState()` is the Compose bridge from a
`StateFlow` to a Compose-observed `State<Int>` — reading `count` inside
`Text(text = "Count: $count")` registers this composable to recompose
whenever the flow emits, without any manual "refresh the UI" call.

## Kotlin-specific traps

- **Configuration changes destroy the Activity, not the `ViewModel`.**
  State stored directly in Activity fields (not a `ViewModel`) is lost on
  rotation — a very common "why did my counter reset" bug for people new
  to Android.
- **`@Composable` functions can be called from any thread Compose
  chooses, potentially many times.** They must be free of side effects
  aside from emitting UI — network calls or mutable state writes belong
  in a `LaunchedEffect` or the `ViewModel`, not directly in the composable
  body.
- **`collectAsState()` needs `import androidx.compose.runtime.getValue`**
  for the `by` delegate syntax to resolve — a surprisingly common missing
  import error, since the delegate operator function lives in a separate
  package from `collectAsState` itself.
- **`Log.d` calls do nothing (and don't crash) outside an Android
  runtime** — unlike `println`, which is why the Android-specific
  snippets above use `Log.d` intentionally, to show idiomatic Android
  logging, but they can't be exercised with plain `kotlinc`.
- **`minSdk`/`targetSdk`/`compileSdk` are three different numbers with
  different meanings** — `compileSdk` is which API surface you compile
  against, `targetSdk` signals which behavior version you've tested for,
  `minSdk` is the oldest OS version the app installs on. Mixing these up
  is a common Gradle configuration mistake.

## Cheat sheet

| Concept | API |
|---|---|
| Single screen | `class X : ComponentActivity()` |
| Lifecycle callbacks | `onCreate`, `onStart`, `onResume`, `onPause`, `onStop`, `onDestroy` |
| Survives rotation | `class X : ViewModel()` |
| Observable state | `MutableStateFlow` + `StateFlow` (or Compose `mutableStateOf`) |
| Declarative UI | `@Composable fun Screen() { ... }` |
| Bridge Flow to Compose | `val x by flow.collectAsState()` |
| Side effects in Compose | `LaunchedEffect(key) { ... }` |

## Exercise

Sketch (don't need to run) a `TodoViewModel` holding a
`MutableStateFlow<List<String>>` of todo items, with `addItem(text:
String)` and `removeItem(index: Int)` methods. Then sketch a `@Composable
TodoScreen` that collects the list, renders each item as a `Text` with a
"Remove" `Button` next to it, and a text field + "Add" button at the
bottom. Focus on getting the state-flow-to-Compose wiring right, since
that's the part this module actually demonstrated working code for.
