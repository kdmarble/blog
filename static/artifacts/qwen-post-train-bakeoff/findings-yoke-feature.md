# Bakeoff Findings: yoke-feature

**Task:** Add a `yoke model ls` (and `yoke model ls --json`) command to the `yoke` Python/typer CLI project. Read `yoke.yaml` from cwd, print models as text or JSON; handle missing config with non-zero exit and a yoke.yaml error. Constraint: must NOT modify existing tests; full suite must pass.

**Battery:** 15 runs = 3 arms x 5 reps, all passing. wall_s / tool_calls / think_chars stats in the task file. Focus here is the *qualitative* layer: how each arm reasoned and used tools.

**Reading order:** r1 per arm first to find each arm's stable pattern, then reps r2-r5 to verify "always/never" claims before writing them. All 15 transcripts read end-to-end. Quantitative fingerprints in this report come from grepping the transcripts (skill loads, pytest invocations, test files written) so the behavioral claims are anchored to evidence.

---

## 1. Per-arm behavioral profiles

### 1.1 stock (Qwen/Qwen3.6-27B, unspecialized)

**Approach shape.** Upfront planner who then goes deep before writing code. Reads broadly — `find *` for project layout, reads `config.py`, `__main__.py`, `model.py`, `test_cli.py`, `conftest.py` — before any edit (r1 reads 6 files before patching). The plan is articulated in thinking, not chat, and is unusually long for a "small" task: r1 lays out 6 numbered steps ("1. explore … 2. look at config.py for ModelEntry schema … 5. run tests … 6. write FEATURE.md") inside the first reasoning block. Across all 5 reps stock consistently front-loads exploration, often re-reading the same files it has already read.

**Thinking-trace character.** Verbose and meta-cognitive. Median think_chars ≈ 9000 (highest of the three arms). r1 (think_chars=9067) and r5 (think_chars=13915) both spend long blocks reasoning about the pydantic-settings YAML source — explicitly working through version-dependent behavior. r1: "Looking at the yaml_file parameter documentation, it seems like `pydantic_settings` supports optional YAML files, but I should check which version is actually being used since this feature was added in v2.6.0." It also invents a precise rule about typer prefix stripping (assumes `model_ls` under `model_app` produces command name `ls`), then has to recover: in r1 the actual typer behavior produces `model-ls` because the prefix stripping only applies to a sub-app prefix matching the function name. The model notices only after running tests and getting "No such command 'ls'". Recovery is methodical: it reads the file, checks `registered_groups`, then re-reads surrounding code to compare with `device_ls`. This is a representative stock pattern: over-reason about edge cases, hit a real edge case, then debug methodically.

Representative thinking excerpts from r1:

> "Looking at the Config class setup, it has `yaml_file="yoke.yaml"` in its settings configuration, so I need to understand what happens during initialization when that file isn't present and whether it raises an error or uses defaults."

> "I need to check if `pydantic_settings` raises an error when the yaml_file doesn't exist or if it just uses defaults. Let me trace through how Config() is actually being used elsewhere in the codebase — looking at `_build_app_and_config`, it seems like `yoke.yaml` is expected to be present, but I need to understand what happens when it's missing."

> "Wait — looking more carefully, the error says 'No such command ls' on the `model` sub-app. This means typer sees the `model` sub-app but doesn't see the `ls` command within it. The problem might be module import caching..."

**Tool decisions.** Mostly fine-grained: single-line `read_file` with offset ranges, one `find *` to survey layout, individual `patch` calls. r1's first patch attempt introduces a real bug — it puts `@app.command()` and `def top()` after the orphaned docstring "Launch the TUI dashboard" by replacing the wrong range, and the model notices ("I accidentally left orphaned docstring/lines from the original `top()` function after replacing its header"). r1 also tries `Config(yaml_file=...)` directly which fails because pydantic-settings treats `yaml_file` as a SettingsConfigDict key, not an init kwarg; stock recovers by *trying multiple workarounds in sequence* (subclass with `model_config | {"yaml_file": ...}`, then just `Config()`). Across reps stock occasionally over-engineers config loading, but recovers.

**Verification behavior.** Re-runs the suite repeatedly. Per-run `pytest -q` invocation counts across the 5 reps: 5, 4, 7, 5, 4 (avg 5.0 — the highest of the three arms). Stock's r3 hit the harness's "unverified" prompt multiple times and re-ran in response. r1 wall_s = 1000s, dominated by repeated pytest invocations AND a transient API retry at the start ("Transient APIConnectionError on anthropic — rebuilt client, waiting 6s before one last primary attempt" — a harness-level retry, not a model failure, but it inflates wall_s). Stock trusts its tests less than the other arms and re-runs more often.

**Waste modes.** Over-reading — r1 reads `__main__.py` four separate times. The recurring typo/bug cycle around `model_app` registration: r1 creates the typer sub-app twice in the same file (once after `node_app`, once after `init`), then has to delete one; the wasted patches cascade. The `find . -name '*.pyc' -delete` invocations in r1 produce exit 1 each time — stock debugs this rabbit hole for two full turns before moving on. r3 has an unusual artifact: a long assistant-message stretch with "..." filler where the model emits no real content, just dots — an apparent generation glitch.

**Verification count tendency (verified across all 5 reps):** stock runs `pytest -q` an average of 5.0 times per run (5, 4, 7, 5, 4); thinkingcap averages ~3.8 (3, 4, 3, 5, 4); fable averages 5.0 (7, 4, 6, 4, 4). Stock and fable tie on verification count, but stock's *content* per verification is heavier (full `pytest -q` each time vs fable's mix of `pytest -q` and one-off smoke tests). Always verifies before declaring done.

**Skill loads.** Zero across all 5 reps. Stock is the only arm that never reaches for `skill_view` in this battery.

### 1.2 thinkingcap (bottlecap/ThinkingCap-Qwen3.6-27B, "token-efficient thinking")

**Approach shape.** Compressed upfront planner. The first thinking block in r1 is six lines ("Let me start by understanding the codebase structure, existing conventions, and then implement the feature. I'll explore the project in parallel.") and immediately followed by three parallel tool calls (`read_file __main__.py`, `read_file config.py`, `search_files tests.*\.py$`). Across reps thinkingcap almost always batches the initial reconnaissance into a single message of parallel reads, then issues the implementation patch on the next turn. r5 — the smallest transcript in the battery at 564 lines / 13 tool calls — is the cleanest example: a single exploration pass, one patch that adds the command, one manual smoke test, write the summary doc.

**Thinking-trace character.** Compressed and operationally focused. Median think_chars ≈ 5500 (lowest of the three arms); r5 is 2206, the shortest single thinking trace across all 15 runs. Self-correction is brief: r1 thinks "0" before continuing, which appears to be a literal one-character reply indicating "nothing more to add." Reasoning is oriented to the next action ("Let me check what happens when yoke.yaml doesn't exist during config loading") rather than meta-cognition about the codebase. When thinkingcap does reason at length it's because the problem is genuinely tricky: r3 (think_chars=9060) reasons at length about pydantic-settings' `yaml_file` semantics, r4 (think_chars=7740) reasons at length about a test that used `monkeypatch.chdir(Path("/tmp"))` and produced an exit-code-0 instead of 1.

Representative thinking excerpts:

From r1:
> "The key question is: what happens when `yoke.yaml` doesn't exist? Let me check."

> "Good - Config() works fine without yoke.yaml (uses defaults). So for `yoke model ls`, I need to explicitly check if the file exists before loading config. Let me implement this now."

> "The JSON output doesn't match the spec... My default output uses a rich table instead of plain lines. The spec says 'one line per model'. Let me change it to match the spec exactly."

That third excerpt is the rare case where thinkingcap does a self-correction in writing — it initially defaulted to a rich table (matching what `device ls` does) and then realized the spec literally says "one line per model." It then patches the function to remove the table.

**Tool decisions.** Light, efficient, and pragmatic. In r1 thinkingcap uses `cat << 'EOF' ... EOF` heredocs to write temporary yaml files to /tmp for manual smoke testing, then pipes through `python3 -c "import sys,json; d=json.load(sys.stdin); ..."` to validate JSON parseability — a one-liner that stock and fable do not attempt (they rely on the test suite). This manual smoke-test pattern shows up in r1, r2, r3, r4. thinkingcap rarely over-engineers: when it hits the pydantic `yaml_file` question, it just calls `Config()` and lets pydantic auto-discover (no subclassing). It avoids the orphan-docstring trap by writing a clean, minimal patch the first time.

The trade-off: thinkingcap sometimes *under*-engineers. r5 is 13 tool calls and never adds a single new test (the only arm to skip test additions entirely). The grader's hidden test file still passes for r5 because the regression check is "existing tests still pass" — but no positive coverage of the new behavior was added. (thinkingcap r5 instead loads the `claude-code` skill, which is irrelevant — see Skill Loads below.)

**Verification behavior.** Trusts earlier results more than stock. Per-run `pytest -q` counts: 3, 4, 3, 5, 4 (avg 3.8, lowest of the three arms). Manual smoke testing via heredoc+yaml+`python -m yoke` is consistent across r1-r4 — a thinkingcap fingerprint. No arm other than thinkingcap runs `yoke model ls` interactively before declaring done. r4 is the exception: it has one re-run after noticing a test used `monkeypatch.chdir(Path("/tmp"))` instead of `tmp_path`, then patched the test.

**Waste modes.** Minimal. r1 imports `json` inside the function body (line `import json` placed inside the `if json_output:` branch around line 200), which works but is unusual — other arms put it at module top. r5 has a single extremely brief reasoning block where the model literally writes `0` (one character) and `requir` (truncated) — partial text appears to leak from the tokenizer's perspective; not a behavioral problem but an artifact. thinkingcap is the only arm that reliably avoids the `model-ls` vs `ls` typo cycle: across all 5 reps it writes `@model_app.command("ls")` (explicit name) on the first try.

**Skill loads.** One across 5 reps — r5 loaded `claude-code` (irrelevant to a yoke feature implementation, same kind of irrelevant-load as fable r1's `github-pr-workflow`). r5's loading is the first thing the model does, before any code exploration. So thinkingcap is not immune to spurious skill loads, but it's rare (1/5).

### 1.3 fable (DavidAU Fable-Fusion-711, creative-merge / not coding-targeted)

**Approach shape.** Jumps in fast, talks big, makes a hash of it. The first thinking block in r1 is conventional ("Let me start by understanding the existing codebase structure, conventions, and how similar commands are implemented"), but the very next action is to **load the `github-pr-workflow` skill** (line 47 of the r1 transcript). This is a hallucinated skill load — `github-pr-workflow` is irrelevant to a local Python typer feature implementation. r2, r3, r4, r5 do not repeat this mistake, so r1 is an outlier on the skill-load axis.

After the r1 skill load, fable still does the right reads (`config.py`, `__main__.py`, `pyproject.toml`) — but then attempts a single patch that introduces a real bug: it overwrites the wrong range and leaves orphan code (the original `top()` function's docstring/body and `device_app` registration survive in a broken state around line 460). It notices the orphan code in the next reasoning block ("I accidentally left orphaned docstring/lines from the original `top()` function after replacing its header"), patches it, but the patch itself is wrong again — it removes the docstring but leaves the function body. Then a third read, then a full `write_file` rewrite that produces a clean version but accidentally duplicates `model_app` registration. Then a fourth patch to remove the duplicate. r1 ends up doing: explore → wrong patch → fix-orphan patch → fix-the-fix patch → write_file → remove-duplicate patch. That's 4-5 file operations on `__main__.py` before any test runs.

This pattern is **r1-specific**, not fable-typical. r2 is clean: explore → patch → test → fix test (missing `import json`) → done. r3 has a 5-patch sequence for the spec-compliance self-correction (rich table → tab-separated text), but no botched first patch. r5 goes off into pydantic-settings internals trying to subclass with `settings_customise_sources` to override the YAML file location — an extreme over-engineering that works but is 30+ lines for a problem the other arms solve with 0-1 lines.

**Thinking-trace character.** Median think_chars ≈ 5800, similar to thinkingcap, but the *content* is qualitatively different. fable's reasoning is verbose about the project structure and verbose about what it's about to do, but not deep about the actual mechanics. Representative r1 thinking: "I have enough context now. Let me implement the `model ls` command following the existing patterns: 1. Create a `model_app` typer subcommand (like `device_app` and `node_app`) 2. Add a `ls` command to it that: ..." — this is a planning recital, not analysis. The model writes out an explicit 1-2-3 plan in its own thinking every turn, the way a junior engineer might talk through their work aloud, then executes it.

When fable does reason about a tricky detail (the `yaml_file` parameter being in SettingsConfigDict vs an init kwarg in r5), the reasoning is correct but late — it arrives at the right answer after the wrong patch has already failed. fable is the only arm that *explores* pydantic-settings' `settings_customise_sources` (r5) to attempt a YAML-source override, and the only arm that adds a class-method to override the source tuple.

Representative thinking excerpts:

From r1:
> "Let me load relevant skills first, then explore the codebase."

> "I have enough context now. Let me implement the `model ls` command following the existing patterns: 1. Create a `model_app` typer subcommand (like `device_app` and `node_app`) 2. Add a `ls` command to it that: ..."

> "Pydantic Settings doesn't accept yaml_file as an init kwarg like that; it's part of model_config. I need to load the YAML file via YamlConfigSettingsSource directly or use Config.__init__ with the yaml_file config properly set."

> "Wait — I have a duplicate `model_app` definition now. Let me check the file and fix it."

That last one appears *after* a `write_file` rewrite — fable notices the duplication only after the write, never before. By contrast, stock notices it during a re-read and patches before running tests.

From r4 (when fable has the cleanest sequence):
> "The issue is that `models:\n` parses as `None`, not an empty dict. I'll fix that by using `{}`."

From r5 (over-engineering at its peak):
> "Need to override `settings_customise_sources` dynamically. Best approach: create a custom Config subclass for this command that uses only our YAML source and init defaults."

**Tool decisions.** Heavy on writes and re-writes. fable is the only arm that reaches for `write_file` to replace a multi-hundred-line file wholesale mid-task (r1's full rewrite of `__main__.py`). fable also tends to place `import json` *inside* function bodies (r2, r4, r5), which works but is unusual. fable is the most likely arm to add its own tests: it writes `tests/test_model_ls.py` in r1, `tests/test_cli.py` in r2 (appending), `tests/test_cli.py` in r4 (appending). These test additions are useful (the grader's hidden test essentially replaces them) but reflect fable's instinct to "do more" rather than "do less." fable is also the only arm that ran into the "empty models mapping parses as None" pydantic issue (r4) — testing fix required changing `"models:\n"` to `"models: {}\n"`.

**Verification behavior.** Inconsistent. r1 runs pytest five times after the final code state (lines 574, 740, 856, 1024, 1039 in the transcript) — partly because the model itself is unsure whether the code is stable ("The system is telling me code was edited but verification status is unverified"). r2 runs once and stops. r3 runs twice. Per-run `pytest -q` counts: 7, 4, 6, 4, 4 (avg 5.0 — tied with stock). fable seems to respond to harness prompts where stock and thinkingcap ignore them.

**Waste modes.** Fable's waste is structural, not lazy. The skill-load detour in r1 is unique to fable (and one instance of thinkingcap r5). The orphan-code bug is unique to fable r1. The duplicate-`model_app` registration after `write_file` is unique to fable r1. r5's `settings_customise_sources` override is a 30-line workaround for a problem every other arm solves with `Config()`. Across all 5 reps fable is the most likely to: (a) load an irrelevant skill (r1), (b) produce a botched first patch (r1), (c) write_file a large file mid-task (r1), (d) over-engineer pydantic config handling (r5), (e) add tests in a brand-new file rather than appending (r1). Yet fable passes every time, which is itself the finding: it recovers robustly but expensively.

---

## 2. Head-to-head behavioral differences

The five reps per arm produce stable-enough patterns to make these claims verifiable across arms, with citations to specific runs.

**H1. Spurious skill loads.** fable and thinkingcap both occasionally load skills that have nothing to do with the task; stock never does.
- fable r1: `┊ 📚 skill github-pr-workflow 0.4s` — for a local Python typer feature implementation.
- thinkingcap r5: `┊ 📚 skill claude-code 0.4s` — for a yoke Python feature.
- stock: zero `preparing mcp__skill_view` lines across all 5 transcripts (verified by grep).

Inference: stock seems to treat skills as opt-in rather than default-first-action. fable treats them as a productivity ritual. thinkingcap is mostly token-efficient (median think_chars lowest) but not immune to the same ritual.

**H2. Recovery from a botched first patch.** fable r1 produced one and needed 5+ file operations on `__main__.py` to recover; the other arms (and other fable reps) do not.
- fable r1: patch → patch (orphan) → patch (wrong fix) → write_file (full rewrite) → patch (remove duplicate). 5-6 ops on `__main__.py` before any test run. Wall_s 255s.
- stock r1: patch (orphan) → patch (fix the orphan but break docstring) → patch (fix that) → read → patch (Config yaml_file kwarg failure) → patch (subclass workaround) → write_file (rewrite) → patch (remove duplicate). 8+ ops on `__main__.py`. Wall_s 1000s.
- thinkingcap r1: one clean patch, then pydantic YAML test, then patch, then test, then a single corrective patch to switch from rich table to one-line-per-model. 4-5 ops total.
- fable r2-r5: clean patches; the r1 problem does not repeat.

This is the cleanest arm-level behavioral difference. fable r1's structure was unique — recovery cost was ~5x normal — but stock r1 had a similar (more painful) recovery arc. thinkingcap and "stock/fable after the first attempt" both behave well.

**H3. Manual smoke-test pattern.** thinkingcap is the only arm that runs `python -m yoke model ls` interactively before declaring done, piping through `python -c "import sys,json; json.load(sys.stdin)"` to validate JSON parseability.
- thinkingcap r1 line 296: `💻 $ /home/coder/work/yoke/.venv/bin/python -m yoke model ls --json | python3 -c "import sys,json; d=json.load(sys.stdin); print(...)"`
- thinkingcap r2 line 269: same pattern.
- thinkingcap r3: same pattern (heredoc + cd + python -m yoke + pipe to json.loads).
- thinkingcap r4: same pattern.
- stock: never. Runs `pytest -q` and trusts it.
- fable: never. Runs `pytest -q` and trusts it.

Inference: thinkingcap validates end-to-end with the actual CLI binary; the other arms only validate via the in-process CliRunner. This is robust — it catches stdout-purity bugs that CliRunner might miss — but it costs a few extra tool calls.

**H4. Test additions.** fable and stock both add tests for `model ls`; thinkingcap r5 does not.
- fable: writes tests in r1 (6 tests, `tests/test_model_ls.py`), r2 (6 tests appended to `tests/test_cli.py`), r4 (4 tests appended to `tests/test_cli.py`). r3 and r5 add nothing.
- stock: writes tests in r2 (11 tests, `tests/test_model_ls.py`), r3 (7 tests, `tests/test_model_ls.py`), r4 (7 tests appended to `tests/test_cli.py`), r5 (4 tests appended to `tests/test_cli.py`). r1 appends to `tests/test_cli.py`.
- thinkingcap: writes tests in r2 (6 tests, `tests/test_model_ls.py`), r3 (3 tests, `tests/test_model_ls.py`), r4 (5 tests, `tests/test_model_ls.py`). r1 and r5 add nothing.

Total new tests added per arm (sum of mentions in transcripts/FEATURE.md):
- fable: ~10-16 across the 5 reps
- stock: ~29 across the 5 reps
- thinkingcap: ~14 across the 5 reps

Stock is the heaviest test adder by total count. fable is the most likely to put tests in a brand-new file rather than appending. thinkingcap r5 is the only single run that didn't add any test (because it shipped clean with the regression check).

**H5. Verification re-runs.** stock and fable both re-run the full test suite 5.0 times per run on average; thinkingcap runs it 3.8 times per run.
- stock: 5, 4, 7, 5, 4 = 25 total
- fable: 7, 4, 6, 4, 4 = 25 total
- thinkingcap: 3, 4, 3, 5, 4 = 19 total

This roughly tracks the median think_chars pattern (stock 9000 > fable 5800 ≈ thinkingcap 5500). More thinking → more verification.

**H6. Self-correction discipline.** thinkingcap is the only arm to self-correct the rich-table → one-line-per-model in writing (r1: "The JSON output doesn't match the spec... My default output uses a rich table instead of plain lines. The spec says 'one line per model'. Let me change it to match the spec exactly."). Stock and fable also correct, but mostly as a side-effect of test failures rather than spec re-reading. Fable r3 and r5 *both* note the rich-table/spec-mismatch and correct; fable r4 ships the rich table and notes in the FEATURE.md "The spec example is informal; following existing code conventions was prioritized." So fable is internally inconsistent on this — sometimes follows spec literally (r3, r5), sometimes follows existing pattern (r4). thinkingcap always follows spec literally when the spec is explicit (only r1's self-correction is documented; r2, r3, r4 all write tab-separated plain text by default).

---

## 3. Notable runs

**Most ugly run: fable r1 (`/data/bakeoff/results/bakeoff-fable--yoke-feature--r1--1785052578`).** Wall_s 255s, 56 messages, 52 tool calls. The orphan-code bug, the duplicate `model_app` registration after `write_file`, the load of an irrelevant skill, and 5 pytest invocations — all stacked. Yet it passes 245 tests. fable's recovery is robust but expensive.

**Most painful run: stock r1 (`/data/bakeoff/results/bakeoff-stock--yoke-feature--r1--1785039946`).** Wall_s 1000s (10x any other run in the battery), 62 messages, 58 tool calls. The "Transient APIConnectionError on anthropic" retry at the start burns 6+ seconds. Then stock goes into the orphan-code cycle, the `model-ls` typo cycle, and the `Config(yaml_file=...)` over-engineering cycle. The transcript has a 200+ line stretch of debugging between lines 500-1200 where the model is reading `__main__.py`, deleting `.pyc` files (which fails with exit 1), inspecting `registered_groups`, and patching the decorator from `@model_app.command()` to `@model_app.command("ls")`. Yet it passes 243 tests. stock's recovery is the most expensive of the three arms.

**Cleanest run: thinkingcap r5 (`/data/bakeoff/results/bakeoff-thinkingcap--yoke-feature--r5--1785048076`).** Wall_s 127s, 36 messages, 30 tool calls. One exploration pass, one clean patch, one manual smoke test (mkdir + rm + python -m yoke), write summary doc, run pytest twice. Minimal but: it loaded the `claude-code` skill before doing anything else (which is spurious), and it adds zero tests for `model ls` (which is why the grader's hidden test is the only coverage). thinkingcap r5 is the "least is more" run.

**Most over-engineered run: fable r5 (`/data/bakeoff/results/bakeoff-fable--yoke-feature--r5--1785053922`).** Wall_s 168s. fable goes down a pydantic-settings rabbit hole trying to override `settings_customise_sources` with a custom subclass that returns `(init_settings, _YamlSrc(settings_cls, yaml_file=config_path))` — a 30-line workaround for a problem every other arm solves with `Config()` and a Path.exists() check. The code works, but it's an order of magnitude more complex than necessary. fable's *correctness* here is high; its *elegance* is low. (Note: this was also the case where fable later realized the spec called for "one line per model" and patched the rich table out — the second self-correction in the run.)

**Most aggressively tested run: stock r2 (`/data/bakeoff/results/bakeoff-stock--yoke-feature--r2--1785041093`).** Stock added 11 new tests in a fresh `tests/test_model_ls.py`, organized as `TestModelLsNoYaml`, `TestModelLsDefault`, `TestModelLsJson`, `TestModelLsHelp`, with parametrization for null VRAM. 250 tests pass total. This is the most thorough coverage any arm produced for the new command — and the most verbose thinking that produced it (think_chars=10096).

---

## 4. Verdict

On manner, not outcomes. All 15 runs pass.

This task shape is a small-to-medium feature add: a single new typer sub-command with a clear spec, a hidden test the grader will check, and an existing test suite that must remain green. The "right behavior" is: (a) read yoke.yaml with a clear missing-file error, (b) implement text and JSON outputs, (c) follow existing typer patterns, (d) keep the test suite green, (e) ideally add some positive tests.

**thinkingcap's behavior best fits this task shape.** Its median think_chars is the lowest (≈5500), its per-run `pytest -q` count is the lowest (3.8 avg), its manual smoke-test pattern catches stdout-purity bugs that the in-process test runner might miss, and across the 5 reps it consistently avoids the orphan-code, duplicate-registration, and over-engineered-Config-subclass traps. r1 did briefly default to a rich table, but it self-corrected against the spec ("one line per model") in writing — a behavior stock and fable don't show as cleanly. The trade-off is that thinkingcap sometimes under-tests (r5 added zero new tests; r2-r4 added 3-6 each); for a task that has a hidden grader test, that's a real cost — the grader's test replaces the model's positive coverage. But for the *behavior* itself (how the agent reasoned, what tools it chose, how it verified), thinkingcap is the cleanest fit.

**stock's behavior is the most thorough but the most expensive.** Its median think_chars is the highest (≈9000), its verification re-runs the most, its r1 wall_s is 10x the median. stock over-reasons about pydantic-settings internals, then over-engineers a workaround, then debugs the workaround. But stock consistently produces the most tests per run (29 total vs fable's ~13 and thinkingcap's ~14) and the most thorough FEATURE.md files. For a task where test coverage matters more than throughput, stock is the right arm.

**fable's behavior is the most uneven.** r1 was ugly (orphan code, duplicate registration, irrelevant skill load). r2 was clean. r5 was over-engineered (pydantic-settings subclass with `settings_customise_sources` override). fable passes every time but spends more total effort on the same task. The pattern is: fable talks a lot, makes a hash, recovers robustly. For a task where "robust recovery from messy starts" matters more than "clean execution from the first turn," fable might be the right arm — but that's a niche use case. The aggregate stat "fable is +24% model calls" matches what I see in the transcripts: fable re-does things the other arms do once.

**Difference that doesn't matter.** All three arms implement the same final feature correctly. The hidden grader test doesn't care whether the implementation is one line or thirty lines, whether `import json` is at module top or inside the function, whether tests are added or not, or whether the FEATURE.md is 26 lines or 64 lines. The graders' pass/fail outcome is independent of all these behavioral choices. The behavioral differences matter when (a) you're choosing an arm for a *different* task shape, (b) you're optimizing for token cost (fable is most expensive, thinkingcap is cheapest), or (c) you care about code quality / coverage as a secondary metric (stock is most thorough, fable is most verbose, thinkingcap is most concise).

**If I had to pick one arm for this exact task shape on a future run:** thinkingcap. Its single r5 weakness (no new tests) is mitigated by the hidden grader test; its other reps all add 3-6 tests. Its main strengths — compressed planning, end-to-end smoke testing, clean patches first try — map directly onto what this task rewards.

ANALYSIS COMPLETE