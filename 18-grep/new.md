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
4. Combine `-v`, `-c`, `-l`, `-o`, and `-q` for different investigative purposes
5. Build practical multi-step log-searching workflows, including correct
   filter-then-extract pipeline ordering
6. Recognize when `grep` is the wrong tool and something else (Demo 19/20/21) fits better

## Directory Structure

```
18-grep/
├── README.md               # this file
├── 18-grep-anki.csv        # Anki flashcard deck
└── 18-grep-quiz.md         # standalone quiz
```

## Environments

| Task | Ubuntu 24.04 | Rocky Linux 9.8 | EC2 Amazon Linux |
|---|---|---|---|
| All lab tasks | ✅ | ✅ | Not required |

## Recall Check — 17-regular-expressions

Answer these from memory before continuing — no looking back at Demo 17.

1. `grep 'error|warning' file.log` returns 0 matches; `grep -E
   'error|warning' file.log` returns 14. Why?

2. What does the pattern `(\w+) \1` actually match?

3. What are the default regex flavors for `grep`, `sed`, and `awk`
   respectively?

<details>
<summary>Answers</summary>

1. In BRE (grep's default without `-E`), `|` is a literal character,
   not alternation — the pattern searches for the literal string
   `error|warning`. `-E` enables ERE, where `|` means alternation.

2. A word immediately followed by that exact same word again — `\1`
   refers to whatever text the first group actually captured on this match,
   not just "any two words."

3. `grep`: BRE (`-E` for ERE, `-P` for PCRE). `sed`: BRE (`-E`/`-r` for
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

**`--include`/`--exclude` match against filenames, not directory names.**
A directory literally named `conf` isn't specially affected by
`--include='*.conf'` — that glob only filters which file *basenames* are
searched, wherever they live in the tree. For skipping an entire directory
regardless of what's inside it, `--exclude-dir` is the correct flag, not
`--exclude` applied to a directory-shaped pattern.

### 1.4 -v, -c, -l, -o, -q — five flags, five different investigative purposes

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

A performance note on `-l`: it can stop scanning a given file the moment it
finds one match, since all it needs to know is whether at least one exists
— unlike `-c` (which must keep counting every matching line, so it can't
stop early) or plain/`-n` output (which must find every match to print
each one). This makes `-l` meaningfully faster than the alternatives across
very large files or many files, when a yes/no per file is genuinely all you
need.

**A fifth, silent flag worth knowing: `-q` (quiet).** `grep -q PATTERN
file` suppresses all output entirely and communicates only through its
exit status — `0` if a match was found, `1` if not (and `2` if an error
occurred, e.g. the file doesn't exist). Every `grep` invocation actually
sets this exit status regardless of `-q` — `-q` just additionally silences
the output you'd otherwise not want cluttering a conditional check:

```bash
if grep -q 'ERROR' app.log; then
    echo 'errors found'
fi
```

This is the standard idiom for using `grep` purely as a yes/no test inside
shell scripting, rather than caring about the matched text itself.

**These flags compose well together**, and combining them is often more
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
grep 'connection refused' app.log | grep -o 'backend=[a-z0-9-]*' | sort | uniq -c
# sort | uniq -c here is a preview of Demo 19 (Text Processing Utilities) —
# grep found the relevant lines and extracted a field; counting/grouping
# unique values is a different tool's job, not grep's.
```

**Order matters in step 4, and it's easy to get backwards:** filter to the
relevant lines *first* (`connection refused`), *then* extract the field
(`-o`) from what's left. Reversing this — extracting the field first and
trying to search for `'connection refused'` within the already-extracted
fragments — searches text that no longer contains that phrase at all,
since `-o` already stripped everything else away in the first step. The
extracted fragments (`backend=app-01`, etc.) never contained "connection
refused" to begin with; that phrase only existed in the original lines
before extraction removed it. Filter, then extract — not the other way
around.

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
| `grep -q 'PATTERN' file` | `-q`: quiet | Yes/no test via exit status only, for shell conditionals |

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
echo "binary garbage listen 9999" > /tmp/lab-configs/.git/objects-fake

grep -rn 'listen' /tmp/lab-configs/
# All three files match, including the .git one.

grep -rn --include='*.conf' 'listen' /tmp/lab-configs/
# Only nginx/site.conf — the glob restricts by filename, regardless of
# which directory each file lives in.

grep -rn --exclude-dir='.git' 'listen' /tmp/lab-configs/
# nginx and app match; the .git file is gone — excluded by directory name,
# regardless of its own filename or content.
```

### Task 3 — -v, -c, -l, -o, -q composed together

```bash
grep -v 'DEBUG' /tmp/app.log
grep -c 'INFO' /tmp/app.log
grep -o 'worker-[0-9]*' /tmp/app.log

# -l is only a meaningful demonstration across more than one file — build
# a second log with no ERROR lines, using grep -v itself:
grep -v 'ERROR' /tmp/app.log > /tmp/app2.log
grep -l 'ERROR' /tmp/app.log /tmp/app2.log
# Only /tmp/app.log is listed — app2.log has none, demonstrating -l's
# "which of these files even contain a match" purpose directly.

grep -v 'DEBUG' /tmp/app.log | grep -c 'INFO'

# -q as a pure yes/no test, no output at all:
grep -q 'ERROR' /tmp/app.log && echo "errors present"
grep -q 'CRITICAL' /tmp/app.log || echo "no CRITICAL entries found"
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
4. ✅ Combined `-v`, `-c`, `-l`, `-o`, and `-q` for different investigative purposes
5. ✅ Built a practical multi-step log-searching workflow, including correct
   filter-then-extract pipeline ordering
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

---

**Q2. A recursive `grep` search across a large directory tree is slow and its output is cluttered with "Binary file matches" lines. What's the fix?**

Scope the search with `--exclude`/`--include` (by file pattern) or
`--exclude-dir` (by directory), depending on what's actually causing the
noise — binary or compressed files specifically call for `--exclude='*.gz'`
or similar, while irrelevant whole directories (like `.git`) call for
`--exclude-dir`. Scoping the search correctly is both a performance fix and
a signal-to-noise fix.

---

**Q3. `grep -c 'ERROR' file.log` gives a lower number than a manual count of every occurrence of "ERROR" in the file. Is `grep -c` broken?**

No — `-c` counts matching *lines*, not total *occurrences*. A single line
containing "ERROR" more than once still only counts once toward `-c`'s
total. For a true occurrence count, `grep -o 'ERROR' file.log | wc -l` is
the correct approach — `-o` emits one line per individual match, which `wc
-l` can then count accurately.

---

**Q4. An investigation needs to find every line matching a pattern, extract a specific field from each, then replace that field's value across the file. Is `grep` alone sufficient?**

No — `grep` can find matching lines and, with `-o`, extract the matched
text, but it has no replace capability at all. Finding is `grep`'s job;
replacing text requires `sed`, and if the task also involves field-aware
processing or aggregation across many lines, `awk` is usually the better
fit for that part specifically. Recognizing this boundary early avoids
building an increasingly awkward `grep`-only pipeline for something a
different tool expresses more directly.

---

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
   irrelevant subdirectories that waste time and clutter output —
   `--include`/`--exclude` match filenames, `--exclude-dir` matches
   directory names; they aren't interchangeable.
4. **`-c` counts matching lines, not total occurrences.** A line with
   multiple matches still counts once — use `-o | wc -l` for a true
   occurrence count; use `-l` instead of `-c` when you only need a yes/no
   per file, since `-l` can stop at the first match while `-c` must count
   every one.
5. **A real log investigation is usually a short sequence of narrowing
   `grep` calls, not one command.** Count → locate → contextualize → break
   down by dimension is a repeatable shape worth defaulting to — and within
   that shape, filter to the relevant lines *before* extracting a field
   from them, not after, or the extraction strips away the very text a
   later filter step would have needed to search.
6. **`grep` finds and filters; it doesn't transform, and it isn't
   field-aware.** The moment a task needs replacement (`sed`) or
   column/field logic and aggregation (`awk`), reach for the right tool
   rather than stretching `grep` past what it's built for.
7. **`-q` turns `grep` into a pure yes/no test for shell conditionals,**
   communicating only through exit status (`0` = match found, `1` = not
   found, `2` = error) with no output to suppress manually — the standard
   idiom behind `if grep -q PATTERN file; then ...`.

> **Demo scope:** Primary concept: using grep effectively for pattern
> search. Supporting concepts: flavor flag selection, context flags,
> recursive search scoping, -v/-c/-l/-o/-q composition, practical
> log-investigation workflows and pipeline ordering.
> Estimated completion time: 33 minutes (reading + hands-on + verification).
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
| `--include='*.ext'` | Only search matching files (by filename) |
| `--exclude='*.ext'` | Skip matching files (by filename) |
| `--exclude-dir=DIR` | Skip entire directories (by directory name) |

### Output shaping

| Flag | Purpose |
|---|---|
| `-v` | Invert — non-matching lines |
| `-c` | Count matching lines (not occurrences) |
| `-l` | List matching filenames only (fast — stops at first match per file) |
| `-o` | Print only the matched text |
| `-o \| wc -l` | True occurrence count |
| `-q` | Silent yes/no test via exit status, for shell conditionals |

## Appendix A — Anki Cards

**`18-grep-anki.csv`:**

```
#deck:Linux Mastery::Module 4 - Text Processing & Scripting for DevOps::Demo 18 - grep
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

**`18-grep-quiz.md`:**

````markdown
# Quiz — Demo 18: grep

> One correct answer per question unless stated otherwise.
> Target: 80% or above before moving to Demo 19.

---

**Q1.** `grep -B 2 'ERROR' app.log` finds two matches far apart in the file. In the output, a line containing only `--` appears between the two context blocks. What does this indicate?

- A) A syntax error in the search
- B) The two match blocks are not contiguous in the source file — `--` is grep's default visual separator between non-adjacent context groups
- C) Exactly two lines were skipped due to a permissions error
- D) The pattern matched twice on the same line

<details>
<summary>Answer</summary>

**B** — When context windows around separate matches don't overlap or touch, `grep` inserts `--` between them by default, signaling a gap in the source file rather than continuous output.

Trap: A, C, and D all invent unrelated meanings for a separator that specifically indicates non-contiguous match groups.

</details>

---

**Q2.** What does `grep -v 'DEBUG' app.log | grep -c 'INFO'` compute?

- A) The total number of DEBUG lines in the file
- B) The number of INFO lines, after first excluding any lines tagged DEBUG
- C) The number of lines containing neither DEBUG nor INFO
- D) The combined count of DEBUG and INFO lines together

<details>
<summary>Answer</summary>

**B** — The first `grep -v` removes DEBUG-tagged lines from the stream; the second `grep -c` then counts how many of the remaining lines mention INFO — a two-stage filter, each `grep` doing one clear job.

Trap: A misreads which grep produces the final count; C and D both invent combinations that don't match what this specific two-command pipeline computes.

</details>

---

**Q3.** You run `grep 'connection refused' app.log | grep -o 'backend=[a-z0-9-]*' | sort | uniq -c` and get sensible output. A colleague reverses the order: `grep -o 'backend=[a-z0-9-]*' app.log | grep 'connection refused' | sort | uniq -c`. What happens?

- A) The reversed version produces identical output — order doesn't matter for grep pipelines
- B) The reversed version produces no useful output, because the first `grep -o` already stripped away everything except the `backend=...` fragments, which never contained the phrase "connection refused" to begin with
- C) The reversed version runs faster but otherwise behaves the same
- D) The reversed version fails with a syntax error

<details>
<summary>Answer</summary>

**B** — Filtering must happen before extraction when extraction would remove the very text a later step needs to search for. Once `-o` extracts just `backend=app-01`, that fragment alone was never going to contain "connection refused" — that phrase only existed in the original, now-discarded, line text.

Trap: A wrongly assumes pipeline order is irrelevant here; C invents a performance-only difference when the actual issue is correctness; D is wrong — this runs without error, it just silently produces nothing useful.

</details>

---

**Q4.** `grep -l 'ERROR' file1.log file2.log` is run, where `file1.log` contains ERROR lines and `file2.log` does not. What does `-l` need more than one file to meaningfully demonstrate, and what's the output?

- A) `-l` requires at least two files to run at all; with one file it errors out
- B) With multiple files, `-l` lists only the filenames that contain at least one match — here, just `file1.log` — making its "which files even mention this" purpose visible in a way a single-file case wouldn't show as clearly
- C) `-l` merges the contents of all listed files before searching
- D) `-l` requires files to be explicitly sorted alphabetically first

<details>
<summary>Answer</summary>

**B** — `-l`'s value is comparative across multiple files — it tells you which ones are relevant without printing any matched lines. A single-file case technically works too, but doesn't illustrate the "which of these" selection purpose as clearly as seeing one file listed and another correctly omitted.

Trap: A invents a minimum-file requirement that doesn't exist; C and D both invent behaviors `-l` doesn't have.

</details>

---

**Q5.** `grep -rn --include='*.conf' 'listen' /etc/` is run on a tree that happens to contain a directory literally named `conf` (not a `.conf` file). Does `--include='*.conf'` cause that directory to be searched specially?

- A) Yes — `--include` treats directory names the same as filenames
- B) No — `--include`/`--exclude` match against filenames only; a directory named `conf` is not itself a "file matching `*.conf`," and files inside it are included or excluded based on their own filenames, not the directory's name
- C) Yes, but only if the directory name is in all lowercase
- D) No — a directory named `conf` would cause `grep` to error out entirely

<details>
<summary>Answer</summary>

**B** — `--include` and `--exclude` operate on filenames (basenames), not directory names — a directory being named similarly to the glob has no special effect. Directory-level scoping is what `--exclude-dir` is specifically for.

Trap: A and C both invent directory-name matching behavior `--include` doesn't have; D invents an error condition that doesn't occur.

</details>

---

**Q6.** `grep -P 'a.+?b'` is used instead of `grep -E 'a.+b'` against `axxbyyb`. What does the `?` after `+` accomplish here, and is this available in plain ERE?

- A) It makes the `+` optional, identical to `*` — available in both ERE and PCRE
- B) It makes the quantifier non-greedy, matching as little as possible (`axxb` instead of the full `axxbyyb`) — this is a PCRE-only feature, not available in standard BRE/ERE
- C) It has no effect in either flavor; both produce identical output
- D) It's a syntax error in PCRE mode specifically

<details>
<summary>Answer</summary>

**B** — `+?` in PCRE requests the shortest possible match satisfying the pattern, rather than the default greedy longest match — exactly the non-greedy capability Demo 17 flagged as absent from standard BRE/ERE, here shown as an actual `grep -P` usage.

Trap: A misreads `?` as affecting `+`'s optionality rather than its greediness; C wrongly claims no difference exists; D invents an error condition for valid PCRE syntax.

</details>

---

**Q7.** `grep -q 'ERROR' app.log` is run inside a shell script with no `echo` or output-handling around it at all. What, if anything, does this command accomplish?

- A) Nothing useful — without capturing output, the command has no effect
- B) It sets an exit status (0 if a match was found, 1 if not, 2 on error) with no printed output at all — useful specifically when the script checks that exit status afterward, e.g. in an `if grep -q ...; then` conditional
- C) It always exits 0 regardless of whether a match was found, since `-q` suppresses error signaling too
- D) `-q` requires `-o` to be combined with it to be meaningful

<details>
<summary>Answer</summary>

**B** — Every `grep` invocation sets an exit status reflecting whether it found a match; `-q` additionally suppresses the printed output, leaving only that exit status as the useful signal — exactly what a shell conditional checks, without needing to discard unwanted text output separately.

Trap: A wrongly assumes an exit status alone is "no effect" — it's the entire point of `-q`; C invents suppressed error signaling that doesn't happen; D invents an unnecessary flag dependency.

</details>

---

**Q8.** Why can `grep -l 'PATTERN' hugefile.log` be noticeably faster than `grep -c 'PATTERN' hugefile.log` on the same very large file, even though both are just checking for a pattern?

- A) `-l` uses a fundamentally different search algorithm than `-c`
- B) `-l` can stop scanning the file the instant it finds one match, since it only needs to know whether at least one exists; `-c` must keep scanning to count every matching line, so it can't stop early
- C) `-c` always re-reads the file twice; `-l` only reads it once
- D) There is no real performance difference between the two

<details>
<summary>Answer</summary>

**B** — `-l`'s job ("does this file contain a match at all") is satisfied by the very first hit, so scanning can stop immediately; `-c`'s job ("how many matching lines are there") requires examining the rest of the file regardless of how early the first match occurs.

Trap: A invents an unrelated algorithmic difference; C invents a double-read behavior `-c` doesn't have; D dismisses a real, practical difference that matters on genuinely large files.

</details>

---

Score guide:

| Score | Action |
|---|---|
| 7/8 or 8/8 | Import Anki cards, move to Demo 19 |
| 6/8 | Review the wrong answers, then proceed |
| 5/8 | Re-read the relevant sections, retry those questions |
| Below 5/8 | Re-read the full demo and redo the walkthrough before proceeding |
````