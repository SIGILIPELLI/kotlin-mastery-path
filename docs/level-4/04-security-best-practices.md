# 04 · Security Best Practices

Most application-level security bugs aren't exotic — they're plaintext
passwords, string-concatenated SQL, and predictable "random" tokens. This
module covers the three fixes that matter most in an everyday Kotlin
backend: password hashing, safe randomness, and parameterized queries
(which [Module 3 of Level 3](../level-3/03-databases-exposed-sqlite.md)
already used correctly, without calling out *why* it matters).

## Password hashing with bcrypt

Never store plaintext passwords, and never use a *fast* general-purpose
hash (MD5, SHA-256 alone) for them — fast hashes are exactly what makes
brute-forcing leaked password databases practical. bcrypt is deliberately
slow and bakes in a random salt automatically, so identical passwords
never produce identical hashes.

```kotlin
import at.favre.lib.crypto.bcrypt.BCrypt

fun main() {
    val password = "correct horse battery staple"
    val hash = BCrypt.withDefaults().hashToString(12, password.toCharArray())
    println("Hash: ${hash.take(29)}... (truncated, includes salt + cost factor)")

    val correctCheck = BCrypt.verifyer().verify(password.toCharArray(), hash)
    val wrongCheck = BCrypt.verifyer().verify("wrong password".toCharArray(), hash)
    println("Correct password verifies: ${correctCheck.verified}")
    println("Wrong password verifies: ${wrongCheck.verified}")

    val hash2 = BCrypt.withDefaults().hashToString(12, password.toCharArray())
    println("Same password, different hash each time: ${hash != hash2}")
}
```

```text
Hash: $2a$12$Vn/ZLatLYAYU15Zbc2nxQu... (truncated, includes salt + cost factor)
Correct password verifies: true
Wrong password verifies: false
Same password, different hash each time: true
```

The `12` passed to `hashToString` is bcrypt's **cost factor** — each
increment roughly doubles the work required to compute a hash, which is
exactly the property that makes brute-force attacks slower. Storing the
hash string alone is enough to verify later logins; the salt is embedded
in it, so no separate salt column is needed.

## Secure randomness

`kotlin.random.Random` and `java.util.Random` are **not** safe for
anything security-sensitive (session tokens, password reset codes, API
keys) — their output is predictable given enough samples or a known seed.
`java.security.SecureRandom` uses a cryptographically strong source.

```kotlin
import java.security.SecureRandom

val secureRandom = SecureRandom()
val tokenBytes = ByteArray(16)
secureRandom.nextBytes(tokenBytes)
val token = tokenBytes.joinToString("") { "%02x".format(it) }
println("Secure random token: $token (${token.length} hex chars)")
```

```text
Secure random token: 4171e82df5dbb0622cd222eed2093e42 (32 hex chars)
```

A 16-byte (128-bit) random token is effectively unguessable — the
important thing is using `SecureRandom` specifically, not the byte count
(which you can tune per use case).

## Parameterized queries prevent SQL injection

Exposed's `eq`, `like`, and friends (used throughout
[Level 3's database module](../level-3/03-databases-exposed-sqlite.md))
build **parameterized** queries under the hood — user input is passed to
the JDBC driver as a bound parameter, never spliced into the SQL string.
This makes classic SQL injection structurally impossible through the DSL,
regardless of what characters the input contains.

```kotlin
import org.jetbrains.exposed.sql.*
import org.jetbrains.exposed.sql.transactions.transaction
import org.jetbrains.exposed.dao.id.IntIdTable

object Users : IntIdTable() {
    val username = varchar("username", 50)
}

fun main() {
    Database.connect("jdbc:sqlite:file:secdemo?mode=memory&cache=shared", driver = "org.sqlite.JDBC")
    transaction {
        SchemaUtils.create(Users)
        Users.insert { it[username] = "alice" }
        Users.insert { it[username] = "bob" }

        val maliciousInput = "alice' OR '1'='1"
        val safeResult = Users.selectAll().where { Users.username eq maliciousInput }.count()
        println("Parameterized query with injection-attempt input: $safeResult rows (expected 0)")
    }
}
```

```text
Parameterized query with injection-attempt input: 0 rows (expected 0)
```

The classic injection payload `alice' OR '1'='1` matches zero rows here —
Exposed treats it as a literal username to search for, not as SQL syntax.
The vulnerable equivalent would be building a raw string like `"SELECT *
FROM Users WHERE username = '$maliciousInput'"` and executing it directly;
once concatenated, the query becomes `... WHERE username = 'alice' OR
'1'='1'`, which matches every row. Never build SQL by string concatenation
with any value that came from a user, ever — not even "just for an admin
tool."

## Kotlin-specific traps

- **`String.hashCode()` is not a cryptographic hash.** It's fast,
  unsalted, and has documented collisions — never use it (or `.hashCode()`
  on any type) for anything resembling a password or security token.
- **Comparing secrets with `==` can leak timing information.** String
  comparison in the JVM short-circuits on the first differing character,
  which — in security-sensitive contexts like comparing a computed HMAC
  against a stored one — can be exploited via timing attacks; use a
  constant-time comparison (e.g.
  `java.security.MessageDigest.isEqual(a, b)`) for that specific case.
  Password *verification* doesn't need this, since bcrypt's `verify`
  function already does constant-time comparison internally.
- **Logging full request/response bodies can leak secrets.** A `CallLogging`
  setup (from [Module 2](02-production-apis-ktor.md)) that logs raw
  bodies will happily log passwords, tokens, and PII sent in JSON — mask
  or exclude sensitive fields before logging in real services.
- **`data class` `toString()` includes every property by default** —
  including a `password` or `token` field, if one exists on the class.
  Exclude sensitive fields from `toString` (override it manually, or keep
  secrets in a separate, non-data class) so a stray `println(user)` or log
  statement doesn't leak them.
- **Hardcoded secrets (like the JWT `SECRET` constant in
  [Module 2](02-production-apis-ktor.md)) are fine for a demo, never for
  production** — real secrets belong in environment variables or a secrets
  manager, never committed to source control.

## Cheat sheet

| Concern | Do | Don't |
|---|---|---|
| Store a password | `BCrypt.withDefaults().hashToString(cost, pw)` | Plaintext, or MD5/SHA-256 alone |
| Generate a token/secret | `SecureRandom` | `Random`/`kotlin.random.Random` |
| Build a query with user input | Exposed DSL (`eq`, `like`), or JDBC `PreparedStatement` | String concatenation into SQL |
| Compare a secret/HMAC | `MessageDigest.isEqual(a, b)` | `==` / `.contentEquals()` |
| Log request data | Mask/exclude sensitive fields | Log raw bodies unconditionally |

## Exercise

Write a small `UserAccount` class with a `passwordHash: String` field
(never a raw `password`), a companion `fun register(username: String,
password: String): UserAccount` that hashes the password with bcrypt
before storing it, and a `fun login(password: String): Boolean` instance
method that verifies against the stored hash. Add a custom `toString()`
that prints the username but never the hash, and demonstrate that logging
a `UserAccount` via `println` never exposes the hash.
