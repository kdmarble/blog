# Findings: homelab-debug task

Analyst: subagent-glm52. Task: diagnose a 502 Bad Gateway from a static filesystem
capture (nginx reverse proxy → gunicorn Flask app) and write rootcause.md.
Ground truth: nginx `proxy_pass` points at port 5001 but gunicorn binds 5000 — a
one-digit port mismatch. All 15 runs passed the grader at score 1.0.

Run IDs below are shorthand: `{arm}-r{rep}` maps to the directory
`bakeoff-{arm}--homelab-debug--r{rep}--<ts>` in the task file's table (e.g.
`fable-r1` = `bakeoff-fable--homelab-debug--r1--1785060765`). Quotes are verbatim
from `transcript.txt` reason boxes unless noted.

This is an easy task — a single line, single file, port-mismatch diagnosis with
all evidence sitting in six small files. The behavioral differences between arms
are therefore narrow and largely stylistic; where a difference does not matter
for the outcome, I say so plainly rather than inflate it.

---

## 1. Per-arm behavioral profiles

### stock (Qwen3.6-27B base)

**Approach shape.** Stock is the most templated and uniform arm. Every one of its
five reps follows the identical three-beat shape: (1) one reason block announcing
"read all the relevant files", (2) a single parallel batch of six `read_file`
calls covering every file the prompt named, (3) one reason block tabulating the
evidence, then `write_file`. It never lists the directory first and never re-reads.
It trusts the prompt's file inventory completely and reads the six named files
verbatim. Reps r2-r5 are nearly byte-for-byte identical in structure.

**Thinking-trace character.** Stock's traces read like an evidence checklist — it
enumerates each file and tags it. From `stock-r1`:

> "1. **nginx.conf** - standard, includes sites-enabled/*  2. **sites-available/dashboard** - proxy_pass to `http://127.0.0.1:5001` (port 5001)  3. **dashboard-error.log** - Connection refused on upstream ... 4. **dashboard.py** - simple Flask app, nothing wrong  5. **dashboard.service** - gunicorn binds to `127.0.0.1:5000`"

There is no self-doubt, no dead reckoning, no "what if I'm wrong" branch. The
trace is purely descriptive: list files, name the mismatch, conclude. The one
flicker of verification intent is in `stock-r4`, which says "Let me verify it
looks good by reading it back" — but it never actually issues the read; it goes
straight to the user summary. So the verification is stated, not performed.

**Tool decisions.** Minimal and uniform: six parallel `read_file` calls, one
`write_file`. The only deviation is `stock-r5`, which appends a lone `find .*`
(`🔎 find .* 0.3s`) *after* it has already read and analyzed everything — a
stray, purposeless exploration that changes nothing. No grep/awk/sed, no symlink
check. No arm used sophisticated pipelines here, but stock is the most rigidly
read-only arm.

**Fix documentation depth (a real differentiator).** Stock is the only arm that
routinely documents *alternative* fixes. `stock-r2` reasons:

> "I could either revert nginx back to port 5000 or update gunicorn to bind on 5001 instead — reverting nginx seems like the safer bet since the systemd unit already had the correct port."

Its rootcause.md ships both Option A (revert nginx) and Option B (move gunicorn),
with a recommendation. `stock-r5` does the same Option A/B split. `stock-r3`
gives the primary fix plus a `sed -i 's/127.0.0.1:5000/127.0.0.1:5001/'` one-liner
alternative. So 3 of 5 stock reps offered an alternative remediation path; fable
and thinkingcap never did (all 10 of their reps give a single fix).

**Verification behavior.** None. Stock never re-ran a check, never confirmed the
sites-enabled symlink, never re-read a file. It treats its first-pass read as
ground truth. Given the task, that trust was warranted — but it means stock
exhibits zero independent verification.

**Waste modes.** Negligible. The stray `find` in `stock-r5` is the only chaff.
The 656s wall-clock on `stock-r1` (vs ~30s for r2-r5) is **not** agent behavior:
its transcript opens with a harness line "🔁 Transient APIConnectionError on
anthropic — rebuilt client, waiting 6s" plus a 5.25s model cold-load, and timing.json
shows `duration_s: 656`. The agent's own reasoning/tool cadence in r1 is
indistinguishable from r2-r5 (3 reason blocks, 6 reads, 1 write). The wall time is
infrastructure overhead, not the model thinking longer. Flagged so the aggregate
isn't misread.

**Narration habit.** `stock-r2` through `stock-r5` each emit a user-facing Hermes
message ("Let me read all the relevant files...") *between* the first reason block
and the read batch — spending a user turn to announce intent before acting. `stock-r1`
does not. (thinkingcap shares this habit in r2-r5; fable never does.)

### fable (Fable-Fusion-711 merge)

**Approach shape.** Fable is the only arm that behaves like an *investigator*
rather than a pattern-matcher, and it shows most clearly in `fable-r1`. r1 does
not trust the prompt's file list blindly: it opens with a `find .*` to survey the
tree, reads four files, then — critically — notices the sites-available config is
missing from what it read and goes looking for it:

> "But my search found only 5 files and none of them are under sites-available or sites-enabled. Let me look more carefully at what exists." … "Actually, wait - let me re-read the prompt."

It then runs a terminal `find ... -type f -o -type l`, discovers the file and the
symlink, reads `dashboard` + `dashboard.service`, and only then writes. This is the
single instance across all 15 runs of an agent catching its own incomplete evidence
gathering and recovering. r1 carries 7 reason blocks / 3868 reason chars — by far
the most reasoning of any run. r2-r5 collapse to the common streamlined pattern
(batch read 6, analyze, write), though r2/r4/r5 still run an `ls -la sites-enabled`
to confirm the symlink.

**Thinking-trace character.** Fable's traces are conversational and self-aware in
a way the other two arms' are not. Where stock enumerates and thinkingcap
compresses, fable *narrates*:

> "Good, the layout is as described. Let me read all relevant files to diagnose the issue." (`fable-r1`)
> "Now I have a clear picture. Let me read the nginx site config and systemd service to confirm, then write the report." (`fable-r1`)

It also voices the alternative hypothesis and dismisses it explicitly:

> "The fix: either change nginx proxy_pass to 5000 (if the app port is correct), or change gunicorn to listen on 5001 (if the new nginx port was intentional). Given the app log shows startup at 13:58 and errors start at 14:03 ... most likely the 'small config change' was to nginx's proxy_pass line." (`fable-r2`)

So fable does reason about alternatives — but, unlike stock, it resolves them in
its head and ships a single recommended fix rather than documenting both.

**Tool decisions.** Fable is the only arm that uses `search_files`/`terminal find`
for genuine discovery (`fable-r1`), and the only arm that verifies the
sites-enabled symlink is actually wired: `ls -la /home/coder/work/scenario/etc/nginx/sites-enabled/`
appears in `fable-r1`, `fable-r2`, `fable-r4`, `fable-r5` (4 of 5 reps). Neither
stock nor thinkingcap ever checked that the site was actually enabled — they assume
the symlink from the prompt. `fable-r3` is the lone streamlined rep (read 6,
write, no symlink check).

**Verification behavior.** Strongest of the three arms. The symlink checks above
are independent confirmation that the diagnosed config is the *active* one, not
just a file on disk. `fable-r1`'s missed-file recovery is the clearest
self-correction event in the dataset.

**Waste modes.** `fable-r1` spins briefly on "where is sites-available" before
recovering — productive churn, not pure waste. Its themed model-load banner
("Transcending dimensional barriers...") is cosmetic, not behavioral. No loops,
no stalls.

### thinkingcap (ThinkingCap-Qwen3.6-27B)

**Approach shape.** Identical skeleton to stock: reason → batch of six reads →
reason → write. Thinkingcap is advertised as "token-efficient thinking," and its
reason_chars bear that out at the low end (`thinkingcap-r3` = 829 chars, the
lowest in the dataset), but its tool-call counts (7-8) match the other arms, so
the efficiency is purely in the reasoning prose, not in fewer actions.

**Thinking-trace character.** The most compressed and businesslike. It skips
stock's per-file preamble and jumps straight to the conclusion chain:

> "So the root cause is a port mismatch — nginx was reconfigured to forward traffic to port 5001, but the gunicorn backend is still listening on port 5000. The fix is straightforward: revert the proxy_pass directive in `etc/nginx/sites-available/dashboard` back to `http://127.0.0.1:5000`." (`thinkingcap-r3`)

Little narration, no alternative hypotheses explored, no self-doubt. The
efficiency claim is visible: same diagnostic yield from less prose.

**The one efficiency blot.** `thinkingcap-r1` is the only run in the dataset that
re-reads a file it already held. After reading all six files and producing a full
analysis block, it issues a second `📖 read dashboard 0.0s` and then a *second*
analysis block that nearly duplicates the first (both list the same four
findings: nginx→5001, gunicorn→5000, app.log confirms, error log shows refused).
Compare `thinkingcap-r1` block 1 ("Port mismatch! nginx proxies to 5001 but
gunicorn listens on 5000") with its block 2 ("The root cause: nginx is configured
to proxy to port 5001, but gunicorn is listening on port 5000. Port mismatch.").
That redundant read + duplicate reasoning is the single clear waste event across
all 15 runs — modest, but it undercuts the "token-efficient" framing for that rep.
No other thinkingcap rep repeats it (r2-r5 are clean single-pass).

**Verification behavior.** None beyond the r1 re-read. Thinkingcap never checks
the symlink and never re-runs a check. `thinkingcap-r3` ends with a truncated
reason box (the body renders as just "…"), a minor rendering artifact, not a
behavioral one.

**Narration habit.** Like stock, `thinkingcap-r2` through `thinkingcap-r5` emit a
user-facing "Let me read all the relevant files" message before the read batch;
`thinkingcap-r1` does not (it goes straight from reason to tool-prep).

---

## 2. Head-to-head

**(a) Discovery vs. blind trust of the prompt's file list.** Fable treats the
prompt's file inventory as a hypothesis to verify; stock and thinkingcap treat it
as ground truth. Fable-r1 explicitly second-guesses it:

> "But my search found only 5 files and none of them are under sites-available or sites-enabled. Let me look more carefully at what exists." (`fable-r1`)

Fable-r2/r4/r5 each run `ls -la .../sites-enabled/` to confirm the layout. Stock
and thinkingcap never issue a `terminal` or `find` call for discovery (0 of 5 reps
each) — they read the six named files straight away. On this task the trust was
rewarded; on a variant with a broken symlink or a renamed file, only fable's
instinct would catch it.

**Verdict on this axis:** fable more robust; stock/thinkingcap faster but fragile.

**(b) Symlink verification (does the diagnosed config actually load?).** This is
the sharpest behavioral split and the one most likely to matter on a harder
variant. Fable confirms the sites-enabled symlink is wired in 4 of 5 reps and is
the only arm to state it in the artifact:

> "sites-enabled symlink exists: `etc/nginx/sites-enabled/dashboard` → `../sites-available/dashboard`, so the site config is actively loaded." (`fable-r5`, rootcause.md)

Stock and thinkingcap never check (0 of 5 reps each). They diagnose the file on
disk and assume it is the active config — correct here, but an unverified
assumption. If the symlink had been the *real* bug (site disabled), stock and
thinkingcap would still have written "port mismatch" and passed the keyword
grader while being wrong about the actual outage.

**Verdict:** fable alone closes the "is this config live?" loop.

**(c) Single fix vs. dual-fix documentation.** Stock routinely documents an
alternative remediation; fable and thinkingcap never do. Stock-r2:

> "Option A — revert nginx to match gunicorn (recommended)" … "Option B — update gunicorn to match nginx" (`stock-r2`, rootcause.md)

Stock-r3 offers the alternative inline ("Alternatively, if the intent was to move
gunicorn to port 5001, update the systemd unit instead: `sed -i ...`"), and
stock-r5 repeats the Option A/B split. That's 3 of 5 stock reps with a documented
alternative path. Fable *reasons* about the same fork ("either change nginx... or
change gunicorn...", `fable-r2`) but resolves it in-head and ships one fix; 0 of 5
fable reps and 0 of 5 thinkingcap reps document a second option.

**Verdict:** stock produces the most actionable artifact for a human operator who
may have constraints (e.g. "I can't touch nginx right now"). The cost is modestly
longer rootcause.md files.

**(d) Self-correction from incomplete evidence.** Exactly one run in the dataset
catches its own gap and recovers: `fable-r1`'s "where is sites-available?" thread
(quoted in §1). No other run exhibits a caught-and-recovered error. Stock-r4
*states* an intent to verify ("Let me verify it looks good by reading it back
quickly") but never performs the read — verification announced, not executed.
Thinkingcap-r1 re-reads `dashboard` but for no diagnostic gain (see below).

**Verdict:** fable-r1 is the lone genuine self-correction.

**(e) "Token-efficient thinking" — prose vs. action.** Thinkingcap's advertised
efficiency shows up in reason_chars (its r3 = 829 chars is the dataset low) but
*not* in action count: its tool-call tally (7-8) matches the other arms. So the
post-train compresses narration, not steps. The clearest counter-example to the
"efficient" label is `thinkingcap-r1`, the only run that re-reads a file it
already held (`📖 read dashboard 0.0s`, a 7th read where every other run in all
three arms stops at 6) and then emits a near-duplicate analysis block. Efficiency
is therefore a prose-density property, not an action-economy one — and it is not
uniform across thinkingcap's own reps.

**Verdict:** thinkingcap is the tersest narrator; it is not the most economical
actor.

**(f) Narration-before-action habit.** Stock (r2-r5) and thinkingcap (r2-r5) each
spend a user-facing turn announcing intent ("Let me read all the relevant files to
diagnose this issue.") before firing the read batch. Fable never does this — it
goes from reason block straight to tool-prep with no intermediate user message.
Minor, but it means fable's transcripts are one user-message shorter per run.
Stock-r1 and thinkingcap-r1 also skip the announcement, so the habit is
rep-dependent, not arm-absolute.

---

## 3. Notable runs

- **fable-r1 (best run in the dataset).** The only run that did genuine discovery
  (initial `find .*` survey), caught that sites-available was missing from its
  first read set, recovered with a terminal `find -type f -o -type l`, then read
  the discovered file plus the service unit, and confirmed the symlink. 7 reason
  blocks / 3868 reason chars — the deepest investigation and the most reasoning of
  any run. Its 61s wall is the highest in the fable arm but is earned by extra
  tool round-trips, not stalls. If a single run is held up as "how to debug from a
  static capture," it is this one.

- **thinkingcap-r1 (the one efficiency blot).** After a complete six-file read and
  a full analysis block ("Port mismatch! nginx proxies to 5001 but gunicorn
  listens on 5000"), it re-reads `dashboard` (a 7th read, unique in the dataset)
  and produces a second analysis block that restates the identical four findings
  ("The root cause: nginx is configured to proxy to port 5001, but gunicorn is
  listening on port 5000. Port mismatch."). Pure redundancy — the one clear waste
  event across all 15 runs, and notably it comes from the arm marketed as
  token-efficient. No other thinkingcap rep repeats it; r2-r5 are clean single-pass.

- **stock-r1 (the timing trap).** 656s wall-clock vs ~30s for stock r2-r5. This is
  **not** the agent thinking longer: its transcript opens with a harness-level
  "🔁 Transient APIConnectionError on anthropic — rebuilt client, waiting 6s" and a
  5.25s model cold-load banner, and timing.json confirms `duration_s: 656` with
  `hermes_rc: 0`. The agent's own cadence — 3 reason blocks, 6 reads, 1 write — is
  byte-for-byte the stock template, identical to r2-r5. Anyone reading the
  aggregate "stock wall_s" column without this note would wrongly conclude stock
  is slow or indecisive on this task. Flagged explicitly so the aggregate isn't
  misread: the variance is infrastructure, not behavior.

---

## 4. Verdict

On *this specific task* — six small files, single-line root cause, all evidence on
disk, generous keyword grader — the three arms converge so tightly that the
behavioral differences barely affect the outcome. All 15 runs pass at 1.0 in a
single pass with zero errors, zero loops, zero misdiagnoses. By outcome, it is a
three-way tie.

Judged by *manner* and by which behavior would survive a harder variant of the
same task shape, **fable fits best**, narrowly:

- It is the only arm that verifies the diagnosed config is actually *active* (the
  symlink check, 4/5 reps). Stock and thinkingcap diagnose the file on disk and
  stop — correct here, but they would silently misdiagnose a variant where the
  symlink was the real fault while still passing a keyword grader.
- It is the only arm that catches and recovers from incomplete evidence
  (`fable-r1`). Neither competitor shows a single caught error.
- Its reasoning is the most investigative (narrates alternatives, voices and
  dismisses hypotheses) without becoming wasteful.

But two honest caveats temper that pick. First, fable's extra verification cost it
real tool round-trips and (in r1) the highest reasoning load in the dataset — for
a task this easy, that is arguably over-engineering. Second, **stock produces the
most useful artifact for a human operator**: 3 of 5 stock reps document an
alternative remediation (Option A/B) that fable and thinkingcap never offer. If
the deliverable is judged as a runbook a human will act on under constraints,
stock's dual-fix habit is the better manner, even though its *investigation* was
the most rote.

Thinkingcap lands as the tersest narrator — its efficiency claim holds for prose
density (lowest reason_chars at the low end) but emphatically not for action
economy (matched tool-call counts, plus the r1 redundant re-read). On a task where
the cost is reasoning tokens, thinkingcap is the cheapest *per correct answer*;
on a task where the cost is tool round-trips or where verifying "is this config
live?" matters, it is the most fragile of the three.

Net: fable investigates best, stock documents best, thinkingcap narrates most
tersely. For a static-capture diagnosis where the risk is "right answer, wrong
reason," fable's verify-the-active-config instinct is the behavior that
generalizes — but on this exact task the distinction is academic, and saying
otherwise would inflate a difference the grader could not see.

ANALYSIS COMPLETE
