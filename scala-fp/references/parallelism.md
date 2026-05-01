# Parallel Programming Patterns

## Cats Effect Parallel Patterns

### parMapN — fixed number of parallel effects

```scala
import cats.effect.IO
import cats.syntax.all.*

// 2 effects in parallel:
(fetchUser(id), fetchOrders(id)).parMapN { (user, orders) =>
  UserWithOrders(user, orders)
}

// 3 effects in parallel:
(ioA, ioB, ioC).parMapN((a, b, c) => Result(a, b, c))

// Tuple syntax (collect results as tuple):
(ioA, ioB).parTupled   // IO[(A, B)]
```

If **any** effect fails, all others are **cancelled** and the error is raised.

### parTraverse — parallel map over a collection

```scala
// Sequential (one at a time):
list.traverse(item => processItem(item))        // IO[List[Result]]

// Parallel (all at once):
list.parTraverse(item => processItem(item))      // IO[List[Result]]

// Parallel with bounded concurrency (max 10 at a time):
import cats.effect.std.Semaphore
Semaphore[IO](10).flatMap { sem =>
  list.parTraverse(item => sem.permit.use(_ => processItem(item)))
}
```

### parSequence — parallel sequence

```scala
val effects: List[IO[Int]] = List(ioA, ioB, ioC)
effects.parSequence      // IO[List[Int]] — all run in parallel
effects.sequence         // IO[List[Int]] — sequential
```

### IO.race — first to complete wins

```scala
IO.race(ioA, ioB)         // IO[Either[A, B]] — loser is cancelled

// Practical: timeout pattern
IO.race(
  longRunningTask,
  IO.sleep(5.seconds)
).flatMap {
  case Left(result)  => IO.pure(result)
  case Right(_)      => IO.raiseError(new TimeoutException())
}

// Or just:
longRunningTask.timeout(5.seconds)   // built-in timeout

// racePair — get the loser's fiber too:
IO.racePair(ioA, ioB).flatMap {
  case Left((a, fiberB))  => fiberB.cancel.as(a)     // A won, cancel B
  case Right((fiberA, b)) => fiberA.cancel.as(b)     // B won, cancel A
}
```

### Fiber-level parallelism (manual)

```scala
// When you need fine-grained control:
for {
  f1     <- task1.start
  f2     <- task2.start
  f3     <- task3.start
  r1     <- f1.join.flatMap(_.embedNever)   // extract result
  r2     <- f2.join.flatMap(_.embedNever)
  r3     <- f3.join.flatMap(_.embedNever)
} yield (r1, r2, r3)
```

**Prefer `parMapN`/`parTraverse` over manual fiber management** — they handle cancellation correctly.

---

## ZIO Parallel Patterns

### zipPar / zipWithPar

```scala
// Two effects in parallel:
effectA.zipPar(effectB)              // ZIO[(A, B)]
effectA.zipWithPar(effectB)((a, b) => Result(a, b))

// Symbolic alias:
effectA <&> effectB                  // ZIO[(A, B)]
```

### ZIO.collectAllPar / foreachPar

```scala
// All in parallel, collect results:
ZIO.collectAllPar(List(eff1, eff2, eff3))   // ZIO[List[A]]

// Parallel map over collection:
ZIO.foreachPar(list)(item => processItem(item))   // ZIO[List[Result]]

// Bounded concurrency (max 8 parallel):
ZIO.foreachPar(list)(processItem).withParallelism(8)

// Parallel fold:
ZIO.foldLeftPar(list)(zero)((acc, item) => combineIO(acc, item))

// Discard results — just run for effects:
ZIO.collectAllParDiscard(List(eff1, eff2, eff3))
ZIO.foreachParDiscard(list)(processItem)
```

### ZIO.race

```scala
ZIO.race(effectA, effectB)                  // A | B — loser interrupted
ZIO.raceFirst(effectA, effectB)             // same as race
effectA.race(effectB)                       // method syntax

// Timeout:
effect.timeout(5.seconds)                   // Option[A]
effect.timeoutFail(TimeoutError)(5.seconds) // ZIO[R, E | TimeoutError, A]
```

### Parallel error handling

```scala
// If any parallel effect fails, all others are interrupted:
ZIO.foreachPar(list)(effect)

// Collect all errors instead of failing fast:
ZIO.validatePar(list)(effect)    // ZIO[..., ::[E], List[A]]

// Partition successes and failures:
ZIO.partitionPar(list)(effect)   // ZIO[..., Nothing, (List[E], List[A])]
```

---

## Structured Concurrency — the Right Way

Both CE and ZIO guarantee **structured concurrency**: all forked effects are properly cleaned up when their parent scope ends.

### DO: use high-level operators

```scala
// CE — parMapN ensures both start AND both are cleaned up:
(task1, task2).parMapN((a, b) => combine(a, b))

// ZIO — zipPar same guarantee:
task1.zipPar(task2)
```

### DO: use Supervisor (CE) or forkScoped (ZIO) for background tasks

```scala
// CE — background fiber tied to Resource lifecycle:
myTask.background.use { outcomeSignal =>
  mainWork
}

// ZIO — fiber tied to current scope:
for {
  _ <- backgroundTask.forkScoped  // cancelled when scope ends
  _ <- mainWork
} yield ()
```

### AVOID: leaking fibers

```scala
// BAD — fiber is never joined or cancelled (leak!):
for {
  _ <- task.start   // fire and forget — dangerous
  _ <- otherWork
} yield ()

// GOOD — use background or Supervisor or explicitly join/cancel:
for {
  fiber <- task.start
  _     <- otherWork
  _     <- fiber.cancel   // or fiber.join
} yield ()
```

---

## Common Concurrency Patterns

### Producer-Consumer Queue

```scala
// CE:
for {
  queue    <- Queue.bounded[IO, Item](100)
  producer <- items.traverse(item => queue.offer(item)).start
  consumer <- queue.take.flatMap(process).foreverM.start
  _        <- IO.sleep(10.seconds)
  _        <- producer.cancel
  _        <- consumer.cancel
} yield ()

// ZIO:
for {
  queue    <- Queue.bounded[Item](100)
  producer <- ZIO.foreach(items)(queue.offer).fork
  consumer <- queue.take.flatMap(process).forever.fork
  _        <- ZIO.sleep(10.seconds)
  _        <- producer.interrupt
  _        <- consumer.interrupt
} yield ()
```

### Fan-out / Fan-in

```scala
// CE — split work across N workers, collect results:
val numWorkers = 8
Queue.bounded[IO, Item](100).flatMap { queue =>
  val producer = items.traverse(queue.offer)
  val worker   = queue.take.flatMap(processItem)
  val workers  = List.fill(numWorkers)(worker.foreverM)
  (producer, workers.parSequence).parMapN((_, results) => results)
}

// ZIO:
for {
  queue   <- Queue.bounded[Item](100)
  _       <- ZIO.foreach(items)(queue.offer).fork
  results <- ZIO.foreachPar(1 to 8)(_ => queue.take.flatMap(processItem))
} yield results
```

### STM — Transactional state (ZIO)

```scala
import zio.stm.*

// Bank transfer — atomic, no race conditions, no locks:
def transfer(from: TRef[Int], to: TRef[Int], amount: Int): UIO[Unit] =
  (for {
    balance <- from.get
    _       <- STM.check(balance >= amount)  // retry if insufficient
    _       <- from.update(_ - amount)
    _       <- to.update(_ + amount)
  } yield ()).commit
```

### Parallel Computation Pipeline

```scala
// CE — multi-stage parallel pipeline with fs2:
import fs2.Stream

Stream.emits(items)
  .parEvalMap(maxConcurrent = 8)(stage1)   // parallel stage 1
  .parEvalMap(maxConcurrent = 4)(stage2)   // parallel stage 2
  .compile.toList

// ZIO Streams:
ZStream.fromIterable(items)
  .mapZIOPar(8)(stage1)
  .mapZIOPar(4)(stage2)
  .runCollect
```

---

## Thread Model Comparison

| | Cats Effect | ZIO |
|--|------------|-----|
| Fibers | Lightweight M:N on CE runtime | Lightweight M:N on ZIO runtime |
| Blocking I/O | `IO.blocking { }` | `ZIO.blocking { }` / `ZIO.attemptBlockingIO` |
| Compute pool | Work-stealing (CPU cores) | Work-stealing (CPU cores) |
| Scheduler | Internal | Internal |
| JVM Virtual Threads | Supported (CE 3.6+) | Supported |
| JS single-thread | Cooperative scheduling | Cooperative scheduling |

**Rule**: Never block a fiber on the compute pool. Always use `IO.blocking` / `ZIO.blocking` for thread-blocking operations (JDBC, file I/O, etc.).
