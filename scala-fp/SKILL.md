---
name: scala-fp
description: >
  Expert guidance on functional programming in Scala — Cats typeclasses, Cats Effect IO, ZIO 2, fs2 streams, parallel/concurrent programming, tagless final, and the full Typelevel/ZIO ecosystem.
  Use this skill whenever the user asks about: Cats, Cats Effect, ZIO, fs2, Monix, IO monad, fibers, concurrency, parallel effects, Ref, Semaphore, Queue, Deferred, ZLayer, tagless final, Functor/Monad/Applicative/Traverse, monad transformers, FP architecture patterns, or wants to write purely functional Scala code.
  Also trigger for: "how do I run things in parallel", "concurrent Scala without threads", "effect systems", "pure functional programming", "Resource/bracket", "error handling in FP", "dependency injection in FP", "streaming with fs2 or ZIO Streams".
  Always consult this skill — even for seemingly simple FP questions — because effect systems, typeclass hierarchies, and concurrency models have non-obvious subtleties that this skill captures precisely.
---

# Scala Functional Programming & Effects Skill

This skill covers the **Typelevel ecosystem (Cats + Cats Effect + fs2)** and **ZIO 2** for building purely functional, concurrent, and resource-safe Scala applications.

## Quick Reference Table

| Topic | Section |
|-------|---------|
| Cats typeclasses (Functor, Monad, etc.) | [references/cats-typeclasses.md](references/cats-typeclasses.md) |
| Cats Effect IO + concurrency | [references/cats-effect.md](references/cats-effect.md) |
| ZIO 2 core + ZLayer | [references/zio.md](references/zio.md) |
| Parallel programming patterns | [references/parallelism.md](references/parallelism.md) |
| fs2 streams | [references/fs2.md](references/fs2.md) |
| Tagless Final architecture | [references/tagless-final.md](references/tagless-final.md) |
| Error handling in FP | [references/error-handling.md](references/error-handling.md) |
| Library versions & ecosystem | [references/ecosystem.md](references/ecosystem.md) |

---

## Mental Model: What Is a Functional Effect?

A **functional effect** is a **description** of a computation — not the computation itself. Rather than running side effects immediately (throwing exceptions, printing, launching threads), you build an immutable value that *describes* what should happen. The effect is only executed at the "end of the world" (your main method).

```
// Imperative: runs immediately, hard to compose
val result = doSomething()

// Functional: a description, composable, lazy
val effect: IO[Result] = IO(doSomething())
// Nothing has run yet — effect is just a value
```

This enables:
- **Referential transparency** — `effect` can be used anywhere without changing behaviour
- **Composability** — effects compose with `map`, `flatMap`, `parMapN`, etc.
- **Resource safety** — resources are guaranteed to be released even on errors/cancellation
- **Structured concurrency** — child fibers are properly scoped and cancelled

---

## Choosing Between Cats Effect and ZIO

| Aspect | Cats Effect 3 | ZIO 2 |
|--------|--------------|-------|
| Core type | `IO[A]` | `ZIO[R, E, A]` |
| Error channel | `IO[A]` (Throwable via `MonadError`) | Typed errors: `ZIO[R, E, A]` |
| DI / Env | Manual / tagless final | `ZLayer` (built-in, typed) |
| Ecosystem | Typelevel (http4s, doobie, skunk, fs2) | ZIO ecosystem (zio-http, quill, ZStream) |
| Style | Typeclass-based, composable abstractions | Concrete types, discoverable combinators |
| Learning curve | Steeper (typeclasses) | More accessible (concrete API) |
| Cross-platform | JVM + Scala.js + Scala Native | JVM + Scala.js + Scala Native |

**When to choose CE**: existing Typelevel library stack, tagless final preference, interop with http4s/doobie/fs2.

**When to choose ZIO**: prefer typed errors, built-in DI, simpler API surface, ZIO ecosystem libraries.

---

## The Typeclass Hierarchy (Cats)

```
Semigroup → Monoid
Functor → Apply → Applicative → Monad
                ↘ FlatMap    ↗
Functor → Contravariant
        → Traverse (+ Foldable)
```

The hierarchy from Cats Effect:
```
Functor → Monad → MonadThrow → Sync → Async → Temporal → Concurrent → GenConcurrent
```

Read [references/cats-typeclasses.md](references/cats-typeclasses.md) for full patterns.

---

## Key Concurrency Primitives (Both Ecosystems)

| Primitive | Cats Effect | ZIO |
|-----------|------------|-----|
| Mutable atomic reference | `Ref[F, A]` | `Ref[A]` |
| Single-use async signal | `Deferred[F, A]` | `Promise[E, A]` |
| Bounded concurrency | `Semaphore[F]` | `Semaphore` |
| Async message queue | `Queue[F, A]` | `Queue[A]` |
| Pub/sub broadcast | — | `Hub[A]` |
| Fiber (lightweight thread) | `Fiber[F, E, A]` | `Fiber[E, A]` |
| Resource with cleanup | `Resource[F, A]` | `ZIO` + `Scope` |

---

## Parallel Patterns — Quick Hits

### Run N effects in parallel, collect all results
```scala
// Cats Effect
(ioA, ioB, ioC).parMapN((a, b, c) => combine(a, b, c))
list.parTraverse(item => processItem(item))

// ZIO
ZIO.collectAllPar(effects)
ZIO.foreachPar(list)(processItem)
```

### Race — take the first to complete, cancel the loser
```scala
// Cats Effect
IO.race(ioA, ioB)                    // Either[A, B]

// ZIO
ZIO.race(effectA, effectB)           // A | B
```

### Timeout
```scala
// Cats Effect
io.timeout(5.seconds)

// ZIO
effect.timeout(5.seconds)
```

Read [references/parallelism.md](references/parallelism.md) for complete patterns and examples.

---

## Resource Safety

Always use `Resource` (CE) or `ZIO.scoped`/`ZLayer` (ZIO) for anything that needs cleanup:

```scala
// Cats Effect
val connection: Resource[IO, Connection] =
  Resource.make(openConnection)(_.close())

connection.use { conn =>
  conn.query("SELECT 1")
}

// ZIO
val connection: ZIO[Scope, Throwable, Connection] =
  ZIO.acquireRelease(openConnection)(_.close().orDie)

ZIO.scoped {
  connection.flatMap(conn => conn.query("SELECT 1"))
}
```

---

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| Using `unsafeRunSync` inside effects | Never block inside an effect; use `IO.blocking` or async lifting |
| `Future` interop with `Await.result` | Use `IO.fromFuture` / `ZIO.fromFuture` |
| Forgetting `parTraverse` vs `traverse` | `traverse` = sequential; `parTraverse` = parallel |
| Calling `.start` without joining or cancelling | Use `Supervisor` or structured concurrency (`parMapN`, etc.) |
| Thread-blocking I/O on the compute pool | Wrap with `IO.blocking { ... }` / `ZIO.blocking { ... }` |
| ZIO 1.x patterns in ZIO 2 | No `Has`, no ZEnv — read [references/zio.md](references/zio.md) |
