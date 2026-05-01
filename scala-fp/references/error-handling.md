# Error Handling in Functional Scala

## Cats Effect Error Handling

```scala
// Raise an error:
IO.raiseError(new Exception("boom"))
MonadThrow[F].raiseError(new Exception("boom"))

// Handle errors:
effect.handleError(_ => defaultValue)
effect.handleErrorWith(e => IO.println(s"err: $e") *> IO.pure(default))
effect.recover { case _: NotFoundException => NotFound }
effect.recoverWith { case _: TimeoutException => retryIO }

// Materialise error:
effect.attempt           // IO[Either[Throwable, A]]
effect.attemptNarrow[MyError]  // IO[Either[MyError, A]] (unsafe cast)

// Ensure / finalise:
effect.guarantee(cleanup)
effect.guaranteeCase {
  case Outcome.Succeeded(_) => IO.unit
  case Outcome.Errored(e)   => logError(e)
  case Outcome.Canceled()   => logCancel
}
```

## Domain Error Hierarchy (Best Practice)

```scala
// Define a sealed hierarchy for your domain errors:
sealed trait AppError extends Throwable
case class NotFound(id: Int)       extends AppError
case class Unauthorized(msg: String) extends AppError
case class ValidationError(field: String, msg: String) extends AppError
case class DatabaseError(cause: Throwable) extends AppError

// Use handleErrorWith to map infrastructure errors to domain errors:
def findUser(id: Int): IO[User] =
  IO.blocking(db.query(id))
    .handleErrorWith {
      case _: SQLException => IO.raiseError(DatabaseError(???))
      case _               => IO.raiseError(NotFound(id))
    }
```

## ZIO Typed Errors

ZIO's `E` type parameter is the superpower: errors are tracked in types.

```scala
// Typed error — compiler forces you to handle it:
def findUser(id: Int): ZIO[Any, NotFoundError, User] =
  ZIO.fail(NotFoundError(id))

// Recover from typed error:
findUser(42)
  .catchAll(err => ZIO.succeed(User.default))
  .catchSome { case NotFoundError(id) => ZIO.succeed(User.default) }

// Map error type:
findUser(42).mapError(AppError.NotFound(_))

// Both errors and success in one fold:
findUser(42).fold(
  err  => Left(err.toString),
  user => Right(user.name)
)

// typed error → Either:
findUser(42).either   // ZIO[Any, Nothing, Either[NotFoundError, User]]
```

## Validated — Parallel Error Accumulation

Use when validating multiple independent things and you want **all** errors:

```scala
import cats.data.{Validated, NonEmptyList}
import cats.syntax.all.*

type V[A] = Validated[NonEmptyList[String], A]

def validateEmail(s: String): V[String] =
  if s.contains("@") then s.validNel else "Invalid email".invalidNel

def validateAge(n: Int): V[Int] =
  if n >= 18 then n.validNel else "Must be 18+".invalidNel

def validateName(s: String): V[String] =
  if s.nonEmpty then s.validNel else "Name required".invalidNel

// Validate all at once — accumulates errors:
(validateName(""), validateEmail("bad"), validateAge(16)).mapN(User.apply)
// Invalid(NonEmptyList("Name required", "Invalid email", "Must be 18+"))
```

---

# Ecosystem Reference

## Library Versions (2025)

| Library | Group ID | Artifact | Latest |
|---------|---------|---------|--------|
| **Cats** | `org.typelevel` | `cats-core` | **2.12.0** |
| **Cats Effect** | `org.typelevel` | `cats-effect` | **3.5.7** |
| **fs2** | `co.fs2` | `fs2-core` | **3.11.0** |
| **ZIO** | `dev.zio` | `zio` | **2.1.15** |
| **ZIO Streams** | `dev.zio` | `zio-streams` | **2.1.15** |
| **ZIO Test** | `dev.zio` | `zio-test` | **2.1.15** |
| **http4s** | `org.http4s` | `http4s-ember-server` | **0.23.x** |
| **doobie** | `org.tpolecat` | `doobie-core` | **1.0.0-RC5** |
| **skunk** | `org.tpolecat` | `skunk-core` | **0.6.x** |
| **tapir** | `com.softwaremill.sttp.tapir` | `tapir-core` | **1.x** |
| **zio-http** | `dev.zio` | `zio-http` | **3.x` |
| **quill** | `io.getquill` | `quill-zio` | **4.x** |
| **Monix** | `io.monix` | `monix` | **3.4.x** |

## Typelevel Stack (CE-based)

```scala
// HTTP server:
libraryDependencies += "org.http4s" %% "http4s-ember-server" % "0.23.27"
libraryDependencies += "org.http4s" %% "http4s-ember-client" % "0.23.27"
libraryDependencies += "org.http4s" %% "http4s-dsl"          % "0.23.27"
libraryDependencies += "org.http4s" %% "http4s-circe"        % "0.23.27"

// Database (JDBC, type-safe, CE):
libraryDependencies += "org.tpolecat" %% "doobie-core"   % "1.0.0-RC5"
libraryDependencies += "org.tpolecat" %% "doobie-hikari" % "1.0.0-RC5"
libraryDependencies += "org.tpolecat" %% "doobie-postgres" % "1.0.0-RC5"

// Postgres (pure, no JDBC):
libraryDependencies += "org.tpolecat" %% "skunk-core" % "0.6.4"

// JSON:
libraryDependencies += "io.circe" %% "circe-core"    % "0.14.10"
libraryDependencies += "io.circe" %% "circe-generic" % "0.14.10"
libraryDependencies += "io.circe" %% "circe-parser"  % "0.14.10"

// Config:
libraryDependencies += "is.cir" %% "ciris" % "3.6.0"

// Structured logging:
libraryDependencies += "org.typelevel" %% "log4cats-slf4j" % "2.7.0"
```

## ZIO Stack

```scala
// Core:
libraryDependencies += "dev.zio" %% "zio"        % "2.1.15"
libraryDependencies += "dev.zio" %% "zio-streams" % "2.1.15"

// HTTP:
libraryDependencies += "dev.zio" %% "zio-http" % "3.0.1"

// Database:
libraryDependencies += "io.getquill" %% "quill-zio"      % "4.8.6"
libraryDependencies += "io.getquill" %% "quill-jdbc-zio"  % "4.8.6"

// JSON:
libraryDependencies += "dev.zio" %% "zio-json" % "0.7.3"

// Config:
libraryDependencies += "dev.zio" %% "zio-config"          % "4.0.3"
libraryDependencies += "dev.zio" %% "zio-config-typesafe" % "4.0.3"

// Logging:
libraryDependencies += "dev.zio" %% "zio-logging-slf4j" % "2.3.1"

// Cache:
libraryDependencies += "dev.zio" %% "zio-cache" % "0.2.3"

// Redis:
libraryDependencies += "dev.zio" %% "zio-redis" % "1.0.0"
```

## Interoperability

### ZIO ↔ Cats Effect interop

```scala
libraryDependencies += "dev.zio" %% "zio-interop-cats" % "23.1.0.3"

import zio.interop.catz.*

// ZIO[R, E, A] can be used where F[_]: Async is expected:
val task: Task[Int] = ZIO.succeed(42)
val io: IO[Int] = task.toEffect[IO]

// Use http4s (CE-based) from ZIO:
import zio.interop.catz.*
val server: ZIO[Any, Throwable, Nothing] = 
  EmberServerBuilder.default[Task].withHttpApp(routes).build.use(_ => ZIO.never)
```

### Future interop

```scala
// CE: Future → IO
IO.fromFuture(IO(myFuture))

// CE: IO → Future
import cats.effect.unsafe.implicits.global  // provides runtime
myIO.unsafeToFuture()

// ZIO: Future → ZIO
ZIO.fromFuture(implicit ec => myFuture)

// ZIO: ZIO → Future
zio.Runtime.default.unsafeRunToFuture(myEffect)
```
