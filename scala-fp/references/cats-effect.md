# Cats Effect 3 Reference

## Setup

```scala
// build.sbt
libraryDependencies ++= Seq(
  "org.typelevel" %% "cats-effect"     % "3.5.7",
  "org.typelevel" %% "cats-effect-std" % "3.5.7",  // Queue, Semaphore, etc.
)
```

## IOApp — the entry point

```scala
import cats.effect.*

object Main extends IOApp.Simple:
  def run: IO[Unit] =
    IO.println("Hello, Cats Effect!")

// Or with full control over exit code:
object Main extends IOApp:
  def run(args: List[String]): IO[ExitCode] =
    myProgram.as(ExitCode.Success)
```

## Building IO values

```scala
import cats.effect.IO
import scala.concurrent.duration.*

IO.pure(42)                      // Lift pure value (must be truly pure!)
IO(sideEffect())                 // Suspend synchronous side effect
IO.delay(sideEffect())           // Alias for IO(...)
IO.unit                          // IO[Unit] = IO.pure(())
IO.never                         // IO that never completes
IO.sleep(1.second)               // Suspend for duration
IO.realTime                      // Current time as FiniteDuration
IO.realTimeInstant               // java.time.Instant
IO.println("msg")                // Print to stdout
IO.raiseError(new Exception("")) // Fail with exception
IO.canceled                      // A cancelled IO
IO.cede                          // Yield to other fibers (fairness)

// Blocking I/O — runs on a separate blocking thread pool:
IO.blocking { scala.io.Source.fromFile("file.txt").mkString }
```

## Composing effects

```scala
for {
  a <- IO(compute())
  b <- IO(computeWith(a))
  _ <- IO.println(s"result: $b")
} yield b

// Map / flatMap:
IO(1).map(_ + 1)                      // IO[Int]
IO(1).flatMap(n => IO(n + 1))         // IO[Int]
IO(1) *> IO(2)                        // sequence, discard left (IO[Int])
IO(1) <* IO(println("side"))          // sequence, discard right (IO[Int])
IO.unit >> IO("next")                 // alias for *>

// Combining:
IO(1).both(IO("a"))                   // IO[(Int, String)] — parallel!
(IO(1), IO("a")).tupled               // IO[(Int, String)] — sequential
```

## Error handling

```scala
IO.raiseError[Int](new Exception("oh no"))
  .handleError(_ => 0)               // IO[Int] — recover with value

  .handleErrorWith(e => IO(log(e)) *> IO.pure(0))  // recover with IO

  .attempt                           // IO[Either[Throwable, Int]]

  .recover { case _: TimeoutException => 0 }

  .onError(e => IO.println(s"Failed: $e"))  // run effect on error, rethrow

// Ensure cleanup:
IO(resource).guarantee(IO(cleanup()))   // always run cleanup
IO(resource).guaranteeCase {
  case Outcome.Succeeded(_) => IO.println("success")
  case Outcome.Errored(e)   => IO.println(s"error: $e")
  case Outcome.Canceled()   => IO.println("canceled")
}
```

## Resource management

```scala
import cats.effect.Resource

val managed: Resource[IO, Connection] =
  Resource.make(
    IO(openConnection())   // acquire
  )(conn =>
    IO(conn.close())       // release — always called, even on error/cancel
  )

managed.use { conn =>
  conn.query("SELECT 1")
}

// Combining resources:
val bothResources: Resource[IO, (Connection, Cache)] =
  (connectionResource, cacheResource).tupled

// From AutoCloseable:
Resource.fromAutoCloseable(IO(new FileInputStream("file.txt")))

// As bracket (lower-level):
IO(open()).bracket(use)(release)
```

## Fibers — lightweight threads

```scala
import cats.effect.{IO, FiberIO}

// Start a fiber (don't block, run concurrently):
val program: IO[Unit] = for {
  fiber <- IO.sleep(2.seconds).flatMap(_ => IO.println("done")).start
  _     <- IO.println("fiber started")
  _     <- fiber.join        // semantically block until complete
} yield ()

// Cancel a fiber:
for {
  fiber  <- longRunningTask.start
  _      <- IO.sleep(1.second)
  _      <- fiber.cancel     // sends cancellation signal
} yield ()

// Outcome — result of a joined fiber:
import cats.effect.Outcome.*
fiber.join.flatMap {
  case Succeeded(ioa) => ioa                        // IO[A]
  case Errored(e)     => IO.raiseError(e)
  case Canceled()     => IO.raiseError(new Exception("canceled"))
}
```

## Cancellation & safety

```scala
// Uncancelable region — cannot be interrupted mid-way:
IO.uncancelable { poll =>
  for {
    _   <- IO(acquireCriticalResource())
    res <- poll(interruptibleWork)   // poll re-enables cancellation here
    _   <- IO(releaseCriticalResource())
  } yield res
}

// onCancel finalizer — always runs if fiber is canceled:
longTask.onCancel(IO.println("I was cancelled, cleaning up..."))
```

## Concurrency primitives

### Ref — atomic mutable reference
```scala
import cats.effect.Ref

val program = for {
  counter <- Ref[IO].of(0)
  _       <- counter.update(_ + 1)
  _       <- counter.update(_ + 1)
  value   <- counter.get
  _       <- IO.println(s"Counter: $value")  // 2
} yield ()

// Atomic modify — read + update atomically:
counter.modify(n => (n + 1, n))   // returns old value
counter.getAndUpdate(_ + 1)       // returns old value
counter.updateAndGet(_ + 1)       // returns new value
```

### Deferred — one-shot async signal
```scala
import cats.effect.Deferred

val program = for {
  signal <- Deferred[IO, String]
  fiber  <- (IO.sleep(1.second) *> signal.complete("ready")).start
  _      <- IO.println("waiting...")
  result <- signal.get      // semantically blocks until completed
  _      <- IO.println(s"Got: $result")
} yield ()
```

### Semaphore — limit concurrency
```scala
import cats.effect.std.Semaphore

val program = Semaphore[IO](3).flatMap { sem =>  // max 3 concurrent
  List.fill(10)(sem.permit.use(_ => doWork)).parSequence
}
```

### Queue — async message passing
```scala
import cats.effect.std.Queue

val program = for {
  queue   <- Queue.bounded[IO, Int](100)
  _       <- queue.offer(1)
  _       <- queue.offer(2)
  item    <- queue.take         // semantically blocks if empty
  _       <- IO.println(item)
} yield ()
```

## Supervisor — manage background fibers

```scala
import cats.effect.std.Supervisor

Supervisor[IO](await = false).use { supervisor =>
  for {
    _ <- supervisor.supervise(backgroundTask1)
    _ <- supervisor.supervise(backgroundTask2)
    _ <- IO.sleep(10.seconds)  // fibers run in background
  } yield ()
  // supervisor cancels all fibers on Resource release
}
```

## Runtime & thread model

Cats Effect 3 uses a **work-stealing scheduler** on the JVM:
- **Compute pool**: CPU-bound fibers (default: N threads where N = CPU cores)
- **Blocking pool**: for `IO.blocking { }` — unbounded, elastic
- **Scheduler thread**: for sleep/timeout

```scala
// Check which thread a fiber is on:
IO(Thread.currentThread().getName).flatMap(IO.println)

// Shift to a specific ExecutionContext:
IO.evalOn(IO(compute()), myExecutionContext)
```
