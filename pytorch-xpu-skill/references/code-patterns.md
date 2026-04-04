# PyTorch XPU Code Patterns Reference

Comprehensive code examples for all common XPU workflows.

## Table of Contents
1. [Device Setup & Detection](#1-device-setup--detection)
2. [Inference Patterns](#2-inference-patterns)
3. [Training Patterns](#3-training-patterns)
4. [Mixed Precision (AMP)](#4-mixed-precision-amp)
5. [torch.compile](#5-torchcompile)
6. [Memory Management](#6-memory-management)
7. [Streams & Events](#7-streams--events)
8. [Multi-Device](#8-multi-device)
9. [Benchmarking & Profiling](#9-benchmarking--profiling)
10. [CUDA to XPU Migration Checklist](#10-cuda-to-xpu-migration-checklist)
11. [Device-Agnostic Code Pattern](#11-device-agnostic-code-pattern)
12. [Common Pitfalls](#12-common-pitfalls)

---

## 1. Device Setup & Detection

```python
import torch

# Basic availability check
print(torch.xpu.is_available())        # True/False
print(torch.xpu.device_count())        # Number of Intel GPUs
print(torch.xpu.current_device())      # Current device index
print(torch.xpu.get_device_name(0))    # e.g. "Intel Arc A770"

# Set default device
torch.xpu.set_device(0)

# Device object for .to() calls
device = torch.device("xpu" if torch.xpu.is_available() else "cpu")

# Context manager for device selection (multi-GPU)
with torch.xpu.device(1):
    tensor = torch.randn(3, 3)  # allocated on device 1
```

---

## 2. Inference Patterns

### Basic FP32 Inference

```python
import torch
import torchvision.models as models

model = models.resnet50(weights="ResNet50_Weights.DEFAULT")
model.eval()
model = model.to("xpu")

data = torch.rand(1, 3, 224, 224).to("xpu")

with torch.no_grad():
    output = model(data)

print("Inference complete. Output shape:", output.shape)
```

### Batch Inference

```python
model.eval()
model = model.to("xpu")

results = []
for batch in dataloader:
    inputs = batch.to("xpu")
    with torch.no_grad():
        output = model(inputs)
    results.append(output.cpu())  # move back to CPU for post-processing

all_results = torch.cat(results)
```

---

## 3. Training Patterns

### Standard FP32 Training Loop

```python
import torch
import torchvision

LR = 0.001
device = torch.device("xpu")

# Dataset & loader
transform = torchvision.transforms.Compose([
    torchvision.transforms.Resize((224, 224)),
    torchvision.transforms.ToTensor(),
    torchvision.transforms.Normalize((0.5, 0.5, 0.5), (0.5, 0.5, 0.5)),
])
train_dataset = torchvision.datasets.CIFAR10(
    root="./data", train=True, transform=transform, download=True
)
train_loader = torch.utils.data.DataLoader(train_dataset, batch_size=128)

# Model, loss, optimizer
model = torchvision.models.resnet50().to(device)
criterion = torch.nn.CrossEntropyLoss().to(device)
optimizer = torch.optim.SGD(model.parameters(), lr=LR, momentum=0.9)

# Training loop
model.train()
for batch_idx, (data, target) in enumerate(train_loader):
    data, target = data.to(device), target.to(device)
    optimizer.zero_grad()
    output = model(data)
    loss = criterion(output, target)
    loss.backward()
    optimizer.step()

    if (batch_idx + 1) % 10 == 0:
        print(f"Batch [{batch_idx+1}/{len(train_loader)}], Loss: {loss.item():.4f}")

# Save checkpoint
torch.save({
    "model_state_dict": model.state_dict(),
    "optimizer_state_dict": optimizer.state_dict(),
}, "checkpoint.pth")
```

### Loading Checkpoints on XPU

```python
checkpoint = torch.load("checkpoint.pth", map_location="xpu")
model.load_state_dict(checkpoint["model_state_dict"])
optimizer.load_state_dict(checkpoint["optimizer_state_dict"])
```

---

## 4. Mixed Precision (AMP)

### Inference with AMP

```python
model.eval()
model = model.to("xpu")
data = data.to("xpu")

with torch.no_grad():
    # FP16
    with torch.autocast(device_type="xpu", dtype=torch.float16, enabled=True):
        output = model(data)

    # OR BF16
    with torch.autocast(device_type="xpu", dtype=torch.bfloat16, enabled=True):
        output = model(data)
```

### Training with AMP + GradScaler

> **Warning:** GradScaler requires FP64 support. Intel Arc A-Series does NOT
> support FP64. Use the "without GradScaler" pattern below for Arc A-series.

```python
# WITH GradScaler (Data Center GPU Max, Arc B-Series, etc.)
use_amp = True
scaler = torch.amp.GradScaler(device="xpu", enabled=use_amp)

model.train()
model = model.to("xpu")

for data, target in train_loader:
    data, target = data.to("xpu"), target.to("xpu")

    with torch.autocast(device_type="xpu", dtype=torch.float16, enabled=use_amp):
        output = model(data)
        loss = criterion(output, target)

    scaler.scale(loss).backward()
    scaler.step(optimizer)
    scaler.update()
    optimizer.zero_grad()
```

```python
# WITHOUT GradScaler (Arc A-Series — no FP64)
model.train()
model = model.to("xpu")

for data, target in train_loader:
    data, target = data.to("xpu"), target.to("xpu")

    with torch.autocast(device_type="xpu", dtype=torch.float16):
        output = model(data)
        loss = criterion(output, target)

    loss.backward()
    optimizer.step()
    optimizer.zero_grad()
```

---

## 5. torch.compile

> Supported on Windows from PyTorch 2.7+. Requires MSVC installed.

### Inference with torch.compile

```python
import torch
import torchvision.models as models

model = models.resnet50(weights="ResNet50_Weights.DEFAULT")
model.eval()
model = model.to("xpu")

# Compile the model
model = torch.compile(model)

data = torch.rand(1, 3, 224, 224).to("xpu")

# First call triggers compilation (slow); subsequent calls are fast
with torch.no_grad():
    output = model(data)
```

### Training with torch.compile

```python
model = torchvision.models.resnet50().to("xpu")
criterion = torch.nn.CrossEntropyLoss().to("xpu")
optimizer = torch.optim.SGD(model.parameters(), lr=0.001, momentum=0.9)

model = torch.compile(model)
model.train()

for data, target in train_loader:
    data, target = data.to("xpu"), target.to("xpu")
    optimizer.zero_grad()
    output = model(data)
    loss = criterion(output, target)
    loss.backward()
    optimizer.step()
```

### Windows torch.compile Setup

Before using torch.compile on Windows, ensure MSVC is configured:

```cmd
:: In cmd.exe, activate MSVC environment first
"C:\Program Files\Microsoft Visual Studio\2022\Community\VC\Auxiliary\Build\vcvarsall.bat" x64

:: Then run your Python script
python my_training_script.py
```

---

## 6. Memory Management

```python
# Check current memory usage
print(f"Allocated: {torch.xpu.memory_allocated() / 1e6:.1f} MB")
print(f"Peak:      {torch.xpu.max_memory_allocated() / 1e6:.1f} MB")

# Release unused cached memory
torch.xpu.empty_cache()

# Reset peak memory tracking
torch.xpu.reset_peak_memory_stats()

# Full memory stats dict
stats = torch.xpu.memory_stats()

# Memory pool management
pool = torch.xpu.MemPool()
with torch.xpu.use_mem_pool(pool):
    # allocations here go through this pool
    tensor = torch.randn(1000, 1000, device="xpu")
```

---

## 7. Streams & Events

```python
# Create and use streams
s1 = torch.xpu.Stream()
s2 = torch.xpu.Stream()

with torch.xpu.stream(s1):
    a = torch.randn(1000, 1000, device="xpu")
    b = a @ a

with torch.xpu.stream(s2):
    c = torch.randn(1000, 1000, device="xpu")
    d = c @ c

# Synchronize a specific stream
s1.synchronize()

# Events for timing
start_event = torch.xpu.Event(enable_timing=True)
end_event = torch.xpu.Event(enable_timing=True)

start_event.record()
# ... your operation ...
end_event.record()
torch.xpu.synchronize()

elapsed_ms = start_event.elapsed_time(end_event)
print(f"Elapsed: {elapsed_ms:.2f} ms")
```

---

## 8. Multi-Device

```python
# Check device count
n_devices = torch.xpu.device_count()
print(f"Found {n_devices} XPU device(s)")

# Place tensors on specific devices
t0 = torch.randn(100, device="xpu:0")
t1 = torch.randn(100, device="xpu:1")  # if 2 GPUs

# Move between devices
t1_on_0 = t1.to("xpu:0")

# DataParallel (basic)
if torch.xpu.device_count() > 1:
    model = torch.nn.DataParallel(model, device_ids=[0, 1])
    model = model.to("xpu:0")
```

---

## 9. Benchmarking & Profiling

### Correct Timing Pattern

Always synchronize before measuring wall-clock time:

```python
import time

model.eval()
model = model.to("xpu")
data = torch.rand(1, 3, 224, 224).to("xpu")

# Warm-up
for _ in range(5):
    with torch.no_grad():
        model(data)

# Benchmark
torch.xpu.synchronize()
start = time.time()

ITERS = 100
for _ in range(ITERS):
    with torch.no_grad():
        model(data)

torch.xpu.synchronize()
end = time.time()

avg_ms = (end - start) / ITERS * 1000
print(f"Average inference: {avg_ms:.2f} ms")
```

### Using XPU Events for Precise Timing

```python
start = torch.xpu.Event(enable_timing=True)
end = torch.xpu.Event(enable_timing=True)

start.record()
with torch.no_grad():
    for _ in range(ITERS):
        model(data)
end.record()

torch.xpu.synchronize()
total_ms = start.elapsed_time(end)
print(f"Average: {total_ms / ITERS:.2f} ms")
```

---

## 10. CUDA to XPU Migration Checklist

| CUDA Code | XPU Equivalent |
|-----------|---------------|
| `torch.cuda.is_available()` | `torch.xpu.is_available()` |
| `.to("cuda")` / `.cuda()` | `.to("xpu")` |
| `torch.cuda.synchronize()` | `torch.xpu.synchronize()` |
| `torch.cuda.empty_cache()` | `torch.xpu.empty_cache()` |
| `torch.cuda.Stream()` | `torch.xpu.Stream()` |
| `torch.cuda.Event()` | `torch.xpu.Event()` |
| `torch.cuda.memory_allocated()` | `torch.xpu.memory_allocated()` |
| `torch.cuda.device_count()` | `torch.xpu.device_count()` |
| `torch.cuda.current_device()` | `torch.xpu.current_device()` |
| `torch.cuda.set_device(i)` | `torch.xpu.set_device(i)` |
| `torch.cuda.get_device_name(i)` | `torch.xpu.get_device_name(i)` |
| `device_type="cuda"` in autocast | `device_type="xpu"` in autocast |
| `torch.amp.GradScaler(device="cuda")` | `torch.amp.GradScaler(device="xpu")` |
| `pin_memory=True` in DataLoader | Same — works with XPU |
| `CUDA_VISIBLE_DEVICES` env var | Not applicable — use `torch.xpu.set_device()` |

### Automated Migration

For large codebases, a simple regex replacement covers most cases:

```bash
# Linux/Mac
sed -i 's/\.cuda()/\.to("xpu")/g; s/"cuda"/"xpu"/g; s/torch\.cuda/torch.xpu/g' your_script.py

# Windows PowerShell
(Get-Content your_script.py) -replace '\.cuda\(\)', '.to("xpu")' -replace '"cuda"', '"xpu"' -replace 'torch\.cuda', 'torch.xpu' | Set-Content your_script.py
```

> **Review after auto-replacement.** Some edge cases (like string comparisons or logging) may need manual fixup.

---

## 11. Device-Agnostic Code Pattern

Write code that works on CUDA, XPU, MPS, and CPU:

```python
def get_device():
    if torch.xpu.is_available():
        return torch.device("xpu")
    elif torch.cuda.is_available():
        return torch.device("cuda")
    elif hasattr(torch.backends, "mps") and torch.backends.mps.is_available():
        return torch.device("mps")
    return torch.device("cpu")

def sync_device(device):
    """Synchronize the appropriate accelerator."""
    if device.type == "xpu":
        torch.xpu.synchronize()
    elif device.type == "cuda":
        torch.cuda.synchronize()
    # MPS and CPU don't need explicit sync

device = get_device()
model = model.to(device)
data = data.to(device)
```

---

## 12. Common Pitfalls

### Pitfall 1: Forgetting to synchronize before timing
XPU operations are asynchronous. Without `torch.xpu.synchronize()`, time measurements will be inaccurate.

### Pitfall 2: Using GradScaler on Arc A-Series
Arc A-Series GPUs lack FP64 support. GradScaler will fail. Use autocast alone.

### Pitfall 3: Installing IPEX alongside native PyTorch XPU
Don't install `intel-extension-for-pytorch` with PyTorch ≥ 2.5 XPU wheels. They conflict. Use native `torch.xpu` only.

### Pitfall 4: Using conda install for XPU
There are no conda packages for XPU. Always use `pip install ... --index-url https://download.pytorch.org/whl/xpu` inside your conda env.

### Pitfall 5: Missing MSVC for torch.compile on Windows
`torch.compile` on Windows XPU requires MSVC. Install Visual Studio 2022 with "Desktop Development with C++" workload, and activate the MSVC environment before running your script.

### Pitfall 6: Python 3.13 compatibility
Some XPU nightly builds have reported issues on Python 3.13. Stick to Python 3.10–3.12 for reliability.
