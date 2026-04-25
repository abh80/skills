# Scala performance reference

Performance considerations for refactoring suggestions. Read this before producing findings in tail recursion, `lazy val`, collection performance, or boxing categories.

## Table of contents

1. [Tail recursion and `@tailrec`](#tailrec)
2. [Branching recursion: `Eval`, trampolines](#branching)
3. [Mutual recursion](#mutual)
4. [`lazy val` cost and pitfalls](#lazy-val)
5. [`def` vs `val` for function values](#def-vs-val)
6. [Collection selection](#collections)
7. [Common collection-API smells](#collection-smells)
8. [String concatenation in loops](#strings)
9. [Boxing and primitive collections](#boxing)
10. [`for` comprehension overhead](#for-comp)
11. [`@specialized`, `@inline`, value classes](#specialization)

---

## 1. Tail recursion and `@tailrec`

A method is tail-recursive when the recursive self-call is the **last** action in every branch. The Scala compiler converts tail-recursive methods into a loop, eliminating stack growth.

```scala
import scala.annotation.tailrec

@tailrec
def length[A](xs: List[A], acc: Long = 0): Long = xs match
  case Nil     => acc
  case _ :: tl => length(tl, acc + 1)
```

**Always annotate intended tail-recursive methods with `@tailrec`.** The annotation makes the compiler verify it — without it, you can have a method that *looks* tail-recursive but isn't, and only discover it at runtime via `StackOverflowError`.

### `@tailrec` requirements

The compiler refuses to optimize a `@tailrec`-annotated method that:
- Has its recursive call in non-tail position (anything happens after the call: `1 + recur(n-1)`, `recur(n-1).map(...)`, `try { recur() } catch …`).
- Is neither `private` nor `final` (because it could be overridden, defeating the loop transformation).
- Calls itself via `super` or through a non-final method.

The error message is explicit: *"could not optimize @tailrec annotated method: it is neither private nor final so can be overridden"* or *"it contains a recursive call not in tail position"*.

**Citation:** "Functions which call themselves as their last action are called tail-recursive. The Scala compiler detects tail recursion and replaces it with a jump back to the beginning of the function." — Martin Odersky / `scala-exercises.org/scala_tutorial/tail_recursion`.

### The accumulator pattern

When the natural recursive form does work *after* the recursive call (`1 + length(tl)`), refactor with an accumulator:

```scala
// Not tail-recursive
def length[A](xs: List[A]): Long = xs match
  case Nil     => 0
  case _ :: tl => 1 + length(tl)  // work happens after the call

// Tail-recursive via accumulator helper
def length[A](xs: List[A]): Long =
  @tailrec def loop(rest: List[A], acc: Long): Long = rest match
    case Nil     => acc
    case _ :: tl => loop(tl, acc + 1)
  loop(xs, 0)
```

Nesting the helper keeps the public signature clean and limits the helper's scope.

**Performance:** `improved` for any recursion deep enough to risk stack overflow. `equivalent` to a hand-written `while` loop in most cases.

---

## 2. Branching recursion that resists tail-call form

Tree traversals where you recurse into both children and combine the results cannot be naïvely tail-recursive — you need work after both calls. Options:

- **Explicit stack:** convert the recursion to a while-loop over a `mutable.Stack[Frame]`. Verbose but allocation-cheap and stack-safe.
- **`cats.Eval`:** wraps recursion in a stack-safe lazy structure. `Eval.defer { … }` defers evaluation until the trampoline is run.
- **Continuations / CPS:** rewrite to pass continuations instead of returning.

Flag deeply-recursive tree code as `needs-redesign` rather than mechanically suggesting `@tailrec`.

---

## 3. Mutual recursion

Scala only optimizes **direct** self-recursion. Mutual recursion (`a` calls `b` calls `a`) is NOT TCO'd and will stack-overflow on deep inputs. Flag candidates and suggest:
- Merging into a single function with a tag/discriminator parameter.
- Trampolining (`scala.util.control.TailCalls.TailRec`).

---

## 4. `lazy val` cost and pitfalls

A `lazy val` compiles into a synchronized double-checked init guard. Each read goes through:
1. Check the `bitmap$0` flag (volatile read).
2. If unset, enter `synchronized(this)` block, recheck, init, set flag.
3. If set, return cached value.

The synchronized block runs only on first access, but the volatile read on the flag happens on **every** read.

**Use `lazy val` when:**
- The value is genuinely expensive to compute.
- It might never be needed.
- It is referenced more than once.

**Avoid `lazy val` when:**
- The value is cheap (a `String` interpolation, a small `case class` literal). The lock-init machinery costs more than just computing it.
- It's accessed millions of times in a tight loop. The volatile read adds up.

### `lazy val` deadlock

Two `lazy val`s in different objects that reference each other can deadlock under concurrent first-access: thread A holds `Foo`'s lock waiting for `Bar`, thread B holds `Bar`'s lock waiting for `Foo`. Documented and reproducible.

**Citation:** "The compiler introduces a monitor for every lazy val. Due to this, compiler locks the whole instance during initialization." — `baeldung.com/scala/lazy-val` (analysis of generated bytecode).

**Performance:** `lazy val` for a cheap value is `regression`, not improvement. Audit accordingly.

---

## 5. `def` vs `val` for function values

```scala
def even1: Int => Boolean = _ % 2 == 0   // allocates new Function1 on every reference
val even2: Int => Boolean = _ % 2 == 0   // allocates once
```

```scala
def even1 eq def even1   // false — different instances each time
val even2 eq val even2   // true  — same instance
```

For functions referenced in hot loops or passed to many `map`/`filter` calls, `val` avoids per-reference `FunctionN` allocations. For functions referenced once or rarely, `def` is fine.

`lazy val` for a cheap callable is the worst of both: per-read lock check on a value that wasn't expensive in the first place.

---

## 6. Collection selection

| Need | Default choice | Why |
|---|---|---|
| Immutable, prepend-heavy linear access | `List` | O(1) prepend (`::`), O(n) index/append |
| Immutable, balanced general-purpose indexed | `Vector` | Effectively O(1) access, append, prepend, update |
| Immutable indexed with primitive performance | `ArraySeq` (Scala 2.13+) | True O(1) access, no boxing for primitives via specialization, lower memory than `Vector` |
| Bulk-build then read | `ListBuffer` / `ArrayBuffer` + `.result()` | Builders avoid intermediate-collection allocations |
| Hot-path numeric work | `Array[Int]` / `Array[Double]` | True primitive arrays, no boxing |
| Lookup-heavy by key | `mutable.HashMap` for tight loops; `immutable.Map` for sharing | Mutable hash lookups are 2–4× faster than immutable trees |

**Citation:** "Vectors are a useful 'default' data structure to reach for, but if it's at all possible, working directly with Lists or Arrays or mutable.Buffers might have an order-of-magnitude less performance overhead." — Li Haoyi, *Benchmarking Scala Collections*.

### `List` index access is a bug

```scala
xs(1_000_000)  // O(n) for List — traverses 999,999 elements
```

If you index into a sequence by position, it should not be a `List`. Flag `List(i)` or `xs.apply(i)` on `LinearSeq`s.

### `List.size == 0`

`size` on a `LinearSeq` (`List`) is O(n). Use `isEmpty` (O(1)). Same for `length`.

```scala
// Bad on List
if (xs.size == 0) ...
if (xs.length > 0) ...
// Good
if (xs.isEmpty) ...
if (xs.nonEmpty) ...
```

---

## 7. Common collection-API smells

| Anti-pattern | Better | Why |
|---|---|---|
| `xs.filter(p).head` / `xs.filter(p).headOption` | `xs.find(p)` | Short-circuits at first match |
| `xs.filter(p).size` | `xs.count(p)` | No intermediate collection |
| `xs.map(f).flatten` | `xs.flatMap(f)` | One pass instead of two |
| `xs.map(f).headOption` | `xs.headOption.map(f)` | Don't transform what you discard |
| `xs.map(f).toList` chained 3+ times | one `foldLeft` or a fused pass | Each step allocates a fresh collection |
| `xs.zipWithIndex.map { case (x, i) => … }` | `xs.iterator.zipWithIndex.map(…)` | Iterator avoids materializing the index pairing |
| `xs.toList.reverse` | `foldLeft(Nil)(_.::(_))` | Builds the result in the right order in one pass |

These are not always wins. Profile before "fixing" cold paths.

---

## 8. String concatenation in loops

```scala
// Bad — allocates a fresh String per iteration, O(n²) overall
var s = ""
for (i <- 1 to 1000) s = s + i.toString

// Good — StringBuilder is O(n)
val sb = new StringBuilder
for (i <- 1 to 1000) sb.append(i)
val s = sb.toString
```

Or `xs.mkString(",")` for the join case. Or `xs.foldLeft(new StringBuilder)(_.append(_)).toString`.

**Performance:** `improved` — measurably so for any loop over more than a few dozen iterations.

---

## 9. Boxing and primitive collections

`List[Int]`, `Vector[Int]`, `Map[Int, Int]`, etc., box every primitive into a `java.lang.Integer`. For hot-path numeric work this is often the dominant cost.

Fixes (in order of preference):
- Use `Array[Int]` (or other primitive array). True primitive storage, no boxing, fast indexed access.
- Use `ArraySeq.ofInt` (immutable, primitive-backed).
- For tight loops, write a `while` loop over an `Array` rather than `xs.foreach(…)` on a generic collection.

When suggesting "switch from `Array` to `List`/`Vector`" for "idiom" reasons in numeric code, classify as `needs-benchmark` — you may be introducing boxing overhead the user didn't know about.

---

## 10. `for` comprehension overhead

`for` is sugar over `flatMap`/`map`/`withFilter`/`foreach`. Each clause becomes a method call. For tight numeric loops, an explicit `while` loop can be measurably faster (reported 2–3× in some benchmarks, though Scala 2.12+ on Java 8+ closed most of the gap).

**Do not** routinely suggest replacing `for` with `while`. Suggest only when:
- The code is in a documented hot path.
- Profiling has identified the `for` as a bottleneck.
- The replacement does not lose pattern-matching destructuring or guard clarity.

**Citation:** "for comprehensions are rewritten internally into a nested sequence of method calls, so be careful for using fors in performance-critical code." — community guideline.

### Variable-declaration position matters

```scala
// Slower — declarations in for header
for
  i <- 0 until n
  pi = arr(i)
  j <- 0 until n
  pj = arr(j)
do compute(pi, pj)

// Faster — declarations in body
for
  i <- 0 until n
  j <- 0 until n
do
  val pi = arr(i)
  val pj = arr(j)
  compute(pi, pj)
```

Reported 2.5–3× difference. Reason is unclear in the bytecode but reproducible.

---

## 11. `@specialized`, `@inline`, value-class zero-cost — do not silently break

When auditing existing code, treat these as load-bearing:

- **`@specialized` on a generic method/class:** the compiler generates separate primitive-type versions to avoid boxing. Replacing the specialized class with a non-specialized equivalent (e.g., a generic `case class`) reintroduces boxing.
- **`@inline final def`:** instructs the compiler (with `-opt:inline-from`) to inline the method body. Removing the annotation OR replacing with a non-final form can lose the inlining.
- **Value classes (`extends AnyVal`):** zero-allocation in the simple call/return case. Replacing with a regular `class` introduces wrapper allocation. Replacing with an opaque type is usually equivalent or better — but the corner case of opaque types over a primitive used through a generic parameter still boxes (same as bare primitives under generics).
- **`final case class`:** allows JIT to devirtualize and inline aggressively. Removing `final` can cost inlining.

For any suggestion that touches code with these annotations, classify as `needs-benchmark` and explain what could regress.
