# mixed-pipeline — Behavioral Findings

## Task shape and grading in one paragraph

Inputs are in `~/work/bundle/`: `dashboard.json.broken` (C-style `//` comments, single-quoted keys/values, trailing commas after `false`,`}`, `},` patterns, and one `${DASHBOARD_PORT}` placeholder); `app.env` (`DASHBOARD_PORT=5000`); and `SPEC.md` (the required output contract). Deliverables are `~/work/fix_config.py` (Python stdlib only) and `~/work/fixed.json`. The grader enforces a tight set of bool checks: strict-JSON parse, exact key-order equality `["service","bind","port","workers","features","upstreams","log_level"]`, `port == 5000` as int, exact dict-equal `features={"metrics":True,"tracing":False,"debug_endpoints":False}`, upstreams list-of-dicts with exactly `name`/`url`, and no `${` or `DASHBOARD_PORT` substring remaining in the output text. The non-trivial mechanical traps that bite every arm are: (a) the `//` inside `http://127.0.0.1:9001` URLs must NOT be stripped as a comment, and (b) the trailing-comma regex has to handle commas both *inside* the inner upstreams objects and *after* them. Every one of the 20 runs passed, so all my analysis is about *how* they passed.

## Per-arm behavioral profiles

### stock36 — verbose, verbose, verbose; one marathon, four tidy runs

**Approach shape.** stock36 always opens with a single-line reading phase (three `read_file` calls in parallel) and then jumps into writing the script. In four of five reps it commits to a character-by-character state-machine stripper; in r1 it instead hand-designs a JSON5-ish parser with a per-key tokenizer (`python_val`, nested-structure tracker, type inference) and burns ~6,800 lines of transcript before settling on a far simpler design. The shape across reps: one first-pass write, one bash run that often fails, one debug print, one fix-and-run, then a verification block.

**Thinking-trace character.** stock36 thinks out loud to an extreme degree. Its thinking often contains literal "simulate-the-regex-on-paper" walkthroughs of the very repair it is about to run. Representative excerpts:

- stock36 r4 (line 144-200, ~7000 chars): after writing the script, the model writes a long trace that traces `re.sub(r',(\s*[}\]])', r'\1', text)` character-by-character through the input, drawing fake match tables ("Match 1: In line 11..."). The whole block is a self-built correctness proof.
- stock36 r1 (lines 70-87, 2200-2400): the model spends multiple paragraphs enumerating edge cases ("when `//` is preceded by whitespace", "block comments before string conversion", "tracking nesting depth") before committing code. The trace reads like a design document rather than runtime reasoning.
- stock36 r5 (lines 162-200): again, ~3500 chars of "the issue is likely..." hypothesizing about where the parse failed, including an explicit guess that "the comment on line 14 might not be getting stripped correctly" before even running anything.

The trace is mostly self-correcting rather than dead-reckoning: stock36 will hypothesize a bug, propose a fix, run it, and then re-hypothesize on the next iteration. Filler is low — every paragraph advances the script.

**Tool decisions.** stock36's signature move is to spend 50-200 lines of inline `python3 -c "..."` shell debugging (printing intermediate states after each repair pass) before committing to a script fix. It writes verify.py files when the inline shell gets too long. It uses `json.dump` with `indent=2` and trusts `dict` insertion order, and it always validates with `json.loads` before writing. No exotic tool choices — `read_file`, `write_file`, `patch`, `terminal` only.

**Verification behavior.** All five stock36 runs end with an explicit verification pass that re-validates each grader criterion by hand. r1's verification is the most elaborate (a full `verify.py` with 9 numbered checks and re-serialization round-trip). r3 writes a smaller 3-line verify inside a terminal call.

**Waste modes.** r1 is the waste case (572s wall, 106K out tokens, 40 tool calls, 802K thinking chars, 8,556 transcript lines). The model over-engineered a tokenizer when a character-loop suffices. The other four runs are efficient (8-18 tool calls, 60-150s wall, 7-24K out tokens). Vertical-render glitch (`The\nissue\nis\nthat`) appears in r1+r2 only (2 and 1 blocks respectively); r3-r5 are clean. Tool-call XML leakage (raw `<function=...>` tags showing up in assistant content) appears in r1+r2 only.

### stock35 — bash-quote scrambler; spawns `mcp__execute_code` instead of fighting `$`

**Approach shape.** stock35 reads the three files and then writes a script. Where stock36 debugs by printing intermediate text inside `terminal`, stock35 (especially r2 and r3) reaches for `mcp__execute_code` — Python interpreter sessions — to drive the debugging. It also fights bash quoting far more than the other arms: f-strings containing `${DASHBOARD_PORT}` explode inside `python3 -c`, forcing it to either escape, rewrite as `<<'PYEOF'` heredocs, or punt to a saved `verify.py` file.

**Thinking-trace character.** stock35's trace is dense and debug-loop oriented. Quotes from reps:

- stock35 r1 (lines 144-146): "There's an invalid control character. This is likely a tab or other escape sequence in the comment text." — then immediately reads the file with `cat -A` to inspect for UTF-8 em-dash artifacts (an actual real issue with the broken file's comment text).
- stock35 r4 (lines 154-156): "There's a Unicode character in the comment (M-bM-^@M-^T is UTF-8 for an em-dash or similar). Let me fix the script to handle this properly" — followed by an explicit `content = content.replace('\u2014', '').replace('\u2013', '')` defensive strip.
- stock35 r2 (lines 142-145): "Need to debug what's on line 13 of the fixed content to see what control character issue is happening."

Where stock36 narrates a regex trace, stock35 narrates a single error then pivots. The trace tends to be shorter per turn but the *tool sequence* is longer (r3 alone made 47 tool calls).

**Tool decisions.** stock35's distinguishing choice is the `mcp__execute_code` heavy use (r2 has many `🐍 exec` calls; r3 fires ~25 of them). It also writes `debug_fix.py`, `debug2.py` helpers. It uses `terminal` heavily with both `python3 -c` and `cat <<'PYEOF'` heredocs to avoid the `$` interpolation. r4 has a particularly gnarly sequence where it literally types `// Is at start of meaningful content (like '// billing...')` *as Python code* inside a write_file tool call — a syntax error the model then has to clean up.

**Verification behavior.** stock35 verifies carefully but bash-fragile. It tries to validate inline first, falls back to a saved `verify.py` file (often called `debug_fix.py` or `verify.py`) once shell escaping gets in the way. r5's verification is comparatively clean: a single inline `python3 -c` with explicit assertions.

**Waste modes.** r3 is the worst stock35 waste (1.36M input tokens, 47 tool calls, 2245 transcript lines, 47 glitched/vertical-render blocks). r1 also burns badly on the em-dash tangent. Vertical-render glitch is universal across all 5 stock35 reps (range 1-6 per run). Tool-call XML leakage is in all 5 reps (counts 2-7, total 18 — the worst of any arm). The `+ 1 command` shell-prefix rendering artifact appears in r1, r2, r3 (total 8 — also worst of any arm).

### ornith — base-Qwen3.5 quirks dominate; XML leaks; verbose bash struggles

**Approach shape.** ornith's first moves are nearly identical to stock36/stock35: read three files in parallel, then write a script. The difference shows up *during* the script execution. ornith consistently trips on the same two mechanical problems the other arms do (URL `//` inside single-quoted strings, trailing-comma regex) but it tends to debug via inline-shell `python3 -c "..."` rather than `mcp__execute_code`, and it leans hard on patches rather than full file rewrites.

**Thinking-trace character.** ornith's thinking tends to be one-word-per-line *glitched* renders in the more complex turns, AND it leaks raw `<function=mcp__terminal>` XML inside its reasoning text. Quotes:

- ornith r1 (lines 184-200): "The / issue / is / that / my / comment / stripping / logic / accidentally / ate / the / opening / `'` character..." — vertical-render breakdown of a single sentence.
- ornith r3 (lines 260-330): the model writes the full tool-call XML inline in its reasoning: `<tool_call><function=mcp__patch><parameter=mode>replace<parameter><parameter=path>~/work/fix_config.py...<parameter=old_string>def strip_comments...<parameter=new_string>def strip_comments...` — i.e., its planning contains the actual tool call it intends to make. This is a stock35/ornith-base-Qwen3.5 quirk that NEVER appears in stock36 r3-r5 and NEVER appears in any kat rep.
- ornith r5 (lines 2025-2034, 300-450): the model's reasoning trails into ~170 lines of "thinking out loud" about what helper functions to call (`is_inside_string`, `_first_quote_char`, `_string_stack` — none of which it actually defines), then catches itself and decides to rewrite the whole script cleanly.

ornith thinking tends to be *exploratory* in a way stock36 thinking isn't — it proposes 3-4 different approaches per debug cycle before picking one. Self-correction is common.

**Tool decisions.** ornith uses `terminal` with inline Python far more than `mcp__execute_code`. It writes `verify.py` files when shell escapes get bad. It uses `patch` more than the other arms (smaller deltas, less rewriting). r2 has a famous 300-second timeout on a `verify.py` run (line 903, `[exit 124]`) because the model spawned `python3 verify.py` without a timeout — it had to wait the full 5 minutes.

**Verification behavior.** Mixed. r5 has a thorough inline verification (checks key order, port type, features bool, upstreams structure, no placeholders, no comments, no trailing commas). r2's verify.py timed out and the model had to rewrite it. r1 doesn't run a separate verify at all — it declares the script "sanity-checked" because it does `json.loads` inside `repair()`.

**Waste modes.** r2 is the worst ornith waste (392s wall, 19 tool calls, 12.6K out, 21K thinking, 6m23s Duration). r5 is also heavy (140s, 21 tool calls, 21.8K out, 53K thinking). The vertical-render glitch hits 4/5 reps (1-4 blocks per run). XML-leakage is in all 5 ornith reps (counts 1-3, total 10). The `+ 1 command` artifact appears in r1 only.

### kat — disciplined, terse, deterministic; the cheap path is correct here

**Approach shape.** kat's first move is invariably the same: read the three input files in parallel, then write a script. No elaborate planning. In r1 the script is 156 lines; r3 is 95 lines; r5 is 89 lines; r2 is 93 lines. kat scripts are noticeably shorter than the other arms' scripts and they use simpler primitives (regex + `json.dumps`, not custom tokenizers). The post-write flow is: bash run → if it fails, one focused debug print → one targeted patch → bash run again → verify. Almost no full-rewrites.

**Thinking-trace character.** kat's trace is short, surgical, and highly self-correcting *only* when needed. Quotes:

- kat r1 (lines 174-200, the bug-hunt): "The issue is that my comment stripper is also removing parts inside strings incorrectly, or the quotes are getting doubled. Let me trace through what's happening." Then a long (~350 line) trace that pinpoints the exact bug (a stale `in_string` state) and rewrites the stripper. This is kat r1's worst case and it's still smaller than stock36 r1's planning block.
- kat r3 (lines 215-218): "I accidentally wrote to ~/fix_config.py first then overwrote it with the corrected version at ~/work/fix_config.py. Let me run the script now." — a self-aware diagnosis of a path mistake, no defensive bluster.
- kat r3 (lines 230-247): when the URL `//` issue hits, kat's thinking is just three short paragraphs identifying the ordering bug ("If I replace single quotes with double quotes FIRST, then the `//` inside URLs would be inside a string and wouldn't be treated as comments. Let me try that order.") and fixing it on the next turn. No multi-iteration walkthrough.
- kat r5 (lines 159-200): when debugging, kat's trace explicitly enumerates the failure modes ("Wait, the original file has `http://127.0.0.1:9001`..."), arrives at a single ordered hypothesis ("convert single quotes to double quotes first, then strip comments"), and applies it.

kat's filler is minimal. Its trace does NOT contain the vertical-render breakdown pattern (3/5 reps have 0 glitched blocks; r1+r2 each have 1). kat's trace NEVER leaks raw tool-call XML.

**Tool decisions.** kat is the only arm with zero leaked `<function=...>` XML across all 5 reps (verified by grepping all 20 transcripts). kat uses inline-shell `python3 -c` sparingly. r3 writes a separate `verify.py` only because the inline shell choked on `$`. kat's path-resolution is sometimes wrong (r1, r2, r3 all hit the `bundle/` subdirectory issue at least once) but each one is fixed in a single patch. kat also occasionally writes `verify.py` for the same shell-escape reason other arms do (r3, r4).

**Verification behavior.** All five kat runs verify explicitly against SPEC.md. The pattern is consistent: list the grader's six requirements as checkmarks in the reasoning, then output a JSON dump of `fixed.json` in the final Hermes message. r3 adds a separate `verify.py` file because the inline shell couldn't safely quote the dollar signs.

**Waste modes.** kat is the cheapest arm here in every dimension: 9-15 tool calls (vs stock36's 8-44, ornith's 7-23, stock35's 9-47), 4.9K-8.0K out tokens (vs stock36's 7K-106K), 2.4K-14.7K thinking chars (vs stock36's 11K-307K). No vertical-render breakdowns, no XML leaks. The only waste mode is the recurring path-resolution one (script looks for `dashboard.json.broken` next to itself instead of in `bundle/`), which costs one patch per run.

## Head-to-head differences on this task

1. **Discipline vs over-engineering on a JSON5-ish repair.** stock36 r1 builds a full JSON5 parser with `python_val`, per-key tokenizer, type inference, and a nesting tracker — and still has to fall back to a simpler approach. Every arm hits the URL `//` confusion at least once (verified by grepping for "URL.*comment" / "://.*comment" patterns in all 20 transcripts: ornith 0-24, stock35 4-15, stock36 15-69, kat 1-21). The discipline difference is in *recovery*: kat's fix is always one targeted patch (e.g., kat r3 trace at line 245: "If I replace single quotes with double quotes FIRST, then the `//` inside URLs would be inside a string and wouldn't be treated as comments. Let me try that order."), while the other arms often pay 2-3 debug cycles with intermediate prints and `debug_fix.py` files before getting the order right.

2. **Tool-call XML leakage as a base-model fingerprint.** Raw `<function=mcp__...>` tags inside the assistant's *reasoning* (not its tool calls) appears in 5/5 ornith runs, 5/5 stock35 runs, 2/5 stock36 runs (r1, r2 only), and 0/5 kat runs. Verified by counting `<function=` substrings per file. This is a stock35-era Qwen3.5 characteristic that the Hermes harness sometimes mishandles when it surfaces the model's planning back as raw text.

3. **Bash-quoting strategy on the `${DASHBOARD_PORT}` placeholder.** The grader checks for `${` substring absence. stock35 r1, r2, r3, r4 all try to verify via inline `python3 -c "...${PLACEHOLDER}..."` and get bitten by bash variable interpolation, forcing them into heredocs or saved files. kat r3 explicitly notes the issue and goes straight to a saved `verify.py` (r3 trace: "The shell is having trouble with the `$` character. Let me write a verification script to a file instead."). stock36 r1, r5 follow the same kat pattern.

4. **Verification depth.** Every arm verifies against SPEC.md but the depth differs. stock36 r1 writes a 9-point `verify.py` with re-serialization round-trip. stock36 r4, r5 and ornith r5 write smaller inline checks. stock35 verifies bash-fragile. kat writes a checkmark list inside its reasoning plus a `json.dumps` of the output in the final message. All sufficient for the grader; only stock36 r1's verify is materially more thorough than required.

5. **Approach convergence vs divergence.** All four arms converge on the same five-step pipeline (strip comments → convert quotes → resolve placeholders → strip trailing commas → validate). They diverge in: (a) whether they handle single-vs-double quote state separately (stock36 yes, ornith yes, kat yes, stock35 yes — but with different bug rates); (b) whether they write `verify.py` or rely on inline (kat mostly inline, stock35 mostly saved); (c) whether they ever hit URL `//` confusion (all except kat, on first script run).

6. **Waste ratios on a task that has a tight optimal.** The grader passes all 20 runs. The optimal run is ~9 tool calls and ~4.9K out tokens. kat r3 hits 9/9/4.9K. stock36 r1 burns 40/40/106K for the same pass — a 22x out-token multiplier for zero outcome benefit. stock35 r3 hits 47 tool calls. ornith r2 hits 19 tool calls and 392s wall-clock. The waste is real but it doesn't move the outcome needle.

## Notable runs

- **stock36 r1** is the most over-engineered run in the dataset (40 tool calls, 572s wall, 106K out tokens, 8,556 transcript lines, 307K thinking chars). It hand-designs a JSON5 parser, gets stuck on URL handling, has to context-compact (line 1388: "🗜️ Compacting context — summarizing earlier conversation so I can continue..."), and only converges on a simpler design at line ~4400. Verdict: over-thought, passed anyway.

- **stock36 r2** is the only stock36 run that triggered context-compacting before settling. It demonstrates the same over-engineering pattern as r1 but at smaller scale (18 tool calls, 147s wall).

- **ornith r2** is the longest single ornith run at 6m 23s (392s wall). Its `verify.py` invocation hit a 300-second timeout (`[exit 124]`) because the model forgot to add a timeout, then it had to rewrite the verify step. The terminal-prefix artifact `+ 1 command` also appears in this run.

- **ornith r5** is the worst display-glitch concentration (3 vertical-render breakdowns, leaked tool XML, and a 140s wall with 21 tool calls). It is also the run with the `dasboard.json.broken` typo (line 30) — the model typed `dasboard` instead of `dashboard` on the first read attempt.

- **stock35 r3** is the heaviest stock35 run (47 tool calls, 1.36M input tokens, 2245 transcript lines). It uses `mcp__execute_code` heavily and writes multiple debug scripts. The vertical-render glitch hits 6 blocks in this run alone.

- **kat r1** is the most thoughtful kat run (14.7K thinking chars — high for kat but still tiny vs the others) because it actually hit a bug and had to debug through a 350-line trace of the quote-doubling problem. Recovered cleanly with a single targeted patch.

- **kat r3** is the textbook kat run (9 tool calls, 4.9K out, 13 messages, 56s wall, 2.4K thinking chars). Reads three files, writes one 95-line script, hits the URL `//` issue, recognizes it in one paragraph of thinking, rewrites, runs, verifies. This is the canonical "discipline, not corner-cutting" exemplar.

## Verdict — manner, not outcomes

This is a mechanical Python task with a tight, well-defined optimal. The grader is unforgiving — every criterion is bool — but the solution shape is well-known (strip comments, convert quotes, resolve placeholders, strip trailing commas). The most interesting comparison is between **kat** and the rest on the dimension of "do the extra tokens buy anything observable in the output?"

**On the manner question:** kat's discipline is real, not corner-cutting. Evidence: (1) every kat script passes on the same criteria every other arm's script passes; (2) kat's verification depth matches stock36 r5 and ornith r5 (spec-checkmark list + dump of output); (3) kat never leaks tool-call XML (0 leaks across all 5 reps vs stock35's 18, ornith's 10, stock36's 6), never breaks into vertical-render mode (only 2 glitched blocks total across 5 reps vs stock35's 14, ornith's 10, stock36's 3), never uses `mcp__execute_code`, rarely rewrites the script from scratch, never writes a `debug_fix.py`; (4) kat hits the URL `//` ordering bug but recovers in *one* targeted patch in every rep (e.g., kat r3 line 245: "If I replace single quotes to double quotes FIRST, then the `//` inside URLs would be inside a string and wouldn't be treated as comments. Let me try that order."), while the other arms pay 2-3 debug cycles with intermediate prints and `debug_fix.py` files before getting the order right; (5) when kat hits a real bug (r1's quote-doubling), it self-corrects in one targeted patch, not a full rewrite. The only "cheaper" thing kat does is spend ~5x less thinking per rep — and on this task the extra thinking in the other arms' traces is mostly exploring hypotheses that turn out to be wrong, not producing net-new information.

**stock36's waste is the polar opposite:** r1's 106K out tokens spent designing a JSON5 parser that the model itself abandons by line 4400. r4's 350-line "trace-the-regex-by-hand" block is a self-built correctness proof for code that passes its grader check trivially. r5's narrative hypothesizing ("the issue is likely...") is filler.

**stock35's distinguishing characteristic is bash-quote fragility and heavy use of `mcp__execute_code`** — the latter is a sign the model is reaching for a debugging harness rather than fighting the shell. That choice is fine but it pushes r3 to 47 tool calls.

**ornith's manner is the most variable** — some reps (r3, r5) are heavily glitched with broken-render blocks and XML leaks, others (r4) are cleaner. This arm also has the worst raw outcome on its difficult reps (r2: 6m23s with a verify.py timeout, r5: typo'd filename).

**Manner ranking on this task:** kat (disciplined, surgical, no waste) > stock36 (competent but verbose; r1-r2 egregious) > stock35 (bash-fragile but functional; r3 marathon) > ornith (highly variable; rendering glitches and XML leaks in 5/5 reps).

The difference between kat and stock36 here isn't that stock36 fails — it passes every grader criterion in 4 of 5 reps too. The difference is that kat never *almost* fails (no debug-spiral, no broken intermediate state, no need for context-compacting) while stock36 frequently dances at the edge and recovers by brute force. On a closed-form task with a known optimal, the arm that finds the optimal fastest and stays there is exhibiting discipline; the arms that find it slower and noisier are not exhibiting corner-cutting (the outcomes are identical) but they are exhibiting *more* behavior for the same result, and that behavior has a higher tail-risk of failing on harder variants of this task shape.

ANALYSIS COMPLETE