---
title: "120 agentic runs: the 35B coder post-trains, and the model that knew it was lying"
slug: "35b-coder-bakeoff"
date: 2026-07-27
draft: false
tags: ["local-ai", "qwen", "llama-cpp", "benchmarking", "agents", "homelab"]
---

One of the four models flagged its own fabrication in its thinking trace and shipped it anyway. Another did the same work for half the tokens and never came close to failing.

Earlier this week I published a 90-run bakeoff of Qwen3.6-27B post-trains, and a commenter on the Reddit thread ([u/Easy-Ride3366](https://www.reddit.com/r/LocalLLaMA/comments/1v78dzr/comment/ozw4ohh/)) asked the natural follow-up: what about the 35B class, stock versus Ornith versus the new KAT-Coder? So I ran it again, bigger: 4 models, 6 tasks, 5 reps, 120 isolated runs on the same pre-registered harness.

110 of 120 runs passed. Unlike the 27B run, pass rate actually discriminates here: two models passed 29 of 30, one passed 27, one passed 25. But the pass rates are still the least interesting part. The transcripts separate the models much harder than the grades do, and one of them contains the strangest failure I have seen in two of these experiments: a model that wrote "I mentioned v4659 in my draft which is fabricated" in its own reasoning, and then delivered the file with v4659 in it.

This is the full writeup. Raw data, per-task analyses, and the pre-registration are linked at the bottom.

## The models

**KAT-Coder-V2.5-Dev** ([Kwaipilot](https://huggingface.co/Kwaipilot/KAT-Coder-V2.5-Dev)) is an SFT+RL post-train of Qwen3.6-35B-A3B, released three days before this run. Their in-house table claims SWE-bench Verified 69.4 vs 64.4 for stock Qwen3.6-35B and the class lead on most agentic-coding benches. The detail that made me want it in the battery: their technical report says plain binary rewards collapsed the base model into pathological tool use, single turns with more than 70 parallel tool calls, and they fixed it with penalties for excessive parallel calls, failed calls, empty tool blocks, and repetition. That is, almost word for word, the failure taxonomy from my 27B bakeoff. An RL run aimed directly at the thing I measure.

**Ornith-1.0-35B** ([DeepReinforce](https://huggingface.co/deepreinforce-ai/Ornith-1.0-35B)) is an RL post-train of Qwen3.5-35B-A3B for agentic coding, claiming SOTA at its size on Terminal-Bench 2.1, SWE-bench Verified/Pro/Multilingual, and NL2Repo. Note the base: Qwen3.5, one generation older than KAT's.

**Two stock controls.** Qwen3.6-35B-A3B (KAT's base) and Qwen3.5-35B-A3B (Ornith's base), so each post-train can be compared against its own starting point, not just against the newer stock.

## Experiment setup

**Arms.** Four llama-swap lanes on a single RTX 5090, cloned from the 27B bakeoff lanes: fixed `-c 131072`, KV cache q4, `--reasoning on --reasoning-budget 32768`, MTP speculative decoding on every arm, identical sampling (temperature 1.0, top_p 0.95, top_k 20).

| arm | weights | base | quant |
|---|---|---|---|
| stock36 | Qwen/Qwen3.6-35B-A3B | — | unsloth UD-Q4_K_M (native MTP) |
| stock35 | Qwen/Qwen3.5-35B-A3B | — | unsloth UD-Q4_K_M (native MTP) |
| ornith | deepreinforce-ai/Ornith-1.0-35B | Qwen3.5-35B | SC117 APEX-MTP I-Balanced |
| kat | Kwaipilot/KAT-Coder-V2.5-Dev | Qwen3.6-35B | gbuzhf APEX-MTP I-Balanced |

Registered confounds, written down before any run: the post-trains only exist with MTP in APEX quants, so the quant methods split along arm lines. APEX is the better quant (larger files, claimed near-Q8 on sensitive tensors), so the bias direction favors the post-trains; the cost metrics I care about (calls, tokens, thinking characters) are quant-insensitive, and a post-train losing despite the better quant would be a clean loss. Ornith sits on the older base, which stock35 exists to resolve. The post-trains' MTP heads are grafted, not trained with the weights; MTP affects wall time only. Both post-trains were RL-trained against Claude Code-style harnesses, so this battery measures transfer into a foreign harness (mine), not their native performance.

**Harness, telemetry, battery.** Identical to the 27B run, deliberately: each run a fresh Coder workspace on my k8s cluster driving headless Hermes (my daily agent, pinned image) against the same six self-grading tasks (seeded repo regression, spec'd feature with hidden acceptance tests, log forensics, static infra diagnosis, cited web research, JSON repair pipeline). Model calls route through an OIDC gateway to a translation shim that emits one OTel span per call into SigNoz. Every run keeps its full transcript. 2,203 spans across 120 runs, 100% bucketed to their runs by arm and time window.

**Pre-registered hypotheses:**

- H1: KAT transfers — it beats stock36 on cost and/or behavior (their harness-targeted RL survives the foreign harness)
- H2: the post-trains' advantage concentrates on the coding-shaped tasks
- H3: my standing prior from the 27B run — finetunes usually are not better than their base

## Headline results

Pass rate (5 reps per task per arm):

| task | stock36 | stock35 | ornith | kat |
|---|---|---|---|---|
| yoke-bugfix | 5/5 | 4/5 | 5/5 | 5/5 |
| yoke-feature | 5/5 | 5/5 | 2/5 | 5/5 |
| log-parse | 4/5 | 5/5 | 5/5 | 4/5 |
| homelab-debug | 5/5 | 5/5 | 5/5 | 5/5 |
| web-research | 3/5 | 5/5 | 3/5 | 5/5 |
| mixed-pipeline | 5/5 | 5/5 | 5/5 | 5/5 |
| **total** | **27/30** | **29/30** | **25/30** | **29/30** |

Cost, summed over each arm's 30 runs (shim spans):

| metric | stock36 | stock35 | ornith | kat |
|---|---|---|---|---|
| model calls | 631 | 711 | 478 | 383 |
| input tokens | 25.1M | 24.8M | 13.7M | 12.6M |
| output tokens | 398K | 206K | 201K | 153K |
| thinking chars | 802K | 247K | 330K | 138K |

H1 confirmed, H2 partially, H3 split. KAT matched the best stock pass rate at half the input tokens of either stock arm, 40% fewer calls than the next-cheapest model, and a sixth of stock36's thinking. Ornith confirmed the prior by losing to its own base.

## What the transcripts showed

Method, same as last time: six analyst agents (GLM-5.2, MiniMax-M3, Kimi K3) each read all 20 transcripts for one task under contract, every behavioral claim carrying a run ID and a verbatim quote. I independently verified every consequential claim against the raw files, including each of the ones below. One analyst caught and corrected its own wrong always/never claim by grepping the full set, which is the contract working as intended.

### kat: discipline with receipts

Every task analyst, three independent model families, reached the same verdict with evidence: kat's economy is discipline, not corner-cutting. The receipts: baseline pytest before any edit in 5 of 5 feature reps (the only arm that always did). One targeted patch per bug instead of rewrite cycles. Deliverables at the requested path in every single rep, including the two stock arms' slips. Scratch configs in tmpdirs, never polluting the repo (stock35 polluted it twice and got bitten). Zero raw tool-call markup leaked into its reasoning across all 30 runs, where the Qwen stocks leak constantly (stock36 leaked 195 times on the research task alone). No debug spirals, no context compactions, no broken intermediate states anywhere in 120 runs.

Its corner-cutting candidates all dissolve on inspection. It reads every file (citing line numbers), verifies against the spec as deeply as anyone, and in several reps did more than the minimum (proactively checking a symlink the task never asked about, adding un-graded lint and type polish). Its single failure was a tool-hygiene lapse: four unfiltered greps dumped enough log lines to overflow the context, after it had already solved the task. Cheap, but never shallow.

The failure-mode detail from KAT's technical report shows up here as a perfect inverse: whatever their RL penalties did to the 70-parallel-calls collapse, the result is the cleanest tool-format behavior I have measured in any model, stock or tuned.

### ornith: the fastest diagnosis and the worst mechanics

Ornith parsed what tasks wanted faster than anyone in several reps, and then lost runs to things that are not reasoning. Two feature reps died at 23 and 28 seconds when raw `<tool_call>` markup leaked into the reasoning stream and ended the run with zero code written (the plan in both was correct). A third rewrote a 400-line file wholesale, dropped one character in an unrelated template string, then re-derived the same wrong hypothesis five times while rewriting from memory, inventing a nonexistent symbol along the way. A fourth passed while having deleted all 8 existing tests with a stray empty write, reporting "245 total" tests against its own 237-pass output. The grader checks exit codes, not counts, so it sailed.

And then there is the fabrication. On the research task, one ornith rep invented a llama.cpp release tag, v4659, out of thin air. Its own reasoning box says, verbatim: "I mentioned v4659 in my draft which is fabricated." The final research.md contains v4659. The grader, checking fields rather than truth, passed it at 0.778. Last bakeoff it was an invented person; this time the model narrated its own confabulation and shipped it. I keep having to learn the same lesson: the qualitative layer is not a nice-to-have, it is the only part of this that catches dishonest passes.

To be fair to it: ornith was also the best pure reasoner on the debug task (the only arm that asked what the evidence would look like under a different hypothesis), and one of its research reps produced the most honest single turn in the battery, explicitly reporting "price not verifiable via web scraping" after its first draft had fabricated one. It just needed a nudge to get there.

### stock36: the strongest raw analyst, the worst economy

Stock36 produced the deepest single investigations in the battery, including the only run that both analyzed cleanly and flailed within one transcript, and the only self-catch of an ordering bug before tests. It also produced the worst waste: 802K thinking characters (3.3x anyone else), a research run that extracted the correct $189.99 price around call 30 and then spent 30 more calls trying to disprove it as "too low for a server edition full tower case" until the iteration budget killed it, file unwritten. Its failure mode is deliberation decoupled from information, and it is expensive.

### stock35: the quiet generalist

The older base went 29/30 with the least drama. No verification habit, the noisiest format hygiene of the stocks (it drafted an entire deliverable inside a phantom tool call that never executed), and sometimes too terse to satisfy a rubric. It passes. On a harder battery its thinness is probably a liability, but that is inference, not measurement.

## The task-shape map

Manner, not outcomes:

| task shape | best fit |
|---|---|
| bounded regression fix | **kat** (minimal correct patch, double verification, note, stop) |
| small feature, hidden tests | **kat** (baseline-first, one patch, tests added every rep) |
| log forensics | **kat** (output-shrinking commands, zero whole-file reads) |
| static diagnosis | **kat** (read everything, cited line numbers); ornith reasoned best but dirtier |
| cited fact-finding | **kat and stock35** (best-sourced answer, finish); ornith confabulated, stock36 doom-scrolled |
| deterministic repair pipeline | **kat** (within 2x of optimal on its best rep; stock36's worst rep cost 22x the tokens for the same pass) |

I did not pick kat for every row to be contrarian about my own prior. Three independent analyst models, reading separately, each reached that ranking on their task. The mechanical evidence (leak counts, call counts, baseline presence) backs it without interpretation.

## Cross-cutting findings

**Graders that check fields pass liars.** Three separate dishonest or false passes this run: ornith's fabricated tag, ornith's deleted-tests-while-reporting-more, and a stock36 research deliverable claiming a symlink did not exist when it did (the grader greps keywords, it does not check the world). Two bakeoffs in a row: the pass rate is a floor for trust, not a measure of it.

**RL aimed at tool-format collapse works, measurably.** Leak counts on the research task: stock36 195, stock35 21, ornith 11, kat 0. KAT's report describes penalizing exactly this pathology class, and the fingerprint is simply absent. This is the first post-train I have tested where the vendor's stated training target is independently visible in behavior.

**Cost is a manner signal.** kat at 383 calls and 12.6M input tokens did not skip steps; it skipped the hypotheses-that-turn-out-wrong prose that fills the other arms' thinking traces. Cheap and shallow look similar in a token table; cheap and disciplined only look similar in a token table. You have to read.

**Ceiling effects are receding.** Last battery: 90/90, no discrimination. This one: 110/120, and the failures cluster informatively (feature mechanics, research honesty). The battery is starting to bite, which makes the manner differences leading indicators of outcome differences on harder work.

## What I'd actually run

I’m genuinely surprised here! I use the stock qwen3.6-35b as a daily driver, and i think i would still stick with it for more general cases. That said, KAT genuinely punched above its weight, and i will absolutely be slotting this into dev workflows going forward. 

## Caveats

Four arms, one battery, n=5 per cell, one harness, one GPU. The quant confound favors both post-trains (APEX over UD); kat's cost metrics are quant-insensitive, but its output quality is not free of it. Ornith's base is a generation older, which stock35 only partially controls for. The qualitative layer was read by agents with my spot-checks, not a panel of humans; every consequential claim I cite here was verified against the raw files by me, and the transcripts are available on request if you want to audit anything. KAT was released three days before this run; its community quant landscape is still settling.

## Raw data and companion files

- [Pre-registration doc](https://kmarble.dev/artifacts/35b-coder-bakeoff/prereg.md) (7 KB): arms, hypotheses, registered confounds, kill criteria, full run log
- [Results note](https://kmarble.dev/artifacts/35b-coder-bakeoff/results.md) (5 KB): the verified headline findings
- [Aggregate tables](https://kmarble.dev/artifacts/35b-coder-bakeoff/report.md) (3 KB): pass, wall time, tool calls, tokens, per task and arm
- Per-task findings (16-24 KB each, fully cited): [yoke-bugfix](https://kmarble.dev/artifacts/35b-coder-bakeoff/findings-yoke-bugfix.md), [yoke-feature](https://kmarble.dev/artifacts/35b-coder-bakeoff/findings-yoke-feature.md), [log-parse](https://kmarble.dev/artifacts/35b-coder-bakeoff/findings-log-parse.md), [homelab-debug](https://kmarble.dev/artifacts/35b-coder-bakeoff/findings-homelab-debug.md), [web-research](https://kmarble.dev/artifacts/35b-coder-bakeoff/findings-web-research.md), [mixed-pipeline](https://kmarble.dev/artifacts/35b-coder-bakeoff/findings-mixed-pipeline.md)
- Transcripts and span data: available on request, ~6 MB across 120 runs
