# Scala 3 idiom catalog

Detailed reference for Scala 3 idioms and Scala 2 → 3 migration. Read this before producing findings in the "idiom adoption" or "migration cleanup" categories.

## Table of contents

1. [`enum` replacing sealed-trait ADTs and `Enumeration`](#enum)
2. [`given`/`using` replacing implicits](#given-using)
3. [`extension` replacing implicit classes](#extension)
4. [`opaque type` replacing value classes](#opaque-type)
5. [`derives` replacing manual typeclass derivation](#derives)
6. [Top-level definitions replacing package objects](#top-level)
7. [`export` clauses replacing forwarder methods](#export)
8. [Union and intersection types](#union-intersection)
9. [`@main` replacing `extends App`](#main)
10. [`inline` and `transparent inline`](#inline)
11. [`Conversion` replacing implicit conversions](#conversion)
12. [New control syntax and optional braces](#control-syntax)
13. [Function values, eta-expansion, placeholder syntax](#function-values)
14. [Migration: Scala 2 features dropped or deprecated in Scala 3](#dropped)

---

## 1. `enum` replacing sealed-trait ADTs and `Enumeration`

**Before (Scala 2):**
```scala
sealed abstract class Color(val name: String)
object Color {
  case object Red extends Color("red")
  case object Green extends Color("green")
  case object Blue extends Color("blue")
}
```

**After (Scala 3):**
```scala
enum Color(val name: String):
  case Red   extends Color("red")
  case Green extends Color("green")
  case Blue  extends Color("blue")
```

**Why it matters:**
- `scala.Enumeration` does NOT give exhaustiveness checks at compile time. Two `Enumeration` objects share the same erased `Enumeration#Value` class, leading to type-erasure surprises.
- `enum` gives exhaustive pattern matching, parameterization, and per-case data.
- `enum` values get a free `.values` accessor and `.ordinal`.

**Performance:** `equivalent` to the sealed-trait + case-object encoding. Better than `Enumeration` because of erasure-friendly types.

**Citation:** "Scala 3 introduces a new construct for defining enumerations: enum [...] cases comprise the values of the enumeration." — `docs.scala-lang.org/scala3/reference/enums/enums.html`

**Caveats:** Cross-build with Scala 2 requires keeping the sealed-trait form. `enum` cases are not `case object`s syntactically; tools that reflect on `case object` may need updating.

---

## 2. `given` / `using` replacing implicits

**Before:**
```scala
implicit val intOrd: Ordering[Int] = Ordering.Int
def sort[A](xs: List[A])(implicit ord: Ordering[A]): List[A] = xs.sorted
```

**After:**
```scala
given intOrd: Ordering[Int] = Ordering.Int
def sort[A](xs: List[A])(using ord: Ordering[A]): List[A] = xs.sorted
```

**Why it matters:**
- `given` separates the contextual-instance concept from method definitions and `val`s, which `implicit` had conflated.
- `using` makes "this argument is implicit" explicit at the call site.
- Scala 3 requires explicit types on `given` definitions — flag any `implicit val` without an annotated type as a portability concern.

**Performance:** `equivalent` — `given` compiles to the same JVM constructs as `implicit val`/`implicit def` with the same caching semantics.

**Citation:** "Given instances can be mapped to combinations of implicit objects, classes and implicit methods." — `docs.scala-lang.org/scala3/reference/contextual/relationship-implicits.html`

**Caveats:** Anonymous givens are common in Scala 3 (`given Ordering[Int] = Ordering.Int`); when porting, deciding whether to keep the name affects source-level imports.

---

## 3. `extension` replacing implicit classes and `AnyVal` syntactic wrappers

**Before:**
```scala
implicit class StringOps(val s: String) extends AnyVal {
  def shout: String = s.toUpperCase + "!"
}
```

**After:**
```scala
extension (s: String)
  def shout: String = s.toUpperCase + "!"
```

**Why it matters:**
- `extension` is a first-class language feature, not implicit-class syntactic sugar.
- No allocation: extension methods are compiled as static methods. The `AnyVal` trick was needed in Scala 2 to avoid wrapper allocation; in Scala 3 it's automatic.
- Generic and contextual extensions are cleaner than the implicit-class equivalent.

**Performance:** `equivalent` to `implicit class … extends AnyVal` for the common case. `improved` over a non-AnyVal `implicit class` because it avoids the wrapper allocation that value classes sometimes still hit (e.g., when stored in a generic container).

**Citation:** "Extension methods have no direct counterpart in Scala 2, but they can be simulated with implicit classes." — `docs.scala-lang.org/scala3/reference/contextual/relationship-implicits.html`

**Caveats:** Extension methods imported via `import obj._` work; importing a `given` to bring its extensions into scope requires `import obj.given`.

---

## 4. `opaque type` replacing value classes (`extends AnyVal`)

**Before:**
```scala
final case class UserId(value: Long) extends AnyVal
```

**After:**
```scala
object Ids:
  opaque type UserId = Long
  object UserId:
    def apply(v: Long): UserId = v
    extension (id: UserId) def value: Long = id
```

**Why it matters:**
- Value classes `extends AnyVal` still allocate a wrapper in surprisingly many cases: when pattern-matched, when stored in a `List`/`Array`, when the type is used as a generic parameter.
- Opaque types fully erase to the underlying type at runtime — `UserId` is literally a `Long` in bytecode, with compile-time type distinction only inside the defining scope.

**Performance:** `improved` over value classes for pattern-match and collection-stored cases. `equivalent` for simple method-arg/return cases the JVM already inlined. `needs-benchmark` for the one corner case: opaque types over a primitive used through a generic parameter still box the same way bare primitives do under generics — there's no magic that beats JVM erasure.

**Citation:** "Opaque type aliases provide type abstraction without any overhead. In Scala 2, a similar result could be achieved with value classes." — `docs.scala-lang.org/scala3/book/types-opaque-types.html`

**Caveats:** Opaque types do NOT inherit the underlying type's API automatically — methods must be added via `extension`. Cross-build with Scala 2 requires keeping the value class.

---

## 5. `derives` replacing manual typeclass instances

**Before:**
```scala
case class Point(x: Int, y: Int)
object Point {
  implicit val pointShow: Show[Point] = ...
  implicit val pointEq:   Eq[Point]   = ...
}
```

**After:**
```scala
case class Point(x: Int, y: Int) derives Show, Eq
```

**Why it matters:**
- `derives` invokes a `derived` method on the typeclass companion at compile time, using `scala.deriving.Mirror` to introspect the type structure.
- Eliminates manual instance boilerplate, reduces drift between case-class fields and instance definitions.
- Works for both product types (case classes) and sum types (sealed hierarchies, enums).

**Performance:** `equivalent` to hand-written instances IF the typeclass's `derived` method is well-implemented (uses `inline` to avoid runtime reflection). `needs-benchmark` if the typeclass uses `Mirror` at runtime via reflection-y patterns — Scala 3's `Mirror` is intentionally low-level and the *cost* of derivation depends on the library.

**Citation:** "scala.deriving.Mirror type class instances provide information at the type level about the components and labelling of the type." — `docs.scala-lang.org/scala3/reference/contextual/derivation.html`

**Caveats:** The typeclass library must provide a `derived` method. Cats, Circe, and similar major libraries support this in their Scala 3 versions.

---

## 6. Top-level definitions replacing `package object`

**Before:**
```scala
package object utils {
  def double(i: Int) = i * 2
  type Result[A] = Either[String, A]
}
```

**After:**
```scala
package utils
def double(i: Int) = i * 2
type Result[A] = Either[String, A]
```

**Why it matters:**
- `package object` was a workaround for Scala 2's restriction that only classes/objects/traits could live at the package level. Scala 3 removes this restriction.
- Faster compilation and fewer surprises around initialization order.

**Performance:** `equivalent`, possibly `improved` for compilation time.

**Citation:** "If you're familiar with Scala 2, this approach replaces package objects." — `docs.scala-lang.org/scala3/book/taste-toplevel-definitions.html`

**Caveats:** Cross-build needs the `package object` form for Scala 2.

---

## 7. `export` clauses replacing forwarder methods

**Before:**
```scala
class Service(repo: Repo) {
  def findById(id: Long) = repo.findById(id)
  def save(x: X)         = repo.save(x)
}
```

**After:**
```scala
class Service(repo: Repo):
  export repo.{findById, save}
```

**Why it matters:** Removes hand-written forwarders that drift from the underlying API.

**Performance:** `equivalent`.

**Citation:** See `docs.scala-lang.org/scala3/reference/other-new-features/export.html` (referenced from "New in Scala 3").

---

## 8. Union (`A | B`) and intersection (`A & B`) types

**Before (union encoding):**
```scala
sealed trait IdLike
final case class Username(s: String) extends IdLike
final case class Password(h: Hash)   extends IdLike
def help(id: IdLike) = ...
```

**After:**
```scala
def help(id: Username | Password) = id match
  case Username(name) => ...
  case Password(hash) => ...
```

**Why it matters:** Avoids forcing types into a marker-trait hierarchy just to express alternation.

**Performance:** `equivalent`. Union types erase to their join (least common supertype).

**Citation:** "A union type A | B includes all values of both types." — `docs.scala-lang.org/scala3/reference/new-types/union-types.html`

**Caveats:** If a function's signature was previously the marker trait, switching to a union type is a source-incompatible change for callers.

---

## 9. `@main` replacing `extends App`

**Before:**
```scala
object MyApp extends App {
  println(args.mkString(" "))
}
```

**After:**
```scala
@main def myApp(args: String*): Unit =
  println(args.mkString(" "))
```

**Why it matters:** `App`'s `DelayedInit` had subtle initialization-order quirks (uninitialized vals at top of object body in some cases). `@main` is a plain method.

**Citation:** See `docs.scala-lang.org/scala3/reference/changed-features/main-functions.html`.

---

## 10. `inline` and `transparent inline`

`inline def` guarantees the method body is inlined at the call site. `transparent inline` additionally lets the inlined return type be more specific than declared.

**Use sparingly:**
- Small, hot methods. Inlining a large method bloats bytecode without runtime benefit (the JIT would have inlined small methods anyway).
- `inline if` and `inline match` for compile-time branching.

**Performance:** `improved` for very hot, small methods; `equivalent` for methods the JIT already inlines; potential `needs-benchmark` for code-bloat cases.

**Citation:** "Inlining is a common compile-time metaprogramming technique, typically used to achieve performance optimizations." — `docs.scala-lang.org/scala3/guides/macros/inline.html`

**Caveats:** `transparent inline` can dramatically increase compile times if abused with deep type-inference chains.

---

## 11. `Conversion` replacing `implicit def` conversions

**Before:**
```scala
implicit def intToSec(i: Int): Second = Second(i)
```

**After:**
```scala
given Conversion[Int, Second] = Second(_)
// And callers must: import scala.language.implicitConversions
```

**Why it matters:** `implicit def`s for conversions were too easy to define accidentally. `Conversion` is a typeclass — defining one is intentional.

**Citation:** "Implicit conversions are done by creating instances of Conversion." — Scala 3 contextual abstractions docs.

---

## 12. New control syntax and optional braces

```scala
if x < 0 then -x else x
while x >= 0 do x = f(x)
for x <- xs if x > 0 yield x * x
```

**Why it matters:** Reduces visual noise. Optional braces (significant indentation) is enabled by default in Scala 3.

**Citation:** "Scala 3 has a new 'quiet' syntax for control expressions that does not rely on enclosing the condition in parentheses." — `docs.scala-lang.org/scala3/reference/other-new-features/control-syntax.html`

**Caveats:** Suggest only when readability genuinely improves — do not push purely on style. Tooling (formatters, syntax highlighters) is still catching up in some setups.

---

## 13. Function values, eta-expansion, placeholder syntax

### Placeholder syntax (already idiomatic in Scala 2, still correct in Scala 3)

```scala
xs.map(_ + 1)            // good
xs.map(x => x + 1)       // verbose, prefer placeholder
xs.reduce(_ + _)         // good — two distinct placeholders
```

Each `_` is a distinct argument. `_ << _` is `(a, b) => a << b`. **Do NOT** suggest replacing this — it is correct.

### Eta-expansion (auto in Scala 3)

```scala
xs.map(println)          // good — eta-expanded
xs.map(x => println(x))  // verbose
xs.foreach(processor.handle)  // good
```

Scala 3 eta-expands automatically almost everywhere. The Scala 2 `m _` syntax is no longer needed.

**Do NOT suggest placeholder when:**
- The same argument is used twice (`x => f(x, x)` cannot be expressed with `_`).
- A type ascription is needed for inference.
- Overload resolution would change.
- SAM conversion vs `FunctionN` matters for the target API.

**Citation:** "Scala 3 introduces Automatic Eta-Expansion which will deprecate the method to value syntax m _." — `docs.scala-lang.org/scala3/guides/migration/incompat-dropped-features.html`

### `def` vs `val` vs `lazy val` for callable values

```scala
def f: A => B  = x => g(x)   // allocates a fresh FunctionN every reference
val f: A => B  = x => g(x)   // allocates once at definition
lazy val f: A => B = x => g(x)  // allocates once on first use, but every read goes through a sync init guard
```

Flag `def f: A => B = …` patterns when the function is referenced many times — `val` may be measurably better. Conversely, flag `lazy val f: A => B = …` for cheap-to-construct functions accessed in tight loops — the lock overhead is real.

---

## 14. Migration: Scala 2 features dropped or deprecated in Scala 3

| Scala 2 form | Scala 3 fix | Citation anchor |
|---|---|---|
| `def f() { … }` (procedure syntax) | `def f(): Unit = { … }` | migration/incompat-dropped-features |
| `do { … } while (…)` | `while { …; cond } do ()` or refactored | migration/incompat-dropped-features |
| `m _` for nullary methods | call as `m()` (no eta-expansion to `() => …` allowed) | migration/incompat-dropped-features |
| `obj.foo` for `def foo()` (auto-application) | `obj.foo()` | migration/incompat-dropped-features |
| `{ x: Int => … }` (untyped lambda param) | `{ (x: Int) => … }` | migration/incompat-syntactic |
| `forSome` (existential types) | dropped — use a wildcard or refactor | migration/incompat-dropped-features |
| `'sym` (Symbol literals) | deprecated — use `String` or domain type | migration/incompat-dropped-features |
| `implicit val ev: A => B` used as a conversion | use `Conversion` typeclass | migration/incompat-contextual-abstractions |
| `package object foo { … }` | top-level definitions | (idiom) |
| `extends App` | `@main def …` | (idiom) |

**Citation root:** `docs.scala-lang.org/scala3/guides/migration/incompat-dropped-features.html`

**Note on `m _`:** For methods with parameters, `m _` is "no longer needed and will be deprecated in the future." For nullary methods, eta-expansion via `m _` was deprecated in 2.13.3 and is removed in Scala 3 — it now requires writing `() => m()` explicitly.
**Note on `m _`:** For methods with parameters, `m _` is "no longer needed and will be deprecated in the future." For nullary methods, eta-expansion via `m _` was deprecated in 2.13.3 and is removed in Scala 3 — it now requires writing `() => m()` explicitly.
