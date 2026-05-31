---
title: "Qwen3.6-35B vs Gemma4-26B: Real Workload Benchmarks on Radeon 7900 XTX"
slug: "qwen36-vs-gemma4-7900xtx-workload-benchmarks"
date: 2026-05-30
draft: false
tags: ["benchmarks", "rocm", "7900xtx", "qwen3.6", "gemma4", "inference", "local-llm"]
---

Most LLM benchmarks measure decode speed on synthetic prompts — 32 tokens in, 128 out, temperature 0.1. That tells you nothing about how a model behaves on actual work: reasoning enabled, variable output lengths, real system prompts, no token cap.

So I ran both Qwen3.6-35B-A3B and Gemma4-26B-A4B through six generic real-world workloads on my 7900 XTX — meeting notes, an incident postmortem, log triage to JSON, a code review, a build-vs-buy decision, and a creative prompt. Both models thinking, both at a matched 32K reasoning budget, nothing capped.

**TL;DR: the model with the slower decoder won the wall clock.** Qwen's MTP speculative decoding is genuinely ~1.65x faster at generating tokens (130 vs 78 tok/s). But Qwen generates almost exactly 2x as many tokens to answer the same prompt — most of it internal reasoning — so it ends up *slower* end to end. Aggregate across all six: Qwen 118.8s vs Gemma 95.6s. Gemma is ~20% faster despite the slower decoder.

## The Setup

Both models run on a Radeon 7900 XTX via ROCm HIP, served through llama-swap. Both are MoE architectures with similar active parameter counts (3B vs 4B). Crucially, **both have reasoning enabled at a matching 32,768-token budget** — this isn't a speed test where one model gets to skip thinking and the other doesn't.

| | Qwen3.6-35B-A3B | Gemma4-26B-A4B |
|-|-|-|
| Quantization | IQ4_XS-Q8nextn hybrid MTP | UD-Q4_K_XL |
| Model size | 19.9 GB | 17.0 GB |
| Active params | 3B (MoE) | 4B (MoE) |
| Context window | 180K tokens | 262K tokens |
| Multi-token prediction | Yes (draft-n-max 3) | No |
| KV cache | q4_0 / q4_0 | q4_0 / q4_0 |
| Slot caching | Yes | Yes |
| Reasoning budget | 32,768 tokens | 32,768 tokens |

**Hardware:** AMD Ryzen 5 9600X, Sapphire NITRO+ 7900 XTX 24GB, 96GB DDR5-6800 CL32. llama.cpp build 9425 (0821c5fcf), built with `GGML_HIP=ON` for the gfx1100 target (`GGML_HIP_ROCWMMA_FATTN=OFF`), ROCm 7.2.3, GCC 16.1.1.

**No output token caps.** Each model generates until it stops naturally; the 32K reasoning budget is the only ceiling, and neither hit it. Sampling was left at each model's production config (temp 1.0 for both, top_k 20 for Qwen, top_k 64 for Gemma), `seed=42` for reproducibility. Thinking and visible output are separated at the API level via `preserve_thinking`, so I can measure them independently.

## The Workloads

Six prompts, all generic enough that any engineer would recognize them:

1. **meeting-notes-summary** — pull decisions, action items with owners, and open questions out of a sprint-planning transcript (~318 prompt tokens)
2. **incident-postmortem** — write a blameless RCA from raw incident notes (~252 prompt tokens)
3. **log-triage-json** — classify a connection-pool failure into a strict JSON schema (~216 prompt tokens)
4. **code-review** — find the bugs in a deliberately broken Python endpoint (~149 prompt tokens)
5. **build-vs-buy-decision** — pick a search backend for a small team and defend it (~185 prompt tokens)
6. **creative-spark** — a metaphor, a loose thread, and an image prompt from a theme (~84 prompt tokens)

## Results: Wall Clock Time

The headline metric is wall clock — how long you actually wait, thinking included.

| Workload | Qwen3.6-35B | Gemma4-26B-Q4 | Winner |
|-|-|-|-|
| meeting-notes-summary | 12.2s | 10.8s | Gemma (+1.4s) |
| incident-postmortem | 28.2s | 21.6s | Gemma (+6.6s) |
| log-triage-json | 10.4s | 9.0s | Gemma (+1.4s) |
| code-review | 20.6s | 23.1s | Qwen (+2.5s) |
| build-vs-buy-decision | 33.1s | 21.5s | Gemma (+11.6s) |
| creative-spark | 14.4s | 9.5s | Gemma (+4.9s) |

**Aggregate: Qwen 118.8s vs Gemma 95.6s — Gemma is ~20% faster.**

Gemma won five of six. The one Qwen took (code review) is the one where both models spent the *least* of their output on reasoning — Gemma actually wrote more visible prose there, so its slower decoder lost the race. Everywhere else, Qwen's reasoning volume dragged its wall clock past Gemma's.

This is the opposite of what the spec sheet predicts. MTP should make Qwen the faster one. So where does the time go?

## Results: Token Breakdown

Here's the whole story in one table. Qwen generates roughly twice as many tokens to answer the same prompt — and most of the extra is internal reasoning.

| Workload | Qwen (total gen tok) | Qwen think/content | Gemma (total gen tok) | Gemma think/content |
|-|-|-|-|-|
| meeting-notes-summary | 1,631 | 86% / 14% | 836 | 78% / 22% |
| incident-postmortem | 3,564 | 72% / 28% | 1,665 | 51% / 49% |
| log-triage-json | 1,389 | 87% / 13% | 699 | 79% / 21% |
| code-review | 2,626 | 56% / 44% | 1,779 | 41% / 59% |
| build-vs-buy-decision | 3,639 | 68% / 32% | 1,661 | 45% / 55% |
| creative-spark | 1,962 | 89% / 11% | 746 | 81% / 19% |

**Aggregate: Qwen generated 14,811 tokens vs Gemma's 7,386 — Qwen produces ~2x more.** Across the board Qwen spends a higher fraction of that on thinking (74% aggregate vs Gemma's 57%). On the JSON-shaped tasks both go deepest — log-triage is 87% thinking for Qwen — while the code review, which is mostly transcription of fixes, has the highest content ratio for both.

So Qwen decodes each token 1.65x faster but emits 2x the tokens. 1.65 < 2. That's the whole result.

### What the extra tokens actually buy

Is that 2x deeper reasoning, or just more words? I read the raw thinking traces. Some of both — but more of the second than I expected.

Gemma thinks in a tight nested outline and stops the moment it has an answer. Its log-triage reasoning ends like this:

> Single JSON object? Yes.
> Fields: severity, category, suspected_root_cause...? Yes.
> Severity constraint: critical/high/medium/low? Yes (critical).
> No prose? Yes.

Qwen does more genuine planning up front — but then it narrates itself well past the finish line, on every single workload:

> Output matches response.
> [Done.]
> *Output Generation* (matches the final refined version)
> ...
> Perfect.✅

Same answer either way. A real chunk of Qwen's token premium is this self-affirmation ceremony — restating the plan, marking steps done, confirming three times that it's about to output. When you pay for those tokens in wall-clock seconds, the gap between *thinking* and *reassuring yourself you've thought* is a gap that costs you.

The funny part: on log-triage, Qwen reasoned its way to the *correct* output format while Gemma reasoned its way to the wrong one. Qwen talked itself out of a code fence — "I'll just output the raw JSON string directly" — and emitted bare JSON, exactly as the prompt demanded. Gemma wrapped its JSON in a ```` ```json ```` block despite "Output only valid JSON, no prose." The verbose model followed the instruction; the terse one didn't.

## Results: Visible Content Quality

Wall clock and token counts don't capture whether the output is any good. Here's what each model actually produced on the workloads where the difference was interesting — same prompt, no caps.

### Code Review (20.6s vs 23.1s)

The one workload Qwen won on speed. I gave both a Python endpoint with three planted problems: a SQL injection, an off-by-one (`range(len(rows) + 1)`), and an undefined `cursor`.

Both caught the SQL injection and the off-by-one, both led with the injection ranked critical, both gave the same parameterized-query fix. The interesting divergence was a fourth bug I didn't plant: **Gemma flagged that `request.args.get('user_id')` returns `None` when the param is missing, which crashes on string concatenation with a `TypeError`.** Qwen didn't surface that one in the same breath. On a code-review task, that's the kind of catch that matters more than formatting — and the slower, leaner model found it.

### Build vs Buy (33.1s vs 21.5s)

I asked both to pick a full-text search backend for a 4-engineer team with no ops, then defend the choice without fence-sitting. They landed on **opposite answers**, and both committed hard.

**Qwen picked managed Algolia:** "Don't overengineer this. You have a 4-person team with no ops dedicated to search infrastructure. Operational toil and feature velocity are your real constraints, not raw search capability." It built a cost/effort/risk matrix and estimated the self-hosted Elasticsearch path at "$15k–30k/yr in eng time" once you factor incident response.

**Gemma picked Postgres:** "my recommendation is blunt: Start with Postgres using `pg_trgm` for fuzzy matching, and do not touch Elasticsearch." It called self-hosted ES a "Death Trap" for a team that size — "When an Elasticsearch cluster hits a heap memory issue or a shard rebalancing loop at 3:00 AM, it's one of your 4 engineers who has to fix it."

Both are *defensible* and both refused to hedge, which is what I asked for. Qwen optimized for DX and time-to-ship; Gemma optimized for zero new infra and cost control. If anything Gemma's answer is the one I'd actually follow for that team — and it got there in two-thirds the time.

### Log Triage → JSON (10.4s vs 9.0s)

Covered above: both produced valid, sensible triage (Qwen called the severity "high," Gemma "critical" — both reasonable for a 20-minute-old connection-pool exhaustion). The only hard difference is schema compliance: Qwen emitted bare JSON as instructed, Gemma fenced it. If you're piping this straight into a parser, Qwen's output drops in clean and Gemma's needs a fence-stripping step.

### Creative Spark (14.4s vs 9.5s)

Theme: "What we quietly give up when everything moves to someone else's cloud." Both produced genuinely good, genuinely different openings — neither was a template fill.

**Qwen** opened: "We are learning to mortgage our memories to vapor, trading the weight of personal archives for the quiet relief of borrowed shelves." Its image prompt was a dim study with empty wooden shelves and luminous clouds cradling photographs that dissolve into mist.

**Gemma** opened: "Our personal histories have become a rented gallery of light, where we pay a monthly fee to walk among ghosts that can be evicted at the flick of a switch." Its loose thread was sharper — "digital sediment," the uncurated fragments too messy for the streamlined cloud — and its image was a sunlit attic with a holographic memory of a child fraying into golden pixels.

Honestly Gemma's was tighter and landed harder, in 9.5s vs 14.4s. But this is a taste call, not a measurement. Both are worth keeping around.

## The Thinking Budget Tradeoff

Neither model came close to exhausting its 32K reasoning budget — they stopped when they had enough. But the *volume* of reasoning is a real, recurring cost:

1. **For simple tasks, both overthink, and Qwen overthinks more.** Log-triage is a five-field classification. Qwen spent 87% of 1,389 tokens thinking about it.
2. **Thinking time isn't free.** Every reasoning token costs wall clock, even with MTP — and MTP doesn't accelerate the part of the run that dominates.
3. **The faster decoder loses if it's the more verbose reasoner.** That's the headline. Decode tok/s is a spec-sheet number; tokens-to-answer is the one that sets your latency.

If you run background jobs that don't need deep reasoning, turning thinking off would cut wall clock dramatically — the thinking fractions above (51–89%) are roughly how much time you'd reclaim. For interactive problem-solving where the chain-of-thought improves the answer, keep it on.

## The MTP Question

On pure decode, MTP delivers. Qwen averaged 130 tok/s of generation across these workloads vs Gemma's rock-steady 78 — a 1.65x edge, exactly what speculative decoding is supposed to buy. MTP draft acceptance ran 41–62% depending on the workload (52.5% overall), lowest on the build-vs-buy decision where the output was least predictable.

But MTP only speeds up *how fast you emit tokens*, not *how many you emit*. With reasoning on, the token count is the bottleneck, and that's where Qwen loses. If you measure visible-content throughput — useful characters per wall-clock second — the two models are basically tied (Qwen ~137 ch/s, Gemma ~130), because Qwen's faster decoder and its larger output volume roughly cancel. The decode-speed advantage that looks decisive on a benchmark chart mostly evaporates once you let the model think.

## Practical Recommendations

**Reach for Qwen3.6-35B when:**
- Output structure matters and you want dense tables/sections by default
- You're doing batch-heavy digest cycles where raw decode throughput compounds across many sequential requests
- You need strict format compliance piped into a parser (it followed the bare-JSON instruction; Gemma didn't)
- You have the VRAM headroom (~20GB model + KV cache)

**Reach for Gemma4-26B-Q4 when:**
- Latency per request matters — it was ~20% faster wall clock here despite the slower decoder
- You want leaner output with less reasoning overhead
- You value the occasional sharper catch (it found the missing-param `TypeError` Qwen glossed)
- The 262K context window or the slightly smaller footprint (~17GB) helps

**Run both** if you're on a multi-model setup like llama-swap. The exclusive GPU group prevents VRAM fragmentation, and you route by job shape rather than defaulting to one model: throughput-bound batch work to Qwen, latency-sensitive single requests to Gemma. That's the split I'm using.

The bigger lesson is one spec sheets won't tell you: a faster decoder doesn't mean faster responses. Tokens-to-answer beats tokens-per-second once reasoning is in the loop, and the only way to know which model wins your workload is to run your workload.

## Raw Prompt/Output Appendix

The complete prompt / reasoning / output traces from all six workloads, both models — nothing truncated — are in a companion file: **[full raw outputs (~105 KB markdown)](https://kmarble.dev/artifacts/qwen36-vs-gemma4-7900xtx-workload-benchmarks/raw-outputs.md)**.

---

*Hardware owned personally. Models run locally via llama-swap on ROCm 7.2.3 / gfx1100. Re-run 2026-05-30 against the live production endpoint; every number in this post comes from that run.*
