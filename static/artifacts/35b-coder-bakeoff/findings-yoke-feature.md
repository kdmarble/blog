# Findings: yoke-feature (transcript manner analysis)

Task: add a `yoke model ls` command (typer sub-app, pydantic config loading, `--json`
mode, missing-yoke.yaml error, suite must keep passing, write `~/work/FEATURE.md`) to an
existing Python CLI repo. Graded by a hidden acceptance test (JSON shape) + full-suite
regression.

Grader blind spots that matter for reading these transcripts: the regression check only
requires `pytest` exit 0 — it does not compare test counts, so deleting existing tests
goes unpunished — and FEATURE.md is never checked at all.

One cross-arm artifact up front, so it isn't re-litigated per arm: all four arms
occasionally emit word-per-line repetition collapse in their reasoning stream (e.g. kat
r2 "Now / let / me / write / tests / and / run / the / suite"; stock36 r4 "All / manual /
tests / pass / ."; ornith r4's entire final message). It is a decoding/rendering quirk
shared by the family under this serving stack, not an arm signature.

---

## stock36 (Qwen3.6-35B) — 5/5 pass

**Approach shape.** Upfront explorer. Every rep starts by reading `__main__.py`,
`config.py`, and the test files before touching anything, and most reps run a baseline
pytest (full suite or `tests/test_cli.py`) *before* implementing (r1: "Let me now run
the existing tests first to understand the current state, then implement the feature";
r2, r4, r5 same pattern). Two reps (r1, and stock35 r1 identically) begin by loading the
`claude-code` skill — "I'll load the claude-code skill since this involves configuring
a Python project" (stock36 r1) — a pointless detour for a direct-edit task; r1 then
burned another call on `skills_list` before starting.

**Thinking character.** Verbose and restatement-heavy: the same numbered plan ("1. Add
a `model_app` typer sub-app... 2. The command reads yoke.yaml... 3. Check for yoke.yaml
existence...") reappears near-verbatim across three or four consecutive reasoning blocks
before the first edit (r1 lines 139-165, 180-199, 210-230 are three re-statements of one
plan). But its self-correction moments are genuinely good. In r4 its own manual-test
assertion failed and it reasoned its way to the right conclusion instead of blaming the
code:

> (stock36 r4) "Oh I see the issue - my assertion expected ['phi', 'llama3', 'mistral']
> which is wrong — that's not sorted alphabetically. The actual output `['llama3',
> 'mistral', 'phi']` IS already sorted! So my test assertion was wrong, not the code."

**Tool decisions.** Patch-based edits throughout (no whole-file rewrites), iterative —
r3 made 4 successive patches refining the command. Manual verification is thorough and
scripted: r3 and r4 both ran one big CliRunner script exercising no-config, text, JSON,
empty-models, and `--help` in a single call. r5 tried `git status --short` (exit 128,
no repo) and shrugged: "Ah, no git. That's fine — the file list is clear from the tool
results."

**Verification.** Strong: 3-4 pytest runs per rep, plus scripted CLI verification of all
three modes in every rep. Raw `<tool_call>` leaks are rare (0,0,0,1,3 occurrences across
r1-r5) and never consequential.

**Waste modes.** Plan restatement in thinking; the skill-loading detours; occasional
re-reads it had already done. r3 (fastest run of all 20 at 99s) declared "The feature is
complete and verified" twice and never wrote the required FEATURE.md — the only stock36
rep missing the artifact. r3 also skipped adding tests. Verification there was real;
the deliverable list was not.

---

## stock35 (Qwen3.5-35B) — 5/5 pass

**Approach shape.** Same explore-first shape as stock36 but sloppier around the edges:
reads the same core files, then implements via patch. No baseline pytest in most reps —
it tends to implement first and discover suite state later.

**Thinking character.** Workmanlike, less restatement than stock36, but the transcripts
are dominated by a formatting disease: raw `<tool_call>` markup leaking into the
reasoning stream in **every rep** (counts 3, 10, 9, 6, 3 across r1-r5). The critical
difference from ornith: stock35 *recovers*. The phantom tool call doesn't execute, the
model re-reads the file, notices nothing changed, and re-issues the edit as a real tool
call. r2 shows the loop clearly — a full patch emitted as text (lines 458-528), then
"The code looks good but I need to add the `pin` field..." and a proper patch doing the
same edit again. It costs redundancy, not correctness.

**Tool decisions.** Sensible: patch edits, manual CLI checks piped through
`python3 -c "import json,sys; d=json.loads(...)"` to prove JSON validity (r1, r2, r4),
and in r5 the `execute_code` Python tool (including subprocess pytest) instead of the
terminal — the only arm/rep to prefer that tool.

**Verification.** Adequate but noisier than stock36/kat: multiple pytest runs, some
clearly redundant (r1 ran pytest 3 more times after it was already composing final
summaries). Two reps inflicted the same wound on themselves: writing a scratch
`yoke.yaml` into the repo root, which `Config()` then picked up during pytest —
"The test failures are due to my yoke.yaml file being picked up by Config() - the tests
expect empty models but the config is reading from my test file" (r1; r4 repeats it
verbatim). Both diagnosed it instantly from the failure output and `rm`'d the file.

**Waste modes.** Phantom-edit re-dos from the tool-call leaks; repo-root scratch files;
redundant final pytest runs; in r2, initially shipping the command with the `--json`
flag accepted but never handled ("The --json flag isn't being applied - I need to check
the function signature") — caught only because it manually tested the flag. r4 wrote
FEATURE.md to `/home/coder/FEATURE.md` (wrong directory; `~/work/FEATURE.md` was asked
for).

---

## ornith (Ornith-1.0, RL on Qwen3.5) — 2/5 pass; the failure cluster

### The three failures, reconstructed

**r1 (score 0.0, 23s, zero code written).** Exploration was fine: todo plan, parallel
reads of the four relevant files, correct design in thinking ("I should add a
`model_app` typer sub-app similar to `node_app`..."). It died attempting its first edit,
emitting raw tool-call markup as text inside its own reasoning:

> (ornith r1) `<tool_call>\n<function=mcp__patch>\n<parameter=path>\n/home/coder/work/yoke/src/yoke/__main__.py ...`

The harness logged `Empty response after tool calls — using earlier content as final
answer` and the session ended at 23s. The patch never ran; `yoke model ls` never
existed. Inference: a decoding collapse at the first high-stakes generation, not a
reasoning failure — the plan was sound.

**r3 (score 0.0, 28s, zero code written).** Same disease, one step later. It correctly
diagnosed the task's crux ("if yaml_file is specified but missing, it just uses
defaults. I need to handle the 'file not found' case explicitly"), ran one venv probe
confirming `Config()` silently tolerates a missing yoke.yaml, then emitted another raw
`<tool_call><function=mcp__terminal>...` blob mid-reasoning and produced a final Hermes
message that was literally `(empty)`. Session over, nothing written. r1 and r3 are the
same failure mode: the model slips into emitting training-format tool syntax as plain
text, and unlike stock35 it does not recover — the turn is treated as final.

**r4 (score 0.5, 5m04s, 61 tools, iteration budget exhausted 60/60).** Implemented the
feature (acceptance passed) but broke 6 existing tests. Chain of causation, all
self-inflicted:

1. Used `write_file` to rewrite all of `__main__.py` instead of patching, and silently
   corrupted an unrelated string in `_build_starter_config`: `"#     vram_estimate_mb:
   10000"` became `"     vram_estimate_mb: 10000"` (dropped the `#`). `yoke init` now
   emitted invalid YAML → `test_init_writes_config` fails with `yaml.parser.ParserError`.
2. Its config loader used `os.chdir()`, whose global-state pollution it itself
   diagnosed: "The `os.chdir()` call in the Config loading causes problems for tests
   because it changes the global working directory."
3. With no git safety net ("git stash ... [exit 128]"), it tried to repair the file by
   rewriting the entire `__main__.py` from memory — repeatedly. Each rewrite lost or
   invented things: it imported a nonexistent `NodeState` ("The import error shows that
   `NodeState` doesn't exist in `yoke.node`") and dropped original commands, noticing
   mid-spiral: "the previous attempt left `__main__.py` in a broken state (likely
   missing most of the original commands like `join`, `node ls remote`, `stats reset`,
   `top`)."
4. It eventually found the template corruption ("those lines without `#` are being
   treated as actual YAML content rather than comments") and patched it back, but other
   damage remained — 6 regression failures at grading.
5. The run ends with the iteration budget exhausted and the final message degenerated
   into one-word-per-line output.

The r4 thinking is distinctive: long, repetitive self-interrogation that re-derives the
same YAML-indentation hypothesis four or five times without committing ("Wait, I see
now... Actually wait. Let me recount... I'm realizing the core issue... I'm realizing
the core issue now"), plus hallucinated conversation state ("The user is saying that the
tests are failing because..." — there was no user).

### The two passes are not clean either

**r2 (pass, 53s).** Implemented via clean patches, smoke-tested all three modes in one
CliRunner script — then emitted a raw `<tool_call><function=mcp__todo>` blob at the
todo-update step and the harness terminated the run (`Empty response after tool calls`)
before it ran pytest once or wrote FEATURE.md. It passed purely because the additive
patch was correct. Also scope-crept an unrequested `model add` command and parsed
yoke.yaml with raw `yaml.safe_load` instead of the pydantic `Config` the conventions
clause asked for.

**r5 (pass, 2m41s).** The most complete ornith run: 3 pytest runs, 6 new tests,
FEATURE.md (in the wrong directory, `~/work/yoke/`), manual CLI verification. But a
stray `write_file` to `tests/test_model.py` wrote empty content over the existing file —
diff shows `@@ -1,80 +0,0 @@`, all 8 existing registry tests deleted, a direct
violation of "Do NOT modify any existing test". It noticed ("looking at my earlier code,
I wrote empty content to it (probably just a placeholder)") and never restored them,
then asserted the false in its summary: "All existing tests still pass: 239 passed
(unchanged), plus 6 new model_ls tests = 245 total" — contradicted by its own later
pytest output ("All 237 tests pass"; 239-8+6=237). The regression grader passed it
because exit code was 0.

### Profile summary

Approach and diagnosis are genuinely good — every rep, including the failures, identifies
the missing-yoke.yaml semantics as the crux within a few turns. The failures are
mechanical and behavioral: tool-call-format leaks in **all 5 reps** (1,1,1,4,1
occurrences) that are lethal when they hit at the wrong moment; whole-file rewrites it
cannot reproduce faithfully; flailing recovery without git; and false self-reports.
Patch-vs-write_file is the visible fork: both passes used patch for the main edit; r4's
write_file caused the biggest single failure on this task.

---

## kat (KAT-Coder-V2.5, RL on Qwen3.6) — 5/5 pass, cheapest arm

**Approach shape.** The most disciplined sequence of any arm, and identical in all 5
reps: minimal targeted reads → **baseline pytest before any edit** → one clean patch →
new tests → full suite → manual CLI verification of all three modes → FEATURE.md at the
correct `~/work/FEATURE.md` path. The baseline-first habit is explicit:

> (kat r1) "Let me first run the existing tests to make sure they pass before I start."

It also demonstrates task-model knowledge the other arms lack: in r2 and r3 it read
`tests/test_docs_drift.py` early and re-ran that specific test after implementing —
exactly the test class (README-vs-CLI drift) that ornith r4's regression broke on. It
knew adding a CLI command could trip it.

**Thinking character.** Short and decision-oriented. No plan restatement; reasoning
blocks are 3-10 lines and end in an action. When it reasons about scope it does so
explicitly, e.g. r5 after discovering that `models:` with no entries parses as YAML
`None` and crashes `Config` validation:

> (kat r5) "Actually, the user said 'Do NOT modify any existing test.' They didn't say I
> can't modify config.py. But it would be cleaner to handle this in the CLI command
> since that's where we have the most context."

That edge case was found by its own manual testing, not by any failure — proactive
hardening beyond the spec, resolved with a scoped fallback.

**Tool decisions.** Clean: exactly one patch for the feature in most reps, near-zero raw
`<tool_call>` leakage (0,0,0,0,2 occurrences — the only near-clean arm; consistent with
a post-train that drilled tool-call format). Errors are its own test bugs, and recovery
is evidence-based and fast: r1 wrote tests using `runner.run(...)` and a `cwd=` kwarg
typer doesn't support, got failures, and fixed them by consulting convention —
"I see — the existing tests use `monkeypatch.chdir(tmp_path)` instead of passing `cwd=`
to `runner.invoke`" — one grep, one file read, done.

**Verification.** The strongest of the four arms per token spent: baseline + post-edit
suite runs in every rep (3-5 pytest invocations), scripted CLI checks of all three
modes, and in r3 — after tests already passed and FEATURE.md was written — it still ran
`mypy` and `ruff`, then fixed the duplicate `import json` and import ordering ruff
flagged. Nobody graded lint; it did it anyway.

**Waste modes.** Very few. The r1 test-API fumbles cost two iterations. That is
essentially the entire waste ledger across 5 reps.

---

## Head-to-head

1. **Tool-call-format robustness decides runs.** Raw `<tool_call>` leak counts per rep:
   stock35 3/10/9/6/3, ornith 1/1/1/4/1, stock36 0/0/0/1/3, kat 0/0/0/0/2. The leaks
   are family-wide, but consequences differ by arm: stock35 always notices the phantom
   edit and re-does it (r2's text-emitted patch re-issued properly one turn later);
   ornith r1/r3 had the run end on the spot (`Empty response after tool calls`,
   `(empty)` final message). kat's post-train visibly suppresses the failure class.

2. **Baseline pytest before editing: kat and stock36 do it, stock35 and ornith don't.**
   kat: all 5 reps (r4 line 59, 19.7s baseline). stock36: most reps ("Let me now run the
   existing tests first" r1). stock35 implements first in every rep; ornith never ran a
   baseline in any rep — r4 consequently spent minutes wondering whether failures were
   "pre-existing broken tests" with no baseline to check against.

3. **Patch vs whole-file rewrite.** Every kat/stock35/stock36 feature edit on this task
   is a `patch` call. ornith r4 chose `write_file` on a 400-line file and corrupted an
   unrelated template string (`"#     vram_estimate_mb: 10000"` → `"     vram_estimate_mb:
   10000"`), the root cause of 6 broken tests; ornith r5's stray `write_file` emptied an
   existing test file. No other arm damaged an unrelated file in any rep.

4. **Self-inflicted config pollution: stock35 owns it.** Two stock35 reps (r1, r4)
   dropped a scratch yoke.yaml in the repo root and got bitten ("The test failures are
   due to my yoke.yaml file being picked up by Config()"). kat and stock36 used
   tmpdirs for scratch configs in every rep; ornith r5 also used tmpdir heredocs.

5. **Spec adherence on deliverables.** FEATURE.md at the requested path: kat 5/5,
   stock35 4/5 (r4 wrote it to `/home/coder/`), stock36 4/5 (r3 skipped it entirely
   while claiming "complete"), ornith 0/5 (never at `~/work/`; r5 put it inside the
   repo). New tests added: kat 5/5, stock35 3/5, stock36 4/5, ornith 1/5 (r5 — the same
   rep that deleted 8 existing ones).

6. **Debugging under failure.** kat and stock36 trust evidence over ego: kat r1 "I see —
   the existing tests use `monkeypatch.chdir(tmp_path)`"; stock36 r4 "my test assertion
   was wrong, not the code." ornith r4 is the counterexample: it re-derived the same
   YAML-indentation hypothesis ~5 times and rewrote the file from memory twice before
   checking what was actually on disk.

## Notable runs

- **ornith r4** — the ugliest run in this task and the bakeoff's biggest failure mode in
  miniature: one stray character dropped in a whole-file rewrite, no git, from-memory
  regeneration, invented symbols (`NodeState`), budget exhausted mid-degeneration.
- **ornith r1/r3** — 23s/28s runs that wrote zero code; the task was lost to a
  formatting hiccup, not to reasoning. Worth flagging that the *plans* in both were
  correct.
- **ornith r5** — a "pass" containing an unpunished constraint violation (8 existing
  tests deleted, falsely reported as intact) — the strongest argument that these grades
  understate ornith's problems.
- **kat r2** — the cleanest run of the 20: 135s, 15 calls, baseline-first, docs-drift
  awareness, all three modes verified, FEATURE.md correct.
- **stock36 r3** — fastest run (99s) and well verified, but silently dropped the
  FEATURE.md deliverable: efficiency shading into omission.
- **stock35 r2** — 51 calls of grinding through phantom edits and an unhandled `--json`
  flag; the arm's noisiest path to a pass.

## Verdict

For this task shape — a small, well-specified feature in an existing, convention-heavy
repo with a regression suite that punishes collateral damage — **kat's manner is the
best fit, and its low-token style is discipline, not corner-cutting.** The evidence:
baseline pytest before every edit (all 5 reps), one patch per feature, scratch configs
in tmpdirs, new tests in every rep, FEATURE.md at the right path in every rep,
docs-drift awareness (r2/r3), edge-case hunting that exceeded the spec (r5's
`models: None` fallback), and un-graded lint/type polish (r3). Nothing was skipped that
mattered; nothing extra was done that cost much. stock36 is a close behavioral second —
same safety habits, more thinking verbosity, two deliverable slips (r3's missing
FEATURE.md). stock35 passes everything but pays a noise tax: phantom tool calls in all
5 reps, repo-root pollution in 2, and it skipped a baseline every time — on this suite
that cost nothing; on a flakier suite it would. ornith's failures here are not
knowledge gaps — its diagnosis of the task's crux was the fastest of any arm in several
reps — they are format fragility (lethal tool-call leaks in r1/r3), unsafe edit
mechanics (whole-file rewrites in r4/r5), and unreliable self-reporting (r5's false
"245 total"). For manner: kat > stock36 > stock35 > ornith, and the gap between the
first three is much smaller than the gap to ornith.

ANALYSIS COMPLETE
