# 09 · Code Quality (detekt/ktlint)

A compiler enforces correctness; it says nothing about naming
consistency, formatting, or the dozens of "technically works but
shouldn't be written this way" patterns that make a codebase hard to
read six months later. `ktlint` (formatting/style) and `detekt` (static
analysis for bugs/complexity) are the two tools that fill that gap in
Kotlin, both runnable locally and in CI to fail a build before a
reviewer even has to comment on style.

## A deliberately messy file

```kotlin
class userAccount(val Name:String,var balance:Double) {
    fun Deposit(amt:Double){
        if(amt>0){
        balance=balance+amt
        }else{
            println("bad")
        }
    }
    fun withdraw(amt: Double): Boolean {
        return if (amt <= balance) { balance -= amt; true } else false
    }
}
```

Running `ktlint` against it directly, no configuration needed:

```console
$ ktlint Messy.kt
Messy.kt:1:1: File 'Messy.kt' contains a single class... file should be named after the class, 'userAccount.kt' (standard:filename)
Messy.kt:1:7: Class or object name should start with an uppercase letter and use camel case (standard:class-naming)
Messy.kt:1:19: Newline expected after opening parenthesis (standard:class-signature)
Messy.kt:1:27: Whitespace after ':' is missing (standard:parameter-list-spacing)
Messy.kt:2:9: Function name should start with a lowercase letter (except factory methods) and use camel case (standard:function-naming)
Messy.kt:3:11: Missing spacing after "if" (standard:keyword-spacing)
Messy.kt:4:1: Unexpected indentation (8) (should be 12) (standard:indent)
Messy.kt:9:40: Function body should be replaced with body expression (standard:function-expression-body)
```

57 lines of violations total from a 13-line file — `userAccount` (should
be `UserAccount`), `Deposit`/`Name` (should be lowercase-first), and
inconsistent spacing/indentation everywhere. Each finding cites the
specific rule (`standard:class-naming`, `standard:indent`, ...), which is
what a CI failure message points a contributor at.

## Autocorrecting what can be fixed automatically

```console
$ ktlint -F MessyFixable.kt
MessyFixable.kt:1:1: File 'MessyFixable.kt' contains a single class... (cannot be auto-corrected) (standard:filename)
MessyFixable.kt:1:7: Class or object name should start with an uppercase letter (cannot be auto-corrected) (standard:class-naming)
MessyFixable.kt:2:12: Function name should start with a lowercase letter (cannot be auto-corrected) (standard:function-naming)
```

`-F` rewrote the file in place — spacing, indentation, and the
`if`/`else` block were converted to an expression body automatically —
and the output afterward shows only what's *left*: renaming decisions
(`userAccount` → `UserAccount`, `Deposit` → `deposit`) that ktlint
deliberately refuses to make for you, because renaming a symbol can
break other code that references it and isn't a safe mechanical rewrite
the way "add a space after a colon" is. The reformatted file:

```kotlin
class userAccount(
    val Name: String,
    var balance: Double,
) {
    fun Deposit(amt: Double) {
        if (amt > 0) {
            balance = balance + amt
        } else {
            println("bad")
        }
    }

    fun withdraw(amt: Double): Boolean =
        if (amt <= balance) {
            balance -= amt
            true
        } else {
            false
        }
}
```

Three findings remain (`filename`, `class-naming`, `function-naming`) —
exactly the ones the tool flagged as unfixable. Fixing those by hand
(renaming the class/function/parameter and the file) and re-running
`ktlint` on the corrected version reports a clean pass.

## detekt: analysis beyond formatting

Where ktlint stops at style, `detekt` catches structural and
correctness-adjacent issues ktlint doesn't look for: cyclomatic
complexity above a threshold, empty `catch` blocks that silently
swallow exceptions, `TooManyFunctions` in one class, magic numbers with
no named constant, and nested code exceeding a configurable depth. A
`detekt.yml` configuration file turns rules on/off and tunes thresholds
per project:

```yaml
complexity:
  LongMethod:
    threshold: 40
  TooManyFunctions:
    thresholdInClasses: 15

style:
  MagicNumber:
    ignoreNumbers: ['-1', '0', '1', '2']

exceptions:
  TooGenericExceptionCaught:
    active: true
```

detekt reports findings the same shape as ktlint's — file, line, rule id
— and, like ktlint, supports a `--build-upon-default-config` mode plus a
`baseline.xml` that grandfathers in existing violations so a legacy
codebase can adopt it without failing on thousands of pre-existing
issues, while still failing CI on any *new* one.

## Wiring both into Gradle and CI

```kotlin
// build.gradle.kts
plugins {
    id("io.gitlab.arturbosch.detekt") version "1.23.6"
    id("org.jlleitschuh.gradle.ktlint") version "12.1.1"
}
```

```yaml
# .github/workflows/ci.yml (quality job)
- name: ktlint
  run: ./gradlew ktlintCheck
- name: detekt
  run: ./gradlew detekt
```

Both Gradle tasks exit non-zero on any violation, which is what makes
`ktlintCheck`/`detekt` steps actually fail a GitHub Actions run rather
than just print warnings — the same exit-code mechanic module 05's test
runner relied on to fail CI on a failing test.

## Traps

- **`ktlint -F` is not always safe to run unattended on someone else's
  in-progress change** — it can reformat lines a reviewer is mid-review
  on, producing a diff that's mostly noise. Running it as a pre-commit
  hook (so it only ever touches code before it's shared) avoids this.
- **detekt's default thresholds are tuned for general use, not your
  codebase.** A `LongMethod` threshold of 40 lines flags plenty of
  legitimate, hard-to-split methods (a `when` over 20 enum cases, say) —
  tuning `detekt.yml` per project, rather than fighting the defaults
  with suppressions everywhere, keeps the tool useful instead of ignored.
- **Suppressing a finding with `@Suppress("MagicNumber")` inline is
  easy to overuse** as an escape hatch that defeats the point of running
  the tool at all — a suppression should have a comment explaining *why*
  the number genuinely doesn't need a name, not just silence the
  warning.
- **Style violations found only in CI (not locally) waste review
  cycles** — running `ktlintCheck`/`detekt` as a local pre-commit hook or
  IDE plugin catches issues before a PR is even opened, instead of after
  a reviewer or CI run flags them.

## Cheat sheet

| Concern | Tool/command |
|---|---|
| Style/formatting check | `ktlint <file>` or `./gradlew ktlintCheck` |
| Auto-fix formatting | `ktlint -F <file>` |
| Static analysis (complexity, smells) | `detekt` or `./gradlew detekt` |
| Grandfather existing violations | detekt `baseline.xml` |
| Tune rule thresholds | `detekt.yml` |
| Fail CI on violations | Both tasks exit non-zero by default |
| Catch issues before PR | Pre-commit hook or IDE plugin |

## Exercise

Take the reformatted `MessyFixable.kt` from this module, manually rename
`userAccount` → `UserAccount`, `Deposit` → `deposit`, `Name` → `name`,
and the file to `UserAccount.kt`. Re-run `ktlint UserAccount.kt` and
confirm it reports zero violations; then add a `detekt.yml` with
`MagicNumber` active and add a method using a raw literal like `42` to
see it flagged, before extracting it into a named `const val`.
