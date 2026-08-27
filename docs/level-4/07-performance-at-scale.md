# 07 · Performance at Scale

Kotlin compiles to JVM bytecode, so most performance work is really JVM
performance work — but Kotlin's own abstractions (boxed collections,
`Sequence` vs. eager collection operators, data class allocations) add a
layer of decisions that matter once code runs millions of times instead
of a few. This module measures the difference with real benchmarks
rather than folklore.

## Primitive loops vs. boxed collections

```kotlin
fun sumLoop(n: Int): Long {
    var total = 0L
    for (i in 0 until n) total += i.toLong()
    return total
}

fun sumBoxed(n: Int): Long {
    val list = ArrayList<Long>()
    for (i in 0 until n) list.add(i.toLong())
    var total = 0L
    for (x in list) total += x
    return total
}
```

Timed with a warmup (to let the JIT compile hot paths before measuring —
the first few calls to any JVM method run interpreted and are much
slower):

```console
$ kotlinc Bench.kt -include-runtime -d bench.jar
$ java -jar bench.jar
primitive    result=12499997500000 time=1.78ms
boxed-list   result=12499997500000 time=132.38ms
```

Same result, 74x slower for the boxed version on this run. The
`ArrayList<Long>` path pays twice: once to box every `Long` into a heap
object (Kotlin's `Long` maps to Java's primitive `long` only when it can
prove one is never nullable and never stored in a generic container —
`ArrayList<Long>` forces boxing), and again for the list's own array
growth and indirection. The primitive loop stays entirely on the stack
as raw `long` values the whole time.

## Sequence vs. eager collection chains

```kotlin
val list = (1..1_000_000).toList()

var listOps = 0
val listResult = list
    .map { listOps++; it * 2 }
    .filter { listOps++; it % 3 == 0 }
    .first()

var seqOps = 0
val seqResult = list.asSequence()
    .map { seqOps++; it * 2 }
    .filter { seqOps++; it % 3 == 0 }
    .first()
```

```console
$ kotlinc Seq.kt -include-runtime -d seq.jar && java -jar seq.jar
list: result=6 ops=2000000
sequence: result=6 ops=6
```

`.map().filter().first()` on a `List` runs `map` over *all* 1,000,000
elements, builds a full intermediate list, then runs `filter` over all
1,000,000 of those, then takes the first match — 2,000,000 total
operations for a chain that only needed to look at one element.
`asSequence()` makes every step lazy and per-element: `first()` pulls one
element through `map` then `filter`, finds it matches, and stops — 6
total operations (3 elements × 2 steps) to find the same answer. The
crossover point matters: for small collections or when every element is
consumed anyway (no early exit like `first`/`take`), `Sequence`'s
per-element overhead (an object wrapping each step) can make it *slower*
than a plain `List` chain. It's an optimization for large collections
with early termination, not a blanket replacement for `.map()`.

## Data classes and allocation pressure

Every `data class` instance is a heap allocation. Code creating millions
of short-lived data class instances per second — say, wrapping every row
of a large result set in a `data class Row(...)` just to pass it one
function call deeper — puts real pressure on the garbage collector. Two
mitigations that matter in practice:

- **`value class` (formerly `inline class`)** wraps a single value with
  zero runtime allocation in most cases — the wrapper is erased at
  compile time and the underlying primitive/reference is used directly,
  as long as it isn't boxed for a generic type or nullable use.
  `value class UserId(val raw: Long)` gives type safety (a `UserId`
  can't be passed where a raw `Long` for, say, an order ID is expected)
  without the allocation cost of a full class.
- **Reusing objects in a hot loop** instead of allocating fresh ones —
  common in interpreters, parsers, and game loops — trades code clarity
  for fewer GC pauses; only worth it once profiling shows allocation is
  the actual bottleneck.

## Profiling before optimizing

Guessing which function is slow and rewriting it is a common way to make
code uglier without making it faster. `async-profiler` (flame graphs) or
even the simpler `measureNanoTime`/`measureTimeMillis` from
`kotlin.system` around suspected hot paths turn "I think this is slow"
into "this function is 80% of wall-clock time" before any code changes.
JMH (Java Microbenchmark Harness) is the standard tool for
production-grade microbenchmarks — it handles JIT warmup, dead-code
elimination by the optimizer, and statistical noise automatically, which
hand-rolled `measureNanoTime` loops like this module's do not.

## Traps

- **Micro-benchmarking without warmup measures the JIT compiler, not
  your code.** The first calls to any function run through the
  interpreter or C1 (client) compiler before HotSpot's C2 promotes hot
  methods to fully optimized machine code — a benchmark that times a
  single cold call can be 10-100x pessimistic compared to steady-state
  performance.
- **`it % 3 == 0` style filters can hide autoboxing** when the receiver
  type is `Int?`/generic rather than primitive `Int` — nullable numeric
  types are always boxed, so a `List<Int?>` pays the boxing cost of
  `ArrayList<Long>` above even though it "looks like" plain `Int`s.
- **String concatenation in a loop (`str += x`) allocates a new `String`
  every iteration** because `String` is immutable — `StringBuilder`
  (which Kotlin's compiler uses automatically for simple `"$a$b"`
  template concatenation, but *not* for `+=` inside a loop) avoids the
  quadratic-time trap of repeated full-string copies.
- **`Sequence` with a `Comparator`-heavy `sortedBy` still has to
  materialize the whole sequence** to sort it — laziness only helps
  operations that can proceed element-by-element (`map`, `filter`,
  `take`, `first`); `sortedBy`, `groupBy`, and `toList` all force full
  evaluation regardless of whether the source is a `List` or `Sequence`.

## Cheat sheet

| Concern | Approach |
|---|---|
| Avoid boxing in hot numeric loops | Primitive arrays/loops, not `ArrayList<Long>` |
| Large collection + early exit | `asSequence()` before `.first()`/`.take()` |
| Small collection, consumes everything | Plain `List` chain — `Sequence` overhead not worth it |
| Type-safe wrapper, zero allocation | `value class` |
| Measure before optimizing | `measureNanoTime`, or JMH for rigor |
| Repeated string building in a loop | `StringBuilder`, not `+=` |

## Exercise

Extend `Bench.kt` with a third variant, `sumPrimitiveArray`, that fills
an `IntArray` (not `ArrayList<Long>`) and sums it. Time all three with
the same warmup pattern and confirm the ranking is
`primitive < primitiveArray < boxed-list`; then explain in a comment why
`IntArray` avoids boxing that `ArrayList<Int>` cannot.
