# 35B Coder Post-Train Bakeoff — Pre-Registration

Trigger: u/Easy-Ride3366 on the [27B bakeoff thread](https://www.reddit.com/r/LocalLLaMA/comments/1v78dzr/) asked for "comparison of the 35b — Stock vs Ornith vs KatCoder V2.5-Dev." the author committed to it publicly.

Follows the same design as the 90-run Qwen3.6-27B bakeoff (2026-07-25/26, [blog](https://kmarble.dev/posts/qwen-post-train-bakeoff/)) per `agentic-model-bakeoff` skill. Battery, harness, telemetry, and analysis layer are unchanged unless noted.

## Question

Do coding-targeted RL post-trains of the 35B-A3B MoE beat stock Qwen3.6-35B-A3B as a daily agent under Hermes — on cost, behavior, or both? the author's standing prior (from the 27B run): finetunes usually aren't better than their base.

## Arms (4, approved by the author 2026-07-26)

MTP-on-everything mandate → quant sources split by necessity (only APEX quants ship MTP for the post-trains). Registered as confound 1.

| Arm | Lane | Model | GGUF (size) | MTP | Base |
|---|---|---|---|---|---|
| stock36 | bakeoff-stock36 | Qwen/Qwen3.6-35B-A3B | unsloth UD-Q4_K_M (22.7 GB) | in-repo | — |
| ornith | bakeoff-ornith | deepreinforce-ai/Ornith-1.0-35B | SC117 APEX-MTP I-Balanced (26.0 GB) | grafted from 3.5-35B | Qwen3.5-35B-A3B |
| kat | bakeoff-kat | Kwaipilot/KAT-Coder-V2.5-Dev | gbuzhf APEX-MTP I-Balanced (26.1 GB) | grafted | Qwen3.6-35B-A3B |
| stock35 | bakeoff-stock35 | Qwen/Qwen3.5-35B-A3B | unsloth UD-Q4_K_M | in-repo | Ornith's control |

All are 35B-total / ~3B-active MoE → single RTX 5090, fixed `-c 131072`, `${kv-q4}`, `--spec-type draft-mtp` with identical spec params, flags cloned from the 2026-07-25 bakeoff lanes (which were cloned from production).

## Hypotheses

- **H1** (KAT transfers): KAT-Coder-V2.5-Dev beats stock on cost (thinking tokens, calls) and/or qualitative behavior on the battery. H0: no meaningful difference — post-train ≈ base under a foreign harness.
- **H2** (task-shape interaction): post-trains' advantage concentrates on coding-shaped tasks (yoke-bugfix, yoke-feature, mixed-pipeline) over research/forensics (web-research, log-parse, homelab-debug). H0: no task-shape interaction.
- **H3** (the author's prior, global): no post-train beats stock on net usefulness once behavior is read, not just graded. Null result reported honestly.

## Primary metrics (pre-registered, committed before results)

1. Pass rate per arm (grader is gate, not discriminator — 90/90 passed last time).
2. Cost: thinking tokens, output tokens, model calls, tool calls per run (shim spans).
3. Wall time per run (comparative only — see confound 2).
4. Qualitative layer (mandatory, per transcript-analysis-playbook): per-arm behavioral profiles, quote-or-no-claim, orchestrator spot-verification.

## Registered confounds

1. **Quant-method split (accepted by the author in exchange for MTP-on-everything)**: stocks at unsloth UD-Q4_K_M (~22.7 GB, imatrix dynamic), post-trains at APEX-MTP I-Balanced (~26 GB, MoE-aware mixed precision, claimed ≥ Q8 on sensitive tensors). Bias direction: favors post-trains (more bits + claimed-better method). Mitigations: cost metrics (tokens, calls, tool counts) are quant-insensitive; the behavioral layer reads manner, not just output quality; if a post-train LOSES despite the quant advantage, that result is clean and strong.
2. **Base-version mismatch**: ornith sits on Qwen3.5-35B-A3B; kat and stock36 on Qwen3.6-35B-A3B. The stock35 arm exists precisely to resolve ornith attribution — ornith-vs-stock35 is the clean finetune-vs-base test; ornith-vs-stock36 reads as "post-trained 3.5 vs newer base."
3. **MTP provenance**: ornith's MTP head is grafted from Qwen3.5-35B-A3B (not trained with the post-train); kat's is grafted similarly; stocks ship their native MTP. MTP affects wall-time only, not token/call metrics. Draft acceptance rates get logged per arm as a sanity check, not a metric.
4. **Identical sampling** (temp 1.0, top_p 0.95, top_k 20, presence_penalty 1.5, thinking on, reasoning budget 32768 — cloned from last run's lanes) over vendor-recommended settings (KAT t=1.0/p=0.95; Ornith t=0.6/top_k 20). We test "as configured in my harness," not vendor-tuned. Same stance as last run.
5. **Harness transfer, not native performance**: both post-trains were RL'd against Claude Code-style scaffolds; Hermes's toolset differs. A loss here doesn't contradict vendor SWE-bench numbers; it measures transfer.
6. **Text-only**: no mmproj; battery is text-only.

## Battery

Unchanged 6 tasks on `/data/bakeoff/tasks/`: yoke-bugfix, yoke-feature, log-parse, homelab-debug, web-research, mixed-pipeline. Keeping it frozen buys cross-size comparability for free (stock-27B vs stock-35B is a scaling datapoint). Graders re-verified against known-good/known-bad before the dry run.

## Scale

- 4 arms × 6 tasks × 5 reps = **120 runs** (approved by the author 2026-07-26).
- Sequential arm-blocking (llama-swap thrash containment), 1 task × 1 rep × all arms dry run first.

## Procedure (lessons applied)

1. Downloads to `~/docker/models` via `hf download` (one `--include` per file; verify shards on disk). ~60-80 GB against 216 GB free — fine, no cleanup needed.
2. Lanes `bakeoff-stock35|ornith|kat[|stock35-base]` in llama-swap config: fixed `-c 131072`, cloned production flags, MTP off. VRAM check: ~20 GB weights + 131k KV ≈ 26 GB on 32 GB — if tight, KV q8→q4 (context is not the lever).
3. `systemctl --user restart llama-swap`; verify `/v1/models`; smoke each lane checking BOTH content and reasoning_content (thinking model + small max_tokens = empty content, not an error).
4. Routing: shim `backends.yaml` + gateway `models:` + image bump + ArgoCD + pod restart. Verify CM content BEFORE trusting any 400 (gotcha 29). Probe a synthetic span and read it back from ClickHouse (dead-emitter lesson: 0.1.0/0.1.1 had emit_span defined but never called).
5. Dry run. Gate per run: transcript + grade.json + shim spans + gateway logs + tetragon. Fix gaps BEFORE scaling. **STOP, notify the author.**
6. Full run overnight with GPU granted. The GPU host-GPU crons paused for the window (morning-briefing, research-synthesis, muse-spark, weekly-data-horoscope, taste-reflection); distill-nightly stays paused; The second node-backed crons unaffected. First request per arm pays ~50s load; gateway 120s timeouts surface as 502s — watchdog handles. Defer cleanup (approval-wall lesson).
7. Analysis per playbook: runs-index rebuild (prefer makeup rows, filter dry-run era); mechanical parse; ClickHouse span bucketing via `attributes_string['model']` / `attributes_number[...]` (NOT JSONExtract on resources); 6 analysts, one per task — minimax ≤2 concurrent, spread with glm52/kimi-k3; 10-min no-output = kill + relaunch; collect verifies marker AND section completeness; orchestrator independently reads outliers + 1 rep/arm/task and spot-verifies every load-bearing claim; synthesis.
8. Close the loop — last run's gap: pre-reg and results never landed in the vault (blog only). This time both live here.
9. Publish: blog via publish-blog-post (PII-scrub companions); Reddit follow-up is self-contained with named tooling up front, r/LocalLLaMA tells applied, AI assist disclosed; reply to u/Easy-Ride3366.

## Kill criteria

- Any arm fails >40% of dry-run or first full-run reps → stop, diagnose, don't scale.
- Grader defect discovered mid-run → stop, fix, rerun affected cells only.
- Telemetry gap (missing spans/logs for any run) → those runs are void, rerun.
- GPU window pressure → drop to 3 arms before cutting reps or tasks.

## Timeline (estimate, dry run sets truth)

- Day 0: this pre-reg → the author's go on arms (3 vs 4), battery, window.
- Day 1: acquisition, lanes, routing, telemetry probe, dry run → STOP → go.
- Night 1: full run.
- Day 2-3: analysis fan-out + verification + synthesis.
- Day 3-4: vault results note, blog, Reddit follow-up.

## Open decisions — RESOLVED 2026-07-26

1. Arms: **4** (stock36, stock35, ornith, kat) — approved.
2. Battery: unchanged — approved.
3. Window: **today, all day, starting immediately** — approved.
4. MTP: enabled on all arms (drove the APEX/unsloth quant split, confound 1).

Remaining stop: dry-run telemetry gate, then the author's final go for the full 120.

## Run log

- 2026-07-26 12:21 — downloads via systemd-run (4 GGUFs, ~97 GB, all OK by ~12:55).
- 12:30 — 4 lanes added to llama-swap config (backup .bak-20260726-35bakeoff), restart, all 4 smoke green (content + reasoning_content verified).
- 12:33 — shim 0.2.2 built+pushed, gateway CM + deployment updated (k8s-lab@64b45f7), ArgoCD synced, pods rolled, gateway config.load verified.
- VRAM management: reranker lane is ttl:0 (5.3 GiB resident) and blocked the 26 GB APEX arms → reranker child killed for the window (llama-swap restarts it on next query post-window). Paused crons: research-synthesis, weekly-data-horoscope, vault-search-index. distill-nightly already paused.
- 12:40 — watchdog + dry run launched (yoke-bugfix × r1 × 4 arms).
- 12:44 — dry-run attempt 1 failed at workspace create: bakeoff template's `arm` coder_parameter option allowlist only had the 3 old arms ("Value must be a valid option"). Added the 4 new options to templates/bakeoff/main.tf, `coder templates push bakeoff -d ...` (the bare `-y` push from the dir failed with "No configuration files" — explicit -d required), committed k8s-lab@d316d7c.
- 13:05 — dry-run attempt 2: 4/4 pass, score 1.0. Gate: transcripts ✓, grades ✓, shim spans ✓ (all 4 arms, tokens+thinking_chars), gateway inference logs ✓ (55 evt:inference, sub+model+status). **Tetragon gap confirmed**: zero events captured from the coder-workspaces namespace (pods provably ran there; other namespaces appear fine). Pre-existing platform issue, likely also true 2026-07-25 — attribution carried by the other four layers. Fix later, not in the window.
- Per-run timing: agent wall 86-106s on yoke-bugfix (MoE speed), ~4 min full workspace cycle → 120 runs ≈ 6-10 h.
- 13:30 — the author approved. Full run launched: 4 arms × 6 tasks × 5 reps = 120, sequential arm-blocked, watchdog active.
- 23:35 — full run complete: 110/120 passed. Pass per arm: kat 29/30, stock35 29/30, stock36 27/30, ornith 25/30. Cost per arm (2,203 spans, 100% assigned): kat 383 calls / 12.6M in / 138K think_chars — cheapest by far; stock36 631 calls / 25.1M in / 802K think — the hog. Aggregates: /data/bakeoff/analysis-35b/report.md.
- 23:50 — window restored (crons resumed, watchdog stopped; reranker self-heals). Analyst fan-out launched: 6 task analysts (2× minimax-m3, 2× glm52, 2× kimi-k3), quote-or-no-claim contract, brief + per-task files in analysis-35b/.
- Failure modes so far (mechanical): ornith yoke-feature cluster (3/5, hidden acceptance test), web-research 60-iteration thrash ×4 (stock36×2, ornith×2), artifact-missing early stops ×3 (stock36 log-parse, stock36 web-research, kat log-parse), stock35 yoke-bugfix r4 fix not landed.
- 2026-07-27 00:45 — analysis complete: all 6 analysts returned in-contract findings; orchestrator spot-verified headline claims (fabrication self-flag, deleted-tests report, baseline presence, compactions, invented symbols — all confirmed). Verdict: H1 confirmed (kat), H2 partial, H3 split. Results note: results.
- Post-window restoration: unpause the 3 crons; reranker self-heals on next query; FileFlows off-hours resumes whenever its schedule fires.
