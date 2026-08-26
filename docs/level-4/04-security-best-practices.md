# 04 · Security Best Practices

Levels 1-3 focused on making code work. Production Kotlin code also has
to survive being attacked: leaked secrets, weak password storage,
malformed input from the network, and vulnerable dependencies. This
module covers four concrete defenses — password hashing, symmetric
encryption, input validation, and dependency auditing — each with runnable
code.

```kotlin
dependencies {
    // Nothing beyond the JDK is required for hashing/encryption below —
    // javax.crypto and java.security ship with every JVM.
}
```

## Hashing passwords with PBKDF2

Never store a password, and never encrypt one either (encryption is
reversible; a leaked key un-does it). Hash it with a slow, salted
algorithm. `PBKDF2WithHmacSHA256`, built into the JDK, is a solid default
when you don't want an extra dependency (bcrypt/argon2 libraries are
better in production, but the principle — salt + iteration count — is
identical).

```kotlin
import java.security.SecureRandom
import javax.crypto.SecretKeyFactory
import javax.crypto.spec.PBEKeySpec
import java.util.Base64

fun hashPassword(password: String, salt: ByteArray): String {
    val spec = PBEKeySpec(password.toCharArray(), salt, 65536, 256)
    val factory = SecretKeyFactory.getInstance("PBKDF2WithHmacSHA256")
    val hash = factory.generateSecret(spec).encoded
    return Base64.getEncoder().encodeToString(hash)
}

fun newSalt(): ByteArray = ByteArray(16).also { SecureRandom().nextBytes(it) }
```

The salt must be random *per user* and stored alongside the hash (it
isn't secret, it just has to be unique) — reusing one global salt across
all users defeats the point, since an attacker who cracks one hash gets a
lookup table for every account that shares it. The iteration count
(`65536` here) is deliberately expensive: hashing should take
milliseconds for a real login but far too long to brute-force billions of
guesses.

## Encrypting data that must come back out

Some data — a third-party API token you need to send again later, a
field a support tool must be able to display — has to be reversible.
AES-GCM is the standard choice: it's authenticated, meaning tampering
with the ciphertext makes decryption fail loudly instead of silently
returning garbage.

```kotlin
import java.security.SecureRandom
import javax.crypto.Cipher
import javax.crypto.spec.SecretKeySpec
import javax.crypto.spec.GCMParameterSpec

fun encrypt(plaintext: String, key: SecretKeySpec): Pair<ByteArray, ByteArray> {
    val iv = ByteArray(12).also { SecureRandom().nextBytes(it) }
    val cipher = Cipher.getInstance("AES/GCM/NoPadding")
    cipher.init(Cipher.ENCRYPT_MODE, key, GCMParameterSpec(128, iv))
    return cipher.doFinal(plaintext.toByteArray()) to iv
}

fun decrypt(ciphertext: ByteArray, key: SecretKeySpec, iv: ByteArray): String {
    val cipher = Cipher.getInstance("AES/GCM/NoPadding")
    cipher.init(Cipher.DECRYPT_MODE, key, GCMParameterSpec(128, iv))
    return String(cipher.doFinal(ciphertext))
}
```

The IV (initialization vector) must never repeat for the same key —
generating a fresh random one per call, as above, and storing it
alongside the ciphertext (it isn't secret either) is the standard
pattern. The *key* itself, unlike the IV or salt, is the one thing here
that genuinely must stay secret — see the secrets-management note below.

## Validating and rejecting untrusted input

Anything arriving over the network — a Ktor request body, a query
parameter — is attacker-controlled until proven otherwise. Validate
before you use it, and return every problem at once rather than the
first one, so a client can fix a request in one round trip.

```kotlin
data class SignupRequest(val username: String, val email: String, val age: Int)

sealed class ValidationResult {
    object Valid : ValidationResult() { override fun toString() = "Valid" }
    data class Invalid(val errors: List<String>) : ValidationResult()
}

fun validateSignup(req: SignupRequest): ValidationResult {
    val errors = mutableListOf<String>()
    if (req.username.length !in 3..20) errors += "username must be 3-20 chars"
    if (!Regex("^[a-zA-Z0-9_]+$").matches(req.username)) errors += "username has invalid characters"
    if (!Regex("^[^@\\s]+@[^@\\s]+\\.[^@\\s]+$").matches(req.email)) errors += "email is malformed"
    if (req.age < 13) errors += "age must be 13 or older"
    return if (errors.isEmpty()) ValidationResult.Valid else ValidationResult.Invalid(errors)
}
```

In a real Ktor service this plugs into the `RequestValidation` plugin
from [module 02](02-production-apis-ktor.md), rejecting bad requests with
a 400 before a handler ever sees them — validation belongs at the edge,
not scattered through business logic.

Running everything together:

```kotlin
fun main() {
    val salt = newSalt()
    val hash1 = hashPassword("correct horse battery staple", salt)
    val hash2 = hashPassword("correct horse battery staple", salt)
    val hash3 = hashPassword("wrong password", salt)
    println("Same password, same salt -> equal hashes: ${hash1 == hash2}")
    println("Different password -> different hash: ${hash1 != hash3}")

    val keyBytes = ByteArray(32).also { SecureRandom().nextBytes(it) }
    val key = SecretKeySpec(keyBytes, "AES")
    val (cipherText, iv) = encrypt("db-password=hunter2", key)
    val roundTrip = decrypt(cipherText, key, iv)
    println("Decrypted matches original: ${roundTrip == "db-password=hunter2"}")

    val good = validateSignup(SignupRequest("alice_92", "alice@example.com", 22))
    val bad = validateSignup(SignupRequest("a b", "not-an-email", 9))
    println("Good signup: $good")
    println("Bad signup: $bad")
}
```

```text
Same password, same salt -> equal hashes: true
Different password -> different hash: true
Decrypted matches original: true
Good signup: Valid
Bad signup: Invalid(errors=[username has invalid characters, email is malformed, age must be 13 or older])
```

## Secrets management and dependency auditing

Two more practices don't fit into runnable snippets but matter as much
as the code above:

- **Never hardcode secrets.** The `key`/`SECRET` constants in this
  module and in [module 02](02-production-apis-ktor.md) are for
  demonstration only — real services read them from environment
  variables or a secrets manager (Vault, AWS Secrets Manager, GCP Secret
  Manager) so a leaked repository doesn't leak production credentials.
  `System.getenv("DB_PASSWORD")` costs nothing and is the minimum bar.
- **Audit dependencies for known vulnerabilities.** `./gradlew
  dependencyCheckAnalyze` (OWASP Dependency-Check Gradle plugin) or
  GitHub's built-in Dependabot alerts scan your dependency tree against
  CVE databases. Kotlin's transitive dependency graph through
  Ktor/coroutines/serialization can be deep — a vulnerable version of a
  library three levels down is easy to miss without automated scanning,
  and easy to catch with it wired into CI.

## Kotlin-specific traps

- **`data class` prints field values in `toString()` by default** —
  convenient for logging, dangerous for a `data class User(val password:
  String)`. Logging such an object (directly, or via a framework that
  calls `toString()` on request/response bodies) leaks the password into
  log files. Override `toString()` to redact sensitive fields, or keep
  them out of `data class`es that get logged.
- **String immutability defeats "clearing" a password from memory.**
  `CharArray` (as `PBEKeySpec` above requires) can be zeroed out after
  use; a Kotlin `String` holding a password cannot — it lives on the
  heap until GC'd, and a heap dump in that window exposes it. Prefer
  `CharArray` for secrets that pass through your own code.
- **A `sealed class` used for validation results, as above, forces
  exhaustiveness at compile time** — a `when` over `ValidationResult`
  without an `else` fails to compile if a new subtype is added later,
  catching a forgotten security check before it ships.
- **`Regex` email "validation" is inherently approximate** — the pattern
  above rejects obviously malformed input but is not a complete RFC 5322
  implementation. Treat client-side/API-level email regex as a UX
  sanity check, and confirm real ownership with a verification email,
  not the regex.
- **AES-GCM silently fails differently than AES-CBC** — decrypting
  tampered GCM ciphertext throws `AEADBadTagException` immediately,
  which is the *desired* behavior (tamper detection); reaching for the
  older `AES/CBC/PKCS5Padding` because "it doesn't throw as much" removes
  that protection rather than fixing a bug.

## Cheat sheet

| Concern | Kotlin/JVM approach |
|---|---|
| Password storage | `PBKDF2WithHmacSHA256` (or bcrypt/argon2 lib) + per-user salt |
| Reversible secret storage | AES/GCM with a random IV per encryption |
| Untrusted input | Validate at the edge, collect all errors, reject before logic runs |
| Secrets in code | Environment variables / secrets manager, never literals |
| Vulnerable dependencies | `dependencyCheckAnalyze` or Dependabot in CI |
| Logging objects with secrets | Override `toString()` to redact, or exclude the field |

## Exercise

Add a `redactedToString()` extension on `SignupRequest` that returns the
same format as the default `toString()` but replaces the `email` value
with `"***"`. Call it from `main()` on the `good` request's underlying
`SignupRequest` and confirm the printed line no longer contains
`alice@example.com`.
