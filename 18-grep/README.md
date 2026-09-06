# Demo 18 — grep

## Overview

Demo 17 built the regex vocabulary; this demo is about wielding `grep`
itself as a fast, practical search tool — not just matching patterns, but
searching efficiently across directory trees, getting the surrounding
context that turns a match into a diagnosis, and building the kind of
log-searching muscle memory that makes incident investigation faster. By the
end, `grep` should feel like a precision instrument, not just "the thing you
pipe to."

## Demo Objectives

1. Choose the right flavor flag (`-E`/`-P`) deliberately, building on Demo 17
2. Use context flags (`-A`/`-B`/`-C`) to see what surrounds a match
3. Search recursively across a directory tree efficiently
4. Combine `-v`, `-c`, `-l`, and `-o` for different investigative purposes
5. Build practical multi-step log-searching workflows
6. Recognize when `grep` is the wrong tool and something else (Demo 19/20/21) fits better

## Directory Structure

```
18-grep/
└── README.md   # This file — full demo content, including Anki (Appendix A)
                 # and Quiz (Appendix B), extracted later via script.
```

## Environments

| Task | Ubuntu 24.04 | Rocky Linux 9.8 | EC2 Amazon Linux |
|---|---|---|---|
| All lab tasks | ✅ | ✅ | Not required |

## Recall Check — 17-regular-expressions

Answer these from memory before continuing — no looking back at Demo 17.

**Q1.** `grep 'error|warning' file.log` returns 0 matches; `grep -E
'error|warning' file.log` returns 14. Why?

**Q2.** What does the pattern `(\w+) \1` actually match?

**Q3.** What are the default regex flavors for `grep`, `sed`, and `awk`
respectively?

<details>
<summary>Answers</summary>

**A1.** In BRE (grep's default without `-E`), `|` is a literal character,
not alternation — the pattern searches for the literal string
`error|warning`. `-E` enables ERE, where `|` means alternation.

**A2.** A word immediately followed by that exact same word again — `\1`
refers to whatever text the first group actually captured on this match,
not just "any two words."

**A3.** `grep`: BRE (`-E` for ERE, `-P` for PCRE). `sed`: BRE (`-E`/`-r` for
ERE). `awk`: ERE by default, no BRE mode.

</details>

## Part 1 — Concepts

### 1.1 Choosing the flavor flag deliberately

Demo 17 covered *why* flavor matters; here's the practical habit: decide the
flavor before writing the pattern, not after it silently doesn't match.

```bash
grep 'ERROR\|WARN' app.log         # BRE, escaped alternation
grep -E 'ERROR|WARN' app.log       # ERE, cleaner and more common in practice
grep -P '(?<=user=)\w+' app.log    # PCRE — lookbehind, a feature BRE/ERE
                                    # don't have at all, not just different
                                    # syntax for something they do have
```

`-P` isn't just "ERE with different escaping" — PCRE adds real capability
BRE/ERE lack entirely: lookaheads/lookbehinds, non-greedy quantifiers (Demo
17 §1.4's greedy trap), and named groups. Reach for `-P` specifically when
the pattern needs one of those, not as a general "stronger" default —
plain `-E` remains more portable (PCRE support is GNU-grep-specific, not
guaranteed on every system).

### 1.2 Context flags — seeing what surrounds a match

A bare match often lacks the context to actually diagnose anything:

```bash
grep -A 3 'OutOfMemoryError' app.log
# -A 3: show 3 lines After each match — what happened right after the error
grep -B 3 'OutOfMemoryError' app.log
# -B 3: show 3 lines Before — what led up to it
grep -C 3 'OutOfMemoryError' app.log
# -C 3: both — 3 lines of Context on each side
```

```bash
grep -A 5 -B 2 'connection refused' app.log
# Asymmetric context — different amounts before/after, when the useful
# information isn't symmetric around the match itself (e.g. more relevant
# to see what happened after a connection failure than what led up to it).
```

When `grep` finds multiple matches with overlapping context windows, it
separates each group with `--` by default — a visual cue that two match
blocks aren't contiguous in the source file, which matters when reading
grouped output quickly.

### 1.3 Recursive search across a directory tree

```bash
grep -r 'novatech_backend' /etc/nginx/
# -r: recursive — search every file under this directory tree

grep -rn 'novatech_backend' /etc/nginx/
# -n: also show line numbers — essential once you're searching multiple
# files, since "which line" becomes as important as "which file"

grep -rl 'novatech_backend' /etc/nginx/
# -l: list matching FILENAMES only, not the matching lines — useful when
# you need to know *which* files to look at next, not read every match now
```

```bash
grep -rn --include='*.conf' 'listen' /etc/nginx/
# --include: restrict recursive search to files matching a glob — avoids
# wasting time (and cluttering output) searching binary files, logs, or
# irrelevant file types under the same tree.

grep -rn --exclude-dir='.git' 'TODO' /home/wadmin/project/
# --exclude-dir: skip entire directories — .git is the canonical example,
# since it contains binary object data that's both slow to search and
# never what you actually want matched.
```

### 1.4 -v, -c, -l, -o — four flags, four different investigative purposes

```bash
grep -v 'DEBUG' app.log
# -v: invert — show lines that do NOT match. Useful for filtering OUT noise
# (verbose debug logging) to see everything else.

grep -c 'ERROR' app.log
# -c: count matching LINES (not occurrences — a line with two matches still
# counts as 1), fast way to get a magnitude without reading every match.

grep -l 'ERROR' /var/log/*.log
# -l: which files contain at least one match, without printing any matches
# themselves — the "which of these logs even mention this" question.

grep -o 'user=[a-zA-Z0-9]*' app.log
# -o: print ONLY the matched text (not the whole line) — turns grep into
# an extraction tool, not just a filter, useful when piping matched
# fragments into further processing.
```

**These four compose well together**, and combining them is often more
useful than any single one alone:

```bash
grep -v 'DEBUG' app.log | grep -c 'ERROR'
# Count ERROR lines, excluding anything also tagged DEBUG — a two-step
# filter pipeline, each grep doing one clear job.
```

### 1.5 Practical log-searching workflows

A realistic investigation rarely stops at one `grep` call — it's usually a
short sequence, each command narrowing based on what the last one revealed:

```bash
# Step 1: how big is the problem?
grep -c 'connection refused' app.log

# Step 2: when did it start?
grep -n 'connection refused' app.log | head -1

# Step 3: what does the surrounding context look like at that point?
grep -A 5 -B 5 'connection refused' app.log | head -20

# Step 4: is this isolated to one backend, or widespread?
grep -o 'backend=[a-z0-9-]*' app.log | grep 'connection refused' -B0 -A0 2>/dev/null
grep 'connection refused' app.log | grep -o 'backend=[a-z0-9-]*' | sort | uniq -c
# sort | uniq -c here is a preview of Demo 19 (Text Processing Utilities) —
# grep found the relevant lines and extracted a field; counting/grouping
# unique values is a different tool's job, not grep's.
```

This progression — count, locate, contextualize, break down by dimension —
is a repeatable shape for any log investigation, not specific to this one
example.

### 1.6 When grep is the wrong tool

`grep` searches and filters; it doesn't transform. The moment an
investigation needs to *change* text (not just find it), or needs
field-aware processing (column N of a structured line, not just "does this
line match"), `grep` isn't the right tool anymore:

- **Need to replace text, not just find it?** That's `sed` (Demo 20).
- **Need to process specific fields/columns, do arithmetic, or aggregate
  across many lines?** That's `awk` (Demo 21).

Recognizing this boundary early avoids the common anti-pattern of building
an increasingly elaborate `grep`+`sed`+manual-parsing pipeline for something
`awk` would express in one clean pass.

## Part 2 — Hands-On Lab

### Commands used in this demo

| Command | Flag meaning | Purpose |
|---|---|---|
| `grep -P 'PATTERN'` | `-P`: PCRE | Lookaheads/behinds, non-greedy — real capability, not just syntax |
| `grep -A N` / `-B N` / `-C N` | After/Before/Context lines | Show surrounding lines |
| `grep -r` / `-rn` / `-rl` | recursive, +line numbers, +filenames-only | Search a directory tree |
| `grep --include=GLOB` / `--exclude-dir=DIR` | — | Scope recursive search |
| `grep -v` | invert | Show non-matching lines |
| `grep -c` | count | Count matching lines |
| `grep -o` | only-matching | Extract just the matched text |

### Task 1 — Context flags on a synthetic log

```bash
cat > /tmp/app.log << 'LOGEOF'
2026-07-25 10:00:01 INFO  Starting request handler
2026-07-25 10:00:02 DEBUG Connection pool size: 20
2026-07-25 10:00:03 ERROR OutOfMemoryError in worker-3
2026-07-25 10:00:04 INFO  Restarting worker-3
2026-07-25 10:00:05 DEBUG Connection pool size: 19
2026-07-25 10:00:06 INFO  Worker-3 restarted successfully
LOGEOF

grep -A 2 -B 1 'ERROR' /tmp/app.log
```

### Task 2 — Recursive search with scoping

```bash
mkdir -p /tmp/lab-configs/{app,nginx,.git}
echo "listen 80;" > /tmp/lab-configs/nginx/site.conf
echo "listen 8080;" > /tmp/lab-configs/app/app.yaml
echo "binary garbage" > /tmp/lab-configs/.git/objects-fake

grep -rn 'listen' /tmp/lab-configs/
grep -rn --include='*.conf' 'listen' /tmp/lab-configs/
grep -rn --exclude-dir='.git' 'listen' /tmp/lab-configs/
```

### Task 3 — -v, -c, -l, -o composed together

```bash
grep -v 'DEBUG' /tmp/app.log
grep -c 'INFO' /tmp/app.log
grep -o 'worker-[0-9]*' /tmp/app.log
grep -v 'DEBUG' /tmp/app.log | grep -c 'INFO'
```

### Task 4 — A four-step log investigation

```bash
cat >> /tmp/app.log << 'LOGEOF'
2026-07-25 10:01:00 ERROR connection refused backend=app-01
2026-07-25 10:01:05 ERROR connection refused backend=app-01
2026-07-25 10:01:10 ERROR connection refused backend=app-02
LOGEOF

grep -c 'connection refused' /tmp/app.log
grep -n 'connection refused' /tmp/app.log | head -1
grep -A 2 -B 2 'connection refused' /tmp/app.log | head -10
grep 'connection refused' /tmp/app.log | grep -o 'backend=[a-z0-9-]*' | sort | uniq -c
```

### Task 5 — PCRE lookahead vs. what BRE/ERE can't do

```bash
echo "user=wadmin session=abc123" | grep -oP '(?<=user=)\w+'
# Extracts just "wadmin" — a lookbehind, matching \w+ only where it's
# preceded by "user=" without including "user=" itself in the match.
echo "user=wadmin session=abc123" | grep -oE 'user=\w+'
# Compare: ERE's -o still includes "user=" in the output, since ERE has no
# lookbehind mechanism to exclude it.
```

### Task 6 — Recognize the tool boundary

```bash
grep 'ERROR' /tmp/app.log
# grep finds it. Now: replace ERROR with CRITICAL in the file — grep alone
# cannot do this (no replace capability); this is what Demo 20 (sed) is for.
# Extract and sum a numeric field across matching lines — grep alone can
# extract the field (-o) but not do arithmetic on it; that's Demo 21 (awk).
```

## What You Learned

1. ✅ Chose the right flavor flag (`-E`/`-P`) deliberately, building on Demo 17
2. ✅ Used context flags (`-A`/`-B`/`-C`) to see what surrounds a match
3. ✅ Searched recursively across a directory tree, scoped with `--include`/`--exclude-dir`
4. ✅ Combined `-v`, `-c`, `-l`, `-o` for different investigative purposes
5. ✅ Built a practical multi-step log-searching workflow
6. ✅ Recognized when `grep` is the wrong tool for the next step

## Part 3 — Break-Fix Scenario

### Scenario — NovaTech Ops: two grep-workflow incidents

> **Incident report — NovaTech Ops, ticket #4634**
>
> Alert 1: "recursive grep across `/var/log/app/` for a deploy ID takes
> forever and returns garbage output mixed with matches." Alert 2: "a
> teammate's `grep -c` count of ERROR lines doesn't match what they counted
> manually — off by several."

**Incident 1 — slow, garbled recursive grep**

```bash
$ grep -r 'deploy-8842' /var/log/app/
Binary file /var/log/app/archive.log.gz matches
2026-07-25 09:12:03 INFO deploy-8842 started
...
```

Tasks:
1. What does "Binary file ... matches" indicate about what's in this
   directory, and why does that slow the search down and clutter the
   output?
2. Reference §1.3 — what flag(s) would you add to scope this search
   correctly, and what would you actually scope it to (or away from)?
3. Would `--include` or `--exclude` (by extension, not directory) be the
   more natural fix here specifically, given the file shown?

**Incident 2 — grep -c count doesn't match manual count**

```bash
$ grep -c 'ERROR' app.log
42
$ # teammate manually counted 47 occurrences of the word "ERROR" in the file
```

Tasks:
1. Reference §1.4 — what does `grep -c` actually count, and how could that
   produce a lower number than counting every occurrence of the word?
2. What's a plausible reason some lines might contain the word "ERROR" more
   than once?
3. What command would give an accurate total occurrence count instead of a
   matching-line count?

<details>
<summary>Answers</summary>

**Incident 1:** "Binary file matches" means `grep` encountered a
non-text (binary/compressed) file — here, a gzip-compressed log archive —
and by default still attempts to search it, which is both slow (binary
data isn't structured the way `grep` expects) and produces a
not-actually-useful "matches" indicator instead of readable line output,
cluttering results. The fix from §1.3 is `--exclude` scoped by extension —
`grep -rn --exclude='*.gz' 'deploy-8842' /var/log/app/` — since the problem
here is a *file type* (compressed archives), not a whole directory to
avoid, making `--exclude` (by pattern) the more natural fit over
`--exclude-dir`, which is for skipping entire directories like `.git`
rather than particular file extensions scattered through a tree.

**Incident 2:** `grep -c` counts matching *lines*, not total *occurrences* —
a single line containing "ERROR" twice still counts as 1 toward `-c`'s
total, which is exactly how a 47-occurrence manual count and a 42-line `-c`
count can both be correct simultaneously: some lines simply contain the word
more than once (e.g. a log line like `ERROR: retrying after ERROR
timeout`). An accurate total-occurrence count needs `grep -o 'ERROR'
app.log | wc -l` — `-o` prints each individual match on its own line
(one per occurrence, not per matching line), and `wc -l` then counts those
output lines, giving the true occurrence total rather than the matching-line
total `-c` provides.

</details>

## Interview Prep

**Q1. When would you reach for `grep -P` instead of `grep -E`, and why isn't `-P` just a "stronger" default choice?**
Reach for `-P` specifically when the pattern needs a capability BRE/ERE
don't have at all — lookaheads/lookbehinds, non-greedy quantifiers, named
groups — not as a general upgrade. `-P` (PCRE support) is a GNU-grep
extension, not guaranteed portable across every system `grep` might run on,
while `-E` (ERE) is standard POSIX behavior available essentially
everywhere — so `-E` remains the more portable default when its capabilities
are sufficient for the pattern at hand.

**Q2. A recursive `grep` search across a large directory tree is slow and its output is cluttered with "Binary file matches" lines. What's the fix?**
Scope the search with `--exclude`/`--include` (by file pattern) or
`--exclude-dir` (by directory), depending on what's actually causing the
noise — binary or compressed files specifically call for `--exclude='*.gz'`
or similar, while irrelevant whole directories (like `.git`) call for
`--exclude-dir`. Scoping the search correctly is both a performance fix and
a signal-to-noise fix.

**Q3. `grep -c 'ERROR' file.log` gives a lower number than a manual count of every occurrence of "ERROR" in the file. Is `grep -c` broken?**
No — `-c` counts matching *lines*, not total *occurrences*. A single line
containing "ERROR" more than once still only counts once toward `-c`'s
total. For a true occurrence count, `grep -o 'ERROR' file.log | wc -l` is
the correct approach — `-o` emits one line per individual match, which `wc
-l` can then count accurately.

**Q4. An investigation needs to find every line matching a pattern, extract a specific field from each, then replace that field's value across the file. Is `grep` alone sufficient?**
No — `grep` can find matching lines and, with `-o`, extract the matched
text, but it has no replace capability at all. Finding is `grep`'s job;
replacing text requires `sed`, and if the task also involves field-aware
processing or aggregation across many lines, `awk` is usually the better
fit for that part specifically. Recognizing this boundary early avoids
building an increasingly awkward `grep`-only pipeline for something a
different tool expresses more directly.

## Key Takeaways

1. **`-P` (PCRE) provides real additional capability, not just different
   syntax for what `-E` already does.** Lookaheads/lookbehinds and
   non-greedy quantifiers don't exist in BRE/ERE at all — reach for `-P`
   specifically when the pattern needs one of those, not as a default
   upgrade, since it's a GNU extension, less portable than `-E`.
2. **Context flags (`-A`/`-B`/`-C`) turn a bare match into something you can
   actually diagnose from.** Asymmetric context (`-A 5 -B 2`) is often more
   useful than symmetric `-C N` when the relevant information isn't evenly
   distributed around the match.
3. **Recursive search needs scoping (`--include`/`--exclude`/
   `--exclude-dir`) to stay fast and clean.** Unscoped recursive `grep`
   across a real directory tree routinely hits binary files, archives, and
   irrelevant subdirectories that waste time and clutter output.
4. **`-c` counts matching lines, not total occurrences.** A line with
   multiple matches still counts once — use `-o | wc -l` for a true
   occurrence count.
5. **A real log investigation is usually a short sequence of narrowing
   `grep` calls, not one command.** Count → locate → contextualize → break
   down by dimension is a repeatable shape worth defaulting to.
6. **`grep` finds and filters; it doesn't transform, and it isn't
   field-aware.** The moment a task needs replacement (`sed`) or
   column/field logic and aggregation (`awk`), reach for the right tool
   rather than stretching `grep` past what it's built for.

> **Demo scope:** Primary concept: using grep effectively for pattern
> search. Supporting concepts: flavor flag selection, context flags,
> recursive search scoping, -v/-c/-l/-o composition, practical
> log-investigation workflows.
> Estimated completion time: 30 minutes (reading + hands-on + verification).
> Checkpoints: 6 natural stopping points (after Task 1 through Task 6).

## Quick Commands Reference

### Flavor and matching

| Flag | Purpose |
|---|---|
| `-E` | ERE (alternation, quantifiers without backslash) |
| `-P` | PCRE — lookaheads/behinds, non-greedy (GNU grep only) |
| `-w` | Whole-word match |

### Context

| Flag | Purpose |
|---|---|
| `-A N` | N lines after each match |
| `-B N` | N lines before each match |
| `-C N` | N lines of context on both sides |

### Recursive and scoped search

| Flag | Purpose |
|---|---|
| `-r` | Recursive |
| `--include='*.ext'` | Only search matching files |
| `--exclude='*.ext'` | Skip matching files |
| `--exclude-dir=DIR` | Skip entire directories |

### Output shaping

| Flag | Purpose |
|---|---|
| `-v` | Invert — non-matching lines |
| `-c` | Count matching lines (not occurrences) |
| `-l` | List matching filenames only |
| `-o` | Print only the matched text |
| `-o \| wc -l` | True occurrence count |

## Appendix A — Anki Cards

**Note:** extracted into a standalone `.csv` by the series extraction script.

```
#deck:Linux Mastery::Module 4 - Text Processing & Scripting for DevOps::18-grep
#separator:Comma
#columns:Front,Back,Tags
"When should you reach for grep -P instead of grep -E?","When the pattern needs a capability BRE/ERE don't have at all — lookaheads/lookbehinds, non-greedy quantifiers, named groups — not as a general stronger default. -P is a GNU extension, less portable than -E.","demo18,grep,pcre"
"grep -c 'ERROR' file.log gives 42, but a manual count of every ERROR occurrence gives 47. Is grep -c broken?","No — -c counts matching lines, not total occurrences. A single line with ERROR appearing twice still counts once. grep -o 'ERROR' file.log | wc -l gives the true occurrence count.","demo18,grep,count,common-mistake"
"A recursive grep across a directory prints 'Binary file X matches' and runs slowly. What's the fix?","Scope the search with --exclude (by file pattern, e.g. --exclude='*.gz') for specific file types, or --exclude-dir for entire directories like .git — this avoids grep wasting time on binary/compressed files.","demo18,grep,recursive,troubleshooting"
"What's the difference between --include and --exclude-dir in a recursive grep?","--include scopes the search TO files matching a glob pattern (e.g. '*.conf'); --exclude-dir skips ENTIRE directories by name (e.g. .git) regardless of what's inside them.","demo18,grep,recursive,fundamentals"
"What does grep -o actually change about grep's output?","It prints only the matched text itself, not the whole line containing the match — turning grep from a line-filter into a text-extraction tool.","demo18,grep,extraction"
"Why is grep -oP '(?<=user=)\\w+' able to extract just the username without 'user=' in the output, while grep -oE 'user=\\w+' cannot?","-P enables PCRE, which supports lookbehind — matching a position based on what precedes it without including that preceding text in the match itself. ERE has no lookbehind mechanism, so -oE's output necessarily includes the full matched text, 'user=' included.","demo18,grep,pcre,lookbehind"
"What's a repeatable four-step shape for a grep-based log investigation?","1) Count matches (-c) to gauge scope. 2) Locate the first occurrence (-n | head). 3) Get surrounding context (-A/-B/-C). 4) Break down by a relevant dimension (extract a field with -o, then sort | uniq -c).","demo18,grep,workflow"
"grep finds every line matching a pattern. Is grep alone sufficient to also replace that pattern's text across the file?","No — grep has no replace/transform capability at all. Finding is grep's job; replacing text is sed's job (Demo 20).","demo18,grep,tool-boundary"
"Why is grep generally the wrong tool for extracting and summing a numeric field across many log lines?","grep can extract matched text (-o) but has no arithmetic or field-aware aggregation capability — that's awk's job (Demo 21), which processes fields and can perform calculations across many lines in one pass.","demo18,grep,tool-boundary,awk"
"grep -A 5 -B 2 'connection refused' app.log uses asymmetric context. When is this more useful than grep -C N?","When the relevant information isn't evenly distributed around the match — e.g. more useful to see what happened after a connection failure than what led up to it, so more After-context than Before-context is deliberately chosen.","demo18,grep,context-flags"
```

## Appendix B — Quiz

**Note:** extracted into a standalone `.md` by the series extraction script.

> One correct answer per question unless stated otherwise.
> Target: 80% or above before moving to Demo 19.

**Q1. When is `grep -P` the right choice over `grep -E`?**

- A) Always — `-P` is strictly better in every case
- B) When the pattern needs a capability BRE/ERE don't have at all, like lookaheads/lookbehinds or non-greedy quantifiers
- C) `-P` is required whenever using character classes
- D) `-P` is faster than `-E` for all patterns

<details>
<summary>Answer</summary>

**B** — `-P` should be reached for based on a genuine capability need, not as a default, since it's a GNU extension and less portable than `-E`. Trap: A, C, and D all overstate or misstate when `-P` is actually needed.

</details>

---

**Q2. `grep -c 'ERROR' file.log` returns 42; manually counting every occurrence of "ERROR" gives 47. What explains this?**

- A) `grep -c` has a counting bug
- B) `-c` counts matching lines, not total occurrences — some lines contain "ERROR" more than once
- C) The file must have changed between the two counts
- D) `grep -c` only counts the first 42 matches by default

<details>
<summary>Answer</summary>

**B** — This is expected `-c` behavior, not a bug. Trap: A and D both invent flaws/limits that don't exist; C assumes an external explanation not indicated by the scenario.

</details>

---

**Q3. A recursive `grep` prints "Binary file X matches" and runs slowly across a large tree. What's the most direct fix?**

- A) Add `-a` to force text-mode on all files unconditionally
- B) Scope the search with `--exclude`/`--exclude-dir` to skip binary/compressed files or irrelevant directories
- C) This can only be fixed by first decompressing every file in the tree
- D) Switch to `grep -P`, which doesn't have this issue

<details>
<summary>Answer</summary>

**B** — Scoping the search away from the actual problem files/directories is the direct, correct fix. Trap: A treats symptoms rather than avoiding the noise source; C overstates the effort required; D is unrelated — `-P` doesn't change recursive file-type handling.

</details>

---

**Q4. What does `grep -o` change about the output compared to default `grep`?**

- A) It prints only the matched text, not the whole line
- B) It only searches the first occurrence per file
- C) It disables regex entirely, doing literal string search only
- D) It reverses the match — showing non-matches

<details>
<summary>Answer</summary>

**A** — `-o` extracts just the matched text, useful for pulling fragments out for further processing. Trap: B, C, and D all describe behaviors of different flags or invent behavior `-o` doesn't have.

</details>

---

**Q5. `grep -oE 'user=\w+'` includes "user=" in its output; `grep -oP '(?<=user=)\w+'` does not. Why?**

- A) `-E` is broken for this pattern
- B) `-P` enables lookbehind, matching based on what precedes a position without including that text in the match; ERE has no lookbehind
- C) `-o` behaves differently depending on which flavor flag is used
- D) The two patterns are actually identical and this is a trick question

<details>
<summary>Answer</summary>

**B** — Lookbehind is a genuine PCRE capability ERE lacks entirely, not a different way of writing the same thing. Trap: A wrongly implies a bug; C misattributes the difference to `-o` rather than the flavor; D ignores the real capability gap.

</details>

---

**Q6. What's the correct command to get a TRUE total occurrence count of "ERROR" in a file (not a matching-line count)?**

- A) `grep -c 'ERROR' file.log`
- B) `grep -o 'ERROR' file.log | wc -l`
- C) `grep -l 'ERROR' file.log`
- D) `grep -v 'ERROR' file.log | wc -l`

<details>
<summary>Answer</summary>

**B** — `-o` emits one line per individual match, which `wc -l` then counts accurately as a true occurrence total. Trap: A gives a matching-line count (the exact limitation this question is testing); C lists filenames, not counts; D counts non-matching lines, the opposite of what's needed.

</details>

---

**Q7. What's the correct repeatable shape for a `grep`-based log investigation, per this demo?**

- A) Extract fields first, then count, then search
- B) Count matches, locate the first occurrence, get surrounding context, then break down by a relevant dimension
- C) Always start with `-v` to filter noise before anything else
- D) There is no useful repeatable pattern — every investigation is unique

<details>
<summary>Answer</summary>

**B** — This count → locate → contextualize → break-down progression is the demo's explicit repeatable workflow shape. Trap: A reverses a sensible order; C overgeneralizes one specific technique as a universal first step; D dismisses a genuinely reusable pattern.

</details>

---

**Q8. An investigation needs to find matching lines, extract a field, and then replace that field's value across the file. Is `grep` alone sufficient?**

- A) Yes, using `grep -o` for extraction and `grep -R` for replacement
- B) No — `grep` has no replace/transform capability at all; that's `sed`'s job
- C) Yes, but only with the `-P` flag enabled
- D) No — this task is impossible with any standard Linux tool

<details>
<summary>Answer</summary>

**B** — `grep` finds and filters (and with `-o`, extracts) but never modifies text; replacement requires `sed`. Trap: A invents a `-R` replace flag that doesn't exist; C wrongly ties the limitation to a flag; D overstates the difficulty — the right combination of tools (grep + sed) handles this easily.

</details>