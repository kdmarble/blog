# log-parse — Behavioral Analysis

Task: investigate a ~6h multi-service log file (`app.log`, services auth/billing/search/dashboard) with CLI tools only, identify exactly three anomalies (an error-rate window, a latency regression with warning signature, a silent retry loop), and write `findings.md` with precise start times, signatures, retry-loop counts, and a paging-priority paragraph. Grader is a regex checklist (auth named, 02:13–02:17 window, 503/timeout, billing named, 04:00–04:01 start, `pool`, worker-7, job 88471, 23 attempts, ≥200 chars summary). Pass at 8/10 checks.

Note on grading artifacts below: the grader regexes `pool`, `88471`, `\b23\b`, `worker-7` are lenient and will match even in malformed findings; the `02:1[3-7]` and `04:0[0-1]` time-window checks are the discriminating ones. So "pass" does not mean the run identified the right window — it means the right tokens appear somewhere in the doc.

---

## Failure-mode reconstructions (the two non-passing runs)

**stock36 r2** (46s wall, findings.md missing, grade 0.0): a generation pathology. After a normal start (head, grep -c ERROR), the reasoning trace degenerates into one-token-per-line "sputtering" and then emits four raw `<tool_call>...<function=mcp__terminal>...` XML blobs *inside the reasoning box* rather than as actual tool invocations — the harness never executes them. The transcript's last assistant message is literally `(empty)`, and the session closes at 37s with 17 messages. Quote (stock36-r2, reasoning box): `gawk not mawk. Let me use simpler awk or grep+sed.` rendered as single tokens per line, followed by `<tool_call><function=mcp__terminal><parameter=command># Use python for proper parsing...`. The model never reached the write_file step. This is the stock36 sputtering pathology (see arm profile) at its most damaging — here it consumed the whole budget.

**kat r5** (101s wall, findings.md missing, grade 0.0): a context-window overflow, not a reasoning failure. kat ran four broad greps that each dumped large slices of app.log into context (kat-r5 transcript: `grep 'ERROR' /home/coder/work/app.log`, `grep 'WARN' ...`, `grep 'worker-7' ...`, `grep 'pool' ...` — all unfiltered, returning full line contents). It then re-grepped `latency_ms` and `$NF` columns raw. The accumulated grep output blew past the provider's 131072-token hard cap. Transcript tail (kat-r5): `request (146479 tokens) exceeds the available context size (131072 tokens)` after three compaction attempts, then `❌ Context length exceeded and cannot compress further.` Critically, kat-r5 had *already correctly identified all three anomalies in reasoning* before the overflow killed it (quote, kat-r5 reasoning): `1. **Elevated error-rate window**: auth service... from 2026-07-24T02:13:40 to around 2026-07-24T02:16:54`. It simply never reached `write_file`. The other four kat reps avoided this by piping greps through `awk '{print $1}'` or `grep -oP` to shrink output — this is the one rep where kat's otherwise-disciplined command style slipped, and the harness's compaction ceiling (131072) was lower than kat's working-set need.

## Per-arm behavioral profiles

### kat (KAT-Coder-V2.5-Dev) — the disciplined minimalist

kat's approach shape is the most consistent across reps: head the file, run `grep -c` for ERROR/WARN counts, and batch the first reconnaissance into a single parallel block of 3–4 commands. Every rep opens nearly identically, with a terse one-sentence plan (kat-r1): `Let me start by examining the log file to understand its structure, then search for the three anomalies.` There is no upfront enumeration of hypotheses, no restating of the task — it goes straight to the terminal.

The thinking-trace character is laconic and operational. kat reasons about *what to extract and how*, not about the domain. Its self-corrections are brief and technical. When a gawk-only `match(...,arr)` construct fails, kat-r1 immediately diagnoses it: `The awk match with three arguments is gawk-specific. Let me use simpler patterns or just grep/awk without the third-argument match form.` When the sed extraction misbehaves, kat-r1 pivots to `grep -oP` capture groups. These corrections cost one turn, not five.

Tool decisions are kat's standout strength. It consistently shrinks command output to avoid context bloat: `grep -oP '^\d{4}-\d{2}-\d{2}T\d{2}:\d{2}'` to get minute-buckets (kat-r1), `grep -oP '(?<=[ ,])auth|billing|search|dashboard(?=[ ,:])'` to enumerate services without dumping lines, and `awk '{print $1}'` after greps to reduce to timestamps only. kat uses zero `execute_code` calls across all five reps (verified: 0 🐍 exec in every transcript). It never reads the whole file — not once in five reps. This is the behavior the brief flags as the efficiency question, and on this task it is unambiguously **discipline, not corner-cutting**: kat extracts exactly the columns it needs, verifies baseline-vs-regression latency with before/after awk windows, and writes a findings.md that names the correct services, times, signatures, and retry counts.

Verification behavior: kat re-runs targeted checks. After writing findings, kat-r1 runs `grep "WARN" ... | grep -oP '(worker-7|billing)'` to confirm the WARN breakdown; kat-r2 re-checks whether the first billing pool WARN is exactly at 04:01:50. Waste modes are minimal — the only rep that failed (r5) did so from context overflow after running four unfiltered greps that returned full line contents, not from any reasoning defect.

### stock36 (Qwen3.6-35B-A3B) — the sputtering thinker

stock36 shares kat's jump-in shape (head, then batched grep) but is defined by a generation pathology: the reasoning trace frequently degenerates into one-token-per-line "sputtering," where normal prose is emitted as discrete tokens separated by newlines. I measured this quantitatively — fraction of reasoning-box lines that are very short single tokens. stock36 scores 0.12–0.35 across all five reps (r1 0.19, r2 0.33, r3 0.32, r4 0.12, r5 0.35), versus kat's 0.00–0.02. In its worst rep, even a mundane sentence renders as (stock36-r5): `The  billing  pool  warnings  seem  to  be  anomaly  # 2  or  could  it  be  a  latency  regression`.

A second pathology: stock36 leaks raw `<tool_call>...<function=mcp__terminal>...` XML *inside the reasoning box* instead of emitting a real tool call. This appears in all five reps (2, 4, 1, 5, 3 leaks respectively). Sometimes the harness still picks up the intended call; sometimes (r2) it does not, and the run dies with an empty final message.

When the sputtering doesn't kill the run, stock36's actual analysis is competent. It correctly identifies all three anomalies, computes the ~43ms→821ms billing latency jump via hourly awk buckets (stock36-r1 reasoning): `The latency regression is clear: billing jumps from ~43ms to ~821-852ms starting in hour 04`. Its findings docs are among the most detailed (stock36-r1 reports 1,096 high-latency requests, per-minute error breakdowns). But this competence is wrapped in enormous thinking-token waste: stock36-r5 burned 33,908 thinking chars and 19,707 output tokens — roughly 3× kat's budget — for the same answer.

Verification: stock36 re-reads its own findings file after writing (stock36-r1: `Let me verify the file looks correct by reading it back`). Waste modes are dominated by the sputtering itself and by redundant latency re-bucketing (stock36-r3 re-runs the hourly awk six times with different field-index guesses after miscounting columns).

### stock35 (Qwen3.5-35B-A3B) — the thrashing explorer

stock35's defining behavior is a refusal to trust the CLI-only instruction and a habit of escalating to Python when awk gets hard. Three of five reps (r1, r2, r5) open by reading the entire app.log via `read_file` — a direct violation of `do not try to read the whole file`. Quote (stock35-r1, first tool action): `📖 read app.log`. The other two reps (r3, r4) at least start with `head`.

The thinking trace is verbose and often confused about the data format. stock35-r2 spends ~15 turns convinced the log is pipe-delimited (`awk -F'|' '{print NF}'` returns 1, then `The field separator isn't working as expected because the log format doesn't have | separators`), re-deriving the space-delimited structure that other arms parse on sight. It also leaks `<tool_call>` XML (7 leaks in r3, 11 in r4) and `` tokens (3 in r2).

Tool decisions show a strong pull toward `execute_code`. stock35 used Python 8, 6, 4, 0, and 11 times across reps — the only arm to lean on it at all on this task. When awk field indexing fails, stock35-r1 abandons CLI and writes inline Python via `from hermes_tools import terminal`. This works but costs ~50–100s per call (stock35-r1: two execute_code calls at 50.3s and 105.8s). The net cost is the highest wall-times in the battery (stock35-r1: 297s, r4: 110s) despite a simple task.

Despite the thrash, stock35's final findings are correct and complete in all five reps. Its analysis quality when it lands is good — stock35-r2 even notes `active=32 idle=0` means "all 32 connections busy, zero available," a sharper causal read than some arms give. The waste is in the journey, not the destination.

### ornith (Ornith-1.0-35B) — the uneven middle

ornith is the most variable arm. It oscillates between kat-like clean runs (r2: 60s, 18 tools, no sputtering) and stock36-like sputtering (r1: 0.35 short-token fraction; r5: 0.39). Two reps (r1, r5) show the same token-sputtering pathology as stock36, and ornith also leaks `<tool_call>` XML (2, 0, 0, 2, 4 across reps). One rep (r4) reads the whole app.log, like stock35.

ornith's first move is inconsistent: r1/r2 use `head` or `wc -l` (disciplined), but r4 leads with `search_files` then `read_file app.log` (whole file). When ornith stays in the terminal, its commands are reasonable — r2 pipes WARN through `awk '{print substr($1,1,16)}'` to bucket by minute, and r3 uses `grep -v 'delivery retry' | grep -v 'pool checkout wait exceeded'` to hunt for a *fourth* anomaly (there isn't one, but the verification instinct is sound).

ornith's standout trait is the most thoughtful paging-priority reasoning in the battery. While most arms reflexively page on the auth 503s (loud, user-facing), ornith-r4 reasons through the trade-off and argues for the *silent retry loop* first (ornith-r4 reasoning): `I'd page on the retry loop first because: - It's invisible to standard error monitoring - Worker-7 job 88471 never succeeds, meaning some user-facing operation is permanently broken`. That is a genuine operational insight — the silent failure is the one monitoring won't catch. kat-r1 makes the opposite (also-defensible) call, paging on the billing regression because it's ongoing and revenue-adjacent.

Waste modes: ornith-r3 reruns the same latency-extraction awk with minor variations ~6 times chasing a field-index bug; ornith-r4 runs `grep -c ERROR app.log` *unquoted* so the shell treats `ERROR` as a filename and returns a wrong count (4 instead of 28), which it initially trusts.

## Head-to-head differences

1. **Whole-file reading.** stock35 reads app.log in full in 3/5 reps (r1, r2, r5: `📖 read app.log`); ornith does so in 1/5 (r4). kat and stock36 never do (0/5 each). This is a direct instruction violation that inflates context. Quote (stock35-r1): the very first tool call is `📖 read app.log`.

2. **Token sputtering.** stock36 sputters in all 5 reps (0.12–0.35 short-token fraction); ornith in 2/5 (r1, r5 at 0.35/0.39); kat essentially never (0.00–0.02); stock35 rarely (0.01–0.10, spiking to 0.10 in r4). Quote (stock36-r5): `The  billing  pool  warnings  seem  to  be  anomaly  # 2` rendered one-token-per-line.

3. **execute_code reliance.** stock35 is the only arm that escalates to Python — 8/6/4/0/11 calls across reps. kat: 0 in all reps. stock36: 0. ornith: 2 (only r4). When awk gets hard, stock35 reaches for `from hermes_tools import terminal`; kat switches to `grep -oP`.

4. **Command-output discipline.** kat consistently shrinks output (`grep -oP '^\d{4}-\d{2}-\d{2}T\d{2}:\d{2}'`, `awk '{print $1}'`). stock35 and stock36 frequently run bare `grep "ERROR" app.log` or `grep "WARN" app.log`, dumping full matched lines into context. Quote (kat-r1): `grep -oP '(?<=[ ,])auth|billing|search|dashboard(?=[ ,:])'` — extracts just service tokens.

5. **Raw XML leak into reasoning.** stock36 leaks `<tool_call>` in all 5 reps; stock35 in all 5 (1–11 leaks); ornith in 3/5; kat in 0/5. This is a generation-format failure where the model emits tool-call syntax inside the thinking box instead of as a structured call.

6. **Paging-priority reasoning depth.** Most arms reflexively page on auth (the loud 503s). ornith-r4 uniquely argues for the silent retry loop (quote above). kat-r1 argues for the billing regression (`Page on Anomaly 2 first... represents a multi-hour degradation... still active`). stock36 and stock35 default to auth with thinner justification.

## Notable runs

- **Best run: kat-r1** (82s, 27 tools, 10 calls). Clean reconnaissance, disciplined `grep -oP` extraction, correct self-diagnosis of the gawk-only `match()` bug in one turn, complete and precise findings. The reference execution for this task.
- **Ugliest passing run: stock35-r2** (167s, 35 tools). Reads whole file, spends ~15 turns confused about pipe-vs-space delimiters, leaks `<tool_call>` and ``, escalates to execute_code 6 times, yet still produces a correct findings.md. Maximum waste for a correct answer.
- **Ugliest run overall: stock36-r2** (37s, failure). The sputtering pathology at its most destructive: four `<tool_call>` XML blobs emitted inside the reasoning box are never executed, the final message is `(empty)`, findings.md is never written.
- **Most thoughtful: ornith-r4** (65s). Despite reading the whole file and miscounting errors via an unquoted grep, ornith-r4 delivers the battery's best paging rationale — recognizing the silent retry loop as the operationally dangerous anomaly because it's invisible to ERROR-based alerting.
- **Highest-cost correct run: stock36-r5** (194s, 19,707 output tokens, 33,908 thinking chars). ~3× kat's budget for identical findings, almost entirely sputtering overhead.

## Verdict (manner, not outcomes)

On this task shape — a bounded log-investigation with a known answer set and a regex grader — **kat's behavior is the best fit**, and its low-token style is **discipline, not corner-cutting**, with direct evidence. kat: never reads the whole file (0/5), never uses execute_code (0/5), never leaks tool-call XML (0/5), consistently shrinks command output with `grep -oP` and awk column projection, self-corrects awk portability bugs in a single turn, and verifies its findings before declaring done. Its one failure (r5) was a context-window overflow from a single slip into unfiltered greps — a tool-hygiene lapse, not a reasoning defect, and it had already solved the task before dying.

stock36 has comparable raw analytical competence when it survives, but the sputtering pathology (present in all 5 reps) and the XML-leak tendency (all 5 reps) make it 2–3× more expensive and, in r2, entirely non-functional. stock35 gets correct answers but through the most wasteful path — whole-file reads, format confusion, and Python escalation that other arms avoid. ornith is too inconsistent to recommend: its best runs rival kat, but its sputtering reps (r1, r5) and whole-file read (r4) show the instability that its 25/30 aggregate pass rate (lowest of the four) reflects.

The differences that matter here: output-shrinking command discipline (kat >> others), instruction adherence on "don't read the whole file" (kat/stock36 > ornith > stock35), and generation stability / absence of sputtering and XML leaks (kat >> stock35 > ornith > stock36). The difference that does not matter: all four arms correctly identified the same three anomalies with the same start times and signatures whenever they reached the write step — the task's ceiling is low enough that manner, not outcome, is the only discriminator.

ANALYSIS COMPLETE
