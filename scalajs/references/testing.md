# Testing Scala.js Applications

Scala.js tests run on Node.js (by default) or in a browser environment. The compiler and linker run your tests through the same pipeline as production code, so tests catch JS interop issues as well as logic bugs.

## MUnit (recommended for Scala 3)

```scala
// build.sbt
libraryDependencies += "org.scalameta" %%% "munit" % "1.0.0" % Test
```

```scala
// src/test/scala/myapp/ModelTest.scala
package myapp

class ModelTest extends munit.FunSuite:

  test("basic assertion") {
    val result = 2 + 2
    assertEquals(result, 4)
  }

  test("async test with Future") {
    val f = Future.successful(42)
    f.map { v => assertEquals(v, 42) }
  }
```

Run: `sbt test` or `mill app.test`

## uTest (popular with Mill)

```scala
// build.sbt / Mill ivyDeps
"com.lihaoyi" %%% "utest" % "0.8.4" % Test
```

```scala
import utest.*

object MyTests extends TestSuite:
  val tests = Tests {
    test("addition") { assert(1 + 1 == 2) }
    test("string") { assert("hello".length == 5) }
  }
```

## ScalaTest

```scala
"org.scalatest" %%% "scalatest" % "3.2.19" % Test
```

```scala
import org.scalatest.funsuite.AnyFunSuite

class MySpec extends AnyFunSuite:
  test("my test") {
    assert(true)
  }
```

## Testing async code

```scala
// MUnit with Future (returns Future from test body)
class AsyncTest extends munit.FunSuite:
  test("async") {
    val f: Future[Int] = fetchSomething()
    f.map { result =>
      assertEquals(result, 42)
    }
  }
```

## Testing Laminar components

Laminar doesn't have a dedicated test library for component rendering, but you can test model logic (pure functions, `Var`/`Signal`) independently:

```scala
class LaminarModelTest extends munit.FunSuite:
  test("Var updates") {
    val counter = Var(0)
    counter.update(_ + 1)
    // Signal.now() is OK in tests
    assertEquals(counter.signal.now(), 1)
  }

  test("Signal derives correctly") {
    val name = Var("Alice")
    val greeting = name.signal.map(n => s"Hello, $n!")
    assertEquals(greeting.now(), "Hello, Alice!")
    name.set("Bob")
    assertEquals(greeting.now(), "Hello, Bob!")
  }
```

## jsdom — browser-like environment

To test DOM interactions without a browser, use jsdom:

```scala
// sbt
jsEnv := new org.scalajs.jsenv.jsdomnodejs.JSDOMNodeJSEnv()
// Requires: npm install jsdom
// And: addSbtPlugin("org.scala-js" % "sbt-jsdependencies" % "1.0.2") — check version
```

## Selenium — real browser testing

```scala
import org.scalajs.jsenv.selenium.SeleniumJSEnv
import org.openqa.selenium.chrome.ChromeOptions

jsEnv := new SeleniumJSEnv(new ChromeOptions())
```

## Test-only dependencies

In sbt, use the `% Test` scope:
```scala
libraryDependencies += "org.scalameta" %%% "munit" % "1.0.0" % Test
```

In Mill, use a nested `test` object:
```scala
object test extends ScalaJSTests with TestModule.MUnit {
  def ivyDeps = super.ivyDeps() ++ Agg(ivy"org.scalameta::munit::1.0.0")
}
```

## Running specific tests

```bash
# sbt
sbt "testOnly myapp.ModelTest"
sbt "testOnly myapp.* -- --include myTestName"

# Mill
mill app.test myapp.ModelTest
```

## CI considerations

- Node.js must be installed in CI (GitHub Actions: `actions/setup-node`)
- Tests run via `sbt test` — no special browser required for Node.js-based tests
- For jsdom/Selenium tests, install the respective dependencies
