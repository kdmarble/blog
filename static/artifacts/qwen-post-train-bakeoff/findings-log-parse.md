# log-parse — Behavioral Findings (fable / stock / thinkingcap)

Task: given a ~6h multi-service `app.log`, find three anomalies (auth error-rate
window, billing latency regression + warning signature, silent worker-7 retry
loop) using grep/awk/sort/uniq, and write `findings.md`. All 15 runs passed the
regex grader (≥0.8). This is a comparison of *how* each arm worked.

---

## 1. Per-arm behavioral profiles

### Fable (r1–r5)

**Approach shape — jump-in explorer.** Every fable rep opens with `head` (or
`head`+`wc -l`/`tail`) and immediately fires focused greps. There is no upfront
planning monologue; the first reasoning block is already describing the next
command. r1: "The user wants me to investigate a log file... Let me start by
understanding the structure of the log file." (fable-r1). r2 opens identically:
"Let me investigate the app.log file... I'll start by understanding the log
format" (fable-r2). The exploration is sequential and grep-first — it does not
batch many commands per turn the way thinkingcap does.

**Thinking-trace character — terse, action-coupled.** Fable's reasoning blocks
are short (2–6 lines) and almost always terminate in "Let me <command>".
Representative: "Good - there's a concentrated window of ERRORs in auth around
02:13-02:16. Let me see the full extent and count." (fable-r1). And: "Great, I
can see two distinct warning patterns: 1. worker-7 retry loop for job 88471...
2. billing pool checkout warnings... Let me see the rest of the WARN lines and
count them." (fable-r1). The trace reasons about *what the data shows* and
*what to run next*, with little meta-reflection. Self-correction appears when a
command misbehaves: "Bad extraction. Let me try again to pull the latency_ms
value properly." (fable-r1, after a failed awk). r3 is the verbose outlier
(13,732 reason chars — highest in the whole battery) and shows sustained
second-guessing: "Actually wait - I need to check if there are other warnings
that accompany this more specifically" and "Let me verify more carefully. Also
let me check if there's anything else going on" (fable-r3).

**Tool decisions — grep|awk|cut|sed pipelines, occasionally bitten by mawk.**
Fable builds multi-stage pipelines. r1 constructs a full per-minute billing
latency aggregator in one awk line: `awk '/INFO billing request completed/
{...; n[min]++; sum[min]+=lat+0; if(lat+0>max[min]) max[min]=lat+0 } END {...
printf "%s count=%d avg_lat=%.0f max_lat=%d\n",...}'` (fable-r1). r4 however
collides with the mawk/gawk gap — `match($0, /latency_ms=([0-9]+)/, a)` (3-arg
form) fails, and it spends several turns diagnosing: "mawk on this system
doesn't support the three-argument match() syntax. Let me simplify my
approach." then "mawk doesn't support named groups or third argument to match"
(fable-r4) before falling back to `sed`/`grep`. Latency onset is usually pinned
by scanning raw lines around the 03:59→04:00 boundary (e.g. fable-r4:
`grep 'billing' ~/work/app.log | grep 'T03:59:[0-9]'`).

**Verification behavior — variable.** r1 is disciplined: "I need to confirm:
does it never escalate to ERROR? Let me verify that job 88471 never appears in
ERROR lines." then `grep "job 88471" ... | grep -c "ERROR"` (fable-r1). r3
*over*-verifies — re-extracting billing latencies at >500, >1000, >300
thresholds repeatedly, re-reading the same column many ways. r2 has purely
redundant duplicate commands (e.g. `grep "WARN worker-7 job 88471"` issued
twice back-to-back at fable-r2, and `grep "ERROR\|status=503" | awk...` twice
identically).

**Waste modes.** (a) Duplicate/identical commands (r2). (b) Re-extraction
loops on the same latency data (r3). (c) awk-syntax dead ends from gawk-only
idioms (r4). (d) r1 exhibits a rendering artifact where its user-facing message
box emits ~1 character per line — a harness/display quirk, not a model defect,
but it inflates the transcript.

---

### Stock (r1–r5)

**Approach shape — upfront parallel planner.** Stock's signature is the
explicit "hunt for all three anomalies in parallel" framing, stated verbatim or
near-verbatim in every rep: "Let me hunt for all three anomalies in parallel"
(stock-r1, r2, r3, r4, r5). It opens with `wc -l` (often + `tail -5` or
`awk '{print $1,$2}'`), then in a single turn dispatches 3–5 independent greps
(`grep ERROR`, `grep WARN`, `grep -i retry`, plus a time-bucketing awk). This
batched-fan-out is consistent across all 5 reps.

**Thinking-trace character — verbose, planful, quantitatively narrative.**
Stock's reasoning blocks are the longest and most propositional. r1 produces
detailed quantitative summaries mid-run: "The billing service has a
dramatically higher average latency (414.3ms) compared to all other services
(~43ms). The max is 1900ms for billing. The high-latency requests only occur
from 04:00 onward (360+371+365 = 1096 high-latency requests in hours 04, 05,
06), while before 04:00 the average is 43.8ms." (stock-r1). It also reasons
about *why* a prior result might be wrong — debugging a false "20 errors at
03:" count: "the issue is that grep '03:' would also match '04:03' since '03:'
appears as a substring... Let me verify this is correct now" (stock-r1).
Planning prose recurs: "1. ERROR lines (elevated error-rate) 2. High latency +
WARN lines (latency regression) 3. Retry patterns that don't escalate"
(stock-r2).

**Tool decisions — aggregation-heavy awk with arrays, multi-stage pipelines.**
Stock is the most sophisticated quantitatively. r1 computes per-hour
high-latency percentages: `... if(v>500) hi[h]++; tot[h]++} END{for(h in tot)
printf "Hour %s: %d high-latency / %d total (%.1f%%)\n", h, hi[h], tot[h],
hi[h]*100/tot[h]}` (stock-r1). r5 buckets billing latency by hour with awk
arrays and computes per-hour averages. r4 uses a clean `sed -n
's/.*T\(HH:MM\).*latency_ms=\([0-9]*\).*/\1 \2/p'` + awk-aggregate pipeline to
get per-minute averages and spots the exact transition: "03:59 avg=43 count=3
(normal) / 04:00 avg=902 count=8 (massive jump!)" (stock-r4). Like fable, stock
also trips the gawk-only `match(...,a)` 3-arg form (r4, r5) but recovers by
switching to `split()`.

**Verification behavior — the most thorough of the three arms.** Stock
explicitly rules out alternative hypotheses. r1 checks every other service for
latency >200ms: "search, dashboard, and auth never have latencies above 200ms
in normal INFO requests" (stock-r1) — eliminating the possibility that the
latency regression lives elsewhere. It then positively confirms "zero billing
high-latency requests before 04:00. The regression starts at 04:00:06."
(stock-r1) after debugging the substring false-positive. Multiple reps confirm
the retry loop never errors via `grep ... | grep -c ERROR` returning exit 1
(stock-r3: `[exit 1]`; stock-r5: `[exit 1]`).

**Waste modes.** r1 is the longest run in the battery (37 tool calls) — partly
genuine thoroughness, partly over-verification (re-bucketing the same latency
data several ways). r2 invokes slow shell `while read` loops (one command times
at 22.5s, another at 26.7s) where an awk one-liner would suffice. r4/r5 lose
turns to the gawk/mawk match() syntax issue before recovering.

---

### Thinkingcap (r1–r5)

**Approach shape — batched parallel hunter, compressed reasoning.** Thinkingcap
also opens with `head`/`wc -l`/`awk '{print $1}'` and states the parallel-hunt
plan, but its distinguishing behavior is *firing more commands per turn*:
transcripts routinely show 4–5 consecutive `┊ ⚡ preparing mcp__terminal…` lines
followed by 4–5 `┊ 💻 $` commands in one assistant turn (e.g. thinkingcap-r2,
r3, r4, r5). This makes its wall-clock efficiency the best (lowest mean wall_s
at 93s vs stock 127s vs fable 109s) despite sometimes high tool-call counts.

**Thinking-trace character — the most token-efficient.** Thinkingcap has the
lowest reason_chars in the battery (mean ~4,242 vs fable ~7,249 vs stock
~8,638), and especially low *per tool call*. Its reasoning is compressed to
essentials with no filler. r1: "Good. Log runs from ~01:00 to ~07:00 UTC on
2026-07-24. Four services with roughly equal traffic (~2700 each). Now let me
hunt for the three anomalies systematically." (thinkingcap-r1). It still
self-corrects — r1 initially mis-attributes a latency signal to the retry-loop
window ("Notice elevated latencies around 03:31-03:42 but that overlaps with
the retry loop. Let me check if there's a separate latency regression tied to a
different warning signature." — thinkingcap-r1) and recovers within two turns.
r4 even reasons about case sensitivity: "No explicit 'WARNING' text - but there
might be warnings with different casing" (thinkingcap-r4) before settling on
`WARN`.

**Tool decisions — clever one-liners, aggressive awk field-splitting.**
Thinkingcap favors compact idioms. r3 computes minute-of-day integers for
bucketing: `awk -F'latency_ms=' '{split($2,a," "); lat=int(a[1])} {ts=$1;
gsub(/T/," ",ts); split(ts,t," "); split(t[2],h,":"); minute=h[1]*60+h[2];
print minute, lat}'` then derives "At minute 240 (04:00) it spikes to 725ms"
(thinkingcap-r3). r3 and r5 both use the `awk -F'latency_ms=' '{split($2,a,"
")}'` technique to peel the value without regex capture groups — sidestepping
the gawk/mawk trap that fable and stock fall into. r4 uses `substr($1,1,16)`
for clean minute bucketing.

**Verification behavior — efficient but present.** Thinkingcap confirms the
retry count and no-ERROR pairing compactly: "The retry loop has exactly 23
attempts and zero ERROR escalations - that confirms anomaly 3"
(thinkingcap-r5). It shares the cross-arm habit of doubting the 3-hour billing
window: r2 "The billing warnings span from 04:01 to 06:59 - that's almost 3
hours. That seems too broad for a single anomaly" (thinkingcap-r2); r5 "267
high-latency billing requests over ~3 hours seems too broad. Let me narrow"
(thinkingcap-r5). Unlike stock it usually resolves the doubt quickly (1–2
commands) rather than re-bucketing extensively.

**Waste modes.** Lowest overall. r3's 38 tool calls look high in the aggregate
table but each is a micro-step with tiny reasoning cost — the inefficiency is
cosmetic. The main genuine waste is occasional re-issuing of near-identical
ERROR greps within a single batch (r2).

---

## 2. Head-to-head

**(a) Reasoning density vs. tool-call density.** Stock writes the most
reasoning per turn (long planful blocks, quantitative narrations); thinkingcap
writes the least (compressed, essentials-only) yet issues the most commands per
turn via batching. Same job, opposite token-allocation strategies. Evidence:
stock-r1's mid-run summary "The billing service has a dramatically higher
average latency (414.3ms)... 1096 high-latency requests in hours 04, 05, 06"
versus thinkingcap-r1's "Good. Log runs from ~01:00 to ~07:00... Now let me
hunt" — stock narrates the computed result, thinkingcap narrates only the
intent.

**(b) Parallelism discipline.** Thinkingcap consistently batches 4–5 commands
per assistant turn (`⚡ preparing mcp__terminal…` x4–5 then `💻 $` x4–5, see
thinkingcap-r2/r3/r4/r5). Stock also hunts "in parallel" but more often as 3
greps per turn. Fable is the most sequential — typically one command per turn
with its result evaluated before the next (fable-r1, r2, r5).

**(c) Command sophistication.** Stock builds the most elaborate aggregation
awk (per-hour percentage tables in stock-r1, per-minute sum/count arrays in
stock-r4/r5). Thinkingcap reaches for the cleverest *one-liners*
(minute-of-day integer math in thinkingcap-r3, `-F'latency_ms='` field-split in
thinkingcap-r3/r5). Fable mixes grep|awk|cut|sed but is the most prone to the
gawk-only `match(s,re,arr)` trap (fable-r4: "mawk doesn't support named groups
or third argument to match").

**(d) Verification thoroughness.** Stock is the most rigorous — it explicitly
sweeps *all* services to rule out alternative locations for the latency
regression ("search, dashboard, and auth never have latencies above 200ms" —
stock-r1) and debugs a substring false-positive to confirm "zero billing
high-latency requests before 04:00" (stock-r1). Thinkingcap verifies
compactly ("exactly 23 attempts and zero ERROR escalations - that confirms
anomaly 3" — thinkingcap-r5). Fable is inconsistent: r1 verifies no-ERROR
properly, but r3 over-verifies the same latency data and r2 issues literal
duplicate commands.

**(e) Shared skepticism about the 3-hour window.** A cross-arm behavioral
fingerprint: reps in all three arms flag that a ~3-hour latency regression
"seems too broad" and re-investigate before accepting it — fable-r3 ("Wait,
there's also billing requests with latency around 500ms earlier in the log?
Let me check"), stock-r2 ("the billing warnings seem very spread out... That's
3 hours of warnings. Let me look more carefully"), thinkingcap-r2 ("That seems
too broad for a single anomaly"). This is the single point where all three
post-trains converge on the same doubt.

**(f) Paging judgment splits.** Fable unanimously pages on auth (5/5). Stock
splits 4 auth / 1 billing (stock-r2 pages billing: "The billing latency
regression is the highest-priority page... sustained and worsening"). 
Thinkingcap splits 3 auth / 2 billing (thinkingcap-r2 and r4 both page billing:
"a sustained resource leak affecting a revenue-critical service for 3 hours").
So the billing-first stance is a minority 3/15, held only by stock and
thinkingcap — fable never deviates from auth-first.

---

## 3. Notable runs

- **stock-r1 (best).** The most thorough, quantitative run in the battery: 37
  tool calls, explicit elimination of search/dashboard/auth as latency
  candidates, per-hour high-latency percentage tables, and a careful debug of a
  `grep "03:"` substring false-positive ("grep '03:' would also match
  '04:03'... Let me verify this is correct now"). Sets the gold standard for
  *not* being fooled by one's own command output.

- **fable-r3 (ugliest).** 27 tool calls but 13,732 reason chars — the most
  verbose reasoning in the entire battery — spent largely re-extracting the
  same billing-latency column at multiple thresholds (>1000, >500, >300, <500)
  and re-asking "Let me verify more carefully. Also let me check if there's
  anything else going on." Loops on confirmation without converging faster.

- **thinkingcap-r3 (most efficient reasoning).** 38 tool calls but only 3,381
  reason chars (lowest in the battery). Reaches the exact 04:00 transition via
  a minute-of-day integer bucketing awk and reads it straight off: "At minute
  240 (04:00) it spikes to 725ms average." Maximum information per thinking
  token.

---

## 4. Verdict

For a **log-parse** task — grep/awk exploration where the difficulty is
distinguishing the real signal (a 3-hour sustained regression) from lookalikes
(false-positive substring matches, overlapping windows) — **stock's behavior
fits best**. Its combination of batched parallel fan-out, aggregation-heavy awk,
and explicit cross-service elimination (ruling out search/dashboard/auth,
debugging its own false positives) is exactly the discipline this task rewards.
The cost is verbosity and occasional over-verification (r1's 37 calls).

**Thinkingcap is a strong efficiency runner-up**: it reaches the same
conclusions with the fewest thinking tokens and the best wall-clock time,
using cleverer one-liners, and it is the only arm where two reps argued for
billing-first paging (a defensible ops stance). Its thinner verification would
matter more on a task with subtler traps.

**Fable is capable but more variable**: clean and fast when its commands work
(r1), but it is the most exposed to the gawk/mawk syntax gap (r4) and the most
prone to either duplicate commands (r2) or reasoning loops (r3). On this
particular task the difference between the three is modest — all 15 runs
passed — but stock's thoroughness and thinkingcap's efficiency are the
behaviors that best match the task shape, with fable a close but noisier third.

ANALYSIS COMPLETE
