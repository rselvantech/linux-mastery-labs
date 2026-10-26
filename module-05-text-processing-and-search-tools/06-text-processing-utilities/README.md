# Demo M05-06 — Text Processing Utilities

## Overview

`grep` finds lines; this demo covers the smaller, sharper tools that shape,
rearrange, and summarize text once you've found it. `cut`, `sort`, `uniq`,
`tr`, `wc`, `xargs`, `tee`, `paste`, and `column` are individually simple —
most do one thing — but composed in a pipeline, they cover a huge share of
everyday text-wrangling without needing `sed` or `awk` at all. Demo M05-05's log
investigation ended with exactly this kind of composition (`sort | uniq
-c`); this demo builds out the full toolkit that pattern draws from.

## Demo Objectives

1. Extract specific fields or character ranges from lines with `cut`,
   including inverting the selection and controlling the output delimiter
2. Sort text numerically, by field, in reverse, and by human-readable
   magnitude (`-h`) with `sort`
3. Deduplicate and count repeated lines with `uniq`, including showing
   only what's duplicated or only what's unique, skipping fields/characters
   during comparison, and why sorting first matters
4. Transform, squeeze, delete, or complement characters with `tr`,
   including its POSIX named classes
5. Count lines, words, characters, and the longest line with `wc`,
   including the filename-argument vs. stdin-redirect difference in its output
6. Convert piped input into command arguments with `xargs`, safely,
   including parallel execution and avoiding a spurious run on empty input
7. Combine `tee`, `paste`, and `column` for output splitting, merging, and
   formatting, including appending and single-file serial joining

## Directory Structure

```
06-text-processing-utilities/
├── README.md                                  # this file
├── m05-06-text-processing-utilities-anki.csv  # Anki flashcard deck
└── m05-06-text-processing-utilities-quiz.md   # standalone quiz
```

## Environments

| Task | Ubuntu 24.04 | Rocky Linux 9.8 | EC2 Amazon Linux |
|---|---|---|---|
| All lab tasks | ✅ | ✅ | Not required |

## Recall Check — 05-grep-search

Answer these from memory before continuing — no looking back at Demo M05-05.

1. `grep -c 'ERROR' file.log` returns 42; a manual count of every
   occurrence gives 47. What explains this, and what command gives the true
   occurrence count?

2. A recursive `grep` prints "Binary file X matches" and runs slowly.
   What's the fix?

3. Is `grep` alone sufficient to find matching lines, extract a field,
   and replace that field's value across a file?

<details>
<summary>Answers</summary>

1. `-c` counts matching lines, not total occurrences — a line
   containing "ERROR" twice still counts once. `grep -o 'ERROR' file.log | wc
   -l` gives the true occurrence count.

2. Scope the search with `--exclude`/`--exclude-dir` to skip
   binary/compressed files or irrelevant directories.

3. No — `grep` has no replace/transform capability at all; that's
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
where `awk` (Demo M05-08), which handles repeated whitespace as a single field
separator by default, is the better tool — `cut` is the right choice for
genuinely delimiter-consistent data (CSV, `/etc/passwd`-style colon-
separated files), not free-form command output.

**Two smaller `cut` options worth knowing:**

```bash
cut -d',' -f2 --complement <<< 'a,b,c,d'
# a,c,d — --complement inverts the field selection: everything EXCEPT
# field 2, instead of field 2 itself. Useful for "drop this one column"
# without listing every other field explicitly.

cut -d',' -f1,3 --output-delimiter='-' <<< 'a,b,c'
# a-c — the output joins selected fields with '-' instead of the input
# delimiter (comma). Input and output delimiters default to the same
# character, but they're independently controllable.
```

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

The `2,2` in `-k2,2n` means "start at field 2, end at field 2" — a
single-field key. Without the explicit end (`-k2` alone, no comma), `sort`
would use field 2 *through the rest of the line* as the sort key instead of
just that one field — usually harmless for single-value trailing fields,
but worth being deliberate about once a line has more fields after the one
you actually want to sort by.

**Alphabetical sort on numbers produces a wrong-looking but technically
correct result** without `-n`: `"10"` sorts before `"9"` because sort
compares character by character, and `'1'` is a lower character than
`'9'` — this is one of the most common "sort looks broken" moments, and
it isn't broken, it's just doing lexical rather than numeric comparison as
requested.

**A closely related but distinct flag: `-h` (human-numeric), for output
that already carries a size suffix** — exactly the shape `du -h`/`df -h`
(Demo M04-01) produce:

```bash
printf '2K\n500\n1M\n10K\n' > /tmp/sizes.txt
sort /tmp/sizes.txt      # 10K, 1M, 2K, 500 — lexical, wrong order of magnitude
sort -h /tmp/sizes.txt   # 500, 2K, 10K, 1M — correct: understands K/M/G/etc.
```
> Live-verified. `-n` alone doesn't understand the `K`/`M`/`G` suffixes —
> it would sort on the leading digits only, still getting the magnitude
> wrong. `-h` is specifically for this "human-readable size" shape of
> data, distinct from plain `-n`.

```bash
sort -u names.txt
# -u: unique — sort AND deduplicate in one step, rather than piping to a
# separate uniq call (though see §1.3 for when uniq alone is still needed —
# specifically, when you need uniq's -c/-d counting/filtering behavior,
# which sort -u alone doesn't provide).
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
# Demo M05-05 used to break down log lines by backend, and one of the single
# most useful pipeline idioms in this whole demo.

sort names.txt | uniq -d
# -d: show ONLY lines that had duplicates (not every unique line) — useful
# when you specifically want to find what's duplicated, not the full
# deduplicated list.

sort names.txt | uniq -u
# -u: the exact complement of -d — show ONLY lines that appear exactly
# ONCE, with every line that had any duplicate omitted entirely. Useful
# for "what's genuinely one-of-a-kind here," the opposite question from -d.
```

**`-f`/`-s` — skipping fields or characters during comparison, without
modifying the actual output:**

```bash
printf 'Jul 25 09:00:01 host sshd: Accepted publickey for wadmin\nJul 25 09:00:01 host sshd: Accepted publickey for wadmin\nJul 25 09:00:01 host sshd: Accepted publickey for wadmin\nJul 25 09:15:03 host sshd: Accepted publickey for wadmin\nJul 25 09:15:03 host sshd: Accepted publickey for wadmin\n' > /tmp/auth.txt
sort /tmp/auth.txt | uniq -c
```
```
      3 Jul 25 09:00:01 host sshd: Accepted publickey for wadmin
      2 Jul 25 09:15:03 host sshd: Accepted publickey for wadmin
```
```bash
sort /tmp/auth.txt | uniq -f3 -c
```
```
      5 Jul 25 09:00:01 host sshd: Accepted publickey for wadmin
```
> Live-verified. `-f3` tells `uniq` to skip the first 3 *whitespace-
> separated fields* (`Jul`, `25`, `09:00:01`) when deciding whether two
> lines are duplicates — the comparison then only looks at "host sshd:
> Accepted publickey for wadmin" on both lines, which matches, collapsing
> all 5 into a single count. The full original line still prints (from
> whichever line `uniq` kept), fields and all — `-f` only affects
> *comparison*, not output content. `-s N` does the same thing but skips
> the first N *characters* instead of N fields, useful when the varying
> part isn't cleanly field-shaped. This is a more direct fix for "the
> timestamp is making otherwise-identical lines look different" than
> stripping the timestamp with a separate `cut`/`sed` step first — one
> flag on `uniq` itself, rather than an extra pipeline stage. Part 3's
> Break-Fix Incident 2 is exactly this scenario, and its answer now
> reflects both approaches.

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
# -d: delete — remove all characters in the given set entirely, giving
# "helloworld"
```

`tr` operates on a stream of individual characters, with no concept of
lines, fields, or delimiters beyond what you explicitly translate — it's
the right tool for character-level transformation, not the tool for
anything field- or line-structure-aware.

**`tr` accepts the same POSIX named classes Demo M05-01 §1.4 covered for
`grep`** — `[:lower:]`, `[:upper:]`, `[:digit:]`, etc. — with the same
locale-correctness rationale applying here too:

```bash
echo "hello" | tr '[:lower:]' '[:upper:]'
# HELLO — equivalent to tr 'a-z' 'A-Z' but locale-correct, same reasoning
# as Demo M05-01's [[:digit:]] vs [0-9] discussion
```

**`-c` complements the first set — operate on everything NOT listed,
instead of what is:**

```bash
echo "abc123XYZ" | tr -c '[:digit:]' '#'
# ###123#### — every character that ISN'T a digit gets replaced with '#';
# the digits themselves pass through untouched. -c inverts which
# characters -d/the translation actually targets.
```

### 1.5 wc — counting lines, words, characters

```bash
wc -l file.txt      # lines
wc -w file.txt       # words
wc -c file.txt       # bytes
wc -m file.txt       # characters (differs from -c with multi-byte/UTF-8 text)
wc -L file.txt       # length of the LONGEST line, not a count of anything
```

`wc -l` piped from another command is one of the most common idioms in
shell scripting for turning "how many results" into a number a script can
branch on:

```bash
if [ "$(grep -c 'ERROR' app.log)" -gt 100 ]; then
    echo "High error volume"
fi
```

**A detail worth knowing when capturing `wc`'s own output directly:** `wc
-l file.txt` prints the count *and* the filename (`3 file.txt`);
redirecting the file into stdin instead of naming it as an argument — `wc
-l < file.txt` — prints just the bare number, with nothing to strip out
before using it:

```bash
COUNT=$(wc -l < file.txt)
# COUNT is exactly "3", not "3 file.txt" needing further parsing.
```

`-L` is a genuinely different kind of measurement from the others — not a
count of anything, but the character length of whichever single line in
the file is longest. Useful for sizing a display column, or flagging a
file with one suspiciously long line (a common symptom of a malformed
export or an accidentally-unwrapped record) before it causes trouble
downstream.

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

**A quiet default worth knowing before it surprises you: plain `xargs`
runs its command once even when given no input at all.**

```bash
printf '' | xargs echo "ran with:"      # prints "ran with:" — ran anyway
printf '' | xargs -r echo "ran with:"   # prints nothing — correctly skipped
```
> Live-verified. `-r` (`--no-run-if-empty`) suppresses that default
> zero-argument run. This matters for any script where "no input" should
> mean "do nothing," not "run the command with no arguments" — the latter
> can be harmless for something like `echo`, but genuinely risky for a
> command where "run with no arguments" has its own unintended meaning
> (some commands behave very differently called with zero arguments than
> called not at all).

**`-P N` runs up to N invocations concurrently, instead of one at a time:**

```bash
printf '1\n2\n3\n4\n' | xargs -P4 -I{} sh -c 'sleep 0.2; echo done-{}'
```
```
done-4
done-1
done-3
done-2
```
> Live-verified — note the output order isn't 1,2,3,4, confirming the
> four invocations genuinely ran concurrently rather than sequentially.
> Useful for CPU- or network-bound batch work (resizing many images,
> pinging many hosts) where invocations don't depend on each other's
> results — not something to reach for by default, since concurrent
> output naturally interleaves and needs each invocation to be safely
> independent of the others.

### 1.7 tee, paste, column — splitting, merging, formatting output

```bash
grep 'ERROR' app.log | tee errors.txt | wc -l
# tee writes its input to a file AND passes it through unchanged to stdout
# — here, saving the matched lines to errors.txt while still counting them
# in the same pipeline, rather than running the same grep command twice.
```

```bash
tee -a errors.txt
# -a: append to the file instead of overwriting it — the same
# overwrite-vs-append distinction as > vs >> in shell redirection,
# applied to tee specifically.
```

```bash
paste file1.txt file2.txt
# Merges corresponding lines from multiple files side by side, tab-separated
# by default — the inverse operation of cut, in a sense: cut extracts a
# column, paste combines separate files into columns.

paste -d',' file1.txt file2.txt
# -d',': use a comma instead of the default tab as the joining delimiter
```

**`-s` changes `paste`'s whole mode — instead of merging multiple files
side by side, it joins one file's own lines into a single row:**

```bash
printf 'a\nb\nc\n' > /tmp/list.txt
paste -s /tmp/list.txt
# a	b	c   — one line, tab-joined
paste -s -d',' /tmp/list.txt
# a,b,c       — one line, comma-joined
```
> Live-verified. This is the "flatten a multi-line file into one
> delimited line" operation — the reverse of `tr ',' '\n'` from §1.4 —
> useful for turning a list of names or values, one per line, into a
> single CSV-style row.

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
| `cut -d',' -f2` / `--complement` / `--output-delimiter` | delimiter, field, invert, output separator | Extract or exclude fields from delimited text |
| `sort -n` / `-h` / `-k` / `-r` | numeric / human-numeric / key / reverse | Sort by different criteria |
| `uniq -c` / `-d` / `-u` / `-f N` / `-s N` | count / dupes-only / unique-only / skip fields / skip chars | Deduplicate adjacent lines with fine control over comparison |
| `tr 'a-z' 'A-Z'` / `-s` / `-d` / `-c` | translate / squeeze / delete / complement | Character-level transformation |
| `wc -l` / `-w` / `-c` / `-L` | lines / words / bytes / longest line | Count, or measure the longest line |
| `wc -l < file` | (stdin redirect) | Bare count, no filename in output |
| `xargs -0` / `-n` / `-I {}` / `-r` / `-P` | null-safe / batch size / placeholder / skip-if-empty / parallel | Convert stdin into command arguments |
| `tee FILE` / `-a` | — / append | Save output to a file while passing it through |
| `paste -d','` / `-s` | delimiter / serial (one file, joined) | Merge files side by side, or flatten one file into a row |
| `column -t -s','` | table / separator | Format delimited text as an aligned table |

### Task 1 — cut on genuinely delimited data, including complement and output-delimiter

```bash
printf 'alice,30,nyc\nbob,25,la\ncarol,35,sf\n' > /tmp/users.csv
cut -d',' -f1 /tmp/users.csv
cut -d',' -f1,3 /tmp/users.csv
cut -d',' -f2 --complement /tmp/users.csv
# Confirm this returns the name and city fields — everything EXCEPT age
cut -d',' -f1,3 --output-delimiter='|' /tmp/users.csv
# Confirm fields are joined with | instead of the input's comma
```

### Task 2 — sort: alphabetical vs. numeric vs. human-numeric, by field, and reversed

```bash
printf '9\n10\n2\n' > /tmp/nums.txt
sort /tmp/nums.txt        # observe "10" sorting before "2" — lexical, not numeric
sort -n /tmp/nums.txt      # now correctly numeric

printf '2K\n500\n1M\n10K\n' > /tmp/sizes.txt
sort /tmp/sizes.txt        # wrong order of magnitude
sort -h /tmp/sizes.txt     # correct — understands K/M/G suffixes

sort -t',' -k2,2n /tmp/users.csv
# Ascending by age: bob(25), alice(30), carol(35)
sort -t',' -k2,2nr /tmp/users.csv
# nr: numeric AND reverse combined — carol(35), alice(30), bob(25)
```

### Task 3 — uniq: sorted input, counts, duplicates-only, unique-only, and skipping fields

```bash
printf 'apple\nbanana\napple\n' > /tmp/fruit.txt
uniq /tmp/fruit.txt              # apple survives twice — not adjacent
sort /tmp/fruit.txt | uniq        # correctly deduplicated
sort /tmp/fruit.txt | uniq -c     # with counts
sort /tmp/fruit.txt | uniq -d     # only "apple" — the line that had a duplicate

printf 'apple\napple\nbanana\ncherry\ncherry\n' > /tmp/fruit2.txt
sort /tmp/fruit2.txt | uniq -u
# Confirm only "banana" — the one line with no duplicate at all

printf 'Jul 25 09:00:01 host sshd: Accepted publickey for wadmin\nJul 25 09:00:01 host sshd: Accepted publickey for wadmin\nJul 25 09:00:01 host sshd: Accepted publickey for wadmin\nJul 25 09:15:03 host sshd: Accepted publickey for wadmin\nJul 25 09:15:03 host sshd: Accepted publickey for wadmin\n' > /tmp/auth.txt
sort /tmp/auth.txt | uniq -c
# Two separate groups (3 and 2) — the timestamps differ
sort /tmp/auth.txt | uniq -f3 -c
# One combined group (5) — the first 3 fields (timestamp) are skipped
# during comparison, so the conceptually-identical lines now merge
```

### Task 4 — tr for case conversion, squeezing, deletion, POSIX classes, and complement

```bash
echo "Hello World" | tr 'a-z' 'A-Z'
echo "field1   field2     field3" | tr -s ' '
echo "hello123world" | tr -d '0-9'
echo "hello" | tr '[:lower:]' '[:upper:]'
echo "abc123XYZ" | tr -c '[:digit:]' '#'
```

### Task 5 — wc: counting, the argument-vs-redirect output difference, and longest line

```bash
wc -l /tmp/users.csv
wc -w /tmp/users.csv
wc -c /tmp/users.csv
wc -L /tmp/users.csv

# Compare filename-argument output vs. redirected-stdin output:
wc -l /tmp/users.csv        # "3 /tmp/users.csv" — includes the filename
wc -l < /tmp/users.csv      # "3" — bare number only

# The scripting idiom this enables:
COUNT=$(wc -l < /tmp/users.csv)
echo "COUNT is exactly: $COUNT"
if [ "$COUNT" -eq 3 ]; then
    echo "exactly 3 users, as expected"
fi
```

### Task 6 — xargs: unsafe then safe, empty-input behavior, and parallel execution

```bash
mkdir -p /tmp/lab-files
touch /tmp/lab-files/normal.tmp
touch "/tmp/lab-files/has space.tmp"

find /tmp/lab-files -name '*.tmp' | xargs -n1 echo "unsafe would process:"
# Observe "has" and "space.tmp" reported as two separate items — the bug.

find /tmp/lab-files -name '*.tmp' -print0 | xargs -0 -n1 echo "safe processes:"
# Observe "has space.tmp" correctly treated as one item.
rm -rf /tmp/lab-files

printf '' | xargs echo "ran with:"      # runs once anyway — the surprise
printf '' | xargs -r echo "ran with:"   # correctly does nothing

printf '1\n2\n3\n4\n' | xargs -P4 -I{} sh -c 'sleep 0.2; echo done-{}'
# Confirm the output order isn't 1,2,3,4 — proof the four ran concurrently
```

### Task 7 — tee (including append), paste (including serial), and column composed

```bash
printf 'alice\nbob\ncarol\n' > /tmp/names.txt
printf '30\n25\n35\n' > /tmp/ages.txt

paste -d',' /tmp/names.txt /tmp/ages.txt | tee /tmp/combined.csv
column -t -s',' /tmp/combined.csv

echo "extra-line" | tee -a /tmp/combined.csv
cat /tmp/combined.csv
# Confirm the new line was APPENDED, not overwriting the earlier content

paste -s /tmp/names.txt
# Confirm all three names flatten into one tab-joined line
```

## What You Learned

1. ✅ Extracted specific fields or character ranges from lines with `cut`,
   including inverting the selection and controlling the output delimiter
2. ✅ Sorted text numerically, by field, in reverse, and by human-readable
   magnitude with `sort`
3. ✅ Deduplicated and counted repeated lines with `uniq`, including
   duplicates-only (`-d`), unique-only (`-u`), and skipping fields/characters
   during comparison (`-f`/`-s`) without altering the output
4. ✅ Transformed, squeezed, deleted, and complemented characters with `tr`,
   including its POSIX named classes
5. ✅ Counted lines, words, characters, and the longest line with `wc`,
   including the filename-argument vs. stdin-redirect output difference
6. ✅ Converted piped input into command arguments safely with `xargs -0`,
   avoided a spurious run on empty input with `-r`, and ran invocations
   concurrently with `-P`
7. ✅ Combined `tee` (including append), `paste` (including serial mode),
   and `column` for splitting, merging, and formatting output

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
3. What are TWO different fixes that would get the single combined count
   of 5 the teammate expected — one that changes the input before `sort |
   uniq -c`, and one that changes the `uniq` invocation itself?

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
actually does (count of byte-identical lines, timestamp included). Two
valid fixes: **(1) change the input** — strip the timestamp out before the
`sort | uniq -c` step (e.g. with `cut` or `sed`), so the remaining text is
what actually gets deduplicated and counted; or, more directly, **(2)
change the `uniq` invocation itself** — per §1.3, `uniq -f3 -c` skips the
first 3 whitespace-separated fields (`Jul`, `25`, `09:00:01`) during
comparison without needing a separate pipeline stage to strip anything,
producing the single combined count of 5 in one step. Option 2 is
generally the more direct fix when the varying part is cleanly
field-shaped, as it is here; option 1 remains useful when the varying
part isn't a clean leading run of fields (`uniq -s N` handles a
leading-character-count version of the same idea, but a varying field
*in the middle* of the line needs the input-transformation approach instead).

</details>

## Interview Prep

**Q1. `find | xargs rm` is used to clean up temp files, and it occasionally deletes the wrong thing. What's the most likely cause, and the fix?**

Unquoted/default-delimited `xargs` splits input on whitespace, including
spaces within a single filename — a filename containing a space gets
treated as two separate arguments, neither of which is the actual
filename. The fix is the null-delimited pattern: `find ... -print0 | xargs
-0 rm`, since null bytes can't appear in filenames at all, making it the
only delimiter that's safe for genuinely arbitrary filenames.

---

**Q2. `sort file.txt | uniq -c` is expected to combine several conceptually-duplicate lines into one count, but instead produces several separate counts, because of a varying leading timestamp. What's the most direct fix?**

`uniq -f N -c`, where N is the number of leading whitespace-separated
fields the timestamp occupies — this skips those fields during comparison
without altering what's printed, collapsing the conceptually-identical
lines into one count in a single step. A separate `cut`/`sed` stage to
strip the timestamp before `sort | uniq -c` also works, but `uniq -f`
solves it more directly when the varying part is a clean leading run of
fields, which a timestamp usually is.

---

**Q3. Why does `cut -d' '` often produce a mess when applied to typical command output like `ps` or `df`, and what's the better tool for that case?**

`cut` treats every occurrence of the delimiter as a field boundary,
including consecutive/repeated ones — output with irregular spacing (common
in fixed-width command output) produces many spurious empty fields rather
than sensibly collapsing repeated whitespace. `awk` handles repeated
whitespace as a single field separator by default, making it the better
tool for genuinely whitespace-separated, irregularly-spaced output; `cut`
remains the right choice for consistently-delimited data like CSV.

---

**Q4. A script pipes a `find` result into `xargs` to process matching files, but on a run where `find` matches nothing, the target command still executes once — sometimes with unintended effect. What's the fix, and why does plain `xargs` behave this way in the first place?**

Add `-r` (`--no-run-if-empty`) to the `xargs` invocation — without it,
`xargs`'s documented default is to run the target command at least once
even given zero arguments, which is fine for a command like `echo` but can
have real unintended effects for a command whose "run with no arguments"
behavior differs meaningfully from "don't run at all." `-r` makes "no
input" mean "do nothing," which is almost always the intended behavior in
a pipeline like this.

---

## Key Takeaways

1. **`uniq` only removes *adjacent* duplicates — sorting first is not
   optional, it's required for `uniq` to work at all on typical unsorted
   input.** `sort | uniq` (or `sort | uniq -c`) is close to a fixed idiom,
   not two independently-optional steps.
2. **`uniq` merges only byte-identical lines, but `-f N`/`-s N` let you
   skip leading fields or characters during comparison without touching
   what's printed.** A varying timestamp or ID within otherwise-identical
   lines is often more directly handled by `uniq -f`/`-s` than by a
   separate `cut`/`sed` stage before the `sort | uniq -c` step. `-u` is
   `-d`'s exact complement — lines with no duplicate at all, not lines
   that had one.
3. **`xargs` with default delimiters breaks on filenames/arguments
   containing spaces.** The `find -print0 | xargs -0` pattern is the only
   fully safe approach for arbitrary filenames, and this should be a
   default habit in scripts, not a fix applied only after being burned.
   Separately, plain `xargs` runs its command once even on empty input by
   default — `-r` is the fix when "no input" should mean "do nothing."
4. **`sort` without `-n` compares lexically, not numerically** — `"10"`
   sorting before `"2"` is correct lexical behavior, not a bug. `-h`
   (human-numeric) is the distinct flag for data already carrying a size
   suffix (`2K`, `1M`), which `-n` alone doesn't understand. `-k2,2n`
   sorts by a single field specifically; dropping the second number
   (`-k2` alone) extends the key through the rest of the line instead.
5. **`cut` is delimiter-based and not whitespace-smart; `awk` handles
   irregular whitespace as a single separator by default.** Reach for
   `cut` on genuinely consistent delimiters (CSV, colon-separated files),
   and `awk` for command output with variable spacing. `cut --complement`
   inverts field selection; `--output-delimiter` decouples the output
   separator from the input one.
6. **`tee`, `paste`, and `column` solve three different composition
   problems**: `tee` splits one stream into a saved file (`-a` to append
   rather than overwrite) and continued pipeline output; `paste` merges
   separate files into columns, or (with `-s`) flattens one file's own
   lines into a single row; `column` is purely a display formatter, not a
   data transformation.
7. **`wc -l < file` (redirect) prints a bare count; `wc -l file` (argument)
   prints the count with the filename attached.** The redirect form is
   what you want for a clean `$(...)` capture in a script. `-L` is a
   different kind of measurement entirely — the longest line's length,
   not a count of anything.

> **Demo scope:** Primary concept: composable single-purpose text
> processing utilities. Supporting concepts: cut (including complement and
> output-delimiter), sort (including human-numeric), uniq (including
> unique-only and field/character-skipping comparison), tr (including
> POSIX classes and complement), wc (including longest-line and its
> argument-vs-redirect difference), xargs (including null-safety,
> empty-input handling, and parallelism), tee (including append), paste
> (including serial mode), column.
> Estimated completion time: 45 minutes (reading + hands-on + verification).
> Checkpoints: 7 natural stopping points (after Task 1 through Task 7).

## Quick Commands Reference

### Field and character extraction

| Command | Purpose |
|---|---|
| `cut -d',' -f2` | Extract field 2 from comma-delimited text |
| `cut -c1-10` | Extract characters 1–10 of each line |
| `cut -f2 --complement` | Extract every field EXCEPT field 2 |
| `cut --output-delimiter='-'` | Join extracted fields with a different separator |

### Sorting and deduplication

| Command | Purpose |
|---|---|
| `sort -n` | Numeric sort |
| `sort -h` | Human-numeric sort (understands `K`/`M`/`G` suffixes) |
| `sort -k2,2n -t','` | Sort by field 2 only, numerically, comma-delimited |
| `sort -k2,2nr -t','` | Same, but reversed (numeric + reverse combined) |
| `sort -u` | Sort and deduplicate in one step |
| `sort file \| uniq -c` | Count occurrences of each unique line (sort first!) |
| `sort file \| uniq -d` | Show only lines that had duplicates |
| `sort file \| uniq -u` | Show only lines with NO duplicate |
| `sort file \| uniq -f N -c` | Count, skipping the first N fields during comparison |

### Character transformation and counting

| Command | Purpose |
|---|---|
| `tr 'a-z' 'A-Z'` / `tr '[:lower:]' '[:upper:]'` | Translate characters (range or POSIX class) |
| `tr -s ' '` | Squeeze repeated characters into one |
| `tr -d '0-9'` | Delete characters in a set |
| `tr -c '[:digit:]' '#'` | Complement — act on everything NOT in the set |
| `wc -l` / `-w` / `-c` / `-L` | Count lines / words / bytes / longest line length |
| `wc -l < file` | Count lines only (bare number, no filename) |

### Arguments, splitting, merging, formatting

| Command | Purpose |
|---|---|
| `find ... -print0 \| xargs -0 CMD` | Null-safe: convert stdin into arguments |
| `xargs -n1` | One argument per invocation |
| `xargs -I {}` | Placeholder for positioning input in the command |
| `xargs -r` | Don't run the command at all on empty input |
| `xargs -P N` | Run up to N invocations concurrently |
| `CMD \| tee FILE` / `tee -a FILE` | Save output while passing it through / append instead of overwrite |
| `paste -d',' f1 f2` | Merge files side by side |
| `paste -s f1` | Flatten one file's lines into a single row |
| `column -t -s','` | Format delimited text as an aligned table |

## Appendix A — Anki Cards

**`m05-06-text-processing-utilities-anki.csv`:**

```
#deck:Linux Mastery::Module 5 - Text Processing & Search Tools::Demo M05-06 - Text Processing Utilities
#separator:Comma
#columns:Front,Back,Tags
"uniq is run on an unsorted file containing 'apple, banana, apple' and doesn't remove the duplicate apple. Why?","uniq only removes ADJACENT duplicate lines, not duplicates anywhere in the file. Since the two 'apple' lines aren't next to each other, uniq sees no duplicates to remove — sort first to make duplicates adjacent.","demo-m05-06,uniq,common-mistake"
"sort file | uniq -c shows two separate counts for log lines that are conceptually the same event but have different timestamps. What's the most DIRECT fix?","uniq -f N -c, where N is the number of leading whitespace-separated fields the timestamp occupies — this skips those fields during comparison without altering the printed output, more direct than a separate cut/sed step to strip the timestamp first.","demo-m05-06,uniq,fields,fix"
"What's the difference between uniq -d and uniq -u?","-d shows ONLY lines that had at least one duplicate (the deduplicated list of what repeated). -u shows the exact opposite — only lines that appear exactly ONCE, with any line that had a duplicate omitted entirely.","demo-m05-06,uniq,comparison"
"sort file.txt (without -n) puts '10' before '2'. Is this a bug?","No — without -n, sort compares lexically (character by character), and '1' is a lower character than '2', so '10' correctly sorts before '2' alphabetically. -n enables numeric comparison instead.","demo-m05-06,sort,common-mistake"
"You have a file of sizes like '2K', '500', '1M', '10K'. Does sort -n put these in correct magnitude order?","No — -n doesn't understand K/M/G suffixes, only leading digits. sort -h (human-numeric) is the correct flag for this shape of data, understanding the size suffixes and sorting by actual magnitude.","demo-m05-06,sort,human-numeric"
"find /tmp -name '*.tmp' | xargs rm deletes the wrong files when a filename contains a space. Why, and what's the fix?","Default xargs splits input on whitespace, so a filename like 'my file.tmp' becomes two separate arguments neither of which is the real file. Fix: find ... -print0 | xargs -0 rm — null-byte delimiting is safe because null can't appear in a filename.","demo-m05-06,xargs,safety,common-mistake"
"Does plain xargs run its target command if given completely empty input?","Yes — by documented default, xargs runs the command at least once even with zero arguments, which can have unintended effects for some commands. -r (--no-run-if-empty) suppresses this, making 'no input' correctly mean 'do nothing'.","demo-m05-06,xargs,empty-input"
"What does xargs -P4 do differently from plain xargs?","Runs up to 4 invocations of the target command CONCURRENTLY instead of one at a time — useful for independent, CPU- or network-bound batch work, confirmed by output arriving out of input order when genuinely parallel.","demo-m05-06,xargs,parallel"
"Why is cut often a poor fit for parsing typical command output like ps or df, compared to awk?","cut is delimiter-based and treats every occurrence of the delimiter (e.g. every space) as a field boundary, including repeated ones — irregular spacing produces many spurious empty fields. awk collapses repeated whitespace into a single separator by default.","demo-m05-06,cut,awk,comparison"
"What does cut -f2 --complement do differently from cut -f2?","--complement inverts the selection — it returns every field EXCEPT field 2, rather than field 2 itself. Useful for 'drop this one column' without listing every other field explicitly.","demo-m05-06,cut,complement"
"What does tr -c do?","Complements the first character set — the translation/deletion applies to everything NOT in the listed set, instead of what IS in it. E.g. tr -c '[:digit:]' '#' replaces every non-digit character with #, leaving digits untouched.","demo-m05-06,tr,complement"
"Does tr support the same POSIX named character classes ([:lower:], [:digit:], etc.) that grep does?","Yes — tr '[:lower:]' '[:upper:]' works the same way as grep's [[:lower:]], for the same locale-correctness reasons covered for grep in Demo M05-01.","demo-m05-06,tr,posix-classes,cross-demo"
"What does wc -L measure, as opposed to wc -l/-w/-c?","The character length of the single LONGEST line in the file — not a count of anything (lines/words/bytes), a genuinely different kind of measurement. Useful for sizing display columns or flagging a suspiciously long line.","demo-m05-06,wc,longest-line"
"What's the difference between paste (default, multiple files) and paste -s (one file)?","Default paste merges corresponding lines from MULTIPLE files side by side into columns. paste -s instead flattens ONE file's own multiple lines into a SINGLE row — the reverse operation of tr ',' '\\n'.","demo-m05-06,paste,serial"
"What does tee -a do differently from plain tee?","Appends to the target file instead of overwriting it — the same overwrite-vs-append distinction as shell's > vs >>, applied to tee specifically.","demo-m05-06,tee,append"
"What's the practical idiom for turning grep's matched-line output into a count of unique values in a specific field?","grep 'pattern' file | grep -o 'field=value-pattern' | sort | uniq -c — extract the field, then sort (required for uniq to work) and count with uniq -c.","demo-m05-06,workflow,cross-demo"
```

## Appendix B — Quiz

**`m05-06-text-processing-utilities-quiz.md`:**

````markdown
# Quiz — Demo M05-06: Text Processing Utilities

> One correct answer per question unless stated otherwise.
> Target: 80% or above before moving to Demo M05-07.

---

**Q1.** `sort -t',' -k2,2n users.csv` is used instead of `sort -t',' -k2n users.csv` (no second number). What does the `,2` specifically restrict the sort key to?

- A) Nothing — `-k2` and `-k2,2` are exactly equivalent in every case
- B) `-k2,2` restricts the sort key to field 2 only; `-k2` alone (with no end field specified) would extend the key through the rest of the line instead
- C) `-k2,2` sorts only the first 2 characters of field 2
- D) `-k2,2` is invalid syntax and would produce an error

<details>
<summary>Answer</summary>

**B** — The comma-separated range in a `-k` spec defines the start and end of the sort key; omitting the end field means the key runs to the end of the line, not just that one field. For a line with more fields after the target one, this distinction can change the sort outcome.

Trap: A wrongly claims equivalence in every case; C misreads the numbers as character positions rather than field positions; D invents an error that doesn't occur — this is valid, meaningful syntax.

</details>

---

**Q2.** A file contains size values like `2K`, `500`, `1M`, `10K`. Does `sort -n` put them in correct magnitude order?

- A) Yes — `-n` fully understands size suffixes like K/M/G
- B) No — `-n` only reads leading digits, not size suffixes; `sort -h` is the correct flag for this shape of data, understanding K/M/G and sorting by actual magnitude
- C) No — no `sort` flag can handle this; the suffixes must be stripped manually first
- D) Yes, but only if the suffixes are lowercase

<details>
<summary>Answer</summary>

**B** — `-n` and `-h` are distinct: `-n` is plain numeric comparison, blind to unit suffixes; `-h` specifically understands human-readable size notation and sorts by real magnitude.

Trap: A overstates `-n`'s capability; C wrongly claims no built-in solution exists when `-h` is exactly that; D invents a case-sensitivity restriction that doesn't apply.

</details>

---

**Q3.** `sort auth.log | uniq -c` produces two separate counts (3 and 2) for log lines that represent the same conceptual event but have different leading timestamps. What is the most DIRECT fix, without adding a separate pipeline stage?

- A) Re-run `sort` a second time before `uniq`
- B) `uniq -f 3 -c` — skip the first 3 whitespace-separated fields (the timestamp) during comparison, collapsing the lines into one count without needing to strip anything beforehand
- C) `uniq -c` has no way to handle this; a separate `cut`/`sed` step is always required
- D) Use `sort -u` instead of `sort | uniq -c`

<details>
<summary>Answer</summary()>

**B** — `uniq -f N` (or `-s N` for a leading character count) skips fields during comparison without altering the actual printed output, solving exactly this "varying leading field" problem in one step.

Trap: A invents an ineffective double-sort that doesn't address the actual issue; C understates `uniq`'s own capability, when `-f` handles this directly; D confuses `sort -u` (plain dedup, no counting at all) with the counting behavior `uniq -c` provides.

</details>

---

**Q4.** `sort names.txt | uniq -u` is run on a file with several unique names and a few that repeat. How does this differ from `sort names.txt | uniq -d`?

- A) They produce identical output
- B) `-u` shows only names that appear exactly ONCE (no duplicate at all); `-d` shows only names that DID have at least one duplicate — the exact opposite selection
- C) `-u` shows counts; `-d` does not
- D) `-u` requires the input to already be deduplicated; `-d` does not

<details>
<summary>Answer</summary>

**B** — `-d` and `-u` are complements of each other: one surfaces what repeated, the other surfaces what didn't, and together they partition the input into two disjoint sets.

Trap: A wrongly claims equivalence; C confuses counting (`-c`) with either filtering flag, neither of which prints counts; D invents an input-precondition that isn't accurate — both operate on already-sorted input the same way.

</details>

---

**Q5.** `echo "abc123XYZ" | tr -c '[:digit:]' '#'` is run. What does `-c` do here, and what's the output?

- A) `-c` counts the digits; output is a number
- B) `-c` complements the set — everything NOT a digit gets replaced with `#`, while digits pass through untouched; output is `###123####`
- C) `-c` converts the string to lowercase first
- D) `-c` has no effect when combined with a POSIX class like `[:digit:]`

<details>
<summary>Answer</summary>

**B** — `-c` inverts which characters the translation/deletion targets — here, everything that ISN'T a digit is what gets replaced, leaving the digits themselves untouched.

Trap: A confuses `tr`'s `-c` with an unrelated counting operation; C invents a case-conversion side effect; D wrongly claims no effect, when `-c` fundamentally changes which characters are targeted.

</details>

---

**Q6.** `printf '' | xargs echo "ran with:"` is compared to `printf '' | xargs -r echo "ran with:"`. What's the difference in behavior?

- A) Both produce identical output — `-r` has no effect on empty input
- B) Without `-r`, `xargs` runs the command once anyway even with zero arguments (documented default behavior); with `-r`, it correctly does nothing when given no input
- C) Without `-r`, `xargs` errors out; with `-r`, it runs successfully
- D) `-r` only affects behavior when combined with `-0`

<details>
<summary>Answer</summary>

**B** — This is `xargs`'s documented default: it runs the target command at least once even with no arguments, unless `-r`/`--no-run-if-empty` is given to suppress that behavior.

Trap: A wrongly claims no difference; C reverses which case actually errors (neither does, by default); D invents a dependency on `-0` that doesn't exist.

</details>

---

**Q7.** `paste file1.txt file2.txt` (multiple files) and `paste -s file1.txt` (one file, `-s`) are both run. How does `-s` change what `paste` does?

- A) `-s` sorts the file's lines before merging
- B) Default `paste` merges corresponding lines from MULTIPLE files side by side into columns; `-s` instead flattens ONE file's own multiple lines into a single row
- C) `-s` has no meaningful effect when only one file is given
- D) `-s` silences all output except the first line

<details>
<summary>Answer</summary>

**B** — `-s` fundamentally changes `paste`'s mode of operation: from merging across files to flattening within one file — the reverse of turning a delimiter into newlines (`tr ',' '\n'`).

Trap: A invents an unrelated sorting behavior; C understates `-s`'s real, distinct effect; D invents a silencing behavior that isn't what `-s` does.

</details>

---

**Q8.** `wc -L file.txt` is run. What does this report, compared to `wc -l`, `-w`, or `-c`?

- A) The number of lowercase letters in the file
- B) The character length of the single longest line in the file — a fundamentally different measurement from a count of lines, words, or bytes
- C) The number of "L" characters in the file
- D) The average line length across the whole file

<details>
<summary>Answer</summary>

**B** — `-L` reports a length (of the longest line specifically), not a count of anything — genuinely different in kind from `-l`/`-w`/`-c`, which all report counts of some unit.

Trap: A and C both misread the flag letter as referring to a specific character; D invents an averaging calculation `wc` doesn't perform — it reports the maximum, not a mean.

</details>

---

Score guide:

| Score | Action |
|---|---|
| 7/8 or 8/8 | Import Anki cards, move to Demo M05-07 |
| 6/8 | Review the wrong answers, then proceed |
| 5/8 | Re-read the relevant sections, retry those questions |
| Below 5/8 | Re-read the full demo and redo the walkthrough before proceeding |
````