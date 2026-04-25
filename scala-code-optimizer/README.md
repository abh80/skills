# scala-code-optimizer

An agent skill for auditing Scala code — Scala 3 modernization, idiom adoption, anti-pattern removal, and performance hygiene.

## Install

```bash
npx skills add https://github.com/abh80/skills --skill scala-code-optimizer
```

Or install for a specific agent:

```bash
npx skills add https://github.com/abh80/skills --skill scala-code-optimizer -a claude-code
```

## What's included

**`scala-code-optimizer`** — Covers:
- Scala 3 idiom adoption (`enum`, `given`/`using`, `extension`, `opaque type`, `derives`, `export`, union/intersection types, `inline`)
- Scala 2 → Scala 3 migration cleanup (dropped/deprecated features, implicit → given/using)
- Pattern-matching anti-patterns (`isInstanceOf`/`asInstanceOf`, non-exhaustive matches, `Option.get` smell)
- `Option`/`Try`/`Either`/`Future` hygiene
- Collection performance & API selection (boxing, `Seq` boundaries, `List` vs `Vector`, fused ops)
- Tail recursion safety (`@tailrec`, accumulator pattern, trampolines)
- `lazy val` pitfalls, implicit/given hygiene, trait & class design
- Weakly-typed APIs, concurrency & blocking issues
- Citation-backed findings from `docs.scala-lang.org` — suggestions only, never edits in place

## Skill structure

```
skills/scala-code-optimizer/
├── SKILL.md                    # Main instructions
└── references/
    ├── idioms.md               # Scala 3 idiom catalog
    ├── anti-patterns.md        # Common anti-patterns
    ├── performance.md          # Perf, recursion, lazy val
    └── citations.md            # Authoritative doc URLs
```

## Compatibility

This skill follows the [Agent Skills specification](https://agentskills.io) and works with Claude Code, Cursor, Codex, Gemini CLI, GitHub Copilot, and 30+ other agents.

## License

MIT
