---
description: "Performance & Profiling — Kotlin's conveniences — collection operator chains, lambdas, boxed generics — mostly compile down to the same bytecode a…"
---

# 09 · Performance & Profiling

Kotlin's conveniences — collection operator chains, lambdas, boxed
generics — mostly compile down to the same bytecode a hand-written Java
loop would produce, but not always. This module measures a few of the
places where idiomatic-looking Kotlin has a real performance cost, using
`kotlin.system.measureTimeMillis`/`measureNanoTime` for quick, honest
before/after numbers (a real profiler like async-profiler or the JVM
Flight Recorder is the right tool for production work — these
micro-benchmarks are for building intuition, not for citing exact
percentages).

## `Sequence` vs. eager collections for long chains

A chain of `List` operators (`.map`, `.filter`) allocates a full
intermediate list at *every* step. `Sequence` operators are lazy — they
process one element through the whole chain before moving to the next,
allocating far less.

```kotlin
import kotlin.system.measureTimeMillis
import kotlin.system.measureNanoTime

fun sumWithList(n: Int): Long {
    val list = (1..n).map { it.toLong() }
    return list.sum()
}

fun sumWithSequence(n: Int): Long {
    return (1..n).asSequence().map { it.toLong() }.sum()
}

fun main() {
    val n = 2_000_000

    val listTime = measureTimeMillis { sumWithList(n) }
    val seqTime = measureTimeMillis { sumWithSequence(n) }
    println("List-based sum: ${listTime}ms")
    println("Sequence-based sum: ${seqTime}ms")
}
```

```text
List-based sum: 103ms
Sequence-based sum: 23ms
```

This particular example only has one `.map` step, so the win comes mostly
from `Sequence` avoiding the intermediate `List<Long>` allocation
entirely; the gap widens further as more operators are chained, since each
extra `List` operator is another full-size intermediate collection.
`Sequence` is not automatically faster for *short* collections or single
operations — the laziness machinery itself has overhead, so measure before
switching hot, small-collection code to sequences.

## Boxing: primitive arrays vs. boxed generic collections

Kotlin's `List<Long>`/`ArrayList<Long>` box every element as a full
`java.lang.Long` object on the JVM (generics can't hold primitives).
`LongArray` (and `IntArray`, `DoubleArray`, etc.) store real unboxed
primitives contiguously in memory.

```kotlin
val boxedTime = measureNanoTime {
    val boxed = ArrayList<Long>(100_000)
    for (i in 0 until 100_000) boxed.add(i.toLong())
    var s = 0L
    for (x in boxed) s += x
}
val primitiveTime = measureNanoTime {
    val arr = LongArray(100_000) { it.toLong() }
    var s = 0L
    for (x in arr) s += x
}
println("Boxed ArrayList<Long>: ${boxedTime / 1_000_000.0}ms")
println("Primitive LongArray: ${primitiveTime / 1_000_000.0}ms")
```

```text
Boxed ArrayList<Long>: 5.494458ms
Primitive LongArray: 0.482958ms
```

Over an order of magnitude difference here, from allocation pressure and
cache locality alone. This is exactly why `IntArray`/`LongArray`/etc. exist
as distinct types instead of Kotlin just always using `Array<Int>` — for
large numeric workloads (image processing, numeric simulation, hot inner
loops), the primitive array types are a meaningful, measurable win.

## `inline` functions: eliminating lambda allocation

A non-`inline` higher-order function allocates a real object implementing
`Function1`/`FunctionN` for every lambda passed to it (unless the JIT
manages to prove it can skip this, which it often can't across generic
boundaries). `inline` tells the compiler to paste the function's body —
and the lambda's body — directly at the call site, at compile time,
eliminating both the function call and the lambda object.

```kotlin
inline fun inlineRepeat(times: Int, action: (Int) -> Unit) {
    for (i in 0 until times) action(i)
}

fun noinlineRepeat(times: Int, action: (Int) -> Unit) {
    for (i in 0 until times) action(i)
}

fun main() {
    val n = 5_000_000
    // Warm up the JIT first so both measurements reflect steady-state
    // performance, not interpreter/compilation overhead.
    repeat(3) {
        var w = 0L
        inlineRepeat(n) { w += it }
        noinlineRepeat(n) { w += it }
    }

    var sum1 = 0L
    val inlineTime = measureNanoTime { inlineRepeat(n) { sum1 += it } }
    var sum2 = 0L
    val noinlineTime = measureNanoTime { noinlineRepeat(n) { sum2 += it } }

    println("inline version: ${inlineTime / 1_000_000.0}ms, sum=$sum1")
    println("non-inline version: ${noinlineTime / 1_000_000.0}ms, sum=$sum2")
}
```

```text
inline version: 2.78925ms, sum=12499997500000
non-inline version: 18.171125ms, sum=12499997500000
```

Nearly 7x here after proper JIT warmup. This is exactly why `map`,
`filter`, `forEach`, and almost every standard-library higher-order
function on `Iterable` are declared `inline` — without it, a `.filter {
}.map { }` chain over a hot loop would allocate two lambda objects per
call.

## Kotlin-specific traps

- **Benchmarking without a warmup loop measures the JIT compiler, not
  your code.** The first few thousand calls to any function run
  interpreted or through a cheap tier-1 JIT compile; skip a warmup and
  your numbers mostly reflect compilation overhead, not steady-state
  performance (notice the warmup loop above before the actual
  measurement).
- **`inline` on a function with a large body bloats bytecode** at every
  call site — it's a real tradeoff, not a free win. Reserve `inline`
  for small, hot, or genuinely higher-order (taking function parameters)
  functions, which is exactly the standard library's own policy.
- **A non-local `return` inside a lambda only works because the
  enclosing function is `inline`.** `list.forEach { if (it == target)
  return }` compiles because `forEach` is inline; the identical code with
  a non-inline higher-order function is a compile error ("'return' is not
  allowed here").
- **`data class` `equals`/`hashCode`/`toString` are generated but not
  free** — for large object graphs or hot-path comparisons, a
  hand-written `equals` that short-circuits early (e.g. checking a
  cheap ID field first) can matter; profile before hand-rolling, though.
- **Sequences don't parallelize by themselves.** `Sequence` is about
  laziness and single-pass evaluation, not concurrency — for real
  parallel processing over large collections, look at
  `Collection.parallelStream()` (Java interop) or splitting work across
  coroutines explicitly.

## How It Actually Works

The `LongArray` vs `ArrayList<Long>` gap traces directly to a JVM memory
layout difference, not anything Kotlin-specific: `LongArray` compiles to a
genuine `long[]` — a single contiguous block of memory holding raw 8-byte
values with no per-element object headers — while `ArrayList<Long>` (backed
by `Object[]`) stores **references** to separately heap-allocated `Long`
objects, each with its own object header (typically 16 bytes of overhead on
a 64-bit JVM before the actual 8-byte value). Iterating the boxed version
means chasing a pointer to a different heap location for every element,
which is cache-unfriendly (each `Long` object can land anywhere the
allocator put it) versus the primitive array's sequential memory scan,
which the CPU's prefetcher handles efficiently — that's most of the
order-of-magnitude gap measured above, on top of the allocation cost of
100,000 separate `Long` objects versus one array allocation.

`Sequence`'s advantage over chained `List` operators is really about
avoiding **allocation churn** the JIT can't always optimize away — each
`.map`/`.filter` on a `List` genuinely calls `new ArrayList<>()` and fills
it, and while the JVM's generational garbage collector is very fast at
reclaiming short-lived objects (this is precisely the case young-generation
GC is tuned for), it's still work the CPU has to do that a single-pass
`Sequence` skips by fusing every operator into one loop over the source,
per element, with no intermediate container ever materializing.

`inline` functions eliminate a different cost entirely: every non-inline
lambda passed as an argument is a `Function1`/`Function2`-style object that
usually has to be allocated (unless it's a captureless singleton, as
discussed in the lambdas module) and then invoked through an interface
call (`invokeinterface`), which the JIT can sometimes — but not always —
inline back out via speculative devirtualization. Marking the higher-order
function `inline` removes the uncertainty: the compiler pastes the lambda's
actual bytecode directly into the call site during compilation, so there is
provably no allocation and no indirect call at all, which is why the
standard library's hottest, most-called higher-order functions (`map`,
`let`, `filter`, `apply`) are all declared `inline`.

## Cheat sheet

| Concern | Fix | When it matters |
|---|---|---|
| Long operator chains | `.asSequence()` before the chain | 3+ chained operations, large collections |
| Numeric collections | `IntArray`/`LongArray`/`DoubleArray` | Hot loops, large numeric datasets |
| Hot higher-order functions | `inline fun` | Called extremely frequently, small body |
| Non-local return needed | Only works in `inline` functions | Early-exit from inside a lambda |
| "Is this actually slow?" | `measureTimeMillis`/`measureNanoTime` + warmup, or a real profiler | Before optimizing anything |

## Exercise

Write a function that computes the sum of squares of even numbers from 1
to 5,000,000 three ways: (1) an eager `List` operator chain
(`.filter.map.sum()`), (2) the same chain with `.asSequence()`, and (3) a
plain `for` loop with an `if`. Measure all three with `measureTimeMillis`
(including a warmup pass) and print the results ordered fastest to
slowest. Which one wins, and is the difference big enough to justify
choosing it over the most readable option in ordinary application code?
