# Authoritative documentation URLs

Quick lookup for citations. When making a finding, find the closest matching topic and use the URL listed.

## Primary sources accepted

- `docs.scala-lang.org` — official Scala documentation
- `docs.scala-lang.org/scala3/reference/` — Scala 3 reference
- `docs.scala-lang.org/scala3/guides/migration/` — Scala 3 migration guide
- `docs.scala-lang.org/scala3/book/` — Scala 3 book
- `dotty.epfl.ch` — Dotty docs (current/nightly Scala 3)
- `nightly.scala-lang.org/docs/reference/` — Scala 3 nightly reference
- `scala-lang.org/api/` — Scaladoc

If a finding cannot be supported by a URL from the above, mark it `uncited` and explain.

---

## URL index by topic

### Scala 3 idioms

| Topic | URL |
|---|---|
| Enums (general) | `docs.scala-lang.org/scala3/reference/enums/enums.html` |
| Enums (ADT-style) | `docs.scala-lang.org/scala3/reference/enums/adts.html` |
| Given instances | `docs.scala-lang.org/scala3/reference/contextual/givens.html` |
| Using clauses | `docs.scala-lang.org/scala3/reference/contextual/using-clauses.html` |
| Extension methods | `docs.scala-lang.org/scala3/reference/contextual/extension-methods.html` |
| Opaque types (book) | `docs.scala-lang.org/scala3/book/types-opaque-types.html` |
| Opaque types (SIP-35) | `docs.scala-lang.org/sips/opaque-types.html` |
| Type class derivation | `docs.scala-lang.org/scala3/reference/contextual/derivation.html` |
| Top-level definitions | `docs.scala-lang.org/scala3/book/taste-toplevel-definitions.html` |
| Export clauses | `docs.scala-lang.org/scala3/reference/other-new-features/export.html` |
| Union types | `docs.scala-lang.org/scala3/reference/new-types/union-types.html` |
| Intersection types | `docs.scala-lang.org/scala3/reference/new-types/intersection-types.html` |
| `@main` methods | `docs.scala-lang.org/scala3/reference/changed-features/main-functions.html` |
| Inline | `docs.scala-lang.org/scala3/guides/macros/inline.html` |
| Conversion (implicit conversions) | `docs.scala-lang.org/scala3/reference/contextual/conversions.html` |
| New control syntax | `docs.scala-lang.org/scala3/reference/other-new-features/control-syntax.html` |
| Optional braces / indentation | `docs.scala-lang.org/scala3/reference/other-new-features/indentation.html` |
| Trait parameters | `docs.scala-lang.org/scala3/reference/other-new-features/trait-parameters.html` |
| Implicits → givens migration | `docs.scala-lang.org/scala3/reference/contextual/relationship-implicits.html` |
| Eta-expansion (changed) | `docs.scala-lang.org/scala3/reference/changed-features/eta-expansion.html` |
| New-in-Scala-3 overview | `docs.scala-lang.org/scala3/new-in-scala3.html` |

### Migration

| Topic | URL |
|---|---|
| Migration overview | `docs.scala-lang.org/scala3/guides/migration/compatibility-intro.html` |
| Dropped features (do-while, m\_, procedure syntax, existentials, Symbol, etc.) | `docs.scala-lang.org/scala3/guides/migration/incompat-dropped-features.html` |
| Syntactic incompatibilities | `docs.scala-lang.org/scala3/guides/migration/incompat-syntactic.html` |
| Contextual abstraction incompatibilities | `docs.scala-lang.org/scala3/guides/migration/incompat-contextual-abstractions.html` |
| Type system incompatibilities | `docs.scala-lang.org/scala3/guides/migration/incompat-type-inference.html` |
| Other incompatibilities | `docs.scala-lang.org/scala3/guides/migration/incompat-other-changes.html` |

### Standard library / patterns

| Topic | URL |
|---|---|
| Pattern matching (tour) | `docs.scala-lang.org/tour/pattern-matching.html` |
| Functional error handling | `docs.scala-lang.org/scala3/book/functional-error-handling.html` (Scala 3 book) or `docs.scala-lang.org/overviews/scala-book/functional-error-handling.html` (Scala 2 book) |
| Futures and Promises | `docs.scala-lang.org/overviews/core/futures.html` |
| Value classes (legacy) | `docs.scala-lang.org/overviews/core/value-classes.html` |
| Implicit classes (legacy) | `docs.scala-lang.org/overviews/core/implicit-classes.html` |
| Collections types (Scala 3 book) | `docs.scala-lang.org/scala3/book/collections-classes.html` |
| Collections performance characteristics | `docs.scala-lang.org/overviews/collections-2.13/performance-characteristics.html` |
| Collections introduction | `docs.scala-lang.org/overviews/collections-2.13/introduction.html` |

### Scala 3 spec / nightly (when reference doesn't have a stable page)

| Topic | URL pattern |
|---|---|
| Eta-expansion full spec | `nightly.scala-lang.org/docs/reference/changed-features/eta-expansion-spec.html` |
| Error code lookup (E001 etc.) | `nightly.scala-lang.org/docs/reference/error-codes/EXXX.html` |

---

## Citation format reminder

Every finding's citation field should contain:

1. A short verbatim quote (≤ 25 words) from one of these sources.
2. The URL.

Format example:

> "Opaque type aliases provide type abstraction without any overhead. In Scala 2, a similar result could be achieved with value classes." — `docs.scala-lang.org/scala3/book/types-opaque-types.html`

Do not paraphrase and present as a quote. Either quote verbatim or cite without quote marks. **Never invent a URL.** If you cannot find a primary source for a claim, write `uncited` and explain why the claim is still defensible (e.g., "follows from JVM semantics, not Scala-specific").
