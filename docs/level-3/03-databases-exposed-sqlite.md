# 03 · Databases (Exposed/SQLite)

The [Ktor module](02-building-apis-ktor.md) kept its "database" in an
in-memory `Map`. Real services need persistence — **Exposed** is
JetBrains's Kotlin SQL library, offering both a typed SQL DSL and a
lightweight DAO (Active Record-style) layer on top of plain JDBC. This
module uses SQLite as the backing database since it needs no server
process, but Exposed works the same way against Postgres, MySQL, or H2.

Like Ktor, Exposed needs a Gradle project with these dependencies:

```kotlin
val exposedVersion = "0.52.0"
dependencies {
    implementation("org.jetbrains.exposed:exposed-core:$exposedVersion")
    implementation("org.jetbrains.exposed:exposed-dao:$exposedVersion")
    implementation("org.jetbrains.exposed:exposed-jdbc:$exposedVersion")
    implementation("org.xerial:sqlite-jdbc:3.46.0.0")
}
```

## Defining a table and a DAO entity

A `Table` (or `IntIdTable`, which adds an auto-increment `id` column)
describes the schema. Pairing it with an `IntEntity`/`IntEntityClass` gives
you an object-oriented view over the rows.

```kotlin
import org.jetbrains.exposed.sql.*
import org.jetbrains.exposed.sql.transactions.transaction
import org.jetbrains.exposed.dao.id.IntIdTable
import org.jetbrains.exposed.dao.IntEntity
import org.jetbrains.exposed.dao.IntEntityClass
import org.jetbrains.exposed.dao.id.EntityID

object Employees : IntIdTable() {
    val name = varchar("name", 100)
    val department = varchar("department", 50)
    val salary = integer("salary")
}

class Employee(id: EntityID<Int>) : IntEntity(id) {
    companion object : IntEntityClass<Employee>(Employees)
    var name by Employees.name
    var department by Employees.department
    var salary by Employees.salary
}
```

`by Employees.name` is a Kotlin delegated property — reading/writing
`employee.name` actually reads/writes the underlying database column
through Exposed's `Column` delegate, no manual getter/setter plumbing
needed.

## Connecting, creating the schema, and CRUD

Every database operation runs inside `transaction { }`, which opens a JDBC
connection, runs the block, and commits (or rolls back on an exception).

```kotlin
fun main() {
    Database.connect("jdbc:sqlite:file:test?mode=memory&cache=shared", driver = "org.sqlite.JDBC")

    transaction {
        SchemaUtils.create(Employees)

        Employee.new { name = "Alice"; department = "Engineering"; salary = 95000 }
        Employee.new { name = "Bob"; department = "Sales"; salary = 65000 }
        Employee.new { name = "Cara"; department = "Engineering"; salary = 105000 }

        println("All employees:")
        Employee.all().forEach { println("  ${it.name} (${it.department}) - ${it.salary}") }

        println("Engineering only:")
        Employee.find { Employees.department eq "Engineering" }.forEach {
            println("  ${it.name} - ${it.salary}")
        }

        println("Raw SQL DSL, sorted by salary desc:")
        Employees.selectAll().orderBy(Employees.salary to SortOrder.DESC).forEach {
            println("  ${it[Employees.name]}: ${it[Employees.salary]}")
        }

        val engTotal = Employees.select(Employees.salary)
            .where { Employees.department eq "Engineering" }
            .sumOf { it[Employees.salary] }
        println("Engineering total payroll: $engTotal")

        val bob = Employee.find { Employees.name eq "Bob" }.first()
        bob.salary = 70000
        println("After raise, Bob: ${bob.salary}")

        Employee.find { Employees.name eq "Bob" }.first().delete()
        println("Remaining count: ${Employee.all().count()}")
    }
}
```

```text
All employees:
  Alice (Engineering) - 95000
  Bob (Sales) - 65000
  Cara (Engineering) - 105000
Engineering only:
  Alice - 95000
  Cara - 105000
Raw SQL DSL, sorted by salary desc:
  Cara: 105000
  Alice: 95000
  Bob: 65000
Engineering total payroll: 200000
After raise, Bob: 70000
Remaining count: 2
```

Two styles are used above: the **DAO** style (`Employee.new { }`,
`Employee.find { }`, mutating `bob.salary` directly) and the **DSL** style
(`Employees.selectAll()`, `Employees.select(...).where { }`) that reads
closer to raw SQL. Both operate on the same table — mix them freely
depending on whether you want object-like ergonomics or precise query
control.

## Kotlin-specific traps

- **Every DB call must be inside a `transaction { }` block.** Accessing an
  entity property (e.g. `bob.salary`) *outside* the transaction that
  fetched it throws `EntityNotFoundException` or a lazy-loading error —
  Exposed entities are transaction-scoped, not detached objects like some
  ORMs.
- **`eq` is an infix function, not `==`.** `Employees.department eq
  "Engineering"` builds a SQL predicate; `Employees.department ==
  "Engineering"` compares a `Column` object to a `String` and is always
  `false` (a common copy-paste trap coming from regular Kotlin code).
- **In-memory SQLite needs `cache=shared`.** Without it, each new JDBC
  connection gets its own private in-memory database and your writes
  "disappear" between transactions — `file:test?mode=memory&cache=shared`
  keeps one instance alive for the process.
- **`Employees.select(...)` (Exposed 0.47+) vs. the older `slice(...).select { }`.**
  Exposed's DSL API has shifted across versions; mixing tutorial code from
  different Exposed versions produces confusing "unresolved reference"
  errors. Pin a version and stick to its docs.
- **Delegated properties infer their type from the column.** `var salary
  by Employees.salary` is `Int`, matching `integer(...)` — declaring the
  entity property with a mismatched type is a compile error, which is
  actually a safety net compared to raw JDBC's stringly-typed result sets.

## Cheat sheet

| Concept | Exposed construct |
|---|---|
| Define schema | `object X : IntIdTable() { val col = varchar(...) }` |
| Create tables | `SchemaUtils.create(X)` inside a transaction |
| Wrap DB work | `transaction { ... }` |
| DAO entity | `class E(id) : IntEntity(id) { companion object : IntEntityClass<E>(X) }` |
| Insert (DAO) | `E.new { field = value }` |
| Query (DAO) | `E.find { X.col eq value }`, `E.all()` |
| Query (DSL) | `X.selectAll()`, `X.select(cols).where { }` |
| Update | mutate a fetched entity's `var` property |
| Delete | `entity.delete()` |

## Exercise

Add a `Department` table (`id`, `name`, `budget`) and a foreign key
`Employees.departmentId` referencing it (`reference("department_id",
Departments)`), replacing the plain `varchar` department column. Write a
query that, for each department, prints its name and the sum of its
employees' salaries as a fraction of its budget — using either the DAO or
DSL style, your choice.
