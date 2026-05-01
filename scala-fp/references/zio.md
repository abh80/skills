# ZIO 2 Reference

## Setup

```scala
libraryDependencies += "dev.zio" %% "zio"      % "2.1.15"
libraryDependencies += "dev.zio" %% "zio-test" % "2.1.15" % Test
```

## The ZIO type: `ZIO[R, E, A]`

```
R = environment (dependencies required)
E = typed error (Nothing = cannot fail)
A = success value
```

### Common type aliases

| Alias | Expanded | Meaning |
|-------|---------|---------|
| `UIO[A]` | `ZIO[Any, Nothing, A]` | Cannot fail, no env |
| `URIO[R, A]` | `ZIO[R, Nothing, A]` | Cannot fail, needs R |
| `Task[A]` | `ZIO[Any, Throwable, A]` | May throw, no env |
| `RIO[R, A]` | `ZIO[R, Throwable, A]` | May throw, needs R |
| `IO[E, A]` | `ZIO[Any, E, A]` | Typed error, no env |

---

## ZIOAppDefault — entry point

```scala
import zio.*

object Main extends ZIOAppDefault:
  def run: ZIO[Any, Throwable, Unit] =
    ZIO.succeed("Hello!").flatMap(ZIO.debug(_))

// Provide layers at the top level:
object Main extends ZIOAppDefault:
  def run =
    myProgram.provide(
      DatabaseLive.layer,
      CacheLive.layer,
      ZLayer.Debug.tree   // prints layer construction tree
    )
```

---

## Building ZIO values

```scala
ZIO.succeed(42)                    // UIO[Int]
ZIO.attempt(sideEffect())          // Task[A] — wraps Throwable
ZIO.fail("error")                  // IO[String, Nothing]
ZIO.die(new Exception("defect"))   // Fatal — unrecoverable
ZIO.unit                           // UIO[Unit]
ZIO.never                          // ZIO that never completes
ZIO.sleep(1.second)                // URIO[Any, Unit]
ZIO.debug("msg")                   // prints to console
ZIO.log("msg")                     // structured logging

// From Scala std:
ZIO.fromOption(Option(42))         // IO[Option[Nothing], Int]
ZIO.fromEither(Right(42))          // IO[Nothing, Int] (or error type)
ZIO.fromFuture(implicit ec => future) // Task[A]
ZIO.fromTry(scala.util.Try(42))    // Task[A]

// Blocking I/O:
ZIO.blocking(ZIO.attempt(readFile())) // runs on blocking thread pool
ZIO.attemptBlockingIO(readFile())    // shorthand
```

---

## Composing ZIO effects

```scala
for {
  user   <- findUser(id)
  orders <- fetchOrders(user.id)
  _      <- sendNotification(user, orders)
} yield orders

// Map / flatMap:
ZIO.succeed(1).map(_ + 1)
ZIO.succeed(1).flatMap(n => ZIO.succeed(n + 1))

// Sequencing without binding:
ZIO.succeed(1) *> ZIO.succeed(2)     // discard left
ZIO.succeed(1) <* ZIO.debug("side")  // discard right
ZIO.succeed(1) zip ZIO.succeed("a")  // sequential: ZIO[(Int, String)]
ZIO.succeed(1) zipPar ZIO.succeed("a") // parallel: ZIO[(Int, String)]
```

---

## Error handling

```scala
// Typed errors — E in ZIO[R, E, A]:
ZIO.fail("not found")
  .mapError(msg => AppError(msg))           // transform error type
  .catchAll(e => ZIO.succeed(defaultValue)) // recover from all errors
  .catchSome { case AppError("not found") => ZIO.succeed(0) }

// Fold — handle both paths:
effect.fold(
  error   => s"failed: $error",
  success => s"got: $success"
)

effect.foldZIO(
  error   => ZIO.debug(s"error: $error") *> ZIO.fail(error),
  success => ZIO.succeed(success)
)

// Either — materialise typed error:
effect.either          // ZIO[R, Nothing, Either[E, A]]

// Cause — inspect full failure cause:
effect.cause           // ZIO[R, Nothing, Cause[E]]

// Defect (Throwable) handling:
ZIO.attempt(riskyJava())
  .orDie                 // convert Throwable to defect (crash fiber)
  .orDieWith(e => new RuntimeException(e.toString))
  .refineToOrDie[IOException]  // keep only IOException, die on others

// Retry:
effect.retry(Schedule.exponential(1.second))
effect.retry(Schedule.recurs(3))
effect.retry(Schedule.fixed(500.millis) && Schedule.recurs(5))
```

---

## ZLayer — Dependency Injection

A `ZLayer[-RIn, +E, +ROut]` is a recipe to build services.

### Service pattern (ZIO 2 — "Service Pattern 2.0")

```scala
// 1. Define the service interface:
trait Database:
  def findUser(id: Int): Task[User]
  def saveUser(user: User): Task[Unit]

// 2. Implement it:
case class DatabaseLive(pool: ConnectionPool) extends Database:
  def findUser(id: Int) = ZIO.attempt(pool.query(s"SELECT * FROM users WHERE id=$id").head)
  def saveUser(user: User) = ZIO.attempt(pool.execute(s"INSERT INTO users VALUES (${user.id})"))

object DatabaseLive:
  val layer: ZLayer[ConnectionPool, Throwable, Database] =
    ZLayer {
      for pool <- ZIO.service[ConnectionPool]
      yield DatabaseLive(pool)
    }

// Or with acquire/release (resource-safe):
object DatabaseLive:
  val layer: ZLayer[Any, Throwable, Database] =
    ZLayer.scoped {
      for
        pool <- ZIO.acquireRelease(ZIO.attempt(ConnectionPool.create()))(p => ZIO.succeed(p.close()))
      yield DatabaseLive(pool)
    }

// 3. Use in program:
def myProgram: ZIO[Database, Throwable, Unit] =
  for
    db   <- ZIO.service[Database]
    user <- db.findUser(42)
    _    <- ZIO.debug(user)
  yield ()

// 4. Provide layers at the end of the world:
myProgram.provide(
  DatabaseLive.layer,
  ConnectionPoolLive.layer,
)
```

### Composing layers

```scala
// Sequential (output of A feeds input of B):
layerA >>> layerB

// Parallel (both layers provided, outputs merged):
layerA ++ layerB

// Auto-wire (macro resolves dependency order):
ZLayer.make[Database & Cache](
  DatabaseLive.layer,
  CacheLive.layer,
  ConnectionPoolLive.layer,
)

// Debug dependency tree:
ZLayer.Debug.tree   // adds compile-time tree dump
ZLayer.Debug.mermaid // adds Mermaid diagram output
```

---

## Fibers

```scala
// Fork a fiber:
for {
  fiber  <- longTask.fork          // ZIO.Fiber[E, A]
  _      <- otherWork
  result <- fiber.join             // wait for result
} yield result

// Fork daemon (outlives parent scope):
longTask.forkDaemon

// Interrupt a fiber:
fiber.interrupt          // sends interrupt signal, waits for cleanup
fiber.interruptFork      // send interrupt, don't wait

// Structured — all forked fibers auto-joined/interrupted:
ZIO.scoped {
  for {
    fiber1 <- task1.fork
    fiber2 <- task2.fork
    r1     <- fiber1.join
    r2     <- fiber2.join
  } yield (r1, r2)
}
```

---

## Concurrency Primitives

### Ref — atomic mutable reference

```scala
for {
  counter <- Ref.make(0)
  _       <- counter.update(_ + 1)
  _       <- counter.update(_ + 1)
  n       <- counter.get
} yield n   // 2

// Atomic modify (returns both new state and a result):
counter.modify(n => (n + 1, n * 2))  // (newState, returnValue)
```

### Promise — single-completion signal (Deferred in CE)

```scala
for {
  promise <- Promise.make[String, Int]
  fiber   <- (ZIO.sleep(1.second) *> promise.succeed(42)).fork
  _       <- ZIO.debug("waiting...")
  result  <- promise.await    // semantically blocks until completed
} yield result
```

### Semaphore

```scala
for {
  sem <- Semaphore.make(3)     // max 3 concurrent
  _   <- ZIO.foreachPar(items)(item => sem.withPermit(process(item)))
} yield ()
```

### Queue

```scala
for {
  queue <- Queue.bounded[Int](100)
  _     <- queue.offer(1)
  _     <- queue.offer(2)
  item  <- queue.take           // semantically blocks if empty
} yield item

// Shut down queue:
queue.shutdown
```

### Hub — broadcast pub/sub

```scala
for {
  hub         <- Hub.bounded[String](100)
  subscriber1 <- hub.subscribe
  subscriber2 <- hub.subscribe
  _           <- hub.publish("hello")
  msg1        <- subscriber1.use(_.take)
  msg2        <- subscriber2.use(_.take)
} yield (msg1, msg2)
```

---

## STM — Software Transactional Memory

```scala
import zio.stm.*

// STM actions compose atomically:
val transfer: STM[Nothing, Unit] = for {
  balanceA <- TRef.make(1000)
  balanceB <- TRef.make(500)
  _        <- balanceA.update(_ - 100)
  _        <- balanceB.update(_ + 100)
} yield ()

// Commit the STM transaction to ZIO:
transfer.commit
```

---

## Testing with ZIO Test

```scala
import zio.test.*
import zio.test.Assertion.*

object MySpec extends ZIOSpecDefault:
  def spec = suite("MySpec")(
    test("addition") {
      assertTrue(1 + 1 == 2)
    },
    test("ZIO effect") {
      for result <- ZIO.succeed(42)
      yield assertTrue(result == 42)
    },
    test("with mock layer") {
      for
        db   <- ZIO.service[Database]
        user <- db.findUser(1)
      yield assertTrue(user.id == 1)
    }.provide(MockDatabase.layer),
  )
```

---

## Schedule — retries and repetition

```scala
// Repeat:
effect.repeat(Schedule.fixed(1.second))        // every second, forever
effect.repeat(Schedule.recurs(5))              // 5 times
effect.repeatN(10)                             // 10 times

// Retry on failure:
effect.retry(Schedule.exponential(100.millis)) // 100ms, 200ms, 400ms...
effect.retry(Schedule.recurs(3) && Schedule.exponential(1.second))

// Combine schedules:
Schedule.spaced(1.second) && Schedule.recurs(5)  // intersection
Schedule.spaced(1.second) || Schedule.recurs(5)  // union
```
