# Cross-Building JVM + JS Reference

Cross-building lets you share Scala code between a JVM backend and a Scala.js frontend.

## The `%%%` vs `%%` distinction

| Operator | Meaning |
|----------|---------|
| `"org.foo" %% "lib" % "1.0"` | JVM-only artifact |
| `"org.foo" %%% "lib" % "1.0"` | Cross-platform: picks JS artifact in ScalaJS project, JVM artifact in JVM project |

Only use `%%%` for libraries that publish cross-platform artifacts.

## sbt-crossproject setup

```scala
// project/plugins.sbt
addSbtPlugin("org.scala-js" % "sbt-scalajs" % "1.20.2")
addSbtPlugin("org.portable-scala" % "sbt-crossproject" % "1.3.2")
```

```scala
// build.sbt
lazy val shared = crossProject(JSPlatform, JVMPlatform)
  .crossType(CrossType.Pure)   // no platform-specific source dirs
  .in(file("shared"))
  .settings(
    scalaVersion := "3.3.3",
    libraryDependencies ++= Seq(
      "com.lihaoyi" %%% "upickle" % "3.3.1",  // JSON, cross-platform
    ),
  )
  .jsSettings(
    // JS-specific settings
  )
  .jvmSettings(
    // JVM-specific settings
  )

lazy val sharedJS  = shared.js
lazy val sharedJVM = shared.jvm

lazy val frontend = project.in(file("frontend"))
  .enablePlugins(ScalaJSPlugin)
  .dependsOn(sharedJS)
  .settings(
    scalaVersion := "3.3.3",
    scalaJSUseMainModuleInitializer := true,
    scalaJSLinkerConfig ~= _.withModuleKind(ModuleKind.ESModule),
    libraryDependencies += "com.raquo" %%% "laminar" % "17.0.0",
  )

lazy val backend = project.in(file("backend"))
  .dependsOn(sharedJVM)
  .settings(
    scalaVersion := "3.3.3",
    libraryDependencies += "com.softwaremill.sttp.client4" %% "core" % "4.0.0-M16",
  )
```

## CrossType options

| Type | Source folders |
|------|---------------|
| `CrossType.Pure` | `shared/src/` only — no platform-specific code |
| `CrossType.Full` | `shared/src/` + `shared/js/src/` + `shared/jvm/src/` |
| `CrossType.Dummy` | Just for aggregation; use platform-specific source fully |

## Platform-specific code in shared module

```scala
// In shared/src/main/scala/myapp/Platform.scala
// Use @JSExport and platform checks where needed

// Provide platform-specific implementations via `expect`/`actual` (Scala 3 Native)
// or by placing files in js/src/ vs jvm/src/
```

With `CrossType.Full`, place platform-specific files in:
- `shared/js/src/main/scala/myapp/Platform.scala`
- `shared/jvm/src/main/scala/myapp/Platform.scala`

## Using `scalajs-stubs` for JVM compilation

When shared code uses Scala.js annotations (`@JSExportTopLevel`, etc.), the JVM project won't have access to them. Add the stub library as a `provided` dependency:

```scala
.jvmSettings(
  libraryDependencies += "org.scala-js" %% "scalajs-stubs" % "1.1.0" % "provided"
)
```

## Mill cross-build pattern

```scala
import mill._, scalalib._, scalajslib._

// Trait for shared settings
trait CommonModule extends ScalaModule {
  def scalaVersion = "3.3.3"
  def ivyDeps = Agg(ivy"com.lihaoyi::upickle::3.3.1")
}

object shared extends Module {
  // Put shared Scala files in shared/src/
  object jvm extends CommonModule {
    def sources = T.sources(millSourcePath / os.up / "src")
  }
  object js extends CommonModule with ScalaJSModule {
    def scalaJSVersion = "1.20.2"
    def sources = T.sources(millSourcePath / os.up / "src")
  }
}

object frontend extends ScalaJSModule with CommonModule {
  def scalaJSVersion = "1.20.2"
  def moduleDeps = Seq(shared.js)
  def moduleKind = T { ModuleKind.ESModule }
  def ivyDeps = super.ivyDeps() ++ Agg(ivy"com.raquo::laminar::17.0.0")
}

object backend extends CommonModule {
  def moduleDeps = Seq(shared.jvm)
}
```

## Popular cross-platform libraries

| Library | Purpose |
|---------|---------|
| `upickle` | JSON serialisation |
| `circe` | JSON (functional) |
| `cats` | Functional abstractions |
| `cats-effect` | Async IO (supports JS) |
| `zio` | Effect system (supports JS) |
| `tapir` | Type-safe HTTP endpoint definitions |
| `http4s` | HTTP server/client (JVM) / client (JS) |
| `scalatags` | HTML generation |
| `monocle` | Optics / lenses |

## Serialisation pattern (full-stack)

```scala
// shared/src/main/scala/myapp/Api.scala
import upickle.default.*

case class User(id: Int, name: String) derives ReadWriter
case class CreateUserRequest(name: String) derives ReadWriter

// backend: receive JSON, deserialise to case class
// frontend: serialise case class to JSON, send via fetch
```

This pattern avoids duplicating data model definitions and keeps API contract type-safe end-to-end.
