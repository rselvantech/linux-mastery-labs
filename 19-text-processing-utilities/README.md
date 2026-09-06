# Demo 19 — Text Processing Utilities

## Overview

`grep` finds lines; this demo covers the smaller, sharper tools that shape,
rearrange, and summarize text once you've found it. `cut`, `sort`, `uniq`,
`tr`, `wc`, `xargs`, `tee`, `paste`, and `column` are individually simple —
most do one thing — but composed in a pipeline, they cover a huge share of
everyday text-wrangling without needing `sed` or `awk` at all. Demo 18's log
investigation ended with exactly this kind of composition (`sort | uniq
-c`); this demo builds out the full toolkit that pattern draws from.

## Demo Objectives

1. Extract specific fields or character ranges from lines with `cut`
2. Sort text numerically, by field, and in reverse with `sort`
3. Deduplicate and count repeated lines with `uniq`, and why sorting first matters
4. Transform or delete characters with `tr`
5. Count lines, words, and characters with `wc`
6. Convert piped input into command arguments with `xargs`, safely
7. Combine `tee`, `paste`, and `column` for output splitting, merging, and formatting

## Directory Structure

```
19-text-processing-utilities/
└── README.md   # This file — full demo content, including Anki (Appendix A)
                 # and Quiz (Appendix B), extracted later via script.
```

## Environments

| Task | Ubuntu 24.04 | Rocky Linux 9.8 | EC2 Amazon Linux |
|---|---|---|---|
| All lab tasks | ✅ | ✅ | Not required |

## Recall Check — 18-grep

Answer these from memory before continuing — no looking back at Demo 18.

**Q1.** `grep -c 'ERROR' file.log` returns 42; a manual count of every
occurrence gives 47. What explains this, and what command gives the true
occurrence count?

**Q2.** A recursive `grep` prints "Binary file X matches" and runs slowly.
What's the fix?

**Q3.** Is `grep` alone sufficient to find matching lines, extract a field,
and replace that field's value across a file?

<details>
<summary>Answers</summary>

**A1.** `-c` counts matching lines, not total occurrences — a line
containing "ERROR" twice still counts once. `grep -o 'ERROR' file.log | wc
-l` gives the true occurrence count.

**A2.** Scope the search with `--exclude`/`--exclude-dir` to skip
binary/compressed files or irrelevant directories.

**A3.** No — `grep` has no replace/transform capability at all; that's
`sed`'s job.

</details>

## Part 1 — Concepts

### 1.1 cut — extracting fields or character ranges

```bash
cut -d',' -f2 users.csv
# -d',': field delimiter is a comma
# -f2: extract field 2 only

cut -d',' -f1,3 users.csv
# multiple fields, comma-separated field list

cut -c1-10 file.txt
# -c: character positions, not fields — first 10 characters of each line,
# regardless of any delimiter
```

**`cut` is delimiter-based, not whitespace-smart by default.** Given a line
with irregular spacing (multiple spaces between fields, common in command
output like `ps` or `df`), `cut -d' '` treats every single space as a
separate field boundary — producing a mess of empty fields rather than
sensibly collapsing repeated whitespace. This is exactly the situation
where `awk` (Demo 21), which handles repeated whitespace as a single field
separator by default, is the better tool — `cut` is the right choice for
genuinely delimiter-consistent data (CSV, `/etc/passwd`-style colon-
separated files), not free-form command output.

### 1.2 sort — beyond alphabetical

```bash
sort names.txt                  # alphabetical, default
sort -n numbers.txt             # -n: numeric sort — "10" sorts after "9",
                                 # not before it as plain alphabetical would
sort -r names.txt               # -r: reverse
sort -k2 -t',' users.csv        # -k2: sort by field 2, -t',': field
                                 # separator is a comma
sort -k2,2n -t',' users.csv     # -k2,2n: sort by field 2 specifically,
                                 # numerically (n suffix on the key spec)
```

**Alphabetical sort on numbers produces a wrong-looking but technically
correct result** without `-n`: `"10"` sorts before `"9"` because sort
compares character by character, and `'1'` is a lower character than
`'9'` — this is one of the most common "sort looks broken" moments, and
it isn't broken, it's just doing lexical rather than numeric comparison as
requested.

```bash
sort -u names.txt
# -u: unique — sort AND deduplicate in one step, rather than piping to a
# separate uniq call (though see §1.3 for when uniq alone is still needed).
```

### 1.3 uniq — and why sorting first is not optional

```bash
uniq names.txt
```

**`uniq` only removes *adjacent* duplicate lines — it has no concept of
"duplicate anywhere in the file."** This means `uniq` on unsorted input
routinely fails to deduplicate anything at all, if the duplicate lines
aren't next to each other:

```bash
printf 'apple\nbanana\napple\n' | uniq
# apple
# banana
# apple
# All three lines survive — "apple" appears twice but not adjacently, so
# uniq sees no duplicates to remove.

printf 'apple\nbanana\napple\n' | sort | uniq
# apple
# apple
# banana
# Sorting first makes duplicates adjacent, which is what uniq actually needs.
```

```bash
sort names.txt | uniq -c
# -c: prefix each unique line with its count — this is exactly the pattern
# Demo 18 used to break down log lines by backend, and one of the single
# most useful pipeline idioms in this whole demo.

sort names.txt | uniq -d
# -d: show ONLY lines that had duplicates (not every unique line) — useful
# when you specifically want to find what's duplicated, not the full
# deduplicated list.
```

### 1.4 tr — character transformation, not field-aware

```bash
echo "Hello World" | tr 'a-z' 'A-Z'
# HELLO WORLD — translates each character in the first set to the
# corresponding character in the second set, position by position

echo "one,two,three" | tr ',' '\n'
# one
# two
# three
# Common use: convert a delimiter into newlines, turning a single
# delimited line into multiple lines for further per-line processing.

echo "  extra   spaces  " | tr -s ' '
# -s: squeeze — collapse repeated consecutive characters into one, useful
# for normalizing irregular whitespace before other tools process it.

echo "hello123world" | tr -d '0-9'
# -d: delete — remove all characters in the given set entirely
```

`tr` operates on a stream of individual characters, with no concept of
lines, fields, or delimiters beyond what you explicitly translate — it's
the right tool for character-level transformation, not the tool for
anything field- or line-structure-aware.

### 1.5 wc — counting lines, words, characters

```bash
wc -l file.txt      # lines
wc -w file.txt       # words
wc -c file.txt       # bytes
wc -m file.txt       # characters (differs from -c with multi-byte/UTF-8 text)
```

`wc -l` piped from another command is one of the most common idioms in
shell scripting for turning "how many results" into a number a script can
branch on:

```bash
if [ "$(grep -c 'ERROR' app.log)" -gt 100 ]; then
    echo "High error volume"
fi
```

### 1.6 xargs — turning piped input into command arguments

Many commands don't read from stdin at all — they expect arguments on the
command line. `xargs` bridges that gap:

```bash
find /tmp/lab -name '*.tmp' | xargs rm
# find produces a list of matching paths on stdout; rm doesn't read stdin —
# xargs converts each line of input into an argument to rm.
```

**Unquoted filenames piped into plain `xargs` break on filenames containing
spaces** — a filename like `my file.tmp` gets split into two separate
arguments (`my` and `file.tmp`), neither of which is the actual file. The
safe pattern pairs a null-delimited producer with `xargs -0`:

```bash
find /tmp/lab -name '*.tmp' -print0 | xargs -0 rm
# -print0 (from find) and -0 (xargs) use a null byte as the separator
# instead of newline/whitespace — null bytes can't legally appear in a
# filename, making this the only fully safe delimiter for arbitrary
# filenames, including ones with spaces or even embedded newlines.
```

```bash
echo "file1.txt file2.txt file3.txt" | xargs -n 1 echo "Processing:"
# -n 1: pass arguments one at a time to the command, rather than all at
# once — useful when the target command doesn't accept multiple arguments
# gracefully, or when you want one invocation per item for clearer output.

find /tmp/lab -name '*.log' | xargs -I {} mv {} {}.bak
# -I {}: define a placeholder token, letting the input be positioned
# anywhere in the constructed command, not just appended at the end.
```

### 1.7 tee, paste, column — splitting, merging, formatting output

```bash
grep 'ERROR' app.log | tee errors.txt | wc -l
# tee writes its input to a file AND passes it through unchanged to stdout
# — here, saving the matched lines to errors.txt while still counting them
# in the same pipeline, rather than running the same grep command twice.
```

```bash
paste file1.txt file2.txt
# Merges corresponding lines from multiple files side by side, tab-separated
# by default — the inverse operation of cut, in a sense: cut extracts a
# column, paste combines separate files into columns.

paste -d',' file1.txt file2.txt
# -d',': use a comma instead of the default tab as the joining delimiter
```

```bash
printf 'name,age,city\nAlice,30,NYC\nBob,25,LA\n' | column -t -s','
# -t: format as an aligned table. -s',': the input is comma-separated.
# Turns delimited text into a readable, aligned table — purely a display
# convenience, not a data transformation the way cut/sort/uniq are.
```

## Part 2 — Hands-On Lab

### Commands used in this demo

| Command | Flag meaning | Purpose |
|---|---|---|
| `cut -d',' -f2` | `-d`: delimiter, `-f`: field | Extract a field from delimited text |
| `sort -n` / `-k` / `-r` | numeric / key / reverse | Sort by different criteria |
| `uniq -c` / `-d` | count / duplicates-only | Deduplicate adjacent lines |
| `tr 'a-z' 'A-Z'` / `-s` / `-d` | translate / squeeze / delete | Character-level transformation |
| `wc -l` / `-w` / `-c` | lines / words / bytes | Count |
| `xargs -0` / `-n` / `-I {}` | null-safe / batch size / placeholder | Convert stdin into command arguments |
| `tee FILE` | — | Save output to a file while passing it through |
| `paste -d','` | delimiter | Merge files side by side |
| `column -t -s','` | table / separator | Format delimited text as an aligned table |

### Task 1 — cut on genuinely delimited data

```bash
printf 'alice,30,nyc\nbob,25,la\ncarol,35,sf\n' > /tmp/users.csv
cut -d',' -f1 /tmp/users.csv
cut -d',' -f1,3 /tmp/users.csv
```

### Task 2 — sort: alphabetical vs. numeric, and by field

```bash
printf '9\n10\n2\n' > /tmp/nums.txt
sort /tmp/nums.txt        # observe "10" sorting before "2" — lexical, not numeric
sort -n /tmp/nums.txt      # now correctly numeric

sort -t',' -k2,2n /tmp/users.csv
```

### Task 3 — uniq requires sorted input

```bash
printf 'apple\nbanana\napple\n' > /tmp/fruit.txt
uniq /tmp/fruit.txt              # apple survives twice — not adjacent
sort /tmp/fruit.txt | uniq        # correctly deduplicated
sort /tmp/fruit.txt | uniq -c     # with counts
```

### Task 4 — tr for case conversion and whitespace squeezing

```bash
echo "Hello World" | tr 'a-z' 'A-Z'
echo "field1   field2     field3" | tr -s ' '
```

### Task 5 — xargs, unsafe then safe

```bash
mkdir -p /tmp/lab-files
touch /tmp/lab-files/normal.tmp
touch "/tmp/lab-files/has space.tmp"

find /tmp/lab-files -name '*.tmp' | xargs -n1 echo "unsafe would process:"
# Observe "has" and "space.tmp" reported as two separate items — the bug.

find /tmp/lab-files -name '*.tmp' -print0 | xargs -0 -n1 echo "safe processes:"
# Observe "has space.tmp" correctly treated as one item.
rm -rf /tmp/lab-files
```

### Task 6 — tee, paste, and column composed

```bash
printf 'alice\nbob\ncarol\n' > /tmp/names.txt
printf '30\n25\n35\n' > /tmp/ages.txt

paste -d',' /tmp/names.txt /tmp/ages.txt | tee /tmp/combined.csv
column -t -s',' /tmp/combined.csv
```

## What You Learned

1. ✅ Extracted specific fields or character ranges from lines with `cut`
2. ✅ Sorted text numerically, by field, and in reverse with `sort`
3. ✅ Deduplicated and counted repeated lines with `uniq`, understanding why sorting first matters
4. ✅ Transformed and squeezed characters with `tr`
5. ✅ Counted lines, words, and characters with `wc`
6. ✅ Converted piped input into command arguments safely with `xargs -0`
7. ✅ Combined `tee`, `paste`, and `column` for splitting, merging, and formatting output

## Part 3 — Break-Fix Scenario

### Scenario — NovaTech Ops: two text-processing incidents

> **Incident report — NovaTech Ops, ticket #4650**
>
> Alert 1: "a cleanup script using `find | xargs rm` deleted the wrong
> files after a deploy that introduced spaces in generated filenames."
> Alert 2: "`sort names.txt | uniq -c` shows a name appearing 3 times when
> it should show 5 — undercounting."

**Incident 1 — xargs deleted the wrong files after filenames got spaces in them**

```bash
$ find /tmp/generated -name '*.tmp' | xargs rm
$ # deleted files with unexpected names, some intended files survived
```

Tasks:
1. Reference §1.6 — what's the specific mechanism by which unquoted `xargs`
   mishandles filenames containing spaces?
2. Rewrite this command using the safe pattern from §1.6.
3. Why does this bug only surface after a deploy changes filename
   generation, rather than being caught earlier — what does that suggest
   about testing pipelines like this?

**Incident 2 — sort | uniq -c undercounts**

```bash
$ grep 'wadmin' /var/log/auth.log | sort | uniq -c
      3 Jul 25 09:00:01 host sshd: Accepted publickey for wadmin
      2 Jul 25 09:15:03 host sshd: Accepted publickey for wadmin
$ # teammate expected one line showing count 5, not two lines showing 3 and 2
```

Tasks:
1. Reference §1.2/§1.3 — `sort` was used before `uniq`, so why are there
   two separate count groups instead of one combined count of 5?
2. What's actually different between the lines in each of the two groups,
   given that `uniq` only merges lines that are *exactly* identical?
3. What would need to change about the input (not the `sort | uniq -c`
   command itself) to get the single combined count of 5 the teammate
   expected?

<details>
<summary>Answers</summary>

**Incident 1:** Unquoted `xargs` (with the default newline/whitespace
delimiter) splits input on any whitespace, including spaces *within* a
single filename — a file named `deploy report.tmp` gets split into two
separate arguments, `deploy` and `report.tmp`, neither of which names an
actual file; `rm` then either errors on nonexistent files or, worse, matches
something unintended if a file happens to share one of those fragment
names. Safe rewrite: `find /tmp/generated -name '*.tmp' -print0 | xargs -0
rm` — null-delimited input can't be confused by spaces, since null bytes
can't legally appear in a filename at all. This bug surfaces specifically
after a deploy changes filename generation because the original filenames
(without spaces) happened to work fine with the unsafe pattern — the
underlying fragility was always there, just never triggered until input
that actually exercised it appeared; this is exactly the kind of thing that
argues for using the null-safe pattern by default in scripts, rather than
only after being burned once, since "works in testing" doesn't mean "safe
for all future filenames."

**Incident 2:** `uniq` only merges lines that are byte-for-byte identical
— per this scenario, the two log lines have different timestamps
(`09:00:01` vs `09:15:03`), which makes them different lines entirely from
`uniq`'s perspective, even though the underlying event ("wadmin logged in
via publickey") is conceptually the same. `sort | uniq -c` is working
exactly as designed; the mismatch is between what the teammate wanted
(count of a conceptual event, ignoring the timestamp) and what the command
actually does (count of byte-identical lines, timestamp included). To get a
single combined count of 5, the timestamp would need to be stripped out
*before* the `sort | uniq -c` step — e.g. `cut`-ing or `sed`-ing off the
timestamp field first, so the remaining text (the actually-repeating part)
is what gets deduplicated and counted, not the whole original line.

</details>

## Interview Prep

**Q1. `find | xargs rm` is used to clean up temp files, and it occasionally deletes the wrong thing. What's the most likely cause, and the fix?**
Unquoted/default-delimited `xargs` splits input on whitespace, including
spaces within a single filename — a filename containing a space gets
treated as two separate arguments, neither of which is the actual
filename. The fix is the null-delimited pattern: `find ... -print0 | xargs
-0 rm`, since null bytes can't appear in filenames at all, making it the
only delimiter that's safe for genuinely arbitrary filenames.

**Q2. `sort file.txt | uniq -c` is expected to combine several conceptually-duplicate lines into one count, but instead produces several separate counts. What's the most likely explanation?**
`uniq` only merges lines that are exactly byte-identical — if the lines
differ in some way that doesn't matter for the intended comparison (a
timestamp, a variable ID, trailing whitespace), `uniq` correctly treats
them as different lines, even though they represent the "same" thing
conceptually. The fix is normalizing the input before the `sort | uniq -c`
step — stripping or ignoring the varying part — not changing the `sort |
uniq -c` command itself, which is working correctly against what it's
actually given.

**Q3. Why does `cut -d' '` often produce a mess when applied to typical command output like `ps` or `df`, and what's the better tool for that case?**
`cut` treats every occurrence of the delimiter as a field boundary,
including consecutive/repeated ones — output with irregular spacing (common
in fixed-width command output) produces many spurious empty fields rather
than sensibly collapsing repeated whitespace into one separator. `awk`
handles repeated whitespace as a single field separator by default, making
it the better tool for genuinely whitespace-separated, irregularly-spaced
output; `cut` remains the right choice for consistently-delimited data
like CSV.

## Key Takeaways

1. **`uniq` only removes *adjacent* duplicates — sorting first is not
   optional, it's required for `uniq` to work at all on typical unsorted
   input.** `sort | uniq` (or `sort | uniq -c`) is close to a fixed idiom,
   not two independently-optional steps.
2. **`uniq` merges only byte-identical lines.** A varying timestamp or ID
   within otherwise-"duplicate" lines means `uniq` correctly sees them as
   different — normalize the input (strip the varying part) before
   deduplicating if that's not the intended comparison.
3. **`xargs` with default delimiters breaks on filenames/arguments
   containing spaces.** The `find -print0 | xargs -0` pattern is the only
   fully safe approach for arbitrary filenames, and this should be a
   default habit in scripts, not a fix applied only after being burned.
4. **`sort` without `-n` compares lexically, not numerically** — `"10"`
   sorting before `"2"` is correct lexical behavior, not a bug, and is one
   of the most common "why is sort broken" moments.
5. **`cut` is delimiter-based and not whitespace-smart; `awk` handles
   irregular whitespace as a single separator by default.** Reach for
   `cut` on genuinely consistent delimiters (CSV, colon-separated files),
   and `awk` for command output with variable spacing.
6. **`tee`, `paste`, and `column` solve three different composition
   problems**: `tee` splits one stream into a saved file and continued
   pipeline output; `paste` merges separate files into columns; `column`
   is purely a display formatter, not a data transformation.

> **Demo scope:** Primary concept: composable single-purpose text
> processing utilities. Supporting concepts: cut, sort, uniq, tr, wc,
> xargs (including null-safety), tee, paste, column.
> Estimated completion time: 32 minutes (reading + hands-on + verification).
> Checkpoints: 6 natural stopping points (after Task 1 through Task 6).

## Quick Commands Reference

### Field and character extraction

| Command | Purpose |
|---|---|
| `cut -d',' -f2` | Extract field 2 from comma-delimited text |
| `cut -c1-10` | Extract characters 1–10 of each line |

### Sorting and deduplication

| Command | Purpose |
|---|---|
| `sort -n` | Numeric sort |
| `sort -k2,2n -t','` | Sort by field 2, numerically, comma-delimited |
| `sort -u` | Sort and deduplicate in one step |
| `sort file \| uniq -c` | Count occurrences of each unique line (sort first!) |
| `sort file \| uniq -d` | Show only lines that had duplicates |

### Character transformation and counting

| Command | Purpose |
|---|---|
| `tr 'a-z' 'A-Z'` | Translate characters |
| `tr -s ' '` | Squeeze repeated characters into one |
| `tr -d '0-9'` | Delete characters in a set |
| `wc -l` / `-w` / `-c` | Count lines / words / bytes |

### Arguments, splitting, merging, formatting

| Command | Purpose |
|---|---|
| `find ... -print0 \| xargs -0 CMD` | Null-safe: convert stdin into arguments |
| `xargs -n1` | One argument per invocation |
| `xargs -I {}` | Placeholder for positioning input in the command |
| `CMD \| tee FILE` | Save output to a file while passing it through |
| `paste -d',' f1 f2` | Merge files side by side |
| `column -t -s','` | Format delimited text as an aligned table |

## Appendix A — Anki Cards

**Note:** extracted into a standalone `.csv` by the series extraction script.

```
#deck:Linux Mastery::Module 4 - Text Processing & Scripting for DevOps::19-text-processing-utilities
#separator:Comma
#columns:Front,Back,Tags
"uniq is run on an unsorted file containing 'apple, banana, apple' and doesn't remove the duplicate apple. Why?","uniq only removes ADJACENT duplicate lines, not duplicates anywhere in the file. Since the two 'apple' lines aren't next to each other, uniq sees no duplicates to remove — sort first to make duplicates adjacent.","demo19,uniq,common-mistake"
"sort file.txt (without -n) puts '10' before '2'. Is this a bug?","No — without -n, sort compares lexically (character by character), and '1' is a lower character than '2', so '10' correctly sorts before '2' alphabetically. -n enables numeric comparison instead.","demo19,sort,common-mistake"
"find /tmp -name '*.tmp' | xargs rm deletes the wrong files when a filename contains a space. Why, and what's the fix?","Default xargs splits input on whitespace, so a filename like 'my file.tmp' becomes two separate arguments neither of which is the real file. Fix: find ... -print0 | xargs -0 rm — null-byte delimiting is safe because null can't appear in a filename.","demo19,xargs,safety,common-mistake"
"sort file | uniq -c produces two separate counts for lines that are conceptually the same event but include different timestamps. Is uniq malfunctioning?","No — uniq only merges lines that are exactly byte-identical. Lines differing by timestamp are genuinely different lines to uniq. Fix: strip/normalize the varying part (e.g. with cut or sed) before the sort | uniq -c step.","demo19,uniq,common-mistake"
"Why is cut often a poor fit for parsing typical command output like ps or df, compared to awk?","cut is delimiter-based and treats every occurrence of the delimiter (e.g. every space) as a field boundary, including repeated ones — irregular spacing produces many spurious empty fields. awk collapses repeated whitespace into a single separator by default.","demo19,cut,awk,comparison"
"What does tee actually do in a pipeline like grep 'ERROR' app.log | tee errors.txt | wc -l?","It writes its input to a file (errors.txt) AND passes it through unchanged to the next command in the pipeline (wc -l) — saving output to disk while continuing to process it in the same pipeline, rather than running the command twice.","demo19,tee,fundamentals"
"What's the difference between what cut and paste each do to columnar data?","cut extracts a column/field FROM a single source; paste is roughly the inverse — it merges corresponding lines from multiple separate files INTO columns, side by side.","demo19,cut,paste,comparison"
"What does xargs -I {} enable that plain xargs (appending arguments at the end) doesn't?","A placeholder token that lets the piped input be positioned anywhere in the constructed command, not just appended at the end — useful for commands like mv that need the input to appear in multiple places or a specific position.","demo19,xargs,fundamentals"
"Is column -t -s',' a data transformation tool like cut or sort?","No — column is purely a display formatter. It arranges delimited text into an aligned, readable table but doesn't change, filter, or reorder the underlying data the way cut/sort/uniq do.","demo19,column,fundamentals"
"What's the practical idiom for turning grep's matched-line output into a count of unique values in a specific field?","grep 'pattern' file | grep -o 'field=value-pattern' | sort | uniq -c — extract the field, then sort (required for uniq to work) and count with uniq -c.","demo19,workflow,cross-demo"
```

## Appendix B — Quiz

**Note:** extracted into a standalone `.md` by the series extraction script.

> One correct answer per question unless stated otherwise.
> Target: 80% or above before moving to Demo 20.

**Q1. `uniq` is run on an unsorted file with "apple" appearing twice, non-adjacently. What happens?**

- A) `uniq` still removes the duplicate, since it scans the whole file
- B) `uniq` does not remove it — it only removes adjacent duplicate lines
- C) `uniq` throws an error on unsorted input
- D) `uniq` automatically sorts the input first

<details>
<summary>Answer</summary>

**B** — This is `uniq`'s actual, documented behavior — adjacency is required, which is why `sort | uniq` is close to a fixed idiom. Trap: A and D both invent smarter behavior `uniq` doesn't have; C invents an error condition that doesn't occur.

</details>

---

**Q2. `sort file.txt` (no flags) places "10" before "2". Is this a bug?**

- A) Yes — sort should always understand numbers correctly
- B) No — without `-n`, sort compares lexically, and "1" is a lower character than "2"
- C) Yes, but only on Ubuntu
- D) No, because `sort` always ignores numbers

<details>
<summary>Answer</summary>

**B** — This is expected lexical sort behavior; `-n` is required for numeric comparison. Trap: A assumes numeric-aware sorting is the default; C invents a distro-specific bug; D misdescribes what's actually happening.

</details>

---

**Q3. `find /tmp -name '*.tmp' | xargs rm` occasionally deletes the wrong file. What's the most likely cause?**

- A) `xargs` has a memory limit that's being exceeded
- B) A filename contains a space, and default `xargs` splits on whitespace, breaking one filename into multiple arguments
- C) `find` returned files in the wrong order
- D) `rm` requires `-f` to work correctly with piped input

<details>
<summary>Answer</summary>

**B** — This is the classic `xargs` whitespace-splitting bug; the fix is the null-delimited `-print0`/`-0` pattern. Trap: A, C, and D all invent unrelated explanations not supported by the actual mechanism.

</details>

---

**Q4. `sort file | uniq -c` produces two separate counts for lines that represent the same conceptual event but have different timestamps. Is this a bug in `uniq`?**

- A) Yes — `uniq` should ignore timestamps automatically
- B) No — `uniq` only merges byte-identical lines; lines differing by timestamp are genuinely different to it
- C) Yes, unless `-i` is used for case-insensitivity
- D) No, but only because the file wasn't sorted correctly

<details>
<summary>Answer</summary>

**B** — `uniq` is working exactly as designed; the fix is normalizing/stripping the varying part before deduplicating, not treating this as a `uniq` defect. Trap: A assumes smart semantic matching that `uniq` doesn't do; C invents an unrelated flag; D wrongly blames sorting when the actual lines genuinely differ.

</details>

---

**Q5. Why is `cut -d' '` often a poor choice for parsing output like `ps` or `df`, compared to `awk`?**

- A) `cut` cannot handle spaces as a delimiter at all
- B) `cut` treats every occurrence of the delimiter as a boundary, including repeated ones, producing many empty fields on irregularly-spaced output
- C) `cut` is significantly slower than `awk` for all inputs
- D) `cut` requires root permissions for space-delimited parsing

<details>
<summary>Answer</summary>

**B** — This is the actual, practical reason; `awk` collapses repeated whitespace by default, which `cut` does not. Trap: A is factually wrong — `cut` can use space as a delimiter, just not smartly; C and D invent unrelated constraints.

</details>

---

**Q6. What does `tee` do in a pipeline like `grep ERROR app.log | tee errors.txt | wc -l`?**

- A) It only writes to the file and produces no further output
- B) It writes input to a file AND passes it through unchanged to the next command
- C) It duplicates the file on disk into two copies
- D) It converts the pipeline's output to uppercase

<details>
<summary>Answer</summary>

**B** — `tee` splits its input to both a file and continued pipeline output, avoiding the need to run the same command twice. Trap: A, C, and D all misdescribe its actual dual-output behavior.

</details>

---

**Q7. What's the relationship between what `cut` and `paste` each do?**

- A) They are functionally identical
- B) `cut` extracts a column from one source; `paste` merges separate files into columns — roughly inverse operations
- C) `paste` is a newer alias for `cut`
- D) `cut` works on files; `paste` only works on piped input

<details>
<summary>Answer</summary>

**B** — This inverse relationship is a useful way to remember what each does. Trap: A and C both wrongly claim equivalence; D invents an input-source restriction that doesn't exist.

</details>

---

**Q8. Is `column -t -s','` a data transformation, the way `sort` or `uniq` are?**

- A) Yes, it also deduplicates while formatting
- B) No — it's purely a display formatter, arranging delimited text into an aligned table without changing the underlying data
- C) Yes, it sorts the data alphabetically as part of formatting
- D) No, and it also strips all delimiters from the output permanently

<details>
<summary>Answer</summary>

**B** — `column` only affects visual presentation; it doesn't filter, reorder, or deduplicate. Trap: A and C both invent transformation behavior `column` doesn't perform; D overstates what happens to the underlying data — it's a display-time formatting choice, not a destructive change to the source.

</details>