# yoke-bugfix — Transcript Analysis (4 arms × 5 reps)

All 20 transcripts read. Per-run stats from `task-yoke-bugfix.md`; quotes taken
verbatim from `transcript.txt` in each run dir. Run `IDs` match the directory
suffixes in `task-yoke-bugfix.md`. The "r3 in stock35" prefix in transcripts
denotes stock35; "bakeoff-kat" / "-ornith" / "-stock35" / "-stock36" tags are
in the run dir names.

Aggregate task outcomes: kat 5/5, ornith 5/5, stock36 5/5, stock35 4/5 (r4
graded fail). Different paths to the same correct fix in 19/20 reps — one run
ran out of runway. Manner differences are the interesting story.

---

## Per-arm behavioral profile

### stock36 — the heavyweight explainer

**Approach shape.** Upfront planner → 2-3 parallel reads → reasoning recap →
patch → verify → write FIX.md → re-verify loop. Always explicitly states the
plan: "Let me first explore the project structure and run the tests." (stock36
r3 line 35). On average 22-27 messages and 11-14 tool calls. Wall time 83-113 s.

**Thinking-trace character.** Long, structured, self-correcting. The model
mostly "thinks out loud" with bullet-point-style reasoning punctuated by
explicit "now I need to…" preambles. Good at self-correction — stock36 r3
caught its own ordering bug *after* issuing the first patch: "Wait, there's a
problem. The `_reserved_ports` is set AFTER I try to use it in the while loop"
(stock36 r3 line 124), then immediately issued a corrected patch. That kind
of post-hoc self-catch shows up at least once in 4 of 5 reps.

Quoted excerpts:

- stock36 r1 lines 80–103: "Now I see the issue. Looking at lines 70-72…
  The `reserved_ports` are stored but never used to skip them when allocating
  ports. On line 306, the port is just taken as `self._next_port` and
  incremented… There's no logic that checks whether the selected port is in
  `_reserved_ports`."
- stock36 r3 lines 73–100: full prose trace covering the trace, the false
  start ("`while self._next_port in self._reserved_ports:`"), the self-catch
  ("`# This references _reserved_ports which isn't set yet!`"), then a
  controlled re-patch using a local `_reserved` variable.
- stock36 r4 lines 73–92: explicitly compares two fix-site options ("either
  initialize `_next_port` to skip past reserved ports… or add a while loop
  that skips reserved ports during allocation") and lands on the `_load()`
  patch.
- stock36 r5 lines 162–193: full prose trace of the entire mental model of the
  fix being correct, including speculation on "if `base_port+1` happens to be
  reserved too."

**Tool decisions.** Always patch-tool, never heredoc/sed. First turn frequently
fires a parallel `pytest` + `git log` (r2, r3) or `pytest` + `find` to amortize
the 20-s test latency. Reads are chunked: reads `test_router.py L90-119` and
`router.py` in parallel.

**Verification behavior.** Re-runs the full suite at least twice, often three
times. The model explicitly justifies re-runs: "The user wants me to re-run the
tests to verify the fix is correct. I already ran the tests and they passed"
(stock36 r3 line 126). Reads FIX.md back at least once. No targetted-test
re-run after the fix in any rep — full-suite only.

**Waste modes.** Over-explaining the same diagnosis up to 4 times before
patching (r3 lines 39–195 — six separate "now I see it" moments). Output
token burn is high: 2198–3073 out_tokens vs. kat's 1501–2732. FIX.md notes are
shorter than ornith's but longer than stock35/kat's (typically 16-22 lines).

---

### stock35 — short, focused, occasionally unfinished

**Approach shape.** Jump-in. Reads test failure, reads router.py, patches,
runs, writes FIX.md. 18-24 messages, 10-15 tool calls. Wall time 75-86 s on
the four passing reps, 60 s on the failed r4.

**Thinking-trace character.** Thin. The model writes 2-4 sentence diagnostic
paragraphs then moves on. Reasoning is mostly "the bug is at lines 306-307,
here's the fix" with minimal self-questioning. Almost no exploration of
alternatives.

Quoted excerpts:

- stock35 r2 line 78: "The issue is clear: `reserved_ports` is passed to the
  router constructor but the port allocation logic doesn't check against
  reserved ports. The code currently just increments `_next_port` without
  skipping over any reserved ports." — then directly to the patch.
- stock35 r3 lines 70–84: standard pattern — identify the two lines, propose
  the while-loop fix, apply.
- stock35 r5 line 60: "The test expects that when `base_port=18080` and
  `reserved_ports={18080}`, the first port assigned should be 18081 (skipping
  the reserved port). Looking at lines 306-307… The issue is that `_next_port`
  starts at `base_port` (line 71: `self._next_port = base_port`) but when a
  port is reserved, it should skip that port."
- stock35 r1 line 73: shows a 5-task todo plan but never finishes step 5
  (FIX.md).

**Tool decisions.** Always patch-tool. Uses `--tb=short`-style flags or
nothing. Often uses `grep` and `read_file` together (r1, r2, r3, r5). Never
fires a parallel test + ls.

**Verification behavior.** Re-runs full suite 2-3 times with the same
prompt-injected nudge. No fresh-verification nudges during the diag phase —
the model is confident in its fix immediately.

**Waste modes.** Empty-task ticks (r1 called the todo tool to log an
incomplete plan then crashed before step 5). r5 wrote FIX.md to the wrong
path: "✍️  write /home/coder/FIX.md 0.2s" (stock35 r5 line 234) — the literal
task asked for `~/work/FIX.md`. The grader doesn't check FIX.md presence, so
r5 still scores pass. The run **that failed** (r4) — see "Notable runs" below —
is the only stock35 rep where the harness's "Empty response after tool calls"
recovery kicked in.

---

### ornith — the thorough scribe

**Approach shape.** Same arc as stock36 but heavier, with longer tool-result
recaps and more user-message-laden turns. 27-32 messages, 14-17 tool calls.
Wall time 92-106 s.

**Thinking-trace character.** Long-form prose, very chatty. The model
narrates its own thinking in detail ("Now I understand the issue", "Found
it. Now I see it clearly — _next_port starts at base_port, but never skips
past any ports that are in _reserved_ports"). Self-correction is verbal, not
in-code: orn-r3 cycles through multiple "now I see it" / "Now I understand"
moments before finally patching. The model often restates the conclusion
several times before tool action.

Quoted excerpts:

- ornith r1 lines 30–37: "The failing test is
  `test_new_model_skips_reserved_controller_port`… This is a bug in the
  router where it's not respecting the `reserved_ports` set. Let me look at
  the relevant source files to diagnose and fix this."
- ornith r1 lines 100–138: hallucinated a `cat > /tmp/fix.py <<EOF` heredoc
  python rewrite inside the thinking trace, then immediately issued the
  correct `mcp__patch` tool call instead. Tool call won; the hallucinated
  command never executed.
- ornith r3 lines 87–195: cycle of explanations — "the bug is clear" →
  "Now I see it clearly" → "The fix: after assigning… check if `port` is in
  `self._reserved_ports`. If it is, increment further" → "Actually wait —
  I need to think about this more carefully" → "The cleanest fix: after each
  port assignment, skip reserved ports" → final patch.
- ornith r4 lines 22–28: opens with "Let me start by running the test suite
  to find the failing test, then diagnose and fix the issue" — same
  planner preamble.

**Tool decisions.** Same patch-tool. Tends to fire multiple `read` + `grep`
+ `sed` calls (orn-r1: `sed -n '80,120p'` then `sed -n '295,320p'`) — belt
and suspenders. Writes FIX.md notes 27-44 lines long (longest in the
bakeoff). Writes a detailed "Why it happened" / "Fix is monotonic and
idempotent" closer (orn-r1 lines 232–234).

**Verification behavior.** Three re-runs of the full suite. orn-r1 also does
a target-test re-run mid-conversation. orn-r5 does an `ls -la` to verify the
FIX.md path.

**Waste modes.** The hallucinated heredoc in orn-r1. The repetitive
"now I see it" cycle in orn-r3. Think-chars 6467–9075 — by far the highest
in the bakeoff — but it's spending that budget on narration, not extra
searches.

---

### kat — small, efficient, accurate

**Approach shape.** Jump-in, parallelize the first two reads. 23-25 messages,
10-12 tool calls. Wall time 76-123 s.

**Thinking-trace character.** Bare. The model emits 2-5 line reasoning
paragraphs that conclude with the fix plan in code form. Then it acts.
Almost no exploration of alternatives; one quick self-check.

Quoted excerpts:

- kat r1 lines 38–60: "Let me look at the failing test and the router code
  to diagnose the issue" → read test_router.py L90-114 + grep reserved_ports
  → "The bug is clear: `reserved_ports` is stored but never used when
  computing the next port. At line 72, `_reserved_ports` is saved, and at
  lines 306-307, `_next_port` is assigned and incremented without checking
  if it's in `_reserved_ports`."
- kat r2 lines 32–59: same compact flow, ends with a 3-line inline patch
  proposal.
- kat r3 lines 49–60: this is the rep where kat fixes in __init__ instead of
  _load. Reasoning: "The fix: in `__init__`, after setting `self._next_port =
  base_port`, advance past any reserved ports." Both fixes pass the test
  (the failing test only exercises the first model, so init-time and
  alloc-time skips are observationally equivalent).
- kat r4 lines 58–71: clean diag → "Found it. The `_load` method at lines
  306-307 does… It never skips reserved ports. The fix should skip over
  reserved ports when assigning."

**Tool decisions.** Always patch-tool. Fires two `read_file` + `grep` calls
in parallel on the first diagnostic turn (r1, r2, r3, r5). Uses
`pytest --tb=short -q` once.

**Verification behavior.** Re-runs pytest 2-4 times. The model treats the
prompt's "You edited code" inject as a request for fresh evidence each turn.
kat r2 line 167: "The system is asking me to verify again - it seems like
the verification state was reset. Let me run the tests again to confirm."
This is the "verification reflex" — re-run on every prompt.

**Waste modes.** Notable: the final user-facing message in multiple reps
shows broken tokenizer rendering. kat r1 lines 211–391: each Python token in
the response is rendered one character per line ("All / r u n / 2 3 9 /
p a s s e d …"). Same in kat r4 lines 213–243. This is a presentation-layer
artifact, not a model failure, but it consumes tokens and looks ugly. Also:
kat's FIX.md gets written correctly and contains real content (typically
22-52 lines depending on the run), more concise than ornith but more
substantive than stock35.

---

## Head-to-head differences

### 1. Where the fix is applied — three different sites

Every arm produces a logically correct fix, but the *site* of the fix varies:
the monotonic allocation site (`_load`, lines 306-307), the constructor
(`__init__`, lines 70-74), or both. The test only observes the first model
allocation, so init-time skipping and alloc-time skipping are
observationally equivalent. It tells you which default the model converges
to.

- **stock36** consistently goes init-style on r1 (`while self._next_port in
  self._reserved_ports: self._next_port += 1`), then alloc-site on r2/r3/r4,
  with r3 taking an extra patch after realizing init-style had an ordering
  bug — re-patches to a local `_reserved` variable. Alloc-site only on r5.
- **stock35** consistently alloc-site. r1, r2, r3, r5 all hit `_load`.
- **ornith** alloc-site on r1, r2, r3, r4, r5. The pattern is stable.
- **kat** alloc-site on r1, r2, r4, r5; **init-site on r3** (only one
  across all 20 reps). k-r3 is the only divergence from the dominant
  alloc-site style.

Net: 19/20 fix at the allocation site; 1/20 (kat r3) at init; 1/20 (stock36
r1) at init; 1/20 (stock36 r3) attempts init then corrects to a hybrid.
All produce passing tests.

### 2. Reasoning budget vs. message count

ornith has the highest think-char counts (6467-9075) and the most messages
per rep (27-32). stock36 is the second-most verbose in both (think 2218-6022,
msgs 21-27). kat is leanest (think 1036-3567, msgs 23-25). stock35 sits
between stock36 and ornith on messages but low on think (2289-5134). The
ratio is interesting:

- ornith: 250 think-chars per message average.
- stock36: 230 per message.
- kat: 75 per message.
- stock35: 180 per message.

kat compresses its reasoning into terse sentences that conclude with code;
ornith tells a story.

### 3. Tool sequencing — parallelism vs. serial

- **ornith** is the only arm that repeatedly fires 2-3 commands in parallel
  on the diagnostic turn (r1 lines 53-55: `sed -n '80,120p'` + `grep -n
  'reserved_ports|next_port|base_port'` + `sed -n '295,320p'`). It will also
  fire `pytest` + `find src/` in parallel (r1 lines 41-43).
- **stock36** does parallel `pytest` + `find` (r3 line 31-32) once but is
  mostly sequential.
- **stock35** rarely parallelizes — r3 is the only one to issue a parallel
  read+search.
- **kat** parallelizes one tool pair (read+grep or read+search) on turn 2-3
  across all 5 reps.

### 4. Verification reflex — how many pytest re-runs

Re-runs of the full suite after the initial fix, per rep. Counts include
the prompt-injected "edited code" re-runs. From the per-run stats table
plus transcript counts:

- **kat**: 2-4 re-runs across all 5 reps. The most consistent verifier.
- **ornith**: 2-3 re-runs. orn-r1 does 3 (line 175, 280, 315). orn-r5 does
  4 plus 1 targeted test (line 286).
- **stock36**: 2-3 re-runs. r1 does 3. r2 does 3. r3 does 2.
- **stock35**: 1-2 re-runs across reps. r1 (10 tool calls total), r2
  (12), r3 (15), r5 (13). r4 = 0 (didn't reach fix).

Verdict: the more verbose arms verify slightly more often. **kat has the
tied-highest verification frequency despite the lowest reasoning budget.**
That's discipline, not corner-cutting.

### 5. Hallucination risk

orn-r1 is the only transcript with a substantive hallucination: a fully
formed `<function=mcp__terminal>` heredoc python-rewrite call embedded
inside the thinking trace (r1 lines 102-138). The model then issued the
correct `mcp__patch` tool call. The hallucinated shell command never
executed. No other arm shows comparable fabrication in thinking. Inference:
ornith's longer reasoning budget gives it more room for stray tool-syntax
imitation.

### 6. FIX.md depth

File presence and length varies:

- **ornith** writes the longest notes — 27-44 lines. orn-r1 is 44 lines,
  orn-r3 is 29, orn-r4 is 27. The notes have a "monotonic and idempotent"
  prose closer (orn-r1) and a discussion of why the regression slipped
  through CI.
- **stock36** writes 16-22 line notes. Tighter, still complete.
- **kat** writes 22-52 line notes depending on rep. k-r2 is 52 lines (the
  longest in the bakeoff) — that rep was the most thorough k-explanation
  of the bug history ("A previous change stored the reserved-port set but
  forgot to wire it into the allocation logic. The test suite had a test
  asserting the skip behaviour, but it was previously passing (possibly
  because base_port != any reserved port) and broke once the combination
  hit.").
- **stock35** writes 22-36 line notes when it writes the file — but
  stock35 **r1 and r5** wrote to the wrong path (`/home/coder/FIX.md`
  instead of `/home/coder/work/FIX.md`), and r4 never wrote FIX.md at all.
  See "Notable runs" below.

### 7. Recycling-prompt pattern (across arms)

All four arms exhibit the same tail-end loop pattern after the task is
"done": the system injects a "you edited code in this turn" nudge, the
model re-runs pytest, says "Fresh verification complete," and exits. This
isn't a per-arm difference — it's environmental — but it's the dominant
shape of the last 30% of every transcript. ornith r1 lines 235-346 is the
archetypal example: 5 separate re-runs of the same suite with
near-identical user-facing messages.

---

## stock35-r4 — what it missed

The single failure. Diagnosis was correct; action was incomplete.

stock35-r4 followed the same path as r1, r2, r3: pytest → see failure →
grep `_next_port|_base_port` → diagnose at lines 306-307 → patch. But the
patch landed and then the harness hit "↻ Empty response after tool calls —
using earlier content as final answer" (r4 line 168). The session
terminated cleanly (harness rc=0, 60 s, agent done) but **without
re-running pytest** and **without writing FIX.md**.

Compare to stock35-r1 which graded pass despite having only 10 tool calls
(no FIX.md written): the test+manifest grader doesn't check FIX.md
presence. stock35-r4 only differed in *what* it cut. The transcript shows
the patch *was* applied at line 134, but the post-patch re-run was
swallowed by the empty-response handler and the run was declared done
with exit=0 — except the harness's grader then ran pytest fresh and saw
the test still failing. Wait — that's inconsistent with my reading. Let
me re-check.

Re-reading stock35-r4 line 134-167: the patch is applied at line 134
showing the diff (lines 137-143 are review-diff output). Then "Now let me
run the full test suite to confirm everything passes" (line 148). Then the
Hermes user-message (lines 152-154) "Now let me re-run the full test suite
to confirm:" — followed by an empty-response marker at line 168. **No
pytest tool call was emitted between the patch and the empty-response.** The
model's intent to re-run was never realized.

Compare to stock35-r3 (passing, 86 s, 24 msgs): same pattern, but stock35-r3
*actually called* `mypy -m pytest` between the patch and the FIX.md write.
stock35-r3 line 195: "💻 $ .venv/bin/python -m pytest 8.4s" — the call
materialized. **stock35-r4 didn't emit the tool call** — the
internal-model intent to verify apparently failed to surface as a tool
call. This is the manner failure.

Inference: with sampling temp 1.0 top_p 0.95, stock35 sometimes fails to
emit the verification tool call after the patch. In r4 the model's
thinking ended with "Now let me run the full test suite" but the
"tool-call turn" boundary got disrupted (likely by the harness's empty-
response handler, possibly because the model's prior prose was being
chunked weirdly across line breaks). In r1 the model just skipped both
verification AND FIX.md and still passed because the grader doesn't
check FIX.md. In r2, r3, r5, the call went through.

Behavioral takeaway: stock35-r4 has **the same shape as the other passing
stock35 reps for the first 11 turns**. The diagnoser is fine. The
plan-then-act sequence goes wrong between "I'll re-run the suite" and the
actual tool call. A more verbose arm that emits a longer assistant turn
(ornith, stock36) has *more chances* to emit tool calls before the
turn-boundary; kat's compact turns have fewer such failure points but also
emit reliably.

---

## Other notable runs

- **ornith r1's hallucinated heredoc** (lines 100-138): a `cat > /tmp/fix.py
  <<'EOF' … python3 /tmp/fix.py` shell fragment rendered into the
  thinking trace as if it were a tool-call instruction, then immediately
  replaced by a correct `mcp__patch` call. The hallucinated command never
  ran. The model recovered cleanly. This is the only fabricated tool-call
  content across all 20 transcripts.

- **stock36 r3's ordering-bug self-catch** (lines 123-181): the model
  issued a patch that referenced `self._reserved_ports` before the field
  was set, then caught the bug in the next turn and re-patched with a
  local `_reserved` variable. This is the only run where the model
  visibly *corrects its own patch mid-conversation*. It's a stronger form
  of verification — synthesis check before the test re-runs.

- **kat r3's init-site fix** (lines 49-86): the only init-site fix outside
  stock36's first-patch attempt. kat's reasoning: "In `__init__`, after
  setting `self._next_port = base_port`, advance past any reserved ports."
  Observationally equivalent for the failing test, but philosophically
  different — it addresses the problem "once at startup" vs "every
  allocation".

- **ornith r3's "now I see it" cycle** (lines 23-300): the model emits at
  least 6 separate diagnostic statements before the patch. "Found the
  bug…" / "The bug is clear…" / "The fix: …" / "Actually wait — I need
  to think about this more carefully…" / "The cleanest fix…" / final
  patch. The most narration-per-action ratio in the bakeoff.

- **stock35 r5's path error** (line 234): wrote FIX.md to
  `/home/coder/FIX.md` instead of `/home/coder/work/FIX.md`. Task says
  explicitly "~/work/FIX.md". The grader doesn't check the file, so the
  run passes. But the literal task wasn't satisfied. This is the only
  path-target error across the 19 FIX.md-writing runs.

- **kat r1's broken-render final message** (lines 391–500+): the assistant's
  closing prose has each Python token on its own line (e.g. line 211
  "All" / 212 "" / 213 "tests" / 214 "pass" / 215 "(" / 216 "2" / 217 "3" /
  218 "9" / 219 "passed"). Same artifact appears in kat r2, r3, r4 but
  milder. It's a tokenizer/sampler presentation bug, not a model error —
  but it consumes tokens and visually pollutes the conversation tail.

---

## Verdict — which behavior fits this task best

The task shape: small, well-bounded regression; one failing test; the
correct fix is mechanical (3-4 lines, one while loop); grading is rigid
(test passes, full suite green, tests/ untouched — FIX.md presence not
checked); fixed answer known to the same 3 lines in 19/20 reps. The
*correct* behavior is: read the test, read the source, write the patch,
run pytest twice, write a brief note.

**kat's behavior fits this task shape best.** 12 tool calls per rep, ~2K
out-tokens, 4-5 reasoning paragraphs per rep, fix-it-correctly,
verify-it, write-a-note, stop. It does the minimum (correctly) and
doesn't burn tokens producing five different phrasings of "I see the
bug now." The discipline shows up in three concrete ways:

1. The verification reflex is consistent across all 5 reps (kat r2, r3,
   r4, r5 each re-run pytest at least twice after the patch).
2. The FIX.md is always written to the correct path (contrast stock35 r1
   and r5).
3. The fix is always minimal and correct.

**stock36 is the second-best fit for this shape**, but not because of
efficiency — rather, its diagnostic thoroughness is real but
overshooting. The stock36 r3 ordering-bug self-catch shows the upside
of its longer thinking: it can find its own mistakes pre-test. But in 4
of 5 reps the longer thinking was just narration, and the wall-time
tax was ~10-20% higher than kat for the same outcome.

**ornith's behavior is the worst fit for this shape** despite its 100%
pass rate. Three of ornith's reps spend the first 25-40% of the
conversation re-stating the diagnosis in different words before
patching. orn-r3's six-step "now I see it" cycle is the canonical
example. The think-char count (6.5K-9K per rep) is two-to-three times
what's needed. The hallucinated heredoc in orn-r1 is a direct
consequence of the longer-thinking budget: more reasoning turns, more
chances for stray tool-syntax imitation.

**stock35 is high-variance.** 4/5 passing reps with the lowest wall
clock and the smallest footprint — but the failure mode is intrinsic:
the model sometimes narrates an intent to verify without emitting the
tool call. The path-target error in r5 (and the missing FIX.md in r1)
also signals that stock35 is less reliable about *completing* the task
specification. The grader doesn't check FIX.md, so this doesn't show up
in the pass-rate, but if the task shape were "you must produce FIX.md
in the right place," stock35 would be 2/5.

**Bottom line, manner-focused:**
- For this specific task (3-line fix, well-defined grader), kat is the
  disciplined runner.
- stock36 has the highest ceiling on harder tasks (its self-catch
  capability is real) but overshoots on small ones.
- ornith shows the cost of agentic-RL post-training on Claude-Code-style
  harnesses applied to a non-Claude-Code harness — its narration density
  is mismatched to the budget.
- stock35 is the smallest and fastest successful arm on average but the
  least reliable about completing the spec.

If the task shape shifted to "longer diagnostic arc, multi-file
refactor, write several test stubs," I'd expect stock36 to lead. On this
task, kat's discipline wins.

ANALYSIS COMPLETE
