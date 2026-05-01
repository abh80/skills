# ScalablyTyped Reference

ScalablyTyped automatically converts TypeScript `.d.ts` type definitions (from DefinitelyTyped and npm libraries) into typed Scala.js façades. This saves you from writing façade types manually for popular JS libraries.

## How it works

1. You add npm libraries to `package.json` (with their `@types/...` if needed)
2. ScalablyTyped reads `node_modules` and generates Scala.js façade libraries
3. The generated libraries are added to your Scala.js compile classpath

## sbt setup

```scala
// project/plugins.sbt
addSbtPlugin("org.scalablytyped.converter" % "sbt-converter" % "1.0.0-beta44")

// build.sbt
import org.scalablytyped.converter.plugin.STImport.*

lazy val myApp = project.in(file("."))
  .enablePlugins(ScalaJSPlugin, ScalablyTypedConverterPlugin)
  .settings(
    scalaVersion := "3.3.3",
    scalaJSUseMainModuleInitializer := true,
    scalaJSLinkerConfig ~= _.withModuleKind(ModuleKind.ESModule),
    // Specify which npm libraries to convert:
    stImport := Seq("d3", "chart.js", "lodash"),
    // Optional: use scala-js-dom types instead of translated std:
    stUseScalaJsDom := true,
    // Optional: for React-based projects:
    stFlavour := Flavour.Japgolly,  // scalajs-react
    // or:
    stFlavour := Flavour.Slinky,    // Slinky
  )
```

## Mill setup

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

// ScalablyTyped scans package.json + node_modules
object `scalablytyped-module` extends Base with ScalablyTyped

object app extends Base {
  def moduleDeps = Seq(`scalablytyped-module`)
  def ivyDeps = Agg(ivy"org.scala-js::scalajs-dom::2.8.1")
  def moduleKind = T { ModuleKind.ESModule }
}
```

## Usage after generation

After setup, generated types live under the `typings` package:

```scala
import typings.d3.mod.*
import typings.chartJs.mod.*

// Use generated façades as if they were hand-written:
val chart = new Chart(canvas, chartOptions)
```

## `package.json` — where you declare npm dependencies

```json
{
  "devDependencies": {
    "vite": "^5.0.0",
    "@scala-js/vite-plugin-scalajs": "^1.0.0"
  },
  "dependencies": {
    "d3": "^7.0.0",
    "chart.js": "^4.0.0",
    "lodash": "^4.17.21"
  },
  "devDependencies": {
    "@types/d3": "^7.0.0",
    "@types/lodash": "^4.14.0"
  }
}
```

Run `npm install` before first build.

## When NOT to use ScalablyTyped

- The library doesn't have TypeScript typings (`@types/...`) — ScalablyTyped has nothing to convert
- You need very precise control over the façade API — write it manually (see interop.md)
- Generated types are too complex/verbose — write a minimal hand-rolled façade for the subset you need
- Quick prototype — use `js.Dynamic` instead

## First-build warning

ScalablyTyped can take **several minutes** on the first build while it converts all TypeScript definitions. Subsequent builds are cached and fast. This is expected — don't kill the build.

## Troubleshooting

| Issue | Fix |
|-------|-----|
| `node_modules` missing | Run `npm install` first |
| Types not found after adding npm dep | Run `sbt reload` (sbt) or re-import (Mill/Metals) |
| Generated types conflict | Use `stIgnore := Set("some-lib")` to exclude a library |
| React Flavour mismatch | Set `stFlavour` to match your React wrapper library |
| Initial build very slow | Expected — just wait; subsequent builds are cached |
