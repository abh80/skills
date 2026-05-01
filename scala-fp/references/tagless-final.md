# Tagless Final Architecture

Tagless Final (TF) is the idiomatic way to write abstract, testable, composable code in the Cats Effect ecosystem. Programs are written against **type class constraints** on an abstract `F[_]`, not a concrete `IO`.

## Why Tagless Final?

1. **Testability** — swap `IO` for `Id`, `SyncIO`, or a test-specific monad in tests
2. **Composability** — algebras compose without coupling to a specific runtime
3. **Documentation** — type signature declares exactly which capabilities are needed
4. **Cross-runtime** — same code works with CE, ZIO, Monix (via interop)

## Basic Pattern

```scala
import cats.Monad
import cats.effect.{Sync, Async, Concurrent}
import cats.syntax.all.*

// 1. Define the algebra (interface):
trait UserRepository[F[_]]:
  def findById(id: Int): F[Option[User]]
  def save(user: User): F[Unit]
  def list: F[List[User]]

// 2. Implement (can use any F[_] that satisfies constraints):
class DatabaseUserRepo[F[_]: Sync](db: DatabaseDriver) extends UserRepository[F]:
  def findById(id: Int): F[Option[User]] =
    Sync[F].blocking(db.queryOne(s"SELECT * FROM users WHERE id = $id"))
  def save(user: User): F[Unit] =
    Sync[F].blocking(db.execute(s"INSERT INTO users VALUES (${user.id})")).void
  def list: F[List[User]] =
    Sync[F].blocking(db.queryList("SELECT * FROM users"))

// 3. In-memory implementation for tests:
class InMemoryUserRepo[F[_]: Monad] extends UserRepository[F]:
  private val store = scala.collection.mutable.Map[Int, User]()
  def findById(id: Int): F[Option[User]] = Monad[F].pure(store.get(id))
  def save(user: User): F[Unit] = Monad[F].pure(store += (user.id -> user)).void
  def list: F[List[User]] = Monad[F].pure(store.values.toList)

// 4. Program that uses the algebra:
def createUserProgram[F[_]: Monad](repo: UserRepository[F])(user: User): F[String] =
  for {
    existing <- repo.findById(user.id)
    result   <- existing match {
      case Some(_) => Monad[F].pure("User already exists")
      case None    => repo.save(user).as("User created")
    }
  } yield result
```

## Typeclass Constraints — What to Use

| Constraint | Provides | Use when you need |
|-----------|---------|-----------------|
| `Functor[F]` | `map` | Pure transformations only |
| `Monad[F]` | `map`, `flatMap`, `pure` | Sequential effects |
| `MonadThrow[F]` | `Monad` + `raiseError` + `handleError` | Error handling |
| `Sync[F]` | `MonadThrow` + synchronous suspension | Wrapping sync side effects |
| `Async[F]` | `Sync` + async callbacks | Callback-based async APIs |
| `Temporal[F]` | `Async` + `sleep` + timers | Time-based effects |
| `Concurrent[F]` | `Temporal` + `start` + `race` | Concurrency / fibers |
| `GenSpawn[F, E]` | Low-level fiber spawning | Custom effect types |

**Rule**: request the *weakest* constraint that satisfies your needs. If you only need `map`, use `Functor`. If you need `flatMap`, use `Monad`. This makes code more general and reusable.

## Composing Multiple Algebras

```scala
// Multiple algebras:
trait Logger[F[_]]:
  def info(msg: String): F[Unit]
  def error(msg: String, cause: Throwable): F[Unit]

trait EmailService[F[_]]:
  def sendWelcome(user: User): F[Unit]

// Service that depends on multiple algebras:
class UserService[F[_]: MonadThrow](
  repo:  UserRepository[F],
  email: EmailService[F],
  log:   Logger[F],
):
  def register(user: User): F[Unit] =
    for {
      _   <- log.info(s"Registering ${user.name}")
      _   <- repo.save(user).handleErrorWith { e =>
                log.error(s"Save failed", e) *> MonadThrow[F].raiseError(e)
             }
      _   <- email.sendWelcome(user)
      _   <- log.info(s"Registered ${user.name} successfully")
    } yield ()
```

## Wiring at the Edge (IOApp)

Only at the entry point do you choose the concrete `F`:

```scala
import cats.effect.*

object Main extends IOApp.Simple:
  def run: IO[Unit] =
    val repo:    UserRepository[IO]  = DatabaseUserRepo[IO](DatabaseDriver.create())
    val emailSvc: EmailService[IO]   = SmtpEmailService[IO](SmtpConfig.load())
    val logger:  Logger[IO]          = Slf4jLogger[IO]

    val userService = UserService[IO](repo, emailSvc, logger)

    for {
      _ <- userService.register(User(1, "Alice", "alice@example.com"))
      _ <- IO.println("Done")
    } yield ()
```

## Testing with Id (pure, synchronous)

```scala
import cats.Id

// Id[A] is just A — no effects, synchronous, great for unit tests
val repoTest: UserRepository[Id] = InMemoryUserRepo[Id]
val result: String = createUserProgram[Id](repoTest)(User(1, "Alice"))
assert(result == "User created")
```

## Clock / Logger injection (common algebras)

```scala
// Rather than relying on concrete IO.realTime:
trait Clock[F[_]]:
  def now: F[java.time.Instant]

class SystemClock[F[_]: Sync] extends Clock[F]:
  def now = Sync[F].delay(java.time.Instant.now())

class FakeClock[F[_]: Monad](fixedTime: java.time.Instant) extends Clock[F]:
  def now = Monad[F].pure(fixedTime)
```

## When NOT to use Tagless Final

- **Simple scripts / small programs** — concrete `IO` is fine
- **ZIO programs** — ZIO has its own DI via `ZLayer`; TF and ZIO have an impedance mismatch
- **Full-stack abstraction overhead isn't needed** — when you're 100% committed to CE + IO

## Real-world Structure

```
src/main/scala/myapp/
├── domain/         — pure domain models, no F[_]
├── algebras/       — traits with F[_] (UserRepository, Logger, etc.)
├── services/       — business logic using algebras
├── infrastructure/ — concrete implementations (DB, SMTP, etc.)
└── Main.scala      — wire everything together with IO
```
