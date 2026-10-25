# Demo 21a — awk Multi-File Processing

## Overview

Every `awk` example in Demo 21 processed exactly one file (or one stream).
Hand `awk` two or more files at once, and a new question appears
immediately: when the second file starts, does the line-numbering carry
on from the first file, or start over? The answer — it does both,
depending on which variable you ask — is the foundation for one of
`awk`'s most useful real capabilities: correlating data across separate
files in a single pass, without a database, a join tool, or reading
either file twice.

## Demo Objectives

1. Explain the difference between `NR` (total records across all files)
   and `FNR` (records within the current file), and predict their values
   at any point during multi-file processing
2. Use `FILENAME` to know which file is currently being processed, and
   detect file boundaries with `FNR==1`
3. Use the `FNR==NR` idiom to distinguish "processing the first file"
   from "processing a later file," without hardcoding filenames
4. Implement the classic two-file lookup-table join pattern: build an
   associative array from one file, then enrich or cross-reference a
   second file against it
5. Use `ARGV`/`ARGC` to access and modify the command-line file
   arguments programmatically
6. Recognize the practical limits of this pattern and when a real join
   tool or database is warranted instead

## Directory Structure

```
21a-awk-multifile/
├── README.md                          # this file
├── 21a-awk-multifile-anki.csv         # Anki flashcard deck
└── 21a-awk-multifile-quiz.md          # standalone quiz
```

## Environments

| Task | Ubuntu 24.04 | Rocky Linux 9.8 | EC2 Amazon Linux |
|---|---|---|---|
| All lab tasks | ✅ (gawk) | ✅ (gawk) | Not required |

## Recall Check — 21-awk-processing

Answer these from memory before continuing — no looking back at Demo 21.

1. What's the difference between a pattern-level condition (`$3 > 100
   {print $1}`) and an `if`/`else` written inside an action block?

2. `awk 'BEGIN{if ("10" > "9") ...}'` evaluates this as false, but `awk
   '$1 > 9 {...}'` against input containing "10" matches it as greater
   than 9. Why the difference?

3. What does `printf` give you that plain `print` doesn't, and what's
   one thing `printf` does *not* do automatically that `print` does?

<details>
<summary>Answers</summary>

1. A pattern condition decides whether the action runs at all for a
   line — non-matching lines are silently skipped. An `if`/`else` inside
   an action runs on every line reaching it and branches between outcomes
   each time — every line still produces output, just a different one.

2. Literal string constants written in the program are compared
   lexically by default; a field read from actual input that looks
   numeric is automatically compared numerically. The behavior depends on
   where the value came from, not just its appearance.

3. `printf` gives real formatting control (column widths, decimal
   precision) via `%`-style specifiers. Unlike `print`, it does NOT add a
   trailing newline automatically — `\n` must be written explicitly.

</details>

## Part 1 — Concepts

### 1.1 NR vs FNR — total count vs. per-file count

```bash
printf 'a\nb\nc\n' > file1.txt
printf 'd\ne\n' > file2.txt
awk '{print "NR="NR, "FNR="FNR, FILENAME, $0}' file1.txt file2.txt
```
```
NR=1 FNR=1 file1.txt a
NR=2 FNR=2 file1.txt b
NR=3 FNR=3 file1.txt c
NR=4 FNR=1 file2.txt d
NR=5 FNR=2 file2.txt e
```
> Live-verified. `NR` keeps counting upward across every file given to
> `awk` — it never resets, reaching 5 total across both files here. `FNR`
> counts records *within the current file only* — it resets to 1 the
> moment `awk` starts reading `file2.txt`, exactly the way `NR` alone
> behaved in every single-file example from Demo 21. Whenever a task
> needs "which line number is this within its own file" (for an error
> message, say), `FNR` is the right variable; whenever it needs "how many
> records have I processed in total," `NR` is.

### 1.2 FILENAME and detecting file boundaries

`FILENAME` holds the path of whichever file is currently being read —
already visible in §1.1's output. Combined with the fact that `FNR`
resets to `1` at the start of every new file, `FNR==1` is the standard
way to detect "a new file just started":

```bash
awk 'FNR==1{print "--- new file:", FILENAME, "---"} {print}' file1.txt file2.txt
```
```
--- new file: file1.txt ---
a
b
c
--- new file: file2.txt ---
d
e
```
> Live-verified. This is directly useful for a common real task: skipping
> a header row in *every* file of a multi-file batch (`FNR==1{next}`
> would skip line 1 of each file specifically, not just the very first
> line of the very first file — a distinction that matters the moment
> more than one file is involved).

### 1.3 The FNR==NR idiom — "am I still in the first file?"

`NR` and `FNR` start out equal and stay equal for as long as `awk` is
reading the *first* file given to it — because every record counted by
`FNR` in that file is also the same record counted by `NR` overall. The
moment the second file starts, `FNR` resets to `1` while `NR` keeps
climbing, so they diverge. **`FNR==NR` is therefore true only while
processing the first file, and false for every file after it** — a
standard `awk` idiom for "treat the first file differently from every
subsequent one," without ever hardcoding a filename:

```bash
awk 'FNR==NR{print "first-file line, NR="NR" FNR="FNR; next}
     {print "second-file line, NR="NR" FNR="FNR}' file1.txt file2.txt
```
```
first-file line, NR=1 FNR=1
first-file line, NR=2 FNR=2
first-file line, NR=3 FNR=3
second-file line, NR=4 FNR=1
second-file line, NR=5 FNR=2
```
> Live-verified. `next` (already familiar from skipping header rows) is
> what makes this idiom work correctly: without it, a first-file line
> would fall through and also match the second, unconditional block below
> it. This exact pattern — `FNR==NR {...; next}` followed by an
> unconditional block — is the skeleton every cross-file lookup in this
> demo builds on.

### 1.4 The lookup-table join pattern

This is the practical payoff of §1.3: load one file into an associative
array while it's "the first file," then use that array to look up or
enrich data from the second file:

```bash
printf '101 alice\n102 bob\n103 carol\n' > users.txt
printf '101 500\n102 300\n104 900\n' > orders.txt

awk '
  FNR==NR { name[$1] = $2; next }
  {
    if ($1 in name) print name[$1], $2
    else print "unknown-user-"$1, $2
  }
' users.txt orders.txt
```
```
alice 500
bob 300
unknown-user-104 900
```
> Live-verified. While reading `users.txt` (the first file, so
> `FNR==NR`), each line populates `name[id] = username` and `next`
> immediately moves to the following record without running the second
> block. Once `orders.txt` starts, `FNR==NR` is false, so every order
> line falls through to the lookup block, which checks `$1 in name` — a
> genuine membership test, not just a truthy check — and either prints
> the resolved username or an explicit "unknown" marker. Note the third
> order (`104`) has no matching entry in `users.txt` at all, and the
> program handles that missing-key case explicitly rather than silently
> producing a blank or wrong result — checking `in` before use is what
> makes that possible, the same defensive habit as confirming `$ETCD_DEV`
> is non-empty before use elsewhere in this series.

**This is a real, single-pass join** — `users.txt` is read once into
memory as a lookup table, and `orders.txt` is streamed through once,
each line looked up in O(1) against the array. For files where the
lookup table fits comfortably in memory, this is often faster and
simpler than `join` (which requires both inputs pre-sorted on the join
key) or standing up an actual database for what's fundamentally a
one-off correlation task.

### 1.5 ARGV and ARGC — the file list as data you can inspect and change

`awk` exposes its own command-line arguments as an array, the same way a
shell script can inspect `$1`, `$2`, etc.:

```bash
awk 'BEGIN{for(i=0;i<ARGC;i++) print "ARGV["i"]="ARGV[i]}' file1.txt file2.txt
```
```
ARGV[0]=awk
ARGV[1]=file1.txt
ARGV[2]=file2.txt
```
> Live-verified. `ARGC` (argument count) is `3` here — the program name
> plus two files. `ARGV[0]` is conventionally the program's own name
> (`awk`, mirroring `$0` in a shell script), and `ARGV[1]`/`ARGV[2]` are
> the actual file arguments, in the order given.

**`ARGV` can be modified from within the program, changing what `awk`
actually processes** — setting an entry to an empty string removes it
from the processing list entirely:

```bash
awk 'BEGIN{ ARGV[2]="" } {print FILENAME, $0}' file1.txt file2.txt
```
```
file1.txt a
file1.txt b
file1.txt c
```
> Live-verified — `file2.txt` never gets processed at all once its
> `ARGV` entry is cleared in `BEGIN`, even though it was named on the
> command line. This is a real, if fairly advanced, technique: a `BEGIN`
> block can inspect `ARGV` and conditionally drop files that don't match
> some criterion (a naming pattern, a check against another list) before
> the main processing loop ever begins — dynamic input selection,
> entirely inside the `awk` program itself, rather than requiring the
> shell to pre-filter the file list beforehand.

### 1.6 When this pattern reaches its limit

The lookup-table join is genuinely powerful, but it has real edges worth
knowing before leaning on it for something it isn't suited to:

- **The lookup table must fit in memory.** For a "first file" of a few
  thousand or even a few million short lines, this is a non-issue; for a
  lookup table that's itself huge, `awk`'s associative array becomes the
  bottleneck, and a real database (or `join` against pre-sorted,
  disk-based input) is the more appropriate tool.
- **This pattern joins on exact key equality only.** Fuzzy matching,
  range joins ("orders placed within 5 minutes of a login event"), or
  multi-column composite keys are possible to hand-roll in `awk` but get
  unwieldy fast — exactly the kind of complexity Demo 21 §1.6 flagged as
  the signal to reach for a proper script or a real data tool instead.
- **More than two files compound the bookkeeping.** `FNR==NR` alone only
  distinguishes "first file" from "everything else" — correlating three
  or more files cleanly typically needs `FILENAME`-based branching
  instead of (or in addition to) `FNR==NR`, since the binary "first vs.
  not-first" test doesn't scale to "which of these four files am I in."

## Part 2 — Hands-On Lab

### Commands used in this demo

| Command | Flag/syntax | Purpose |
|---|---|---|
| `NR` / `FNR` | built-in variables | Total record count / current-file record count |
| `FILENAME` | built-in variable | Path of the file currently being read |
| `FNR==1{...}` | file-boundary test | Detect the start of each new file |
| `FNR==NR{...; next}` | idiom | Distinguish "first file" from later files |
| `key in array` | membership test | Check whether a lookup key exists before using it |
| `ARGV[i]` / `ARGC` | built-in array/variable | Inspect or modify the command-line file list |

### Task 1 — NR vs FNR across two files

```bash
mkdir -p /tmp/awk-mf && cd /tmp/awk-mf
printf 'a\nb\nc\n' > file1.txt
printf 'd\ne\n' > file2.txt

awk '{print "NR="NR, "FNR="FNR, FILENAME, $0}' file1.txt file2.txt
# Confirm NR climbs from 1 to 5 across both files without resetting,
# while FNR resets to 1 the moment file2.txt starts.
```

### Task 2 — Detecting file boundaries with FNR==1

```bash
awk 'FNR==1{print "--- new file:", FILENAME, "---"} {print}' file1.txt file2.txt
```

### Task 3 — The FNR==NR idiom

```bash
awk 'FNR==NR{print "first-file line, NR="NR" FNR="FNR; next}
     {print "second-file line, NR="NR" FNR="FNR}' file1.txt file2.txt
# Confirm every file1.txt line is caught by the first block (with next
# preventing fallthrough), and every file2.txt line is caught by the
# second, unconditional block.
```

### Task 4 — Building and using a lookup-table join

```bash
printf '101 alice\n102 bob\n103 carol\n' > users.txt
printf '101 500\n102 300\n104 900\n' > orders.txt

awk '
  FNR==NR { name[$1] = $2; next }
  {
    if ($1 in name) print name[$1], $2
    else print "unknown-user-"$1, $2
  }
' users.txt orders.txt
# Confirm 101 and 102 resolve to real usernames, and 104 (absent from
# users.txt) is explicitly flagged as unknown rather than silently
# printing a blank or incorrect value.
```

### Task 5 — Inspecting and modifying ARGV

```bash
awk 'BEGIN{for(i=0;i<ARGC;i++) print "ARGV["i"]="ARGV[i]}' file1.txt file2.txt

awk 'BEGIN{ ARGV[2]="" } {print FILENAME, $0}' file1.txt file2.txt
# Confirm file2.txt is never processed, even though it was named on the
# command line — the empty ARGV entry removed it before processing began.
```

## What You Learned

1. ✅ Explained and confirmed the `NR` vs `FNR` distinction across a
   real two-file run
2. ✅ Used `FILENAME` and `FNR==1` to detect file boundaries during
   multi-file processing
3. ✅ Used the `FNR==NR` idiom to distinguish the first file from later
   files without hardcoding filenames
4. ✅ Implemented a real two-file lookup-table join, including explicit
   handling of an unmatched key
5. ✅ Inspected `ARGV`/`ARGC` and modified `ARGV` to change which files
   `awk` actually processes
6. ✅ Recognized this pattern's practical limits — memory-bound lookup
   tables, exact-key-only joins, and more-than-two-file bookkeeping

## Interview Prep

**Q1. A script needs to print the line number within each individual log file of a multi-file batch, not a running total across all of them. Which built-in variable is correct, and why would the other one give a wrong answer?**

`FNR` — it resets to 1 at the start of every new file, giving the correct
per-file line number. `NR` keeps counting upward across every file given
to `awk` and never resets, so using it here would report an
ever-increasing number that has nothing to do with position within any
individual file once more than one file is involved.

**Q2. Why does `FNR==NR` work as a reliable way to detect "I'm still processing the first file," without ever naming that file explicitly?**

`FNR` and `NR` start equal and stay equal for as long as `awk` reads the
first file, because every record `FNR` counts there is also the same
record `NR` is counting overall. The instant the second file begins,
`FNR` resets to 1 while `NR` keeps climbing, so they diverge — `FNR==NR`
is true exactly during the first file and false for every file after it,
which works regardless of what any file is actually named.

**Q3. Two files need to be correlated by a shared ID column — one holds lookup data, the other holds records to enrich. What's the `awk` pattern for this, and what happens to a record whose ID has no match in the lookup file?**

The `FNR==NR {...; next}` skeleton: while reading the first (lookup)
file, populate an associative array keyed by the ID; `next` skips the
second block for those lines. Once the second file starts, `FNR==NR` is
false, so every line falls through to a lookup against the array. A
record whose ID has no match should be checked explicitly with `key in
array` before use — printing a placeholder or a clearly-marked "unknown"
result — rather than letting an unchecked lookup silently produce a
blank or incorrect value.

## Key Takeaways

---

1. **`NR` counts records across every file given to `awk` and never resets; `FNR` counts records within the current file only and resets to 1 at the start of each new file.** Use `FNR` for "which line is this within its own file"; use `NR` for "how many records total have I seen."
2. **`FNR==1` is the standard test for "a new file just started," and `FILENAME` reports which one.** This is the mechanism behind per-file logic like skipping a header row in every file of a batch, not just the very first line overall.
3. **`FNR==NR` is true only while processing the first file given to `awk`, and false for every file after it — a filename-free way to distinguish "first file" from "later files."** Pair it with `next` to prevent first-file lines from also falling through to logic meant only for later files.
4. **The `FNR==NR {arr[$1]=$2; next} {...}` skeleton is a genuine single-pass join** — one file loaded into memory as a lookup table, the second streamed through once against it in O(1) per lookup. Faster and simpler than `join` or a database for correlation tasks where the lookup table comfortably fits in memory.
5. **Always check `key in array` before trusting a lookup result.** An unmatched key in an unchecked lookup can silently produce a blank or wrong value — explicit membership testing surfaces the "no match" case instead of hiding it.
6. **`ARGV`/`ARGC` expose the command-line file list as inspectable, modifiable data.** Clearing an `ARGV` entry (setting it to `""`) in `BEGIN` removes that file from processing entirely, enabling dynamic input selection from inside the `awk` program itself.
7. **This pattern has real limits: the lookup table must fit in memory, joins are exact-key-only, and more than two files need `FILENAME`-based logic instead of the binary `FNR==NR` test.** Past these edges, a real join tool, a database, or a proper script (Demo 21 §1.6) is the more appropriate choice.

> **Demo scope:** Primary concept: correlating data across multiple files
> in a single awk pass, via NR/FNR/FILENAME and associative-array lookups.
> Supporting concepts: file-boundary detection, the FNR==NR idiom, the
> lookup-table join pattern, ARGV/ARGC inspection and modification,
> practical limits of this approach.
> Estimated completion time: 30 minutes (reading + hands-on + verification).
> Checkpoints: 5 natural stopping points (after Task 1 through Task 5).
> **Verification status:** fully live-verified in this session — every
> command and output shown was actually run, not illustrative.

## Quick Commands Reference

### File-position variables

| Variable | Meaning |
|---|---|
| `NR` | Total record count across all files, never resets |
| `FNR` | Record count within the current file, resets to 1 per file |
| `FILENAME` | Path of the file currently being read |
| `FNR==1` | True at the start of each new file |
| `FNR==NR` | True only while processing the first file |

### Lookup-join skeleton

| Syntax | Purpose |
|---|---|
| `FNR==NR { arr[$1]=$2; next }` | Load the first file into a lookup array |
| `$1 in arr` | Membership test — does this key exist in the array? |
| `{ if ($1 in arr) ... else ... }` | Handle matched and unmatched keys explicitly |

### Command-line arguments

| Syntax | Purpose |
|---|---|
| `ARGC` | Number of command-line arguments (including the program name) |
| `ARGV[i]` | The i-th argument; `ARGV[0]` is conventionally the program name |
| `ARGV[i]=""` (in `BEGIN`) | Remove that file from processing entirely |

## Appendix A — Anki Cards

**`21a-awk-multifile-anki.csv`:**

```
#deck:Linux Mastery::Module 5 - Text Processing & Search Tools::Demo 21a - awk Multi-File Processing
#separator:Comma
#columns:Front,Back,Tags
"awk processes file1.txt (3 lines) then file2.txt (2 lines) in one command. What are NR and FNR at the first line of file2.txt?","NR=4 (it keeps counting across both files, never resetting). FNR=1 (it resets to 1 at the start of every new file).","demo21a,awk,nr-fnr,fundamentals"
"What does FNR==1 detect, and what's it useful for in a multi-file batch?","The start of each new file — FILENAME changes and FNR resets to 1 at that moment. Useful for applying per-file logic, like skipping a header row in EVERY file of a batch, not just the very first line overall.","demo21a,awk,file-boundary"
"Why is FNR==NR true only while awk is processing the FIRST file given to it?","FNR and NR start equal and stay equal as long as every record FNR counts in the first file is also the same record NR counts overall. The moment a second file starts, FNR resets to 1 while NR keeps climbing, so they diverge — permanently false for every file after the first.","demo21a,awk,fnr-nr-idiom,fundamentals"
"What's the standard awk skeleton for a two-file lookup-table join?","FNR==NR { arr[$1] = $2; next } { if ($1 in arr) print arr[$1], $2; else print \"unknown\", $2 } — load the first file into an array while FNR==NR, then look up each line of the second file against it.","demo21a,awk,join,skeleton"
"In the lookup-join pattern, why does the first block need 'next' at the end?","Without next, a line from the first file would fall through to the second, unconditional block as well — next stops that fallthrough, ensuring first-file lines only populate the array and never also run the lookup logic meant for the second file.","demo21a,awk,next,join"
"Why check 'key in array' before using a lookup result, instead of just using arr[key] directly?","An unmatched key in an unchecked lookup can silently produce a blank or incorrect value (awk creates an empty entry on first reference). Explicit 'in' testing surfaces the no-match case so it can be handled deliberately, e.g. printing an explicit 'unknown' marker.","demo21a,awk,lookup,safety"
"What does ARGV[0] conventionally hold in an awk program, and what do ARGV[1], ARGV[2], etc. hold?","ARGV[0] is conventionally the program's own name ('awk'), mirroring $0 in a shell script. ARGV[1] onward are the actual file arguments given on the command line, in order.","demo21a,awk,argv,fundamentals"
"Setting ARGV[2]=\"\" inside a BEGIN block — what effect does this have on processing?","The file originally at that ARGV position is removed from processing entirely — awk skips it as if it had never been named on the command line, even though it was. This enables dynamic input selection from inside the program itself.","demo21a,awk,argv,dynamic-input"
"What's a real practical limit of the awk lookup-table join pattern?","The lookup table (the first file's data) must fit comfortably in memory — for a lookup table that's itself huge, awk's associative array becomes the bottleneck, and a real database or join against pre-sorted disk-based input is the more appropriate tool.","demo21a,awk,limits,cross-demo"
"Does FNR==NR alone scale to distinguishing which of THREE OR MORE files awk is currently reading?","No — FNR==NR only distinguishes 'first file' from 'everything else,' a binary test. Correlating three or more files cleanly typically needs FILENAME-based branching instead of, or in addition to, the FNR==NR idiom.","demo21a,awk,limits,multiple-files"
```

## Appendix B — Quiz

**`21a-awk-multifile-quiz.md`:**

````markdown
# Quiz — Demo 21a: awk Multi-File Processing

> One correct answer per question unless stated otherwise.
> Target: 80% or above before moving to the next demo.

---

**Q1.** `awk '{print NR, FNR}' file1.txt file2.txt` is run, where `file1.txt` has 3 lines and `file2.txt` has 2. What are `NR` and `FNR` on the very first line of `file2.txt`?

- A) `NR=1`, `FNR=1` — both reset for each new file
- B) `NR=4`, `FNR=1` — `NR` keeps counting across files without resetting; `FNR` resets to 1 for each new file
- C) `NR=4`, `FNR=4` — both continue counting across files
- D) `NR=1`, `FNR=4` — `NR` resets per file; `FNR` counts across all files

<details>
<summary>Answer</summary>

**B** — This is the core distinction: `NR` is a running total across every file `awk` reads, never resetting; `FNR` is scoped to the current file and resets to 1 whenever a new file begins.

Trap: A wrongly resets both; C wrongly keeps both climbing; D swaps which variable actually resets and which doesn't.

</details>

---

**Q2.** Why does `FNR==NR` correctly identify "I'm processing the first file" without ever naming that file?

- A) `awk` has a built-in flag specifically for "first file," and `FNR==NR` checks it indirectly
- B) `FNR` and `NR` are equal for exactly as long as every record counted by `FNR` in the first file is also the same record `NR` counts overall — they diverge the moment a second file starts, since `FNR` resets but `NR` doesn't
- C) `FNR==NR` is coincidental and not something `awk` guarantees generally
- D) This only works if both files have the same number of lines

<details>
<summary>Answer</summary>

**B** — This is a direct, guaranteed consequence of how the two variables are defined and maintained, not a coincidence or a special-cased flag — it holds for any first file, regardless of its length or name.

Trap: A invents a nonexistent dedicated mechanism; C wrongly dismisses reliable, well-defined behavior as coincidental; D invents an equal-length requirement that doesn't exist — divergence happens based on which file is being read, not file sizes matching.

</details>

---

**Q3.** In the lookup-join skeleton `FNR==NR { arr[$1]=$2; next } { ... }`, what would happen if `next` were omitted from the first block?

- A) Nothing would change — `next` is purely stylistic here
- B) Lines from the first file would also fall through to the second, unconditional block, running lookup logic meant only for the second file's records
- C) The program would fail to compile without `next`
- D) `next` only affects performance, not correctness

<details>
<summary>Answer</summary>

**B** — Without `next`, execution continues to subsequent blocks after the first one finishes for that record — first-file lines would incorrectly also execute the lookup block intended only for the second file.

Trap: A and D both understate `next`'s real, necessary role here; C invents a compilation requirement that doesn't exist — this is a logic error, not a syntax error.

</details>

---

**Q4.** A lookup-join program uses `if ($1 in arr) print arr[$1] else print "unknown"`. Why check `$1 in arr` instead of just using `arr[$1]` directly?

- A) `arr[$1]` alone would cause a fatal error for a missing key
- B) An unmatched key, if referenced directly without a membership check, can silently produce a blank or incorrect value rather than surfacing the no-match case explicitly
- C) `in` is required syntax — `awk` doesn't allow direct array indexing at all
- D) There's no real difference; both approaches behave identically

<details>
<summary>Answer</summary>

**B** — This is the defensive-check reasoning: explicitly testing membership surfaces a missing key as a deliberate, handled case (printing "unknown") rather than letting an unchecked reference produce an unremarkable blank value that's easy to miss.

Trap: A overstates the consequence — `awk` doesn't fatally error on a missing array key; C invents a syntax restriction that doesn't exist — direct indexing is valid, just less safe here; D dismisses a real, meaningful difference in behavior.

</details>

---

**Q5.** `awk 'BEGIN{for(i=0;i<ARGC;i++) print ARGV[i]}' file1.txt file2.txt` is run. What does `ARGV[0]` conventionally hold?

- A) The first filename, `file1.txt`
- B) The program's own name, `awk` — mirroring `$0` in a shell script; the actual file arguments start at `ARGV[1]`
- C) An empty string, always
- D) The total argument count

<details>
<summary>Answer</summary>

**B** — `ARGV[0]` is the invoking program's name, consistent with how shell scripts treat `$0`; the files given on the command line occupy `ARGV[1]` onward.

Trap: A misreads which index holds the first real file argument; C invents an always-empty value that isn't accurate; D confuses `ARGV[0]` with the unrelated `ARGC` variable.

</details>

---

**Q6.** `awk 'BEGIN{ARGV[2]=""} {print FILENAME}' file1.txt file2.txt` is run. What happens to `file2.txt`?

- A) It's still processed normally; clearing `ARGV[2]` has no effect once execution begins
- B) It's skipped entirely — clearing its `ARGV` entry to an empty string in `BEGIN` removes it from processing, even though it was named on the command line
- C) `awk` throws an error due to the modified `ARGV` array
- D) `file1.txt` is processed twice instead, to compensate

<details>
<summary>Answer</summary>

**B** — Modifying `ARGV` inside `BEGIN`, before the main processing loop starts, genuinely changes which files `awk` reads — an emptied entry is treated as absent.

Trap: A wrongly assumes `ARGV` is fixed once set; C invents an error condition for valid, intentional `ARGV` manipulation; D invents an unrelated compensating behavior that doesn't happen.

</details>

---

**Q7.** A lookup file is genuinely too large to fit comfortably in memory as an `awk` associative array. What does this suggest about the `FNR==NR` join pattern?

- A) The pattern will still work correctly, just somewhat slower
- B) This is a real practical limit of the pattern — the lookup table must fit in memory, and past that scale, a real database or `join` against pre-sorted disk-based input becomes the more appropriate tool
- C) `awk` automatically spills associative arrays to disk when they grow too large, so this isn't actually a concern
- D) Only the second file has a size limit; the lookup file (first file) can be arbitrarily large

<details>
<summary>Answer</summary>

**B** — This is an explicitly acknowledged boundary of the pattern, not a minor performance footnote — an in-memory associative array fundamentally can't scale past available memory, which is exactly when a purpose-built tool becomes the right choice instead.

Trap: A understates a genuine hard limit as a mere slowdown; C invents automatic disk-spilling behavior `awk` doesn't have; D reverses which file (the lookup table, loaded into memory) is actually size-constrained.

</details>

---

**Q8.** Does the `FNR==NR` idiom, by itself, scale cleanly to distinguishing which of four different files `awk` is currently reading?

- A) Yes — `FNR==NR` generalizes automatically to any number of files
- B) No — `FNR==NR` is a binary test (first file vs. not-first-file); distinguishing among three or more files cleanly typically requires `FILENAME`-based branching instead of, or alongside, `FNR==NR`
- C) Yes, but only if all four files have exactly the same number of lines
- D) No — `awk` cannot process more than two files in a single invocation at all

<details>
<summary>Answer</summary>

**B** — `FNR==NR` only ever answers "am I in the first file or not" — it has no way to distinguish the second file from the third or fourth. `FILENAME` comparisons are the tool for genuinely multi-way file distinction.

Trap: A overstates the idiom's scope; C invents an equal-length requirement that isn't relevant to this limitation; D is factually wrong — `awk` processes any number of files given on the command line, the limitation is specifically about `FNR==NR`'s binary nature, not a hard file-count cap.

</details>

---

Score guide:

| Score | Action |
|---|---|
| 7/8 or 8/8 | Import Anki cards, move to the next demo |
| 6/8 | Review the wrong answers, then proceed |
| 5/8 | Re-read the relevant sections, retry those questions |
| Below 5/8 | Re-read the full demo and redo the walkthrough before proceeding |
````