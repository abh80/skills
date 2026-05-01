# Mill Build Reference for Scala.js

## Minimal `build.mill` (Mill 0.12 / Scala.js 1.x)

```scala
//| mill-version: 0.12.5

package build

import mill._, scalalib._, scalajslib._
import scalajslib.api._

object app extends ScalaJSModule {
  def scalaVersion   = "3.3.3"
  def scalaJSVersion = "1.20.2"

  def ivyDeps = Agg(
    ivy"org.scala-js::scalajs-dom::2.8.1",
  )

  def moduleKind = T { ModuleKind.ESModule }

  object test extends ScalaJSTests with TestModule.Utest {
    def ivyDeps = super.ivyDeps() ++ Agg(
      ivy"com.lihaoyi::utest::0.8.4",
    )
  }
}
```

> Note: In Mill, Scala.js library dependencies use `::` (not `%%%`); Mill infers the platform automatically via `ScalaJSModule`.

## Common Mill tasks

| Task | Purpose |
|------|---------|
| `mill app.fastLinkJS` | Dev link (fast) |
| `mill app.fullLinkJS` | Production link (optimised) |
| `mill -w app.fastLinkJS` | Watch mode (re-link on source change) |
| `mill app.test` | Run tests via Node.js |
| `mill app.run` | Run the main method on Node.js |
| `mill app.compile` | Compile only (no link) |

## With Laminar + MUnit

```scala
object app extends ScalaJSModule {
  def scalaVersion   = "3.3.3"
  def scalaJSVersion = "1.20.2"

  def ivyDeps = Agg(
    ivy"org.scala-js::scalajs-dom::2.8.1",
    ivy"com.raquo::laminar::17.0.0",
  )

  def moduleKind = T { ModuleKind.ESModule }

  def moduleSplitStyle = T { ModuleSplitStyle.SmallModulesFor(List("myapp")) }

  object test extends ScalaJSTests with TestModule.MUnit {
    def ivyDeps = super.ivyDeps() ++ Agg(
      ivy"org.scalameta::munit::1.0.0",
    )
  }
}
```

## Vite integration with Mill

Mill doesn't have an official Vite plugin, but the workflow is:

1. Set `moduleKind = ModuleKind.ESModule` and `moduleSplitStyle`.
2. In `vite.config.js`, point to Mill's output directory:
   ```js
   import { defineConfig } from "vite";

   export default defineConfig({
     // Mill outputs to: out/app/fastLinkJS.dest/
     resolve: {
       alias: {
         "scalajs:": new URL("./out/app/fastLinkJS.dest/", import.meta.url).pathname,
       },
     },
   });
   ```
3. Run two terminals:
   ```
   mill -w app.fastLinkJS   # terminal 1
   npm run dev              # terminal 2
   ```

Alternatively, use the community plugin `mill-vite` or follow the `vite-scala-js-mill` starter pattern.

## Cross JVM + JS with Mill

```scala
import mill._, scalalib._, scalajslib._

trait AppModule extends ScalaModule {
  def scalaVersion = "3.3.3"
  def ivyDeps = Agg(ivy"com.lihaoyi::upickle::3.3.1")
}

// JVM target
object jvm extends AppModule

// JS target
object js extends AppModule with ScalaJSModule {
  def scalaJSVersion = "1.20.2"
  def moduleKind = T { ModuleKind.ESModule }
}

// Shared sources via a trait (both inherit from AppModule)
// Put shared Scala files in a `src/` directory visible to both.
```

A cleaner approach using a `shared` cross-module:

```scala
import mill._, scalalib._, scalajslib._

trait Shared extends ScalaModule {
  def scalaVersion = "3.3.3"
  // shared ivyDeps
}

object shared extends Module {
  object jvm extends Shared
  object js  extends Shared with ScalaJSModule {
    def scalaJSVersion = "1.20.2"
  }
}

object frontend extends ScalaJSModule {
  def scalaVersion   = "3.3.3"
  def scalaJSVersion = "1.20.2"
  def moduleDeps = Seq(shared.js)
  def moduleKind = T { ModuleKind.ESModule }
  def ivyDeps = Agg(ivy"com.raquo::laminar::17.0.0")
}

object backend extends ScalaModule {
  def scalaVersion = "3.3.3"
  def moduleDeps = Seq(shared.jvm)
}
```

## ScalablyTyped with Mill

```scala
//| mill-version: 1.0.5
//| mvnDeps:
//|   - com.github.lolgab::mill-scalablytyped::0.4.0

import mill._, scalalib._, scalajslib._
import com.github.lolgab.mill.scalablytyped._

trait Base extends ScalaJSModule {
  def scalaVersion   = "3.3.3"
  def scalaJSVersion = "1.20.1"
}

// ScalablyTyped reads package.json + node_modules and generates façades
object `scalablytyped-module` extends Base with ScalablyTyped

object app extends Base {
  def moduleDeps = Seq(`scalablytyped-module`)
  def ivyDeps = Agg(ivy"org.scala-js::scalajs-dom::2.8.1")
}
```
Run `npm install` before the first build so `node_modules` is populated.

## Module split style options

```scala
import scalajslib.api.ModuleSplitStyle

def moduleSplitStyle = T { ModuleSplitStyle.SmallModulesFor(List("myapp")) }
// or
def moduleSplitStyle = T { ModuleSplitStyle.SmallestModules }
// or (default)
def moduleSplitStyle = T { ModuleSplitStyle.FewestModules }
```
