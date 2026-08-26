# 02 · Production APIs with Ktor

[Level 3's Ktor module](../level-3/02-building-apis-ktor.md) built routing,
JSON, and basic error handling. A service headed for production needs
more: authentication, request validation, rate limiting, and structured
logging. This module adds all four to a small API, with a real client
exercising every path.

```kotlin
dependencies {
    implementation("io.ktor:ktor-server-core-jvm:$ktorVersion")
    implementation("io.ktor:ktor-server-netty-jvm:$ktorVersion")
    implementation("io.ktor:ktor-server-content-negotiation-jvm:$ktorVersion")
    implementation("io.ktor:ktor-serialization-kotlinx-json-jvm:$ktorVersion")
    implementation("io.ktor:ktor-server-status-pages-jvm:$ktorVersion")
    implementation("io.ktor:ktor-server-auth-jvm:$ktorVersion")
    implementation("io.ktor:ktor-server-auth-jwt-jvm:$ktorVersion")
    implementation("io.ktor:ktor-server-rate-limit-jvm:$ktorVersion")
    implementation("io.ktor:ktor-server-call-logging-jvm:$ktorVersion")
    implementation("io.ktor:ktor-server-request-validation-jvm:$ktorVersion")
}
```

## JWT authentication

`Authentication` with a `jwt(...)` provider verifies a bearer token's
signature and issuer, then hands you a `JWTPrincipal` inside `authenticate
{ }` blocks. Issuing tokens (`/login`) is just building and signing a JWT
with the `auth0` JWT library that `ktor-server-auth-jwt` pulls in.

```kotlin
import io.ktor.server.auth.*
import io.ktor.server.auth.jwt.*
import com.auth0.jwt.JWT
import com.auth0.jwt.algorithms.Algorithm
import java.util.Date

private const val SECRET = "demo-secret-do-not-use-in-real-code"
private const val ISSUER = "book-service"

fun generateToken(username: String): String =
    JWT.create()
        .withIssuer(ISSUER)
        .withClaim("username", username)
        .withExpiresAt(Date(System.currentTimeMillis() + 60_000))
        .sign(Algorithm.HMAC256(SECRET))

fun Application.module() {
    install(Authentication) {
        jwt("auth-jwt") {
            verifier(JWT.require(Algorithm.HMAC256(SECRET)).withIssuer(ISSUER).build())
            validate { credential ->
                if (credential.payload.getClaim("username").asString() != null) JWTPrincipal(credential.payload) else null
            }
        }
    }
    routing {
        post("/login") {
            val req = call.receive<LoginRequest>()
            call.respond(TokenResponse(generateToken(req.username)))
        }
        authenticate("auth-jwt") {
            get("/me") {
                val principal = call.principal<JWTPrincipal>()!!
                call.respond(mapOf("username" to principal.payload.getClaim("username").asString()))
            }
        }
    }
}
```

The secret used to *sign* tokens in `generateToken` and the one used to
*verify* them in the `jwt("auth-jwt") { }` block must be the same
(`Algorithm.HMAC256(SECRET)` in both places) — a mismatched secret (a
common copy-paste error when moving from a demo to environment-variable
based config) fails validation silently, returning 401 with no further
detail unless you add logging inside `validate { }`.

## Request validation and rate limiting

`RequestValidation` runs a check against a deserialized body *before* your
route handler runs, turning a bad request into a clean 400. `RateLimit`
caps how often a client can hit a given route group within a window,
returning 429 once exhausted.

```kotlin
import io.ktor.server.plugins.requestvalidation.*
import io.ktor.server.plugins.ratelimit.*
import kotlin.time.Duration.Companion.seconds

install(RequestValidation) {
    validate<Order> { order ->
        if (order.quantity <= 0) ValidationResult.Invalid("quantity must be positive")
        else ValidationResult.Valid
    }
}

install(RateLimit) {
    register(RateLimitName("public")) {
        rateLimiter(limit = 3, refillPeriod = 60.seconds)
    }
}

install(StatusPages) {
    exception<RequestValidationException> { call, cause ->
        call.respond(HttpStatusCode.BadRequest, mapOf("errors" to cause.reasons))
    }
    status(HttpStatusCode.TooManyRequests) { call, status ->
        call.respond(status, mapOf("error" to "rate limit exceeded, slow down"))
    }
}

routing {
    rateLimit(RateLimitName("public")) {
        get("/public/ping") { call.respond(mapOf("message" to "pong")) }
    }
    post("/orders") {
        val order = call.receive<Order>()
        call.respond(HttpStatusCode.Created, order)
    }
}
```

Running the full service (JWT login, an authenticated route, order
validation, and five rapid requests against the rate-limited endpoint)
against a real `HttpClient`:

```text
Login -> token issued (len=159)
GET /me with valid token -> status=200 OK, body={"username":"alice"}
GET /me with no token -> status=401 Unauthorized
POST /orders with quantity=0 -> status=400 Bad Request, body={"errors":["quantity must be positive"]}
POST /orders with quantity=5 -> status=201 Created
GET /public/ping attempt 1 -> status=200 OK
GET /public/ping attempt 2 -> status=200 OK
GET /public/ping attempt 3 -> status=200 OK
GET /public/ping attempt 4 -> status=429 Too Many Requests
GET /public/ping attempt 5 -> status=429 Too Many Requests
```

`limit = 3` means the fourth request within the window is rejected —
exactly what happens above. Note `POST /orders` isn't behind
`rateLimit(...)` at all in this example, so it isn't throttled; rate
limiting in Ktor is opt-in per route group via `rateLimit(RateLimitName)
{ ... }`, not global by default.

## Kotlin-specific traps

- **`ktor-server-rate-limit`'s `refillPeriod` takes `kotlin.time.Duration`**,
  not `java.time.Duration` — importing the wrong `Duration` (an easy
  mistake, since both are common) produces a confusing "expected
  kotlin.time.Duration" type error rather than a missing-import error.
- **The call-logging package name is `callloging` (one "l"), not
  `calllogging`** — a genuinely easy typo, and one that produces
  "unresolved reference" rather than a clearer "package not found."
- **`install(RequestValidation)` only validates types you explicitly
  register with `validate<T> { }`.** A different `@Serializable` type
  with no registered validator sails through unchecked — it's opt-in per
  type, not a blanket validation-on-by-default plugin.
- **JWT signing and verification must use matching `Algorithm` instances
  built from the same secret.** They don't need to be the *same object*,
  just constructed with equal parameters — a subtly different secret
  (extra whitespace from an environment variable, for instance) fails
  every token verification with a generic 401, not a descriptive error.
- **`authenticate("auth-jwt") { }` wraps a whole routing subtree** — routes
  defined outside it are simply unauthenticated, which is easy to forget
  when adding a new route "near" existing authenticated ones but outside
  the block.

## Cheat sheet

| Concern | Plugin/API |
|---|---|
| Verify a bearer JWT | `install(Authentication) { jwt("name") { verifier(...); validate { } } }` |
| Protect routes | `authenticate("name") { get("/x") { ... } }` |
| Access the verified token | `call.principal<JWTPrincipal>()` |
| Validate request bodies | `install(RequestValidation) { validate<T> { } }` |
| Map validation failures | `exception<RequestValidationException> { }` in `StatusPages` |
| Throttle a route group | `install(RateLimit) { register(...) }` + `rateLimit(name) { }` |
| Structured request logs | `install(CallLogging) { level = Level.INFO }` |

## Exercise

Add a `role` claim to the JWT (`.withClaim("role", "admin")`), and a second
authentication provider or a manual check inside `/me` that returns 403
(not 401) if the token's `role` claim isn't `"admin"`. Write client code
that logs in as a non-admin user, calls the protected route, and confirms
it gets 403 rather than 200.
