---
name: pytorch-xpu-setup
description: >
  Guide for installing, configuring, and using PyTorch with Intel XPU (GPU) support.
  Use this skill whenever the user mentions: PyTorch with Intel GPU, XPU, torch.xpu,
  Intel Arc GPU for deep learning, Intel discrete GPU with PyTorch, installing PyTorch
  for Intel graphics, XPU device in PyTorch, migrating CUDA code to XPU, Intel oneAPI
  with PyTorch, or any task involving running PyTorch workloads on Intel GPUs
  (Arc A-series, B-series, Meteor Lake, Arrow Lake, Lunar Lake, Panther Lake,
  Data Center GPU Max). Also trigger when the user asks about mixed precision on XPU,
  torch.compile on XPU, or training/inference on Intel GPUs. Even if they just say
  "Intel GPU" and "PyTorch" in the same breath, use this skill.
---

# PyTorch XPU Setup & Usage

This skill covers end-to-end setup and usage of PyTorch on Intel GPUs (XPU) on **Windows** and **Linux**.

> **Important Context (as of 2026):**
> - Intel Extension for PyTorch (IPEX) reached **end of life in March 2026**. All XPU support is now **native in upstream PyTorch**.
> - There are **no conda packages** for XPU. Only pip wheels exist from the XPU index URL.
> - The recommended workflow: create a conda env → install via pip inside it.

---

## Quick Reference — Installation

### Prerequisites

1. **Intel GPU Driver** — required for all users.
   - **Windows:** Download from [Intel Arc & Iris Xe Graphics - Windows](https://www.intel.com/content/www/us/en/download/785597/intel-arc-iris-xe-graphics-windows.html). Include **LevelZeroSDK** in the install for `torch.compile` support.
   - **Linux:** Follow [Client GPU installation instructions](https://dgpu-docs.intel.com/driver/client/overview.html#ubuntu-latest).
   - Full prerequisites page: https://www.intel.com/content/www/us/en/developer/articles/tool/pytorch-prerequisites-for-intel-gpu/2-11.html

2. **Intel Deep Learning Essentials** — only needed if **building from source**. If installing from pip wheels, the runtime packages are bundled automatically.

### Supported Hardware

| Platform | Supported GPUs |
|----------|---------------|
| Windows 11 / Ubuntu 24.04+ | Intel Arc A-Series (Alchemist), Arc B-Series (Battlemage), Core Ultra with Arc Graphics (Meteor Lake-H, Arrow Lake-H, Lunar Lake) |
| Windows 11 / Ubuntu 25.10 | Panther Lake |
| Linux (RHEL 9.2, SUSE 15 SP5, Ubuntu 22.04+) | Intel Data Center GPU Max Series (Ponte Vecchio) |

### Installation Steps

```bash
# 1. Create conda environment
conda create -n pytorch-xpu python=3.11 -y
conda activate pytorch-xpu

# 2. Install PyTorch with XPU support (stable release)
pip3 install torch torchvision torchaudio --index-url https://download.pytorch.org/whl/xpu

# --- OR for nightly builds ---
pip3 install --pre torch torchvision torchaudio --index-url https://download.pytorch.org/whl/nightly/xpu

# --- OR for a specific version (e.g. 2.10.0) ---
pip install torch==2.10.0 torchvision==0.25.0 torchaudio==2.10.0 --index-url https://download.pytorch.org/whl/xpu

# 3. Verify
python -c "import torch; print(torch.xpu.is_available()); print(torch.xpu.get_device_name(0))"
```

> **Python version note:** Python 3.10–3.12 are well-tested. Python 3.13 has reported issues with some XPU nightly builds.

### Troubleshooting Installation

| Symptom | Fix |
|---------|-----|
| `torch.xpu.is_available()` returns `False` | Update Intel GPU driver to latest version. Verify with `sycl-ls` (Linux) or Device Manager (Windows). |
| `AssertionError: Torch not compiled with XPU enabled` | You installed a non-XPU wheel. Reinstall with `--index-url https://download.pytorch.org/whl/xpu`. |
| Import errors about missing DLLs on Windows | Ensure LevelZeroSDK was included during driver install. |
| `torch.compile` fails on Windows | Requires PyTorch ≥ 2.7 + MSVC (Desktop Development with C++). |

---

## Quick Reference — Code Usage

The `torch.xpu` API mirrors `torch.cuda` almost 1:1. Migrating from CUDA is a find-and-replace of `"cuda"` → `"xpu"`.

For **detailed code patterns** (inference, training, AMP, torch.compile, memory management, multi-device, profiling), read the reference file:

→ **`references/code-patterns.md`** — Comprehensive code examples for all XPU workflows.

### Minimal Example

```python
import torch

device = torch.device("xpu" if torch.xpu.is_available() else "cpu")

model = MyModel().to(device)
data = torch.randn(32, 3, 224, 224).to(device)

with torch.no_grad():
    output = model(data)
```

### Key API Surface

| Function | Purpose |
|----------|---------|
| `torch.xpu.is_available()` | Check if XPU device is available |
| `torch.xpu.device_count()` | Number of Intel GPU devices |
| `torch.xpu.current_device()` | Index of selected device |
| `torch.xpu.get_device_name(i)` | Device name string |
| `torch.xpu.synchronize()` | Wait for all kernels to finish |
| `torch.xpu.empty_cache()` | Free unused cached memory |
| `torch.xpu.memory_allocated()` | Current memory usage (bytes) |
| `torch.xpu.max_memory_allocated()` | Peak memory usage |
| `torch.xpu.memory_stats()` | Dict of allocator statistics |
| `torch.xpu.Stream()` | Create a compute stream |
| `torch.xpu.Event()` | Create a timing event |
| `torch.autocast(device_type="xpu")` | Enable AMP |
| `torch.amp.GradScaler(device="xpu")` | Gradient scaling for AMP training |

### Important Caveats

1. **GradScaler & FP64:** `GradScaler` requires FP64 hardware support. Intel Arc A-Series does NOT support FP64 natively. On Arc A-series, use `autocast` without `GradScaler`.
2. **torch.compile on Windows:** Supported from PyTorch 2.7+. Requires MSVC (Visual Studio 2022, Desktop Development with C++ workload).
3. **No IPEX needed:** Do NOT install `intel-extension-for-pytorch` for new projects. It is EOL. Use native `torch.xpu` directly.
4. **No conda XPU packages:** Always use pip with `--index-url https://download.pytorch.org/whl/xpu` inside a conda env.

---

## Official Documentation Links

| Resource | URL |
|----------|-----|
| `torch.xpu` API Reference | https://docs.pytorch.org/docs/stable/xpu.html |
| Getting Started on Intel GPU | https://docs.pytorch.org/docs/stable/notes/get_start_xpu.html |
| torch.compile on Windows XPU | https://docs.pytorch.org/tutorials/unstable/inductor_windows.html |
| Prerequisites for Intel GPUs | https://www.intel.com/content/www/us/en/developer/articles/tool/pytorch-prerequisites-for-intel-gpu/2-11.html |
| Intel GPU Driver (Windows) | https://www.intel.com/content/www/us/en/download/785597/intel-arc-iris-xe-graphics-windows.html |
| PyTorch Previous Versions | https://pytorch.org/get-started/previous-versions/ |
