# AI Infra Hardware — NPU vs GPU vs CPU for Local Inference

> "A 16 TOPS NPU and a 40 TOPS NPU aren't the same product with a different number — they're often different silicon generations with a different instruction set your runtime may not even support yet. Check the software support matrix before you check the spec sheet." — lesson from a full afternoon lost to this

---

## At a Glance

| | |
|---|---|
| **What** | How to pick the right on-device compute target (CPU / GPU / NPU) for running AI models locally |
| **Used by** | Local LLM tools (Ollama, LM Studio), on-device vision models, Windows Copilot+ features |
| **Key promise** | NPUs are the most power-efficient, but only for the workloads their software stack actually supports |
| **Trade-off** | Newest/most efficient hardware often has the least mature and most restrictive software support |

---

## The Problem This Solves

Every modern laptop CPU ships with three distinct places code can run: the CPU cores,
an integrated GPU, and increasingly, a dedicated NPU (Neural Processing Unit). All
three can technically "run AI models" — but they are not interchangeable, and
picking wrong wastes hours chasing a dead end that a spec sheet would have ruled out
in five minutes.

```
Same laptop, three compute targets:

  CPU          — general purpose, always works, slowest for parallel math
  iGPU         — parallel compute via Vulkan/DirectML/ROCm, good LLM throughput
  NPU          — purpose-built matrix multiply silicon, best perf-per-watt,
                 but narrowest software support and most fragmented by generation
```

The instinct — "there's a dedicated AI chip sitting right there, use that" — is
often wrong for LLM workloads specifically, for reasons that have nothing to do
with how well you configure it.

---

## Why NPUs Are Not a Drop-in Replacement for GPU Inference

### 1. NPU generations are not interchangeable — and the differences aren't cosmetic

AMD's own NPU line illustrates this well:

| Generation | Codename example | TOPS | LLM support in AMD's official stack |
|---|---|---|---|
| XDNA1 | Phoenix, Hawk Point (Ryzen 7000/8000/200-series) | ~16 | ❌ CNN INT8 only — no LLM/BF16 |
| XDNA2 | Strix Point, Krackan Point, Strix Halo, Ryzen AI 300+ | 50+ | ✅ Full LLM support |

This is not a driver limitation or a configuration flag — it's stated directly in
AMD's Ryzen AI Software release notes as a hard capability split by silicon
generation. Installing the latest version of AMD's own official toolchain on an
XDNA1 chip will complete successfully, generate no install errors, and then simply
be unable to run the workload you wanted, because the *runtime itself* routes LLM
ops away from unsupported hardware.

**Lesson:** "same vendor, same NPU brand name" ≠ "same capabilities." Always check
the specific chip generation against the specific software stack's support matrix
— not just "does this laptop have an NPU."

### 2. The "TOPS" number is a floor set by the OS/ecosystem, not just a spec

Microsoft's Copilot+ PC certification requires a minimum of **40 TOPS** of NPU
performance to unlock on-device Windows AI features (Phi Silica, Windows Studio
Effects acceleration, Recall, etc.). A perfectly real, working NPU at 16 TOPS
simply doesn't clear that bar — not a bug, a certification threshold.

```
NPU TOPS  <  40   →  No Copilot+ PC NPU features, regardless of chip health
NPU TOPS  ≥  40   →  Eligible for OS-level NPU-accelerated features
```

This means the same physical laptop can be "AI-capable" in a marketing sense while
being locked out of an entire tier of software that assumes a higher-end chip.

### 3. NPUs are architecturally optimized for sustained low-power inference, not peak throughput

NPUs are matrix-multiply accelerators designed to run *continuously* at very low
power draw — think always-on background tasks like webcam bokeh, noise
suppression, or small CNN classification, running for hours on battery without
meaningfully affecting battery life. They are not designed to win at raw
tokens/second for a chat-sized LLM the way a discrete or even integrated GPU with
much higher memory bandwidth will.

| Target | Optimized for | Typical local LLM fit |
|---|---|---|
| CPU | General-purpose, always available | Works, but slowest — fine as a fallback |
| iGPU (Vulkan/DirectML/ROCm) | Parallel throughput | Best fit for chat-sized local LLMs (3B–8B range) on a laptop |
| NPU | Sustained low-power, narrow op support | Best for small always-on CNN/vision tasks, not general LLM chat |

---

## Common Pitfalls (Applies Broadly, Not Just to AMD)

### Pitfall: iGPU silently falls back to CPU

Some inference runtimes (Ollama included) detect a Vulkan-capable integrated GPU
but **disable it by default** unless explicitly told to use it — an integrated GPU
sharing system RAM is a less "safe" default than a discrete GPU with dedicated
VRAM, so runtimes tend to be conservative. If a runtime seems slower than
expected, check whether it's silently running on CPU before assuming the hardware
is the bottleneck.

```bash
# Verify what a runtime actually used, don't assume from the config:
ollama ps   # PROCESSOR column shows "100% CPU" vs "100% GPU" — check this every time
```

### Pitfall: A successful install ≠ a working inference path

An installer completing without errors only proves the *software* installed
correctly — not that the target hardware can execute the workload. Always run the
vendor's own verification/quicktest script and inspect its logs for warnings, not
just its exit code. A script can print "success" while every actual inference call
silently fell back to a slower or unsupported path.

### Pitfall: Chasing PATH/environment issues can mask (or be mistaken for) a hardware limitation

Windows does not propagate environment variable changes to already-running
processes — not `explorer.exe`, not shells spawned before the change, nothing
alive when the registry was written. This can make a completely unrelated PATH
problem look identical to "the hardware doesn't support this," and vice versa. Rule
out environment/PATH issues with a clean process (or a full restart) *before*
concluding the hardware itself is the blocker — but also don't let a fixed PATH
issue create false confidence that the underlying hardware capability question is
answered. They're independent failure modes that can occur in the same debugging
session.

---

## Decision Framework

```
Need to run an LLM locally on a laptop with CPU + iGPU + NPU?

1. Check the NPU generation against the specific runtime's support matrix FIRST.
   → If it doesn't explicitly list your exact chip/generation, assume no support
     until proven otherwise. Don't rely on "same vendor" or "has an NPU" as a proxy.

2. If the NPU doesn't support LLM workloads (very common on last-gen chips):
   → Use the iGPU via Vulkan/DirectML/ROCm. This is usually the best throughput
     option for chat-sized models (3B-8B) on a laptop without a discrete GPU.

3. Reserve the NPU for what it's actually good at:
   → Small, sustained, low-power CNN/vision inference running continuously
     (webcam effects, wake-word detection, background classification) —
     not general-purpose LLM chat.

4. Always verify with the vendor's own test script AND check the runtime's
   own process/utilization reporting — don't trust "install succeeded" or
   "script printed success" alone.
```

---

## Real-World Grounding

- **Windows Copilot+ PC tier**: Microsoft draws the line at 40 TOPS specifically
  so that on-device Phi Silica and other OS AI features have a guaranteed
  performance floor across all "Copilot+" branded hardware — a fragmented
  experience across NPU generations would undermine the feature's reliability
  promise.
- **AMD Ryzen AI software stack**: Explicitly scopes LLM/BF16 support to
  XDNA2-generation silicon (Strix Point onward) while keeping XDNA1
  (Phoenix/Hawk Point) limited to CNN INT8 — a deliberate scoping decision
  reflected in the official release notes, not an oversight.
- **Ollama's iGPU handling**: Defaults to *not* using an integrated GPU unless
  explicitly enabled, because integrated GPUs share system memory with the CPU
  and can have less predictable behavior under memory pressure than a discrete
  GPU — a conservative default that trades a bit of performance for stability out
  of the box.

---

## Related

- Full incident writeup with the exact commands, error messages, and timeline
  from chasing this exact problem on real hardware:
  [`ops-runbooks: AMD NPU Won't Run LLMs — A Hardware Dead End`](https://github.com/suphakin-th/ops-runbooks/blob/main/docs/incidents/2026-08-19-ryzen-npu-llm-dead-end.md)
