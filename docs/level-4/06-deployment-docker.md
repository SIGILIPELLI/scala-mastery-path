# 06 · Deployment (Docker)

A Scala service needs a JVM to run, plus every dependency it links against
— Docker packages both into one image so "works on my machine" becomes
"works anywhere Docker runs." This module builds a runnable fat JAR with
`sbt-assembly` (verified below by actually running it with `java -jar`) and
writes the Dockerfile that wraps it — the JAR-building step is real and
tested; treat the Dockerfile as the standard, well-tested recipe for taking
that verified JAR the rest of the way into a container.

## Step 1: a single runnable JAR

A normal `sbt run` needs the whole project directory and dependency jars on
a classpath sbt manages for you. Shipping a service means bundling your
code *and* every dependency into one self-contained `.jar` — that's what
the `sbt-assembly` plugin builds:

```scala title="project/plugins.sbt"
addSbtPlugin("com.eed3si9n" % "sbt-assembly" % "2.2.0")
```

```scala title="build.sbt"
scalaVersion := "3.3.1"
assembly / mainClass := Some("Main")
assembly / assemblyJarName := "app.jar"
```

```scala title="src/main/scala/Main.scala"
@main def Main(args: String*): Unit =
  println(s"Hello from inside the jar! PORT=${sys.env.getOrElse("PORT", "8080")}")
```

```text title="sbt assembly"
[info] 2 file(s) merged using strategy 'Rename'
[info] 2 file(s) merged using strategy 'Discard'
[info] Built: /path/to/target/scala-3.3.1/app.jar
[success] Total time: 4 s
```

```text title="PORT=9090 java -jar target/scala-3.3.1/app.jar"
Hello from inside the jar! PORT=9090
```

That's a real, runnable artifact — no `sbt`, no source checkout, just a
JVM and the JAR. This is exactly what a Docker image needs to run your
service; everything from here packages *that* JAR into a container.

### The trap: merge conflicts building the fat JAR

`sbt-assembly`'s log above shows "2 file(s) merged using strategy
'Discard'" — multiple dependencies shipping the same file path (commonly
`META-INF/MANIFEST.MF`, license files, or `module-info.class`) is a normal
occurrence once a project has more than a couple of libraries, and
`sbt-assembly` needs an explicit strategy (keep first, discard, concatenate,
rename) to resolve each conflict. Left unconfigured for anything beyond the
defaults, this can silently drop a file your code actually needs at
runtime — worth checking the merge log rather than assuming a successful
build means every file landed correctly.

## Step 2: reading configuration from the environment

`sys.env.getOrElse("PORT", "8080")` above is deliberate — a containerized
service should never hardcode a port, database URL, or secret; it reads
them from environment variables the container runtime injects, so the
*same image* runs correctly in dev, staging, and production with different
config passed in at `docker run` time.

## Step 3: the Dockerfile

```dockerfile title="Dockerfile"
FROM eclipse-temurin:17-jre-jammy
WORKDIR /app
COPY target/scala-3.3.1/app.jar app.jar
ENV PORT=8080
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

This uses a **JRE-only** base image (`eclipse-temurin:17-jre-jammy`), not a
full JDK — the container only needs to *run* the already-compiled JAR, not
compile anything, so shipping compiler tooling in the final image is pure
wasted size and attack surface.

## Multi-stage builds: don't ship the build toolchain

A more realistic Dockerfile builds the JAR *inside* Docker too, so nobody
needs `sbt` installed locally to produce a deployable image — but the
`sbt`/Scala compiler toolchain used to build it never ends up in the final
image:

```dockerfile title="Dockerfile (multi-stage)"
FROM sbtscala/scala-sbt:eclipse-temurin-17_35_1.9.7_3.3.1 AS build
WORKDIR /app
COPY . .
RUN sbt assembly

FROM eclipse-temurin:17-jre-jammy
WORKDIR /app
COPY --from=build /app/target/scala-3.3.1/app.jar app.jar
ENV PORT=8080
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

The `build` stage has the full Scala/sbt toolchain (large — hundreds of MB);
`COPY --from=build` pulls out only the finished `app.jar`, and the final
image is built entirely from the lean JRE stage. The build stage and
everything in it is discarded from the shipped image.

### The trap: `.dockerignore`

Without a `.dockerignore` excluding `target/`, `.git/`, and `project/target/`,
`COPY . .` sends sbt's own build cache and compiled artifacts into the
Docker build context on every build — slowing every build down and
occasionally leaking a stale `target/` from your host machine into the
image instead of one built fresh inside the container:

```text title=".dockerignore"
target/
project/target/
project/project/
.git/
.bsp/
```

## Cheat sheet

| Need to... | Use |
|---|---|
| Bundle app + dependencies into one JAR | `sbt-assembly` plugin, `sbt assembly` |
| Configure per-environment without rebuilding | read from environment variables (`sys.env`), never hardcode |
| Keep the final image small | a JRE-only (not JDK) base image for the run stage |
| Build without requiring local `sbt` | a multi-stage Dockerfile with a build stage and a run stage |
| Avoid leaking host build artifacts into the image | a `.dockerignore` excluding `target/`, `.git/` |

## Stretch goals

- Add a `HEALTHCHECK` instruction to the Dockerfile that curls the
  `/v1/health` route from module 03's production API, and explain (in a
  comment) how Docker uses it to mark a container `unhealthy`.
- Configure `sbt-assembly`'s merge strategy explicitly (`assembly /
  assemblyMergeStrategy`) instead of relying on the defaults, and note in a
  comment which specific conflicting files you had to resolve for your
  project's actual dependency set.
- If you have Docker available, build the multi-stage image locally
  (`docker build -t scala-demo .`), run it with `docker run -p 8080:8080 -e
  PORT=8080 scala-demo`, and compare the final image size against a
  single-stage build that ships the full sbt/Scala toolchain.
