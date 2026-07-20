# 10 · Project — CLI To-Do App

A small end-to-end project combining everything from Level 1: data classes,
collections, null safety, extension functions, and functions with default
arguments. You'll build a command-line to-do list that persists tasks to a
plain-text file between runs.

## What you'll build

A single-file Kotlin CLI to-do app that:

- Adds tasks with a description
- Lists all tasks, showing done/pending status
- Marks a task done by its number
- Removes a task by its number
- Persists everything to a plain-text file between runs

## Project layout

```text
todoapp/
    Todo.kt
```

Everything lives in one file for this project — real multi-file Kotlin
projects normally use Gradle (introduced in
[Level 2](../level-2/08-gradle-basics.md)), but a single `.kt` file compiled
directly with `kotlinc` is simplest while you're still in Level 1.

## `Todo.kt` — the data class

```kotlin
// Todo.kt

data class Task(val description: String, val isDone: Boolean = false) {

    // Serialize to a single line for file storage: "done|description" or "pending|description"
    fun toStorageLine(): String {
        val status = if (isDone) "done" else "pending"
        return "$status|$description"
    }

    companion object {
        // Parse a stored line back into a Task; returns null if the line is malformed
        fun fromStorageLine(line: String): Task? {
            val parts = line.split("|", limit = 2)
            if (parts.size != 2) return null
            val isDone = parts[0] == "done"
            return Task(parts[1], isDone)
        }
    }
}
```

## Extension functions for display formatting

```kotlin
fun Task.displayLine(index: Int): String {
    val marker = if (isDone) "[x]" else "[ ]"
    return "$index. $marker $description"
}

fun List<Task>.countDone(): Int = this.count { it.isDone }
```

## Storage — loading and saving

```kotlin
import java.io.File

class TaskStorage(private val fileName: String) {

    fun load(): MutableList<Task> {
        val file = File(fileName)
        if (!file.exists()) {
            return mutableListOf()
        }
        return file.readLines()
            .filter { it.isNotBlank() }
            .mapNotNull { Task.fromStorageLine(it) }
            .toMutableList()
    }

    fun save(tasks: List<Task>) {
        val file = File(fileName)
        file.writeText(tasks.joinToString("\n") { it.toStorageLine() })
    }
}
```

`mapNotNull` transforms each line and drops any that came back `null` (i.e.
malformed lines) in one step — a small preview of the collection operations
covered fully in [Level 2](../level-2/05-collections-deep-dive.md).

## The CLI logic

```kotlin
fun printUsage() {
    println("Usage: todo [add <description> | list | done <number> | remove <number>]")
}

fun addTask(tasks: MutableList<Task>, storage: TaskStorage, description: String?) {
    val desc = description ?: run {
        println("Error: add requires a description")
        return
    }
    tasks.add(Task(desc))
    storage.save(tasks)
    println("Added: $desc")
}

fun listTasks(tasks: List<Task>) {
    if (tasks.isEmpty()) {
        println("No tasks yet.")
        return
    }
    tasks.forEachIndexed { index, task ->
        println(task.displayLine(index + 1))
    }
    println("${tasks.countDone()}/${tasks.size} done")
}

fun markDone(tasks: MutableList<Task>, storage: TaskStorage, numberArg: String?) {
    val number = numberArg?.toIntOrNull()
    if (number == null || number < 1 || number > tasks.size) {
        println("Error: done requires a valid task number")
        return
    }
    val index = number - 1
    tasks[index] = tasks[index].copy(isDone = true)
    storage.save(tasks)
    println("Marked done: ${tasks[index].description}")
}

fun removeTask(tasks: MutableList<Task>, storage: TaskStorage, numberArg: String?) {
    val number = numberArg?.toIntOrNull()
    if (number == null || number < 1 || number > tasks.size) {
        println("Error: remove requires a valid task number")
        return
    }
    val removed = tasks.removeAt(number - 1)
    storage.save(tasks)
    println("Removed: ${removed.description}")
}

fun main(args: Array<String>) {
    val storage = TaskStorage("tasks.txt")
    val tasks = storage.load()

    if (args.isEmpty()) {
        printUsage()
        return
    }

    when (args[0]) {
        "add" -> addTask(tasks, storage, args.drop(1).joinToString(" ").ifBlank { null })
        "list" -> listTasks(tasks)
        "done" -> markDone(tasks, storage, args.getOrNull(1))
        "remove" -> removeTask(tasks, storage, args.getOrNull(1))
        else -> printUsage()
    }
}
```

Notice how much of Level 1 shows up here: `when` for command dispatch, the
Elvis operator and `?:`/`run {}` for validating optional input, `toIntOrNull()`
for safely parsing user input without a `try`/`catch`, `copy()` to update an
immutable `Task` in place in the list, and an extension function
(`displayLine`) to keep formatting logic out of the data class itself.

## Running it

```bash
kotlinc Todo.kt -include-runtime -d todo.jar

java -jar todo.jar add "Buy groceries"
# Added: Buy groceries

java -jar todo.jar add "Write Kotlin lesson"
java -jar todo.jar list
# 1. [ ] Buy groceries
# 2. [ ] Write Kotlin lesson
# 0/2 done

java -jar todo.jar done 1
# Marked done: Buy groceries

java -jar todo.jar list
# 1. [x] Buy groceries
# 2. [ ] Write Kotlin lesson
# 1/2 done

java -jar todo.jar remove 2
# Removed: Write Kotlin lesson

java -jar todo.jar list
# 1. [x] Buy groceries
# 1/1 done
```

Each run reloads `tasks.txt` from disk, so tasks persist across separate
invocations of the program.

## Stretch goals

- Add a `clear-done` command that removes every task where `isDone` is true.
- Add priority levels (`Task(description: String, isDone: Boolean = false,
  priority: Int = 0)`) and sort `list` output by priority.
- Replace the plain-text storage format with JSON (you'll use a real JSON
  library for this in [Level 2, Module 7](../level-2/07-working-with-json.md)).
- Add a `search <keyword>` command using `filter` on the task list.

Completing this project means you're ready for **Level 2 · Intermediate**.
