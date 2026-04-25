# pytorch-xpu-skill

An agent skill for setting up and using PyTorch on Intel GPUs (XPU).

## Install

```bash
npx skills add https://github.com/abh80/skills --skill pytorch-xpu-skill
```

Or install for a specific agent:

```bash
npx skills add https://github.com/abh80/skills --skill pytorch-xpu-skill -a claude-code
```

## What's included

**`pytorch-xpu-skill`** — Covers:
- Installing PyTorch with XPU support (pip wheels, no conda packages)
- Supported hardware (Arc A/B-Series, Core Ultra, Data Center GPU Max)
- Driver & prerequisite skill for Windows and Linux
- `torch.xpu` API reference
- CUDA → XPU migration (it's mostly find-and-replace)
- AMP, torch.compile, profiling, multi-device patterns
- Common troubleshooting (missing drivers, wrong wheel, DLL errors)

## Skill structure

```
skills/pytorch-xpu-skill/
├── SKILL.md                        # Main instructions
└── references/
    └── code-patterns.md            # Detailed code examples
```

## Compatibility

This skill follows the [Agent Skills specification](https://agentskills.io) and works with Claude Code, Cursor, Codex, Gemini CLI, GitHub Copilot, and 30+ other agents.

## License

MIT
