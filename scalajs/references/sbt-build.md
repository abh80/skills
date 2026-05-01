# sbt Build Reference for Scala.js

## File: `project/build.properties`
```
sbt.version=1.10.0
```

## File: `project/plugins.sbt`
```scala
addSbtPlugin("org.scala-js" % "sbt-scalajs" % "1.20.2")

// Optional: for cross JVM/JS builds
addSbtPlugin("org.portable-scala" % "sbt-crossproject" % "1.3.2")

// Optional: for bundling npm deps without external bundler
addSbtPlugin("ch.epfl.scala" % "sbt-scalajs-bundler" % "0.21.1")
```

## File: `build.sbt` — Minimal application

```scala
import org.scalajs.linker.interface.ModuleSplitStyle

lazy val myApp = project.in(file("."))
  .enablePlugins(ScalaJSPlugin)
  .settings(
    name := "my-app",
    scalaVersion := "3.3.3",

    // Required for apps with a @main / main method
    scalaJSUseMainModuleInitializer := true,

    // ESModule output for Vite integration
    scalaJSLinkerConfig ~= {
      _.withModuleKind(ModuleKind.ESModule)
        .withModuleSplitStyle(
          ModuleSplitStyle.SmallModulesFor(List("myapp")))
    },

    libraryDependencies += "org.scala-js" %%% "scalajs-dom" % "2.8.1",
  )
```

## File: `build.sbt` — With Laminar + MUnit

```scala
import org.scalajs.linker.interface.ModuleSplitStyle

lazy val myApp = project.in(file("."))
  .enablePlugins(ScalaJSPlugin)
  .settings(
    name := "my-app",
    scalaVersion := "3.3.3",
    scalaJSUseMainModuleInitializer := true,
    scalaJSLinkerConfig ~= {
      _.withModuleKind(ModuleKind.ESModule)
        .withModuleSplitStyle(ModuleSplitStyle.SmallModulesFor(List("myapp")))
    },
    libraryDependencies ++= Seq(
      "org.scala-js"  %%% "scalajs-dom" % "2.8.1",
      "com.raquo"     %%% "laminar"     % "17.0.0",
      "org.scalameta" %%% "munit"       % "1.0.0" % Test,
    ),
  )
```

## Vite config — `vite.config.js`

```js
import { defineConfig } from "vite";
import scalaJSPlugin from "@scala-js/vite-plugin-scalajs";

export default defineConfig({
  plugins: [scalaJSPlugin()],
});
```

Install the Vite plugin once:
```
npm install -D @scala-js/vite-plugin-scalajs
```

## `main.js` — entry point after Vite scaffolding

```js
import './style.css'
import 'scalajs:main.js'   // resolved by vite-plugin-scalajs
```

## Key sbt tasks

| Task | Purpose |
|------|---------|
| `sbt ~fastLinkJS` | Incremental compile + link (dev mode, watch) |
| `sbt fullLinkJS` | Optimised production link |
| `sbt test` | Run Scala.js tests via Node.js |
| `sbt fastLinkJS::webpack` | (scalajs-bundler) bundle with webpack |
| `sbt reload` | Reload build after editing build.sbt |

## Multi-module sbt build

```scala
// shared code module
lazy val shared = crossProject(JSPlatform, JVMPlatform)
  .crossType(CrossType.Pure)
  .in(file("shared"))
  .settings(scalaVersion := "3.3.3")

lazy val sharedJS  = shared.js
lazy val sharedJVM = shared.jvm

// frontend Scala.js module
lazy val frontend = project.in(file("frontend"))
  .enablePlugins(ScalaJSPlugin)
  .dependsOn(sharedJS)
  .settings(
    scalaVersion := "3.3.3",
    scalaJSUseMainModuleInitializer := true,
    scalaJSLinkerConfig ~= _.withModuleKind(ModuleKind.ESModule),
    libraryDependencies += "com.raquo" %%% "laminar" % "17.0.0",
  )

// backend JVM module
lazy val backend = project.in(file("backend"))
  .dependsOn(sharedJVM)
  .settings(
    scalaVersion := "3.3.3",
    libraryDependencies += "com.softwaremill.sttp.client4" %% "core" % "4.0.0",
  )
```

## Important sbt settings explained

| Setting | Meaning |
|---------|---------|
| `scalaJSUseMainModuleInitializer := true` | Call `main()` automatically at startup |
| `scalaJSLinkerConfig ~= ...` | Modify linker configuration inline |
| `ModuleKind.ESModule` | Emit ES6 `import`/`export` modules |
| `ModuleSplitStyle.SmallModulesFor(pkgs)` | Split only app package into small files; stdlib stays big |
| `%%%` | sbt cross-version operator for Scala.js libraries |

## Scala.js test configuration (sbt)

```scala
// To test with Node.js (default):
// Just `sbt test` — sbt-scalajs picks up Node.js automatically

// To test with jsdom (browser-like environment):
jsEnv := new org.scalajs.jsenv.jsdomnodejs.JSDOMNodeJSEnv()
// requires: addSbtPlugin("org.scala-js" % "sbt-jsdependencies" % "...")
//           and npm install jsdom

// For running tests in a real browser:
jsEnv := new org.scalajs.jsenv.selenium.SeleniumJSEnv(new ChromeOptions())
```
