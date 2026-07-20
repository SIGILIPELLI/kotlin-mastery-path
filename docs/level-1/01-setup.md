# 01 · Setup & First Program

Kotlin runs on the Java Virtual Machine (JVM), so you need a JDK installed
before you can install Kotlin itself. This module gets your toolchain working
and walks through your very first program, both as a compiled file and in the
interactive REPL.

## Installing the JDK

Kotlin needs a JDK (Java Development Kit) underneath it — the Kotlin compiler
produces JVM bytecode, and the `kotlin` command launches a JVM to run it.

```bash
# macOS (Homebrew)
brew install openjdk@21

# Ubuntu/Debian
sudo apt install openjdk-21-jdk

# Verify
java -version
# openjdk version "21.0.x" ...
```

## Installing the Kotlin compiler

```bash
# macOS (Homebrew)
brew install kotlin

# SDKMAN (works on macOS/Linux, manages multiple versions cleanly)
curl -s "https://get.sdkman.io" | bash
sdk install kotlin

# Verify
kotlinc -version
# Kotlin version 2.0.x ...
```

Most people never touch `kotlinc` directly in real projects — Gradle or an
IDE (IntelliJ IDEA, which has first-class Kotlin support since JetBrains
builds both) drives the compiler for you. But knowing the raw command-line
workflow first makes everything built on top of it easier to understand.

## Your first program

Kotlin's entry point is a top-level function named `main` — no wrapping
class required, unlike Java.

```kotlin
// Hello.kt
fun main() {
    println("Hello, Kotlin!")
}
```

Compile and run it in two steps:

```bash
kotlinc Hello.kt -include-runtime -d hello.jar
java -jar hello.jar
# Hello, Kotlin!
```

Or skip the explicit jar step and run the source file directly — convenient
while learning:

```bash
kotlinc -script Hello.kts
# or, for a plain .kt file with a main function:
kotlin Hello.kt
# Hello, Kotlin!
```

## Command-line arguments

`main` can optionally take an `Array<String>` of arguments, matching what was
passed on the command line.

```kotlin
// Greet.kt
fun main(args: Array<String>) {
    if (args.isEmpty()) {
        println("Usage: Greet <name>")
        return
    }
    println("Hello, ${args[0]}!")
}
```

```bash
kotlinc Greet.kt -include-runtime -d greet.jar
java -jar greet.jar World
# Hello, World!
```

## The Kotlin REPL

For quick experiments, the interactive shell evaluates expressions one at a
time without a compile step.

```bash
kotlinc
```

```kotlin
Welcome to Kotlin version 2.0.0
>>> val x = 10
>>> val y = 20
>>> println(x + y)
30
>>> fun square(n: Int) = n * n
>>> square(5)
res0: Int = 25
>>> :quit
```

The REPL is great for testing a one-off standard-library function or
confirming how an expression evaluates before pasting it into a real file.

## `println` vs `print`

```kotlin
fun main() {
    print("No newline here... ")
    print("still the same line.\n")
    println("This adds a newline automatically.")
}
```

```text
No newline here... still the same line.
This adds a newline automatically.
```

## Cheat sheet

| Task | Command |
|------|---------|
| Check JDK version | `java -version` |
| Check Kotlin compiler version | `kotlinc -version` |
| Compile to a runnable jar | `kotlinc File.kt -include-runtime -d app.jar` |
| Run a jar | `java -jar app.jar` |
| Run a `.kt` file directly | `kotlin File.kt` |
| Start the REPL | `kotlinc` |
| Exit the REPL | `:quit` |

## Exercise

Write a program `Info.kt` whose `main` function accepts two command-line
arguments — a name and an age — and prints a sentence like
`Alice is 30 years old.` If fewer than two arguments are given, print a usage
message instead of crashing. Compile it to a jar and run it from the command
line with `java -jar info.jar Alice 30`.
