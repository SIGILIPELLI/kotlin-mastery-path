# 07 · DSLs in Kotlin

Gradle's `build.gradle.kts`, Ktor's `routing { get("/x") { } }` from
[Module 2](02-building-apis-ktor.md), and Exposed's `transaction { }`
from [Module 3](03-databases-exposed-sqlite.md) all *look* like
special syntax, but they're plain Kotlin functions taking lambdas. This
module covers the language features that make that possible: lambdas with
receivers, infix functions, and `@DslMarker`.

## Lambda with receiver: the core trick

A regular lambda parameter `(Tag) -> Unit` requires calling methods on an
explicit parameter (`{ tag -> tag.text = "hi" }`). A lambda *with
receiver*, `Tag.() -> Unit`, runs the lambda body **as if it were a member
of `Tag`** — so `this` (or an implicit receiver) refers to the `Tag`
directly, letting the block read like nested markup.

```kotlin
class Tag(val name: String) {
    val children = mutableListOf<Tag>()
    val attributes = mutableMapOf<String, String>()
    var text: String = ""

    fun render(indent: String = ""): String {
        val attrs = if (attributes.isEmpty()) "" else " " + attributes.entries.joinToString(" ") { "${it.key}=\"${it.value}\"" }
        return if (children.isEmpty()) {
            "$indent<$name$attrs>$text</$name>"
        } else {
            val inner = children.joinToString("\n") { it.render("$indent  ") }
            "$indent<$name$attrs>\n$inner\n$indent</$name>"
        }
    }
}

fun Tag.tag(name: String, block: Tag.() -> Unit): Tag {
    val child = Tag(name)
    child.block()
    children.add(child)
    return child
}

fun html(block: Tag.() -> Unit): Tag {
    val root = Tag("html")
    root.block()
    return root
}

fun main() {
    val page = html {
        tag("head") {
            tag("title") { text = "My Page" }
        }
        tag("body") {
            attributes["class"] = "main"
            tag("h1") { text = "Welcome" }
            tag("p") { text = "Hello, DSL!" }
        }
    }
    println(page.render())
}
```

```text
<html>
  <head>
    <title>My Page</title>
  </head>
  <body class="main">
    <h1>Welcome</h1>
    <p>Hello, DSL!</p>
  </body>
</html>
```

Inside `tag("body") { attributes["class"] = "main"; tag("h1") { ... } }`,
`attributes` and the nested `tag(...)` call both resolve against the
implicit `Tag` receiver — no `it.` or explicit parameter name needed. This
pattern (a function taking a `Receiver.() -> Unit` lambda, called "type-safe
builder") is exactly what `routing { }`, `transaction { }`, and Gradle's
`dependencies { }` are built from.

## Infix functions and operator overloading

`infix` lets a single-parameter function be called without the dot or
parentheses, which reads naturally for small assertion- or config-style
APIs.

```kotlin
class Requirement(val name: String) {
    val rules = mutableListOf<String>()
    infix fun mustBe(value: String) { rules.add("$name must be $value") }
}

fun main() {
    val r = Requirement("status")
    r mustBe "active"     // instead of r.mustBe("active")
    println(r.rules)
}
```

```text
[status must be active]
```

Kotlin's own `to` (building a `Pair`), `and`/`or` (on `Boolean`), and
`..`/`rangeTo` are all infix or operator functions defined in the standard
library the same way — there's no special compiler magic beyond what's
available to any library author.

## `@DslMarker`: preventing scope confusion

Nested builder lambdas each bring their own implicit receiver into scope.
Without help, an inner block could accidentally call a method meant for an
*outer* receiver, producing a DSL call in the wrong place with no compile
error. `@DslMarker` on an annotation, applied to each builder class, tells
the compiler to only expose the *nearest* enclosing receiver implicitly.

```kotlin
@DslMarker
annotation class SpecDsl

@SpecDsl
class Requirement(val name: String) {
    val rules = mutableListOf<String>()
    infix fun mustBe(value: String) { rules.add("$name must be $value") }
}

@SpecDsl
class Spec {
    val requirements = mutableListOf<Requirement>()
    fun requirement(name: String, block: Requirement.() -> Unit) {
        val r = Requirement(name)
        r.block()
        requirements.add(r)
    }
}

fun spec(block: Spec.() -> Unit): Spec {
    val s = Spec()
    s.block()
    return s
}

fun main() {
    val s = spec {
        requirement("status") {
            this mustBe "active"
            // requirement("nested") { }  -- would NOT compile here
        }
    }
    println(s.requirements.size)
}
```

```text
1
```

Uncommenting `requirement("nested") { }` inside the `Requirement` block
produces:

```text
error: 'fun requirement(name: String, block: Requirement.() -> Unit): Unit'
cannot be called in this context with an implicit receiver. Use an
explicit receiver if necessary.
```

Without `@SpecDsl` on both classes, that call would silently compile —
`Spec.requirement` is technically still reachable through the outer
receiver — and you'd get a `Requirement` nested inside another
`Requirement`'s block, which makes no structural sense for this DSL.
`@DslMarker` turns that mistake into a compile error instead of a
confusing runtime structure.

## Kotlin-specific traps

- **A receiver lambda's `this` shadows outer receivers one level at a
  time**, not all at once — without `@DslMarker`, *every* enclosing
  receiver's members stay implicitly callable inside a deeply nested
  block, which is the exact bug `@DslMarker` prevents.
- **`infix` functions must have exactly one parameter**, no default value,
  and can't be vararg — trying to add a second required parameter is a
  compile error, forcing infix APIs to stay genuinely binary-operator-like.
- **Type-safe builders return `Unit`-typed lambdas by convention**, but
  the outer function (`html { }`, `spec { }`) usually returns the built
  object — mixing these up (returning the lambda's result instead of the
  accumulated builder) is a common bug when first writing a DSL.
- **Operator overloading can make code *less* readable if overused.**
  Kotlin lets you overload `+`, `*`, `..`, `[]`, and more on your own
  types, but an overload whose behavior doesn't match the operator's
  common meaning (e.g. `+` that doesn't commute) actively misleads
  readers — this is a taste/API-design trap, not a compiler-enforced one.

## Cheat sheet

| Concept | Syntax | Purpose |
|---|---|---|
| Lambda with receiver | `block: Receiver.() -> Unit` | `this`/implicit receiver inside the lambda |
| Type-safe builder | `fun receiver(block: T.() -> Unit): T` | The core DSL-building pattern |
| Infix function | `infix fun T.name(x: X)` | Operator-like call syntax: `a name b` |
| Operator overload | `operator fun T.plus(x: X)` | Reuse `+`, `-`, `[]`, etc. on custom types |
| `@DslMarker` | Annotation on an annotation, applied to builder classes | Restricts implicit receivers to the nearest scope |

## Exercise

Extend the `html` DSL with a `ul`/`li` pair: `tag("ul") { tag("li") {
text = "One" }; tag("li") { text = "Two" } }` should already work with the
existing `tag()` function — verify it renders correctly. Then add a
dedicated `Tag.li(text: String)` extension function so callers can write
`li("One")` instead of `tag("li") { text = "One" }`, and re-render the page
using it.
