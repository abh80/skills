# scalajs

An agent skill for building, scaffolding, and debugging Scala.js applications — browser apps, Node.js targets, and cross-platform libraries.

## Install

```bash
npx skills add https://github.com/abh80/skills --skill scalajs
```

Or install for a specific agent:

```bash
npx skills add https://github.com/abh80/skills --skill scalajs -a claude-code
```

## What's included

**`scalajs`** — Covers:
- Project setup with sbt and Mill (sbt-scalajs, CrossJSModule)
- Linker modes (`fastLinkJS`, `fullLinkJS`) and module kinds (ESModule, CommonJS, NoModule)
- Vite + Scala.js integration via `@scala-js/vite-plugin-scalajs`
- UI stacks — Laminar, Tyrian, scalajs-react, Slinky
- JS interop and façade types for npm libraries
- ScalablyTyped (npm → Scala types)
- Cross-building JVM + JS with sbt-crossproject
- Testing on Scala.js targets
- WebAssembly output
- Common linker / interop errors and fixes

## Skill structure

```
skills/scalajs/
├── SKILL.md                    # Main instructions
└── references/
    ├── sbt-build.md            # sbt configuration
    ├── mill-build.md           # Mill configuration
    ├── laminar.md              # Laminar UI patterns
    ├── interop.md              # JS interop / façades
    ├── scalablytyped.md        # npm → Scala types
    ├── cross-build.md          # JVM + JS cross-building
    ├── testing.md              # Testing on Scala.js
    ├── wasm.md                 # WebAssembly output
    └── errors.md               # Common errors
```

## Compatibility

This skill follows the [Agent Skills specification](https://agentskills.io) and works with Claude Code, Cursor, Codex, Gemini CLI, GitHub Copilot, and 30+ other agents.

## License

MIT
