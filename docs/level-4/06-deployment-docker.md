# 06 · Deployment (Docker)

A Kotlin service that only runs on your laptop isn't shipped. Docker
packages the JVM, your compiled classes, and everything they need into an
image that runs identically on any machine — your laptop, a teammate's,
or a production host. This module builds a tiny HTTP-shaped service,
containerizes it with a multi-stage `Dockerfile`, and covers the traps
that make Kotlin/JVM images bigger or slower to start than they need to
be.

## The service

```kotlin
import java.time.Instant

data class HealthStatus(val status: String, val uptimeSeconds: Long, val checkedAt: String)

class Server(private val startedAt: Long = System.currentTimeMillis()) {
    fun health(): HealthStatus {
        val uptime = (System.currentTimeMillis() - startedAt) / 1000
        return HealthStatus("UP", uptime, Instant.now().toString())
    }

    fun greet(name: String): String {
        val port = System.getenv("PORT") ?: "8080"
        return "Hello, $name! Serving on port $port"
    }
}

fun main() {
    val server = Server()
    println(server.greet("world"))
    println(server.health())
}
```

Compiling to a runnable jar and executing it with an environment
variable set, exactly as a container would receive it:

```console
$ kotlinc App.kt -include-runtime -d app.jar
$ PORT=9090 java -jar app.jar
Hello, world! Serving on port 9090
HealthStatus(status=UP, uptimeSeconds=0, checkedAt=2026-08-27T06:17:25.764283Z)
```

`PORT` came from the shell here; in a container it comes from the
orchestrator (Docker Compose, Kubernetes, a PaaS) via `ENV` or `-e`, and
the code doesn't change at all — this is the whole point of reading
configuration from the environment instead of hardcoding it.

## Multi-stage Dockerfile

A naive Dockerfile installs the full Kotlin compiler *and* ships it in
the final image just to run a jar that's already compiled. A multi-stage
build separates "things needed to build" from "things needed to run":

```dockerfile
# Stage 1: compile with the full Kotlin/JDK toolchain
FROM gradle:8.7-jdk21 AS build
WORKDIR /app
COPY . .
RUN gradle build --no-daemon -x test

# Stage 2: run on a slim JRE only — no compiler, no Gradle, no source
FROM eclipse-temurin:21-jre-alpine
WORKDIR /app
COPY --from=build /app/build/libs/*-all.jar app.jar
ENV PORT=8080
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

The `COPY --from=build` line is the key mechanic: it reaches into the
first stage's filesystem and pulls out only the built jar, discarding
the ~600MB Gradle/JDK build stage entirely. The final image is built from
`eclipse-temurin:21-jre-alpine` (a JRE, not a full JDK, and Alpine Linux
rather than a full distro), which is typically 5-8x smaller than an image
that carries the whole build toolchain.

Building and running it locally follows the same commands regardless of
which machine executes them:

```console
$ docker build -t kotlin-service:latest .
$ docker run -p 8080:8080 -e PORT=8080 kotlin-service:latest
Hello, world! Serving on port 8080
HealthStatus(status=UP, uptimeSeconds=0, checkedAt=...)
```

## .dockerignore

Without one, `COPY . .` in the build stage sends the entire project
directory — including `.git`, `build/`, and any local `.gradle` cache —
into the Docker build context, slowing every build and risking secrets
or stale artifacts leaking into the image:

```
.git
.gradle
build
*.iml
.idea
Dockerfile
```

## JVM startup and container memory

Two JVM-specific traps show up almost immediately once a Kotlin service
runs inside a container instead of on bare metal:

- **The JVM used to ignore container memory limits entirely**, reading
  the *host's* total RAM via `/proc/meminfo` and sizing its default heap
  off that — a container capped at 512MB could still let the JVM think
  it had 32GB to work with, right up until it got OOM-killed by the
  container runtime. Modern JDKs (10+) are container-aware by default,
  but it's still worth being explicit: `-XX:MaxRAMPercentage=75.0` caps
  the heap at a percentage of the container's actual limit rather than
  trusting either side's defaults.
- **JVM startup time (JIT warmup) is a real cost in container
  orchestration**, where a scheduler expects a health check to pass
  within seconds. A Kotlin/JVM service that takes 3-4 seconds to become
  responsive needs `initialDelaySeconds` set accordingly in a Kubernetes
  readiness probe — or `-Xshare:auto` / a smaller heap for common cases
  — otherwise the orchestrator can kill and restart a container that was
  simply still starting up, a restart loop that looks like a crash but
  is actually an impatient health check.

## Traps

- **Layer ordering defeats Docker's build cache.** `COPY . .` before
  running `gradle build` means *any* source change invalidates every
  layer after it, forcing a full dependency re-download each build. A
  faster pattern copies `build.gradle.kts`/`settings.gradle.kts` first,
  runs a dependency-resolution step, and only then copies source — so
  editing a `.kt` file doesn't also re-fetch every Maven dependency.
- **Running as root inside the container** is the default unless a
  Dockerfile adds `USER appuser` after creating a non-root user — fine
  for a throwaway demo, a real liability if the container is ever
  compromised, since a root process inside the container that can escape
  to the host has full host privileges.
- **`EXPOSE` in a Dockerfile is documentation, not enforcement.** It
  doesn't open a port by itself; `docker run -p 8080:8080` (or the
  orchestrator's equivalent) is what actually maps the container's port
  to something reachable. Forgetting the `-p` flag produces a container
  that's running and healthy but completely unreachable from outside.
- **`-include-runtime` on `kotlinc` bundles the Kotlin stdlib into the
  jar** — convenient for a single-file demo like this module's, but a
  Gradle/Maven build with a `shadowJar`/`fatJar` plugin is the real-world
  equivalent, since it also merges dependency jars (not just the
  stdlib) into one runnable artifact, which is what `COPY --from=build`
  above expects to find.

## Cheat sheet

| Concern | Approach |
|---|---|
| Small final image | Multi-stage build, `COPY --from=build` |
| Base image for running | `eclipse-temurin:21-jre-alpine` (JRE, not JDK) |
| Exclude files from build context | `.dockerignore` |
| Cap JVM heap to container limit | `-XX:MaxRAMPercentage=75.0` |
| Map container port to host | `docker run -p HOST:CONTAINER` |
| Don't run as root | `USER appuser` in the Dockerfile |
| Fat jar with all dependencies | `shadowJar`/`-include-runtime` |

## Exercise

Add a third stage to the Dockerfile above that runs `gradle test` before
the final `FROM eclipse-temurin` stage, so `docker build` fails the whole
build if any test fails — not just the app compile. Then modify `Server`
to read a `SERVICE_NAME` environment variable (defaulting to
`"kotlin-service"`) and include it in `greet`'s output; rebuild the jar
with `kotlinc` and confirm `SERVICE_NAME=orders java -jar app.jar` prints
`orders` in its greeting.
