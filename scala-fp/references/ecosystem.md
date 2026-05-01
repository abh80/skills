# Scala FP Ecosystem Quick Reference

## Cats Core patterns

```scala
// Typical imports:
import cats.*
import cats.syntax.all.*
import cats.effect.*
import cats.effect.implicits.*

// Derive typeclass instances in Scala 3:
case class User(id: Int, name: String) derives Eq, Show, Order

// Derive instances in Scala 2 (via magnolia/shapeless):
import io.circe.generic.semiauto.*
implicit val encoder: Encoder[User] = deriveEncoder
```

## Cats Effect — IOApp boilerplate

```scala
import cats.effect.*
import cats.syntax.all.*

object Main extends IOApp.Simple:
  def run: IO[Unit] =
    for {
      _    <- IO.println("Starting...")
      data <- fetchData
      _    <- processData(data)
      _    <- IO.println("Done.")
    } yield ()

  def fetchData: IO[List[String]] =
    IO.blocking(expensiveIO())

  def processData(data: List[String]): IO[Unit] =
    data.parTraverse_(item => IO(process(item)))
```

## ZIO — ZIOAppDefault boilerplate

```scala
import zio.*

object Main extends ZIOAppDefault:
  def run =
    program.provide(
      ServiceALive.layer,
      ServiceBLive.layer,
    )

  val program: ZIO[ServiceA & ServiceB, Throwable, Unit] =
    for {
      a <- ZIO.service[ServiceA]
      b <- ZIO.service[ServiceB]
      _ <- a.doSomething()
      _ <- b.doSomethingElse()
    } yield ()
```

## Key decision matrix

| I need to... | Cats Effect | ZIO |
|---|---|---|
| Run effects in parallel | `(a, b).parMapN(...)` | `a.zipPar(b)` |
| Traverse list in parallel | `list.parTraverse(f)` | `ZIO.foreachPar(list)(f)` |
| Acquire + release resource | `Resource.make(acq)(rel).use(...)` | `ZIO.acquireRelease(acq)(rel)` |
| Retry on failure | `effect.retry(policy)` | `effect.retry(Schedule.xxx)` |
| Timeout | `effect.timeout(dur)` | `effect.timeout(dur)` |
| Atomic mutable state | `Ref[IO].of(init)` | `Ref.make(init)` |
| One-shot signal | `Deferred[IO, A]` | `Promise.make[E, A]` |
| Limit concurrency | `Semaphore[IO](n)` | `Semaphore.make(n)` |
| Message queue | `Queue.bounded[IO, A](n)` | `Queue.bounded[A](n)` |
| Dependency injection | Tagless final / Kleisli | `ZLayer` |
| Stream processing | `fs2.Stream` | `ZStream` |
| Structured logging | `log4cats` | `ZIO.log*` built-in |
