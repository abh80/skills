# Scala anti-patterns catalog

Detailed reference for common Scala anti-patterns. Read this before producing findings in pattern-matching, error-handling, type-design, or concurrency categories.

## Table of contents

1. [`isInstanceOf` / `asInstanceOf` cascades](#isinstanceof)
2. [`Option.get` and friends](#option-get)
3. [`Option`/`null` mixing](#option-null)
4. [Throwing exceptions from non-throwing signatures](#exceptions)
5. [`catch _: Throwable`](#catch-throwable)
6. [Pattern matching on erased generics](#erased-generics)
7. [Non-exhaustive matches on sealed types](#non-exhaustive)
8. [`if`/`else` chain over a sealed hierarchy](#if-else-sealed)
9. [`Future[Either]` chains without `EitherT`](#future-either)
10. [`Await.result` outside test code](#await)
11. [`ExecutionContext.Implicits.global` in libraries](#global-ec)
12. [Blocking on the global `ExecutionContext`](#blocking-global)
13. [Returning `Seq[A]` from public APIs](#seq-public)
14. [Weakly-typed APIs (primitive obsession)](#primitive-obsession)
15. [Boolean flag parameters](#bool-flags)
16. [Mutable state in shared traits](#trait-mutable)
17. [Deep linearization chains](#linearization)
18. [Public method without explicit return type](#explicit-return)
19. [`case class` extending `case class`](#case-class-extends)
20. [Cake pattern for DI](#cake-pattern)

---

## 1. `isInstanceOf` / `asInstanceOf` cascades

```scala
// Bad
if (x.isInstanceOf[Person]) x.asInstanceOf[Person].name else "unknown"

// Good
x match
  case p: Person => p.name
  case _         => "unknown"
```

**Why:** Pattern matching is checked statically (the compiler errors if the scrutinee type can't possibly match the pattern), short-circuits cleanly, and binds the cast result to a name automatically. `asInstanceOf` is unchecked at compile time and produces noisier code.

**Citation:** "It's more idiomatic in Scala to use a pattern match rather then the isInstanceOf / asInstanceOf combo." — Scala mailing list / community consensus; for typed patterns see `docs.scala-lang.org/tour/pattern-matching.html`.

**Performance:** `equivalent`. Both forms compile to a single `instanceof` check + checkcast.

**Exception:** `isInstanceOf` is acceptable for narrow interop with reflective code or as a one-liner type guard where pattern-matching syntax would be heavier than the test.

---

## 2. `Option.get` and friends

```scala
// Bad
val name: String = userOpt.get

// Good — pick the one that fits the situation
userOpt.getOrElse("anonymous")
userOpt.fold("anonymous")(_.name)
userOpt match
  case Some(u) => u.name
  case None    => "anonymous"
userOpt.map(_.name).getOrElse("anonymous")
```

**Why:** `.get` throws `NoSuchElementException` on `None`. It defeats the type-system value of `Option` — if you knew the value was present, you would have refined the type already. Treat every `.get` as a code smell unless preceded by structural proof (e.g., directly after `if (opt.isDefined)` — but even then, pattern matching is clearer).

The same applies to `Try.get`, `Either.toOption.get`, `head` on a possibly-empty `List`, and similar partial functions.

**Citation:** "Avoid `get` on Option: Use map, flatMap, or fold to handle Option safely." — community best-practice guides; for the type itself see `docs.scala-lang.org/scala3/book/functional-error-handling.html`.

**Performance:** `equivalent`.

---

## 3. `Option` / `null` mixing

```scala
// Bad — Some(null) is a footgun, and Java APIs return null
val name: Option[String] = Some(javaApi.getName())  // could be Some(null)

// Good
val name: Option[String] = Option(javaApi.getName())  // None if null, Some(s) otherwise
```

**Why:** `Option(x)` is the safe wrapper that maps `null` to `None`. `Some(x)` always wraps, including `null`, producing `Some(null)` — a value that pattern-matches as `Some` but explodes when accessed.

**Performance:** `equivalent`.

---

## 4. Throwing exceptions from non-throwing signatures

```scala
// Bad — signature lies about failure modes
def parseUser(s: String): User =
  if (s.isEmpty) throw new IllegalArgumentException("empty")
  else User(s)

// Good — error is part of the type
def parseUser(s: String): Either[ParseError, User] =
  if (s.isEmpty) Left(ParseError("empty"))
  else Right(User(s))
```

**Why:** Functional Scala treats errors as values. Encoding failure in the return type makes it impossible for callers to forget. Scala does not have checked exceptions, so `throw` is invisible in the signature.

Use `Option[A]` when the only failure is "absent". Use `Either[E, A]` (or a custom ADT) for known business errors. Use `Try[A]` when bridging code that throws.

**Citation:** "In Scala and functional programming languages it is common to make the errors that can occur explicit in the functions signature." — community guidance.

**Performance:** `improved` for the common path — exceptions on the JVM are expensive when actually thrown (stack-trace capture). `equivalent` for the success path.

---

## 5. `catch _: Throwable` swallowing fatal errors

```scala
// Bad
try riskyOp()
catch { case _: Throwable => fallback }

// Good
import scala.util.control.NonFatal
try riskyOp()
catch { case NonFatal(e) => fallback }
```

**Why:** `Throwable` includes `OutOfMemoryError`, `StackOverflowError`, `InterruptedException`, `LinkageError`, and other JVM-fatal conditions you should never silently swallow. `NonFatal` lets these through while catching ordinary exceptions.

**Citation:** See `scala.util.control.NonFatal` docs. The `Future` execution context also uses this distinction internally.

**Performance:** `equivalent`.

---

## 6. Pattern matching on erased generics

```scala
// Bad — types are erased; this matches any List
xs match
  case _: List[Int]    => "ints"
  case _: List[String] => "strings"  // unreachable
```

**Why:** JVM type erasure means `List[Int]` and `List[String]` are the same class at runtime. Compiler should warn (`unchecked` warning) — flag the warning, suggest using a tagged type, sealed hierarchy, or `TypeTag`/`ClassTag` for runtime type info.

**Performance:** N/A — this is a correctness bug.

---

## 7. Non-exhaustive matches on sealed types

```scala
// Bad — silently broken when a new case is added
sealed trait Shape
case class Circle(r: Double) extends Shape
case class Square(s: Double) extends Shape
case class Triangle(a: Double, b: Double, c: Double) extends Shape

def area(s: Shape): Double = s match
  case Circle(r) => math.Pi * r * r
  case Square(s) => s * s
  // missing Triangle — compiler warns; do not silence with `case _ => 0.0`
```

**Why:** Suppressing exhaustiveness via `case _ =>` defeats the safety the sealed hierarchy gives you. When a new case is added, the compiler should tell you about every match that needs updating. Adding a fall-through `case _` makes the warning stop and the bug start.

**Citation:** "If we try to extend a sealed trait outside of the parent class file, the compiler will throw an exception" + exhaustiveness checks documented at `docs.scala-lang.org/tour/pattern-matching.html`.

---

## 8. `if` / `else` chain over a sealed hierarchy

When dispatching on the subtype of a sealed trait, prefer `match`. Long `if (x.isInstanceOf[A])` chains lose exhaustiveness checking and are noisier.

---

## 9. `Future[Either[E, A]]` chains without a transformer

```scala
// Painful — every step has two layers to unwrap
def step1: Future[Either[E, A]] = ???
def step2(a: A): Future[Either[E, B]] = ???

val result: Future[Either[E, B]] =
  step1.flatMap {
    case Right(a) => step2(a)
    case Left(e)  => Future.successful(Left(e))
  }
```

**Better — `EitherT` from cats:**
```scala
import cats.data.EitherT
val result: EitherT[Future, E, B] =
  for
    a <- EitherT(step1)
    b <- EitherT(step2(a))
  yield b
```

**Performance:** `EitherT` adds a thin wrapper allocation per step but the readability win is large. If you can't add a cats dependency, write a small `flatMapEither` helper rather than repeating the `case Left/Right` pattern.

---

## 10. `Await.result` outside test code

`Await.result(f, timeout)` blocks the calling thread. In production code paths this defeats the point of `Future`, can deadlock pools, and adds latency. Acceptable in tests, top-level `main`, or shutdown hooks. Flag elsewhere.

---

## 11. `ExecutionContext.Implicits.global` in libraries

A library should accept `using ec: ExecutionContext` (or `implicit ec` in Scala 2) rather than import the global EC itself. The choice of executor is an application-level concern.

---

## 12. Blocking calls on the global `ExecutionContext`

```scala
Future {
  Thread.sleep(5000)  // blocks a thread in the global pool
  jdbcCall()          // blocks
  Source.fromFile(p)  // blocks
}
```

**Why:** The global EC is sized for CPU-bound work (one thread per core by default). Blocking calls starve it and tank throughput.

**Fix:** Wrap blocking work in `scala.concurrent.blocking { … }` (signals to the pool to compensate) or — better — use a dedicated `ExecutionContext` backed by a cached thread pool for blocking I/O.

**Performance:** `improved` once fixed — under load, the pool starvation is dramatic.

---

## 13. Returning `Seq[A]` from public APIs

`Seq[A]` is a supertype of `List`, `Vector`, `LazyList`, `ArraySeq`, etc. — each with very different cost profiles. Returning `Seq` forces callers to either commit to the worst-case behavior or call `.toList`/`.toVector` defensively (which copies).

Pick the concrete type at API boundaries: `List` for stack-like prepend-heavy work, `Vector` for general indexed access, `ArraySeq` for read-heavy primitive data.

**Citation:** "An overly generic data structure can increase ambiguity in the use of the API. Therefore, it is best to create a more specific type." — community guidance.

---

## 14. Weakly-typed APIs (primitive obsession)

```scala
// Bad — easy to mix up
def transfer(from: Long, to: Long, amount: Long, currency: String): Unit

// Good — unmistakable at the call site
def transfer(from: AccountId, to: AccountId, amount: Money, currency: Currency): Unit
```

Use opaque types for zero-cost wrappers (Scala 3) or value classes (Scala 2). Even a small `case class` is better than raw `String`/`Long`/`UUID` when the domain meaning matters.

---

## 15. Boolean flag parameters

```scala
// Bad — call sites read as `process(data, true)` with no clue what true means
def process(data: Bytes, isAdmin: Boolean): Result

// Good
enum Role:
  case Admin, User
def process(data: Bytes, role: Role): Result

// Or split into two methods
def processAsAdmin(data: Bytes): Result
def processAsUser(data: Bytes): Result
```

---

## 16. Mutable state in shared traits

A trait with `var` or mutable collection state, mixed into many classes, becomes a hidden global. Each instance of each class that mixes the trait carries its own copy, but the pattern often signals confused ownership.

---

## 17. Deep trait-stacking and linearization chains

When 5+ traits are stacked with overlapping method overrides and `super.method()` calls, the resolution order (linearization) becomes hard to reason about and fragile under reordering. Flag deep stacks; suggest composition or smaller, orthogonal traits.

---

## 18. Public method without explicit return type

```scala
// Bad — return type is whatever the body happens to infer
def processAll(items: List[Item]) = items.map(process).filter(_.isValid)

// Good — explicit, stable public contract
def processAll(items: List[Item]): List[ProcessedItem] = ...
```

Inference is fine for `private`/local definitions. For public/protected members, the return type is part of the contract. Inference couples the public API to implementation details — refactoring the body can silently change the public type.

**Performance:** `equivalent`.

---

## 19. `case class` extending `case class`

Deprecated and discouraged. `case class` extending `case class` produces broken `equals`/`hashCode`/`unapply` semantics. Use composition or a sealed hierarchy with `final case class`es that share a parent trait/abstract class.

---

## 20. Cake pattern for DI

Self-types and trait-stacking-as-DI ("the cake pattern") was popular in Scala 2 but is widely considered over-engineered for most projects. Constructor injection or `using` clauses are lighter, easier to test, and easier to read. Flag cake-pattern usage; do not mandate removal — note the tradeoff.
