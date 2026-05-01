# skills

Collection of agent skills for Claude Code, Cursor, Codex, and other agents following the [Agent Skills specification](https://agentskills.io).

## Skills

| Skill | Description |
|-------|-------------|
| [`pytorch-xpu-skill`](./pytorch-xpu-skill) | Install, configure, and use PyTorch on Intel GPUs (XPU) — Arc A/B-Series, Core Ultra, Data Center GPU Max. CUDA → XPU migration, AMP, `torch.compile`, profiling. |
| [`scala-code-optimizer`](./scala-code-optimizer) | Audit Scala code for refactoring opportunities — Scala 3 modernization, idiom adoption, anti-pattern removal, performance hygiene, tail recursion safety. Citation-backed suggestions only. |
| [`scalajs`](./scalajs) | Build, scaffold, and debug Scala.js applications — sbt/Mill setup, Laminar/Tyrian/scalajs-react UIs, JS interop, ScalablyTyped, cross-building JVM+JS, WebAssembly output. |

## Install

Install any skill by name:

```bash
npx skills add https://github.com/abh80/skills --skill <skill-name>
```

For a specific agent:

```bash
npx skills add https://github.com/abh80/skills --skill <skill-name> -a claude-code
```

## Compatibility

Works with Claude Code, Cursor, Codex, Gemini CLI, GitHub Copilot, and 30+ other agents.

## License

MIT
