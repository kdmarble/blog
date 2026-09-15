---
title: "Two Vendors, One Case: What an NVIDIA + AMD Local AI Build Actually Takes"
slug: "two-vendors-one-case-nvidia-amd-local-ai-build"
date: 2026-09-15
draft: false
tags: ["local-ai", "llm-inference", "homelab", "5090", "r9700", "vllm", "radiance", "exl3"]
---

# Two vendors, one case: what an NVIDIA + AMD local AI build actually takes

## Part I: How I got here

I’ve spent a lot of time trying to figure out how to maximize the hardware I have. My homelab, much like any other I’m sure, has grown organically over time. Piece by piece, upgraded and updated as both time and funds afford. The first leap was selling the strix halo box, and funding an upgrade to a 5090. I got a good deal at a microcenter, apparently just in time as well. That left me with a 5090, a 7900xtx, and a single slot motherboard. Through countless iterations, including three different case swaps, and an additional gpu sale/purchase round, I’ve landed on quite a wacky setup. But it’s mine, and damn does it work well. I decided I would get the r9700 as well, shipping off the 7900xtx, and test it against the 5090. Not quite fair, I know, but if the r9700 was good enough, perhaps I could sell the 5090 and go double, perhaps triple, amd gpu. Alas, the speed of the 5090 is alluring, and I kept both. Here’s what I’m working with now:

- Ryzen 9 9900X (12c/24t)
- ASUS ProArt X870E-Creator WiFi, three x16 connectors, but only two cards fit riser-free once anything 3.5-slot or wider is installed
- 192GB DDR5
- Lancool 216
- 1600W PSU (PG-1600G, two native 12V-2x6 connectors)
- Gigabyte RTX 5090 Gaming OC, 4-slot, middle Gen5 slot at x8
- Radeon AI PRO R9700, single-slot blower, top Gen5 slot at x8, the only reason the pair fits without a riser
- Bottom chipset slot sits under the 5090's overhang
- Storage: 2x16TB Seagate Exos in ZFS, plus four NVMe (1TB root, 2TB, 2TB, 4TB)

So of course, after conquering the hardware battle, I had to optimize the software. Usually you just get a gguf, recompile llama.cpp, and you’re off to the races right? Well, my sweet summer child, let me show you the world of fp4, vllm, and exl. Let’s dig in.

## Part II: One scheduler, two vendors

First, the scheduler. One llama-swap instance runs everything, with resident processes on TTLs, owned by one scheduler that knows which GPU each lane may touch. Three engines on it: exl3/TabbyAPI for the interactive lane on the 5090, vllm with nvfp4 weights for fast workers on the 5090, and Radiance on the r9700, which is a ROCm serving stack built on vllm 0.27 with ggz kernels, running the same model family in mxfp4. llama.cpp routes are still wired up too when I want to test something.

The routing table it actually runs:

| Lane | What it gets | Behavior |
|---|---|---|
| qwen3.8-flash-next (exl3) | interactive chat | resident on the 5090, checkpoints instead of unloading |
| qwen3.8-27b | foreground workers | CUDA first, AMD spillover |
| qwen3.8-27b-background | crons, bots, aux tasks | Radiance only, never touches the 5090 |
| qwen3.8-27b-vllm | explicit fast or long-context work | NVFP4 on the 5090, may suspend flash-next |
| qwen3.8-27b-radiance | explicit AMD worker | MXFP4 + DFlash2 on the r9700 |

 Measured worker traffic is prefill-dominated, 10.9 prompt tokens per generated token across 137M prompts, so heavy workers want the NVIDIA card. Background traffic doesn't need peak speed, and before this routing the r9700 sat at 8% utilization, structurally idle. Now it takes all the background load, different quant format, different engine, same scheduler.

And when the 5090 gets handed over mid-day, cuda-checkpoint suspends the resident 32GB model to disk and restores it later with 202,752 cached tokens intact, 9.88 seconds round trip. The first deploy rolled back because my memory reserve was 4MiB too large, which tells you how tight the admission checks are.

## Part III: The numbers, and the configs

Now the numbers. Everything below is measured on this box, real source fixtures with tool round trips, thinking on, no output caps, sampling matching the model cards (temp 1.0, top_p 0.95). Decode rates are client estimates, completion tokens minus one over generation seconds, so treat them approximate.

Current fleet, September 2026:

| Lane | Engine and quant | Context | Prefill | Decode | Batch notes |
|---|---|---|---|---|---|
| interactive, 5090 + CPU experts | exl3, 2.05 bpw, 192 experts on CPU, MTP2 | 262144 | 600 to 1400 t/s | 40 to 70 t/s | full-depth numbers from my own daily use |
| workers, 5090 | vllm nvfp4, fp8 KV, MTP3 | 262144 | 0.79s first token at ~8K | 173 to 179 t/s | four workflows in 8.15s; c4 429 and c8 742 t/s aggregate on cached prompts (August probe) |
| QAT trial, 5090 | vllm, QUASAR nvfp4 | 229376 | 1.42s first token at ~8K | 150 to 154 t/s | completed a 220,139-token prompt same session |
| background, r9700 | radiance mxfp4 + dflash2 depth 7, fp8 KV | 196608 | 3.68s first token at ~8K | 71 to 73 t/s | four workflows in 17.90s; completed 180,138 tokens |
| opt-in worker, 5090 + CPU | exl3 GLM-5.3-Flash, 2.05 bpw, 224 experts on CPU | 131072 | | 21.5 t/s at 32K (16.7 before tuning) | |

Real depth timings, September fixtures, whole-task wall time including reasoning and tool calls:

| Lane | Prompt depth | Wall time | Output tokens |
|---|---|---|---|
| exl3 interactive | ~40K | 161s | 3751 |
| exl3 interactive | ~64K | 139s | 803 |
| exl3 interactive | ~128K | 254s | 1026 |
| exl3 interactive | ~239K | 307s | 778 |
| radiance, four-way | ~44K | 151s batch, 51s median first token | four tasks (Sept 6 shape, before dflash2) |

A 239K-token source question with a tool round trip completes in under six minutes on the interactive lane. The r9700 at four-way concurrency takes about a minute to first token on 44K prompts, which is exactly why it holds background traffic and not my chat. Fresh 128K prefill on the nvfp4 lane wasn't timed standalone this quarter, the QAT trial completed a 220K prompt and the nvfp4 lane served a 202K production request during a checkpoint swap, so it works, I just don't have a clean rate for it.

Depth curves from the tuning era, August, llama.cpp spill shapes that later got superseded by exl3 and radiance, but these taught me the physics:

| Shape | pp @8K | pp @60K | tg @8K | tg @60K |
|---|---|---|---|---|
| 5090 + CPU spill, q8 KV, ncmoe 4, 131K | 1146 | 819 | 51.9 | 27.4 |
| 5090 + CPU spill, q8 KV, ncmoe 8, 262K | 1031 | 766 | 45.5 | 27.3 |
| r9700 solo, vulkan with graphics queue, ncmoe 32 | 427 | 318 | 22.2 | 18.4 |
| r9700 solo, same backend, ncmoe 99, min VRAM | 353 | 281 | 17.7 | 13.7 |

Three things fall out of that table. Spill decode is RAM-bandwidth-bound, decode drops around 40% from 8K to 60K depth while prefill only gives up about 30%. Q8 KV cache nearly doubled prefill against f16 KV on this architecture, 1146 versus 507 at 8K, with exact needle retrieval at depth, because the upstream Hadamard-rotation fix made the rotated q8 path fast. And the r9700 is a prefill-poor, decode-decent card in spill mode, which is how I ended up routing exactly the traffic that doesn't care about prefill to it.

So here are the actual launch commands, sanitized, paths shortened and UUIDs swapped for placeholders. The interactive lane is exl3 through TabbyAPI, and its whole config is one yaml:

```yaml
# loopback only, no auth, proxied by the scheduler
model:
  backend: exllamav3
  model_name: qfn-opt-exl3-2.05
  max_seq_len: 262144
  cache_size: 262144
  cache_mode: Q8
  gpu_split_auto: true
  autosplit_reserve: [96]
  max_batch_size: 1
  chunk_size: 2048
  cpu_moe_split_experts: 192
  cpu_moe_threads: 8
  template_vars_force: {enable_thinking: true}
  reasoning: true
  tool_format: qwen3_coder
draft_model:
  draft_mode: mtp
  draft_cache_mode: Q8
  draft_num_tokens: 2
memory:
  sysmem_recurrent_cache: 4096
```

Batch size 1, chunk 2048, MTP draft on Q8 cache, 192 experts pinned to CPU, recurrent state parked in pinned host RAM, and EXL3_MOE_PINNED_ARENA=1 in the environment. That shape pulls 40 to 70 t/s decode with 600 to 1400 t/s prefill at full 262K depth on one 5090 plus DDR5, xhigh reasoning included. Same recipe tuned later for GLM-5.3-Flash, 224 experts on CPU, 12 threads, and the 21.5 t/s in the table above.

The worker lane on the NVIDIA card is stock vllm, pinned by image sha not tag:

```bash
docker run --gpus device=0 --ipc=host --memory 64g --memory-swap 64g \
  -p 127.0.0.1:PORT:8000 -v <weights-dir>/qwen38-nvfp4:/model:ro \
  --entrypoint vllm vllm/vllm-openai@sha256:<pinned-sha> serve /model \
  --kv-cache-dtype fp8 --gpu-memory-utilization 0.90 \
  --max-model-len 262144 --max-num-seqs 4 --max-num-batched-tokens 1024 \
  --compilation-config '{"cudagraph_mode":"PIECEWISE"}' \
  --enable-prefix-caching --mamba-cache-mode align \
  --reasoning-parser qwen3 --tool-call-parser qwen3_xml \
  --speculative-config '{"method":"mtp","num_speculative_tokens":3}' \
  --limit-mm-per-prompt '{"image":0,"video":0}'
```

PIECEWISE cudagraphs are required, not taste, the 262K KV floor doesn't fit when default graph capture runs.

The AMD card runs Radiance. The wrapper finds the r9700's render node by PCI ids, vendor 0x1002 device 0x7551, instead of trusting indexes, then hands /dev/kfd plus that node to the container with memlock unlimited:

```bash
docker run --device /dev/kfd --device /dev/dri/renderD<N> \
  --ulimit memlock=-1:-1 --shm-size 8g --memory 64g \
  -p 127.0.0.1:PORT:8000 \
  -v <weights-dir>/qwen38-27b-quark-mxfp4:/model:ro \
  -v <draft-dir>:/draft:ro \
  -e ROCR_VISIBLE_DEVICES=GPU-<amd-uuid> -e HIP_VISIBLE_DEVICES=0 \
  -e CUDA_VISIBLE_DEVICES=0 \
  -e VLLM_ROCM_USE_AITER=1 -e VLLM_ROCM_USE_AITER_UNIFIED_ATTENTION=1 \
  -e RADIANCE_MXFP4=1 -e RADIANCE_MXFP4_W4A8=1 -e RADIANCE_USE_R4D=1 \
  --entrypoint /opt/radiance_entrypoint.sh <radiance-image> /model \
  --tensor-parallel-size 1 --max-model-len 196608 \
  --max-num-seqs 4 --max-num-batched-tokens 1024 \
  --gpu-memory-utilization 0.98 --kv-cache-dtype fp8 \
  --attention-backend R4D \
  --speculative-config '{"method":"dflash","model":"/draft","quantization":"fp8","num_speculative_tokens":7,"attention_backend":"TRITON_ATTN","disable_padded_drafter_batch":true,"draft_sample_method":"greedy"}' \
  --compilation-config '{"pass_config":{"fuse_norm_quant":true,"fuse_act_quant":true}}' \
  --enable-prefix-caching --mamba-cache-mode align \
  --reasoning-parser qwen3 --tool-call-parser qwen3_coder \
  --default-chat-template-kwargs '{"enable_thinking":true,"preserve_thinking":true}'
```

Behind that snippet sits a long tail of RADIANCE_* env knobs for the MXFP4 kernels, weight permutation, skinny gemm, fused rms-quant, dynamic draft width. The flags above are the ones that decide whether it runs at all. Every lane here pins its GPUs by uuid or device id, one vendor's engines never wander onto the other card, and that plus the launch flags is the whole recipe.

## Part IV: What broke

What doesn't work, and I actually tested it.

exl3 on ROCm. The r9700 has the memory, exl3 is exactly the format this build loves, so I tried the port. HIP compiles fine for gfx1201, but exl3's matrix primitive leans on a PTX instruction, mma.sync.aligned.m16n8k16, that just doesn't exist on RDNA4. 

Gemma 31B on Vulkan at depth. This config passed admission and a four-turn tool task fine, then a 253K token request wedged the r9700 mid-generation. Compute-ring timeouts, then the kernel declared hung_task and panicked, and the box auto-rebooted. kdump caught it: GPU devcoredump work waiting on a mutex held by the wedged process, which was itself waiting on a GPU DMA fence. The panic mechanism is proven, the root cause of the GPU timeout still isn't. 

np-pooling on AMD. llama.cpp aggregate throughput is flat against np on Vulkan and HIP, capped around 115 to 145 t/s no matter what you set. np8 pooling only works on CUDA. Don't build r9700 np-pool twins.

## Part V: Tuning worth stealing

Speculative decoding on every lane: native MTP3 on the CUDA vllm lane, MTP2 on exl3, DFlash2 draft at depth 7 on Radiance. Live acceptance on the 5090 runs 80%+. At heavy concurrency K=3 costs around 30% aggregate, at single stream it's pure speed. If you want the background on exl3 quant formats, Tonbi's AI Garage has a good explainer: https://youtu.be/ROHgM8uV_8M

FP8 KV cache on both vllm lanes, Q8 on exl3. I don't auto-crunch KV dtype anymore, because KV quality affects output quality, so it gets weighed against context length per lane, not crushed by default.

The exl3 CPU expert recipe: pinned expert arena env, MTP head fully on GPU, fixed depth 2, 12 CPU threads. That's what turned 16.7 t/s into 21.5 on the GLM row above.

Retention is its own lever. TTL raised to one hour plus checkpoint, so hourly cron traffic hits warm cache instead of cold starts, and the checkpoint lets retention survive a card handoff.

Caveat: all of these are measured operating points on this specific pair of cards, not universal optima. Your mileage on a 9070 xt or a 4090 will differ.

## Part VI: Fin

And that’s where I’ve landed. This is letting me go full local again, and it feels great. Admittedly it did take “agi” Astra to get here, but it was worth it in the end I’d say. Qwen3.8, both flash-next and 27b, are a killer combo. I should start prepping hardware for Qwen4-Flash…
