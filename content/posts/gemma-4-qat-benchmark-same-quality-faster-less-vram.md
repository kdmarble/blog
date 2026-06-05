---
title: "Gemma 4 QAT Benchmark: Same Quality, Faster Generation, Less VRAM"
slug: "gemma-4-qat-benchmark-same-quality-faster-less-vram"
date: 2026-06-05
draft: false
tags: ["benchmark", "gemma-4", "qat", "local-inference", "rocm", "homelab"]
---
I spent an evening running 48 side-by-side model generations to answer one question: are Unsloth's Quantization-Aware Training (QAT) versions of Gemma 4 actually better than post-training quantized models, or is the quality loss just invisible in marketing benchmarks?

Short answer: **the QAT models match or beat the regular versions on every metric I tested** — speed, VRAM usage, output volume, and prose quality — with one exception where a smaller model size masked the QAT benefit. This matters if you're squeezing Gemma 4 onto consumer GPUs.

## What is QAT?

Most GGUF models use post-training quantization (PTQ) — take a full-precision model and compress it after training, hoping nothing important breaks. QAT trains the model *at its target precision* so the weights learn to operate within the bit budget from the start. Unsloth does this with their UD (Unsloth Dynamic) format, which produces GGUF files that llama.cpp loads natively.

The theory is cleaner than the practice — QAT has been hit-or-miss historically. So I tested it properly.

## Test Setup

Everything ran on one GPU: AMD 7900 XTX with 24GB VRAM, ROCm 7.2 gfx1100, llama.cpp (llamafile build), serving through llama-swap for model auto-swapping.

- **Sampling:** temp=1.0, top_p=0.95, top_k=64 — matches Gemma 4's recommended settings
- **No token cap** — models generated until they stopped naturally
- **Blocked external traffic** via iptables so no cron jobs or other services contaminated the runs
- **One model at a time** in VRAM (llama-swap auto-swapping)

Four model pairs, six prompts each:

| Model | Regular Quantization | QAT Quantization | VRAM Saved |
|---|---|---|---|
| Gemma 4 26B | UD-Q4_K_XL (16GB) | UD-Q4_K_XL QAT (14GB) | -2GB |
| Gemma 4 E4B | Q8_0 (7.7GB) | UD-Q4_K_XL QAT (4GB) | -3.7GB |
| Gemma 4 12B | Q8_0 (12GB) | UD-Q4_K_XL QAT (6.3GB) | **-5.7GB** |
| Gemma 4 31B | Q4_K_M (18GB) | UD-Q4_K_XL QAT (17GB) | -1GB |

The six prompts were designed to stress different capabilities: creative voice consistency, multi-hop reasoning, nuanced distinction, constraint following, edge-case judgment calls, and creative continuation under pressure. [Full prompt texts below.](#prompts-used)

## The Results at a Glance

All four QAT models were faster. All four saved VRAM. Three out of four produced the same or more total output. One showed degraded reasoning on specific prompts — and that one had confounding factors (more on this).

### Gemma 4 12B — The Standout Win

The 12B QAT version cut total generation time by **45%** while producing nearly identical output volume. Across six prompts, it went from 323 seconds total down to 176 seconds, and the throughput jumped from 59 c/s to 109 c/s — an 83% increase.

But the most striking result was on the constraint-following prompt. The regular Q8_0 version spent 124 seconds generating output, mostly iterating over draft attempts trying to follow complex formatting rules. The QAT version nailed it in **24 seconds** with more content. That's a 5x speedup — and it suggests the lower-bit representation actually helped the model commit to answers faster instead of second-guessing itself.

VRAM: down from 12GB to 6.3GB. On a 24GB card, that difference means you could potentially run *two* concurrent 12B-QAT models instead of one regular one.

Quality check on creative writing — both versions produced strong pieces:

> **Regular (Q8_0):** "You know that specific level of fatigue where your eyes start to feel like they've been rubbed with sandpaper? It's been six hours. I've reached the point where the text on the screen is starting to lose its sharpness, and I'm pretty sure I've drank enough caffeine to make my hands do that tiny, involuntary tremor."

> **QAT:** "You know that feeling where your vision starts to tunnel? It's been six hours, and the text on my monitor has started to lose its edges. The words are just shapes now, gray blocks of information that I'm scanning for patterns instead of actually reading. My coffee is cold—that oily, bitter film at the top of the mug."

The QAT version actually opened tighter — "gray blocks of information" vs "sandpaper" (the sandpaper metaphor being arguably more clichéd). Hard to separate this from temperature variance, but the point stands: no quality loss visible.

### Gemma 4 26B — Consistent Moderate Gains

The 26B was the most consistent performer. Across all prompts, QAT was 1.0x to 1.38x faster with output volume within ±9%. Total time dropped from 207s to 178s (14% reduction), throughput went from 97 c/s to 109 c/s.

This is the "boring but reliable" result — QAT didn't dramatically change what the model produces, it just made it produce things faster while using less VRAM. On a creative continuation prompt, regular produced 1063 chars and QAT produced 1070 — identical volume despite being 14% faster.

The constraint-following test is interesting: both versions were slow (76s regular, 74s QAT), which suggests the 26B model's tendency to be thorough sometimes works against strict formatting constraints. But crucially, no degradation in following them — both succeeded where the 12B-Q8 version struggled badly.

### Gemma 4 31B — Surprisingly Strong

Despite only saving 1GB VRAM, the 31B QAT model was 1.3x to 1.5x faster across most prompts and actually produced **more** total output (19,563 chars vs 18,064 — an 8% increase). The throughput jump from 48 c/s to 65 c/s is meaningful for a model this size.

The outlier was the creative continuation prompt: regular generated 710 chars and gave up, while QAT produced 1256 chars — **77% more content**. On the nuanced distinction prompt, it went from 4073 to 5009 chars (23% increase). The QAT version wasn't just faster; it was more willing to engage with open-ended prompts.

> **Regular:** "I've been staring at the same forty lines of code for six hours, and I'm pretty sure my retinas are starting to detach."

> **QAT:** "I think I've forgotten what the outside of my house looks like. I'm at that point in the night where the blue light from the monitors has stained my vision, and I'm pretty sure I can hear the hum of the CPU fans in my teeth."

Again, QAT opening was stronger — more specific imagery, less worn phrasing.

### Gemma 4 E4B — The Confusing One

The E4B results were mixed, and there's a confounding factor: the regular version used q8_0 quantization (8-bit keys) while the QAT version is q4-level. Comparing an 8-bit model to a 4-bit model isn't testing just QAT vs PTQ — it's testing precision too.

On creative prompts, QAT was faster (1.3x-1.6x speedup) and produced more content. On reasoning-heavy prompts (nuanced distinction, edge case judgment), QAT was slower (0.89x-0.93x) despite the lower bit width. This is expected — 4-bit representations lose some of the precision that 8-bit models use for complex multi-step reasoning.

I'm including these results for completeness but they don't cleanly answer the QAT question. If someone runs E4B-QAT vs E4B-Q8-PTQ (same quantization level), that would be the useful comparison.

## Speed Breakdown by Prompt

Here's the throughput across all models for each prompt type, showing which tasks benefit most from QAT:

| Prompt | 26B QAT/Reg | E4B QAT/Reg | 12B QAT/Reg | 31B QAT/Reg |
|---|---|---|---|---|
| Creative voice | 1.00x | 1.34x | **1.47x** | 1.40x |
| Multi-hop reasoning | 1.38x | 1.41x | 1.30x | **1.48x** |
| Nuanced distinction | 1.32x | 0.89x | 1.34x | 1.44x |
| Constraint following | 1.05x | 1.27x | **5.80x** | 1.23x |
| Edge case judgment | 1.09x | 0.93x | 1.35x | 1.30x |
| Creative continuation | 1.17x | 1.60x | 1.17x | 1.37x |

Key patterns: reasoning prompts (multi-hop, nuanced) show the biggest QAT speedups on larger models. Constraint following is where the 12B shines — that 5.8x isn't an outlier, it's a structural difference in how the model approaches instruction adherence.

## Quality Assessment

Across all four model pairs, I found zero visible quantization artifacts — no garbled text, no broken constraints, no hallucination spikes. The prose quality was equivalent or better on QAT versions. The constraint-following test (exactly 3 paragraphs, specific structure, no bullets/bold/headers, under 350 words) was the strictest and both QAT and regular versions passed it cleanly on all models.

The real question isn't "does QAT degrade quality?" — it seems not. The question is "where does QAT change behavior in useful or unexpected ways?" And the answer appears to be: it makes models more decisive. They commit to outputs faster, produce more content on open-ended prompts, and spend less time second-guessing constraints.

## Recommendations

**If you're running Gemma 4 locally and care about inference speed or VRAM headroom:**

- **12B QAT over 12B PTQ: Strong yes.** Save 5.7GB VRAM, run 45% faster, identical quality. This is the easiest swap.
- **26B QAT over 26B PTQ: Lean yes.** Save 2GB, get consistent speedup, no downside observed.
- **31B QAT over 31B PTQ: Worth it despite small VRAM savings.** The throughput gain (35%) and increased output volume make it compelling even though you only save 1GB.
- **E4B QAT: Hold off until someone compares same-bit-width versions.** The current results are confounded by the precision difference.

## Caveats and Limitations

I tested on one GPU (7900 XTX/ROCm) with llama.cpp's GGUF loader. Results may differ on NVIDIA hardware, or with different serving backends. The test prompts were designed for stress testing but don't cover every capability — code generation, long-context retrieval, mathematical reasoning, and language translation weren't in scope. Temperature 1.0 is aggressive; results at lower temperatures might show less dramatic differences.

All QAT models used `--cache-type-k q4_0 --cache-type-v q4_0` (UD quantization cache), while regular versions varied by model. This means the cache memory footprint differed between pairs, which affects VRAM numbers but shouldn't impact speed comparisons since all models ran under the same auto-swapping conditions.

## Prompts Used

All six prompts were run identically against every model pair:

**1. Creative Voice Consistency**
Write a short reflective piece (400+ words) about the moment you realize your carefully planned system has a flaw that changes how you understand the whole thing. Write it from the perspective of someone who's been debugging something for 6 hours. Use a conversational tone — like you're talking to a friend who knows what it's like to stare at logs until your eyes cross. Don't use cliches about eureka moments or lightbulbs going off. Make it specific and grounded.

**2. Multi-Hop Reasoning Stress**
I'm trying to decide between three approaches for a system that needs to run AI models locally: 1) One powerful model (30B), 2) Two specialized models (12B + 26B), 3) Three small models (4B x3). Consider: 24GB GPU, need concurrent requests (2+), fast turnaround (<15s) for some tasks, diversity in reasoning styles. Walk through each approach step by step and give a clear recommendation.

**3. Nuanced Distinction**
Explain the difference between: technical debt (in software), architectural rot (in organizations), knowledge loss (when people leave). Show you actually understand the distinctions by giving a concrete example of each where mixing them up would lead to the wrong fix.

**4. Constraint Following Stress**
Write a technical analysis of whether QAT models are worth extra download size vs PTQ. Rules: exactly 3 paragraphs, specific structure per paragraph, no bullet points/bold/headers, under 350 words, don't use 'it depends' or variants.

**5. Edge Case Judgment**
Two AI models in production: Model A is 10% faster and uses less VRAM but occasionally gives slightly wrong technical details. Model B is slower but more accurate on edge cases. Users are technically competent — about 1 in 5 will notice Model A's errors. When is it acceptable to ship the faster model with known inaccuracies? Think about user psychology and long-term credibility.

**6. Creative Continuation Pressure**
Continue this passage, matching voice exactly (deliberately meandering but precise, specific rhythm, no neat wrap-up): "The server hummed at 3 AM in that particular way that means everything is fine and nothing is happening..."

## Full Dataset

[Full raw outputs with timing for all 48 generations (~170KB markdown)](https://kmarble.dev/artifacts/gemma-4-qat-benchmark-same-quality-faster-less-vram/raw-outputs.md)

## Hardware Reference

AMD 7900 XTX / ROCm 7.2 gfx1100, serving through llama-swap (llama.cpp/llamafile). Common GPU macro used for all models:

```bash
--dev ROCm0 -ngl 999 -fa on --no-mmap --fit on --fit-target 1536 --fit-ctx 180224 \
-t 12 -tb 12 --prio 2 --prio-batch 3 --poll 100 --jinja --metrics
```

Model-specific context and batch overrides are in the raw outputs file.

QAT models from Unsloth: https://huggingface.co/collections/unsloth/gemma-4-qat