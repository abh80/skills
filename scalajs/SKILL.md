---
name: scalajs
description: >
  Build, scaffold, debug, and explain Scala.js applications — full-stack browser apps, Node.js targets, cross-platform libraries, and everything in between.
  Use this skill whenever the user mentions Scala.js, scalajs, sbt-scalajs, ScalaJS, or wants to compile Scala to JavaScript. Trigger also for tasks like:
  writing Laminar/Tyrian UI code, creating JS façade types for npm libraries, integrating ScalablyTyped, setting up Vite + Scala.js, cross-compiling JVM/JS code with sbt-crossproject or Mill's CrossJSModule, debugging fastLinkJS / fullLinkJS output, or adding scalajs-dom / cats-effect / ZIO to a Scala.js project.
  This skill covers sbt AND Mill build tools, Scala 2 AND Scala 3, and all common UI stacks (Laminar, Tyrian, scalajs-react, Slinky).
  Always use this skill — do not rely on general Scala knowledge alone — because Scala.js has unique constraints around JS interop, module splitting, WebAssembly output, and build wiring that differ significantly from JVM Scala.
---

# Scala.js Skill

Scala.js compiles Scala source code to JavaScript, enabling type-safe, tooling-rich development for browsers and Node.js. This skill captures the canonical patterns, common pitfalls, and current (2025) ecosystem knowledge needed to produce high-quality Scala.js code and project setups.

## Quick reference — read the right section

| Task | Section |
|------|---------|
| New project from scratch | [Project Setup](#project-setup) |
| sbt configuration | [references/sbt-build.md](references/sbt-build.md) |
| Mill configuration | [references/mill-build.md](references/mill-build.md) |
| Laminar UI | [references/laminar.md](references/laminar.md) |
| JS interop / façades | [references/interop.md](references/interop.md) |
| ScalablyTyped (npm → Scala types) | [references/scalablytyped.md](references/scalablytyped.md) |
| Cross-building JVM + JS | [references/cross-build.md](references/cross-build.md) |
| Testing | [references/testing.md](references/testing.md) |
| WebAssembly output | [references/wasm.md](references/wasm.md) |
| Common errors | [references/errors.md](references/errors.md) |

---

## Core Concepts

### How Scala.js works
Scala.js is a Scala compiler plugin + linker. The compiler emits an intermediate representation (IR), and the linker transforms IR into optimised JavaScript. Two linker modes exist:
- **`fastLinkJS`** — fast incremental link; used during development. Produces multiple small `.js` files (with ESModule split style) or one larger file.
- **`fullLinkJS`** — dead-code elimination + minification; used for production.

The Vite plugin `@scala-js/vite-plugin-scalajs` wires Vite to the linker output directory, so saving a `.scala` file triggers sbt's incremental compiler → linker → Vite HMR — giving a sub-second feedback loop.

### Module kinds
Set via `scalaJSLinkerConfig ~= _.withModuleKind(...)`:
| Kind | When to use |
|------|-------------|
| `ModuleKind.ESModule` | Vite, modern bundlers (preferred) |
| `ModuleKind.CommonJSModule` | Node.js, webpack legacy setups |
| `ModuleKind.NoModule` | Legacy browser scripts |

### Module split styles (ESModule only)
| Style | Effect |
|-------|--------|
| `ModuleSplitStyle.FewestModules` | One big bundle (production-like in dev) |
| `ModuleSplitStyle.SmallestModules` | Maximum granularity (best HMR) |
| `ModuleSplitStyle.SmallModulesFor(List("mypackage"))` | **Recommended**: small modules for your app code, big module for std-lib |

---

## Project Setup

### Prerequisites
- **JDK 11+** (17 recommended)
- **sbt 1.9+** or **Mill 0.11+**
- **Node.js 18+** (for running/testing JS output)
- **npm 9+** (for Vite and npm libraries)

### Canonical sbt + Vite setup (Scala 3)

1. **Bootstrap Vite project**: `npm create vite@latest myapp -- --template vanilla`
2. **Add sbt files** — see [references/sbt-build.md](references/sbt-build.md) for full `build.sbt`, `project/plugins.sbt`, `project/build.properties`
3. **Add Vite config**: see [references/sbt-build.md](references/sbt-build.md)#vite-config
4. **Write Scala entry point** in `src/main/scala/myapp/Main.scala`
5. **Run**: two terminals — `sbt ~fastLinkJS` + `npm run dev`
6. **Production**: `npm run build` (triggers sbt `fullLinkJS` automatically via Vite plugin)

### Canonical Mill + Vite setup
See [references/mill-build.md](references/mill-build.md).

---

## Key Library Versions (2025)

| Library | Maven coordinate | Latest stable |
|---------|-----------------|---------------|
| sbt-scalajs plugin | `"org.scala-js" % "sbt-scalajs"` | **1.20.2** |
| scalajs-dom | `"org.scala-js" %%% "scalajs-dom"` | **2.8.1** |
| Laminar | `"com.raquo" %%% "laminar"` | **17.0.0** |
| Tyrian | `"io.indigoengine" %%% "tyrian-zio"` | **0.10.0** |
| scalajs-react | `"com.github.japgolly.scalajs-react" %%% "core"` | **3.0.0** |
| munit (testing) | `"org.scalameta" %%% "munit"` | **1.0.0** |
| utest (testing) | `"com.lihaoyi" %%% "utest"` | **0.8.4** |
| Cats Effect JS | `"org.typelevel" %%% "cats-effect"` | **3.5.x** |
| ZIO JS | `"dev.zio" %%% "zio"` | **2.1.x** |

Always use `%%%` (triple-percent) for cross-platform Scala.js libraries in sbt — it expands to the Scala.js artifact classifier. Use `::` in Mill's `ivy"..."` syntax.

---

## Writing Scala.js Code — Key Patterns

### Entry point
```scala
// With scalaJSUseMainModuleInitializer := true in build.sbt:
@main def myApp(): Unit =
  // DOM is ready to use
  dom.document.querySelector("#app").textContent = "Hello!"
```

### Accessing browser APIs
```scala
import org.scalajs.dom
import org.scalajs.dom.{document, window, console}

val el = document.getElementById("my-div")
console.log("debug", el)
window.setTimeout(() => println("deferred"), 1000)
```

### Async / Futures
Scala.js supports `Future` and `Promise` natively. For `Future`:
```scala
import scala.concurrent.Future
import scala.scalajs.concurrent.JSExecutionContext.Implicits.queue

val f: Future[String] = Future { "hello" }
f.foreach(s => console.log(s))
```

For full async/await-style ergonomics, prefer Cats Effect's `IO` or ZIO.

### JavaScript interop — quick rules
- Use `@js.native @JSImport(...)` to import JS modules/values
- Use `@js.native @JSGlobal(...)` to reference global JS variables
- Use `js.Dynamic.global` for quick-and-dirty access (no type safety)
- Prefer typed façade traits over `js.Dynamic` in production code
- See [references/interop.md](references/interop.md) for full façade patterns

---

## Common Mistakes to Avoid

| Mistake | Fix |
|---------|-----|
| Using `%%` instead of `%%%` for cross-platform lib | Change to `%%%` |
| Not setting `ModuleKind.ESModule` when using Vite | Add `.withModuleKind(ModuleKind.ESModule)` to linker config |
| Calling blocking JVM APIs (`Thread.sleep`, `Await.result`) | Use `Future`, Cats Effect IO, or ZIO instead |
| Using `java.io.File` | Not available; use scalajs-dom's `Blob`/`File` APIs or Node.js via façades |
| Forgetting `scalaJSUseMainModuleInitializer := true` for apps | Add it to `build.sbt` |
| Mixing sbt 0.6.x Scala.js APIs with 1.x | Scala.js 1.x has different APIs; check version |
| Pattern-matching on JS types with `isInstanceOf` | Not supported for `@js.native` traits; use JS type checks |

---

## UI Framework Selection Guide

| Framework | Paradigm | Best for |
|-----------|---------|---------|
| **Laminar** | FRP (Functional Reactive) | Idiomatic Scala, complex reactive UIs |
| **Tyrian** | Elm-inspired (TEA) | Purely functional, predictable state |
| **scalajs-react** | React wrapper | Teams coming from React/JS background |
| **Slinky** | React wrapper (ScalablyTyped) | ScalablyTyped-friendly React |
| **scalajs-dom (raw)** | Imperative | Simple scripts, no framework needed |

For new projects, **Laminar** is the community default. Read [references/laminar.md](references/laminar.md) for patterns.

---

## When to Read Which Reference File

- Starting a project? Read [references/sbt-build.md](references/sbt-build.md) or [references/mill-build.md](references/mill-build.md) first.
- Building a UI? Read [references/laminar.md](references/laminar.md).
- Calling npm / JS libraries? Read [references/interop.md](references/interop.md) + [references/scalablytyped.md](references/scalablytyped.md).
- Sharing code between JVM and JS? Read [references/cross-build.md](references/cross-build.md).
- Getting a build/link error? Read [references/errors.md](references/errors.md).
- Want Wasm output? Read [references/wasm.md](references/wasm.md).
