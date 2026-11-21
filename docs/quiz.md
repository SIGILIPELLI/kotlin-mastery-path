---
title: "Final Quiz"
description: "Test what you have learned across all four levels of Kotlin Mastery Path with a 20-question final quiz."
---

# Final Quiz

20 questions drawn from all four levels of this course. You need **70% (14 out of 20)** to pass —
you can retake it as many times as you like. Your best result is saved in this browser.

<div id="mp-quiz-root"></div>

<script type="application/json" id="mp-quiz-data">
[
  {
    "question": "What does this print?\n```kotlin\nfun main() {\nval a = 7\nvar b = 2.5\nb += a\nprintln(\"$b ${a / 2} ${a / 2.0}\")\n}\n```\n",
    "options": [
      "9.5 3 3.5",
      "Error",
      "9 3 3",
      "9.5 3.5 3.5"
    ],
    "correctIndex": 0,
    "explanation": "val is read-only; integer division truncates. Verified by compiling and running it."
  },
  {
    "question": "What does this print?\n```kotlin\nfun main() {\nval s: String? = null\nprintln(\"${s?.length} ${s ?: \"none\"} ${\"hi\".length}\")\n}\n```\n",
    "options": [
      "null null 2",
      "Error",
      "0 none 2",
      "null none 2"
    ],
    "correctIndex": 3,
    "explanation": "Safe call and Elvis. Verified by compiling and running it."
  },
  {
    "question": "What does this print?\n```kotlin\ndata class User(val name: String, val age: Int)\nfun main() {\nval u = User(\"ada\", 30)\nprintln(\"${u.copy(age = 31)} ${u == User(\"ada\", 30)}\")\n}\n```\n",
    "options": [
      "User@1 false",
      "User(name=ada, age=31) true",
      "Error",
      "User(ada,31) true"
    ],
    "correctIndex": 1,
    "explanation": "Data classes generate equals, toString and copy. Verified by compiling and running it."
  },
  {
    "question": "What does this print?\n```kotlin\nfun main() {\nval n = 3\nprintln(when (n) { 1 -> \"one\"; in 2..4 -> \"few\"; else -> \"many\" })\n}\n```\n",
    "options": [
      "few",
      "many",
      "one",
      "3"
    ],
    "correctIndex": 0,
    "explanation": "when is an expression with ranges. Verified by compiling and running it."
  },
  {
    "question": "What does this print?\n```kotlin\nfun String.cap() = replaceFirstChar { it.uppercase() }\nfun <T> List<T>.second(): T = this[1]\nfun main() {\nprintln(\"kotlin\".cap() + \" \" + listOf(1,2,3).second())\n}\n```\n",
    "options": [
      "Error",
      "Kotlin 2",
      "Kotlin 1",
      "kotlin 2"
    ],
    "correctIndex": 1,
    "explanation": "Extension functions add members without inheritance. Verified by compiling and running it."
  },
  {
    "question": "What does this print?\n```kotlin\nfun main() {\nval xs = listOf(\"apple\", \"bob\", \"cat\", \"avocado\")\nprintln(xs.groupBy { it.first() }.mapValues { it.value.size })\n}\n```\n",
    "options": [
      "{a=2, b=1, c=1}",
      "[a, b, c]",
      "{a=2}",
      "Error"
    ],
    "correctIndex": 0,
    "explanation": "groupBy and mapValues. Verified by compiling and running it."
  },
  {
    "question": "What does this print?\n```kotlin\nfun main() {\nval xs = (1..10).toList()\nprintln(xs.asSequence().filter { it % 2 == 0 }.map { it * it }.first())\n}\n```\n",
    "options": [
      "4",
      "2",
      "16",
      "100"
    ],
    "correctIndex": 0,
    "explanation": "Sequences are lazy and stop at the first match. Verified by compiling and running it."
  },
  {
    "question": "What does this print?\n```kotlin\nsealed interface Shape\nclass Circle(val r: Double) : Shape\nclass Square(val s: Double) : Shape\nfun area(s: Shape): Double = when (s) { is Circle -> 3.0 * s.r * s.r; is Square -> s.s * s.s }\nfun main() {\nval s: Shape = Square(2.0)\nprintln(area(s))\n}\n```\n",
    "options": [
      "2.0",
      "Error",
      "4.0",
      "0.0"
    ],
    "correctIndex": 2,
    "explanation": "Exhaustive when over a sealed type. Verified by compiling and running it."
  },
  {
    "question": "Why are Kotlin classes final by default?",
    "options": [
      "Style",
      "Law",
      "Safer design: inheritance must be opted into with open",
      "Speed"
    ],
    "correctIndex": 2,
    "explanation": "Effective Java advice built in."
  },
  {
    "question": "When does `!!` throw?",
    "options": [
      "When the value is null (NullPointerException); prefer safe calls and Elvis",
      "On empty",
      "Always",
      "Never"
    ],
    "correctIndex": 0,
    "explanation": "Avoid it."
  },
  {
    "question": "What does `suspend` mean?",
    "options": [
      "The function can pause without blocking a thread and may only be called from a coroutine or another suspend function",
      "It sleeps",
      "It's async",
      "It's a thread"
    ],
    "correctIndex": 0,
    "explanation": "Coroutines."
  },
  {
    "question": "A coroutine started with GlobalScope outlives its screen and leaks. What's the fix?",
    "options": [
      "Use a structured scope tied to the lifecycle (viewModelScope/coroutineScope)",
      "Retry",
      "delay",
      "More threads"
    ],
    "correctIndex": 0,
    "explanation": "Structured concurrency."
  },
  {
    "question": "A child coroutine fails. What happens to its parent scope by default?",
    "options": [
      "Ignored",
      "Nothing",
      "Retry",
      "The parent and siblings are cancelled (use supervisorScope to isolate)"
    ],
    "correctIndex": 3,
    "explanation": "Fail-fast propagation."
  },
  {
    "question": "Which dispatcher should blocking file/DB calls use?",
    "options": [
      "Dispatchers.IO",
      "Default",
      "Main",
      "Unconfined"
    ],
    "correctIndex": 0,
    "explanation": "Keep the event loop free."
  },
  {
    "question": "A Flow collector is slower than the producer. Which operator helps?",
    "options": [
      "delay",
      "distinct",
      "sort",
      "buffer or conflate (or collectLatest)"
    ],
    "correctIndex": 3,
    "explanation": "Backpressure."
  },
  {
    "question": "A StateFlow vs SharedFlow for UI state?",
    "options": [
      "Channel",
      "Flow",
      "SharedFlow with no replay",
      "StateFlow: it always holds the latest value"
    ],
    "correctIndex": 3,
    "explanation": "State vs events."
  },
  {
    "question": "A Ktor endpoint throws and clients see a stack trace. What do you install?",
    "options": [
      "StatusPages to map exceptions to safe, consistent responses",
      "Logging",
      "Compression",
      "CORS"
    ],
    "correctIndex": 0,
    "explanation": "Error handling."
  },
  {
    "question": "A SQL query is built with string templates from user input. What's wrong?",
    "options": [
      "Typing",
      "SQL injection; use parameterized queries/the Exposed DSL",
      "Nothing",
      "Slow"
    ],
    "correctIndex": 1,
    "explanation": "Never concatenate."
  },
  {
    "question": "Kotlin code called from Java needs default parameters as overloads. Which annotation?",
    "options": [
      "@JvmStatic",
      "@JvmField",
      "@Throws",
      "@JvmOverloads"
    ],
    "correctIndex": 3,
    "explanation": "Interop."
  },
  {
    "question": "What do Detekt and ktlint do in CI?",
    "options": [
      "Static analysis and style enforcement that fail the build on violations",
      "Compile",
      "Run tests",
      "Deploy"
    ],
    "correctIndex": 0,
    "explanation": "Quality gates."
  }
]
</script>

## Coding challenges

Try these on your own computer, then compare your output. Nothing you write is sent anywhere.

### A todo repository with a sealed Result

Implement TaskRepo.add (reject blank titles, trim, assign ids), complete (copy the task with done = true, or an error for unknown ids) and summary using partition. Results are a sealed Result type handled with an exhaustive when.

**Starter code**

```kotlin
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

**Expected output**

```text
ok: Task(id=1, title=write quiz, done=false)
error: title must not be blank
ok: Task(id=2, title=publish, done=false)
ok: Task(id=1, title=write quiz, done=true)
error: no task with id 9
open=[publish] done=[write quiz]
```

<details><summary>Show solution and explanation</summary>

<pre><code>sealed interface Result&lt;out T&gt; {
    data class Ok&lt;T&gt;(val value: T) : Result&lt;T&gt;
    data class Err(val message: String) : Result&lt;Nothing&gt;
}

data class Task(val id: Int, val title: String, val done: Boolean = false)

class TaskRepo {
    private val tasks = mutableListOf&lt;Task&gt;()
    private var nextId = 1

    fun add(title: String): Result&lt;Task&gt; {
        if (title.isBlank()) return Result.Err(&quot;title must not be blank&quot;)
        val t = Task(nextId++, title.trim())
        tasks += t
        return Result.Ok(t)
    }

    fun complete(id: Int): Result&lt;Task&gt; {
        val i = tasks.indexOfFirst { it.id == id }
        if (i &lt; 0) return Result.Err(&quot;no task with id $id&quot;)
        tasks[i] = tasks[i].copy(done = true)
        return Result.Ok(tasks[i])
    }

    fun summary(): String {
        val (done, open) = tasks.partition { it.done }
        return &quot;open=${open.map { it.title }} done=${done.map { it.title }}&quot;
    }
}

fun describe(r: Result&lt;Task&gt;): String = when (r) {
    is Result.Ok -&gt; &quot;ok: ${r.value}&quot;
    is Result.Err -&gt; &quot;error: ${r.message}&quot;
}

fun main() {
    val repo = TaskRepo()
    println(describe(repo.add(&quot;write quiz&quot;)))
    println(describe(repo.add(&quot;   &quot;)))
    println(describe(repo.add(&quot;publish&quot;)))
    println(describe(repo.complete(1)))
    println(describe(repo.complete(9)))
    println(repo.summary())
}
</code></pre>

<p>A sealed Result makes failure explicit and forces callers to handle every case, data classes give copy for immutable updates, and partition splits done/open in one pass. Verified by compiling and running the solution with kotlinc.</p>
</details>

### Extensions, grouping and pagination

Write the extension pageOf(page, size) with argument checks, and topSpenders(orders, n) that totals each customer's spend, sorts highest first (ties by name) and formats name=12.34.

**Starter code**

```kotlin
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

**Expected output**

```text
[ada=16.50, bo=15.00]
[5, 6, 7, 8]
[9, 10]
[]
error: page and size must be >= 1
9.3
```

<details><summary>Show solution and explanation</summary>

<pre><code>fun &lt;T&gt; List&lt;T&gt;.pageOf(page: Int, size: Int): List&lt;T&gt; {
    require(page &gt;= 1 &amp;&amp; size &gt;= 1) { &quot;page and size must be &gt;= 1&quot; }
    return drop((page - 1) * size).take(size)
}

data class Order(val customer: String, val item: String, val qty: Int, val price: Double)

fun Order.total() = qty * price

fun topSpenders(orders: List&lt;Order&gt;, n: Int): List&lt;String&gt; =
    orders.groupBy { it.customer }
        .mapValues { (_, os) -&gt; os.sumOf { it.total() } }
        .entries.sortedWith(compareByDescending&lt;Map.Entry&lt;String, Double&gt;&gt; { it.value }.thenBy { it.key })
        .take(n)
        .map { &quot;%s=%.2f&quot;.format(it.key, it.value) }

fun main() {
    val orders = listOf(
        Order(&quot;ada&quot;, &quot;pen&quot;, 3, 1.5), Order(&quot;bo&quot;, &quot;book&quot;, 1, 12.0), Order(&quot;ada&quot;, &quot;book&quot;, 1, 12.0),
        Order(&quot;cy&quot;, &quot;pen&quot;, 10, 1.5), Order(&quot;bo&quot;, &quot;pen&quot;, 2, 1.5),
    )
    println(topSpenders(orders, 2))
    println((1..10).toList().pageOf(2, 4))
    println((1..10).toList().pageOf(3, 4))
    println((1..10).toList().pageOf(4, 4))
    try { listOf(1).pageOf(0, 5) } catch (e: IllegalArgumentException) { println(&quot;error: ${e.message}&quot;) }
    println(orders.map { it.total() }.average())
}
</code></pre>

<p>Extension functions keep helpers close to the types they extend, require() validates preconditions, and groupBy/mapValues/sortedWith express the report declaratively. Verified by compiling and running the solution with kotlinc.</p>
</details>
