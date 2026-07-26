# Findings — yoke-bugfix task

Task: diagnose a single failing pytest (`test_new_model_skips_reserved_controller_port`), fix the root cause in source with the smallest correct change, re-run the full suite, and write FIX.md. All 15 runs passed the grader (target test green, full suite 239 passed / 4 skipped, tests/ untouched).

The bug itself was trivial and uniform across arms: `LocalRouter._load()` in `src/yoke/router.py` (lines ~306-307) assigned ports via a monotonic counter (`port = self._next_port; self._next_port += 1`) without ever consulting `self._reserved_ports`, which was stored in `__init__` (line 72) but never read at allocation time. Every arm converged on adding a `while port in self._reserved_ports` skip loop. The interesting variation is in *how* each arm reasoned about the fix and in its tool-execution and verification habits — analyzed below.

---

## 1. Per-arm behavioral profiles

### stock (Qwen3.6-27B)

**Approach shape.** Consistent jump-in planner. All 5 reps open the same way: a short reasoning block that enumerates the task steps, then a *parallel* first turn that runs `pytest` and a `find`/`search_files` for `*.py` together (e.g. r1: `┊ 💻 $ .venv/bin/python -m pytest` + `┊ 🔎 find *.py`; r2: `find . -type f -name "*.py"` + pytest; r4: `ls /home/coder/work/yoke/src/` + pytest). After the failure is found it reads `test_router.py` and `router.py` together in one batched read. The pattern is stable and efficient: pytest → read(test+source) → patch → pytest → write FIX.md.

**Thinking-trace character.** Stock produces the most explicitly structured reasoning of the three arms. It restates the task as a numbered plan up front (r1: "Let me break down this task: 1. Run the test suite... 5. Write a note at ~/work/FIX.md"), then traces the bug mechanistically with code excerpts and line numbers. r1 reasons through the fix cleanly:
> "The problem is that `self._next_port` starts at `self._base_port` (18080), but port 18080 is reserved, so the router should skip it... The fix is to add a check before assigning the port that skips any reserved ports by incrementing `_next_port` until it finds an available one." (stock-r1)

r4 shows the most deliberative reasoning of any rep across arms — it explicitly weighs lazy vs proactive placement of the guard:
> "I'm weighing whether to handle this lazily only during assignment or proactively in both places... I'm going to add a loop that increments the port past any reserved ones, using `while` instead of `if` to handle consecutive reserved ports." (stock-r4)

It also considers an edge case ("this logic has to work whether we're starting from the initial position or after eviction already moved us forward") — no other rep reasons about post-eviction state. r2 actually simulates the loop execution step-by-step before patching ("`_next_port` starts at `18080`... `port += 1` → `port = 18081`... `_next_port = 18082`. So the decision has `port=18081`. That matches the test expectation.").

**Tool decisions.** Stock's fix variant is the cleanest and most idiomatic across reps: it consistently mutates a *local* `port` then reconciles the counter at the end —
> `port = self._next_port` / `while port in self._reserved_ports: port += 1` / `self._next_port = port + 1` (stock-r1, r2, r4, r5 — diff hunk `@@ -304,7 +304,9 @@`)

This is the 2-new-line form. The exception is r1, which instead pre-advances `_next_port` in a loop *before* reading it (`while self._next_port in self._reserved_ports: self._next_port += 1` then the original 2 lines unchanged) — functionally equivalent but a 3-line add rather than a replacement. r1 is also the only rep that includes a comment line ("Skip past any ports in the reserved set (e.g. controller port)") in the diff. Stock uses `patch` (not `write_file`) on the source in all 5 reps — never rewrites the file.

**Verification behavior.** This is stock's most distinctive (and wasteful) trait. In all 5 reps the model runs the full suite green *immediately after the patch*, writes FIX.md, and then — when the harness's verification nudger fires — runs pytest *again* (often twice more, once full and once `pytest tests/test_router.py`). r1 runs pytest 3 times after the fix (full, then router-only, then full again); r2 runs it twice post-fix; r3 runs router-only then is pushed to full; r4 and r5 both re-run after already having a green run. Stock's reasoning shows it *knows* the re-run is redundant but complies anyway, and occasionally argues back:
> "The system note seems to be a generic reminder from the plugin. My work is fully verified — the full suite passed at 239/243 with 0 failures. No further action needed." (stock-r1)
> "The system is asking me to run verification. Let me do that now." (stock-r2)

**Waste modes.** Redundant pytest re-runs driven by the verification nudger (adds 1-2 extra tool calls per run). Otherwise tight — no exploration of unrelated files, no dead-end diagnosis paths, no search loops.

---

### thinkingcap (ThinkingCap-Qwen3.6-27B)

**Approach shape.** Same skeleton as stock but slightly more spare. All 5 reps open with a one-line reasoning block ("The user wants me to debug a failing test... Let me start by running the test suite") followed by a single `pytest` call — notably *without* the parallel `find`/`ls` that stock pairs it with. After the failure it reads the test (sometimes via `read_file` with a line range like `L90-114`, sometimes a grep for `class LocalRouter` first) then reads `router.py`. thinkingcap is the only arm where reps sometimes search for the class *before* reading the file (r2: three searches — `find LocalRouter|reserved_ports|def route`, `find LocalRouter`, `grep class LocalRouter` — before the read).

**Thinking-trace character.** The "token-efficient thinking" post-train is visible. thinkingcap's reasoning blocks are noticeably terser than stock's and rarely restate the task plan; they jump to the finding. Representative:
> "Found the failure: `test_new_model_skips_reserved_controller_port`... The test expects port 18081 but got 18080. Let me look at the test and the source code." (thinkingcap-r1) — a 2-sentence block where stock would have written a paragraph.

Where stock deliberates, thinkingcap declares. Compare stock-r4's lazy-vs-proactive weighing to thinkingcap-r5, which considers three candidate loop shapes in one block, briefly rejects two, and commits:
> "Or more elegantly, we could just skip reserved ports after incrementing... Wait, that's not quite right either. Let me think about this more carefully... Actually, let me simplify: just do a while loop that advances both... This is cleaner." (thinkingcap-r5)

This is the deepest reasoning in any thinkingcap rep, and it is still more compact than stock-r4's equivalent. The advertised efficiency holds: per the stats table, thinkingcap has the lowest median reason_chars (1402-3040) and think_chars of the three arms on this task.

**Tool decisions.** Here thinkingcap diverges from stock in a meaningful way. Two of its five reps (r3, r5) produce a *different, more awkward* fix variant — one that mutates `self._next_port` inside the loop and re-reads it into `port`, rather than mutating a local:
> `port = self._next_port` / `while port in self._reserved_ports:` / `self._next_port += 1` / `port = self._next_port` (thinkingcap-r3, r5)

This is functionally correct but reads less cleanly (it touches `self` twice per iteration and leaves a redundant assignment). r5 actually layers a *third* increment on top — it keeps the trailing `self._next_port += 1` after the loop, producing a 3-new-line patch where the loop body advances `_next_port` and the post-loop line advances it again. The reasoning in r5 even walks through three drafts of the loop before landing on this form. r1, r2, r4 use the stock-style local-`port` variant. So thinkingcap is the least *consistent* arm in the exact patch shape it emits (2-of-5 reps differ).

**Verification behavior.** Same verification-nudger pattern as stock: run green after patch, then re-run once or twice more when nudged. thinkingcap-r2 is the most compliant ("The system is asking me to run verification again. Let me do that."). The reasoning acknowledges redundancy explicitly: "Already verified — the full suite passed in the previous turn (239 passed, 4 skipped). No action needed" (r3).

**Waste modes.** Redundant post-fix pytest re-runs (same as stock). The extra `find LocalRouter` search round in r2 (two no-op searches before grepping the class) is a minor search inefficiency unique to this arm. No diagnostic dead-ends.

---

### fable (Fable-Fusion-711)

**Approach shape.** fable matches stock's parallel-first-turn habit (pytest + find/ls together) in r2, r4, r5 but diverges in r1 (single pytest) and r3 (single pytest then read test). It is the only arm to occasionally take *two* reads of the test before reading the source (r3 reads `test_router.py` then `test_router.py L78-117` then greps `class LocalRouter|reserved_ports|base_port` then reads `router.py`). fable also shows the broadest spread in tool-call count (7-9) and is the only arm with a genuine runtime hiccup.

**Thinking-trace character.** fable's reasoning is the most discursive and occasionally self-interrupting. It tends to restate findings multiple times across consecutive reasoning blocks where stock/thinkingcap state them once. In r1, after the first pytest, it second-guesses the exit code:
> "Wait, exit_code is 0 even though it says \"FAILED\". That might be because pytest exits with non-zero only on failures but this output shows the tail... Let me check the actual exit code properly." (fable-r1)

This is a minor misread (pytest *does* exit non-zero on failure; the harness rendered exit 1 but fable fixated on a display ambiguity), and it prompts a redundant second pytest run before it proceeds — a form of paranoia the other arms don't show. r1 is also the run with the API blip ("🔁 Transient APIConnectionError on anthropic — rebuilt client, waiting 6s").

fable's reasoning about the fix itself is solid but chatty. r2 works through the counter-update semantics carefully:
> "Wait, I need to think about this carefully. `self._next_port` is used for monotonic assignment - each new model gets a higher port. If we skip a reserved port when assigning, we should also update `_next_port` past it." (fable-r2)

r3 also deliberates on whether changing `_next_port` advancement could break monotonicity ("Wait, but that would change how we advance _next_port. Let me think more carefully...").

**Tool decisions.** fable is the *least consistent* arm in patch shape across reps:
- r1, r5: pre-advance `_next_port` in a `while` loop *before* the read (stock-r1 style), leaving the original 2 lines intact (`+2` lines).
- r2, r3: the stock-style local-`port` variant with trailing `self._next_port = port + 1` (`+2`/-`1` hunk).
- r4: a unique `if/else` variant —
  > `if port in self._reserved_ports: port += 1; self._next_port = port + 1` / `else: self._next_port += 1` (fable-r4)

  This is the only single-`if` (not `while`) fix in the entire 15-run set. It works for the test (one reserved port) but would fail to skip *consecutive* reserved ports — a latent correctness gap the other arms' `while` loops avoid. fable-r4's reasoning never considers the consecutive-reserved-port case.

fable also self-caught a rendering bug: r2 wrote a FIX.md whose ```diff block was mangled (lost the `while` keyword and indentation), then *noticed* by re-reading FIX.md and rewrote it:
> "The diff in the FIX.md got mangled (lost `while` and indentation). Let me rewrite it cleanly." (fable-r2)

This is the only artifact-verification act across all 15 runs.

**Verification behavior.** Same nudger-driven double/triple pytest re-runs as the other arms. fable-r1 is notable for running pytest a fourth time in the final turn (the transcript ends mid-stream with character-by-character token rendering of "239 passed").

**Waste modes.** fable is the chattiest arm and it shows: highest aggregate llm_calls (per brief) and the r1 outlier at 831s wall time (vs ~85-111s for its other reps) driven by the API retry plus an extra pytest re-run. The two-reads-of-the-test habit (r3) and the FIX.md rewrite (r2) are fable-specific. The single-`if` patch (r4) is a subtle correctness-quality dip masked by the trivial test.

---

## 2. Head-to-head

**(a) Patch-shape consistency.** stock emits the cleanest, most uniform fix: 4 of 5 reps produce the identical local-`port` variant (`port += 1` in loop, `self._next_port = port + 1` after). thinkingcap splits 3 clean / 2 with the redundant `self._next_port += 1; port = self._next_port` body (thinkingcap-r3, thinkingcap-r5). fable is the most scattered: three distinct shapes across 5 reps (pre-advance loop, local-`port` loop, and the unique single-`if`/`else`). Evidence:
> stock-r2 diff: `port = self._next_port` / `- self._next_port += 1` / `+ while port in self._reserved_ports:` / `+     port += 1` / `+ self._next_port = port + 1`
> thinkingcap-r5 diff: `while port in self._reserved_ports:` / `self._next_port += 1` / `port = self._next_port` (plus a trailing `self._next_port += 1`)
> fable-r4 diff: `if port in self._reserved_ports:` / `port += 1; self._next_port = port + 1` / `else:` / `self._next_port += 1`

**(b) Consecutive-reserved-port robustness.** Only stock reasons about this edge case explicitly. stock-r4: "using `while` instead of `if` to handle consecutive reserved ports". fable-r4's single-`if` would silently hand out a port if two reserved ports were adjacent — a latent bug the test (which reserves exactly one port) does not catch. thinkingcap and fable both default to `while` but neither articulates *why*; stock does.

**(c) Reasoning verbosity vs. depth.** thinkingcap's "token-efficient" claim is behaviorally real on this task: it reaches the same diagnosis in roughly half the reason_chars of stock (median 2140 vs stock's ~2381, and far below r4's 2919). Yet the depth is comparable — thinkingcap-r5 works through three loop drafts in fewer characters than stock-r4 spends weighing lazy-vs-proactive. fable spends *more* tokens than either (r3 reason_chars 4033, the highest single value) without more depth, largely due to restating findings and deliberating aloud ("Wait, I need to think about this carefully").

**(d) First-turn tool batching.** stock and fable both fire pytest + a structure probe (`find`/`ls`/`search_files`) in the same turn. thinkingcap does not — it opens with pytest alone in all 5 reps, then adds the structure/read calls on turn 2. This costs thinkingcap ~1 extra round-trip but keeps its first turn minimal. Compare:
> stock-r1: `┊ 💻 $ .venv/bin/python -m pytest` + `┊ 🔎 find *.py` (parallel)
> thinkingcap-r1: `┊ 💻 $ .venv/bin/python -m pytest --tb=short` (alone)

**(e) Redundant verification (all arms).** Every arm exhibits the same post-fix loop: pytest-green → write FIX.md → nudger fires → pytest-again. This is a harness-interaction artifact, not an arm trait, but the *response* differs. stock pushes back in reasoning ("The system note seems to be a generic reminder... No further action needed") yet still complies. thinkingcap complies without protest. fable complies and sometimes over-complies (r1's extra run). None of the three simply stops — all add 1-2 redundant tool calls.

**(f) Paranoia / self-interruption.** fable uniquely second-guesses transient signals mid-run: r1 doubts the pytest exit code ("Wait, exit_code is 0 even though it says FAILED") and re-runs; r2 catches its own mangled FIX.md and rewrites it. stock and thinkingcap never second-guess a clean tool result. This makes fable more self-correcting on artifacts but more wasteful on already-correct signals.

---

## 3. Notable runs

- **stock-r4 (best single run).** The most deliberative reasoning of any rep: weighs lazy vs proactive guard placement, chooses `while` over `if` explicitly for the consecutive-reserved-port case, reasons about post-eviction state, and emits the clean local-`port` patch. If one transcript were shown as a model answer, this is it.

- **fable-r1 (ugliest run).** 831s wall (7x its siblings), opened with a transient API error, misread the pytest exit code and re-ran it, then over-verified with repeated pytest calls. Still passed cleanly, but the path was the noisiest in the set — and it's the run that inflates fable's "+24% model calls" aggregate from the brief.

- **fable-r4 (latent-correctness concern).** The only `if`/`else` patch instead of `while`. Passes the grader because the test reserves a single port, but would fail on adjacent reserved ports. The reasoning never engages the multi-reserved case. A behavioral yellow flag that the aggregate pass-rate hides.

- **thinkingcap-r5 (most loop-design iteration).** Drafted three loop variants in reasoning before committing, and ended with the most over-engineered patch (loop that re-reads `self._next_port` *plus* a trailing increment). Illustrates that "token-efficient thinking" did not translate to the most economical *code*.

---

## 4. Verdict

For *behavioral fit* on this task — manner, not outcome — **stock is the best match.** The yoke-bugfix task rewards (1) fast convergent diagnosis, (2) a minimal, idiomatic, edge-case-aware patch, and (3) disciplined verification. stock delivers all three most consistently: uniform patch shape across 4/5 reps, explicit reasoning about the `while`-vs-`if` and consecutive-reserved-port edge cases (stock-r4), and clean `patch`-not-`write_file` source edits. Its only real weakness — redundant pytest re-runs under the verification nudger — is shared by all three arms and is harness-driven.

thinkingcap is a close second: its token efficiency is genuine and welcome, and its diagnosis is just as fast. It loses on code-economy (2/5 reps emit an awkward double-mutation patch) and on first-turn parallelism (pytest-only opener). For a task where the bug is this obvious, thinkingcap's terseness is arguably the *better* fit — but its patch inconsistency is the differentiator.

fable is the weakest fit here. It reaches the right answer every time but with the most variance: three patch shapes, one latent `if`-instead-of-`while` gap (fable-r4), the worst token economy, a self-caught FIX.md rendering bug, and one severe wall-time outlier (fable-r1). Its discursive, self-interrupting reasoning style (the "Wait, let me think..." habit) is well-suited to ambiguous creative tasks but is pure overhead on a clean single-cause regression like this one.

**Where the difference doesn't matter:** all three arms diagnosed the identical root cause from the identical pytest output, located the bug in the same two lines of the same file, and shipped a passing fix in 6-9 tool calls. On a trivial regression, the behavioral spread is real but the task is too easy for it to change the outcome. The differentiators above would only become decisive on a harder, multi-cause, or edge-case-sensitive bug.

ANALYSIS COMPLETE
