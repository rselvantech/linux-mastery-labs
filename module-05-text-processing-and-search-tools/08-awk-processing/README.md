# Demo M05-08 — awk Processing

## Overview

`grep` finds, `sed` transforms line-by-line — `awk` is the tool that thinks
in fields and can do arithmetic, conditionals, and aggregation across an
entire file in one pass. This is the tool Demo M05-05's log investigation
gestured toward with `sort | uniq -c` — `awk` can do that kind of
field-aware summarization directly, often replacing a multi-command
pipeline with one `awk` program. By the end, you'll read and write `awk`
one-liners confidently and know when its field/record model is genuinely
the right fit.

## Demo Objectives

1. Explain `awk`'s field/record model and access fields by position,
   including `$0` and out-of-range fields
2. Write pattern-action pairs, including patterns with no explicit action,
   actions with no explicit pattern, and real `if`/`else` conditionals
   inside an action block
3. Use `BEGIN`/`END` blocks for setup and summary logic
4. Use built-in variables (`NR`, `NF`, `FS`, `OFS`) to control processing,
   and `printf` for formatted output
5. Use associative arrays to aggregate and count by key
6. Use `gsub` and `awk`'s other string functions (`length`, `substr`,
   `split`, `toupper`/`tolower`) for in-field processing, and recognize
   when a task has outgrown awk and belongs in a full script instead

## Directory Structure

```
08-awk-processing/
├── README.md                       # this file
├── m05-08-awk-processing-anki.csv  # Anki flashcard deck
└── m05-08-awk-processing-quiz.md   # standalone quiz
```

## Environments

| Task | Ubuntu 24.04 | Rocky Linux 9.8 | EC2 Amazon Linux |
|---|---|---|---|
| All lab tasks | ✅ (gawk) | ✅ (gawk) | Not required |

## Recall Check — 07-sed-editing

Answer these from memory before continuing — no looking back at Demo M05-07.

1. `sed 's/a/b/'` on input `"aaa"` produces `"baa"`, not `"bbb"`. Why?

2. `sed '5p' file.txt` (no `-n`) shows line 5 twice, while every other
   line shows once. Why?

3. Why is `sed -i` with no backup suffix considered genuinely risky?

<details>
<summary>Answers</summary>

1. `s///` replaces only the first match per line by default — `g` is
   needed to replace every match on the line.

2. `sed` prints every line by default; `p` adds an explicit print on
   top of that rather than replacing it — without `-n` to suppress the
   default, the addressed line gets both.

3. It's irreversible — the original content is overwritten with no
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
whitespace** — directly solving the Demo M05-06 problem where `cut -d' '`
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

**Referencing a field beyond how many actually exist on a line doesn't
error** — `$10` on a 3-field line simply evaluates to an empty string,
silently, not a crash or an out-of-range error. This matters when a
script assumes a fixed field count that turns out to be wrong for some
lines — the symptom is quietly missing/empty output, not a loud failure,
similar in spirit to `awk`'s silent numeric coercion covered in §1.5.

### 1.2 Pattern-action pairs, and real conditionals inside an action

An `awk` program is a series of `pattern { action }` pairs. Either half can
be omitted, with a specific meaning each time:

```bash
awk '{print}' file.txt
# No pattern = matches every line. print with no args = print $0 (whole line).
# Equivalent to cat.

awk '1' file.txt
# An even terser version of the same idea: a bare "1" as the pattern with
# no action at all. 1 is always true in awk, so this matches every line
# and falls back to the default action (print $0) — functionally
# identical to awk '{print}', and a common idiom worth recognizing even
# though it reads cryptically the first time you see it.

awk '/ERROR/' file.txt
# Pattern with NO action = default action is print. Equivalent to grep 'ERROR'.

awk '$3 > 100 {print $1}' file.txt
# Pattern is a field comparison (not a regex) — matches lines where the
# 3rd field, treated as a number, exceeds 100. Action prints field 1 for
# those lines only. This filters WHICH lines run the action at all — a
# different mechanism from the if/else below, which branches WITHIN an
# action that already runs on every line.

awk '/ERROR/ {print NR": "$0}' file.txt
# Pattern AND action together — for lines matching /ERROR/, print the line
# number (NR, see §1.4) and the full line.
```

**A pattern with no action defaulting to "print the whole line" is exactly
what makes `awk '/pattern/'` a drop-in `grep` replacement** for simple
cases — worth recognizing, though `grep` remains the more idiomatic choice
when field logic isn't needed at all.

**`awk` also supports real `if`/`else` branching *inside* an action
block** — a genuinely different mechanism from the pattern-level filtering
above. A pattern like `$3 > 100 {...}` decides *whether the action runs at
all* for a given line; an `if`/`else` inside the action runs on *every*
line and decides *which of several outcomes* happens each time:

```bash
printf '150\n90\n220\n' | awk '{if ($1 > 100) print $1, "high"; else print $1, "low"}'
```
```
150 high
90 low
220 high
```
> Live-verified — every line produces output (unlike a pattern-filtered
> program, which would silently skip non-matching lines entirely), and
> the `if`/`else` decides which message applies each time. This is real
> conditional branching, part of what the Overview means by "awk can do
> ... conditionals" — worth distinguishing clearly from a pattern
> condition, which is a filter, not a branch.

**A precision worth having, since it can silently produce a wrong
result: `awk` compares two literal string constants lexically, but
compares a field read from input numerically if that field looks like a
number** — these are two different default behaviors, not one:

```bash
awk 'BEGIN{ if ("10" > "9") print "string: TRUE"; else print "string: FALSE" }'
# string: FALSE — "10" and "9" here are literal string constants written
# directly in the program; awk compares them character-by-character,
# same lexical logic as Demo M05-06's plain sort without -n, and '1' < '9'.

printf '9\n10\n' | awk '$1 > 9 {print $1, "matched"}'
# 10 matched — $1 here is a FIELD read from actual input. awk detects
# that it looks like a number and compares it numerically against the
# numeric constant 9, correctly treating 10 as greater than 9.
```
> Live-verified both cases. The distinction is *where the value comes
> from*: a value typed directly into the program as `"10"` is a string
> constant and stays lexical unless explicitly converted; a field read
> from input that looks numeric is automatically treated as a number for
> comparison purposes. Getting this backwards — assuming input fields
> compare lexically like hardcoded string literals do, or vice versa —
> is exactly the kind of silent, no-error mistake this series flags
> repeatedly for other tools' defaults.

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

`NR` (see §1.4) still holds a meaningful value inside `END` — it's not
reset once input processing finishes, so `END {print "processed", NR,
"lines"}` correctly reports the total line count, the same running
counter that was live throughout the main loop.

### 1.4 Built-in variables — NR, NF, FS, OFS — and printf for formatted output

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

**`-F','` on the command line and `BEGIN{FS=","}` inside the program are
equivalent** — `-F` is simply a shorthand that sets `FS` before processing
begins, the same effect as setting it explicitly in a `BEGIN` block.
Forgetting either one when the data genuinely isn't whitespace-separated
means `awk` falls back to its default whitespace splitting, which for a
comma-separated line with no embedded spaces produces exactly one field —
the entire line — rather than the several fields you likely intended.

**`print` is fine for quick output, but `printf` gives real formatting
control — column widths, decimal precision, padding — the same `%`-style
specifiers C, Python, and most other languages use:**

```bash
printf 'alice,150\nbob,9000\n' | awk -F',' '{printf "%-10s $%8.2f\n", $1, $2}'
```
```
alice      $  150.00
bob        $ 9000.00
```
> Live-verified. `%-10s` left-justifies a string in a 10-character-wide
> field (the `-` means left-align; without it, `%10s` right-aligns);
> `%8.2f` right-aligns a floating-point number in an 8-character field
> with exactly 2 decimal places. Unlike `print`, `printf` does **not**
> add a trailing newline automatically — a `\n` has to be written
> explicitly, easy to forget when switching from `print` to `printf` for
> the first time. This is the tool for genuinely aligned, report-style
> output — the kind `column` (Demo M05-06) formats after the fact, but
> `printf` produces directly, with numeric precision control `column`
> doesn't have at all.

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

### 1.6 gsub and awk's other string functions

`awk` isn't limited to reading and comparing fields — `gsub` (global
substitute) edits field or record content in place, `awk`'s equivalent of
`sed`'s `s///g`:

```bash
echo "a.b.c" | awk '{gsub(/\./, "-"); print}'
# a-b-c — gsub(regex, replacement) rewrites every match within $0 (the
# whole record) by default; gsub(/\./, "-", $2) would instead target just
# field 2, leaving other fields untouched.
```

This is directly useful for cleaning up a field before treating it as a
number — a field like `" 150"` (leading space from a delimiter like `,
` instead of a bare `,`) can silently coerce to `0` in arithmetic (§1.5's
silent-coercion trap); `gsub(/ /, "", $2)` strips the stray whitespace
before the value is used, turning a debugging session into a one-line fix
once the actual field content has been confirmed with a direct `print`.

**`gsub` is one of several built-in string functions worth knowing by
name — the others cover length, extraction, and case, none of which
involve regex matching at all:**

```bash
echo "hello world" | awk '{print length($0)}'
# 11 — length() returns the character count of its argument ($0 here);
# also usable on a specific field, length($1), or a plain string.

echo "hello world" | awk '{print substr($0, 1, 5)}'
# hello — substr(string, start, length): extract 5 characters starting
# at position 1 (awk string positions are 1-indexed, not 0-indexed).

echo "hello" | awk '{print toupper($0)}'
# HELLO — toupper()/tolower() convert case, the awk-native equivalent of
# tr 'a-z' 'A-Z' (Demo M05-06), scoped to a single value rather than a stream.

echo "a:b:c:d" | awk '{n = split($0, parts, ":"); print n, parts[1], parts[3]}'
```
```
4 a c
```
> Live-verified all four. `split(string, array, separator)` is distinct
> from `-F`/`FS`, which splits the *whole record* into `$1`/`$2`/etc. —
> `split()` instead splits *any string value* (often a single field
> that itself contains a delimited sub-list) into a fresh array,
> returning the number of pieces produced. This is the tool for a field
> that's itself structured data — a comma-list embedded inside one CSV
> column, for instance — which the record-level `FS` alone can't reach
> into.

`awk` is genuinely a small programming language — loops, conditionals,
functions, arrays — and it's tempting to keep extending an `awk` one-liner
indefinitely. The practical line: once a transformation needs external
program calls, complex control flow across many conditions, robust error
handling, or state that spans multiple distinct files in a non-trivial way,
a proper shell script (Demo M06-01/24) — possibly calling `awk` for the parts
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
| `awk '1'` / `awk '{print}'` | pattern-only / action-only | Print every line (cat-equivalent) |
| `awk '/PAT/'` | pattern, no action | Grep-equivalent filtering |
| `awk '{if (...) ...; else ...}'` | real conditional | Branch within an action, on every line |
| `awk 'BEGIN{...} {...} END{...}'` | — | Setup, per-line, and summary blocks |
| `NR` / `NF` | built-in variables | Record number / field count |
| `printf "%-10s %8.2f\n", a, b` | format specifiers | Aligned, precision-controlled output |
| `awk '{arr[$1]++} END{for (k in arr) print k, arr[k]}'` | associative array | Aggregate and count by key |
| `gsub` / `length` / `substr` / `split` / `toupper` | string functions | In-field editing, measuring, extracting, splitting, case conversion |

### Task 1 — Fields vs. cut on irregular whitespace

```bash
ps aux | head -5 > /tmp/ps-sample.txt
cat /tmp/ps-sample.txt
awk '{print $2, $11}' /tmp/ps-sample.txt
cut -d' ' -f2 /tmp/ps-sample.txt
# Compare the cut output (likely mangled by irregular spacing) against
# awk's clean field extraction.
```

### Task 2 — Pattern-action pairs, and a real if/else conditional

```bash
printf 'INFO starting\nERROR disk full\nINFO running\nERROR oom\n' > /tmp/awk-lab.txt
awk '{print}' /tmp/awk-lab.txt
awk '1' /tmp/awk-lab.txt
# Confirm both produce identical output — two ways to write "print every
# line" — before moving to the pattern-only and pattern+action forms:
awk '/ERROR/' /tmp/awk-lab.txt
awk '/ERROR/ {print NR": "$0}' /tmp/awk-lab.txt

printf '150\n90\n220\n' | awk '{if ($1 > 100) print $1, "high"; else print $1, "low"}'
# Confirm every line produces output (high or low), unlike a pattern
# filter which would silently drop non-matching lines entirely.

awk 'BEGIN{ if ("10" > "9") print "string: TRUE"; else print "string: FALSE" }'
printf '9\n10\n' | awk '$1 > 9 {print $1, "matched"}'
# Confirm the string-constant comparison is FALSE (lexical) while the
# field-from-input comparison correctly matches "10" as numerically
# greater than 9 — two different default behaviors depending on source.
```

### Task 3 — BEGIN/END for a running total

```bash
printf 'alice,150\nbob,90\ncarol,220\n' > /tmp/sales.csv
awk -F',' 'BEGIN{print "Processing sales..."} {sum += $2} END{print "Total:", sum}' /tmp/sales.csv
awk -F',' '{sum += $2} END{print "processed", NR, "lines, total:", sum}' /tmp/sales.csv
# Confirm NR still holds the correct total line count inside END.
```

### Task 4 — NR/NF for data validation, and printf for a formatted report

```bash
printf 'a,b,c\nd,e\nf,g,h,i\n' > /tmp/awk-lab2.csv
awk -F',' 'NF != 3 {print NR": expected 3 fields, got "NF}' /tmp/awk-lab2.csv

echo "a b c" | awk '{print "["$10"]"}'
# Confirm a field beyond NF prints as an empty string ("[]"), not an error.

awk -F',' '{printf "%-10s $%8.2f\n", $1, $2}' /tmp/sales.csv
# Confirm aligned, decimal-precise output — compare visually against the
# plain `print $1, $2` version's unaligned output.
```

### Task 5 — Associative array aggregation

```bash
printf 'app-01 ERROR\napp-02 ERROR\napp-01 ERROR\napp-03 INFO\napp-01 ERROR\n' > /tmp/awk-lab3.txt
awk '$2=="ERROR" {count[$1]++} END {for (k in count) print k, count[k]}' /tmp/awk-lab3.txt
# Compare mentally against the Demo M05-05/19 equivalent:
grep 'ERROR' /tmp/awk-lab3.txt | awk '{print $1}' | sort | uniq -c
```

### Task 6 — FS/OFS, gsub, and awk's other string functions

```bash
printf 'alice,30,nyc\nbob,25,la\n' > /tmp/awk-lab4.csv
awk -F',' 'BEGIN{OFS="|"} {print $1, $3}' /tmp/awk-lab4.csv

# Confirm -F',' and BEGIN{FS=","} produce the same result:
awk 'BEGIN{FS=","; OFS="|"} {print $1, $3}' /tmp/awk-lab4.csv

echo "a.b.c" | awk '{gsub(/\./, "-"); print}'
printf 'alice, 150\nbob, 90\n' | awk -F',' '{gsub(/ /, "", $2); sum += $2} END {print "Total:", sum}'
# Confirm stripping the stray leading space before summation gives a
# correct, non-zero total, unlike the un-stripped version in the Break-Fix
# scenario below.

echo "hello world" | awk '{print length($0)}'
echo "hello world" | awk '{print substr($0, 1, 5)}'
echo "hello" | awk '{print toupper($0)}'
echo "a:b:c:d" | awk '{n = split($0, parts, ":"); print n, parts[1], parts[3]}'
```

## What You Learned

1. ✅ Explained `awk`'s field/record model, extracted fields by position,
   and confirmed out-of-range fields evaluate to an empty string rather
   than erroring
2. ✅ Wrote pattern-action pairs, including pattern-only, action-only, and
   the bare-`1` idiom, and wrote a real `if`/`else` conditional inside an
   action block — distinct from pattern-level filtering
3. ✅ Used `BEGIN`/`END` blocks for setup and summary logic, confirming
   `NR` remains valid inside `END`
4. ✅ Used `NR`, `NF`, `FS`, `OFS` to control and validate processing,
   confirmed `-F` and `BEGIN{FS=...}` are equivalent, and used `printf`
   for aligned, precision-controlled output
5. ✅ Used associative arrays to aggregate and count by key in one pass
6. ✅ Used `gsub` plus `length`/`substr`/`split`/`toupper` for in-field
   processing, and recognized when a task has outgrown awk and belongs in
   a shell script

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
`ERROR`/`INFO` level field), or — best, if a structured source is
available — using something like JSON logs parsed with a proper
structured-data tool instead of positional text parsing entirely, since
JSON keys don't shift when a new field is added.

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
unexpected whitespace or formatting (`gsub(/ /, "", $2)`, per §1.6) before
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
instead of positional text parsing when the source format isn't reliably
consistent.

---

**Q2. An `awk` summation script silently reports a total of 0 instead of erroring on malformed numeric input. Why does it fail silently rather than loudly?**

`awk` performs implicit numeric coercion — a field that can't cleanly
convert to a number is treated as `0` in a numeric context rather than
raising an error. This means malformed or unexpectedly-formatted input
produces a silently wrong result instead of a loud failure, which is
exactly why debugging an unexpected `awk` numeric result should start with
directly printing the field in question (`print $2`) to confirm what it
actually contains, rather than assuming the arithmetic logic itself is
wrong.

---

**Q3. What's the difference between a pattern-level condition like `$3 > 100 {print $1}` and an `if`/`else` written inside an action block?**

A pattern-level condition decides whether the action runs *at all* for a
given line — non-matching lines are silently skipped entirely, producing
no output for them. An `if`/`else` inside an action runs on every line
that reaches it and branches between outcomes each time — every line
still produces some output, just a different one depending on the
condition. They look similar but serve different purposes: filtering
versus branching.

---

**Q4. Why does `awk 'BEGIN{if ("10" > "9") ...}'` evaluate the comparison differently from `awk '$1 > 9 {...}'` run against a file containing "10"?**

The first compares two literal string constants written directly in the
program, which `awk` treats lexically (character-by-character) by
default — `"10" > "9"` is false, the same logic as an unsorted lexical
comparison. The second compares a field read from actual input, which
`awk` automatically detects as numeric-looking and compares numerically —
`10 > 9` is true. The behavior depends on where the value came from, not
just what it looks like, which is easy to get backwards if you assume
input fields and hardcoded string literals compare the same way.

---

## Key Takeaways

1. **`awk` splits fields on whitespace by default, collapsing repeated
   whitespace automatically.** This directly solves the Demo M05-06 problem
   where `cut -d' '` produced spurious empty fields on irregular
   command-output spacing. A field beyond `NF` (e.g. `$10` on a 3-field
   line) is an empty string, not an error.
2. **A pattern-level condition filters which lines run an action at all;
   an `if`/`else` inside an action branches between outcomes on every
   line that reaches it.** These look similar but serve different
   purposes — confusing one for the other is a real source of "why did
   this line disappear entirely" versus "why did every line print
   something."
3. **`awk` compares literal string constants lexically, but automatically
   compares input fields that look like numbers numerically.** `"10" >
   "9"` (both string constants) is false; `$1 > 9` where `$1` was read
   from input as "10" is true — the behavior depends on where the value
   came from, not just its appearance.
4. **A pattern with no action defaults to printing the whole line — making
   `awk '/pattern/'` a drop-in `grep` replacement for simple cases**,
   though `grep` remains the more idiomatic choice when no field logic is
   needed. The bare `1` pattern (always true, no explicit action) is a
   common, terser idiom for the same "print everything" effect.
5. **`BEGIN`/`END` run once each, outside the per-line loop — the right
   place for setup and for summaries that only make sense after all lines
   have been processed**, like a running total. `NR` still holds the
   correct total line count inside `END`, since it isn't reset.
6. **`-F','` and `BEGIN{FS=","}` are equivalent** — `-F` is shorthand for
   setting `FS` before processing starts. `printf` (unlike `print`) gives
   real formatting control — column widths and decimal precision via
   `%`-style specifiers — but doesn't add a trailing newline automatically,
   unlike `print`.
7. **Associative arrays keyed by a field, aggregating another field, cover
   a huge share of real log-summarization tasks** — and do things `sort |
   uniq -c` structurally cannot, since `uniq` has no concept of fields.
8. **`awk` performs silent numeric coercion — malformed input becomes `0`
   in arithmetic, not an error.** When a numeric result looks wrong, `print
   $N` directly to confirm what the field actually contains before
   assuming the arithmetic logic is at fault; `gsub` can then clean up the
   field (e.g. stripping stray whitespace) before retrying the arithmetic.
9. **`length`, `substr`, `split`, and `toupper`/`tolower` round out `awk`'s
   string-handling alongside `gsub` — `split` in particular reaches
   inside a single field's own delimited sub-content**, something
   record-level `FS` splitting can't do on its own.

> **Demo scope:** Primary concept: field-aware text processing and simple
> programmatic logic. Supporting concepts: fields/records, pattern-action
> pairs and real conditionals, BEGIN/END, built-in variables, printf,
> associative arrays, gsub and other string functions.
> Estimated completion time: 42 minutes (reading + hands-on + verification).
> Checkpoints: 6 natural stopping points (after Task 1 through Task 6).

## Quick Commands Reference

### Fields and records

| Syntax | Purpose |
|---|---|
| `$1`, `$2`, ... `$0` | Field N / whole record |
| `-F','` | Set input field separator (equivalent to `BEGIN{FS=","}`) |
| `NF` | Number of fields in current record |
| `NR` | Current record number (valid inside `END` too) |

### Pattern-action forms and conditionals

| Syntax | Meaning |
|---|---|
| `'{action}'` | No pattern — every line |
| `'1'` | Bare truthy pattern, no action — same effect as `'{print}'` |
| `'/regex/'` | Pattern, no action — default print (grep-like) |
| `'condition {action}'` | Pattern filters WHICH lines run the action |
| `'{if (c) X; else Y}'` | Conditional branches on EVERY line reaching it |

### Setup/summary and output

| Syntax | Purpose |
|---|---|
| `BEGIN{...}` | Runs once, before input |
| `END{...}` | Runs once, after all input |
| `OFS="X"` | Output field separator |
| `printf "%-10s %8.2f\n", a, b` | Formatted output — width, precision, alignment (no auto-newline) |

### Aggregation and string functions

| Syntax | Purpose |
|---|---|
| `arr[$1]++` | Count occurrences keyed by field 1 |
| `arr[$1] += $3` | Sum field 3, keyed by field 1 |
| `for (k in arr) print k, arr[k]` | Iterate and print an associative array |
| `gsub(/PAT/, REPL, $N)` | Global substitute within field N (or `$0` if omitted) |
| `length(s)` | Character count of `s` |
| `substr(s, start, len)` | Extract a substring (1-indexed) |
| `split(s, arr, sep)` | Split a string into an array, return piece count |
| `toupper(s)` / `tolower(s)` | Case conversion |

## Appendix A — Anki Cards

**`m05-08-awk-processing-anki.csv`:**

```
#deck:Linux Mastery::Module 5 - Text Processing & Search Tools::Demo M05-08 - awk Processing
#separator:Comma
#columns:Front,Back,Tags
"Why does awk '{print $2}' handle irregularly-spaced command output (like ps) correctly, while cut -d' ' -f2 produces garbage on the same input?","awk splits fields on whitespace by default and automatically collapses repeated whitespace into a single separator; cut treats every single space as a separate field boundary, producing many spurious empty fields on irregular spacing.","demo21,awk,cut,comparison"
"What's the difference between a pattern-level condition ($3 > 100 {print $1}) and an if/else written INSIDE an action block?","A pattern condition decides whether the action runs AT ALL for a line — non-matching lines are silently skipped, no output. An if/else inside an action runs on every line reaching it and branches between outcomes — every line produces some output, just a different one depending on the condition.","demo21,awk,conditionals,fundamentals"
"awk 'BEGIN{if (\"10\" > \"9\") ...}' evaluates this comparison as FALSE. But awk '$1 > 9 {...}' against input containing '10' matches it as greater than 9. Why the difference?","Literal string constants written in the program are compared lexically by default (\"10\" > \"9\" is false, character-by-character). Fields read from actual input that look numeric are automatically compared numerically. The behavior depends on where the value came from, not just its appearance.","demo21,awk,string-vs-numeric,precision"
"What does awk's printf give you that plain print doesn't?","Real formatting control via %-style specifiers — column widths, alignment (left/right), and decimal precision (e.g. %-10s %8.2f). Unlike print, printf does NOT add a trailing newline automatically — \\n must be written explicitly.","demo21,awk,printf,fundamentals"
"What does awk's split(string, array, separator) do, and how is it different from -F/FS?","split() splits any STRING VALUE (often one field's own delimited sub-content) into a fresh array, returning the piece count. -F/FS splits the WHOLE RECORD into $1/$2/etc. at read time — split() reaches inside a single already-extracted field, which FS alone can't do.","demo21,awk,split,fundamentals"
"What does awk '/ERROR/' do, given it has a pattern but no explicit action?","A pattern with no action defaults to printing the whole matching line — making this a drop-in grep 'ERROR' equivalent, though grep remains more idiomatic when no field logic is needed.","demo21,awk,pattern-action"
"When do awk's BEGIN and END blocks each run, and why is END the right place for a running total's final print?","BEGIN runs once before any input is read; END runs once after all input is processed. A running total only represents the COMPLETE sum once every line has contributed, which is exactly when END fires — printing it per-line during normal processing would be premature.","demo21,awk,begin-end"
"What's the key advantage of an awk associative array (arr[$1]++) over sort | uniq -c for aggregation?","sort | uniq -c only counts occurrences of entire identical lines. An awk associative array can group by ONE specific field while counting or summing a DIFFERENT field — something uniq structurally cannot express, since it has no concept of fields at all.","demo21,awk,associative-arrays,cross-demo"
"An awk summation script (sum += $2) reports Total: 0 on input that clearly has numbers. Why does awk fail silently instead of erroring?","awk performs implicit numeric coercion — a field that can't cleanly convert to a number is silently treated as 0 in arithmetic context rather than raising an error, producing a wrong-but-quiet result instead of a loud failure.","demo21,awk,coercion,common-mistake"
"What's the first debugging step when an awk numeric result looks wrong?","print the field directly (e.g. print $2) to confirm what it actually contains, rather than assuming the arithmetic logic itself is at fault — silent numeric coercion means the real problem is often the field's actual content, not the calculation.","demo21,awk,debugging"
"FS and OFS are independent in awk. What does this let you do in a single command?","Read input with one field separator (e.g. comma-separated CSV via -F',') and print output with a completely different separator (e.g. OFS='|') — print's comma-separated arguments join using OFS, not FS.","demo21,awk,fs,ofs"
"What's the practical signal that an awk one-liner has outgrown awk and should become a proper shell script?","Once the logic needs external program calls, complex multi-branch control flow, robust error handling, or coordination across multiple distinct files in a non-trivial way — awk's sweet spot is field-aware record processing with aggregation, not general-purpose scripting.","demo21,awk,tool-boundary"
```

## Appendix B — Quiz

**`m05-08-awk-processing-quiz.md`:**

````markdown
# Quiz — Demo M05-08: awk Processing

> One correct answer per question unless stated otherwise.
> Target: 80% or above before moving to Demo M05-10.

---

**Q1.** `awk '{print $0}'` and `awk '{print $1, $2}'` are run on the same three-field line. What's the difference between what `$0` and `$1`/`$2` each refer to?

- A) `$0` refers to field 0, which is always empty; `$1`/`$2` are the only real fields
- B) `$0` refers to the entire original record (the whole line, unsplit); `$1`/`$2` refer to individual fields by position after splitting
- C) `$0` and `$1` are interchangeable aliases for the first field
- D) `$0` only works inside a `BEGIN` block

<details>
<summary>Answer</summary>

**B** — `$0` is the whole record as originally read, regardless of how it was split into fields; `$1`, `$2`, etc. are the individual fields produced by that split. They represent different things, not aliases of each other.

Trap: A invents a nonexistent "field 0" concept; C wrongly claims equivalence; D invents a scope restriction that doesn't exist — `$0` works anywhere in the main per-line processing too.

</details>

---

**Q2.** `awk '$3 > 100 {print $1}'` and `awk '{if ($3 > 100) print $1}'` are compared against a file where some lines have `$3 <= 100`. Do they produce the same output?

- A) No — the first is invalid syntax
- B) Yes — for this specific comparison, both produce identical output; the pattern-level version filters which lines run the action, while the if/else version would only differ in behavior once an `else` branch adds output for the non-matching case too
- C) No — the `if` version always runs on every line regardless of the condition
- D) Yes, but only if `NF` equals exactly 3

<details>
<summary>Answer</summary>

**B** — With no `else`, both forms produce identical visible output here: only lines where `$3 > 100` produce a printed `$1`. The real difference emerges once an `else` branch is added, which would make every line produce output instead of silently skipping non-matching ones.

Trap: A invents an syntax error that doesn't exist; C misreads what "runs on every line" means — the `if` condition is still checked per line, it just happens to produce identical results here without an `else`; D invents an unrelated field-count dependency.

</details>

---

**Q3.** `awk 'BEGIN{if ("10" > "9") print "A"; else print "B"}'` prints "B". Why, given that 10 is numerically greater than 9?

- A) This is a bug in `awk`
- B) `"10"` and `"9"` here are literal string constants in the program, which `awk` compares lexically (character-by-character) by default — `'1'` is a lower character than `'9'`, so `"10" > "9"` is false as a string comparison
- C) `awk` cannot compare numbers larger than one digit
- D) The `BEGIN` block ignores comparison operators entirely

<details>
<summary>Answer</summary>

**B** — String constants written directly into an `awk` program are compared lexically unless explicitly converted to numbers — this is the same lexical-vs-numeric distinction Demo M05-06 covered for `sort`, just with `awk`'s own default applying to hardcoded string literals specifically.

Trap: A misdiagnoses correct, documented behavior as a bug; C invents an arbitrary digit-count limitation; D invents an unrelated restriction on `BEGIN` blocks.

</details>

---

**Q4.** `printf '9\n10\n' | awk '$1 > 9 {print $1}'` correctly prints `10`. Why does this differ from the string-constant comparison in the previous question, even though both involve the values 9 and 10?

- A) It doesn't actually differ; both examples behave identically
- B) `$1` here is a field read from actual input, which `awk` automatically detects as numeric-looking and compares numerically — a different default behavior from comparing literal string constants typed directly into the program
- C) `printf` forces all piped values to be treated as numbers
- D) The pipe (`|`) itself changes how `awk` interprets comparisons

<details>
<summary>Answer</summary>

**B** — This is the core distinction: fields read from input get automatic numeric comparison when they look like numbers, while string constants in the program stay lexical by default — two different rules depending on where the value originates.

Trap: A wrongly claims no difference exists, missing the actual point of the comparison; C and D both invent unrelated causes (a formatting command and a shell operator) for behavior that's actually about `awk`'s own field-typing rules.

</details>

---

**Q5.** `awk -F',' '{printf "%-10s $%8.2f\n", $1, $2}' sales.csv` is run. What does the `-` in `%-10s` do, and what happens if the trailing `\n` is accidentally omitted?

- A) `-` right-aligns the string; omitting `\n` has no visible effect since `printf` adds one automatically
- B) `-` left-aligns the string within a 10-character field; omitting `\n` means output runs together with no line breaks, since unlike `print`, `printf` does not add a newline automatically
- C) `-` subtracts 10 from the string's length before printing
- D) `-10s` is invalid syntax and would cause an error

<details>
<summary>Answer</summary>

**B** — `-` in a `printf` format specifier requests left-alignment (right-alignment is the default without it); and critically, `printf` never adds a newline on its own the way `print` does, so omitting `\n` runs all output together.

Trap: A reverses the alignment default and wrongly assumes automatic newlines, which is exactly the opposite of `printf`'s actual behavior; C invents an arithmetic meaning for `-`; D invents a syntax error for valid, standard formatting syntax.

</details>

---

**Q6.** `echo "a:b:c:d" | awk '{n = split($0, parts, ":"); print n, parts[1]}'` is run. What does `split()` do here, and how does it differ from setting `FS=":"`?

- A) They are identical; `split()` is just another way to set the record's field separator
- B) `split()` splits the given STRING (here, the whole record `$0`, but it could be any single field) into a separate array, returning the count of pieces — distinct from `FS`, which splits the whole incoming record into `$1`/`$2`/etc. at read time
- C) `split()` only works on numeric strings, not general text
- D) `split()` requires the target array to already contain existing values

<details>
<summary>Answer</summary>

**B** — `split()` operates on-demand on any string value you give it (often a single field that itself contains delimited sub-content), producing a fresh array — a genuinely different mechanism from `FS`, which governs how `awk` divides the entire incoming record into its numbered fields in the first place.

Trap: A wrongly conflates two different mechanisms; C invents a numeric-only restriction that doesn't exist; D invents an unnecessary pre-population requirement — `split()` populates the array itself.

</details>

---

**Q7.** Inside an `END` block, is `NR` still usable, and if so, what does it report?

- A) `NR` is reset to 0 once `END` begins, since input processing has finished
- B) `NR` still holds its value from the last line processed — the total number of records read — since it isn't reset when `END` starts
- C) `NR` becomes undefined and referencing it causes an error
- D) `NR` only works inside `BEGIN`, not `END`

<details>
<summary>Answer</summary>

**B** — `NR` is a running counter maintained throughout the main processing loop; by the time `END` runs, it simply holds its final value — the total record count — which is exactly why `END {print NR, "lines processed"}` is a common, correct idiom.

Trap: A and C both invent a reset/undefined state that doesn't happen; D reverses the actual scoping — `NR` is meaningful in both `BEGIN` (where it's typically 0) and `END`.

</details>

---

**Q8.** Is `awk -F',' '{print $2}' file.csv` functionally different from `awk 'BEGIN{FS=","} {print $2}' file.csv`?

- A) Yes — `-F` only affects the first line of input; `BEGIN{FS=...}` affects all lines
- B) No — `-F` is shorthand that sets `FS` before processing begins, exactly the same effect as setting it explicitly in a `BEGIN` block
- C) Yes — `-F` is deprecated in favor of `BEGIN{FS=...}` and may stop working in future versions
- D) Yes — `BEGIN{FS=...}` only works with single-character separators, while `-F` supports multi-character ones

<details>
<summary>Answer</summary>

**B** — Both approaches achieve the identical result: setting `FS` before any input is split. `-F` is just command-line shorthand for what `BEGIN{FS="..."}` does explicitly inside the program.

Trap: A invents an incorrect line-scoped restriction; C invents a deprecation that isn't happening; D invents a capability difference that doesn't exist between the two equivalent approaches.

</details>

---

Score guide:

| Score | Action |
|---|---|
| 7/8 or 8/8 | Import Anki cards, move to Demo M05-10 |
| 6/8 | Review the wrong answers, then proceed |
| 5/8 | Re-read the relevant sections, retry those questions |
| Below 5/8 | Re-read the full demo and redo the walkthrough before proceeding |
````