# Demo 21 — awk

## Overview

`grep` finds, `sed` transforms line-by-line — `awk` is the tool that thinks
in fields and can do arithmetic, conditionals, and aggregation across an
entire file in one pass. This is the tool Demo 18's log investigation
gestured toward with `sort | uniq -c` — `awk` can do that kind of
field-aware summarization directly, often replacing a multi-command
pipeline with one `awk` program. By the end, you'll read and write `awk`
one-liners confidently and know when its field/record model is genuinely
the right fit.

## Demo Objectives

1. Explain `awk`'s field/record model and access fields by position
2. Write pattern-action pairs, including patterns with no explicit action
3. Use `BEGIN`/`END` blocks for setup and summary logic
4. Use built-in variables (`NR`, `NF`, `FS`, `OFS`) to control processing
5. Use associative arrays to aggregate and count by key
6. Recognize when a task has outgrown awk and belongs in a full script instead

## Directory Structure

```
21-awk/
└── README.md   # This file — full demo content, including Anki (Appendix A)
                 # and Quiz (Appendix B), extracted later via script.
```

## Environments

| Task | Ubuntu 24.04 | Rocky Linux 9.8 | EC2 Amazon Linux |
|---|---|---|---|
| All lab tasks | ✅ (gawk) | ✅ (gawk) | Not required |

## Recall Check — 20-sed

Answer these from memory before continuing — no looking back at Demo 20.

**Q1.** `sed 's/a/b/'` on input `"aaa"` produces `"baa"`, not `"bbb"`. Why?

**Q2.** `sed '5p' file.txt` (no `-n`) shows line 5 twice, while every other
line shows once. Why?

**Q3.** Why is `sed -i` with no backup suffix considered genuinely risky?

<details>
<summary>Answers</summary>

**A1.** `s///` replaces only the first match per line by default — `g` is
needed to replace every match on the line.

**A2.** `sed` prints every line by default; `p` adds an explicit print on
top of that rather than replacing it — without `-n` to suppress the
default, the addressed line gets both.

**A3.** It's irreversible — the original content is overwritten with no
`sed`-level undo available, unlike piped `sed` which only affects stdout.

</details>

## Part 1 — Concepts

### 1.1 The field/record model

`awk` processes input one **record** (by default, one line) at a time,
automatically splitting each record into **fields** on whitespace:

```bash
printf 'alice 30 nyc\nbob 25 la\n' | awk '{print $1}'
# alice
# bob
# $1, $2, $3... reference fields by position. $0 refers to the WHOLE
# record (the entire line), not a field.
```

**This whitespace-splitting is automatic and collapses repeated
whitespace** — directly solving the Demo 19 problem where `cut -d' '`
produced spurious empty fields on irregularly-spaced output:

```bash
ps aux | awk '{print $2, $11}'
# PID and COMMAND columns from ps output, correctly extracted despite ps's
# irregular column spacing — cut -d' ' would have mangled this badly.
```

```bash
awk -F',' '{print $2}' users.csv
# -F',': set the field separator to comma instead of default whitespace —
# now awk splits CSV-style, not on spaces.
```

### 1.2 Pattern-action pairs

An `awk` program is a series of `pattern { action }` pairs. Either half can
be omitted, with a specific meaning each time:

```bash
awk '{print}' file.txt
# No pattern = matches every line. print with no args = print $0 (whole line).
# Equivalent to cat.

awk '/ERROR/' file.txt
# Pattern with NO action = default action is print. Equivalent to grep 'ERROR'.

awk '$3 > 100 {print $1}' file.txt
# Pattern is a field comparison (not a regex) — matches lines where the
# 3rd field, treated as a number, exceeds 100. Action prints field 1 for
# those lines only.

awk '/ERROR/ {print NR": "$0}' file.txt
# Pattern AND action together — for lines matching /ERROR/, print the line
# number (NR, see §1.4) and the full line.
```

**A pattern with no action defaulting to "print the whole line" is exactly
what makes `awk '/pattern/'` a drop-in `grep` replacement** for simple
cases — worth recognizing, though `grep` remains the more idiomatic choice
when field logic isn't needed at all.

### 1.3 BEGIN and END — setup and summary

```bash
awk 'BEGIN {print "Report starting"} {print} END {print "Report complete"}' file.txt
```

`BEGIN` runs once, before any input is read — the right place for
initialization (setting a field separator, printing a header, initializing
a counter to a specific value other than 0). `END` runs once, after all
input has been processed — the right place for a summary that needs the
*complete* picture, not something computable line by line:

```bash
awk '{sum += $3} END {print "Total:", sum}' sales.csv
# Accumulates $3 across every line during normal processing (no explicit
# pattern = every line), then prints the final total only once, after all
# lines are done — this couldn't be expressed correctly as a per-line
# action alone, since "the total" only exists once every line has
# contributed.
```

### 1.4 Built-in variables — NR, NF, FS, OFS

```bash
awk '{print NR, $0}' file.txt
# NR: current record (line) number — a running counter, automatically
# maintained, no manual counting needed.

awk '{print NF, $0}' file.txt
# NF: number of fields in the CURRENT line — varies per line if lines have
# different field counts, useful for finding malformed/short rows.

awk 'NF != 5 {print NR": expected 5 fields, got "NF}' data.txt
# Combining NR and NF: flag exactly which lines don't match an expected
# field count, and which line number they're on — a genuinely practical
# data-validation one-liner.
```

```bash
awk -F',' 'BEGIN{OFS="|"} {print $1, $2}' users.csv
# FS (input field separator, set here via -F) and OFS (output field
# separator) are independent — you can read comma-separated input and
# print pipe-separated output in the same command, since print with
# comma-separated arguments joins them using OFS, not FS.
```

### 1.5 Associative arrays — aggregate and count by key

`awk` arrays are associative (keyed by string, not just numeric index) by
default — this is what makes `awk` genuinely powerful for aggregation:

```bash
awk '{count[$1]++} END {for (key in count) print key, count[key]}' access.log
# For every line, use field 1 (e.g. an IP address or backend name) as an
# array key and increment its counter. After all lines, iterate the array
# and print each key with its final count — this is sort | uniq -c's
# entire job, done in a single pass, with the added flexibility of using
# ANY field as the grouping key, not just the whole line.
```

```bash
awk '{sum[$1] += $3} END {for (key in sum) print key, sum[key]}' sales.csv
# Same shape, but summing a numeric field per key instead of just counting
# occurrences — something sort | uniq -c has no way to express at all,
# since it only counts identical lines, never sums a different field.
```

**This single pattern — key on one field, aggregate (count or sum) another
— covers a large share of real log/data summarization tasks**, and it's
the clearest illustration of why `awk` earns its place over a longer
`grep`/`sort`/`uniq` pipeline once the task needs anything beyond simple
counting of whole matching lines.

### 1.6 When a task has outgrown awk

`awk` is genuinely a small programming language — loops, conditionals,
functions, arrays — and it's tempting to keep extending an `awk` one-liner
indefinitely. The practical line: once a transformation needs external
program calls, complex control flow across many conditions, robust error
handling, or state that spans multiple distinct files in a non-trivial way,
a proper shell script (Demo 23/24) — possibly calling `awk` for the parts
it's genuinely good at — is more maintainable than one increasingly dense
`awk` program. `awk` shines at "process this stream of records, field by
field, with some aggregation" — it's not meant to replace general-purpose
scripting once the logic grows past that shape.

## Part 2 — Hands-On Lab

### Commands used in this demo

| Command | Flag/syntax | Purpose |
|---|---|---|
| `awk '{print $N}'` | `$N`: field N | Extract a field by position |
| `awk -F','` | `-F`: input field separator | Split on a custom delimiter |
| `awk '/PAT/'` | pattern, no action | Grep-equivalent filtering |
| `awk 'BEGIN{...} {...} END{...}'` | — | Setup, per-line, and summary blocks |
| `NR` / `NF` | built-in variables | Record number / field count |
| `awk '{arr[$1]++} END{for (k in arr) print k, arr[k]}'` | associative array | Aggregate and count by key |

### Task 1 — Fields vs. cut on irregular whitespace

```bash
ps aux | head -5 > /tmp/ps-sample.txt
cat /tmp/ps-sample.txt
awk '{print $2, $11}' /tmp/ps-sample.txt
cut -d' ' -f2 /tmp/ps-sample.txt
# Compare the cut output (likely mangled by irregular spacing) against
# awk's clean field extraction.
```

### Task 2 — Pattern-action pairs

```bash
printf 'INFO starting\nERROR disk full\nINFO running\nERROR oom\n' > /tmp/awk-lab.txt
awk '/ERROR/' /tmp/awk-lab.txt
awk '/ERROR/ {print NR": "$0}' /tmp/awk-lab.txt
```

### Task 3 — BEGIN/END for a running total

```bash
printf 'alice,150\nbob,90\ncarol,220\n' > /tmp/sales.csv
awk -F',' 'BEGIN{print "Processing sales..."} {sum += $2} END{print "Total:", sum}' /tmp/sales.csv
```

### Task 4 — NR/NF for data validation

```bash
printf 'a,b,c\nd,e\nf,g,h,i\n' > /tmp/awk-lab2.csv
awk -F',' 'NF != 3 {print NR": expected 3 fields, got "NF}' /tmp/awk-lab2.csv
```

### Task 5 — Associative array aggregation

```bash
printf 'app-01 ERROR\napp-02 ERROR\napp-01 ERROR\napp-03 INFO\napp-01 ERROR\n' > /tmp/awk-lab3.txt
awk '$2=="ERROR" {count[$1]++} END {for (k in count) print k, count[k]}' /tmp/awk-lab3.txt
# Compare mentally against the Demo 18/19 equivalent:
grep 'ERROR' /tmp/awk-lab3.txt | awk '{print $1}' | sort | uniq -c
```

### Task 6 — FS/OFS together

```bash
printf 'alice,30,nyc\nbob,25,la\n' > /tmp/awk-lab4.csv
awk -F',' 'BEGIN{OFS="|"} {print $1, $3}' /tmp/awk-lab4.csv
```

## What You Learned

1. ✅ Explained `awk`'s field/record model and extracted fields by position
2. ✅ Wrote pattern-action pairs, including pattern-only and action-only forms
3. ✅ Used `BEGIN`/`END` blocks for setup and summary logic
4. ✅ Used `NR`, `NF`, `FS`, `OFS` to control and validate processing
5. ✅ Used associative arrays to aggregate and count by key in one pass
6. ✅ Recognized when a task has outgrown awk and belongs in a shell script

## Part 3 — Break-Fix Scenario

### Scenario — NovaTech Ops: two awk incidents

> **Incident report — NovaTech Ops, ticket #4683**
>
> Alert 1: "an `awk` field-extraction script that worked on staging logs
> extracts garbage on production logs." Alert 2: "an `awk` summation script
> reports `Total: 0` even though the input clearly has numbers in it."

**Incident 1 — field extraction works on staging, garbage on production**

```bash
$ # staging log format — space-separated:
$ echo "2026-07-25 10:00:01 app-01 ERROR disk full" | awk '{print $3}'
app-01
$ # production log format — an extra bracketed request-ID field was added
$ # upstream of the hostname, which staging's format doesn't have:
$ echo "2026-07-25 10:00:01 [req-8842] app-01 ERROR disk full" | awk '{print $3}'
[req-8842]
```

Tasks:
1. Reference §1.1 — why does the extra bracketed field shift what `$3`
   actually contains between the two formats?
2. Is this an `awk` bug, or a mismatch between the script's assumption and
   the actual data? Justify using how `awk` splits fields by default.
3. What would need to change about the approach (not necessarily the field
   number) to make this robust across log lines with a varying number of
   leading fields?

**Incident 2 — summation script reports Total: 0**

```bash
$ cat sales.csv
alice, 150
bob, 90
carol, 220
$ awk -F',' '{sum += $2} END {print "Total:", sum}' sales.csv
Total: 0
```

Tasks:
1. Look closely at the CSV's formatting — there's a space after each
   comma. Reference §1.1 — what does `$2` actually contain here, and why
   would that break numeric summation?
2. Why does `awk` report `Total: 0` instead of an error, given that `$2`
   isn't a clean number?
3. What's the fix?

<details>
<summary>Answers</summary>

**Incident 1:** `awk` splits on whitespace by default and numbers fields
purely by position, left to right, with no awareness of what each field is
"supposed to" mean. In the staging format, `$3` lands on `app-01` because
it's the third whitespace-separated token. In production, a new field
(`[req-8842]`) was inserted between the timestamp and the hostname — every
field after it shifts right by one position, so `$3` now lands on
`[req-8842]` instead, and `app-01` has moved to `$4`. This is not an
`awk` bug — `awk` is splitting and numbering fields exactly as documented,
every time; the actual problem is the script's implicit assumption that
"field 3 is always the hostname" doesn't hold once a field gets inserted
upstream of it. A more robust approach avoids hardcoding a fixed field
position when the field count can vary — options include matching on a
distinguishing pattern instead of blind positional indexing (e.g. a regex
shaped like the hostname format itself, or anchoring off the constant
`ERROR`/`INFO` level field), or — best, if available — using a genuinely
structured source like JSON logs, parsed with a proper structured-data
tool rather than positional text parsing, if one is available in your
text parsing entirely, since JSON keys don't shift when a new field is
added.

**Incident 2:** The CSV has a space after each comma, so with `-F','`,
`$2` contains `" 150"` — a string with a leading space, not a clean
number. `awk`'s numeric coercion is generally tolerant of leading
whitespace when converting a field to a number for arithmetic, so the
leading space alone often isn't actually fatal — the debugging step that
actually resolves this is printing `$2` directly (`awk -F',' '{print
"["$2"]"}' sales.csv`) to see its exact content with visible boundaries,
rather than assuming the cause. Whatever the field's exact content turns
out to be once inspected this way, the underlying mechanism is the same:
`awk` performs implicit, silent numeric coercion — a field that can't
cleanly convert to a number is treated as `0` in a numeric context rather
than raising an error, which is why malformed or unexpectedly-formatted
input produces a silently wrong `0` total instead of a loud failure. The
fix is always to `print $2` (or `print NF`) as a debugging step before
trusting field-position or formatting assumptions, and to strip/trim any
unexpected whitespace or formatting (e.g. `gsub(/ /, "", $2)`) before
arithmetic if the delimiter assumption doesn't perfectly match the data's
actual formatting.

</details>

## Interview Prep

**Q1. An `awk` script that extracts a specific field by position works on one log format and breaks on another. Is this an `awk` bug?**
No — `awk` splits fields exactly as documented (whitespace by default, or
whatever `-F` specifies); the break happens because the script's assumption
that a given position always contains a specific piece of data doesn't
hold once the line's overall shape changes (extra brackets, different
delimiters, variable-length preceding fields). This is a data-format
assumption mismatch, not a tool malfunction — the fix is either normalizing
input before field-splitting, matching on pattern rather than blind
position, or using a genuinely structured format parsed by a proper tool
instead of positional text parsing when format consistency isn't
positional text parsing when the source format isn't reliably consistent.

**Q2. An `awk` summation script silently reports a total of 0 instead of erroring on malformed numeric input. Why does it fail silently rather than loudly?**
`awk` performs implicit numeric coercion — a field that can't cleanly
convert to a number is treated as `0` in a numeric context rather than
raising an error. This means malformed or unexpectedly-formatted input
produces a silently wrong result instead of a loud failure, which is
exactly why debugging an unexpected `awk` numeric result should start with
directly printing the field in question (`print $2`) to confirm what it
actually contains, rather than assuming the arithmetic logic itself is
wrong.

**Q3. When would a `sort | uniq -c` pipeline be insufficient compared to an `awk` associative-array approach?**
`sort | uniq -c` only counts occurrences of entire identical lines. The
moment the task needs to group by *one specific field* while summing or
counting a *different* field (not just counting whole-line repeats), or
needs any field-aware logic at all, an `awk` associative array
(`arr[$1] += $3`, for example) does something `sort | uniq -c` structurally
cannot express, since `uniq` has no concept of fields at all.

**Q4. What's the practical signal that an `awk` one-liner should become a proper shell script instead?**
Once the logic needs external program calls, complex multi-branch control
flow, robust error handling, or coordination across multiple distinct
files in a non-trivial way, an increasingly dense `awk` program becomes
harder to read and maintain than a shell script that calls `awk` for the
specific stream-processing parts it's actually good at. `awk`'s sweet spot
is field-aware record processing with aggregation — pushing much further
than that is a signal to switch tools, not a reason to write a more
elaborate `awk` program.

## Key Takeaways

1. **`awk` splits fields on whitespace by default, collapsing repeated
   whitespace automatically.** This directly solves the Demo 19 problem
   where `cut -d' '` produced spurious empty fields on irregular
   command-output spacing.
2. **A pattern with no action defaults to printing the whole line — making
   `awk '/pattern/'` a drop-in `grep` replacement for simple cases**,
   though `grep` remains the more idiomatic choice when no field logic is
   needed.
3. **`BEGIN`/`END` run once each, outside the per-line loop — the right
   place for setup and for summaries that only make sense after all lines
   have been processed**, like a running total.
4. **Associative arrays keyed by a field, aggregating another field, cover
   a huge share of real log-summarization tasks** — and do things `sort |
   uniq -c` structurally cannot, since `uniq` has no concept of fields.
5. **`awk` performs silent numeric coercion — malformed input becomes `0`
   in arithmetic, not an error.** When a numeric result looks wrong, `print
   $N` directly to confirm what the field actually contains before
   assuming the arithmetic logic is at fault.
6. **Field-position assumptions break when log format shape changes.** A
   script hardcoding "field 3 is always X" is fragile against format
   drift — pattern-based matching, input normalization, or a structured
   format parsed by a proper structured-data tool are more robust when
   format consistency
   isn't guaranteed.

> **Demo scope:** Primary concept: field-aware text processing and simple
> programmatic logic. Supporting concepts: fields/records, pattern-action
> pairs, BEGIN/END, built-in variables, associative arrays.
> Estimated completion time: 31 minutes (reading + hands-on + verification).
> Checkpoints: 6 natural stopping points (after Task 1 through Task 6).

## Quick Commands Reference

### Fields and records

| Syntax | Purpose |
|---|---|
| `$1`, `$2`, ... `$0` | Field N / whole record |
| `-F','` | Set input field separator |
| `NF` | Number of fields in current record |
| `NR` | Current record number |

### Pattern-action forms

| Syntax | Meaning |
|---|---|
| `'{action}'` | No pattern — every line |
| `'/regex/'` | Pattern, no action — default print (grep-like) |
| `'condition {action}'` | Both — field comparisons or regex as pattern |

### Setup/summary and output

| Syntax | Purpose |
|---|---|
| `BEGIN{...}` | Runs once, before input |
| `END{...}` | Runs once, after all input |
| `OFS="X"` | Output field separator |

### Aggregation

| Syntax | Purpose |
|---|---|
| `arr[$1]++` | Count occurrences keyed by field 1 |
| `arr[$1] += $3` | Sum field 3, keyed by field 1 |
| `for (k in arr) print k, arr[k]` | Iterate and print an associative array |

## Appendix A — Anki Cards

**Note:** extracted into a standalone `.csv` by the series extraction script.

```
#deck:Linux Mastery::Module 4 - Text Processing & Scripting for DevOps::21-awk
#separator:Comma
#columns:Front,Back,Tags
"Why does awk '{print $2}' handle irregularly-spaced command output (like ps) correctly, while cut -d' ' -f2 produces garbage on the same input?","awk splits fields on whitespace by default and automatically collapses repeated whitespace into a single separator; cut treats every single space as a separate field boundary, producing many spurious empty fields on irregular spacing.","demo21,awk,cut,comparison"
"What does the awk program awk '/ERROR/' file.txt do, given it has a pattern but no explicit action?","A pattern with no action defaults to printing the whole matching line — making this a drop-in grep 'ERROR' equivalent, though grep remains more idiomatic when no field logic is needed.","demo21,awk,pattern-action"
"When do awk's BEGIN and END blocks each run, and why is END the right place for a running total's final print?","BEGIN runs once before any input is read; END runs once after all input is processed. A running total only represents the COMPLETE sum once every line has contributed, which is exactly when END fires — printing it per-line during normal processing would be premature.","demo21,awk,begin-end"
"What's the key advantage of an awk associative array (arr[$1]++) over sort | uniq -c for aggregation?","sort | uniq -c only counts occurrences of entire identical lines. An awk associative array can group by ONE specific field while counting or summing a DIFFERENT field — something uniq structurally cannot express, since it has no concept of fields at all.","demo21,awk,associative-arrays,cross-demo"
"An awk summation script (sum += $2) reports Total: 0 on input that clearly has numbers. Why does awk fail silently instead of erroring?","awk performs implicit numeric coercion — a field that can't cleanly convert to a number is silently treated as 0 in arithmetic context rather than raising an error, producing a wrong-but-quiet result instead of a loud failure.","demo21,awk,coercion,common-mistake"
"What's the first debugging step when an awk numeric result looks wrong?","print the field directly (e.g. print $2) to confirm what it actually contains, rather than assuming the arithmetic logic itself is at fault — silent numeric coercion means the real problem is often the field's actual content, not the calculation.","demo21,awk,debugging"
"Why can a field-position-based awk script (extracting $3 for 'the backend name') break when the log format changes, even slightly?","awk splits purely by position after applying its field separator — if the line's overall shape changes (extra brackets, different token counts before the target field), the position that used to correspond to the intended data now points at something else. This is a data-assumption mismatch, not an awk bug.","demo21,awk,fields,troubleshooting"
"What's a more robust alternative to hardcoded field-position extraction when a log format isn't guaranteed consistent?","Match on a distinguishing pattern rather than blind positional indexing, normalize/strip unexpected characters before splitting, or use a genuinely structured format parsed by a proper structured-data tool instead of positional text parsing entirely.","demo21,awk,robustness"
"FS and OFS are independent in awk. What does this let you do in a single command?","Read input with one field separator (e.g. comma-separated CSV via -F',') and print output with a completely different separator (e.g. OFS='|') — print's comma-separated arguments join using OFS, not FS.","demo21,awk,fs,ofs"
"What's the practical signal that an awk one-liner has outgrown awk and should become a proper shell script?","Once the logic needs external program calls, complex multi-branch control flow, robust error handling, or coordination across multiple distinct files in a non-trivial way — awk's sweet spot is field-aware record processing with aggregation, not general-purpose scripting.","demo21,awk,tool-boundary"
```

## Appendix B — Quiz

**Note:** extracted into a standalone `.md` by the series extraction script.

> One correct answer per question unless stated otherwise.
> Target: 80% or above before moving to Demo 22.

**Q1. Why does `awk '{print $2}'` handle `ps aux` output correctly while `cut -d' ' -f2` often doesn't?**

- A) `awk` is simply faster than `cut`
- B) `awk` collapses repeated whitespace into a single field separator automatically; `cut` treats every space as a separate boundary
- C) `cut` cannot process command output, only files
- D) `awk` requires a config file that `ps` already provides

<details>
<summary>Answer</summary>

**B** — This is the actual, practical field-splitting difference between the two tools. Trap: A invents an unrelated performance claim; C is factually wrong; D invents a nonexistent requirement.

</details>

---

**Q2. `awk '/ERROR/'` with no action prints the matching line. Why?**

- A) This is a syntax error that `awk` silently ignores
- B) A pattern with no explicit action defaults to printing the whole matching line
- C) `awk` requires at least an empty `{}` to do anything
- D) `/ERROR/` is treated as a variable assignment, not a pattern

<details>
<summary>Answer</summary>

**B** — This default-print behavior is exactly what makes `awk '/pattern/'` a `grep`-equivalent for simple filtering. Trap: A and C both wrongly claim this is invalid without an action; D misidentifies what `/ERROR/` actually is here.

</details>

---

**Q3. Why is `END` the correct place to print a running total accumulated across all lines, rather than printing it per line?**

- A) `END` runs faster than a per-line print
- B) The total only represents the complete sum once every line has contributed — printing it per-line would show incomplete, premature values
- C) `awk` doesn't allow variables to persist between lines without `END`
- D) `END` is required syntax for any use of the `+=` operator

<details>
<summary>Answer</summary>

**B** — This is the actual reasoning: a running total is only "the total" after all input is processed. Trap: A invents a performance claim; C is factually wrong — variables do persist across the main per-line loop; D invents a false syntax requirement.

</details>

---

**Q4. What can an `awk` associative array do that `sort | uniq -c` cannot?**

- A) Sort output alphabetically
- B) Group by one field while counting or summing a different field — `uniq` has no concept of fields at all
- C) Handle input larger than a certain file size
- D) Remove duplicate lines

<details>
<summary>Answer</summary>

**B** — This field-aware grouping is the genuine capability gap between the two approaches. Trap: A and D both describe things `sort`/`uniq` already do; C invents an unrelated size limitation.

</details>

---

**Q5. An `awk` summation (`sum += $2`) on malformed numeric input reports `Total: 0` instead of an error. Why?**

- A) `awk` always initializes sums to 0 regardless of input
- B) `awk` performs implicit numeric coercion — an unparseable field is silently treated as 0 in arithmetic context
- C) This indicates the input file is empty
- D) `+=` requires explicit type declarations in `awk`, and the error is suppressed

<details>
<summary>Answer</summary>

**B** — Silent coercion to 0 is `awk`'s actual, documented behavior for non-numeric fields in numeric context. Trap: A misreads the cause as a fixed initialization rather than coercion of bad input; C and D invent unrelated explanations.

</details>

---

**Q6. What's the recommended first debugging step when an `awk` numeric result looks wrong?**

- A) Rewrite the entire script from scratch
- B) `print` the field in question directly to confirm what it actually contains
- C) Switch to `sed` instead, since `awk` is unreliable for math
- D) Add more `BEGIN` blocks

<details>
<summary>Answer</summary>

**B** — Confirming the field's actual content directly is the fastest way to catch a silent-coercion issue before assuming the arithmetic logic is wrong. Trap: A overreacts before diagnosing; C wrongly abandons a capable tool; D invents an irrelevant fix.

</details>

---

**Q7. A field-position-based `awk` script (`$3` for "backend name") breaks when a log line gains an extra bracketed timestamp segment. Is this an `awk` bug?**

- A) Yes — `awk` should detect format changes automatically
- B) No — `awk` splits by position exactly as documented; the script's assumption that a fixed position always holds specific data broke due to a format change
- C) Yes, unless `-F` is set explicitly
- D) No, but only because the log wasn't sorted first

<details>
<summary>Answer</summary>

**B** — This is a data-assumption mismatch, not a tool defect — `awk` behaves exactly as documented. Trap: A expects unrealistic automatic format detection; C and D both invent unrelated conditions.

</details>

---

**Q8. What's the practical signal that an `awk` one-liner should become a full shell script instead?**

- A) The `awk` program exceeds 80 characters
- B) The logic needs external program calls, complex control flow, robust error handling, or multi-file coordination beyond field-aware record processing
- C) More than one field is being referenced
- D) `BEGIN` and `END` are both used in the same program

<details>
<summary>Answer</summary>

**B** — This is the genuine complexity threshold worth watching for, not an arbitrary length or feature-count rule. Trap: A, C, and D all invent superficial thresholds that don't reflect real complexity.

</details>