# scala-fp

An agent skill for functional programming in Scala — Cats, Cats Effect, ZIO 2, fs2, and the Typelevel/ZIO ecosystem.

## Install

```bash
npx skills add https://github.com/abh80/skills --skill scala-fp
```

Or install for a specific agent:

```bash
npx skills add https://github.com/abh80/skills --skill scala-fp -a claude-code
```

## What's included

**`scala-fp`** — Covers:
- Cats typeclasses (Functor, Applicative, Monad, Traverse, monad transformers)
- Cats Effect 3 — `IO`, fibers, `Ref`, `Semaphore`, `Queue`, `Deferred`, `Resource`/bracket
- ZIO 2 — `ZIO[R, E, A]`, typed errors, `ZLayer` dependency injection
- Parallel programming (`parMapN`, `parTraverse`, structured concurrency)
- fs2 streams and ZIO Streams
- Tagless Final architecture
- Error handling in FP
- Ecosystem libraries (http4s, doobie, skunk, zio-http, quill)

## Skill structure

```
skills/scala-fp/
├── SKILL.md                      # Main instructions
└── references/
    ├── cats-typeclasses.md       # Functor, Monad, Applicative, Traverse
    ├── cats-effect.md            # IO + concurrency primitives
    ├── zio.md                    # ZIO 2 core + ZLayer
    ├── parallelism.md            # Parallel patterns
    ├── fs2.md                    # fs2 streams
    ├── tagless-final.md          # Tagless final architecture
    ├── error-handling.md         # FP error handling
    └── ecosystem.md              # Library versions & ecosystem
```

## Compatibility

This skill follows the [Agent Skills specification](https://agentskills.io) and works with Claude Code, Cursor, Codex, Gemini CLI, GitHub Copilot, and 30+ other agents.

## License

MIT
