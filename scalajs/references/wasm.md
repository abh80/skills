# Emitting WebAssembly from Scala.js

As of Scala.js 1.16+, the linker can emit WebAssembly (Wasm) instead of JavaScript. This is an experimental but functional feature for performance-critical code paths.

## When to use Wasm output

- Computationally intensive Scala code (algorithms, data processing)
- When JS output size is not a concern (Wasm modules can be larger)
- Experimental / cutting-edge applications

Note: Wasm output is not supported in all JavaScript environments. Requires browser support for the Wasm GC proposal (Chrome 119+, Firefox 120+, Safari 18+).

## sbt configuration

```scala
import org.scalajs.linker.interface.{ModuleKind, OutputPatterns}
import org.scalajs.linker.interface.unstable.ReportToLinkerOutputAdapter

lazy val app = project.in(file("."))
  .enablePlugins(ScalaJSPlugin)
  .settings(
    scalaVersion := "3.3.3",
    scalaJSUseMainModuleInitializer := true,

    // Enable Wasm output
    scalaJSLinkerConfig ~= {
      _.withExperimentalUseWebAssembly(true)
        .withModuleKind(ModuleKind.ESModule)
    },
  )
```

## Mill configuration

```scala
object app extends ScalaJSModule {
  def scalaVersion   = "3.3.3"
  def scalaJSVersion = "1.20.2"

  def scalaJSLinkerConfig = T {
    super.scalaJSLinkerConfig().withExperimentalUseWebAssembly(true)
  }

  def moduleKind = T { ModuleKind.ESModule }
}
```

## Wasm output structure

The linker produces:
- `main.wasm` — the compiled WebAssembly binary
- `main.js` — JavaScript "loader" that instantiates the Wasm module

Both files must be served together.

## Limitations (as of 2025)

- `js.Dynamic` is not supported in Wasm output — use typed façades
- Some reflection-based APIs are limited
- Output is larger than equivalent JS for small programs (overhead from Wasm module format)
- Not all browsers support Wasm GC yet (check caniuse.com)
- `fastLinkJS` in Wasm mode is slower than JS mode

## Checking Wasm support at runtime

```scala
// In JS bootstrap code (main.js or similar):
// WebAssembly.validate() can check if Wasm is supported
```

For production apps, consider feature-detecting and falling back to JS output.

## Further reading

- [Official Scala.js Wasm docs](https://www.scala-js.org/doc/project/webassembly.html)
- Wasm GC specification: github.com/WebAssembly/gc
