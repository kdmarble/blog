# Findings — mixed-pipeline (config repair)

Task shape: agent must (a) read three input files (broken JSONC-ish config, env
file, SPEC), (b) author a Python stdlib-only fixer, (c) run it, (d) verify the
output matches SPEC (strict JSON, key order, port = int from env, no placeholder
leakage, features booleans, upstreams list of {name,url} dicts). Small,
deterministic, single-deliverable — read → write → run → verify. No ambient
search/exploration.

All 15 runs passed with score 1.0.

---

## 0. What every arm did identically (the task's floor)

Before contrasting the arms, three behaviors are universal across all 15 reps
and so are properties of the task, not of any post-train:

1. **Batched triple-read opener.** All 15 runs open with the same move — three
   `read_file` calls prepared in a single turn (dashboard.json.broken, app.env,
   SPEC.md), then one reasoning block. The openers are nearly interchangeable:
   fable-r1 "Let me start by reading all three files to understand what needs to
   be done" (`bakeoff-fable--mixed-pipeline--r1--1785056598`); stock-r1 "Let me
   start by reading the contents of the workspace to understand what I'm working
   with" (`bakeoff-stock--mixed-pipeline--r1--1785044212`); thinkingcap-r3 "Let
   me start by reading all three files to understand what I'm working with"
   (`bakeoff-thinkingcap--mixed-pipeline--r3--1785050375`).
2. **Text-preprocess-then-`json.loads` architecture.** No arm attempted a
   hand-rolled tokenizer or a real state-machine JSONC parser. Every rep is a
   variant of: strip `//` comments → swap single→double quotes → strip trailing
   commas → resolve `${...}` → `json.loads` as validator → `json.dump`. And
   because the architecture is identical, the failure modes are identical.
3. **The `//`-in-URL trap and the newline-then-`}` comma trap.** Of 15 reps,
   ~11 hit the comment-stripper eating `http://` (the `//` looks like a comment
   to a naive regex), and a large fraction separately hit trailing commas where
   the `,` sits on one line and the closing `}`/`]` on the next. These are the
   task's two hidden teeth, and they bite every arm roughly equally.

So the interesting variation is in *recovery manner*, not avoidance.

---

## 1. Per-arm behavioral profiles

### stock (Qwen/Qwen3.6-27B)

**Approach shape — checklist planner.** stock consistently reasons about the
SPEC as a numbered requirement list *before* writing code. stock-r1 lays out the
five defects line-by-line, then a numbered plan ("1. Strip comments... 2.
Replace `${DASHBOARD_PORT}`... 3. Convert single quotes..."), then traces the
comment-stripper char-by-char in its head before ever running
(`bakeoff-stock--mixed-pipeline--r1--1785044212`). stock-r2 does the same
explicit enumeration, including pre-empting the URL hazard: "I need to be
careful about URLs containing `//`. Like `'http://127.0.0.1:9001'` — the `//` in
the URL is inside quotes" (`bakeoff-stock--mixed-pipeline--r2--1785044341`).

**Thinking-trace character.** Moderate length (1.0k–11k reason chars), heavy on
"let me think more carefully" re-reasoning after a failure. The most striking
trait is **pre-run mental simulation that still fails**: in stock-r1 the model
walks through its scanner, concludes the single-quote tracking will protect the
URL — "when we enter the single-quoted string `'http://127.0.0.1:9001'`, we
toggle `in_sq = True`. Then scanning `http://127...`, at `//` we check `not
(in_sq or in_dq)` — but `in_sq` is True, so we don't break! ... Looks correct.
Let me run it." (`bakeoff-stock--mixed-pipeline--r1--1785044212`) — then exit 1.
The trace is locally coherent and globally wrong. stock-r3's 11k-char trace is
the deepest, descending into a long audit of why the trailing-comma regex fails
across newlines: "the regex only looks at current line but `\n` is between `,`
and `}`. After stripping trailing spaces from each line, the `,` stays because
it's on the same line as the value" (`bakeoff-stock--mixed-pipeline--r3--1785044483`).

**Tool decisions.** stock leans on `patch` for fixes (4 patches in r3) rather
than whole-file rewrites. It is the only arm to try the **`(?<!\w)` negative-
lookbehind** fix for the URL bug — and then correctly reason that it fails
("since `:` is not a word char... it would still match!") before settling on
`\s+//` (`bakeoff-stock--mixed-pipeline--r3--1785044483`). Command sophistication
is mid: it prefers several targeted `python3 -c` debug one-liners over a single
awk/sed pipeline.

**Verification.** Universal and rigorous: every rep reads `fixed.json` back AND
runs a programmatic `python3 -c` check against every SPEC clause. stock is the
most prone to the **false-positive verification loop**: stock-r2 writes
`assert '//' not in text`, watches it fail on `http://`, then re-explains to
itself that the `//` is inside a URL string value, not a comment
(`bakeoff-stock--mixed-pipeline--r2--1785044341`). stock-r5 hits the same
`'//' not in text` false positive and recovers with a string-stripping check
(`bakeoff-stock--mixed-pipeline--r5--1785044765`).

**Waste modes.** Re-running nearly-identical verification scripts after a
self-inflicted false positive; stock-r3's 18-tool-call run is bloated by
repeated "let me print the intermediate text" debug cycles that re-derive the
same conclusion.

### thinkingcap (ThinkingCap-Qwen3.6-27B)

**Approach shape — terse jump-in, then rewrite.** thinkingcap's opening
reasoning blocks are the shortest of the three arms on its good runs — r3, r4,
r5 all go "Now I have all the context. Let me write the script and run it" with
minimal enumeration (`bakeoff-thinkingcap--mixed-pipeline--r5--1785050592`).
This fits the post-train's "token-efficient thinking" billing *on its lean
runs* — but the claim inverts on its heavy runs (see below).

**Thinking-trace character — bimodal.** This is the arm's defining feature on
this task. r3/r4/r5 are lean (1.8k/3.4k/3.1k reason chars); r1 is the single
most expensive run of the entire task at 9,286 reason chars and 16 LLM calls.
The r1 blow-up is a genuine reasoning spiral: thinkingcap wrote a comment
stripper that split the line on `"` to protect quoted regions, then rejoined
with `''.join(parts)` — which **drops the `"` delimiters entirely**. It took a
long debug chain to surface this, culminating in the correct diagnosis: "YES!
That's the bug. `str.split()` drops the delimiter. When I do `line.split('\"')`
and then `''.join(parts)`, the `\"` characters that were delimiters don't come
back. So `\"service\"` becomes just `service`"
(`bakeoff-thinkingcap--mixed-pipeline--r1--1785050039`). That is a real, hard-
won insight — but it cost 35 messages to reach.

**Tool decisions — rewrite-over-patch.** thinkingcap is the arm most willing to
`write_file` the whole script again rather than `patch` a region: r1 = 4 writes
+ 1 patch, r2 = 2 writes + 1 patch, r3 = 2 writes + 1 patch. stock and fable
patch more readily. Inference (marked as inference, not fact): shorter thinking
traces correlate with preferring a fresh full-file write, which is cheaper to
reason about than a diff context.

**Recovery strategy for the URL bug.** thinkingcap's most common fix is
**reordering** — moving the single→double quote conversion *before* comment
stripping, so the scanner only has to track one quote type: "I need to either 1.
Convert single quotes before stripping comments, or 2. Also treat single quotes
as quote characters during comment stripping. Option 1 is simpler"
(`bakeoff-thinkingcap--mixed-pipeline--r4--1785050482`). This is arguably the
cleanest recovery strategy any arm uses.

**Verification.** Same universal pattern (read-back + `python -c`). thinkingcap-
r5 additionally reaches for `python3 -m json.tool` as a second validator
(`bakeoff-thinkingcap--mixed-pipeline--r5--1785050592`) — the only rep across
all arms to do so.

**Waste modes.** The r1 spiral is the clearest waste event: a 9k-char reasoning
chain to find a one-line join bug. r2 separately burns a turn on a broken patch
that extracted a closure out of scope ("That patch made the code worse. Let me
rewrite the file cleanly", `bakeoff-thinkingcap--mixed-pipeline--r2--1785050239`).

### fable (Fable-Fusion-711)

**Approach shape — confident jump, react to errors.** fable's opener is the most
action-forward: fable-r1 reads the three files and immediately writes a 98-line
script with *no* intermediate plan block beyond "Let me create the Python script
and run it" (`bakeoff-fable--mixed-pipeline--r1--1785056598`). When it fails it
fixes fast — fable-r1's only bug is a regex that misses nested-object
single-quoted keys (`{'name':'auth'}` not preceded by whitespace), patched in
one shot by broadening the character class
(`bakeoff-fable--mixed-pipeline--r1--1785056598`).

**Thinking-trace character — high variance.** fable spans the widest range:
r1 is the leanest trace of any arm (916 reason chars, near-first-try success),
while r5 is the second-most-expensive run of the whole task (8,859 reason chars,
16 tool calls). fable's traces read more "narrative" and less "checklist" than
stock's — fewer numbered plans, more "let me just debug first" reactions.

**Tool decisions — the only arm to use `execute_code`.** fable-r3 is the sole
rep across all 15 to reach for the `🐍 exec` (execute_code) tool to inspect
intermediate text, rather than a `python3 -c` one-liner
(`bakeoff-fable--mixed-pipeline--r3--1785056851`). This is a genuine tool-
selection difference: where stock and thinkingcap debug via terminal one-liners
(and repeatedly fight shell-quoting of `${` and `"`), fable-r3 sidesteps that
entirely with a sandboxed Python exec.

**The shell-escaping rabbit hole (r5).** fable-r5 is fable's signature waste
event and is **unique to this arm**. Its inline `python3 -c` debug scripts
failed to match `${DASHBOARD_PORT}` because of shell quoting, but the model
mis-attributed the failure to the *file contents* and spent multiple turns
inspecting the bytes of the placeholder: "There's a weird issue. The raw line
shows `${DASHBOARD_PORT}` but the regex doesn't match it. Let me check if these
are special Unicode characters that look like `$` and `{`"
(`bakeoff-fable--mixed-pipeline--r5--1785057127`) — complete with a per-
character `ord()` dump — before finally realizing "It works in heredoc mode but
not when embedded directly. The issue must be shell escaping in my inline `-c`
strings." No other arm fell into this particular trap.

**Verification.** Same universal read-back + programmatic check. fable-r1 is the
most economical verifier — a single compact `python3 -c` covering keys, port
type, and placeholder scan (`bakeoff-fable--mixed-pipeline--r1--1785056598`).

**Waste modes.** The r5 Unicode/escaping spiral (above). fable-r2 also repeats
the now-familiar "didn't write where expected" detour, reasoning in circles
about `__file__.parent` resolution before re-running
(`bakeoff-fable--mixed-pipeline--r2--1785056718`).

---

## 2. Head-to-head

**(a) Planner vs jumper on the opener.** stock reasons about the SPEC as a
checklist before coding ("Let me think more carefully... a cleaner approach:
parse it line by line...", `bakeoff-stock--mixed-pipeline--r1--1785044212`);
fable jumps straight to a full script ("Let me create the Python script and run
it", `bakeoff-fable--mixed-pipeline--r1--1785056598`); thinkingcap is between
but trends toward fable's jump style on lean runs ("Now I have all the context.
Let me write the script and run it", `bakeoff-thinkingcap--mixed-pipeline--r5--1785050592`).
On this deterministic task the jumper loses nothing — there is no exploration to
short-circuit.

**(b) Recovery from the `//`-in-URL bug — three strategies.** stock tries a
regex lookbehind then `\\s+//` (`bakeoff-stock--mixed-pipeline--r3--1785044483`);
thinkingcap reorders pipeline stages (quotes before comments,
`bakeoff-thinkingcap--mixed-pipeline--r4--1785050482`); fable extends the
scanner to track both quote char types (`bakeoff-fable--mixed-pipeline--r2--1785056718`).
All three converge on correctness, but thinkingcap's reorder is the smallest
diff and the most robust — it eliminates the hazard class rather than patching
one symptom.

**(c) Patch vs full rewrite under failure.** stock patches (4 patches in r3);
fable patches or uses `execute_code` to debug (`bakeoff-fable--mixed-pipeline--r3--1785056851`);
thinkingcap rewrites the whole file (4 writes in r1,
`bakeoff-thinkingcap--mixed-pipeline--r1--1785050039`). thinkingcap's rewrite
habit is the most token-expensive recovery mode.

**(d) Key-order paranoia.** Most reps trust Python 3.7+ dict insertion order
implicitly ("Python dicts preserve order now, so it's fine"). Only a minority
reach for `object_pairs_hook=OrderedDict` as an explicit guarantee: stock-r2,
fable-r1, fable-r5, thinkingcap-r1. stock-r2 is the most explicit — "Parse into
ordered dict... `object_pairs_hook=OrderedDict`" (`bakeoff-stock--mixed-pipeline--r2--1785044341`).
This is a trust-the-runtime bet vs. an assert-the-contract bet; the trusting
majority happened to be correct here.

**(e) The false-positive verification loop — a shared blind spot.** Roughly 6 of
15 reps write a brittle `'//' not in text` (or `in json.dumps(data)`) check,
watch it fail on `http://`, then have to re-explain that the `//` is valid
string content. stock-r2 ("the `//` in URLs is fine because it's part of a URL
string value, not a comment", `bakeoff-stock--mixed-pipeline--r2--1785044341`),
thinkingcap-r2, thinkingcap-r4 all do this. The grader already proves "no
comments" via a successful strict `json.loads`; the agents re-litigate it with a
worse check. No arm is innocent here.

**(f) Token-efficiency claim vs. reality.** ThinkingCap is advertised as
"same quality, fewer thinking tokens." On this task the claim holds only on its
*best* runs: thinkingcap-r5 (3,068 reason chars, 38s) is the cheapest successful
run of all 15, and r3 (1,830) is the leanest by reason chars. But its *worst*
run, r1 (9,286 reason chars, 101s, 16 LLM calls), is the most expensive run of
the entire task — more wasteful than any stock or fable rep. The post-train
trades average-case efficiency for a fatter tail.

---

## 3. Notable runs

- **Cleanest overall: thinkingcap-r5** (`bakeoff-thinkingcap--mixed-pipeline--r5--1785050592`).
  38s, 7 tool calls, 6 LLM calls, first-try script success, the only rep to add
  `python3 -m json.tool` as a second validator. The shape the task rewards.
- **Clean stock: stock-r5** (`bakeoff-stock--mixed-pipeline--r5--1785044765`).
  First-try script (exit1=0), 49s, 8 tool calls. Stock's most efficient rep.
- **Clean-ish fable: fable-r1** (`bakeoff-fable--mixed-pipeline--r1--1785056598`).
  Near-first-try (one targeted patch for nested-object keys), 44s, the leanest
  reasoning trace of any arm (916 chars).
- **Ugliest overall: thinkingcap-r1** (`bakeoff-thinkingcap--mixed-pipeline--r1--1785050039`).
  101s, 18 tool calls, 16 LLM calls, 35 messages — a long spiral to find a
  `str.split`/`join` delimiter-drop bug. Eventually correct, but the cost
  dwarfs every sibling rep.
- **Wasteful detour: fable-r5** (`bakeoff-fable--mixed-pipeline--r5--1785057127`).
  91s, 16 tool calls; a multi-turn rabbit hole hexdumping the `${...}`
  placeholder looking for "special Unicode characters" that were actually shell-
  quoting artifacts in the model's own debug scripts.

---

## 4. Verdict

For a read → write → run → verify pipeline this deterministic, the behavior
that best fits is the **lean one-shot with fast recovery** — the
thinkingcap-r5 / stock-r5 shape: read the three files, write the script, run
once, verify once, done. The task does not reward upfront planning (there is no
design space to explore) and does not reward deep thinking (the bugs are
mechanical, not conceptual). What it rewards is *not hitting the `//`-in-URL
trap on the first try*, and no arm manages that reliably — so the real
differentiator is recovery speed, where thinkingcap's reorder-the-pipeline fix
and fable's confident-patch style both beat stock's deeper re-derivation.

Between the arms, **stock is the most consistent** (narrow variance, textbook
checklist manner, rarely surprising), **fable is the most variable and the only
arm to produce a genuine non-shared rabbit hole** (the r5 Unicode/escaping
detour), and **thinkingcap is bimodal**: its best run is the cheapest of all 15
and its worst is the most expensive, so the "token-efficient thinking" claim is
true on average but comes with a heavier tail than either baseline.

If the difference doesn't matter, say so plainly: on *this* task, all three
arms converge to the same architecture, hit the same two bugs, and recover to
the same 1.0. The manner differences (stock's checklist, fable's jump, 
thinkingcap's rewrite-and-reorder) are real and quotable, but none of them
changes the outcome on a single-shot deterministic config fix. These manner
signals would matter far more on a task with genuine branching — search,
multi-file refactor, or open-ended debugging — where planning depth and recovery
style actually move the result.

ANALYSIS COMPLETE
