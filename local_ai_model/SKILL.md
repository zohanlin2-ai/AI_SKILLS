# SKILL: Local AI Model Management

## 1. Overview

Running AI models locally is fundamentally different from calling a cloud API — you own the hardware constraints, and every decision around precision, memory layout, and threading has immediate, observable consequences. The biggest differentiators are VRAM limits (a cloud API never says "CUDA out of memory"), cold-start latency (loading a 4 GB model from disk takes 5–30 seconds), and the need to manage the full model lifecycle yourself (load, infer, unload). Precision tradeoffs that are invisible in cloud APIs become critical locally: fp32 vs fp16 vs int4 changes both VRAM usage and output quality in ways you must consciously choose. This skill captures real pitfalls encountered while building local image generators (Diffusers + GPU pipeline) and local LLM setups, so future decisions start from experience rather than trial-and-error.

---

## 2. Tool & Framework Selection

> [!TIP]
> Choose based on the **task requirements and hardware**, not familiarity with the framework. The "easiest to start" option is often the hardest to tune for production-quality output.

| Framework | Best For | Tradeoffs | Minimum VRAM |
|---|---|---|---|
| **Diffusers** (HuggingFace) | Image generation, fine-tuned model support (LoRA, ControlNet), SD/SDXL/Flux pipelines | Heavy dependencies (~8 GB install with CUDA), complex pipeline API with many moving parts | 4 GB (fp16, SD 1.5); 8–12 GB (SDXL) |
| **llama.cpp / GGUF** | Local LLM inference, CPU+GPU hybrid, very low VRAM via quantization | Output quality varies significantly with quantization level; less Pythonic API | 2 GB (Q4 7B); 6 GB (Q8 7B) |
| **Ollama** | Easiest local LLM setup, REST API similar to OpenAI, model management CLI | Less control over precision/sampling parameters; black-box VRAM management | 4 GB (depends on model) |
| **Transformers** (HuggingFace) | NLP tasks, text generation, custom model research | Requires manual VRAM management; auto-loading defaults to fp32 silently | 8 GB (7B fp16); 4 GB (7B int8) |

### Selection Rationale

- **Diffusers** is the right choice when you need fine-grained control over image generation — LoRA loading, ControlNet conditioning, scheduler swapping. The API complexity is worth it for the ecosystem breadth.
- **llama.cpp / GGUF** is preferred when VRAM is tight and you're willing to accept some quality loss for a 4× memory reduction. A Q4_K_M 7B model runs in 4 GB VRAM with acceptable quality for chat tasks.
- **Ollama** is best for rapid prototyping or when you want a simple REST interface without managing Python dependencies. Don't use it when you need per-request control over temperature, top-p, or quantization level.
- **Transformers** directly is justified for research tasks or when you need a model that isn't packaged into a specialized framework. Always explicitly set `torch_dtype=torch.float16` — the default fp32 will silently consume 2× the VRAM.

---

## 3. VRAM Management Strategy

VRAM is the primary bottleneck in all local AI workloads. Unlike system RAM, VRAM fragmentation is hard to recover from without a full unload, and silent overcommit leads to crashes rather than graceful degradation.

### Before Loading: Check Available VRAM

Always verify available VRAM before loading a model. This prevents cryptic crashes mid-load:

```python
import torch

def check_vram():
    if not torch.cuda.is_available():
        return 0
    props = torch.cuda.get_device_properties(0)
    total_mb = props.total_memory / 1024**2
    allocated_mb = torch.cuda.memory_allocated(0) / 1024**2
    free_mb = total_mb - allocated_mb
    print(f"VRAM: {free_mb:.0f} MB free / {total_mb:.0f} MB total")
    return free_mb

# Usage: gate model loading behind a VRAM check
MIN_VRAM_MB = 4096  # e.g., require 4 GB free
if check_vram() < MIN_VRAM_MB:
    raise RuntimeError("Insufficient VRAM — unload other models first.")
```

### CPU Offload for VRAM-Constrained Systems

When VRAM is tight, model CPU offload moves layers to RAM while keeping active tensors on GPU. It's slower but prevents OOM:

```python
# Diffusers: enable sequential CPU offload
pipe.enable_model_cpu_offload()

# OR for even more aggressive offloading (Diffusers ≥ 0.20):
pipe.enable_sequential_cpu_offload()
```

> [!NOTE]
> `enable_model_cpu_offload()` keeps the entire model in GPU VRAM at once but offloads between pipeline stages. `enable_sequential_cpu_offload()` offloads layer by layer — slower but works on 4 GB cards for SDXL.

### Explicit Cleanup: The Correct Order Matters

```python
# ✅ Correct cleanup sequence
del pipe          # Remove Python reference first
torch.cuda.empty_cache()  # Now the allocator can reclaim

# ❌ Wrong — empty_cache() does nothing if references still exist
torch.cuda.empty_cache()
del pipe  # Too late; memory wasn't freed during empty_cache()
```

> [!CAUTION]
> Calling `torch.cuda.empty_cache()` without first deleting all references to model tensors is a no-op. The CUDA allocator only releases memory when Python's garbage collector has removed the reference, which `del` triggers explicitly.

### Multi-Model Workflow Pattern

For workflows that switch between models (e.g., upscaler → generator → refiner):

```python
# Preference: load → infer → unload → load next
# Never hold two large models in VRAM simultaneously unless you have 24+ GB

def run_with_model(model_loader_fn, inputs):
    model = model_loader_fn()
    try:
        result = model(inputs)
        return result
    finally:
        del model
        torch.cuda.empty_cache()
```

**Why this matters**: Loading two models "to be ready" and switching between them seems efficient but typically causes OOM on consumer GPUs. The sequential pattern is slower per-switch but far more stable.

---

## 4. Model Loading Patterns

### Startup Load vs On-Demand Load

| Strategy | Latency | Memory | Best When |
|---|---|---|---|
| **Load at startup** | First request is instant | VRAM held even when idle | Interactive apps, frequent use |
| **Load on first request** | First request is slow (5–30s) | VRAM free until needed | Batch tools, infrequent use |
| **Load per request** | Every request is slow | VRAM freed between uses | Very limited VRAM, rare use |

**Personal preference**: For interactive GUI applications (Qt, Tkinter, etc.), load at startup with a visible loading indicator. Users accept a one-time wait but are frustrated by unexpected 20-second freezes on their first click.

### Singleton Pattern for Model Loaders

Loading the same model twice (e.g., across multiple function calls) pays full disk I/O cost each time. A singleton prevents this:

```python
import torch
from diffusers import StableDiffusionPipeline
from threading import Lock

class ModelManager:
    _instance = None
    _lock = Lock()

    def __init__(self, model_id: str, device: str = "cuda"):
        self.model_id = model_id
        self.device = device
        self._pipe = None

    def get_pipeline(self):
        """Load once, reuse across all requests."""
        if self._pipe is None:
            with self._lock:  # Thread-safe initialization
                if self._pipe is None:  # Double-checked locking
                    print(f"Loading model: {self.model_id}")
                    self._pipe = StableDiffusionPipeline.from_pretrained(
                        self.model_id,
                        torch_dtype=torch.float16,
                        safety_checker=None,
                    ).to(self.device)
                    # Keep VAE in fp32 to prevent NaN artifacts
                    self._pipe.vae = self._pipe.vae.to(torch.float32)
        return self._pipe

    def unload(self):
        """Explicit unload when switching models."""
        del self._pipe
        self._pipe = None
        torch.cuda.empty_cache()

# Usage
manager = ModelManager("runwayml/stable-diffusion-v1-5")
pipe = manager.get_pipeline()  # Loads from disk first time
pipe = manager.get_pipeline()  # Reuses cached instance
```

> [!CAUTION]
> Never load a model inside a loop. Every call to `from_pretrained()` performs disk I/O and CUDA tensor allocation — even if the model was previously loaded. Pre-check with the singleton pattern.

```python
# ❌ Extremely costly: loads model N times
for prompt in prompts:
    pipe = StableDiffusionPipeline.from_pretrained(model_id).to("cuda")
    result = pipe(prompt)

# ✅ Load once, reuse
pipe = StableDiffusionPipeline.from_pretrained(model_id).to("cuda")
for prompt in prompts:
    result = pipe(prompt)
```

---

## 5. Precision & Quality Tradeoffs

Precision selection is one of the highest-leverage decisions in local AI model management — it directly controls VRAM consumption and output quality simultaneously.

| Precision | Memory Use (relative) | Quality | Use When |
|---|---|---|---|
| **fp32** | ~4× fp16 | Reference quality | Debugging only — never for production |
| **fp16** | Baseline | Near-lossless | Most cases; default choice on CUDA |
| **bf16** | Same as fp16 | Better numerical stability for LLMs | Ampere+ GPUs (RTX 30xx / 40xx, A100) |
| **int8** | ~0.5× fp16 | Slight quality loss (usually imperceptible) | VRAM-constrained systems |
| **int4 / GGUF** | ~0.25× fp16 | Noticeable quality loss at extreme settings | Very limited VRAM (4 GB or less) |

### How to Specify Precision in Code

```python
import torch
from diffusers import StableDiffusionPipeline
from transformers import AutoModelForCausalLM

# Diffusers: fp16 (most common)
pipe = StableDiffusionPipeline.from_pretrained(model_id, torch_dtype=torch.float16)

# Transformers: bf16 for LLMs on Ampere+ GPU
model = AutoModelForCausalLM.from_pretrained(model_id, torch_dtype=torch.bfloat16)

# Transformers: int8 via bitsandbytes
model = AutoModelForCausalLM.from_pretrained(model_id, load_in_8bit=True, device_map="auto")

# Transformers: int4 via bitsandbytes
from transformers import BitsAndBytesConfig
bnb_config = BitsAndBytesConfig(load_in_4bit=True, bnb_4bit_compute_dtype=torch.float16)
model = AutoModelForCausalLM.from_pretrained(model_id, quantization_config=bnb_config)
```

### VAE Precision: The Silent Image Corruption Pitfall

> [!CAUTION]
> Casting the VAE (Variational Autoencoder) to fp16 causes **NaN (Not a Number) propagation** during decoding on many Stable Diffusion models. The result is a black, white, or corrupted image with no error message — the model "succeeds" but produces garbage.

**The fix**: Always keep VAE in fp32 even when running the UNet in fp16:

```python
pipe = StableDiffusionPipeline.from_pretrained(
    model_id,
    torch_dtype=torch.float16,  # UNet, text encoder in fp16
).to("cuda")

# Critical: override VAE to fp32 after moving to device
pipe.vae = pipe.vae.to(torch.float32)  # Prevent NaN artifacts in image decoding
```

This adds a small VRAM overhead (VAE is ~330 MB at fp32 vs ~165 MB at fp16) but prevents the most common source of corrupted output. The tradeoff is clearly worth it in all practical scenarios.

**Why it happens**: fp16 has limited dynamic range. The VAE decoder involves operations (exponentiation, normalization) where intermediate values can overflow or underflow to NaN, which then propagates through the entire output tensor.

---

## 6. Long-Running Inference Stability

Single-run inference is relatively easy to get right. Long-running applications (serving many requests, interactive tools) accumulate problems that only appear over time.

### Memory Leak Detection

Monitor VRAM usage after each inference. A healthy system should return close to pre-inference VRAM levels:

```python
import torch

def log_vram(tag: str = ""):
    allocated = torch.cuda.memory_allocated() / 1024**2
    reserved = torch.cuda.memory_reserved() / 1024**2
    print(f"[VRAM {tag}] Allocated: {allocated:.1f} MB | Reserved: {reserved:.1f} MB")

log_vram("before inference")
result = pipe(prompt)
log_vram("after inference")
torch.cuda.empty_cache()
log_vram("after empty_cache")
```

If `Allocated` grows monotonically across runs, there is a memory leak — likely a tensor being accumulated somewhere (e.g., stored in a list, appended to a history buffer).

### Background Thread Pattern: Never Block the UI Thread

Inference can take 5–60 seconds. Running it on the UI thread freezes the entire application (the window becomes unresponsive, OS may flag it as "not responding").

```python
# PyQt5 / PySide6 pattern using QThread
from PyQt5.QtCore import QThread, pyqtSignal

class InferenceThread(QThread):
    result_ready = pyqtSignal(object)   # Emits result when done
    error_occurred = pyqtSignal(str)    # Emits error message on failure

    def __init__(self, pipe, prompt: str):
        super().__init__()
        self.pipe = pipe
        self.prompt = prompt

    def run(self):
        try:
            image = self.pipe(self.prompt).images[0]
            self.result_ready.emit(image)
        except RuntimeError as e:
            if "CUDA out of memory" in str(e):
                torch.cuda.empty_cache()
                self.error_occurred.emit("Out of VRAM — try a smaller image size or free memory.")
            else:
                self.error_occurred.emit(f"Inference failed: {e}")

# In main window:
self.thread = InferenceThread(pipe, prompt)
self.thread.result_ready.connect(self.display_result)
self.thread.error_occurred.connect(self.show_error_dialog)
self.thread.start()
```

> [!IMPORTANT]
> Keep a reference to the `QThread` object (e.g., `self.thread`) — if it's garbage collected while running, the thread is terminated silently with no error. This is a subtle but common pitfall.

### CUDA Out-of-Memory: Graceful Recovery

```python
def safe_infer(pipe, prompt: str, **kwargs):
    try:
        return pipe(prompt, **kwargs).images[0]
    except RuntimeError as e:
        if "CUDA out of memory" in str(e):
            # Attempt recovery
            torch.cuda.empty_cache()
            # Optionally retry with smaller parameters
            try:
                return pipe(prompt, height=512, width=512, num_inference_steps=20).images[0]
            except RuntimeError:
                raise RuntimeError(
                    "VRAM exhausted. Unload other models and try again."
                ) from e
        raise  # Re-raise non-OOM errors
```

### Timeout Handling for CUDA Deadlocks

CUDA operations can deadlock (infinite wait) under rare conditions (driver bugs, interrupted ops). Use a timeout wrapper with threading:

```python
import threading

def run_with_timeout(fn, timeout_seconds=120, *args, **kwargs):
    result = [None]
    exception = [None]

    def target():
        try:
            result[0] = fn(*args, **kwargs)
        except Exception as e:
            exception[0] = e

    t = threading.Thread(target=target)
    t.daemon = True
    t.start()
    t.join(timeout=timeout_seconds)

    if t.is_alive():
        raise TimeoutError(f"Inference exceeded {timeout_seconds}s — possible CUDA deadlock.")
    if exception[0]:
        raise exception[0]
    return result[0]
```

### VRAM Fragmentation in Long Sessions

After dozens of inference runs, VRAM fragmentation can degrade performance or cause phantom OOM errors even when the allocator reports sufficient free memory. Periodic model process restarts (in a daemon/server architecture) are more reliable than trying to defragment in-process.

**Rule of thumb**: If inference latency degrades by >20% over a session, a process restart is cheaper than debugging fragmentation.

---

## 7. Common Pitfalls & Resolutions

| Problem | Root Cause | Resolution |
|---|---|---|
| CUDA out of memory on second run | Previous model was not unloaded — tensors still held in VRAM | `del pipe` then `torch.cuda.empty_cache()` between runs; verify with `torch.cuda.memory_allocated()` |
| Generated image is black or corrupted | VAE cast to fp16 produces NaN artifacts during decoding | Always set `pipe.vae = pipe.vae.to(torch.float32)` after loading, even if UNet is fp16 |
| Model loading hangs indefinitely | HuggingFace auto-download triggers a network fetch that stalls (firewall, rate limit, no internet) | Pre-download all models with `huggingface-cli download`; set `TRANSFORMERS_OFFLINE=1` in environment |
| Inference output degrades after many runs | Accumulating VRAM fragmentation over a long session | Periodically restart the model process; monitor allocated vs reserved VRAM ratio |
| App freezes / becomes unresponsive during inference | Inference running on the UI thread blocks the event loop | Move all inference to a `QThread` or `threading.Thread`; never call `pipe()` from main thread |
| Model outputs gibberish text after quantization | Quantization level too aggressive — int4 loses too much precision for the chosen model | Try int8 first (`load_in_8bit=True`); only drop to int4 if int8 still exceeds VRAM budget |
| `torch.cuda.empty_cache()` has no visible effect | Python still holds references to model tensors — allocator cannot reclaim | `del model`, `del pipe`, and any variables holding `.images[0]` references before calling `empty_cache()` |
| Model loads successfully but inference crashes immediately | `torch_dtype` mismatch between model and input tensors (e.g., fp32 input to fp16 model) | Ensure all inputs are cast to match the model's dtype: `tensor.to(torch.float16)` |
| LoRA weights cause visual artifacts or NaN | LoRA trained on a different base model version or merged at wrong scale | Match LoRA to exact base model checkpoint; adjust `lora_scale` (try 0.6–0.8 instead of 1.0) |

---

## 8. Quick Reference: Environment Setup

```bash
# Install with CUDA 11.8 support
pip install torch torchvision --index-url https://download.pytorch.org/whl/cu118

# Install Diffusers ecosystem
pip install diffusers transformers accelerate

# For quantization (int8/int4)
pip install bitsandbytes

# Offline mode: prevent unwanted network calls
export TRANSFORMERS_OFFLINE=1        # Linux/macOS
$env:TRANSFORMERS_OFFLINE = "1"      # Windows PowerShell

# Pre-download a model to local cache
huggingface-cli download runwayml/stable-diffusion-v1-5
```

```python
# Verify CUDA setup at startup
import torch
assert torch.cuda.is_available(), "CUDA not available — check driver and PyTorch install"
print(f"GPU: {torch.cuda.get_device_name(0)}")
print(f"VRAM: {torch.cuda.get_device_properties(0).total_memory / 1024**3:.1f} GB")
print(f"CUDA: {torch.version.cuda} | PyTorch: {torch.__version__}")
```

> [!NOTE]
> Set `TRANSFORMERS_OFFLINE=1` in production/demo environments. Without it, every `from_pretrained()` call attempts to fetch model metadata from HuggingFace Hub, which adds latency and can cause hangs if connectivity is unreliable.
