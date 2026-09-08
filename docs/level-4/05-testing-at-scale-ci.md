# 05 · Testing at Scale & CI

A handful of test files ([Level 2](../level-2/05-scalatest-testing.md),
[Level 3](../level-3/05-testing-advanced.md)) run fast enough that nobody
thinks about *how* they run. A codebase with thousands of tests needs to
think about it: which tests are slow, which can run in parallel, and how a
CI pipeline enforces that every change actually gets tested before it
merges.

## Tagging tests to separate fast from slow

ScalaTest lets you tag individual tests, then run (or skip) by tag —
essential once a suite has both a millisecond unit test and a test that
spins up a real database or waits on I/O:

```scala
import org.scalatest.funsuite.AnyFunSuite
import org.scalatest.Tag

object SlowTest extends Tag("SlowTest")

class TaggedSpec extends AnyFunSuite:
  test("fast test") {
    assert(1 + 1 == 2)
  }

  test("slow test", SlowTest) {
    Thread.sleep(50)
    assert(true)
  }
```

```text title="sbt \"testOnly TaggedSpec -- -l SlowTest\""
[info] TaggedSpec:
[info] - fast test
[info] Tests: succeeded 1, failed 0, canceled 0, ignored 0, pending 0
```

`-l SlowTest` excludes anything tagged `SlowTest` — this is exactly what a
"fast feedback" CI stage runs on every push, saving the full (including
slow/integration) suite for a less frequent stage or before merging.

### The trap: an untagged slow test poisons the fast suite

One accidentally-untagged integration test that takes 30 seconds, added to
a suite that used to run in 2, turns a "run tests on every save" workflow
into one people stop bothering with. Tagging discipline — every test that
touches the network, a real database, or `Thread.sleep`-scale waiting gets
a tag — is what keeps the untagged (default, always-run) suite trustworthy
and fast.

## Parallel test execution

sbt runs test *suites* (classes) in parallel by default but tests *within*
one suite sequentially. For suites with genuinely independent tests,
enabling `ParallelTestExecution` on the suite splits its own tests across
threads too:

```scala
import org.scalatest.ParallelTestExecution

class FastSpec extends AnyFunSuite with ParallelTestExecution:
  // each test method runs on its own thread
```

### The trap: shared mutable state breaks under parallelism

A suite with a `var` field or a shared mutable collection referenced by
multiple tests works by accident when tests run sequentially and breaks
unpredictably — different results on different runs — once they run in
parallel. `ParallelTestExecution` (and cross-suite parallelism, sbt's
default) requires each test to set up and own its own state; a shared
`Db.conn` written to Level 3's project needs a check here before you
parallelize its test suite.

## What CI actually enforces

A CI pipeline (GitHub Actions, in this example) runs on every push and pull
request, giving everyone the same guarantee a passing build on your laptop
doesn't: the code compiles and tests pass on a clean checkout, not just
"on my machine."

```yaml title=".github/workflows/ci.yml"
name: CI
on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: '17'
      - name: Run fast tests
        run: sbt "testOnly * -- -l SlowTest"
      - name: Run full suite
        run: sbt test
```

Splitting into two steps mirrors the tag split above: a fast step that
fails quickly on the common case, and a full step (including slow/tagged
tests) that still has to pass before the change is considered good. A
failing "Run fast tests" step should block the workflow before ever
spending time on the slow one — `run:` steps in GitHub Actions already
fail the job (and stop subsequent steps in most default configurations) the
moment `sbt` exits non-zero.

## How It Actually Works

Test tags in ScalaTest compile down to plain string constants attached as
annotations/metadata on each registered test entry (the same
"registered-at-construction-time" list described in [Level 2's testing
module](../level-2/05-scalatest-testing.md)) — `sbt "testOnly * --
-l SlowTest"` filters that list by tag string before running anything,
which is why an untagged slow test can't be excluded: there's no metadata
for the filter to match against, so it always lands in the "everything
else" bucket the fast suite runs.

Parallel test execution in sbt works by running multiple test classes'
JVM-level test methods concurrently across a thread pool sized to
available cores — but critically, parallelism in sbt's default test
runner is typically at the *suite* level (different test classes run
concurrently with each other), not necessarily within a single suite's
individual tests. The mutable-shared-state trap arises because two test
suites running truly concurrently can both read/write the same
`object`-held mutable field or the same on-disk file/database row at the
same time — since (as established in [Level 2](../level-2/05-scalatest-testing.md))
a suite's tests execute against shared instance state by default, and
`object`s are process-wide singletons (see [Level 1's classes and
objects module](../level-1/06-classes-objects.md)), any `var` on an
`object` used by tests is exactly the kind of shared mutable state two
threads can race on with no synchronization, producing intermittent,
hard-to-reproduce failures that only show up under parallel execution.

"What CI actually enforces" is, mechanically, just running the exact same
`sbt compile`/`sbt test` commands you'd run locally, on a fresh,
disposable environment/container with none of your local machine's
incidental state (stale `target/` artifacts, environment variables,
locally-cached test data) — its value comes entirely from reproducibility
of the build/test pipeline, not from any testing capability sbt itself
doesn't already have.

## Cheat sheet

| Need to... | Use |
|---|---|
| Mark a test as slow/integration | `object SlowTest extends Tag("SlowTest")`, pass it to `test(name, SlowTest)` |
| Run everything except tagged tests | `sbt "testOnly * -- -l SlowTest"` |
| Run only a tagged group | `sbt "testOnly * -- -n SlowTest"` |
| Parallelize tests within one suite | mix in `ParallelTestExecution` (only if tests own their state) |
| Guarantee every push is tested | a CI workflow that runs `sbt test` on a clean checkout |
| Fail fast in CI | run a quick, untagged-only step before the full suite |

## Exercise

Take the `LibSpec` from [Level 3's testing module](../level-3/05-testing-advanced.md)
and add an `IntegrationTest` tag to the mock-based test (pretend it hits a
real service instead of a mock). Confirm `sbt "testOnly * -- -l
IntegrationTest"` skips it while the property-based test still runs. Then
write a `ci.yml` workflow (as above) with a third step that runs `sbt
scalafmtCheckAll` (or another formatting/lint check if you don't have
scalafmt configured) *before* the test steps, so a badly formatted PR fails
fast without ever running the test suite.
