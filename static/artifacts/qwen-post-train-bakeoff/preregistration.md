
# 27B Post-Train Bakeoff — Pre-Registration

Agent-authored experiment contract (human review welcome, #ingest). Written BEFORE any model runs. Frozen inputs, explicit hypotheses, kill criteria.

## Question

Do the community post-trains **ThinkingCap** (bottlecapai) and **Fable Fusion 711** (DavidAU) beat **stock Qwen3.6-27B** for agentic work in the author's actual usage pattern — Hermes-style multi-turn tool use — across coding and non-coding agent tasks?

Origin: r/LocalLLaMA claim (2026-07-25, u/Important_Quote_1180) that "Thinkingcap and Fable Fusion are two well done post trained that really do beat the OG."

## Hypotheses (registered before any run)

- H1: ThinkingCap > stock on tool-call consistency and turns-to-green. (Targeted "token-efficient thinking" tune; plausible.)
- H2: Fable Fusion < stock on tool-call consistency. It is a multi-stage uncensored/heretic creative+coder merge, not a coding-targeted post-train; the Reddit claim fails for agentic work specifically.
- H3: ThinkingCap uses fewer thinking tokens per completed task than stock (its advertised token-efficiency claim is directly measurable in traces).
- H0: no difference in task success rate across arms.

If results contradict H2 (Fable matches/beats stock on consistency), that is a genuinely interesting finding and gets reported as such.

## Arms (3 primary + 1 reference)

All primary arms: 4-bit-class GGUF, llama.cpp draft-mtp speculative decoding where the file supports it, server flags otherwise identical to the production stock lane.

| Arm | Weights | Quant file |
|-----|---------|------------|
| stock | Qwen/Qwen3.6-27B | unsloth UD-Q4_K_XL (already local) |
| thinkingcap | bottlecapai/ThinkingCap-Qwen3.6-27B | protoLabsAI Q4_K_M-MTP |
| fable | DavidAU Fable-Fusion-711 | NEO-MTP-Q4_K_M |
| stock-nvfp4 (reference) | Qwen/Qwen3.6-27B | production NVFP4-MTP lane, as deployed |

Registered confounds:
- Quant scheme is not byte-identical across primary arms (UD-Q4_K_XL vs Q4_K_M vs NEO Q4_K_M). Direction of bias is mildly pro-stock (XL raises some tensors); if a tune beats stock it does so despite a slightly better stock quant. Noted, accepted.
- MTP head quality differs per tune and affects SPEED secondaries only; spec decoding does not change output distribution, so quality primaries are unaffected. Spec acceptance rates recorded as telemetry.
- Fable Fusion is refusal-ablated ("heretic"). None of the battery tasks probe refusals; noted as scope limit.

Fixed across arms: context size, KV q4 cache, temperature 1.0, thinking enabled, reasoning budget, agent (Hermes, pinned image digest), prompts, task repos (pinned commits), tools available.

## Battery (6 tasks, frozen, self-grading)

Substrate mix per the author 2026-07-25: yoke (private repo — zero training-data contamination) + non-coding agent tasks.

1. **yoke-bugfix** — seeded bug in yoke (private, uncontaminated), failing pytest; agent must reach green. Grade: tests pass.
2. **yoke-feature** — multi-file feature with hidden acceptance tests. Grade: tests pass + diff touches expected modules.
3. **log-parse** — synthetic multi-MB service log with planted anomalies (error burst, latency step-change, one silent retry loop). Agent writes findings file. Grade: required-findings checklist.
4. **homelab-debug** — fully simulated broken service inside the workspace (container that crashloops + logs + compose file). Agent writes root-cause file. Grade: root-cause keyword match + no destructive actions.
5. **web-research** — research question with a verifiable answer set, agent uses SearXNG via curl. Agent writes cited summary. Grade: required-facts checklist + citation presence.
6. **mixed-pipeline** — multi-step: parse a provided log bundle, identify fault, write a small fix script, run it, verify output. Grade: end-state file checks.

Reps: k=5 per task per arm = 90 primary runs (+30 reference-arm runs if time allows). Execution arm-blocked (all tasks for one arm, then next) to avoid llama-swap model-swap thrash; task order within arm fixed; lane slots flushed between reps.

## Harness

Each run = one Coder workspace (workspace-otel base image) on k8s-lab, Hermes inside, model calls routed: Hermes → claude-apps-gateway (minted JWT) → anthropic-shim → llama-swap experiment lane on Voyager.

Trace capture (the point of using Coder):
- SigNoz: per-workspace OTel spans (Hermes tool calls), gateway structured inference logs (sub/model/upstream/status/ms per call).
- anthropic-shim: full request/response transcripts per run (thinking content included → thinking-token share measurable).
- Tetragon: every exec the agent runs.
- Grader output + run manifest per run, written to results dir on NFS.

## Metrics

Primary: task success rate (self-graded, no human judgment in the loop).
Secondary: turns-to-green, wall time, tokens in/out, malformed tool-call count, derailment events (repeated identical tool call >=4x, abandonment, off-task drift), thinking-token share, edits-to-green (coding tasks).

Analysis: per-run trace cards auto-generated from SigNoz + transcripts. Blinded rubric scoring of a 10% transcript sample (judge model + author spot-check). Report honest nulls; no post-hoc metric additions (exploratory observations clearly labeled exploratory).

## Kill criteria

- Any arm failing the 3-prompt smoke test (valid tool call, basic file edit, coherent answer) is dropped and reported, not patched.
- Telemetry gaps (missing token attribution, missing transcripts) halt the run at dry-run gate until fixed.
- GPU contention with the author's use: Artemis stays free for the author; if Voyager lane conflicts with anything the author needs, experiment yields.

## Phases

- P0 this doc. P1 model acquisition + llama-swap lanes. P2 battery build. P3 dry run (1 task x 1 rep x 3 arms) verifying traces end-to-end. STOP for the author go/no-go. P4 full run overnight(s). P5 analysis + writeup.

Working dir: ~/homelab/qwen-bakeoff (tasks, harness, results). Results writeup lands here when done.
