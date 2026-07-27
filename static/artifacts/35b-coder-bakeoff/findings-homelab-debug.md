# Findings: homelab-debug (manner analysis, 20 runs)

Task: static-capture diagnosis of a 502 Bad Gateway — nginx `proxy_pass` points at
:5001 while gunicorn binds :5000. Grader is a keyword checklist on
`~/work/rootcause.md`. All 20 runs passed; median wall ~30s. This is the battery's
purest manner comparison: the correct procedure is read-six-files → connect two port
numbers → write one markdown. What differs is how each arm walked that path.

The scenario contains one deliberate trap: `search_files`-style content greps do not
list the `sites-enabled/dashboard` symlink, so an arm that "verifies" the symlink
with the wrong tool can talk itself into a false second root cause. The arms'
handling of that trap is the most diagnostic behavioral difference on this task.

## 1. Per-arm behavioral profiles

### kat (5/5 pass, 28–37s, 4–5 calls)

Approach shape: jump-in, but a disciplined jump. All 5 reps open with a single
parallel batch reading all six scenario files (kat-r1: "Let me read all the relevant
files to diagnose the issue." → six `read_file` calls in one block). r4 and r5 add
a `find` and an `ls -la …/sites-enabled/` *in the same first batch* — proactively
verifying the symlink without being asked, and getting it right (kat-r5 rootcause.md:
"symlink is valid | The active site config is indeed the one with the wrong port").

Thinking-trace character: short, evidence-first, structured as a numbered chain.
kat-r1: "1. The nginx config at `etc/nginx/sites-available/dashboard` line 6 proxies
to `http://127.0.0.1:5001` / 2. The systemd service file at `app/dashboard.service`
line 8 starts gunicorn on port 5000…" — four facts, one inference, done. No
dead reckonings, no filler. Two glitches worth noting: kat-r3's closing reasoning
degenerates into one-word-per-line fragments ("Written / to / /home /coder /work
/root /cause .md ."), and kat-r5's final reasoning box contains a partial tool-call
leak ("`</parameter>\n</function>\n</tool_call>`The user is asking me to confirm the
diagnosis."). Neither affected the artifact.

Tool decisions: minimal and correct. The only oddity is kat-r3 prepending a
gratuitous `execute_code` (`import os` directory walk) alongside the six reads —
pure redundancy, the reads already had every path. kat-r5's fix section is the best
artifact in the set: a proper unified diff with context lines
(`-proxy_pass http://127.0.0.1:5001; / +proxy_pass http://127.0.0.1:5000;` inside a
`@@ -3,7 +3,7 @@` hunk) plus "A `reload` (not a full restart) is sufficient".

Verification behavior: verifies in reasoning, not with tools. kat-r4 post-write:
"Let me verify the symlink is correct and double-check my analysis before presenting.
The facts: …" — a mental re-check of the four evidence lines against what it already
saw. No rep ever re-read rootcause.md after writing; the write-diff was trusted.

Waste modes: essentially none. Zero re-reads, zero loops, zero failed tool calls
across 5 reps (the r3 `execute_code` is the only wasted call in the arm).

### ornith (5/5 pass, 30–44s, 3–8 calls)

Approach shape: near-identical to kat — parallel batch read of all six files, one
analysis turn, one write. r2/r3/r5 prepend a `search_files find *` to enumerate the
tree first. ornith-r1 states the batching strategy explicitly, the only arm to do
so: "Let me read all of these in parallel since they're independent reads."

Thinking-trace character: the most genuinely *diagnostic* reasoning of the four
arms. When its `find *` didn't surface the sites-enabled symlink, ornith-r5 reasoned
through the alternative hypothesis instead of just asserting it: "Wait - actually
nginx's `include /etc/nginx/sites-enabled/*;` with an empty directory would be fine —
nginx tolerates unmatched globs… If sites-enabled doesn't have a symlink… nginx would
just serve on port 80 with defaults… which wouldn't produce a 502 — it would likely
just return 'Welcome to nginx!'." That is correct elimination reasoning (a missing
symlink cannot produce a 502), and it resolved the doubt with the right tool
(`ls -la …/sites-enabled/`). ornith-r4 also shows audience awareness no other arm
matched: "since this is CLI context (no markdown rendering expected). I should keep
it terminal-friendly." Zero tool-call format leaks across all 5 reps — the only arm
clean on that axis.

Tool decisions: sophisticated fixes. r1 and r3 present Option A (change nginx,
"recommended, least risk") and Option B (change gunicorn) with restart commands for
each, and justify the choice operationally (ornith-r1: "changing nginx (Option A)
avoids an application restart"). One blemish: ornith-r4's fix section includes a
verify command referencing a file that doesn't exist in the capture
("diff <(cat /etc/nginx/sites-available/dashboard.bak) …") — an unverified
assumption baked into the runbook (inference: harmless here, sloppy habit).

Verification behavior: r5 is the only ornith rep that re-verified after writing —
but it did so clumsily: it ran the *same* `ls -la sites-enabled/` a second time
after the first write ("Let me check the sites-enabled symlink - it might be a
broken symlink."), then rewrote rootcause.md a second time to fold in the symlink
line. Correct conclusion, wasteful path (8 calls, 137K input tokens — the arm's
outlier).

Waste modes: r5's double symlink-check + double-write; otherwise tight.

### stock35 (5/5 pass, 26–32s, 3–5 calls)

Approach shape: the leanest arm. Same six-file parallel batch, but with the least
thinking of any arm (r3: 295 thinking chars total). Where kat/ornith put their
analysis in the reasoning box, stock35 habitually skips reasoning and puts the
analysis *in the user-facing message*. stock35-r1's reasoning box is one line, then
the Hermes box carries the diagnosis: "Now I have all the pieces… the author findings: 1.
nginx is configured to proxy to port 5001 (line 6 in sites-available/dashboard)…".
stock35-r2's reasoning box is literally an empty `<think>` tag.

Thinking-trace character: thin but adequate when present. r4 does keep a proper
reasoning chain ("1. **nginx config**… proxies to `http://127.0.0.1:5001` (line 6)…
The mismatch is clear"). The arm's characteristic failure is format integrity, not
logic: stock35-r5's reasoning contains a *complete* fake tool call — the entire
rootcause.md drafted inside a literal `<tool_call><function=mcp__write_file>…`
block that never executed — followed by "I've gathered all the evidence - now I need
to analyze what's wrong and write the root cause document.", after which it wrote
the file for real. The artifact was fine; the thinking channel was corrupted.

Tool decisions: nothing fancy, nothing wrong. r2 read the dashboard config twice
(once via the sites-enabled symlink path, judging by the doubled "read dashboard"
lines) — the only redundant read in the arm. Never checked the symlink explicitly;
never used the terminal at all in 5 reps.

Verification behavior: none beyond trusting the write-diff. No re-reads, no
post-hoc checks in any rep. On this task that costs nothing; on a subtler bug it
would be a risk (inference).

Waste modes: none in tools. The waste is in output hygiene — analysis spilling into
the user channel (r1, r2, r3) and the r5 reasoning leak.

### stock36 (5/5 pass, 33–60s, 3–11 calls, the variance arm)

Approach shape: same opening as everyone (parallel batch read), but the only arm
whose runs routinely fail to *stop*. r3 and r5 are normal single-pass runs. r1, r2,
r4 each show a distinct derailment.

stock36-r1 is the single ugliest run of the 20. It completed the task — read all
files, checked the symlink with `ls -la`, wrote a correct rootcause.md — and then
started the entire task over from scratch, twice: "The user is asking me to diagnose
a 502 Bad Gateway issue from a captured filesystem. Let me read all the relevant
files to understand the setup and find the root cause." appears after the first
write, and again ("I need to read the scenario directory files to diagnose the 502…")
after the second. Three full read rounds, three versions of rootcause.md, 38 tool
calls, 210K input tokens (4× the next-most-expensive run), 60s — for a task its
siblings finished in 21s. To its credit, inside the loop it handled the symlink
trap correctly: "Wait, the `search_files` returned 0 results for the sites-enabled
directory… The missing symlink in sites-enabled could be the culprit" → runs
`ls -la` → "Okay, so the symlink is present and correctly points to
`../sites-available/dashboard`." It was also the only run in the whole bakeoff to
re-read rootcause.md after writing ("Let me verify the file was written correctly").

stock36-r2 fell into the trap r1 avoided. It "verified" the symlink with three
content greps ("grep sites-enabled.*dashboard", "grep sites-enabled.*", "grep .*"),
which of course return nothing for a symlink, and concluded: "Good — so there's no
symlink in sites-enabled". That false finding went into the deliverable:
"**No symlink exists in sites-enabled.** … so nginx would not even load this upstream
block", plus a fix recommendation to "Create: `ln -s /etc/nginx/sites-available/…`".
(r1's `ls -la` proves the symlink exists; inference: the grader's keyword list
doesn't penalize a wrong extra claim, so this passed anyway.) The run then ends with
a non sequitur: after writing the file, it reasons "The user's prompt doesn't have
a clear task… Let me just acknowledge and ask what they'd like me to do" and its
final user-facing message is "What would you like me to do with it?" — task done,
model apparently unaware.

stock36-r4 shows thinking degeneration rather than looping: one reasoning box
contains the analysis drafted three times with a stray brace — "Let me write the
root cause document.\n}\n\nNow let me write the root cause document:\n\nNow I have
all the information needed. Let me write the root cause document.Now I have all the
files read." The artifact was still correct. stock36-r5 leaks a tool call as literal
text inside reasoning ("`<tool_call><function=mcp__search_files>…sites-enabled…`"
— never executed) and then never verified the symlink it had intended to check.
stock36-r3's fix adds "sudo systemctl restart dashboard" after the reload —
unnecessary given its own evidence that gunicorn was healthy (minor over-fix).

Waste modes: full-task redo loops (r1), failed-grep flailing (r2), repeated drafts
in thinking (r4), format leaks (r5). Every waste mode in the battery's catalog,
one per rep.

## 2. Head-to-head differences

1. **The symlink trap separates tool choice from tool trust.** Confronted with
   "grep shows no symlink", stock36-r2 believed the tool and wrote a false root-cause
   element ("No symlink exists in sites-enabled"). Ornith-r5 disbelieved the symptom
   on physical grounds ("nginx would just serve on port 80 with defaults… which
   wouldn't produce a 502") and checked with `ls -la`. kat never fell for it at
   all — r4/r5 ran `ls -la` up front ("symlink is valid"). Same ambiguity, three
   grades of epistemics: assertion, verification, preemption.

2. **Stopping behavior.** kat, ornith, and stock35 wrote the file once and stopped
   in 19/20 of their combined runs. stock36 failed to stop in 2/5: r1's triple redo
   ("Let me read all the relevant files…" ×3, 38 tool calls) and r2's confused
   sign-off ("What would you like me to do with it?"). The best stock36 runs (r3,
   r5) match kat's shape; the worst are an order of magnitude more expensive
   (r1: 210K vs kat-r1: 50K input tokens).

3. **Reasoning-channel integrity.** Three arms leaked raw tool-call syntax into
   thinking exactly once each — stock35-r5 (a full phantom `write_file` call with
   the whole report inside), stock36-r5 (a phantom `search_files` call), kat-r5
   (a trailing `</parameter></function></tool_call>` fragment). Ornith leaked zero
   times in five reps. Frequency is too low to rank arms harshly on this, but only
   ornith was clean.

4. **Where the analysis lives.** stock35 repeatedly routes its evidence chain into
   the user-facing message with an empty or one-line reasoning box (r2's reasoning
   is a bare `<think>`; r1's findings appear in the Hermes box). kat and ornith keep
   the chain in reasoning and give the user a TL;DR (kat-r2: "TL;DR: nginx's
   proxy_pass in etc/nginx/sites-available/dashboard:6 points to port 5001…").
   stock36 does both, sometimes repeatedly (r4's triple draft).

5. **Fix craftsmanship.** kat-r5 (unified diff with hunk context, "reload (not a
   full restart) is sufficient") and ornith-r1/r3 (Option A/B with operational
   justification: "avoids an application restart") write fixes a human on-call could
   paste. stock35's fixes are correct but generic; stock36-r2's fix includes an
   action premised on a false finding (creating the symlink), and stock36-r3's
   includes an unneeded `systemctl restart dashboard`.

6. **Verification of the deliverable.** Only stock36-r1 re-read rootcause.md after
   writing. Ornith-r5 re-verified one fact (symlink) post-write and rewrote. kat
   re-checked facts in reasoning only (r4: "double-check my analysis before
   presenting"). stock35 verified nothing, ever. On a task this simple the difference
   is invisible in outcomes but diagnostic of habit.

## 3. Notable runs

- **stock36-r1** — the cautionary tale: correct answer reached three separate times,
  38 tool calls, 210K input tokens, and the model re-framing its own completed work
  as a fresh user request ("The user wants me to diagnose a 502 Bad Gateway issue…"
  after the file was already written). Also the only run that both fell toward the
  symlink trap *and* rescued itself with `ls -la`.
- **stock36-r2** — the only run whose deliverable contains a factually wrong claim
  ("No symlink exists in sites-enabled"), produced by trusting a content-grep over
  a directory listing, followed by a sign-off asking the user what the task was.
- **ornith-r5** — the best single piece of diagnostic reasoning in the set (the
  "a missing symlink would yield the default page, not a 502" elimination), marred
  by checking the same directory twice and writing the report twice.
- **kat-r5** — the cleanest end-to-end run: proactive `ls -la` in the first batch,
  unified-diff fix, reload-only prescription; its only flaw is the tool-call
  fragment leaked into the final reasoning box.

## 4. Verdict

kat's behavior fits this task shape best, and here its low-token style is
unambiguously **discipline, not corner-cutting**. The evidence: in all 5 reps it
read every one of the six files (nothing skipped, nothing guessed), cited line
numbers from what it read (kat-r1: "line 6 proxies to `http://127.0.0.1:5001`"),
and in r4/r5 it did *more* than the minimum by proactively verifying the
sites-enabled symlink with the correct tool. Corner-cutting would look like
stock35's empty `<think>` boxes or stock36-r2's unverified symlink claim; kat shows
neither. Its waste across five reps is one redundant `execute_code` call.

Ornith is a close second and arguably the best *thinker*: the only arm with zero
format leaks and the only one that reasoned about what evidence would look like
under alternative hypotheses (r5). It loses points for r5's double-check/double-write
and r4's invented `.bak` file in the fix runbook.

stock35 is fast and harmless but shallow: analysis in the wrong channel, no
verification habit, and the worst single format leak (r5's phantom write). On a
harder incident its thinness is a liability (inference — nothing on this task forced
it).

stock36 is the weakest manner despite 5/5 passes. Its good reps (r3, r5) are fine;
its bad reps exhibit the full taxonomy of agentic waste — redo loops, tool-distrust
inversions, degenerate repetition, and losing track of whether the task is done.
The 802K-thinking-char aggregate from the brief shows up here as r1's 210K-token,
three-times-around-the-block performance: the only run in the set that investigated
both cleanly *and* flailed, in alternation, within a single transcript.

ANALYSIS COMPLETE
