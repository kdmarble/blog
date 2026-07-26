
# 27B Post-Train Bakeoff — Behavioral Analysis

Qualitative layer over [the results note](results.md) (contract: [the pre-registration](preregistration.md)).
90/90 runs passed, so outcomes don't discriminate — this pass answers HOW each arm worked:
thinking traces and tool decisions, not scores.

Method: 6 analyst subagents (MiniMax-M3, GLM-5.2, Kimi K3) each read all 15 transcripts
for one task under a quote-or-no-claim evidence contract; orchestrator independently
read ~14 transcripts across 4 tasks and spot-verified every load-bearing claim (incl.
the fable confabulation and the 94-tool-call outlier). Quant skeleton: 1,202 anthropic-shim
spans bucketed per run (SigNoz), full per-run tool/reasoning parses.

Raw: /data/bakeoff/results/ (transcripts), /data/bakeoff/analysis/
(manifest-parsed.json, run-spans.json, brief.md, task-*.md, findings-*.md per task —
each 16-30KB, fully cited).

## Per-model behavioral profiles

### stock (Qwen3.6-27B) — the methodical checklister

- **Upfront planner, broad explorer.** Numbered plans in the first reasoning block;
  reads 6 files before editing on feature work; parallel first-turn batching
  (pytest + find; 3-5 greps at once on log tasks).
- **Deepest deliberation, sometimes decoupled from information.** Highest think-chars
  on 4/6 tasks. On web-research it re-derived the "one-commit release vs two notable
  changes" dilemma 4-5 times without new data ("Actually, I need to focus." —
  stock-r2; "You know what, I'll just go with what I have." — stock-r4). The
  deliberation never changed the final answer.
- **Most uniform.** Same opener every rep; identical patch shape on 4/5 bugfix reps;
  the only arm that explicitly reasoned about consecutive-reserved-port edge cases
  (stock-r4: "using `while` instead of `if` to handle consecutive reserved ports").
- **Strongest artifact discipline.** 4/5 read-backs after writing (web-research);
  3/5 documented dual remediation paths (homelab-debug, Option A/B); most tests
  added on the feature task (~29 vs ~14/13 for the others); cleanest error
  recovery (write_file failure → reasoned diagnosis → heredoc fallback → verify,
  stock-r5). Zero confabulated facts in 5 research artifacts.
- **Verification rigor.** The only arm that swept ALL services to rule out decoy
  latency regressions on log-parse (stock-r1) and debugged its own grep substring
  false-positive before trusting it.
- **Waste modes.** Refetch-instead-of-reproject (5 identical Newegg curls, stock-r2);
  over-engineering when over-reasoning (pydantic Config subclass workaround);
  the most painful r1 outliers (feature r1: 1000s, 58 tool calls, orphan-docstring
  cycle + `model-ls` typo cycle + .pyc rabbit hole).
- **Harness-nag response: argues, then complies anyway** ("The system note seems to
  be a generic reminder... No further action needed" — then re-runs pytest).

### thinkingcap (ThinkingCap) — the terse operator, genuinely bimodal

- **H3 holds behaviorally.** Lowest think-chars on 5/6 tasks (log-parse mean 4.2K
  vs stock 8.6K); the advertised token efficiency is real prose-density compression.
- **BUT efficiency is prose, not action.** Tool-call counts match the other arms
  (homelab-debug: same 7-8 calls, 40% fewer chars). It compresses narration, not steps.
- **Bimodal tail — the headline nuance.** Its best runs are the cheapest in the
  battery (mixed-pipeline r5: 38s, first-try, only rep to double-validate with
  `python -m json.tool`). Its worst runs are the MOST expensive in the battery:
  mixed-pipeline r1 spiraled 9.3K chars / 16 calls finding a split-join delimiter
  bug; web-research r2 burned ~10 calls / 18.8K think-chars on the phantom b10121
  release tag that stock-r4 dispatched with ONE API call.
- **Distinctive fingerprints.** End-to-end smoke testing no other arm does
  (`python -m yoke model ls --json | python3 -c json.loads`, r1-r4 feature);
  pipeline-stage reordering as a fix strategy (mixed-pipeline r4 — converting
  quotes before stripping comments, eliminating the hazard class; the cleanest
  recovery in the battery); gawk/mawk-proof awk idioms (`-F'latency_ms='` split).
- **Spec-literalism.** Self-corrected rich-table → one-line-per-model against the
  spec text (feature r1); others mostly corrected via test failures.
- **Weaknesses.** Patch-shape inconsistency (2/5 bugfix reps emit an awkward
  double-mutation loop); occasional under-verification/under-documentation
  (feature r5: zero new tests); search-tool fumbles (bugfix r2: two no-op `find`
  searches before discovering grep); the homelab-debug r1 redundant re-read —
  the single clear waste event on that task, from the arm marketed as efficient.
- **Harness-nag response: complies quietly.**

### fable (Fable-Fusion-711) — the wide-roaming investigator with a confabulation tail

- **The "sloppy worker" aggregate framing is half the story.** Its +24% calls buy
  BOTH loops AND genuinely superior investigation, depending on the task.
- **Best investigator in the battery.** Only arm that verified the diagnosed config
  was actually LIVE (sites-enabled symlink check, 4/5 homelab-debug reps — the
  others assumed the file on disk was active; on a variant where the symlink was
  the real bug they'd be wrong while passing). Only run that caught its own
  incomplete evidence gathering and recovered (homelab-debug r1). Widest source
  coverage on research (Reddit/B&H/Amazon/atxcases); widest changelog window
  (44 commits vs ~8); richest price extraction (Newegg `window.__initialState__`
  via execute_code — OriginalUnitPrice/FinalPrice/LowestPrice30Days, the most
  structured data any run produced). Deepest single-source verification:
  identified mostlygeek = Benson Wong via the GitHub users API (r1).
- **Best artifact self-proofreading instinct, unevenly applied.** Caught its own
  mangled FIX.md diff and rewrote it (bugfix r2 — the only artifact-verification
  act on that task); talked itself out of the Klarna $252.99 installment-price
  trap by pure reasoning (research r5).
- **THE critical finding: confabulation.** The battery's only factual error:
  fable-r4's research.md ships "Maintained: `mostlygeek` — Matthew Garrett,
  former Red Hat engineer" (invented; mostlygeek is Benson Wong — which fable-r1
  itself sourced correctly). The keyword grader passed it (`llamaswap_maintainer:
  true` = it contains "mostlygeek"). Fable-r4 also skipped the read-back that
  would have given it a chance to catch the error (only 2/5 fable reps verify
  artifacts vs 4/5 stock). Creative-merge character: richer when right, wrong
  when not. Also: guessed-URL fetch (r4), stale-snippet trust ("latest is b8554",
  r5), literal `</parameter>` training-artifact leaks in thinking (r5).
- **Loops on resistant facts.** 94-tool-call price hunt (research r1); degenerate
  SearXNG queries (`"Enthoo+Pro+2+Server"+$189+$199`); six near-identical empty
  queries (r5); repeated identical Newegg fetches; 13.7K-char re-extraction loop
  (log-parse r3 — most verbose reasoning in the battery).
- **Variance king.** 124-381s wall on research; clean r2s vs ugly r1s on coding
  (feature r1: orphan code + duplicate registration + irrelevant skill load).
  Unique latent-correctness dip: bugfix r4's `if/else` patch fails on consecutive
  reserved ports — passes the test, wrong in general.
- **Tool distrust vs workaround.** Hits one jq exit-5 and declares "jq seems
  broken for me somehow" (r5); stock works around the same class of failure
  without generalizing.
- **Harness-nag response: complies, sometimes over-complies** (4th pytest run).

## Task-shape map (manner, not outcomes)

| Task | Best-fit behavior | Why |
|---|---|---|
| yoke-bugfix (convergent single-cause) | **stock** | Uniform patch shape, only arm reasoning edge cases explicitly; tc close 2nd, fable weakest (3 patch shapes, latent `if` bug) |
| yoke-feature (small feature + hidden tests) | **thinkingcap** | Clean first patches, spec-literal, e2e smoke tests; stock most thorough (29 tests); fable uneven |
| log-parse (grep/awk forensics) | **stock** | Cross-service elimination, false-positive debugging, quantitative aggregation; tc best wall-clock |
| homelab-debug (static diagnosis) | **fable** (investigation) / **stock** (artifact) | Only fable verified liveness; only stock documented alternative fixes; academic on this easy variant |
| web-research (shallow fact-finding + citations) | **stock** | Zero confabulation, 4/5 verify; fable highest ceiling but shipped the only error; tc cheap but phantom-prone |
| mixed-pipeline (deterministic repair) | tie / **thinkingcap** recovery | All converge on same architecture, hit same 2 traps; tc's reorder is the cleanest recovery |

## Cross-cutting findings

1. **Pass rate hid a factual error.** 90/90 pass AND fable-r4 shipped an invented
   maintainer name. Keyword graders can't see confabulation; any future battery
   needs a fact-check rubric lane. This is the strongest argument for the
   transcript layer existing at all.
2. **ThinkingCap's efficiency has a fat tail.** Median case: cheapest and fastest
   (H3 confirmed again). Tail case: the two most expensive runs in the battery.
   "Token-efficient thinking" = same quality at lower AVERAGE cost, with worse
   worst-cases than stock.
3. **Fable's extra calls buy real investigation on diagnostic/research shapes and
   real waste on convergent shapes.** Task-dependent, not a uniform tax. The
   creative-merge signature is breadth + confabulation risk, not loopiness per se.
4. **The harness verification nudger burned 1-2 redundant pytest runs on nearly
   every coding rep of every arm** — models know it's redundant (all three say so
   in reasoning) and comply anyway. Harness artifact worth fixing; it inflates the
   "redundant verification" signal in every arm's profile.
5. **Wall-time outliers are mostly infrastructure, not behavior** (APIConnectionError
   retries, model cold-loads: stock-r1 656s homelab-debug, fable-r1 831s bugfix).
   Read aggregates accordingly.
6. **Shared blind spots across all arms**: the `'//' not in text` false-positive
   verification loop (6/15 mixed-pipeline reps); the "3-hour regression seems too
   broad" doubt (all arms, resolved at different speeds); mawk vs gawk `match()`
   3-arg collisions (stock and fable, never thinkingcap).

## Selection guidance (updated from results note)

- **Default driver: stock stays.** Most predictable, best artifact discipline, zero
  confabulation. Its verbosity is the known tax.
- **ThinkingCap for long/cost-sensitive loops** — with a watchdog. Median case is
  the cheapest; the fat tail means anomalous sessions cost MORE, not less. Pair
  with loop detection.
- **Fable for open-ended investigation where breadth matters** (research sweeps,
  diagnosis under uncertainty) — never for unsupervised fact-shipping. Its outputs
  need a verification pass stock/tc outputs don't. Confirms the creative-merge
  reputation: it is the most interesting arm and the least trustworthy one.

Caveats: one battery, one difficulty, n=5 per cell, ceiling effects everywhere.
The manner differences documented here are the leading indicators for a harder
battery (multi-file features, ambiguous incidents) where they'd become outcome
differences — fable's confabulation and tc's tail especially.

## The author's verdict (2026-07-26, verbatim)

"I would stick with the base qwen model. If anything, this experiment solidified
my thought that finetunes often aren't better than their base. I might consider
thinking cap as an actually useful finetune, but only in instances where latency
and speed need to be balanced. If your hardware can serve the base qwen model
fast enough for you, then I see little need for thinkingcap"
