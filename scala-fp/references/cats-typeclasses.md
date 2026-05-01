# Cats Typeclasses Reference

Cats (`org.typelevel::cats-core`) provides the foundational typeclass hierarchy for functional Scala. These are the abstractions you compose with, regardless of which effect system you pick.

## Imports

```scala
import cats.*
import cats.syntax.all.*          // extension methods: .map, .flatMap, .traverse, etc.
import cats.instances.all.*       // typeclass instances (rarely needed with syntax.all)
```

## Functor — map over a context

```scala
trait Functor[F[_]] {
  def map[A, B](fa: F[A])(f: A => B): F[B]
}

// Usage:
Option(42).map(_ * 2)           // Some(84)
List(1, 2, 3).map(_ + 1)        // List(2, 3, 4)
Either.right[String, Int](5).map(_ * 2)  // Right(10)

// Via Functor syntax:
Functor[List].map(List(1, 2))(x => x + 10)
```

**Law**: `fa.map(identity) == fa` (identity); `fa.map(f).map(g) == fa.map(g compose f)` (composition)

---

## Applicative — independent effects, parallel

```scala
trait Applicative[F[_]] extends Functor[F] {
  def pure[A](a: A): F[A]
  def ap[A, B](ff: F[A => B])(fa: F[A]): F[B]
}

// The most useful method via mapN / product:
(Option(1), Option("hello")).mapN((n, s) => s"$s $n")  // Some("hello 1")
(Option(1), None: Option[String]).mapN((n, s) => "x")  // None

// parMapN (for parallel effects in CE/ZIO) — same shape, concurrent semantics
(ioA, ioB).parMapN { (a, b) => combine(a, b) }
```

**Key insight**: Applicative represents *independent* effects. If either effect fails (for `Option`/`Either`), the whole thing fails, but the effects don't depend on each other's values.

---

## Monad — sequential, dependent effects

```scala
trait Monad[F[_]] extends Applicative[F] {
  def flatMap[A, B](fa: F[A])(f: A => F[B]): F[B]
  def pure[A](a: A): F[A]
}

// for-comprehension is syntactic sugar for flatMap:
for {
  user   <- findUser(id)       // F[User]
  orders <- findOrders(user)   // F[List[Order]] — depends on user
  _      <- sendEmail(orders)  // F[Unit]
} yield ()
```

**Key insight**: Monad allows the *next* effect to depend on the *result* of the *previous* one — inherently sequential.

---

## Semigroup & Monoid

```scala
trait Semigroup[A] { def combine(x: A, y: A): A }
trait Monoid[A] extends Semigroup[A] { def empty: A }

// Usage:
"hello" |+| " world"          // "hello world"
List(1, 2) |+| List(3, 4)     // List(1, 2, 3, 4)
Option(5) |+| Option(3)       // Some(8)
Map("a" -> 1) |+| Map("a" -> 2, "b" -> 3)  // Map("a" -> 3, "b" -> 3)
```

---

## Traverse — sequence effects over a structure

```scala
trait Traverse[F[_]] extends Functor[F] with Foldable[F] {
  def traverse[G[_]: Applicative, A, B](fa: F[A])(f: A => G[B]): G[F[B]]
  def sequence[G[_]: Applicative, A](fga: F[G[A]]): G[F[A]]
}

// Examples:
List(1, 2, 3).traverse(n => Option(n * 2))    // Some(List(2, 4, 6))
List(1, 2, 3).traverse(_ => None)             // None — short-circuits

// With IO (Cats Effect) — sequential traverse:
List(url1, url2, url3).traverse(fetchUrl)     // IO[List[Response]]

// Parallel traverse — all 3 fetches run concurrently:
List(url1, url2, url3).parTraverse(fetchUrl)  // IO[List[Response]] — parallel!

// sequence — flip F[G[A]] into G[F[A]]:
List(Option(1), Option(2)).sequence           // Some(List(1, 2))
List(Option(1), None).sequence               // None
```

---

## Foldable

```scala
// Fold over any traversable structure:
List(1, 2, 3).foldLeft(0)(_ + _)   // 6
List("a", "b").foldMap(_.length)   // 3 (monoid fold)
List(1, 2, 3).exists(_ > 2)        // true
List(1, 2, 3).forall(_ > 0)        // true
```

---

## EitherT, OptionT — Monad Transformers

Monad transformers allow stacking two monadic contexts:

```scala
import cats.data.EitherT

// EitherT[IO, E, A] = IO[Either[E, A]]
type Result[A] = EitherT[IO, String, A]

def findUser(id: Int): Result[User] =
  EitherT(IO(if id > 0 then Right(User(id)) else Left("Not found")))

def validate(user: User): Result[User] =
  EitherT.fromEither(if user.active then Right(user) else Left("Inactive"))

val program: Result[String] = for {
  user      <- findUser(42)
  validated <- validate(user)
} yield validated.name
// Runs as IO[Either[String, String]]
val result: IO[Either[String, String]] = program.value
```

**OptionT** similarly wraps `F[Option[A]]`:
```scala
import cats.data.OptionT
type MaybeIO[A] = OptionT[IO, A]
```

**When to use**: monad transformers are good for small stacks. For large apps with many effects, prefer ZIO's built-in typed error channel or tagless final.

---

## Validated — Accumulating errors

Unlike `Either` (fail fast), `Validated` accumulates all errors:

```scala
import cats.data.{Validated, NonEmptyList}

type ValidationResult[A] = Validated[NonEmptyList[String], A]

def validateName(name: String): ValidationResult[String] =
  if name.nonEmpty then name.valid else "Name is empty".invalidNel

def validateAge(age: Int): ValidationResult[Int] =
  if age >= 0 then age.valid else "Age must be non-negative".invalidNel

// Applicative — collects both errors, not just the first:
(validateName(""), validateAge(-1)).mapN(User.apply)
// Invalid(NonEmptyList("Name is empty", "Age must be non-negative"))

(validateName("Alice"), validateAge(30)).mapN(User.apply)
// Valid(User("Alice", 30))
```

---

## NonEmptyList (NEL), Chain, NonEmptyChain

```scala
import cats.data.{NonEmptyList, Chain, NonEmptyChain}

val nel: NonEmptyList[Int] = NonEmptyList.of(1, 2, 3)
nel.head  // 1 — guaranteed, no Option needed

val chain: Chain[Int] = Chain(1, 2, 3)  // O(1) append AND prepend
```

Use `Chain`/`NonEmptyChain` when you need efficient both-end appending (e.g., building error lists).

---

## WriterT, StateT, ReaderT / Kleisli

```scala
// ReaderT[F, R, A] = R => F[A] — dependency injection via reader
import cats.data.Kleisli
type AppEnv = DatabaseConfig
type App[A] = Kleisli[IO, AppEnv, A]

def fetchUser(id: Int): App[User] =
  Kleisli { env => IO(env.db.findUser(id)) }

// Run it by providing the environment:
fetchUser(42).run(AppEnv(db))

// StateT[F, S, A] — threading state through effects
import cats.data.StateT
type Stateful[A] = StateT[IO, MyState, A]
```

---

## Key Cats Imports Cheatsheet

```scala
// Most common — covers 95% of usage:
import cats.*
import cats.syntax.all.*

// If you need specific instances explicitly:
import cats.instances.list.*
import cats.instances.option.*

// Data types:
import cats.data.*   // EitherT, OptionT, Validated, NEL, Kleisli, ...
```
