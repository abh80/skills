# pytorch-xpu-setup

An agent skill for setting up and using PyTorch on Intel GPUs (XPU).

## Install

```bash
npx skills add YOUR_GITHUB_USERNAME/pytorch-xpu-setup
```

Or install for a specific agent:

```bash
npx skills add YOUR_GITHUB_USERNAME/pytorch-xpu-setup -a claude-code
```

## What's included

**`pytorch-xpu-setup`** — Covers:
- Installing PyTorch with XPU support (pip wheels, no conda packages)
- Supported hardware (Arc A/B-Series, Core Ultra, Data Center GPU Max)
- Driver & prerequisite setup for Windows and Linux
- `torch.xpu` API reference
- CUDA → XPU migration (it's mostly find-and-replace)
- AMP, torch.compile, profiling, multi-device patterns
- Common troubleshooting (missing drivers, wrong wheel, DLL errors)

## Skill structure

```
skills/pytorch-xpu-setup/
├── SKILL.md                        # Main instructions
└── references/
    └── code-patterns.md            # Detailed code examples
```

## Compatibility

This skill follows the [Agent Skills specification](https://agentskills.io) and works with Claude Code, Cursor, Codex, Gemini CLI, GitHub Copilot, and 30+ other agents.

## License

MIT
