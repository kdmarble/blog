# Bakeoff Findings: web-research task

Three arms (Qwen3.6-27B-class post-trains), five reps each. All 15 passed the grader.
This document captures behavioral patterns of how each arm worked — not whether they
succeeded.

Task shape: 3 simple web-research questions answered via curl + SearXNG, output
`research.md` with one section per question and a URL per claim. The grader checks
presence of a `b\d{4,}` tag, llama-swap maintainer, a USD price, etc. — relatively
forgiving. Behavioral differences between arms show up in *how* they get the same
three answers.

---

## 1. Per-arm behavioral profile

### 1a. thinkingcap (bottlecapai/ThinkingCap-Qwen3.6-27B)

**Approach shape — confident parallel-first.** thinkingcap's first move in all 5
runs is to fire three SearXNG queries simultaneously. The opening reasoning block is
always short (3-4 sentences):

> "I need to answer three research questions using a SearXNG instance. Let me start
> by searching for all three questions in parallel since they're independent."
> — thinkingcap-r1 line 24

Then it immediately verifies the parallel strategy was correct:

> "Good, I have initial results. Now I need to: 1. Get the actual latest release tag
> and its changelog from the GitHub releases page 2. Confirm llama-swap details
> 3. Get the actual current price"
> — thinkingcap-r1 line 35

It has internalized the curl/follow-up pattern and treats the question set as a
plan to dispatch in two waves: parallel SearXNG → parallel direct curls.

**Thinking-trace character.** thinkingcap's reasoning is short and operational. The
reason_blocks are 6-12 lines (vs 14-31 for fable, 7-20 for stock); think_chars per
run: 3504 / 4369 / 4177 / 6031 / 18882 (the r2 outlier is 18k; it got stuck
investigating whether b10121 existed). It enumerates what to do next, runs the
curls, and re-states the data it has. There is little visible metacognition or
self-correction. Filler level is low: no purple narration, no role-play, no
restating the user's prompt.

**Tool decisions.** thinkingcap reaches for the right canonical sources quickly:
`https://api.github.com/repos/ggml-org/llama.cpp/releases/latest`,
`https://raw.githubusercontent.com/mostlygeek/llama-swap/main/README.md`,
`https://www.phanteks.store/...`. It pipelines these — `curl ... | jq '.results[:5]'`
— to keep output small. It uses `python3 -c` pipes for HTML price extraction
(r5 lines 229, 141) and JSON-LD parsing (r5 line 229), which is a step up from
naïve `grep price`. It also tries `-L` follow-redirects and `-H User-Agent` headers
when it hits Amazon/Best Buy blocks (r5 lines 211-212, 220-221).

**Verification behavior.** thinkingcap re-fetches when results look thin. The
canonical pattern is: SearXNG → GitHub API → if body is sparse, compare API. It does
this in r1 (lines 60, 90), r3 (lines 84, 125, 150, 165), r4 (lines 84, 125, 152,
165), r5 (lines 93, 105, 115, 140, 144). Two of those (`b10106...b10107` and
`b10105...b10107` compare) are duplicate commands with slightly different jq
projection, suggesting it loops when the JSON is not what it wanted.

**Waste modes.**
- **Repeated curl-of-the-same-API with different jq filters.** r5 lines 93 and 105
  both fetch `releases/latest` and extract `.body` or `.object.sha`; r3 fires
  `compare/b10106...b10107` twice (lines 84, 125), and `compare/b10105...b10107`
  twice (lines 125, 165). It pays the latency cost of the API call rather than
  re-projecting locally.
- **Too-wide grep patterns that swallow HTML noise.** r3 line 54 greps the Newegg
  page for `[0-9]*\.[0-9]*`; r5 line 64 greps for `\$[0-9]+\.[0-9]{2}` which
  matches "$10.00" from a shipping line.
- **r2 stuck on a hallucinated b10121.** A SearXNG hit mentioned b10121; r2 spent
  ~10 tool calls trying to verify whether b10121 existed before the GitHub API
  and the GitHub HTML page both confirmed b10107 (lines 76-103). This is the
  thinkingcap run with 18k think chars and 12 LLM calls — half its budget burned
  on a phantom release tag.

### 1b. stock (Qwen/Qwen3.6-27B)

**Approach shape — parallel-first, then methodical canonical sources.** All 5 stock
runs open with the same move: three simultaneous SearXNG queries, one per question
(stock-r1 lines 36-38, stock-r4 lines 35-37). The opening reasoning is one or two
sentences ("I need to answer three research questions using a SearXNG instance. Let
me start by searching for all three questions in parallel" — stock-r2 line 24), so
on first move stock is indistinguishable from thinkingcap. Wave two is also stable
across reps: `api.github.com/.../releases/latest`, the llama-swap README or repo
API, and a first scrape attempt at Newegg or phanteks.store. Where stock diverges
from thinkingcap is wave three onward: it settles into a patient, single-question-at-
a-time grind, dominated by one specific problem — b10107 is a one-commit release,
and the task demands two notable changes.

**Thinking-trace character — anxious deliberation with heavy self-correction loops.**
Stock's distinguishing texture is long reasoning blocks full of "Actually, wait"
reversals as it negotiates with itself about what counts as a change "in" b10107.
The extreme is stock-r1 (lines 143-191), a single block with five "Actually"
restarts; stock-r2 has "Actually, I need to focus." (line 366) mid-spiral, and
stock-r4 gives up on resolution entirely: "You know what, I'll just go with what
I have." (line 200) followed later by "Actually, I think I'm overcomplicating
this." (line 223-225). Stock-r3 shows the same loop but lands on an honesty norm:
"Actually, let me be more honest. The b10107 release has one main code change."
(lines 451-452). This is metacognition, but of a hand-wringing kind — it re-derives
the same dilemma (one commit vs. "two notable changes") four or five times without
new information, and the deliberation does not change the final answer, which in
all 5 reps is some combination of the hexagon fix plus an adjacent-release commit.

**Tool decisions — solid jq, iterative grep refinement, some redundant refetching.**
Stock reaches the GitHub API immediately and uses clean jq projections
(`{tag_name, published_at, body}`). On price scraping it iterates grep patterns
against the same page rather than saving it once: stock-r2 curls the identical
Newegg URL five times with five different filters (lines 114, 449, 521, 579-581,
630), and fires `compare/b10098...b10107` twice (lines 520, 541). When it does
level up, the refinement is real: stock-r3 ends at
`grep -oP '\$<strong>[0-9]+</strong><sup>\.[0-9]+</sup>'` (line 323), which is
Newegg's actual price HTML structure. It shows good source skepticism in places —
stock-r3 discards "a $860 sponsored ad price that doesn't seem relevant" (lines
502-504), and stock-r4 dispatches the phantom b10121 tag (from a releasealert.dev
snippet) with a single API call: "b10121 doesn't exist as a released tag - the
search result from releasealert.dev was misleading." (lines 121-122). Contrast
thinkingcap-r2, which burned ~10 calls on the same phantom.

**Verification behavior — consistent read-back after write.** In 4 of 5 reps stock
verifies the artifact after writing: "Let me confirm it was created successfully
by reading it back." (stock-r1 lines 371-372; also r2 lines 716-717, r3 lines
606-608). Stock-r5 is the best recovery episode in the arm: `write_file` failed
on a tmp-file error, it reasoned "The write_file tool failed because it tried to
resolve the path. Let me try writing directly using the terminal approach instead"
(lines 299-301), fell back to a `cat << 'ENDOFFILE'` heredoc, then `ls` + read-back
to confirm. Only stock-r4 (the leanest run, 15 tool calls) wrote and walked away
without a check.

**Waste modes.**
- **Deliberation without information gain.** The "two notable changes" dilemma is
  re-thought, not re-searched; stock-r2 pairs this with 45 tool calls including
  compare ranges b10106, b10105, b10104, b10100, b10095, b10098, b10103 — several
  returning empty or duplicate data.
- **Refetch-instead-of-reproject.** Repeated identical curls with different grep
  filters (stock-r2's five Newegg fetches) instead of one fetch saved to disk.
- Otherwise low: no filler narration, no invented facts, no role-play.

### 1c. fable (DavidAU Fable-Fusion-711)

**Approach shape — parallel-first, then opportunistic and wide-ranging.** Fable
opens identically to the other arms (three parallel SearXNG queries in all 5 reps;
fable-r1 lines 35-37, fable-r5 lines 42-44) but with slightly longer framing blocks
that enumerate the questions back. After the first wave it is the most willing of
the three arms to roam: fable-r1's price hunt touches Newegg, Best Buy, B&H, Amazon,
Reddit (`.json` endpoint, line 382), atxcases.us, and phanteks.com; fable-r5
abandons the terminal mid-run for `execute_code` Python blocks (~15 of them) to
parse Newegg's `window.__initialState__` JSON. Its strategy is less "plan and
execute" and more "try things until something yields."

**Thinking-trace character — short and operational, but with confabulated specifics
and artifact bleed.** Most fable reasoning blocks are brisk status updates, but two
failure textures recur. First, invented specifics delivered with confidence: fable-r4
writes in its artifact that llama-swap is "Maintained by `mostlygeek` — Matthew
Garrett, former Red Hat engineer" (diff line 690) — a confabulation (mostlygeek is
Benson Wong, which fable-r1 itself confirmed via `api.github.com/users/mostlygeek`,
line 111). Fable-r4 also curls a guessed URL that was never observed anywhere
(`raw.githubusercontent.com/ggml-org/llama.cpp/master/examples/batch-release-notes.md`,
line 214). Fable-r5 initially trusts a stale snippet — "llama.cpp latest release tag
is b8554" (line 48) — before correcting via the API one call later. Second, literal
training-artifact bleed: fable-r5's reasoning twice ends with a stray `</parameter>`
token (lines 746, 781). Neither other arm shows either behavior on this task.

**Tool decisions — the most sophisticated extraction of the three arms, plus tool
distrust.** Fable's ceiling is high: fable-r3 pivots from a failed Python one-liner
(exit 1, line 263) to grepping Newegg's embedded search JSON directly
(`grep -oP '"UnitCost":\d+\.\d+'`, line 283) and correctly maps SKUs to variants;
fable-r5's execute_code parse yields `OriginalUnitPrice=179.99, FinalPrice=191.98,
LowestPrice30Days=178.19` (cited in its artifact, line 916) — the richest price data
any run produced. Fable-r1 chose the widest changelog window of all 15 runs
(`compare/b10063...b10107`, 44 commits, line 313) and surfaced genuinely notable
changes (Vulkan vk_queue refactor, CUDA GET_ROWS quants) that stock and thinkingcap
never saw. Its floor is low too: fable-r5 hits one jq exit-5 and declares "jq seems
broken for me somehow. Let me use raw json output and grep instead" (lines 353-354)
— blaming the tool for its own malformed filter — and fable-r1 fires a string of
degenerate SearXNG queries (`"Enthoo+Pro+2+Server"+$189+$199`, `bought+paid+$`;
lines 467, 475).

**Verification behavior — strong self-correction on data, weak on artifact checks.**
Fable catches its own misreads impressively: fable-r5 talks itself out of the Klarna
trap — "the Klarna results showed recurring prices of $252.99, which appears to be
an installment math example ... That's not an actual retail price for the case!"
(lines 775-777) — and fable-r2 reasons "let me just report honestly: b10107 ...
contains one code change" (lines 149-152) before flagging the one-commit release in
its artifact. But only 2 of 5 fable runs verify research.md after writing (r1 line
658, r3 line 494); r2, r4, r5 declare done on the write result alone ("Done. Written
to ~/work/research.md with three sections, all claims backed by URLs." — fable-r4
lines 713-714), and fable-r4's unverified artifact is exactly the one carrying the
confabulated maintainer name.

**Waste modes.**
- **Degenerate query spirals when a source resists.** fable-r1's price hunt (~30
  tool calls on question 3 alone) and fable-r5's empty-SearXNG loop (lines 529-611,
  six near-identical queries returning nothing).
- **Redundant reasoning restatement.** fable-r5 re-explains the one-commit situation
  to itself at lines 624-632, 837-842, and 845-859 — the stock-style loop, though
  less frequent.
- **Variance.** Wall times span 124s (r4) to 381s (r5, the slowest of all 15 runs);
  this arm's cost is the least predictable of the three.

---

## 2. Head-to-head

**1. All three arms open identically; the task only separates them after wave two.**
Every one of the 15 runs fires three parallel SearXNG queries first, with a near-
identical one-line justification ("since they're independent" — stock-r2 line 24,
thinkingcap-r1 line 24, fable-r3 line 25). On a task with an obvious parallel
decomposition, opening strategy carries zero signal. Everything below is wave-three
behavior.

**2. Resolving "two notable changes" in a one-commit release: stock agonizes, fable
widens or discloses, thinkingcap iterates API calls.** Stock re-thinks the dilemma
repeatedly ("these are two separate releases. The question asks for notable changes
in b10107 specifically." — stock-r3 lines 479-480) and usually lands on adjacent-
release commits (hexagon fix + CUDA q1_0 MMQ, or the args refactor). Fable either
widens the compare window aggressively — fable-r1 went straight to
`compare/b10063...b10107` (44 commits, line 313) and picked the Vulkan refactor and
CUDA GET_ROWS support — or discloses the ambiguity in the artifact itself ("This was
a very small point release containing only one commit" — fable-r2 diff line 329;
fable-r4 and fable-r5 artifacts do the same). Stock's artifacts, by contrast, present
two changes as simply "in this release" (stock-r1 diff lines 329-333) without
flagging that one came from b10106. Thinkingcap neither agonized in prose nor
disclosed — it just re-fetched the API with different jq filters (1a).

**3. Unsourced specifics: fable confabulates, stock stays vague, thinkingcap stays
terse.** The only factual error in any of the 15 artifacts is fable-r4's "Maintained
by `mostlygeek` — Matthew Garrett, former Red Hat engineer" (diff line 690). Notably,
fable-r1 shows the arm could have known better — it fetched `api.github.com/users/
mostlygeek` and reported "Benson Wong" (line 111, diff line 637), the deepest
maintainer sourcing of any run. Stock never attempts a real name in 5/5 reps
("maintained by mostlygeek on GitHub" — stock-r1 line 306), and neither does
thinkingcap. Fable also produced the only guessed-URL fetch (fable-r4 line 214) and
the only stale-snippet trust ("latest release tag is b8554" — fable-r5 line 48,
corrected one API call later). This is the creative-merge character showing: richer
when right, wrong when not.

**4. Price extraction: grep refinement (stock) vs structured parsing (fable) vs
python one-liners (thinkingcap).** Stock iterates regexes against the Newegg HTML
until one bites, ending at the `<strong><sup>` price structure (stock-r3 line 323)
and filtering noise by judgment ("$860 sponsored ad price that doesn't seem
relevant" — stock-r3 lines 502-504). Fable goes for the embedded JSON: Newegg search
state (`"UnitCost":\d+\.\d+` — fable-r3 line 283) and `window.__initialState__` via
execute_code (`OriginalUnitPrice=179.99, FinalPrice=191.98, LowestPrice30Days=
178.19` — fable-r5 line 811-816), the most structured data any run extracted.
Thinkingcap used python3 pipes and JSON-LD parsing (1a). All three arms also share
one failure: naive `\$[0-9]+\.[0-9]{2}` greps that catch $10.00 shipping lines
(stock-r1 lines 107-110, thinkingcap-r5 line 64, fable-r5 line 122-124), and all
three recognize and discard the bad match.

**5. Error recovery style: stock works around, fable re-tools and complains.**
Stock-r5's write_file failure is handled cleanly: diagnose ("tried to resolve the
path", line 299), heredoc fallback, read-back verify. Fable-r5 hits one jq exit-5
and generalizes to tool distrust — "jq seems broken for me somehow. Let me use raw
json output and grep instead" (lines 353-354) — then later abandons curl entirely
for execute_code. Fable also shows the best data-level self-correction of the
battery: "the Klarna results showed recurring prices of $252.99, which appears to
be an installment math example ... That's not an actual retail price for the case!"
(fable-r5 lines 775-777). Stock-r3 independently filtered the equivalent trap
($739.99 third-party / $860 sponsored listings, lines 543-545).

**6. Artifact verification: stock 4/5, fable 2/5.** Stock reads research.md back
after writing in r1, r2, r3, r5 ("Let me verify the file was written correctly by
reading it back." — stock-r2 lines 716-717). Fable verifies only in r1 and r3; r2,
r4, r5 end on the write confirmation ("Done. Written to ~/work/research.md with
three sections, all claims backed by URLs." — fable-r4 lines 713-714). The one
unverified fable artifact (r4) is the one containing the confabulated maintainer —
the read-back habit is not decorative on this task shape.

## 3. Notable runs

- **fable-r5 (381s, 31 LLM calls, 20k out tokens — slowest and wordiest of all 15).**
  The whole fable story in one run: stale b8554 snippet trusted then corrected, "jq
  seems broken" tool distrust (line 353), a mid-run pivot to ~15 execute_code blocks,
  the Klarna $252.99 installment trap fallen into and then escaped by pure reasoning
  (lines 775-777), two `</parameter>` leaks in its thinking (lines 746, 781) — and
  then the best price extraction of the battery (`OriginalUnitPrice / FinalPrice /
  LowestPrice30Days`, line 811-816) and an honest one-commit disclosure in the
  artifact. Ugly process, strong landing.
- **fable-r1 (238s, 42 LLM calls).** Deepest research of the 15: only run to identify
  mostlygeek as Benson Wong via the users API (line 111), widest changelog window
  (b10063...b10107, 44 commits), only run to report availability status ($179.99 but
  sold out). Cost: ~30 tool calls of degenerate price-query spirals.
- **fable-r4 (124s, fastest fable).** Clean and quick — and the only run of 15 to
  ship a factual error ("Matthew Garrett, former Red Hat engineer", diff line 690),
  plus a guessed-URL fetch (line 214). Skipped the read-back that would have given
  it a chance to catch it.
- **stock-r2 (198s, 45 tool calls, 14.3k think chars — heaviest stock).** The
  "Actually" spiral at maximum: seven different compare ranges (several empty), five
  fetches of the same Newegg page, and self-interruptions like "Actually, I need to
  focus." (line 366). Passed comfortably anyway — the waste is pure overhead.
- **stock-r4 (123s, 15 tool calls — leanest stock).** The efficient ideal: phantom
  b10121 dispatched with one API call (lines 117-122), no redundant fetches. The one
  blemish is no post-write verification.
- **stock-r5.** Best error recovery of the arm: write_file tmp-file failure →
  reasoned diagnosis → heredoc write → ls + read-back (lines 289-348).

## 4. Verdict

**stock's behavior fits this task shape best.** The task is shallow fact-finding
with a citation requirement: three questions, canonical sources, one file. What it
rewards is reaching the GitHub API and one retailer page quickly, extracting without
inventing, and checking the artifact before declaring done. Stock does exactly that:
same parallel-first open as everyone, the most disciplined verification habit (4/5
read-backs), zero confabulated facts in 5 artifacts, and the battery's cleanest
error recovery (r5). Its tax is the anxious "Actually, wait" deliberation — stock-r2
burned 14.3k thinking chars re-deriving a dilemma that changed nothing — but on this
task the tax is latency, not correctness.

fable is the most interesting arm and the least trustworthy. Its ceiling is the
highest (structured-JSON price extraction, Benson Wong, 44-commit changelog
synthesis, honest one-commit disclosures), and its good reps (r2, r3) are arguably
better artifacts than stock's. But the creative-merge character produces the only
factual error of the 15 runs (r4's invented maintainer), the only invented URL, the
only stale-snippet trust, and the only markup leaks in thinking — plus the widest
cost variance (124s to 381s). For a research task where every claim needs a URL,
"occasionally asserts things with no source" is the worst failure mode available.

Where the difference does not matter: questions 1 and 2 converge across all arms
(same tag, same hexagon fix, same mostlygeek/llama-swap answer), and all three arms
independently fall for and then discard the same $10.00 grep trap. The opening
strategy is identical everywhere. If you only read the artifacts' Q1/Q2 sections
(minus fable-r4's name), you could not tell the arms apart — the behavioral gap
lives entirely in process cost, verification habits, and fable's unsourced-specific
risk. thinkingcap remains the budget option (1a); stock is the safe default; fable
is the high-variance pick you would only choose with a verifier downstream.

ANALYSIS COMPLETE