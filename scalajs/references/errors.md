# Common Scala.js Errors & Fixes

## Build Errors

### `error: not found: type ScalaJSPlugin`
**Cause**: Plugin not declared.
**Fix**: Add to `project/plugins.sbt`:
```scala
addSbtPlugin("org.scala-js" % "sbt-scalajs" % "1.20.2")
```

### `sbt.librarymanagement.ResolveException: unresolved dependency`
**Cause**: Using `%%` instead of `%%%` for a cross-platform library.
**Fix**: Change `%%` to `%%%`:
```scala
// Wrong:
libraryDependencies += "com.raquo" %% "laminar" % "17.0.0"
// Correct:
libraryDependencies += "com.raquo" %%% "laminar" % "17.0.0"
```

### `ModuleKind not found` / `ModuleSplitStyle not found`
**Cause**: Missing import in `build.sbt`.
**Fix**: Add at top of `build.sbt`:
```scala
import org.scalajs.linker.interface.ModuleSplitStyle
```

---

## Linker Errors

### `Uncaught ReferenceError: exports is not defined`
**Cause**: Using `ModuleKind.CommonJSModule` but loading as a script (not via CommonJS require).
**Fix**: Switch to `ModuleKind.ESModule` if using Vite, or ensure the output is loaded via a CommonJS bundler.

### `Linking error: ... is not defined`
**Cause**: Using a JVM-only API (e.g., `java.io.File`, `java.sql.*`) in Scala.js code.
**Fix**: Replace with browser-compatible API or a cross-platform library. Common replacements:
- `java.io.File` → `org.scalajs.dom.Blob` or Node.js `fs` façade
- `Thread.sleep` → `js.timers.setTimeout`
- `java.time.*` → limited; use `js.Date` or a cross-platform date library

### `Linker: Cannot call method on a JavaScript type`
**Cause**: Calling `isInstanceOf` on a `@js.native` trait.
**Fix**: Remove the `isInstanceOf` check. Use `js.typeOf(x) == "object"` or a duck-type check instead.

### `Duplicate module ID`
**Cause**: Two modules emit to the same output filename.
**Fix**: Ensure different `name` settings for each sbt sub-project.

---

## Runtime Errors

### `TypeError: Cannot read properties of undefined (reading 'xxx')`
**Cause**: Accessing a property on a JS object that is `undefined`/`null`.
**Fix**: Use `js.UndefOr[A]` in your façade and call `.toOption` before accessing. Or add a null check.

### `ClassCastException: [object Object] is not an instance of ...`
**Cause**: Casting a JS object to a Scala class with `asInstanceOf`.
**Fix**: Don't cast JS objects to Scala classes — only to JS traits/classes. Or use a proper façade type.

### `Cannot call method on undefined` in Laminar
**Cause**: DOM element not yet in the document when accessed.
**Fix**: Wrap DOM access in `renderOnDomContentLoaded` or inside a Laminar element lifecycle callback.

---

## Vite + Scala.js Errors

### `[plugin:vite-plugin-scalajs] Could not find fastLinkJS output`
**Cause**: sbt `~fastLinkJS` is not running, or the output directory doesn't match.
**Fix**:
1. Make sure `sbt ~fastLinkJS` is running in a separate terminal
2. Ensure `scalaJSLinkerConfig` uses `ModuleKind.ESModule`
3. Check that `vite.config.js` has `import scalaJSPlugin from "@scala-js/vite-plugin-scalajs"` and the plugin is listed

### Browser shows old version after Scala source change
**Cause**: HMR didn't pick up the change; possibly sbt is not watching.
**Fix**: Confirm `sbt ~fastLinkJS` (with `~`) is running. Restart both sbt and `npm run dev` if needed.

### `import 'scalajs:main.js'` not found
**Cause**: Either the Vite plugin isn't installed/configured, or `scalaJSUseMainModuleInitializer := true` is missing.
**Fix**:
1. `npm install -D @scala-js/vite-plugin-scalajs`
2. Add `scalaJSUseMainModuleInitializer := true` to `build.sbt`
3. Add the plugin to `vite.config.js`

---

## ScalablyTyped Errors

### `object typings is not a member of package ...`
**Cause**: ScalablyTyped hasn't generated the façades yet, or the library isn't in `stImport`.
**Fix**:
1. Add the npm package name to `stImport`
2. Run `sbt reload`
3. Wait for ScalablyTyped to finish generating (can take several minutes first time)

### ScalablyTyped takes forever on first build
**Expected behavior**: ScalablyTyped converts potentially thousands of TypeScript definitions. Run `sbt fastLinkJS` once manually before opening in Metals/IntelliJ to avoid IDE timeouts.

---

## Debugging Tips

- **Source maps**: sbt-scalajs emits source maps automatically in `fastLinkJS` mode. In Chrome DevTools, you'll see original `.scala` file names in the Sources panel.
- **`println` debugging**: `println(...)` in Scala.js prints to the browser console via `console.log`.
- **`js.Dynamic.global.console.log(x)` debugging**: for inspecting raw JS objects.
- **Check linker output**: The `.js` files in `target/scala-3.x/xxx-fastopt/` contain the linked output. Inspect them if something seems wrong.
