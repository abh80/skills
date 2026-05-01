# fs2 Streams Reference

fs2 is the streaming library for the Typelevel ecosystem (works with Cats Effect). ZIO has its own `ZStream` (see ZIO reference).

## Setup

```scala
libraryDependencies ++= Seq(
  "co.fs2" %% "fs2-core"   % "3.11.0",
  "co.fs2" %% "fs2-io"     % "3.11.0",  // file, network I/O
)
```

## Core type: `Stream[F[_], O]`

A `Stream[IO, Int]` is a sequence of `Int` values produced by `IO` effects. Streams are lazy and pull-based.

## Building Streams

```scala
import fs2.*
import cats.effect.IO

Stream.empty[IO]                           // empty stream
Stream.emit(1)                             // single element
Stream.emits(List(1, 2, 3))               // from iterable
Stream(1, 2, 3)                           // shorthand for emits
Stream.eval(IO(42))                        // lift IO into a stream (one element)
Stream.evalSeq(IO(List(1, 2, 3)))          // lift IO[Seq] into stream
Stream.iterate(0)(_ + 1)                   // infinite: 0, 1, 2, ...
Stream.unfold(0)(n => Some((n, n + 1)))    // unfold: state → (output, nextState)
Stream.repeatEval(IO(Random.nextInt()))    // infinite IO evaluations
Stream.fixedRate[IO](1.second)             // tick every second
```

## Transforming Streams

```scala
Stream(1, 2, 3)
  .map(_ * 2)                       // 2, 4, 6
  .filter(_ > 2)                    // 4, 6
  .take(5)                          // at most 5 elements
  .takeWhile(_ < 10)
  .drop(2)
  .dropWhile(_ < 3)
  .flatMap(n => Stream(n, n))       // 1, 1, 2, 2, 3, 3
  .mapAccumulate(0)((acc, x) => (acc + x, acc + x))  // running sum
  .scan(0)(_ + _)                   // running total
  .chunks                           // expose underlying Chunk[A]
  .unchunks                         // flatten chunks back
  .covary[IO]                       // widen Pure stream to IO stream
```

## Effectful Transforms

```scala
// Run an IO for each element:
stream.evalMap(item => IO(process(item)))

// Tap — run effect, keep element:
stream.evalTap(item => IO.println(s"processing: $item"))

// Filter with IO:
stream.evalFilter(item => IO(item.isValid))
```

## Parallel Processing

```scala
// Evaluate N effects concurrently, preserving order:
stream.parEvalMap(maxConcurrent = 8)(item => processIO(item))

// Evaluate N effects concurrently, any order:
stream.parEvalMapUnordered(8)(item => processIO(item))

// Merge two streams (interleave outputs as they arrive):
stream1.merge(stream2)

// Zip streams:
stream1.zip(stream2)              // pairs elements pairwise
stream1.zipAll(stream2)(a0, b0)   // zip with defaults when one ends
stream1.zipLeft(stream2)          // keep left elements, advance both
```

## Collecting / Running Streams

All streams must be "compiled" to an IO to execute:

```scala
stream.compile.toList      // IO[List[O]]
stream.compile.toVector    // IO[Vector[O]]
stream.compile.drain       // IO[Unit] — discard all output, run for effects
stream.compile.count       // IO[Long] — count elements
stream.compile.fold(0)(_ + _)  // IO[Int] — fold to single value
stream.compile.last        // IO[Option[O]] — last element
stream.compile.lastOrError // IO[O] — last or exception
```

## Resource Management in Streams

```scala
// Bracket within a stream:
Stream.bracket(IO(openFile()))(file => IO(file.close()))
  .flatMap(file => Stream.evalSeq(IO(file.readLines())))

// Or using resource:
Stream.resource(fileResource)
  .flatMap(file => ...)
```

## Piping / Composition

A `Pipe[F, I, O]` is a `Stream[F, I] => Stream[F, O]`:

```scala
val myPipe: Pipe[IO, Int, String] =
  _.filter(_ > 0).map(_.toString)

// Apply a pipe:
stream.through(myPipe)

// Built-in pipes:
import fs2.text.*
stream.through(text.utf8.decode)          // bytes → strings
stream.through(text.lines)               // strings → lines
stream.through(text.utf8.encode)         // strings → bytes
```

## File I/O (fs2-io)

```scala
import fs2.io.file.{Files, Path}

// Read file as stream of bytes:
Files[IO].readAll(Path("data.csv"))

// Read as text lines:
Files[IO].readAll(Path("data.csv"))
  .through(text.utf8.decode)
  .through(text.lines)

// Write to file:
stream
  .through(text.utf8.encode)
  .through(Files[IO].writeAll(Path("output.txt")))
  .compile.drain
```

## Error Handling in Streams

```scala
// Handle errors:
stream.handleErrorWith(err => Stream.eval(IO.println(s"Error: $err")) >> Stream.empty)

// Attempt — emit Either:
stream.attempt   // Stream[IO, Either[Throwable, O]]

// Mask errors (log and continue):
stream.evalMap(item => processIO(item).attempt.flatMap {
  case Right(r) => IO.pure(Some(r))
  case Left(e)  => IO.println(s"skipping $item: $e").as(None)
}).unNone
```

## Queue-based Streams (CE + fs2)

```scala
import cats.effect.std.Queue

// Turn a Queue into a stream:
for {
  queue  <- Queue.unbounded[IO, Option[Item]]
  producer = items.traverse(i => queue.offer(Some(i))) *> queue.offer(None)
  consumer = Stream.fromQueueNoneTerminated(queue).evalMap(process)
  _      <- (producer, consumer.compile.drain).parTupled
} yield ()
```

## Common Stream Patterns

### Retry failed effects in stream
```scala
import retry.*
stream.evalMap(item =>
  retryingOnAllErrors(
    policy = RetryPolicies.exponentialBackoff[IO](100.millis),
    onError = (err, details) => IO.println(s"Retry ${details.retriesSoFar}: $err")
  )(processIO(item))
)
```

### Rate limiting
```scala
import scala.concurrent.duration.*
Stream.fixedRate[IO](100.millis)   // tick every 100ms
  .zip(stream)                     // zip with data stream — rate-limits it
  .map(_._2)                       // drop the tick
```

### Batch processing
```scala
stream
  .chunkN(100)                    // group into chunks of 100
  .evalMap(chunk => processBatch(chunk.toList))
```
