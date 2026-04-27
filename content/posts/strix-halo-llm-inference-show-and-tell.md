---
title: "Strix Halo LLM Serving: 25 tok/s at 151k Context Under 100W"
slug: "strix-halo-llm-inference-show-and-tell"
date: 2026-04-26
draft: false
tags: ["ai", "homelab", "llm-inference", "strix-halo", "llama-cpp", "qwen", "local-ai"]
---

I have been tuning this box for my own daily agent/chat workflow, extensively, over the course of a month or so. I think I'm finally dialing things in to a very useful baseline, but I'd love to hear from others that are getting better perf. I really think we're hitting the "good enough" stage of local AI, in that the models we have available, their capabilities, and open source harness options, all mean you can reach for local AI _first_. Now, you really only need to reach for cloud models for seriously complex stuff. Hope others find this useful!

<div>
<video controls preload="metadata" poster="/artifacts/strix-halo-llm-inference-loading/strix-halo-llm-inference-screencast-poster.jpg" style="width: 100%; border: 1px solid #2a2a4a; border-radius: 8px; background: #0a0a1a;">
<source src="https://assets.kmarble.dev/artifacts/strix-halo-llm-inference-loading/strix-halo-llm-inference-screencast-2026-04-26.webm" type="video/webm">
<a href="https://assets.kmarble.dev/artifacts/strix-halo-llm-inference-loading/strix-halo-llm-inference-screencast-2026-04-26.webm">Open the screencast</a>
</video>
</div>

---

I have a small Strix Halo machine, lovingly named `artemis` (after both the new NASA missions, and the Greek goddess), doing the heavy chat and reasoning work for my homelab. The headline number is simple: Qwen3.6-35B-A3B serving through llama.cpp at about 25 tok/s, with the live server metrics showing a largest observed context of 153,562 tokens. In the screencast, that run is under 100W.

This isn't a clean-room benchmark. It's a technical show and tell for a setup I actually use: OpenAI-compatible API, llama-swap in front, Hermes routing chat and cron jobs to the right host, and enough context headroom that long-running agent sessions are practical instead of hypothetical.

The short version: Strix Halo is an interesting local inference target because the 128GB unified LPDDR5X pool changes what "fits" means. It's not a 5090, and it isn't trying to be. The useful trick is that a large quantized model, its KV cache, and the compute buffers all live in one shared memory system behind a competent Vulkan/RADV stack.

I also packaged the loading and quantization visuals as a standalone artifact. You can open the interactive version here:

<p>
  <a href="/artifacts/strix-halo-llm-inference-loading/">Open the interactive artifact</a>
  ·
  <a href="https://assets.kmarble.dev/downloads/strix-halo-llm-inference-loading-2026-04-26.zip">Download the standalone package</a>
</p>

<iframe src="/artifacts/strix-halo-llm-inference-loading/" title="Strix Halo LLM inference loading and quantization visuals" loading="lazy" style="width: 100%; height: 760px; max-height: 82vh; border: 1px solid #2a2a4a; border-radius: 8px; background: #0a0a1a;"></iframe>

## The live setup

Captured on 2026-04-26 at 22:34 CDT.

| Component | artemis |
|-|-|
| Machine | Minisforum MS-S1 MAX |
| APU | AMD Ryzen AI Max+ 395 with Radeon 8060S |
| CPU layout | 16 cores / 32 threads |
| Memory | 124 GiB visible LPDDR5X unified memory |
| OS | CachyOS |
| Kernel | Linux 7.0.0-1-cachyos |
| GPU backend | Vulkan, Mesa RADV |
| Vulkan device | Radeon 8060S Graphics (RADV STRIX_HALO) |
| Mesa | 26.0.5-arch2.2 |
| llama.cpp | b8890, commit `8bccdbbff` |
| llama-swap | version 205, commit `66639e83f7be4f1354817e45321f50bdb8e3227d` |

`artemis` isn't the only inference host. It's the long-context chat/high-reasoning tier. `voyager`, (also lovingly named after the NASA missions) the main homelab box, runs a 7900 XTX and handles the local cron tier.

| Host | Role | Model |
|-|-|-|
| artemis | Chat and high-reasoning work | `qwen3.6-35b-q8` |
| voyager | Local cron tier | `qwen3.6-27b` |

The two machines are tied together over a direct 2.5GbE link:

```text
voyager  10.10.30.1/30
artemis  10.10.30.2/30
```

Hermes and other clients hit llama-swap through an OpenAI-compatible endpoint. llama-swap keeps one large model loaded and makes model routing boring.

## The number from the screencast

The live llama.cpp metrics on `artemis` at capture time:

```text
llamacpp:prompt_tokens_total              1,848,300
llamacpp:prompt_seconds_total             5,727.88
llamacpp:tokens_predicted_total           101,816
llamacpp:tokens_predicted_seconds_total   4,230.47
llamacpp:n_tokens_max                     153,562
llamacpp:prompt_tokens_seconds            322.685
llamacpp:predicted_tokens_seconds         24.0673
llamacpp:requests_processing              0
llamacpp:requests_deferred                0
```

That's where the "25 tok/s over 151k context" framing comes from. The largest observed token working set was 153,562 tokens, and average generation throughput was 24.1 tok/s. The screencast adds the power angle: generation sitting under 100W.

Post-run sensor baseline, for context only:

```text
amdgpu-pci-f400
edge: 30.0 C
PPT:  11.01 W
sclk: 600 MHz
```

That idle number isn't the under-load claim — that comes from the screencast. The sensor block is here just to show the box drops back to low idle once the run is done.

## The model config

This is the current production preload on `artemis`:

```yaml
qwen3.6-35b-q8:
  proxy: http://127.0.0.1:5804
  cmd: >
    ${llama} ${gpu}
    -m ${models}/qwen3.6-35b-a3b/Qwen3.6-35B-A3B-UD-Q8_K_XL.gguf
    -c 524288 -np 3 --kv-unified
    --cache-type-k q8_0 --cache-type-v q8_0
    -b 4096 -ub 4096
    --cache-ram 16384
    --cache-idle-slots
    --slot-prompt-similarity 0.8
    --mlock
    --reasoning on
    --reasoning-budget 65536
    --ctx-checkpoints 256
    --checkpoint-every-n-tokens 2048
  ttl: 0
```

And the sanitized GPU macro:

```yaml
gpu: >
  --port ${PORT} --host 0.0.0.0
  -ngl 999 -fa on --no-mmap -fit off
  -t 16 -tb 32
  --prio 2 --prio-batch 3
  --poll 100
  --jinja --metrics
```

The live process line matches the config:

```text
/home/key/llama.cpp/build/bin/llama-server
  --port 5804
  -ngl 999
  -fa on
  --no-mmap
  -t 16 -tb 32
  -m /home/key/models/qwen3.6-35b-a3b/Qwen3.6-35B-A3B-UD-Q8_K_XL.gguf
  -c 524288
  -np 3
  --kv-unified
  --cache-type-k q8_0
  --cache-type-v q8_0
  -b 4096
  -ub 4096
  --cache-ram 16384
  --cache-idle-slots
  --slot-prompt-similarity 0.8
  --mlock
  --reasoning on
  --reasoning-budget 65536
  --ctx-checkpoints 256
  --checkpoint-every-n-tokens 2048
```

The server reports:

```text
model_alias: Qwen3.6-35B-A3B-UD-Q8_K_XL.gguf
total_slots: 3
n_ctx: 262144
build_info: b8890-8bccdbbff
endpoint_metrics: true
```

The launch uses a 524,288-token unified KV pool with three slots. The reported request context is 262,144 tokens, and the largest observed live run so far is 153,562 tokens.

## Why this works

Four pieces have to line up for this to work.

The model is a MoE/hybrid-attention architecture. Qwen3.6-35B-A3B has a decent total parameter count but only a smaller active set per token. That keeps generation in the range where an APU can be useful instead of turning every token into a memory bandwidth wall.

Strix Halo has enough unified memory to run the reference-ish quant. I'm running `UD-Q8_K_XL` for the daily chat tier right now. Lower quants are absolutely interesting, and prior local eval notes make `UD-Q3_K_XL` look worth testing, but Q8 is the stable reference while there's room for it.

KV cache size isn't just "context length times model size." Model architecture matters. Hybrid attention models can keep long contexts practical because not every layer needs the same full KV footprint as a conventional dense-attention model. That's the difference between "the model loads" and "the model is still useful at 150k tokens."

Vulkan/RADV matters on this box. This isn't a ROCm setup. The important local fix is enabling RADV's unified heap behavior for the APU:

```xml
<driconf>
  <device driver="radv">
    <application name="Default">
      <option name="radv_enable_unified_heap_on_apu" value="true"/>
    </application>
  </device>
</driconf>
```

Without that, you can hit split-heap behavior that makes large-model residency more fragile on APUs. With the unified heap, the machine behaves much more like the hardware promise: one big memory pool shared by CPU and GPU.

## Loading, quantization, and what the visuals are showing

The companion visual explainer started from a video on inference loading and quantization. Four ideas pull most of the weight here: loading is a memory hierarchy problem, quantization is compression with quality tradeoffs, K-quants preserve more local structure than a flat int4 story suggests, and salient-weight methods try to spend precision where the model is most sensitive.

That maps directly onto this homelab setup.

Model loading isn't just "open a file and go." The weights live on NVMe, get staged through host memory, then become resident for the inference backend. On `artemis`, I use `--no-mmap` and `--mlock` because this machine has the unified memory to hold the loaded model directly and I care more about predictable residency than minimizing initial RAM pressure. On `voyager`, the 7900 XTX setup uses a different path because it has discrete VRAM and ROCm.

Quantization is the reason any of this fits at all. A full precision 35B model isn't the target here. The practical question is which quant preserves enough quality while leaving enough memory for KV cache, compute buffers, and idle-session cache. On `artemis`, Q8 is currently viable. On `voyager`, the 24GB VRAM envelope forces a much more aggressive quant for the always-on cron tier.

The K-quant and salient-weight story matters because quality doesn't degrade uniformly. Some weights are more important than others, and some layers tolerate compression better than others. That's why the real tuning loop isn't "pick the smallest GGUF." It's: pick a quant, run the actual prompts, look at pass rate and token behavior, then decide whether the memory saved is worth the loss.

## Lessons so far

The boring service shape matters as much as the benchmark number. The model is behind llama-swap, the endpoint is OpenAI-compatible, the service is user-systemd managed, and the config is tracked in the homelab repo. That means I can restart it, inspect it, diff it, and route real applications to it without a pile of manual launch commands.

Long context is a system property, not a model-card checkbox. You need the model architecture, the quant, the KV settings, the backend, the memory behavior, and the scheduler to line up. A model that technically advertises 262k context can still be useless if prompt processing crawls, the backend OOMs, or a second request evicts the wrong state.

APUs are more interesting than old GPU mental models give them credit for. On raw bandwidth alone, Strix Halo loses to a top discrete GPU every time. Look at the whole small-box deployment — memory capacity, idle power, heat, noise, enough token speed to keep a chat loop interactive — and it starts to make sense.

The result isn't magic. Just a good hardware/software fit: Strix Halo unified memory, RADV, llama.cpp, Qwen3.6-35B-A3B, a quant that fits comfortably, and a service layer that keeps it useful.

For my workflow, that's the bar. I don't need this machine to win an internet benchmark. I need it to sit in the corner, hold a huge working context, answer at a speed that still feels interactive, and stay cheap enough to leave on.

Right now, it does.
