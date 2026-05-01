# JavaScript Interoperability Reference

Scala.js provides multiple ways to interact with JavaScript. Choose the approach that matches your type-safety needs.

## 1. Façade Traits (typed, zero-overhead)

Façades are Scala types that describe the shape of a JavaScript object without adding any runtime overhead.

### Native JS trait (read-only API description)
```scala
import scala.scalajs.js
import scala.scalajs.js.annotation.*

@js.native
trait MyLibOptions extends js.Object {
  val timeout: Int = js.native
  val baseUrl: String = js.native
}
```

### Native JS class (instantiable)
```scala
@js.native
@JSGlobal("MyLib")
class MyLib(options: MyLibOptions) extends js.Object {
  def doSomething(x: String): js.Promise[String] = js.native
  def getValue: Int = js.native
}

// Usage:
val lib = new MyLib(js.Dynamic.literal(timeout = 5000, baseUrl = "/api").asInstanceOf[MyLibOptions])
```

### Native JS object (singleton)
```scala
@js.native
@JSGlobal("JSON")
object JSON extends js.Object {
  def parse(text: String): js.Any = js.native
  def stringify(value: js.Any): String = js.native
}
```

### Importing from an ES module
```scala
// Equivalent to: import MyLib from "my-lib"
@js.native
@JSImport("my-lib", JSImport.Default)
class MyLib extends js.Object {
  def method(): Unit = js.native
}

// Equivalent to: import { namedExport } from "my-lib"
@js.native
@JSImport("my-lib", "namedExport")
def namedExport: js.Function1[String, Unit] = js.native

// Importing a non-class value (e.g., svg, json asset)
@js.native
@JSImport("/assets/logo.svg", JSImport.Default)
val logoPath: String = js.native
```

## 2. js.Dynamic (untyped, quick access)

Use for rapid prototyping or one-off accesses when no façade exists:

```scala
import scala.scalajs.js

val dyn = js.Dynamic.global
dyn.console.log("hello from dynamic")
val result = dyn.myLib.doSomething("arg").asInstanceOf[String]

// Create a JS object literal dynamically:
val opts = js.Dynamic.literal(
  timeout = 1000,
  retries = 3
)
```

**Warning**: `js.Dynamic` bypasses the type checker — use façades in production code.

## 3. js.Object Literals (typed object creation)

When you need to *create* a JavaScript object with known fields (not just describe one):

```scala
import scala.scalajs.js

// Approach A: js.Dynamic.literal (returns js.Dynamic)
val obj = js.Dynamic.literal(name = "Alice", age = 30)

// Approach B: typed anonymous object (Scala.js-defined JS class)
@js.native
trait Person extends js.Object {
  val name: String = js.native
  val age: Int = js.native
}

val person = new js.Object { val name = "Alice"; val age = 30 }.asInstanceOf[Person]

// Approach C: use js.Object.freeze for truly typed literal creation
// Prefer Approach B or a case class + upickle serialization for fullstack apps
```

## 4. Scala.js-Defined JS Classes (non-native)

When you want to *implement* a JS interface from Scala code:

```scala
// A Scala.js class that *is* a JS object (no @js.native)
class ScalaPoint(val x: Double, val y: Double) extends js.Object

val p = new ScalaPoint(1.0, 2.0)
// p is a real JS object: { x: 1.0, y: 2.0 }
```

Use this when you need to pass Scala objects into JS callbacks that expect plain JS objects.

## 5. Calling Scala.js from JavaScript

### Export a top-level function
```scala
import scala.scalajs.js.annotation.*

@JSExportTopLevel("greet")
def greet(name: String): String = s"Hello, $name!"
```

Then in JS: `greet("World")` → `"Hello, World!"`

### Export an object's methods
```scala
@JSExportAll
object MyAPI {
  def add(a: Int, b: Int): Int = a + b
  def multiply(a: Int, b: Int): Int = a * b
}
```

## 6. Type Conversions

| Scala type | JS type | Notes |
|-----------|---------|-------|
| `String` | `string` | Direct |
| `Int`, `Double` | `number` | Direct |
| `Boolean` | `boolean` | Direct |
| `Unit` | `undefined` | Direct |
| `js.Array[A]` | `Array` | Mutable JS array |
| `Array[A]` (Scala) | _wrapped_ | Use `js.Array` for interop |
| `js.Dictionary[A]` | `object` (string keys) | Like `Map[String, A]` |
| `js.Promise[A]` | `Promise` | Convert via `.toFuture` / `Future.toJSPromise` |
| `js.UndefOr[A]` | `A \| undefined` | Use `.toOption` / `.getOrElse` |

### Converting between Scala Future and JS Promise
```scala
import scala.scalajs.js
import js.JSConverters.*

// Future → Promise
val promise: js.Promise[Int] = Future.successful(42).toJSPromise

// Promise → Future
val future: Future[Int] = someJSPromise.toFuture
```

### Converting Scala collections to JS
```scala
import js.JSConverters.*

val scalaSeq: Seq[Int] = Seq(1, 2, 3)
val jsArr: js.Array[Int] = scalaSeq.toJSArray

val scalaMap: Map[String, Int] = Map("a" -> 1)
val jsDict: js.Dictionary[Int] = scalaMap.toJSDictionary
```

## 7. `@JSName` — Rename JS member

```scala
@js.native
trait JQuery extends js.Object {
  @JSName("val")
  def value(): String = js.native   // avoids Scala keyword conflict
  
  @JSName("val")
  def value(v: String): this.type = js.native
}
```

## 8. Bracket access (`obj[key]`)

```scala
@js.native
trait AssocArray extends js.Object {
  @JSBracketAccess
  def apply(key: String): js.Any = js.native
  
  @JSBracketAccess
  def update(key: String, value: js.Any): Unit = js.native
}
```

## 9. Union types in façades

```scala
@js.native
trait Input extends js.Object {
  // JS method that accepts string | number
  def setValue(v: String | Double): Unit = js.native
}
```

## Common Pitfalls

- **`isInstanceOf[T]` is forbidden** on `@js.native` traits — use JS `typeof` checks via `js.typeOf(x)` or `js.Dynamic`.
- **`= js.native`** is required as the body of all members in `@js.native` types — it's a compiler-required placeholder.
- **Do not call `@js.native` constructors with `new` on traits** — only on classes with `@JSGlobal` or `@JSImport`.
- **Structural equality** (`==`) between JS objects uses reference equality, not structural comparison.
