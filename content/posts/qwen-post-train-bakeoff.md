---
title: "90 agentic runs, zero failures, and one invented person"
slug: "qwen-post-train-bakeoff"
date: 2026-07-26
draft: false
tags: ["local-ai", "qwen", "llama-cpp", "benchmarking", "agents", "homelab"]
---

All 90 runs passed. One of them still shipped a made-up person.

Last week a [comment on r/LocalLLaMA](https://reddit.com/r/LocalLLaMA/comments/1v6jlva/deepseek_v4_flash_hy3_or_is_qwen36_27b_still_the/ozr19qt/) claimed that ThinkingCap and Fable Fusion, two community post-trains of Qwen3.6-27B, "really do beat the OG" for agentic work. I have a 5090, a Kubernetes cluster, and an agent harness, so I ran the experiment: 6 tasks, 5 reps, 3 models, 90 isolated runs, everything pre-registered.

Every single run passed its grader. If I had stopped at pass rates and token counts, the takeaway would have been a shrug: the models are equivalent, pick whichever. Then I read all 90 transcripts, and the shrug fell apart. The models behave so differently that the equivalence looks like an artifact of what we measure. One model verified its own work more than the others and also swapped one maintainer's identity for a completely different real person's, cited the real GitHub profile as the source, and passed the grader anyway.

This is the full writeup: the setup, the eval design, the numbers, and what the transcripts actually showed. Raw data and per-task analyses are linked at the bottom.

## The claim under test

The comment (from u/Important_Quote_1180, in a "best agentic model" thread):

> I feel like bang for your buck it's 27b all day. Thinkingcap and Fable Fusion are two well done post trained that really do beat the OG.

Two specific post-trains:

**ThinkingCap** ([bottlecapai/ThinkingCap-Qwen3.6-27B](https://huggingface.co/bottlecapai/ThinkingCap-Qwen3.6-27B)) is a deliberately minimal finetune of Qwen3.6-27B advertising "capability of Qwen3.6-27B with 50% less thinking tokens on average, and over 90% less in best cases." Their own card is more careful than the Reddit claim: macro-average accuracy 81.5 vs 80.7 (within noise), thinking tokens down 45.8% on average, and on their agentic eval (Claw-Eval) accuracy dips from 87.0 to 84.4 with thinking down 25.2% per task. So the vendor's own agentic number predicts a smaller efficiency gain and a small accuracy cost, which is exactly the kind of thing worth checking independently.

**Fable Fusion 711** ([DavidAU's repo](https://huggingface.co/DavidAU/Qwen3.6-27B-Fable-Fusion-711-Uncensored-Heretic-NM-DAU-NEO-MAX-MTP-GGUF)) is a multi-stage uncensored/heretic creative-plus-coder merge. The card leads with "the first fine tune to exceed 700 ARC-C in both 8 bit and 4 bit" and claims it beats the base model on 6 of 7 benchmarks. It is a creative-writing-targeted merge that also codes, not a coding-targeted post-train. My pre-registered expectation was that it would be measurably worse at disciplined agent work. That expectation was wrong in an interesting way.

## Experiment setup

**Arms.** Three llama-swap lanes on a single RTX 5090, cloned from my production Qwen3.6-27B lane with one deliberate change: a fixed `-c 131072` context for every arm instead of `--fit`, so context size couldn't vary between arms.

| arm | weights | quant |
|---|---|---|
| stock | Qwen/Qwen3.6-27B | unsloth UD-Q4_K_XL |
| thinkingcap | bottlecapai/ThinkingCap-Qwen3.6-27B | protoLabsAI Q4_K_M-MTP |
| fable | DavidAU Fable-Fusion-711 | NEO-MTP Q4_K_M |

Serving: llama.cpp (b9986), CUDA, `-ngl 999`, flash attention on, KV cache q4, `--reasoning on --reasoning-budget 32768`, batch 2048/1024, identical sampling for all arms (temperature 1.0, thinking enabled). MTP speculative decoding where the quant ships a draft head.

Registered confounds, written down before any run: the quant schemes are not byte-identical (UD-Q4_K_XL is a slightly better quant than Q4_K_M, so any bias direction is mildly pro-stock; if a tune beats stock it does so despite that). MTP head quality differs per tune, but that only affects speed, not output distribution. Fable is refusal-ablated and none of my tasks probe refusals.

**Harness.** Each run is a fresh Coder workspace on my k8s-lab cluster (k0s, single node) running headless Hermes (the same agent I use daily, pinned container image). Model calls route from the workspace through an OIDC gateway to a translation shim, then to the experiment lane on the 5090. One workspace per run, nothing shared between runs, task materials staged by a setup script before the agent starts.

**Telemetry.** The shim emits one OTel span per model call into SigNoz: tokens in/out, thinking characters, tool calls, time to first token, duration. The gateway logs every request with workspace attribution. Tetragon captures every exec the agent runs. And each run keeps its full transcript, the literal Hermes session output, reasoning blocks and tool calls and all. 1,202 model-call spans across the 90 runs, bucketed per run by arm and time window.

**Pre-registered hypotheses.** Written into the experiment doc before the first run:

- H0: no difference in task success rate across arms
- H1: ThinkingCap beats stock on tool-call consistency and turns-to-green
- H2: Fable is worse than stock on tool-call consistency (the Reddit claim fails for agentic work specifically)
- H3: ThinkingCap uses fewer thinking tokens per completed task than stock

## Evaluation setup

Six self-grading tasks, chosen to cover coding and non-coding agent work. Self-grading means the primary metric has no human judgment in it: pytest suites, regex checklists against ground truth, file validation. Every task ships a prompt, a setup script, and a grader that emits one JSON verdict.

1. **yoke-bugfix.** A seeded regression in yoke, my own private repo (zero training-data contamination, which matters for a clean signal). The suite was green at v1.4.1; one test now fails. Find it, fix the source, don't touch the tests. Grade: full suite green.
2. **yoke-feature.** Implement `yoke model ls --json` against a spec, with hidden acceptance tests the grader copies in at grade time. Grade: acceptance plus regression suite.
3. **log-parse.** An 11K-line synthetic multi-service log with three planted anomalies (an error burst, a latency regression with a warning signature, a silent retry loop). Ground truth in an answers file. Grade: required-findings checklist.
4. **homelab-debug.** A static filesystem capture of a broken nginx-to-gunicorn deployment. Write the root cause. Grade: keyword match plus a no-destructive-actions check.
5. **web-research.** Three questions with verifiable answers (latest llama.cpp release and two notable changes; who maintains llama-swap and what it solves; the price of a specific PC case), researched through a local SearXNG instance. Grade: required-facts checklist plus citation presence.
6. **mixed-pipeline.** Repair a broken JSONC-ish config (comments, single quotes, trailing commas, env-var placeholders) with a stdlib-only Python script, run it, match a strict spec. Grade: end-state file checks.

Five reps per task per arm, 90 runs total, executed arm-blocked (all 30 runs of one arm before the next) to avoid model-swap thrash on the GPU.

**What went wrong, honestly.** The first full pass produced 71 of 90 clean runs. The failures were all infrastructure, not model behavior: the gateway's default 120-second upstream timeout was amputating long thinking turns (a real production bug I fixed at 600s as a result), llama-swap wedged its event loop four times (TCP accepts but handlers never read; a watchdog restart cures it instantly, root cause still open), and an orchestrator bug mangled one batch of workspace names. Every failed run was re-run after the fix, and only the makeup-era runs count in what follows. I'm noting this because "90/90" without the makeup-pass context would be dishonest about how the sausage got made.

## Headline results

90 of 90 valid runs passed. H0 confirmed: at this battery's difficulty, success rate does not discriminate these models at all. The signal is entirely in cost and behavior.

Median wall time per run (seconds):

| task | stock | thinkingcap | fable |
|---|---|---|---|
| yoke-bugfix | 97 | 90 | 95 |
| yoke-feature | 154 | 136 | 168 |
| log-parse | 120 | 91 | 101 |
| homelab-debug | 35 | 35 | 39 |
| web-research | 127 | 92 | 158 |
| mixed-pipeline | 60 | 51 | 64 |

Totals across each arm's 30 runs (shim spans):

| metric | stock | thinkingcap | fable |
|---|---|---|---|
| model calls | 380 | 350 | 472 |
| input tokens | 10.87M | 8.74M | 13.36M |
| output tokens | 149.7K | 115.5K | 148.3K |
| thinking tokens (est) | 47.6K | 31.6K | 41.8K |
| tool calls | 550 | 497 | 574 |

Verdicts on the pre-registered hypotheses:

- **H3 confirmed.** ThinkingCap used 34% fewer thinking tokens than stock (31.6K vs 47.6K), 23% fewer output tokens, and was fastest on 5 of 6 tasks. Notably its per-call generation is slower (9.2s average vs stock's 6.0s); it wins by needing fewer turns. Also notably, 34% is much closer to the vendor's own agentic number (25.2%) than to their headline "50% less," which is a good reminder that benchmark-suite efficiency and agentic-loop efficiency are different things.
- **H2 rejected on outcomes.** Fable passed everything, including 5/5 on both coding tasks. But it made 24% more model calls than stock for identical results.
- **H1 not confirmed.** No tool-call derailments from anyone. The battery is too easy to separate consistency.

If I'd published at this point, the post would have been boring and slightly misleading. Pass rates and token totals make three genuinely different models look interchangeable.

## What the transcripts showed

Method for the qualitative layer: six analyst agents (GLM-5.2, MiniMax-M3, Kimi K3) each read all 15 transcripts for one task under a strict contract, every behavioral claim had to carry a run ID and a verbatim quote. I independently read 14 transcripts across four tasks and spot-verified every load-bearing claim, including the two most consequential ones (the confabulation and the 94-tool-call outlier) against the raw files myself. Three MiniMax analysts died to provider rate limits mid-wave and were relaunched on other models; one wrote a premature "complete" marker over placeholder sections before dying, which is its own lesson about trusting agent self-reports.

### stock: the methodical checklister

Stock plans upfront in numbered lists, explores broadly before editing, and is the most uniform model in the battery. On the bugfix task, 4 of 5 reps produced the byte-identical patch, and it was the only model that articulated the edge case out loud ("using `while` instead of `if` to handle consecutive reserved ports", and the test only reserves one port, so the distinction is invisible to the grader). It added the most tests on the feature task (29 across 5 reps), documented alternative fixes on the debug task (3 of 5 reps offer an Option A and Option B), and read its own output file back to verify it in 4 of 5 research runs.

Its failure mode is deliberation that decouples from information. On web-research, the task asks for "two notable changes" in the latest llama.cpp release, and that release (b10107) turns out to be a one-commit daily build. Stock re-reasoned the same dilemma four or five times per run without new data: one reasoning block contains five separate "Actually, wait" restarts, and one rep curled the identical Newegg URL five times with five different grep filters. It also argues with the harness: when a verification plugin nudged it to re-run an already-green test suite, stock's reasoning explicitly called the nudge redundant ("The system note seems to be a generic reminder from the plugin... No further action needed") and then complied anyway.

### thinkingcap: the terse operator, genuinely bimodal

The efficiency claim is behaviorally real. ThinkingCap's reasoning blocks are compressed and action-oriented, lowest thinking chars on 5 of 6 tasks, and it has habits the other models lack: it smoke-tests end-to-end with the actual binary (`python -m yoke model ls --json | python3 -c "import json; json.load(...)"`, catching stdout-purity bugs the in-process test runner would miss), and on the pipeline task it fixed a comment-stripper bug by reordering pipeline stages (convert quotes before stripping comments), eliminating the whole hazard class where the other two patched symptoms.

But the distribution has a fat tail, and this is the part the vendor's macro averages hide. ThinkingCap's best runs were the cheapest in the entire battery (one pipeline run: 38 seconds, first-try, double-validated). Its worst runs were the most expensive in the entire battery: one rep spiraled for 9.3K reasoning chars and 16 model calls hunting a one-line `str.split`/`join` bug, and another burned about 10 tool calls and 18.8K thinking chars chasing a phantom llama.cpp release tag (b10121) that a search snippet had hallucinated into existence, a tag stock dispatched with a single API call in a parallel run. Its efficiency is also prose density, not action economy: on the debug task it made the same number of tool calls as everyone else, just with 40% fewer words.

There were smaller inconsistencies too: 2 of 5 bugfix reps emitted an awkward double-mutation patch where stock was uniform, and one rep skipped adding any tests at all on the feature task.

### fable: the wide-roaming investigator with a confabulation tail

Fable is where the pass rate lies hardest. The aggregate read ("24% more calls, same results") makes it look like a sloppy worker. The transcripts say something stranger: fable is simultaneously the best investigator and the least trustworthy narrator in the battery.

The investigator side is real. On the debug task, fable was the only model that checked whether the broken nginx config was actually the live one (it listed the sites-enabled symlink in 4 of 5 reps; the others diagnosed the file on disk and assumed (correct here, silently wrong in any variant where the symlink is the actual bug)). One fable rep noticed its evidence set was incomplete and went back for the missing file, the only caught-and-recovered evidence gap in all 90 runs. On web-research it pulled the richest data of anyone: it parsed Newegg's embedded JSON state for variant-level pricing, identified llama-swap's maintainer by name via the GitHub users API, widened the changelog window to 44 commits and found genuinely notable changes the other models missed, and talked itself out of a $252.99 Klarna installment-price trap by pure reasoning ("That's not an actual retail price for the case!").

The narrator side is also real. Fable produced the battery's only factual error, and it's a doozy: one rep's research file states llama-swap is "Maintained by `mostlygeek` — Matthew Garrett, former Red Hat engineer." I double-checked the attribution before writing this, because it's the strongest claim in the piece. Matthew Garrett is a real person: he's mjg59, genuinely an ex-Red-Hat engineer, known for the Linux Secure Boot work. He has nothing to do with llama-swap. mostlygeek is Benson Wong (per the GitHub API: name "Benson Wong," Tailscale and Elethink), which a different fable rep verified directly from the users endpoint. So the model didn't invent a person from nothing; it fused two real open-source identities into one wrong one, then cited the real repo as the source for it. The confabulated rep never read its own file back (only 2 of 5 fable reps verify their artifacts, vs 4 of 5 for stock), so the wrong identity sailed into the deliverable, and the grader passed it, because the grader checks that the maintainer field contains "mostlygeek," not that the prose around it is true. The same model also fetched a guessed URL that existed nowhere, trusted a stale search snippet claiming the latest release was b8554, leaked literal `</parameter>` training artifacts into its thinking, and spent 94 tool calls on a single price question (fetching one retailer's search page six times) when its siblings used 20-30.

This is the creative-merge character in one sentence: richer when right, wrong when not, and you can't tell which from the score.

## The task-shape map

Manner, not outcomes, since everything passed:

| task shape | best-fit behavior |
|---|---|
| convergent single-cause bugfix | **stock** (uniform patches, explicit edge-case reasoning) |
| small feature with hidden tests | **thinkingcap** (clean first patches, spec-literal, smoke tests) |
| log forensics | **stock** (cross-service elimination, false-positive debugging) |
| static diagnosis | **fable** for investigation, **stock** for the artifact |
| shallow fact-finding with citations | **stock** (zero confabulation, verifies artifacts) |
| deterministic repair pipeline | tie; thinkingcap had the cleanest recovery style |

## Cross-cutting findings

**Keyword graders are blind to confabulation.** 90/90 pass and one run shipped an invented identity. Any benchmark that grades on field presence rather than factual accuracy can pass a model that's making things up. If I extend this battery, it gets a fact-check lane.

**ThinkingCap's efficiency has a fat tail.** Median case: cheapest and fastest, claim confirmed. Worst case: the two most expensive runs in the battery. If you adopt it for long agent loops, pair it with loop detection, because the anomalous sessions cost more than stock's, not less.

**Fable's extra calls buy real investigation on diagnostic and research shapes, and real waste on convergent shapes.** It is task-dependent, not a uniform tax.

**The harness's verification nudger burned 1-2 redundant test-suite runs on nearly every coding rep of every model.** All three models know it's redundant (all three say so in their reasoning, in those words) and all three comply anyway. That's a harness bug masquerading as a model behavior, and it's now on my fix list.

**Wall-time outliers are mostly infrastructure.** API retries and model cold-loads, not model dithering. Read aggregate timing tables accordingly.

**Shared blind spots across all three models.** Six of 15 pipeline reps wrote a brittle `'//' not in text` verification check that fails on URLs, then had to re-explain to themselves that `http://` is fine. All three models doubted the 3-hour latency regression ("seems too broad") before accepting it. Stock and fable hit the mawk/gawk `match()` three-arg incompatibility; thinkingcap's awk idiom sidestepped it entirely.

## What I'd actually run

I would stick with the base qwen model. If anything, this experiment solidified my thought that finetunes often aren't better than their base.

I might consider thinking cap as an actually useful finetune, but only in instances where latency and speed need to be balanced. If your hardware can serve the base qwen model fast enough for you, then I see little need for thinkingcap.

## Caveats

One battery at one difficulty, n=5 per cell, one quant class, one agent harness, one GPU. Ceiling effects everywhere: the tasks were solvable by all three models, so any capability gap is hidden. The manner differences documented here are leading indicators: on a harder battery (multi-file features, ambiguous incidents, adversarial inputs) confabulation and tail behavior become outcome differences, not curiosities. The quant confound is registered above. And the qualitative layer, while quote-verified, was read by agents with my spot-checks, not by a panel of humans; the raw transcripts are linked below if you want to audit any claim yourself.

## Raw data and companion files

- [Pre-registration doc](https://kmarble.dev/artifacts/qwen-post-train-bakeoff/preregistration.md) (5 KB): hypotheses, arms, registered confounds, kill criteria, written before any run
- [Headline results note](https://kmarble.dev/artifacts/qwen-post-train-bakeoff/results.md) (3 KB): the aggregate pass/time/token analysis
- [Full behavioral analysis](https://kmarble.dev/artifacts/qwen-post-train-bakeoff/behavioral-analysis.md) (11 KB): per-model profiles, every quote cited by run ID
- Per-task findings (15-28 KB each, fully cited):
  - [yoke-bugfix](https://kmarble.dev/artifacts/qwen-post-train-bakeoff/findings-yoke-bugfix.md), [yoke-feature](https://kmarble.dev/artifacts/qwen-post-train-bakeoff/findings-yoke-feature.md), [log-parse](https://kmarble.dev/artifacts/qwen-post-train-bakeoff/findings-log-parse.md), [homelab-debug](https://kmarble.dev/artifacts/qwen-post-train-bakeoff/findings-homelab-debug.md), [web-research](https://kmarble.dev/artifacts/qwen-post-train-bakeoff/findings-web-research.md), [mixed-pipeline](https://kmarble.dev/artifacts/qwen-post-train-bakeoff/findings-mixed-pipeline.md)
- Transcripts and span data: available on request, ~4.6 MB of transcripts across 90 runs
