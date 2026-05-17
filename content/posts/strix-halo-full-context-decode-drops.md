---
title: "Strix Halo at Full Context — Why Your Decode Drops 64% and What Actually Fixes It"
slug: "strix-halo-full-context-decode-drops"
date: 2026-05-16
draft: false
tags: ["strix-halo", "benchmarks", "rocm", "vulkan", "llama.cpp", "qwen", "mtp", "inference"]
---

I've been running a Minisforum MS-S1 Max (AMD Ryzen AI MAX+ 395, Radeon 8060S iGPU, 128GB LPDDR5X) as my homelab's long-context inference tier for months. The headline from the first round: Qwen3.6-35B-A3B at ~25 tok/s with 153k tokens of live context, all under 100W.

A lot has changed since then. ROCm 7.13 finally got gfx1151 codegen working (7.2.2 could see the GPU but couldn't compile shaders). MTP merged to llama.cpp main on May 16. I've run three models across two backends at three prompt lengths plus a dedicated full-context decode test.

These are all the numbers.

## Hardware and Software

| Component | Spec |
|-----------|------|
| APU | AMD Ryzen AI MAX+ 395 (Radeon 8060S iGPU, gfx1151) |
| Memory | 128GB LPDDR5X unified |
| Practical weight limit | ~100GB (48GB reserved for system/KV/compute) |
| Practical max context | 262k (DeltaNet hybrid), 131k (dense) |
| ROCm backend | 7.13.0a20260515, therock-gfx1151 codegen path |
| Vulkan backend | Vulkan 1.3 RADV, unified heap enabled |
| llama.cpp | b9188 (604990613), May 16, 2026 |
| ROCm build | -DGGML_HIP=ON, CMAKE_PREFIX_PATH=therock-gfx1151 |
| Vulkan build | -DGGML_VULKAN=ON |
| Serving | llama-swap proxy, temperature 0.1 for all tests |

All tests used -c 262144, -np 1, -t 16, -tb 32, -ngl 999, -fa on, --no-mmap. Prompts were made unique per model to prevent KV cache hits between tests.

## Models Tested

| Model | Architecture | Quant | Size | Active Params |
|-------|-------------|-------|------|---------------|
| Qwen3.6-35B-A3B | MoE (3B active) | Q8_K_XL | 38.5GB | 3B |
| Qwen3.6-27B | Dense | Q8_0 | ~27GB | 27B |
| Qwen3.5-122B-A10B | MoE (10B active) | Q4_K_L | ~99GB | 10B |

Each tested with and without MTP (`--spec-type draft-mtp --spec-draft-n-max 2`) on both ROCm and Vulkan: 12 total configurations.

## Historical Results (for reference)

### April 2026 — Vulkan RADV, 9-Model Shootout (llama.cpp b8637)

| Model | Quant | Decode tok/s | Prefill tok/s | Notes |
|-------|-------|-------------|---------------|-------|
| Qwen3-Coder-Next 80B | Q8_0 | 36.2 | ~700 | Fastest MoE at the time |
| Gemma 4 26B-A4B | Q8_0 | 31.2 | ~600 | |
| GPT-oss 120B | MXFP4 | 38.3 | ~800 | Fastest, but zero needle retrieval |
| MiniMax M2.5 | Q3_K_S | 27.2 | ~500 | |
| **Qwen3.5-122B** | **Q6_K_L** | **18.2** | **~400** | **Production baseline** |
| Step-3.5-Flash | Q3_K_XL | 25.9 | ~450 | |
| Devstral-2 123B | Q5_K_M | 2.7 | ~100 | Too slow |
| dots1 142B | Q4_K_M | 3.1 | ~80 | Too slow |

### April 2026 — Vulkan Qwen3.6-35B Q8 at Various Context Depths (b8762)

| Context | Prefill tok/s | Decode tok/s |
|---------|---------------|-------------|
| 1k tokens | 1043.2 | 32.0 |
| 32k tokens | 703.1 | 30.0 |
| 64k tokens | 460.6 | 28.6 |

BF16 was tested at the same time: 9.8 tok/s decode (3x slower than Q8 due to bandwidth saturation). Abandoned.

### May 2026 — ROCm 7.13 Initial Results (b9188)

First ROCm results on gfx1151 with therock-gfx1151 codegen. Empty context, Qwen3.6-35B Q8:

| Prompt | Prefill tok/s | Decode tok/s | Power (W) | Temp (°C) |
|--------|---------------|-------------|-----------|-----------|
| Simple math (17 tok) | 167.6 | 48.1 | 74-89 | 48 |
| Haiku (17 tok) | 165.8 | 45.9 | 74-89 | 48 |
| CPU vs GPU essay (26 tok) | 186.9 | 46.1 | 74-89 | 48 |
| Longer prompt (26 tok) | 192.0 | 46.1 | 74-89 | 48 |

ROCm at 46 tok/s was 2.3x the Vulkan baseline of 20 tok/s. Decode was remarkably stable across prompt depths — DeltaNet's linear layers shine. Prefill degraded 56% from 1k to 64k on Vulkan (1043 to 461 tok/s).

## Round 1: Empty Context, Single Prompt

55-token prompt (TCP/UDP explanation), 200 output tokens.

### Qwen3.6-35B-A3B (MoE, Q8_K_XL)

| Config | Prefill tok/s | Decode tok/s | MTP draft/accept |
|--------|---------------|-------------|-----------------|
| ROCm non-MTP | 237 | 46.0 | - |
| ROCm MTP | 205 | 58.3 | 152/122 (80%) |
| Vulkan non-MTP | 259 | 32.6 | - |
| Vulkan MTP | 252 | 45.6 | 140/128 (91%) |

### Qwen3.6-27B (dense, Q8_0)

| Config | Prefill tok/s | Decode tok/s | MTP draft/accept |
|--------|---------------|-------------|-----------------|
| ROCm non-MTP | 104 | 7.7 | - |
| ROCm MTP | 85 | 13.2 | 145/126 (87%) |
| Vulkan non-MTP | 143 | 7.3 | - |
| Vulkan MTP | 94 | 10.7 | 144/127 (88%) |

### Qwen3.5-122B-A10B (MoE, Q4_K_L)

| Config | Prefill tok/s | Decode tok/s | MTP draft/accept |
|--------|---------------|-------------|-----------------|
| ROCm non-MTP | 114 | 23.2 | - |
| ROCm MTP | 101 | 30.1 | 146/125 (86%) |
| Vulkan non-MTP | 93 | 26.7 | - |
| Vulkan MTP | 89 | 22.7 | 142/127 (89%) |

Key finding from this round: ROCm wins decode on 35B and 27B. 122B Vulkan non-MTP (26.7) edges ROCm (23.2) — likely ROCm HIP overhead scales worse with 10B active params. But MTP flips it back (ROCm 30.1 vs Vulkan 22.7). Dense 27B is unusable for interactive work at 7-13 tok/s regardless of backend.

## Round 2: Three Prompt Lengths

Three lengths: chat (~25 words, 100 output), coding (~87 words, 400 output), max context (~24,600 words / ~78k tokens, 30 output). Unique prompts per model.

### Qwen3.6-35B-A3B (PP tok/s / TG tok/s)

| Config | Chat | Coding | Max Context |
|--------|------|--------|-------------|
| ROCm non-MTP | 257 / 46.2 | 418 / 45.8 | 453 / 35.9 |
| ROCm MTP | 226 / 63.7 | 368 / 51.1 | 490 / 41.7 |
| Vulkan non-MTP | 154 / 32.7 | 304 / 32.5 | 533 / 30.0 |
| Vulkan MTP | 153 / 46.8 | 268 / 37.8 | 465 / 38.8 |

### Qwen3.6-27B (PP tok/s / TG tok/s)

| Config | Chat | Coding | Max Context |
|--------|------|--------|-------------|
| ROCm non-MTP | 119 / 7.7 | 162 / 7.6 | 157 / 6.4 |
| ROCm MTP | 101 / 14.2 | 133 / 11.5 | 154 / 10.1 |
| Vulkan non-MTP | 65 / 7.4 | 117 / 7.3 | 190 / 7.0 |
| Vulkan MTP | 55 / 11.3 | 103 / 9.8 | 158 / 10.7 |

### Qwen3.5-122B-A10B (PP tok/s / TG tok/s)

| Config | Chat | Coding | Max Context |
|--------|------|--------|-------------|
| ROCm non-MTP | 98 / 23.3 | 176 / 23.0 | 266 / 14.3 |
| ROCm MTP | 82 / 31.0 | 166 / 26.1 | 215 / 21.4 |
| Vulkan non-MTP | 105 / 26.8 | 179 / 26.6 | 233 / 24.5 |
| Vulkan MTP | 60 / 23.2 | 108 / 24.5 | 185 / 23.7 |

MTP accept rates (chat/coding/max context):
- 35B ROCm: 91%/65%/90%
- 35B Vulkan: 94%/67%/90%
- 27B ROCm: 96%/69%/90%
- 27B Vulkan: 97%/68%/100%
- 122B ROCm: 88%/68%/90%
- 122B Vulkan: 93%/72%/82%

Decode is remarkably stable across prompt lengths (TG barely changes between 100 and 400 output tokens). Prefill scales with length. At max context (78k tokens), prefill dominates wall time: 27B takes 8.3 minutes just for prefill.

MTP on 122B Vulkan regresses across all prompt lengths (-11% vs non-MTP). The MTP overhead exceeds the compute benefit at this model size on Vulkan.

## Round 3: Full Context Decode

The one that matters. ~76k token prompt (120 Python classes with random parameters), 5000 output tokens. Each model got a unique suffix so the KV cache never hit between models.

### Qwen3.6-35B-A3B

| Config | Decode tok/s | Empty Context Decode | Drop | Wall Time |
|--------|-------------|---------------------|------|-----------|
| ROCm non-MTP | 16.6 | 46.2 | **-64%** | 471s |
| ROCm MTP | 37.5 | 63.7 | -41% | 290s |
| Vulkan non-MTP | 28.9 | 32.7 | **-12%** | 317s |
| Vulkan MTP | 34.3 | 46.8 | -27% | 310s |

### Qwen3.6-27B

| Config | Decode tok/s | Empty Context Decode | Drop | Wall Time |
|--------|-------------|---------------------|------|-----------|
| ROCm non-MTP | 6.2 | 7.7 | **-19%** | 1296s |
| ROCm MTP | 9.3 | 14.2 | -34% | 1032s |
| Vulkan non-MTP | 6.7 | 7.4 | **-9%** | 1143s |
| Vulkan MTP | 9.0 | 11.3 | -20% | 1034s |

### Qwen3.5-122B-A10B

| Config | Decode tok/s | Empty Context Decode | Drop | Wall Time |
|--------|-------------|---------------------|------|-----------|
| ROCm non-MTP | 13.9 | 23.3 | **-40%** | 647s |
| ROCm MTP | 19.2 | 31.0 | -38% | 614s |
| Vulkan non-MTP | 23.7 | 26.8 | **-12%** | 536s |
| Vulkan MTP | 21.9 | 23.2 | -6% | 638s |

All MTP configs achieved 76-78% acceptance on long outputs (vs 88-100% on short prompts).

## The Decode Drop Story

At empty context, ROCm was 2.3x faster than Vulkan for 35B decode (46 vs 32 tok/s). At full context with 76k tokens in the KV cache, ROCm non-MTP drops to 16.6 tok/s — a 64% collapse. Vulkan only drops 12% (32.7 to 28.9).

Why: ROCm's HIP backend pays higher overhead per KV access. With 76k tokens in the cache, each decode step requires attention over the full cache, and the ROCm path can't keep up with the bandwidth demands. Vulkan's shader path is simpler and more memory-efficient for this workload.

MTP recovers most of the gap. ROCm MTP at 37.5 tok/s still beats Vulkan non-MTP at 28.9. But the margin has narrowed from 2.3x to 1.3x.

### What This Means for Real Work

If you're running agentic sessions with long context and long outputs:

- **35B MoE ROCm MTP** (37.5 tok/s at full context) is the sweet spot for interactive chat
- **122B Vulkan non-MTP** (23.7 tok/s) is a viable quality lane if you don't need MTP
- **27B dense** (6-9 tok/s) is not viable regardless of backend — the MoE architecture is the real speed multiplier
- **Vulkan MTP on 122B** regresses (-6% at full context) — skip it and use non-MTP instead

## Power and Thermal

| Metric | ROCm 7.13 | Vulkan |
|--------|-----------|--------|
| Socket power | 74-89W | N/A |
| Temperature | 48°C | N/A |
| GFX activity | 78-99% | N/A |
| GTT used (35B loaded) | 43GB/114GB | N/A |

ROCm gives better GPU utilization and lower idle overhead (157MB vs unknown for Vulkan). Power stays consistent across all workloads.

## What Changed

- **ROCm 7.2.2 → 7.13**: the therock-gfx1151 codegen path finally works. 7.2.2 could enumerate the GPU but couldn't compile shaders.
- **MTP merged to llama.cpp main**: May 16. `--spec-type draft-mtp --spec-draft-n-max 2` enables it. Accept rates are 76-90% depending on output length.
- **Voyager node swap**: voyager (Ryzen 9900X, RTX 5090) is now the foreground Hermes session; artemis (Strix Halo) is the parallel worker/auxiliary node.

## Setup Notes

ROCm 7.13 + therock-gfx1151 requires the custom codegen path from kyuz0's TheRock toolbox. Build llama.cpp with:

```
cmake -B build-rocm -DGGML_HIP=ON -DCMAKE_PREFIX_PATH=/home/therock-gfx1151
cmake --build build-rocm --config Release
```

BF16 models don't work at full context on Strix Halo — the drivers can't handle it. Q8 is the sweet spot for 35B, Q4 for 122B.

`amd_iommu` is off — it's safe and gives a 2-6% speed boost on this hardware.

RADV unified heap is enabled via ~/.drirc. TTM pages_limit is set to 112GB.

## Summary

ROCm is faster when the KV cache is small. Vulkan is more stable when it's full. MTP recovers most of ROCm's loss at full context but doesn't help on 122B Vulkan. The MoE architecture (3B or 10B active) is the real differentiator — dense 27B is 5x slower than 35B MoE because it has 27B active params to process per token.

For my setup, ROCm 7.13 with MTP on the 35B MoE remains the production choice: 37.5 tok/s at full context, under 100W, with 262k context available.

---

*Hardware: Minisforum MS-S1 Max, Ryzen AI MAX+ 395, 128GB LPDDR5X, Radeon 8060S iGPU (gfx1151). Software: ROCm 7.13.0a20260515, Vulkan 1.3 RADV, llama.cpp b9188. All numbers from live llama-swap proxy, not llama-bench synthetic runs. Raw data available on request.*
