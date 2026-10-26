# Demo M05-07 — sed Editing

## Overview

Demo M05-05's `grep` finds; this demo's `sed` changes. `sed` ("stream editor")
reads text line by line and applies editing commands to it — substitution
is the command you'll use 90% of the time, but `sed`'s addressing model and
its hold-space mechanism let it do genuinely programmatic text
transformation, not just find-and-replace. By the end, you'll write
confident one-liners and understand exactly when `sed` is the right tool
versus when it's being stretched past what it's built for.

## Demo Objectives

1. Perform substitution with `s///`, including flags for global,
   case-insensitive, and Nth-occurrence-only replacement, and reference the
   whole match (`&`) or captured groups (`\1`, `\2`) in the replacement text
2. Address specific lines or ranges of lines for a command to apply to,
   including inserting, appending, or replacing whole lines (`a`/`i`/`c`)
   and stopping processing early (`q`)
3. Explain pattern space and hold space, and use hold space hands-on to
   accumulate and reorder content across lines
4. Edit files in place safely, with and without a backup
5. Chain multiple sed commands into a script, both inline and as a
   reusable script file
6. Recognize when a task has outgrown sed and belongs in awk instead

## Directory Structure

```
07-sed-editing/
├── README.md                    # this file
├── m05-07-sed-editing-anki.csv  # Anki flashcard deck
└── m05-07-sed-editing-quiz.md   # standalone quiz
```

## Environments

| Task | Ubuntu 24.04 | Rocky Linux 9.8 | EC2 Amazon Linux |
|---|---|---|---|
| All lab tasks | ✅ (GNU sed) | ✅ (GNU sed) | Not required |

## Recall Check — 06-text-processing-utilities

Answer these from memory before continuing — no looking back at Demo M05-06.

1. `uniq` is run on an unsorted file with a duplicate line that isn't
   adjacent to its twin. What happens, and why?

2. `find /tmp -name '*.tmp' | xargs rm` occasionally deletes the wrong
   file. What's the most likely cause, and the safe fix?

3. Why is `cut -d' '` often a poor choice for parsing output like `ps`
   or `df`, compared to `awk`?

<details>
<summary>Answers</summary>

1. It survives — `uniq` only removes *adjacent* duplicate lines, with
   no concept of duplicates elsewhere in the file. Sorting first makes
   duplicates adjacent, which is what `uniq` actually needs.

2. A filename contains a space, and default `xargs` splits on
   whitespace, breaking one filename into multiple arguments. Fix:
   `find ... -print0 | xargs -0 rm`.

3. `cut` treats every occurrence of the delimiter as a boundary,
   including repeated ones — irregular spacing produces many spurious empty
   fields. `awk` collapses repeated whitespace into a single separator by
   default.

</details>

## Part 1 — Concepts

### 1.1 Substitution — s/// and its flags

```bash
echo "hello world" | sed 's/world/there/'
# hello there — replaces the FIRST match on each line only, by default

echo "aaa" | sed 's/a/b/'
# baa — only the first 'a' replaced, not all of them
echo "aaa" | sed 's/a/b/g'
# bbb — g flag: global, replace every match on the line, not just the first

echo "Hello World" | sed 's/hello/hi/i'
# hi World — i flag: case-insensitive matching (GNU sed extension — not
# guaranteed portable to every sed, the same category of caveat Demo M05-01
# flagged for BRE's \| and \w)

echo "aaa" | sed 's/a/b/2'
# aba — a number instead of g replaces only that specific occurrence
# (the 2nd match), not all or just the first
```

**`sed` uses BRE by default, same as `grep` without `-E`** (Demo M05-01's
flavor table) — `sed -E` (or `-r` on some systems) switches to ERE, exactly
the same distinction and the same reason to be deliberate about which one a
pattern was written for.

```bash
echo "path/to/file" | sed 's/\//-/g'
# path-to-file — the delimiter (/) needs escaping when it also appears in
# the pattern or replacement, which gets visually noisy fast.
echo "path/to/file" | sed 's#/#-#g'
# Same result, using # as the delimiter instead of / — sed lets you choose
# any character as the delimiter, specifically to avoid this escaping mess
# when the pattern itself contains slashes (common with file paths).
```

**`&` in the replacement text means "the whole matched text" — reuse what
was found, without retyping it:**

```bash
echo "hello" | sed 's/hello/[&]/'
```
```
[hello]
```
> Live-verified. `&` is replaced with whatever the pattern actually
> matched on that line — here, "hello" itself, wrapped in brackets by
> the replacement. This is the simplest way to "decorate" a match
> (bracket it, quote it, prefix/suffix it) without writing the matched
> text out a second time in the replacement.

**If a literal `&` is actually needed in the replacement (not "the whole
match"), it must be escaped as `\&`:**

```bash
echo "price 100" | sed 's/100/\&99/'
```
```
price &99
```
> Live-verified — `\&` produces a literal ampersand character; an
> unescaped `&` here would instead have inserted "100" again (the match),
> giving the wrong result "price 10099".

**Captured groups from the pattern — `\1`, `\2`, etc. — can be referenced
in the replacement, not just within the pattern itself (Demo M05-01's
backreferences were pattern-internal; this is the same capturing
mechanism, used across the `s///`'s two halves instead):**

```bash
echo "John Smith" | sed 's/\(.*\) \(.*\)/\2, \1/'
```
```
Smith, John
```
> Live-verified. `\(.*\)` and `\(.*\)` (BRE grouping, per Demo M05-01 §1.2)
> capture "John" and "Smith" as groups 1 and 2; the replacement `\2, \1`
> reorders them. This is arguably the single most practically useful
> `sed` capability beyond plain substitution — reordering fields,
> reformatting structured text — and it's a different use of grouping
> from Demo M05-01's backreference-within-a-pattern (`(\w+) \1`, matching a
> repeated word): here the capture happens once in the pattern and gets
> *placed* somewhere new in the replacement, rather than requiring the
> same text to repeat within the match itself.

```bash
echo "2026-07-25" | sed 's/\([0-9]*\)-\([0-9]*\)-\([0-9]*\)/\3\/\2\/\1/'
```
```
25/07/2026
```
> Live-verified — a practical date-reformatting example: three captured
> groups (year, month, day) reassembled in a different order and with a
> different separator, entirely via `\1`/`\2`/`\3` in the replacement.

### 1.2 Addressing — which lines a command applies to

Without an address, a `sed` command applies to every line. Addresses
restrict that:

```bash
sed '3s/foo/bar/' file.txt
# Only line 3

sed '2,5s/foo/bar/' file.txt
# Lines 2 through 5 inclusive

sed '/START/,/END/s/foo/bar/' file.txt
# Every line between one matching /START/ and the next matching /END/,
# INCLUSIVE of both the START and END lines themselves — pattern-based
# ranges, not just numeric ones

sed '$s/foo/bar/' file.txt
# $ : the last line, whatever line number that actually is

sed '/^#/d' file.txt
# d: delete matching lines entirely (not a substitution at all) — every
# line starting with # is removed from the output
```

```bash
sed -n '2,4p' file.txt
# -n: suppress sed's normal automatic printing of every line
# p: explicitly print — combined, this prints ONLY lines 2-4, nothing else.
# Without -n, this would print every line normally AND print 2-4 an extra
# time, since p adds an explicit print on top of the default behavior.
```

**`-n` combined with `p` is the correct pattern for "extract specific
lines," and forgetting `-n` is a common source of unexpectedly duplicated
output** — every line still prints once by default, and `p` adds an
additional print for the matched lines specifically.

**`d` deletes whole lines; three siblings insert or replace them instead
— `a` (append), `i` (insert), and `c` (change):**

```bash
printf 'line1\nline2\nline3\n' > /tmp/sed-lines.txt
sed '2a\inserted-after' /tmp/sed-lines.txt    # new line AFTER line 2
sed '2i\inserted-before' /tmp/sed-lines.txt   # new line BEFORE line 2
sed '2c\replaced-line' /tmp/sed-lines.txt     # line 2 REPLACED entirely
```
```
-- a (append after) --
line1
line2
inserted-after
line3
-- i (insert before) --
line1
inserted-before
line2
line3
-- c (change/replace) --
line1
replaced-line
line3
```
> Live-verified. All three take an address (a line number, a range, or a
> pattern) exactly the way `d` does — the difference is what they do at
> that address: `a` adds a new line after it, `i` adds one before it, `c`
> replaces the addressed line(s) entirely rather than editing their
> existing content via `s///`. Useful for structural edits — adding a new
> config directive, inserting a header, replacing an entire line whose
> content doesn't lend itself to a targeted substitution.

**`q` stops processing entirely once reached — a `head`-like use of
`sed`:**

```bash
printf '1\n2\n3\n4\n5\n' | sed '3q'
```
```
1
2
3
```
> Live-verified — lines 1 through 3 print normally, then `sed` quits the
> moment line 3's cycle finishes, never reading (or processing) lines 4
> and 5 at all. This isn't just "print only the first 3 lines" (which
> `-n '1,3p'` would also achieve) — `q` genuinely stops the whole `sed`
> process early, which matters for performance on a very large input
> where you only need an early portion and don't want `sed` to keep
> reading the rest of the file at all.

### 1.3 Pattern space and hold space

Every `sed` command operates on the **pattern space** — the current line
being processed, one at a time by default. The **hold space** is a
separate, persistent buffer that survives across lines, letting `sed`
remember something from an earlier line and use it later:

```bash
sed '1!G;h;$!d' file.txt
```

This one-liner reverses the entire file (`sed`'s equivalent of `tac`).
Reading it in execution order, per line: `h` always copies the current
line into hold space, overwriting whatever was there; `1!G` (every line
*except* the first) runs *before* that copy, appending whatever hold space
currently holds onto the end of the new line — since hold space at that
point still contains everything accumulated so far (already in reversed
order), the new line effectively lands in front of it. By the time the
last line is processed, this accumulation holds the entire file in
reverse; `$!d` suppresses printing on every line except the last, so only
that final, fully-reversed accumulation actually gets printed, via sed's
normal automatic print at the end of the last line's cycle.

`h`/`H` and `g`/`G` are the four hold-space commands worth knowing by
name: lowercase `h`/`g` **overwrite** (hold space or pattern space,
respectively); uppercase `H`/`G` **append**, adding a newline and the new
content rather than replacing what's already there. This one-liner uses
both patterns deliberately — `h` overwrites hold space fresh each time
(since the running accumulation is about to be rebuilt with the new line
in front anyway), while `G` appends onto pattern space, since that's where
the growing reversed result is actually being assembled.

**The practical takeaway isn't memorizing that specific one-liner** — it's
recognizing that hold space is what makes `sed` capable of anything beyond
"one line in, one line out" transformations: multi-line context,
accumulation across lines, or lookback/lookahead logic that a pure
line-by-line substitution can't express. Most everyday `sed` usage never
touches hold space at all — reaching for it is a signal the task has grown
past simple substitution.

### 1.4 In-place editing — with and without a backup

```bash
sed -i 's/foo/bar/g' file.txt
# -i: edit the file in place, overwriting it directly — no output printed
# to stdout at all.

sed -i.bak 's/foo/bar/g' file.txt
# -i.bak: same, but first saves the ORIGINAL content to file.txt.bak before
# overwriting — the suffix is appended to the existing filename, not used
# as a prefix or a separate directory; file.txt becomes file.txt.bak, not
# bak.file.txt or a copy elsewhere. This is a safety net specifically
# because -i alone gives you no way to recover the original if the
# substitution was wrong.
```

**`-i` with no backup suffix is genuinely destructive and irreversible** —
there's no "undo" once it runs, unlike a piped `sed` without `-i`, which
only ever affects what's printed to stdout, leaving the source file
untouched. The habit worth building: test a `sed` command without `-i`
first (confirm the output looks right), *then* add `-i` once confirmed —
not the other way around.

```bash
sed -i 's/foo/bar/g' *.conf
# Applies to every matching file — the destructive-without-testing-first
# risk multiplies across every file matched by the glob.
```

### 1.5 Chaining commands into a sed script

```bash
sed -e 's/foo/bar/' -e 's/baz/qux/' file.txt
# -e: multiple expressions, applied in sequence on each line
sed 's/foo/bar/; s/baz/qux/' file.txt
# Equivalent, semicolon-separated in a single -e (or the implicit one)
```

```bash
cat > script.sed << 'EOF'
# lines starting with # are comments, same convention as shell scripts
s/foo/bar/
s/baz/qux/
/^#/d
EOF
sed -f script.sed file.txt
# -f: read commands from a script file instead of the command line —
# worth doing once a transformation grows past 2-3 simple substitutions,
# for readability and reuse. Prefer -f over a long chain of -e flags once
# the script is something you'd want to save, version, or hand to someone
# else, rather than a genuinely one-off command.
```

### 1.6 When a task has outgrown sed

`sed` is a line-oriented text transformer with limited field awareness and
no arithmetic — the moment a task needs to reference "the third
whitespace-separated value on this line" by position, sum values across
lines, or do conditional logic based on a field's actual value (not just
its text pattern), `awk` (Demo M05-08) is almost always the better tool.
`sed`'s hold space *can* technically express some of that with enough
one-liner cleverness, but at real cost to readability — if you're reaching
for hold space to do something field- or arithmetic-related, that's usually
the signal to switch tools rather than push `sed` further.

## Part 2 — Hands-On Lab

### Commands used in this demo

| Command | Flag meaning | Purpose |
|---|---|---|
| `sed 's/PAT/REPL/g'` | `g`: global | Replace every match on each line |
| `sed 's/PAT/REPL/i'` | `i`: case-insensitive | GNU extension |
| `sed 's/PAT/REPL/N'` | `N`: Nth match only | Replace only one specific occurrence |
| `sed 's/PAT/[&]/'` | `&`: whole match | Reuse the matched text in the replacement |
| `sed 's/\(a\)\(b\)/\2\1/'` | `\1`/`\2`: captured groups | Reorder captured content in the replacement |
| `sed -n 'N,Mp'` | `-n`+`p`: suppress default, explicit print | Extract specific lines only |
| `sed '/PAT/d'` | `d`: delete | Remove matching lines |
| `sed 'Na\text'` / `'Ni\text'` / `'Nc\text'` | append / insert / change | Add or replace whole lines |
| `sed 'Nq'` | `q`: quit | Stop processing after line N |
| `sed -i.bak 's/PAT/REPL/g'` | `-i.bak`: in-place with backup | Edit a file directly, safely |
| `sed '1!G;h;$!d'` | hold space | Reverse a file's line order |
| `sed -f script.sed` | `-f`: script file | Run multiple commands from a file |

### Task 1 — Basic substitution, delimiter choice, and the i/N flags

```bash
printf 'the quick brown fox\n' > /tmp/sed-lab.txt
sed 's/quick/slow/' /tmp/sed-lab.txt
sed 's/o/0/g' /tmp/sed-lab.txt

echo "Hello World" | sed 's/hello/hi/i'
# Confirm the case-insensitive match works despite the case difference
# between the pattern and the actual text.

echo "aaa" | sed 's/a/b/2'
# Confirm only the 2nd 'a' is replaced: "aba"

printf '/var/log/app.log\n' > /tmp/sed-lab2.txt
sed 's/\//-/g' /tmp/sed-lab2.txt
sed 's#/#-#g' /tmp/sed-lab2.txt
```

### Task 2 — Whole-match (&) and captured-group (\1, \2) replacement

```bash
echo "hello" | sed 's/hello/[&]/'
# Confirm "hello" reappears inside brackets, without retyping it

echo "price 100" | sed 's/100/\&99/'
# Confirm the output is "price &99" — an actual literal ampersand,
# not the matched "100" reinserted

echo "John Smith" | sed 's/\(.*\) \(.*\)/\2, \1/'
# Confirm the output is "Smith, John" — captured groups reordered

echo "2026-07-25" | sed 's/\([0-9]*\)-\([0-9]*\)-\([0-9]*\)/\3\/\2\/\1/'
# Confirm the output is "25/07/2026" — a practical date reformat
```

### Task 3 — Addressing: line numbers, ranges, patterns, and a/i/c/q

```bash
printf 'one\ntwo\nthree\nfour\nfive\n' > /tmp/sed-lab3.txt
sed '3s/^/>>> /' /tmp/sed-lab3.txt
sed '2,4s/^/>>> /' /tmp/sed-lab3.txt
sed -n '2,4p' /tmp/sed-lab3.txt

printf 'text\nSTART\nkeep this\nEND\ntext\n' > /tmp/sed-lab4.txt
sed -n '/START/,/END/p' /tmp/sed-lab4.txt
# Confirm the output includes the START and END lines themselves, not just
# the content between them — the range is inclusive of both endpoints.

sed '2a\inserted-after' /tmp/sed-lab3.txt
sed '2i\inserted-before' /tmp/sed-lab3.txt
sed '2c\replaced-line' /tmp/sed-lab3.txt
sed '3q' /tmp/sed-lab3.txt
# Confirm this stops after line 3, never showing "four" or "five" at all
```

### Task 4 — Delete matching lines

```bash
printf '# comment\nreal line 1\n# another comment\nreal line 2\n' > /tmp/sed-lab5.txt
sed '/^#/d' /tmp/sed-lab5.txt
```

### Task 5 — In-place editing, test-first workflow

```bash
printf 'debug=true\nport=8080\n' > /tmp/sed-lab6.conf
sed 's/debug=true/debug=false/' /tmp/sed-lab6.conf
# Confirm the output looks right BEFORE adding -i:
sed -i.bak 's/debug=true/debug=false/' /tmp/sed-lab6.conf
cat /tmp/sed-lab6.conf
cat /tmp/sed-lab6.conf.bak
```

### Task 6 — Chaining commands: inline vs. a script file

```bash
printf 'foo=1\nbaz=2\n# remove me\n' > /tmp/sed-lab7.txt
sed -e 's/foo/FOO/' -e 's/baz/BAZ/' -e '/^#/d' /tmp/sed-lab7.txt

# The same transformation, as a reusable script file instead of inline
# -e flags — worth doing once a transformation is something you'd want to
# save, version, or hand to someone else:
cat > /tmp/sed-lab7.sed << 'EOF'
s/foo/FOO/
s/baz/BAZ/
/^#/d
EOF
sed -f /tmp/sed-lab7.sed /tmp/sed-lab7.txt
# Confirm identical output to the -e version above.
```

### Task 7 — Hold space, hands-on: reversing a file

```bash
printf 'a\nb\nc\n' | sed -n '$p'
# $ addresses only the last line — confirms you understand $ as "last
# line" before the more elaborate hold-space example below.

printf 'first\nsecond\nthird\n' > /tmp/sed-lab8.txt
sed '1!G;h;$!d' /tmp/sed-lab8.txt
# Confirm the output is "third", "second", "first" — the file genuinely
# reversed, using hold space to accumulate lines in reverse order across
# the whole file, not something a single-line substitution could express.
```

## What You Learned

1. ✅ Performed substitution with `s///`, including `g`, `i`, and
   Nth-occurrence flags, and reused the whole match (`&`) or reordered
   captured groups (`\1`, `\2`) in the replacement text
2. ✅ Addressed specific lines or ranges — numeric, pattern-based, and `$`,
   confirming pattern ranges are inclusive of both endpoints — and used
   `a`/`i`/`c` to insert, append, or replace whole lines, and `q` to stop
   processing early
3. ✅ Explained pattern space and hold space, and used hold space hands-on
   to reverse a file's line order
4. ✅ Edited files in place safely, testing without `-i` first
5. ✅ Chained multiple sed commands with `-e` and a reusable script file (`-f`)
6. ✅ Recognized when a task has outgrown sed and belongs in awk

## Part 3 — Break-Fix Scenario

### Scenario — NovaTech Ops: two sed incidents

> **Incident report — NovaTech Ops, ticket #4667**
>
> Alert 1: "a config-update script ran `sed -i` across every `.conf` file
> in a directory and now several services won't start — no backups exist."
> Alert 2: "a `sed -n 'Np'` command meant to extract one line is printing
> that line twice."

**Incident 1 — sed -i with no backup, services broken**

```bash
$ sed -i 's/max_connections=100/max_connections=500/' *.conf
$ # several services fail to start after this
```

Tasks:
1. Reference §1.4 — what safety step was skipped here, and what would it
   have provided if something went wrong?
2. Given no backup exists, what's the actual recovery path now (this demo's
   tools alone can't undo a completed `-i` edit)?
3. What single change to the original command would have made recovery
   trivial instead of requiring an external backup source?

**Incident 2 — sed -n 'Np' printing a line twice**

```bash
$ sed '5p' access.log | head -1
$ # (teammate expected just line 5 once)
```

Tasks:
1. Reference §1.2 — what's missing from this command compared to the
   correct pattern for extracting a specific line?
2. Explain exactly why the current command prints line 5 twice while every
   other line prints once — walk through what `p` and sed's default
   behavior each contribute.
3. Fix it.

<details>
<summary>Answers</summary>

**Incident 1:** The `.bak` backup suffix (`-i.bak` instead of bare `-i`) was
skipped — per §1.4, `-i` alone is genuinely destructive and irreversible,
overwriting the original with no recovery path built into the command
itself. With no backup, recovery now depends entirely on something outside
`sed`'s own capability — a version control system (if the configs are
tracked in git), a separate system backup, or manual reconstruction from
memory/documentation of the correct values; there's no `sed`-level undo
available at this point. The single change that would have made this
trivial: `sed -i.bak '...' *.conf` instead of `sed -i '...' *.conf` — the
`.bak` suffix would have preserved every original file as `filename.conf.bak`
before any modification, making recovery a simple restore rather than a
crisis.

**Incident 2:** The command is missing `-n`. Per §1.2, `sed` prints every
line once by default; `p` (print) is an *additional*, explicit print
command layered on top of that default behavior, not a replacement for it.
Without `-n` suppressing the default output, line 5 gets its normal
automatic print AND an extra print triggered by the `5p` address+command —
two total prints of that specific line, while every other line only gets
its one normal default print (which is why only line 5 shows the doubling,
not the whole file). Fix: `sed -n '5p' access.log` — `-n` suppresses the
default printing entirely, leaving only the explicit `p` output, which is
exactly one print of line 5 and nothing else.

</details>

## Interview Prep

**Q1. Why is `sed -i` (with no backup suffix) considered risky enough to warrant a specific habit around it, rather than just "being careful"?**

Because it's genuinely irreversible at the point it runs — there's no
`sed`-level undo, and the original content is gone the instant the command
completes, unlike a piped `sed` (no `-i`) which only ever affects stdout
and leaves the source file untouched. The safer habit is testing the
substitution without `-i` first to visually confirm correctness, then
re-running with `-i.bak` (not bare `-i`) so a recovery path exists if
something was still wrong despite the visual check.

---

**Q2. A `sed -n '5p' file.txt` command is expected to print line 5 once. What would happen if `-n` were accidentally omitted, and why?**

Line 5 would print twice — once from `sed`'s normal default behavior of
printing every line automatically, and a second time from the explicit `p`
command, which adds a print rather than replacing the default one. Every
other line in the file would still print exactly once (their normal
default print, with no `p` command applying to them), so the doubling would
be specific and isolated to line 5, which is often the confusing part of
this particular mistake.

---

**Q3. A teammate needs to swap "Last, First" formatted names to "First Last" across a file. What's the `sed` approach, and what regex concept does it rely on beyond plain substitution?**

`sed 's/\(.*\), \(.*\)/\2 \1/' file.txt` — this relies on captured groups
(Demo M05-01 §1.2's grouping) being referenced in the *replacement* text via
`\1`/`\2`, not just matched in the pattern. The pattern captures the last
name and first name as two groups; the replacement reassembles them in a
different order. This is distinct from a pattern-internal backreference
(matching a repeated word within the match itself) — here the capture
happens once and gets relocated into a new position in the output.

---

**Q4. When would you reach for `sed`'s hold space instead of a simple substitution, and what's the tradeoff?**

Hold space becomes relevant when a transformation needs to reference or
accumulate content from a different line than the one currently being
processed — multi-line context, reordering, or lookback logic that a
pure line-by-line substitution can't express. The tradeoff is readability:
hold-space one-liners get cryptic fast, and if the task is fundamentally
about fields, arithmetic, or structured comparison rather than line
accumulation, `awk` is usually a clearer and more maintainable choice than
pushing `sed`'s hold space further.

---

## Key Takeaways

1. **`s///` replaces only the first match per line by default — `g` is
   required for every match.** This is one of the most common "my sed
   command only half-worked" moments.
2. **`&` in the replacement reuses the whole matched text; `\1`/`\2` reuse
   specific captured groups from the pattern — and both can be reordered
   or decorated freely.** This is what makes `sed` capable of real
   restructuring (swapping fields, reformatting dates), not just literal
   find-and-replace — and it's a different use of grouping from Demo M05-01's
   pattern-internal backreferences, since the capture here gets placed
   somewhere new in the output rather than required to repeat in the match.
3. **`-n` combined with `p` is the correct pattern for extracting specific
   lines; forgetting `-n` doubles the output for matched lines.** `sed`
   prints every line by default, and `p` adds to that rather than replacing
   it. Pattern-based ranges (`/START/,/END/`) include both endpoint lines.
4. **`a`/`i`/`c` insert, append, or replace whole lines — structurally
   different from `s///`'s targeted in-line editing.** `q` stops `sed`
   entirely once reached, genuinely halting further reads, not just
   suppressing further output the way `-n '1,Np'` would.
5. **`sed -i` with no backup suffix is genuinely irreversible.** `-i.bak`
   costs nothing and provides a real recovery path (the suffix is appended
   to the filename, not used as a prefix) — test without `-i` first, then
   add `-i.bak`, not bare `-i`, once confirmed.
6. **Hold space is what makes `sed` capable of multi-line, accumulating
   transformations — but reaching for it is a signal worth noticing.**
   Lowercase `h`/`g` overwrite; uppercase `H`/`G` append. Most everyday
   `sed` usage never needs any of them; if a task needs field awareness or
   arithmetic, `awk` is almost always the better fit.
7. **`sed` uses BRE by default, same as `grep`.** `sed -E`/`-r` switches
   to ERE — the same flavor-deliberateness from Demo M05-01 applies here
   exactly, and `i` (case-insensitive) is a GNU extension, not guaranteed
   portable to every `sed`.
8. **Choosing a non-`/` delimiter (`s#...#...#`) avoids escaping noise when
   the pattern itself contains `/`.** Any character can serve as the
   delimiter — pick one that doesn't collide with the content.
9. **`-e` chains expressions inline; `-f` reads them from a reusable
   script file.** Prefer `-f` once a transformation is something you'd
   want to save, version, or hand off, rather than a genuine one-off —
   script files also support `#`-prefixed comments, the same convention
   as shell scripts.

> **Demo scope:** Primary concept: stream editing — substitution and text
> transformation. Supporting concepts: substitution flags, whole-match
> and captured-group replacement references, addressing (including
> insert/append/change/quit), pattern/hold space, in-place editing safety,
> command chaining (inline and script file).
> Estimated completion time: 42 minutes (reading + hands-on + verification).
> Checkpoints: 7 natural stopping points (after Task 1 through Task 7).

## Quick Commands Reference

### Substitution

| Command | Purpose |
|---|---|
| `sed 's/PAT/REPL/'` | Replace first match per line |
| `sed 's/PAT/REPL/g'` | Replace every match per line |
| `sed 's/PAT/REPL/i'` | Case-insensitive (GNU extension) |
| `sed 's/PAT/REPL/2'` | Replace only the Nth match |
| `sed 's/PAT/[&]/'` | Reuse the whole match in the replacement |
| `sed 's/\(a\)-\(b\)/\2-\1/'` | Reorder captured groups in the replacement |
| `sed 's#PAT#REPL#'` | Alternate delimiter, avoids escaping `/` |

### Addressing

| Command | Purpose |
|---|---|
| `sed 'Ns/.../.../'` | Line N only |
| `sed 'N,Ms/.../.../'` | Lines N through M |
| `sed '/PAT1/,/PAT2/s/.../.../'` | Pattern-delimited range (inclusive of both) |
| `sed -n 'N,Mp'` | Print only lines N-M (needs `-n`) |
| `sed '/PAT/d'` | Delete matching lines |
| `sed 'Na\text'` | Append text after line N |
| `sed 'Ni\text'` | Insert text before line N |
| `sed 'Nc\text'` | Replace (change) line N entirely |
| `sed 'Nq'` | Stop processing after line N |

### Hold space

| Command | Purpose |
|---|---|
| `h` / `g` | Overwrite hold space / pattern space |
| `H` / `G` | Append to hold space / pattern space |
| `sed '1!G;h;$!d'` | Reverse a file's line order |

### In-place and scripting

| Command | Purpose |
|---|---|
| `sed -i.bak 's/.../.../g' file` | In-place edit with backup |
| `sed -e '...' -e '...'` | Multiple expressions, inline |
| `sed -f script.sed file` | Run commands from a reusable script file |

## Appendix A — Anki Cards

**`m05-07-sed-editing-anki.csv`:**

```
#deck:Linux Mastery::Module 5 - Text Processing & Search Tools::Demo M05-07 - sed Editing
#separator:Comma
#columns:Front,Back,Tags
"sed 's/a/b/' on the input 'aaa' produces 'baa', not 'bbb'. Why?","s/// replaces only the FIRST match per line by default. The g flag (s/a/b/g) is required to replace every match on the line.","demo-m05-07,sed,substitution,common-mistake"
"What does & mean in a sed replacement string, e.g. sed 's/hello/[&]/'?","& represents the WHOLE matched text — here it's replaced with 'hello' itself, so the output is '[hello]'. It lets you reuse the match in the replacement without retyping it.","demo-m05-07,sed,ampersand,fundamentals"
"How do you insert a LITERAL ampersand character in a sed replacement, instead of the whole-match meaning?","Escape it: \\& — an unescaped & means 'the whole match'; \\& produces an actual '&' character in the output.","demo-m05-07,sed,ampersand,escaping"
"sed 's/\\(.*\\) \\(.*\\)/\\2, \\1/' is run on 'John Smith'. What's the output, and what regex concept does this rely on?","'Smith, John' — this relies on captured groups from the PATTERN being referenced in the REPLACEMENT via \\1/\\2, reordering them. Different from Demo M05-01's pattern-internal backreferences (matching a repeated word within the match itself).","demo-m05-07,sed,backreferences,cross-demo"
"What's the difference between sed's d command and its a/i/c commands?","d deletes the addressed line(s) entirely. a appends a new line AFTER the address; i inserts a new line BEFORE the address; c replaces (changes) the addressed line(s) entirely with new content — structurally different from s///'s targeted in-line editing.","demo-m05-07,sed,commands,fundamentals"
"What does sed '3q' do differently from sed -n '1,3p'?","Both show only the first 3 lines, but 3q genuinely STOPS sed processing entirely once line 3 finishes — it never reads lines 4+ at all. -n '1,3p' still reads the whole file, just suppresses printing most of it. q matters for performance on very large input.","demo-m05-07,sed,quit,performance"
"sed '5p' file.txt | head -1 prints line 5, but the raw sed '5p' file.txt command (without head) shows line 5 twice while every other line appears once. Why?","sed prints every line once by default; p is an ADDITIONAL explicit print layered on top, not a replacement. Without -n suppressing the default output, the addressed line (5) gets both its normal print and the extra p print, while unaddressed lines only get their one normal print.","demo-m05-07,sed,addressing,common-mistake"
"What's the correct sed pattern for extracting ONLY lines 2 through 4 from a file, printing nothing else?","sed -n '2,4p' file.txt — -n suppresses sed's default automatic printing, leaving only the explicit p output for the addressed range.","demo-m05-07,sed,addressing"
"Why is sed -i (with no backup suffix) considered genuinely risky, more than 'just be careful'?","It's irreversible the instant it completes — there's no sed-level undo, unlike a piped sed without -i which only affects stdout and leaves the source file untouched. -i.bak preserves the original as filename.bak before overwriting.","demo-m05-07,sed,in-place,safety"
"What is hold space actually for in sed, and when would you reach for it?","A persistent buffer that survives across lines, letting sed remember content from an earlier line and use it on a later one. Reach for it when a transformation needs multi-line context or accumulation — pure line-by-line substitution can't express that.","demo-m05-07,sed,hold-space,fundamentals"
"Why would you use sed 's#/#-#g' instead of sed 's/\\//-/g' for the same substitution on a file path?","Choosing a delimiter (# here) that doesn't collide with the pattern/replacement content avoids having to escape every literal / in the pattern, which gets visually noisy fast — sed allows any character as the delimiter for exactly this reason.","demo-m05-07,sed,delimiter"
"What regex flavor does sed use by default, and how do you switch it?","BRE (Basic Regular Expressions), same default as grep without -E. sed -E (or -r on some systems) switches to ERE.","demo-m05-07,sed,bre,ere,cross-demo"
```

## Appendix B — Quiz

**`m05-07-sed-editing-quiz.md`:**

````markdown
# Quiz — Demo M05-07: sed Editing

> One correct answer per question unless stated otherwise.
> Target: 80% or above before moving to Demo M05-08.

---

**Q1.** `echo "hello" | sed 's/hello/[&]/'` is run. What's the output?

- A) `[]` — `&` is discarded
- B) `[hello]` — `&` represents the whole matched text, reused in the replacement
- C) `[&]` — literally, since `&` is always treated as a literal character
- D) An error — `&` is invalid in a `sed` replacement

<details>
<summary>Answer</summary>

**B** — `&` in the replacement stands for whatever the pattern actually matched, letting you reuse it (here, wrapped in brackets) without retyping the matched text.

Trap: A wrongly assumes `&` is dropped; C reverses `&`'s actual default meaning (special, not literal); D invents an error that doesn't occur — this is standard, valid `sed` syntax.

</details>

---

**Q2.** `sed 's/100/\&99/'` is run against "price 100". What's the output, and why?

- A) `price 10099` — `\&` still means "the whole match"
- B) `price &99` — `\&` escapes the ampersand into a literal character, distinct from unescaped `&`'s "whole match" meaning
- C) An error — `\&` is invalid syntax
- D) `price 99` — `\&` deletes the matched text entirely

<details>
<summary>Answer</summary>

**B** — Escaping `&` with a backslash flips its meaning from "insert the whole match" to "insert a literal ampersand character" — the same escape-to-get-literal pattern used elsewhere in regex syntax.

Trap: A confuses escaped and unescaped `&`; C invents an error for valid, common syntax; D invents an unrelated deletion behavior.

</details>

---

**Q3.** `sed 's/\(.*\) \(.*\)/\2 \1/'` is run against "John Smith". What's the output, and what capability does this rely on?

- A) `John Smith` — unchanged, since the pattern only matches, it doesn't transform
- B) `Smith John` — captured groups (`\(...\)`) from the pattern are referenced and reordered in the replacement via `\1`/`\2`
- C) An error — backreferences can't be used in the replacement, only the pattern
- D) `\2 \1` literally, since backslash-digit sequences aren't interpreted in the replacement

<details>
<summary>Answer</summary>

**B** — This is exactly what makes `sed` capable of restructuring text, not just literal substitution: the pattern captures two groups, and the replacement places them in a different order.

Trap: A wrongly assumes no transformation happens; C wrongly claims a limitation that doesn't exist — captured groups are routinely referenced in the replacement; D misreads `\1`/`\2` as literal text rather than interpreted references.

</details>

---

**Q4.** `sed '2a\inserted'` and `sed '2i\inserted'` are both run against a 3-line file. How do their outputs differ?

- A) They produce identical output
- B) `2a\inserted` places the new line AFTER line 2; `2i\inserted` places it BEFORE line 2 — otherwise identical in effect
- C) `a` only works on the last line; `i` only works on the first line
- D) `a` deletes line 2 before inserting; `i` does not

<details>
<summary>Answer</summary>

**B** — `a` (append) and `i` (insert) both add a new line at the given address, differing only in whether the new content lands after or before that line.

Trap: A wrongly claims no difference; C invents line-position restrictions that don't exist — both work at any addressed line; D invents a deletion side-effect neither command has (that's `c`'s job).

</details>

---

**Q5.** `sed '3q' file.txt` and `sed -n '1,3p' file.txt` both produce output showing only the first 3 lines. Do they behave identically overall?

- A) Yes, in every respect
- B) No — `3q` genuinely stops `sed` from reading or processing anything past line 3, while `-n '1,3p'` still reads the entire file, just suppressing output for everything outside the range
- C) No — `-n '1,3p'` is faster in every case
- D) No — `3q` requires `-n` to work at all

<details>
<summary>Answer</summary>

**B** — Both produce the same visible output for this simple case, but `q` actually halts processing early (relevant for performance on very large files), while `-n '1,3p'` still reads through the whole input even though it only prints a subset.

Trap: A overlooks the real processing difference; C reverses which approach is actually faster on large input; D wrongly claims a dependency `q` doesn't have — it works standalone.

</details>

---

**Q6.** `sed -i.bak 's/foo/bar/' myfile.conf` is run. What filename does the backup end up with?

- A) `bak.myfile.conf`
- B) `myfile.conf.bak` — the suffix is appended to the existing filename
- C) A file named exactly `.bak` in the same directory
- D) `myfile.bak.conf`

<details>
<summary>Answer</summary>

**B** — The backup suffix given to `-i` is appended directly onto the original filename, not used as a prefix or a separate file. `myfile.conf` becomes `myfile.conf.bak`.

Trap: A, C, and D all invent incorrect naming schemes for where the suffix actually attaches.

</details>

---

**Q7.** In `sed '1!G;h;$!d'`, what does the `1!G` part specifically accomplish on any given line?

- A) It deletes line 1 of the file entirely
- B) On every line except line 1, it appends whatever is currently in hold space onto the end of the current pattern space — building the reversed accumulation by placing the new line in front of what's already been collected
- C) It globally replaces the digit "1" wherever it appears
- D) It runs the rest of the script only if the line number is greater than 1

<details>
<summary>Answer</summary>

**B** — `1!` means "every line except line 1"; `G` appends hold space's current content onto pattern space. Combined with `h` copying the result back into hold space each time, this incrementally builds the file's content in reverse order.

Trap: A misreads `1!` as deleting line 1; C confuses `G` (append hold space) with a substitution command; D correctly captures the "except line 1" idea but wrongly reframes it as a numeric conditional rather than what `G` actually does.

</details>

---

**Q8.** `sed -n '/START/,/END/p' file.txt` is run on a file containing a `START` line, several lines after it, and an `END` line. Does the output include the `START` and `END` lines themselves, or only the lines strictly between them?

- A) Only the lines strictly between `START` and `END`, excluding both
- B) Both `START` and `END` are included, along with everything between them — the range is inclusive of both endpoint lines
- C) Only `START` is included; `END` is excluded
- D) Only `END` is included; `START` is excluded

<details>
<summary>Answer</summary>

**B** — Pattern-based ranges in `sed` are inclusive on both ends — the line that matches the first pattern and the line that matches the second pattern are both part of the addressed range, not just the content between them.

Trap: A, C, and D all wrongly exclude one or both endpoint lines from what's actually a fully inclusive range.

</details>

---

Score guide:

| Score | Action |
|---|---|
| 7/8 or 8/8 | Import Anki cards, move to Demo M05-08 |
| 6/8 | Review the wrong answers, then proceed |
| 5/8 | Re-read the relevant sections, retry those questions |
| Below 5/8 | Re-read the full demo and redo the walkthrough before proceeding |
````