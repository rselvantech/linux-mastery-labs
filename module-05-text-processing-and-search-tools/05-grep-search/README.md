# Demo M05-05 — grep Search

## Overview

Demo M05-01 built the regex vocabulary; this demo is about wielding `grep`
itself as a fast, practical search tool — not just matching patterns, but
searching efficiently across directory trees, getting the surrounding
context that turns a match into a diagnosis, and building the kind of
log-searching muscle memory that makes incident investigation faster. By the
end, `grep` should feel like a precision instrument, not just "the thing you
pipe to."

## Demo Objectives

1. Choose the right matching mode deliberately — `-E`/`-P` for regex
   power, `-F` for literal strings — building on Demo M05-01, and search for
   multiple patterns at once with `-e`/`-f`
2. Use context flags (`-A`/`-B`/`-C`) to see what surrounds a match
3. Search recursively across a directory tree efficiently, with correct
   filename-prefix behavior
4. Combine `-v`, `-c`, `-l`, `-o`, `-q`, and `-x` for different
   investigative purposes, including what `-c`'s output means for exit status
5. Build practical multi-step log-searching workflows, including correct
   filter-then-extract pipeline ordering
6. Recognize when `grep` is the wrong tool and something else (Demo M05-06/20/21) fits better

## Directory Structure

```
05-grep-search/
├── README.md                    # this file
├── m05-05-grep-search-anki.csv  # Anki flashcard deck
└── m05-05-grep-search-quiz.md   # standalone quiz
```

## Environments

| Task | Ubuntu 24.04 | Rocky Linux 9.8 | EC2 Amazon Linux |
|---|---|---|---|
| All lab tasks | ✅ | ✅ | Not required |

## Recall Check — 01-regular-expressions

Answer these from memory before continuing — no looking back at Demo M05-01.

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

### 1.1 Choosing the matching mode deliberately

Demo M05-01 covered *why* flavor matters; here's the practical habit: decide the
mode before writing the pattern, not after it silently doesn't match.

```bash
grep 'ERROR\|WARN' app.log         # BRE, escaped alternation
grep -E 'ERROR|WARN' app.log       # ERE, cleaner and more common in practice
grep -P '(?<=user=)\w+' app.log    # PCRE — lookbehind, a feature BRE/ERE
                                    # don't have at all, not just different
                                    # syntax for something they do have
```

`-P` isn't just "ERE with different escaping" — PCRE adds real capability
BRE/ERE lack entirely: lookaheads/lookbehinds, non-greedy quantifiers (Demo
M05-01 §1.5's greedy trap), and named groups. Reach for `-P` specifically when
the pattern needs one of those, not as a general "stronger" default —
plain `-E` remains more portable (PCRE support is GNU-grep-specific, not
guaranteed on every system).

**A fourth, genuinely distinct mode: `-F` (fixed strings) turns off
pattern matching entirely** — every character in the search term is
treated as a literal, with no regex interpretation of `.`, `*`, `[`, or
anything else:

```bash
printf '192.168.1.1\n192x168x1x1\nprice: $9.99\n' > /tmp/grep-lab0.txt
grep '192.168' /tmp/grep-lab0.txt      # unescaped . matches BOTH lines
grep -F '192.168' /tmp/grep-lab0.txt   # literal string — only the real IP
```
```
-- grep '192.168' --
192.168.1.1
192x168x1x1
-- grep -F '192.168' --
192.168.1.1
```
> Live-verified. This is the practical alternative to Demo M05-01 §1.1's
> manual escaping (`192\.168`) — for a search term that's already a plain
> string with no intentional regex metacharacters in it (a version
> number, a literal filename, a price), `-F` is both simpler to write and
> immune to accidentally-unescaped metacharacters causing an overmatch,
> since nothing in the term is interpreted as special in the first place.

**Two legacy names worth recognizing, not using:** `egrep` and `fgrep`
predate `-E`/`-F` as flags and still exist on most systems — but as thin
wrapper scripts, not separate tools:

```bash
cat $(which egrep)
```
```
#!/bin/sh
cmd=${0##*/}
exec grep -E "$@"
```
> Live-verified — `egrep` on this system is literally `exec grep -E`;
> `fgrep` is the equivalent for `-F`. GNU `grep`'s own documentation
> describes these as obsolescent. You'll still see them in older scripts
> and documentation, which is the only reason to recognize them — write
> `grep -E`/`grep -F` directly in anything new.

**One more foundational fact worth stating plainly, since every example
so far has used it without saying so: `grep` reads from stdin when no
file argument is given** — this is what makes `echo "..." | grep
'PATTERN'` and every other piped `grep` usage in this series work at all,
not a special piping mode, just the ordinary behavior of "no file named →
read stdin instead."

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

**Filename-prefix display isn't tied to `-r` specifically — it's tied to
"how many files are actually being searched":**

```bash
grep 'ERROR' single.log             # one file named → no filename prefix
grep 'ERROR' log1.log log2.log      # multiple files named → prefix added automatically
grep -H 'ERROR' single.log          # -H: force the prefix even for one file
grep -h 'ERROR' log1.log log2.log   # -h: suppress the prefix even for many
```
> Live-verified. This matters for scripting specifically: a command whose
> file-count varies at runtime (a glob that might expand to one file or
> several) can silently change output format between runs unless `-H`
> pins the prefix on deliberately — worth knowing before a downstream
> parser breaks on a run where only one log file happened to match.

### 1.4 Multiple patterns in one search: `-e` and `-f`

```bash
grep -e 'error' -e 'warning' app.log
# -e PATTERN, repeated: search for EITHER pattern — equivalent to
# 'error|warning' in ERE, but useful when patterns are complex enough
# that combining them into one alternation would hurt readability, or
# when patterns are generated programmatically and appended one at a time
```

```bash
printf 'error\nwarning\n' > /tmp/grep-patterns.txt
grep -f /tmp/grep-patterns.txt app.log
# -f FILE: read patterns from a file, one per line, matching any of them —
# the natural choice when the pattern list is long, shared across scripts,
# or maintained separately from the command that uses it
```
> Live-verified — both produce identical results to a single `'error|warning'`
> alternation, via a different construction mechanism. `-e`/`-f` don't add
> new matching capability beyond what alternation already provides; they
> add a more maintainable way to build up that alternation, especially
> when it comes from multiple sources or grows over time.

### 1.5 -v, -c, -l, -o, -q, -x — six flags, six different investigative purposes

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

**A precision worth having on `-c`: it can legitimately print `0`, and
that `0` still comes with a non-zero exit status.**

```bash
grep -c 'ERROR' clean.log
echo "exit code: $?"
```
```
0
exit code: 1
```
> Live-verified. `-c`'s printed count and `grep`'s exit status are two
> separate signals that happen to agree here, but conflating them is a
> real scripting trap: `if [ "$(grep -c PATTERN file)" -gt 0 ]` works, but
> it's the wrong idiom — it runs `grep` once, captures output, then runs a
> *second* comparison — when `grep -q PATTERN file` (§1.6) does the same
> job in one step, using the exit status directly rather than parsing a
> count back out of text output.

**`-x` — whole-line match, distinct from `-w`'s whole-word match (Demo
M05-01 §1.3):**

```bash
printf 'root\nrooter\nthe root user\n' > /tmp/grep-lab5.txt
echo "-- -w: root as a whole WORD, anywhere in the line --"
grep -w 'root' /tmp/grep-lab5.txt
echo "-- -x: the entire LINE must equal 'root', nothing more --"
grep -x 'root' /tmp/grep-lab5.txt
```
```
-- -w: root as a whole WORD, anywhere in the line --
root
the root user
-- -x: the entire LINE must equal 'root', nothing more --
root
```
> Live-verified. `-w` matched two lines (both contain "root" as a whole
> word); `-x` matched only the line that is *exactly* "root" and nothing
> else — "the root user" fails `-x` even though "root" appears as a
> clean whole word within it. `-x PATTERN` is functionally equivalent to
> anchoring the pattern with `^PATTERN$` (Demo M05-01 §1.3), just without
> writing the anchors explicitly — useful for exact-value checks like "is
> this config value set to precisely `enabled`, not `enabled-partial` or
> similar."

```bash
grep -q 'ERROR' app.log
# -q: quiet — suppresses all output, communicates only through exit status:
# 0 if a match was found, 1 if not, 2 on an error (e.g. file doesn't exist).
```

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

### 1.6 Practical log-searching workflows

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
# sort | uniq -c here is a preview of Demo M05-06 (Text Processing Utilities) —
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

### 1.7 When grep is the wrong tool

`grep` searches and filters; it doesn't transform. The moment an
investigation needs to *change* text (not just find it), or needs
field-aware processing (column N of a structured line, not just "does this
line match"), `grep` isn't the right tool anymore:

- **Need to replace text, not just find it?** That's `sed` (Demo M05-07).
- **Need to process specific fields/columns, do arithmetic, or aggregate
  across many lines?** That's `awk` (Demo M05-08).

Recognizing this boundary early avoids the common anti-pattern of building
an increasingly elaborate `grep`+`sed`+manual-parsing pipeline for something
`awk` would express in one clean pass.

## Part 2 — Hands-On Lab

### Commands used in this demo

| Command | Flag meaning | Purpose |
|---|---|---|
| `grep -P 'PATTERN'` | `-P`: PCRE | Lookaheads/behinds, non-greedy — real capability, not just syntax |
| `grep -F 'STRING'` | `-F`: fixed string | Literal matching, no regex interpretation at all |
| `grep -e P1 -e P2` / `-f FILE` | multiple patterns | Match any of several patterns |
| `grep -A N` / `-B N` / `-C N` | After/Before/Context lines | Show surrounding lines |
| `grep -r` / `-rn` / `-rl` | recursive, +line numbers, +filenames-only | Search a directory tree |
| `grep --include=GLOB` / `--exclude-dir=DIR` | — | Scope recursive search |
| `grep -H` / `-h` | force/suppress filename prefix | Control filename display independent of file count |
| `grep -v` | invert | Show non-matching lines |
| `grep -c` | count | Count matching lines |
| `grep -o` | only-matching | Extract just the matched text |
| `grep -x 'PATTERN'` | `-x`: whole-line | Entire line must equal the pattern |
| `grep -q 'PATTERN' file` | `-q`: quiet | Yes/no test via exit status only, for shell conditionals |

### Task 1 — Mode selection: BRE, ERE, PCRE, and fixed-string

```bash
printf '192.168.1.1\n192x168x1x1\nprice: $9.99\n' > /tmp/app-ips.txt
grep '192.168' /tmp/app-ips.txt        # unescaped . overmatches
grep -F '192.168' /tmp/app-ips.txt     # fixed-string — only the real IP

echo "user=wadmin session=abc123" | grep -oP '(?<=user=)\w+'
# PCRE lookbehind — extracts just "wadmin"
```

### Task 2 — Context flags on a synthetic log

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

### Task 3 — Recursive search with scoping, and filename-prefix control

```bash
mkdir -p /tmp/lab-configs/app /tmp/lab-configs/nginx /tmp/lab-configs/.git
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

grep -h 'listen' /tmp/lab-configs/nginx/site.conf /tmp/lab-configs/app/app.yaml
# -h: filenames suppressed even with two files named explicitly
```

### Task 4 — Multiple patterns with -e and -f

```bash
printf 'error\nwarning\ninfo\ndebug\n' > /tmp/levels.txt
grep -e 'error' -e 'warning' /tmp/levels.txt

printf 'error\nwarning\n' > /tmp/patterns.txt
grep -f /tmp/patterns.txt /tmp/levels.txt
# Confirm both approaches return the same two lines
```

### Task 5 — -v, -c, -l, -o, -q, -x composed together

```bash
grep -v 'DEBUG' /tmp/app.log
grep -c 'INFO' /tmp/app.log
grep -o 'worker-[0-9]*' /tmp/app.log

# -c on a file with no matches — confirm the printed 0 AND the exit status:
grep -c 'CRITICAL' /tmp/app.log
echo "exit code: $?"

# -l is only a meaningful demonstration across more than one file — build
# a second log with no ERROR lines, using grep -v itself:
grep -v 'ERROR' /tmp/app.log > /tmp/app2.log
grep -l 'ERROR' /tmp/app.log /tmp/app2.log
# Only /tmp/app.log is listed — app2.log has none, demonstrating -l's
# "which of these files even contain a match" purpose directly.

grep -v 'DEBUG' /tmp/app.log | grep -c 'INFO'

# -x vs -w on a small test file:
printf 'worker\nworker-3\nthe worker restarted\n' > /tmp/worker-lines.txt
grep -w 'worker' /tmp/worker-lines.txt    # matches lines 1 and 3
grep -x 'worker' /tmp/worker-lines.txt    # matches only line 1 exactly

# -q as a pure yes/no test, no output at all:
grep -q 'ERROR' /tmp/app.log && echo "errors present"
grep -q 'CRITICAL' /tmp/app.log || echo "no CRITICAL entries found"
```

### Task 6 — A four-step log investigation

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

### Task 7 — Recognize the tool boundary

```bash
grep 'ERROR' /tmp/app.log
# grep finds it. Now: replace ERROR with CRITICAL in the file — grep alone
# cannot do this (no replace capability); this is what Demo M05-07 (sed) is for.
# Extract and sum a numeric field across matching lines — grep alone can
# extract the field (-o) but not do arithmetic on it; that's Demo M05-08 (awk).
```

## What You Learned

1. ✅ Chose the right matching mode deliberately — `-E`/`-P` for regex,
   `-F` for literal strings — and searched for multiple patterns with `-e`/`-f`
2. ✅ Used context flags (`-A`/`-B`/`-C`) to see what surrounds a match
3. ✅ Searched recursively across a directory tree, scoped with
   `--include`/`--exclude-dir`, with correct filename-prefix control (`-H`/`-h`)
4. ✅ Combined `-v`, `-c`, `-l`, `-o`, `-q`, and `-x` for different
   investigative purposes, including what `-c`'s zero-count output means
   for exit status
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
1. Reference §1.5 — what does `grep -c` actually count, and how could that
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

**Q2. A search term contains characters like `.` and `$` that are meaningful in the data (a price, a version number) but shouldn't be treated as regex metacharacters. What's the cleanest way to search for it?**

`grep -F` — fixed-string mode disables regex interpretation entirely, so
every character in the search term is matched literally with no escaping
needed at all. This is simpler and less error-prone than manually escaping
every metacharacter in the term (`\.` for each dot, `\$` for each dollar
sign), and it removes the risk of an accidentally-unescaped character
silently overmatching.

---

**Q3. A recursive `grep` search across a large directory tree is slow and its output is cluttered with "Binary file matches" lines. What's the fix?**

Scope the search with `--exclude`/`--include` (by file pattern) or
`--exclude-dir` (by directory), depending on what's actually causing the
noise — binary or compressed files specifically call for `--exclude='*.gz'`
or similar, while irrelevant whole directories (like `.git`) call for
`--exclude-dir`. Scoping the search correctly is both a performance fix and
a signal-to-noise fix.

---

**Q4. `grep -c 'ERROR' file.log` gives a lower number than a manual count of every occurrence of "ERROR" in the file. Is `grep -c` broken?**

No — `-c` counts matching *lines*, not total *occurrences*. A single line
containing "ERROR" more than once still only counts once toward `-c`'s
total. For a true occurrence count, `grep -o 'ERROR' file.log | wc -l` is
the correct approach — `-o` emits one line per individual match, which `wc
-l` can then count accurately.

---

**Q5. An investigation needs to find every line matching a pattern, extract a specific field from each, then replace that field's value across the file. Is `grep` alone sufficient?**

No — `grep` can find matching lines and, with `-o`, extract the matched
text, but it has no replace capability at all. Finding is `grep`'s job;
replacing text requires `sed`, and if the task also involves field-aware
processing or aggregation across many lines, `awk` is usually the better
fit for that part specifically. Recognizing this boundary early avoids
building an increasingly awkward `grep`-only pipeline for something a
different tool expresses more directly.

---

## Key Takeaways

1. **`-F` (fixed strings) disables regex interpretation entirely — the practical alternative to manually escaping every metacharacter in a literal search term.** For a search term with no intentional regex meaning (a version number, a price, a literal filename), `-F` is simpler and immune to accidental overmatch from an unescaped `.` or similar.
2. **`-P` (PCRE) provides real additional capability, not just different syntax for what `-E` already does.** Lookaheads/lookbehinds and non-greedy quantifiers don't exist in BRE/ERE at all — reach for `-P` specifically when the pattern needs one of those, since it's a GNU extension, less portable than `-E`.
3. **`-e`/`-f` build up multi-pattern searches without needing one large alternation.** Functionally equivalent to `'pattern1|pattern2'`, but more maintainable when patterns come from multiple sources, grow over time, or are shared across scripts via a pattern file.
4. **Context flags (`-A`/`-B`/`-C`) turn a bare match into something you can actually diagnose from.** Asymmetric context (`-A 5 -B 2`) is often more useful than symmetric `-C N` when the relevant information isn't evenly distributed around the match.
5. **Recursive search needs scoping (`--include`/`--exclude`/`--exclude-dir`) to stay fast and clean, and filename-prefix display (`-H`/`-h`) can be pinned deliberately** rather than left to vary silently based on how many files a glob happens to expand to at runtime.
6. **`-c` counts matching lines, not total occurrences, and its printed count is a separate signal from its exit status.** A line with multiple matches still counts once; a `0` count still comes with a non-zero exit code — `grep -q` is the correct, more direct idiom for a shell conditional, not parsing `-c`'s text output back into a comparison.
7. **`-x` requires the whole line to equal the pattern; `-w` (Demo M05-01) only requires a whole word within the line.** `-x PATTERN` behaves like `^PATTERN$` without writing the anchors explicitly — the right tool for exact-value checks, not substring-within-a-word checks.
8. **A real log investigation is usually a short sequence of narrowing `grep` calls, not one command.** Count → locate → contextualize → break down by dimension is a repeatable shape worth defaulting to — and within that shape, filter to the relevant lines *before* extracting a field from them, not after.
9. **`grep` finds and filters; it doesn't transform, and it isn't field-aware.** The moment a task needs replacement (`sed`) or column/field logic and aggregation (`awk`), reach for the right tool rather than stretching `grep` past what it's built for.

> **Demo scope:** Primary concept: using grep effectively for pattern
> search. Supporting concepts: matching-mode selection (BRE/ERE/PCRE/fixed
> string), multi-pattern search, context flags, recursive search scoping
> and filename-prefix control, -v/-c/-l/-o/-q/-x composition, practical
> log-investigation workflows and pipeline ordering.
> Estimated completion time: 40 minutes (reading + hands-on + verification).
> Checkpoints: 7 natural stopping points (after Task 1 through Task 7).

## Quick Commands Reference

### Matching mode

| Flag | Purpose |
|---|---|
| `-E` | ERE (alternation, quantifiers without backslash) |
| `-P` | PCRE — lookaheads/behinds, non-greedy (GNU grep only) |
| `-F` | Fixed string — no regex interpretation at all |
| `-e PATTERN` (repeatable) | Match any of several patterns |
| `-f FILE` | Read patterns from a file, one per line |

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
| `-H` / `-h` | Force / suppress filename prefix, independent of file count |

### Output shaping

| Flag | Purpose |
|---|---|
| `-v` | Invert — non-matching lines |
| `-c` | Count matching lines (not occurrences); still exits 1 if count is 0 |
| `-l` | List matching filenames only (fast — stops at first match per file) |
| `-o` | Print only the matched text |
| `-o \| wc -l` | True occurrence count |
| `-w` / `-x` | Whole word (Demo M05-01) / whole line — different scopes |
| `-q` | Silent yes/no test via exit status, for shell conditionals |

## Appendix A — Anki Cards

**`m05-05-grep-search-anki.csv`:**

```
#deck:Linux Mastery::Module 5 - Text Processing & Search Tools::Demo M05-05 - grep Search
#separator:Comma
#columns:Front,Back,Tags
"What does grep -F do differently from plain grep, and when is it the better choice?","-F (fixed strings) disables regex interpretation entirely — every character is matched literally. It's the better choice for a search term with no intentional regex meaning (a version number, a price), avoiding the need to manually escape every metacharacter and the risk of an accidentally-unescaped one overmatching.","demo-m05-05,grep,fixed-string,cross-demo"
"Are egrep and fgrep separate tools from grep?","No — they're thin, obsolescent wrapper scripts. egrep runs exec grep -E; fgrep runs the equivalent for -F. Still seen in older scripts, but grep -E/-F should be used directly in anything new.","demo-m05-05,grep,egrep,fgrep,legacy"
"What's the difference between grep -e 'p1' -e 'p2' and grep -f patternfile, versus just writing grep -E 'p1|p2'?","All three produce the same match results — -e/-f are alternative, more maintainable ways to build up the same alternation, useful when patterns come from multiple sources, grow over time, or are shared across scripts via a file.","demo-m05-05,grep,multi-pattern"
"When should you reach for grep -P instead of grep -E?","When the pattern needs a capability BRE/ERE don't have at all — lookaheads/lookbehinds, non-greedy quantifiers, named groups — not as a general stronger default. -P is a GNU extension, less portable than -E.","demo-m05-05,grep,pcre"
"grep -c 'ERROR' file.log gives 42, but a manual count of every ERROR occurrence gives 47. Is grep -c broken?","No — -c counts matching lines, not total occurrences. A single line with ERROR appearing twice still counts once. grep -o 'ERROR' file.log | wc -l gives the true occurrence count.","demo-m05-05,grep,count,common-mistake"
"grep -c 'PATTERN' file prints 0 (no matches). What is grep's exit status in this case?","1 (no match found) — the printed count and the exit status are two separate signals. This matters for scripting: grep -q PATTERN file is the correct, more direct idiom for a conditional, not parsing -c's text output back into a numeric comparison.","demo-m05-05,grep,exit-status,count"
"What's the difference between grep -w and grep -x?","-w (Demo M05-01) matches a whole WORD anywhere within the line. -x requires the ENTIRE LINE to equal the pattern exactly — 'the root user' matches -w 'root' but not -x 'root'.","demo-m05-05,grep,whole-line,cross-demo"
"A recursive grep across a directory prints 'Binary file X matches' and runs slowly. What's the fix?","Scope the search with --exclude (by file pattern, e.g. --exclude='*.gz') for specific file types, or --exclude-dir for entire directories like .git — this avoids grep wasting time on binary/compressed files.","demo-m05-05,grep,recursive,troubleshooting"
"What's the difference between --include and --exclude-dir in a recursive grep?","--include scopes the search TO files matching a glob pattern (e.g. '*.conf'); --exclude-dir skips ENTIRE directories by name (e.g. .git) regardless of what's inside them.","demo-m05-05,grep,recursive,fundamentals"
"Does grep show a filename prefix when searching a single file? What controls this independent of file count?","No filename prefix by default for a single file; it appears automatically once multiple files are given. -H forces the prefix even for one file; -h suppresses it even for many — useful when a script's file count varies at runtime (e.g. via a glob) and consistent output format matters.","demo-m05-05,grep,filename-prefix"
"What does grep -o actually change about grep's output?","It prints only the matched text itself, not the whole line containing the match — turning grep from a line-filter into a text-extraction tool.","demo-m05-05,grep,extraction"
"What's a repeatable four-step shape for a grep-based log investigation?","1) Count matches (-c) to gauge scope. 2) Locate the first occurrence (-n | head). 3) Get surrounding context (-A/-B/-C). 4) Break down by a relevant dimension (extract a field with -o, then sort | uniq -c).","demo-m05-05,grep,workflow"
"grep finds every line matching a pattern. Is grep alone sufficient to also replace that pattern's text across the file?","No — grep has no replace/transform capability at all. Finding is grep's job; replacing text is sed's job (Demo M05-07).","demo-m05-05,grep,tool-boundary"
```

## Appendix B — Quiz

**`m05-05-grep-search-quiz.md`:**

````markdown
# Quiz — Demo M05-05: grep Search

> One correct answer per question unless stated otherwise.
> Target: 80% or above before moving to Demo M05-06.

---

**Q1.** A script needs to search for the literal string `v1.2.3` in a file, without treating the dots as "any character." What's the cleanest approach?

- A) `grep 'v1.2.3'` — the dots are close enough to literal in practice
- B) `grep -F 'v1.2.3'` — fixed-string mode disables regex interpretation entirely, matching every character literally with no escaping needed
- C) `grep -P 'v1.2.3'` — PCRE mode automatically treats all patterns as literal
- D) `grep 'v1\.2\.3'` is the only correct approach; `-F` doesn't exist

<details>
<summary>Answer</summary>

**B** — `-F` is specifically designed for this case: literal string matching with zero regex interpretation, avoiding the need to manually escape each metacharacter and the risk of forgetting one.

Trap: A ignores the real overmatch risk from unescaped dots; C misattributes literal-matching behavior to PCRE mode, which is actually MORE regex-capable, not less; D wrongly dismisses `-F` as nonexistent when it's a standard, simpler alternative to manual escaping.

</details>

---

**Q2.** Are `egrep` and `fgrep` separate search tools with their own implementations, distinct from `grep`?

- A) Yes, entirely separate codebases optimized for different use cases
- B) No — they are thin, obsolescent wrapper scripts that invoke `grep -E` and `grep -F` respectively
- C) `egrep` is separate; `fgrep` is an alias for `grep -F`
- D) They were separate tools historically but have been removed from all modern systems

<details>
<summary>Answer</summary>

**B** — Both are wrapper scripts around plain `grep` with the corresponding flag — confirmed directly by inspecting the script content. They still exist on most systems (for compatibility with older scripts) but GNU `grep` documents them as obsolescent.

Trap: A overstates their independence; C wrongly claims only one is a wrapper; D wrongly claims removal, when they remain present specifically for backward compatibility.

</details>

---

**Q3.** `grep -e 'error' -e 'warning' file.log` and `grep -E 'error|warning' file.log` are both run against the same file. Do they produce the same matches?

- A) No — `-e` only matches the first pattern given
- B) Yes — both match any line containing either "error" or "warning"; `-e` is an alternative, more maintainable way to build the same alternation, not a different matching capability
- C) No — `-e` requires both patterns to match on the same line
- D) Yes, but only if the patterns are listed in alphabetical order

<details>
<summary>Answer</summary>

**B** — Repeated `-e` flags match any of the given patterns, functionally equivalent to an ERE alternation — the difference is construction and maintainability, not matching behavior.

Trap: A misreads `-e`'s repeat-for-OR semantics as first-pattern-only; C invents an AND requirement that isn't how `-e` works; D invents an ordering requirement that doesn't exist.

</details>

---

**Q4.** `grep -c 'CRITICAL' file.log` prints `0`. What is `grep`'s exit status in this case, and why does this matter for scripting?

- A) Exit status 0, matching the printed count — the two are always identical
- B) Exit status 1 (no match found) — the printed count and the exit status are separate signals; `grep -q` is the more direct idiom for a conditional than parsing `-c`'s output
- C) Exit status 2, since printing 0 counts as an error condition
- D) `grep -c` never sets an exit status at all

<details>
<summary>Answer</summary>

**B** — A count of `0` still means "no match found," which sets exit status 1 — the numeric text `0` and the exit code are two different outputs that happen to correlate here but shouldn't be conflated in scripts.

Trap: A wrongly assumes the printed number and exit code are the same value; C invents an incorrect error-status mapping; D wrongly claims no exit status is set at all.

</details>

---

**Q5.** `grep -w 'root' file.txt` and `grep -x 'root' file.txt` are run against a file containing the line "the root user". Which one(s) match this line?

- A) Both match
- B) Only `-w` matches — "root" appears as a whole word within the line, but the line as a whole is not exactly "root", so `-x` doesn't match
- C) Only `-x` matches
- D) Neither matches

<details>
<summary>Answer</summary>

**B** — `-w` only requires the word to appear as a whole word anywhere in the line, which "the root user" satisfies. `-x` requires the entire line to equal the pattern exactly, which this line does not — it has additional words before and after "root."

Trap: A wrongly assumes whole-word and whole-line matching are equivalent; C reverses which flag actually matches; D wrongly claims neither matches, ignoring `-w`'s correct match.

</details>

---

**Q6.** `grep -B 2 'ERROR' app.log` finds two matches far apart in the file. In the output, a line containing only `--` appears between the two context blocks. What does this indicate?

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

**Q7.** You run `grep 'connection refused' app.log | grep -o 'backend=[a-z0-9-]*' | sort | uniq -c` and get sensible output. A colleague reverses the order: `grep -o 'backend=[a-z0-9-]*' app.log | grep 'connection refused' | sort | uniq -c`. What happens?

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

**Q8.** `grep 'ERROR' single.log` shows no filename prefix, but `grep 'ERROR' log1.log log2.log` shows one automatically. A script needs consistent output regardless of how many files a glob expands to at runtime. What flag addresses this?

- A) `-r`, since it forces recursive-style output formatting
- B) `-H`, which forces the filename prefix even for a single file — pinning the format so it doesn't silently vary based on file count
- C) `-c`, since counting implies consistent formatting
- D) There is no way to control this; scripts must handle both cases separately

<details>
<summary>Answer</summary>

**B** — `-H` forces the filename prefix regardless of how many files are actually being searched, which is exactly the fix for output that would otherwise vary silently depending on how many files a glob happens to match at runtime.

Trap: A misapplies `-r`, which controls directory traversal, not filename display; C misapplies `-c`, an unrelated counting flag; D overlooks the flag that directly solves this.

</details>

---

Score guide:

| Score | Action |
|---|---|
| 7/8 or 8/8 | Import Anki cards, move to Demo M05-06 |
| 6/8 | Review the wrong answers, then proceed |
| 5/8 | Re-read the relevant sections, retry those questions |
| Below 5/8 | Re-read the full demo and redo the walkthrough before proceeding |
````