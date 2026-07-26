
# 27B Post-Train Bakeoff — Results

Full run completed 2026-07-26. Contract: [the pre-registration](preregistration.md). 90 valid runs (6 tasks x 5 reps x 3 arms) after a 21-run makeup pass for infra failures (gateway 120s TTFB aborts, llama-swap event-loop wedges, orchestrator name-gen bug — all fixed mid-run or rerun). Raw data: /data/bakeoff/results/ (report.md, runs-assembled.json, per-run transcripts + grades).

## Headline: 90/90 PASS

Every arm passed every rep of every task. At this battery's difficulty, success rate does not discriminate. H0 confirmed. The signal is entirely in efficiency.

## Median Hermes wall time (s)

| task | stock | thinkingcap | fable |
|---|---|---|---|
| yoke-bugfix | 97 | 90 | 95 |
| yoke-feature | 154 | 136 | 168 |
| log-parse | 120 | 91 | 101 |
| homelab-debug | 35 | 35 | 39 |
| web-research | 127 | 92 | 158 |
| mixed-pipeline | 60 | 51 | 64 |

## Totals across the 30-run battery (shim spans, SigNoz)

| metric | stock | thinkingcap | fable |
|---|---|---|---|
| model calls | 380 | 350 | 472 |
| input tokens | 10.87M | 8.74M | 13.36M |
| output tokens | 149.7K | 115.5K | 148.3K |
| thinking tokens (est) | 47.6K (31.8% of out) | 31.6K (27.4%) | 41.8K (28.2%) |
| tool calls | 550 | 497 | 574 |
| total gen time | 2289s | 3209s | 2708s |

## Hypothesis verdicts

- **H0 (no success-rate difference): CONFIRMED.** 5/5 everywhere.
- **H1 (thinkingcap > stock on tool-call consistency): NOT CONFIRMED.** Both were perfectly consistent; no derailments from either. Battery too easy to discriminate.
- **H2 (fable < stock for agentic coding): REJECTED on outcomes.** Fable went 5/5 on both yoke tasks. But it is the least efficient arm: +24% more model calls than stock, +23% more input tokens, most tool calls — same results, more work. The creative-merge signature shows up as chattiness/loopiness, not failure.
- **H3 (thinkingcap fewer thinking tokens): CONFIRMED.** 34% fewer thinking tokens than stock (31.6K vs 47.6K), 23% fewer output tokens, 20% fewer input tokens (fewer turns -> less context resend). Fastest median wall time on 5 of 6 tasks.

## Interpretation

ThinkingCap delivers stock-identical outcomes at ~20-25% lower token cost and the best wall times. Per-call generation is actually LONGER (avg 9.2s vs stock 6.0s) — it wins by needing fewer turns, not by faster turns. That is the efficient-thinking claim holding up in a realistic harness.

Fable Fusion matches outcomes but wastes ~25% more calls getting there. Fine for creative work (its actual target), no reason to prefer it for agentic loops.

Stock remains a perfectly good default. There is no correctness argument for switching; the argument for ThinkingCap is cost/latency efficiency on long agentic sessions.

Caveats: one battery at one difficulty, Q4_K_M-MTP quant, Hermes harness only, n=5 per cell. Tasks were solvable by all three, so ceiling effects hide any capability gap. A harder battery (multi-file features, longer-horizon debugging) might separate outcomes.

## Infra lessons folded back

- claude-gateway default upstream TTFB (120s) aborts long thinking turns -> timeouts.upstream_ttfb_ms: 600000 now in git.
- anthropic-shim defaults max_tokens=4096 for clients that omit it (truncates thinking models) -> passthrough in 0.2.1.
- llama-swap wedged its event loop 4x (swap transitions + one idle period): TCP accepts, handlers never read, recv-Q grows, zero journal activity. systemctl restart cures instantly. Root cause open — watchdog script at ~/homelab/qwen-bakeoff/llama_swap_watchdog.py.
- Shim per-call spans + harness run spans now land in SigNoz (anthropic-shim, bakeoff-harness services) — reusable trace layer for future experiments.
