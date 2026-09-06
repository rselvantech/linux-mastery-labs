# Demo 20 — sed

## Overview

Demo 18's `grep` finds; this demo's `sed` changes. `sed` ("stream editor")
reads text line by line and applies editing commands to it — substitution
is the command you'll use 90% of the time, but `sed`'s addressing model and
its hold-space mechanism let it do genuinely programmatic text
transformation, not just find-and-replace. By the end, you'll write
confident one-liners and understand exactly when `sed` is the right tool
versus when it's being stretched past what it's built for.

## Demo Objectives

1. Perform substitution with `s///`, including flags for global and case-insensitive replacement
2. Address specific lines or ranges of lines for a command to apply to
3. Explain pattern space and hold space, and what they're actually for
4. Edit files in place safely, with and without a backup
5. Chain multiple sed commands into a script
6. Recognize when a task has outgrown sed and belongs in awk instead

## Directory Structure

```
20-sed/
└── README.md   # This file — full demo content, including Anki (Appendix A)
                 # and Quiz (Appendix B), extracted later via script.
```

## Environments

| Task | Ubuntu 24.04 | Rocky Linux 9.8 | EC2 Amazon Linux |
|---|---|---|---|
| All lab tasks | ✅ (GNU sed) | ✅ (GNU sed) | Not required |

## Recall Check — 19-text-processing-utilities

Answer these from memory before continuing — no looking back at Demo 19.

**Q1.** `uniq` is run on an unsorted file with a duplicate line that isn't
adjacent to its twin. What happens, and why?

**Q2.** `find /tmp -name '*.tmp' | xargs rm` occasionally deletes the wrong
file. What's the most likely cause, and the safe fix?

**Q3.** Why is `cut -d' '` often a poor choice for parsing output like `ps`
or `df`, compared to `awk`?

<details>
<summary>Answers</summary>

**A1.** It survives — `uniq` only removes *adjacent* duplicate lines, with
no concept of duplicates elsewhere in the file. Sorting first makes
duplicates adjacent, which is what `uniq` actually needs.

**A2.** A filename contains a space, and default `xargs` splits on
whitespace, breaking one filename into multiple arguments. Fix:
`find ... -print0 | xargs -0 rm`.

**A3.** `cut` treats every occurrence of the delimiter as a boundary,
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
# hi World — i flag: case-insensitive matching (GNU sed extension)

echo "aaa" | sed 's/a/b/2'
# aba — a number instead of g replaces only that specific occurrence
# (the 2nd match), not all or just the first
```

**`sed` uses BRE by default, same as `grep` without `-E`** (Demo 17's
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
# inclusive — pattern-based ranges, not just numeric ones

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

### 1.3 Pattern space and hold space

Every `sed` command operates on the **pattern space** — the current line
being processed, one at a time by default. The **hold space** is a
separate, persistent buffer that survives across lines, letting `sed`
remember something from an earlier line and use it later:

```bash
sed -n '1h; 1!H; $!d; $x; $p' file.txt
```

This one-liner reverses the entire file (`sed`'s equivalent of `tac`):
`1h` copies line 1 into hold space (overwriting it); `1!H` (every line
except line 1) appends to hold space instead; `$!d` deletes every line
except the last from normal output; `$x` (only on the last line) swaps
pattern space and hold space; `$p` prints the result — the net effect
accumulates every line into hold space in order, then dumps it all out at
the end.

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
# overwriting — a safety net specifically because -i alone gives you no
# way to recover the original if the substitution was wrong.
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
s/foo/bar/
s/baz/qux/
/^#/d
EOF
sed -f script.sed file.txt
# -f: read commands from a script file instead of the command line —
# worth doing once a transformation grows past 2-3 simple substitutions,
# for readability and reuse.
```

### 1.6 When a task has outgrown sed

`sed` is a line-oriented text transformer with limited field awareness and
no arithmetic — the moment a task needs to reference "the third
whitespace-separated value on this line" by position, sum values across
lines, or do conditional logic based on a field's actual value (not just
its text pattern), `awk` (Demo 21) is almost always the better tool.
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
| `sed -n 'N,Mp'` | `-n`+`p`: suppress default, explicit print | Extract specific lines only |
| `sed '/PAT/d'` | `d`: delete | Remove matching lines |
| `sed -i.bak 's/PAT/REPL/g'` | `-i.bak`: in-place with backup | Edit a file directly, safely |
| `sed -f script.sed` | `-f`: script file | Run multiple commands from a file |

### Task 1 — Basic substitution and delimiter choice

```bash
printf 'the quick brown fox\n' > /tmp/sed-lab.txt
sed 's/quick/slow/' /tmp/sed-lab.txt
sed 's/o/0/g' /tmp/sed-lab.txt

printf '/var/log/app.log\n' > /tmp/sed-lab2.txt
sed 's/\//-/g' /tmp/sed-lab2.txt
sed 's#/#-#g' /tmp/sed-lab2.txt
```

### Task 2 — Addressing: line numbers, ranges, and patterns

```bash
printf 'one\ntwo\nthree\nfour\nfive\n' > /tmp/sed-lab3.txt
sed '3s/^/>>> /' /tmp/sed-lab3.txt
sed '2,4s/^/>>> /' /tmp/sed-lab3.txt
sed -n '2,4p' /tmp/sed-lab3.txt

printf 'text\nSTART\nkeep this\nEND\ntext\n' > /tmp/sed-lab4.txt
sed -n '/START/,/END/p' /tmp/sed-lab4.txt
```

### Task 3 — Delete matching lines

```bash
printf '# comment\nreal line 1\n# another comment\nreal line 2\n' > /tmp/sed-lab5.txt
sed '/^#/d' /tmp/sed-lab5.txt
```

### Task 4 — In-place editing, test-first workflow

```bash
printf 'debug=true\nport=8080\n' > /tmp/sed-lab6.conf
sed 's/debug=true/debug=false/' /tmp/sed-lab6.conf
# Confirm the output looks right BEFORE adding -i:
sed -i.bak 's/debug=true/debug=false/' /tmp/sed-lab6.conf
cat /tmp/sed-lab6.conf
cat /tmp/sed-lab6.conf.bak
```

### Task 5 — Chaining commands

```bash
printf 'foo=1\nbaz=2\n# remove me\n' > /tmp/sed-lab7.txt
sed -e 's/foo/FOO/' -e 's/baz/BAZ/' -e '/^#/d' /tmp/sed-lab7.txt
```

### Task 6 — A hold-space taste: print only the last line

```bash
printf 'a\nb\nc\n' | sed -n '$p'
# $ addresses only the last line — no hold space needed for this simple
# case, but confirms you understand $ as "last line" before the more
# elaborate hold-space example in §1.3.
```

## What You Learned

1. ✅ Performed substitution with `s///`, including `g`, `i`, and numbered-occurrence flags
2. ✅ Addressed specific lines or ranges — numeric, pattern-based, and `$`
3. ✅ Explained pattern space and hold space, and what they're actually for
4. ✅ Edited files in place safely, testing without `-i` first
5. ✅ Chained multiple sed commands with `-e` and script files
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

**Q2. A `sed -n '5p' file.txt` command is expected to print line 5 once. What would happen if `-n` were accidentally omitted, and why?**
Line 5 would print twice — once from `sed`'s normal default behavior of
printing every line automatically, and a second time from the explicit `p`
command, which adds a print rather than replacing the default one. Every
other line in the file would still print exactly once (their normal
default print, with no `p` command applying to them), so the doubling would
be specific and isolated to line 5, which is often the confusing part of
this particular mistake.

**Q3. When would you reach for `sed`'s hold space instead of a simple substitution, and what's the tradeoff?**
Hold space becomes relevant when a transformation needs to reference or
accumulate content from a different line than the one currently being
processed — multi-line context, reordering, or lookback logic that a
pure line-by-line substitution can't express. The tradeoff is readability:
hold-space one-liners get cryptic fast, and if the task is fundamentally
about fields, arithmetic, or structured comparison rather than line
accumulation, `awk` is usually a clearer and more maintainable choice than
pushing `sed`'s hold space further.

**Q4. What's the practical reason `sed` lets you choose a delimiter other than `/` for substitution, like `s#...#...#`?**
Readability, specifically when the pattern or replacement text itself
contains `/` — file paths being the most common case. Using `/` as the
delimiter in that situation requires escaping every literal `/` in the
pattern (`\/`), which gets visually noisy fast; choosing a delimiter that
doesn't collide with the content (like `#`) avoids that escaping entirely.

## Key Takeaways

1. **`s///` replaces only the first match per line by default — `g` is
   required for every match.** This is one of the most common "my sed
   command only half-worked" moments.
2. **`-n` combined with `p` is the correct pattern for extracting specific
   lines; forgetting `-n` doubles the output for matched lines.** `sed`
   prints every line by default, and `p` adds to that rather than replacing
   it.
3. **`sed -i` with no backup suffix is genuinely irreversible.** `-i.bak`
   costs nothing and provides a real recovery path — test without `-i`
   first, then add `-i.bak`, not bare `-i`, once confirmed.
4. **Hold space is what makes `sed` capable of multi-line, accumulating
   transformations — but reaching for it is a signal worth noticing.** Most
   everyday `sed` usage never needs it; if a task needs field awareness or
   arithmetic, `awk` is almost always the better fit.
5. **`sed` uses BRE by default, same as `grep`.** `sed -E`/`-r` switches
   to ERE — the same flavor-deliberateness from Demo 17 applies here
   exactly.
6. **Choosing a non-`/` delimiter (`s#...#...#`) avoids escaping noise when
   the pattern itself contains `/`.** Any character can serve as the
   delimiter — pick one that doesn't collide with the content.

> **Demo scope:** Primary concept: stream editing — substitution and text
> transformation. Supporting concepts: substitution flags, addressing,
> pattern/hold space, in-place editing safety, command chaining.
> Estimated completion time: 30 minutes (reading + hands-on + verification).
> Checkpoints: 6 natural stopping points (after Task 1 through Task 6).

## Quick Commands Reference

### Substitution

| Command | Purpose |
|---|---|
| `sed 's/PAT/REPL/'` | Replace first match per line |
| `sed 's/PAT/REPL/g'` | Replace every match per line |
| `sed 's/PAT/REPL/i'` | Case-insensitive (GNU extension) |
| `sed 's/PAT/REPL/2'` | Replace only the Nth match |
| `sed 's#PAT#REPL#'` | Alternate delimiter, avoids escaping `/` |

### Addressing

| Command | Purpose |
|---|---|
| `sed 'Ns/.../.../'` | Line N only |
| `sed 'N,Ms/.../.../'` | Lines N through M |
| `sed '/PAT1/,/PAT2/s/.../.../'` | Pattern-delimited range |
| `sed -n 'N,Mp'` | Print only lines N-M (needs `-n`) |
| `sed '/PAT/d'` | Delete matching lines |

### In-place and scripting

| Command | Purpose |
|---|---|
| `sed -i.bak 's/.../.../ g' file` | In-place edit with backup |
| `sed -e '...' -e '...'` | Multiple expressions |
| `sed -f script.sed file` | Run commands from a script file |

## Appendix A — Anki Cards

**Note:** extracted into a standalone `.csv` by the series extraction script.

```
#deck:Linux Mastery::Module 4 - Text Processing & Scripting for DevOps::20-sed
#separator:Comma
#columns:Front,Back,Tags
"sed 's/a/b/' on the input 'aaa' produces 'baa', not 'bbb'. Why?","s/// replaces only the FIRST match per line by default. The g flag (s/a/b/g) is required to replace every match on the line.","demo20,sed,substitution,common-mistake"
"sed '5p' file.txt | head -1 prints line 5, but the raw sed '5p' file.txt command (without head) shows line 5 twice while every other line appears once. Why?","sed prints every line once by default; p is an ADDITIONAL explicit print layered on top, not a replacement. Without -n suppressing the default output, the addressed line (5) gets both its normal print and the extra p print, while unaddressed lines only get their one normal print.","demo20,sed,addressing,common-mistake"
"What's the correct sed pattern for extracting ONLY lines 2 through 4 from a file, printing nothing else?","sed -n '2,4p' file.txt — -n suppresses sed's default automatic printing, leaving only the explicit p output for the addressed range.","demo20,sed,addressing"
"Why is sed -i (with no backup suffix) considered genuinely risky, more than 'just be careful'?","It's irreversible the instant it completes — there's no sed-level undo, unlike a piped sed without -i which only affects stdout and leaves the source file untouched. -i.bak preserves the original as filename.bak before overwriting.","demo20,sed,in-place,safety"
"What's the safer workflow habit for using sed -i, given its destructiveness?","Test the substitution WITHOUT -i first to visually confirm the output is correct, then re-run with -i.bak (not bare -i) once confirmed — never add -i to an untested command.","demo20,sed,in-place,workflow"
"What is hold space actually for in sed, and when would you reach for it?","A persistent buffer that survives across lines, letting sed remember content from an earlier line and use it on a later one. Reach for it when a transformation needs multi-line context or accumulation — pure line-by-line substitution can't express that.","demo20,sed,hold-space,fundamentals"
"Most everyday sed usage never touches hold space. What does needing it usually signal?","That the task has grown past simple line-by-line substitution — and if the underlying need is field awareness or arithmetic rather than line accumulation, awk is usually a clearer, more maintainable tool than pushing sed's hold space further.","demo20,sed,hold-space,tool-boundary"
"Why would you use sed 's#/#-#g' instead of sed 's/\\//-/g' for the same substitution on a file path?","Choosing a delimiter (# here) that doesn't collide with the pattern/replacement content avoids having to escape every literal / in the pattern, which gets visually noisy fast — sed allows any character as the delimiter for exactly this reason.","demo20,sed,delimiter"
"What regex flavor does sed use by default, and how do you switch it?","BRE (Basic Regular Expressions), same default as grep without -E. sed -E (or -r on some systems) switches to ERE.","demo20,sed,bre,ere,cross-demo"
"sed '/^#/d' file.txt — what does this command do?","Deletes every line starting with # from the output entirely (comment-stripping) — d is a delete command, not a substitution, and applies to every line matching the /^#/ address.","demo20,sed,delete,addressing"
```

## Appendix B — Quiz

**Note:** extracted into a standalone `.md` by the series extraction script.

> One correct answer per question unless stated otherwise.
> Target: 80% or above before moving to Demo 21.

**Q1. `sed 's/a/b/'` on input `"aaa"` produces `"baa"`. Why not `"bbb"`?**

- A) `sed` has a bug with repeated characters
- B) `s///` replaces only the first match per line by default; `g` is needed for all matches
- C) The pattern needs to be `s/a+/b/` to match all occurrences
- D) `sed` only processes the first 3 characters of each line

<details>
<summary>Answer</summary>

**B** — This is expected default `s///` behavior. Trap: A invents a bug; C confuses quantifier syntax with the actual missing flag; D invents an unrelated character limit.

</details>

---

**Q2. `sed '5p' file.txt` (no `-n`) shows line 5 twice, while every other line shows once. Why?**

- A) Line 5 must actually be duplicated in the source file
- B) `sed` prints every line by default; `p` adds an explicit print on top rather than replacing the default
- C) `p` always prints matched lines twice as a built-in behavior
- D) This only happens with GNU sed, not other implementations

<details>
<summary>Answer</summary>

**B** — Without `-n`, the default automatic print plus the explicit `p` print combine for the addressed line specifically. Trap: A wrongly blames the source data; C invents a fixed double-print behavior; D invents a portability distinction not relevant here.

</details>

---

**Q3. What's the correct command to print ONLY lines 2 through 4 of a file, nothing else?**

- A) `sed '2,4p' file.txt`
- B) `sed -n '2,4p' file.txt`
- C) `sed -p '2,4' file.txt`
- D) `sed '2:4p' file.txt`

<details>
<summary>Answer</summary>

**B** — `-n` suppresses the default print, leaving only the explicit `p` output for the addressed range. Trap: A omits the required `-n`, producing every line plus doubled 2-4; C and D invent incorrect syntax.

</details>

---

**Q4. Why is `sed -i` with no backup suffix considered genuinely risky?**

- A) It runs significantly slower than piped sed
- B) It's irreversible — the original content is overwritten with no sed-level undo available
- C) It requires root privileges, increasing blast radius
- D) It only works on the first line of the file

<details>
<summary>Answer</summary>

**B** — This is the actual, specific risk `-i.bak` is designed to mitigate. Trap: A invents a performance concern; C is factually wrong — no elevated privileges are inherently required; D invents a false scope limitation.

</details>

---

**Q5. What is `sed`'s hold space actually for?**

- A) Storing the sed script itself for logging purposes
- B) A persistent buffer surviving across lines, enabling multi-line context or accumulation that pure line-by-line substitution can't express
- C) Temporarily disabling output during processing
- D) Caching regex compilation for performance

<details>
<summary>Answer</summary>

**B** — This is hold space's actual purpose and the reason it exists alongside pattern space. Trap: A, C, and D all invent unrelated purposes.

</details>

---

**Q6. Needing sed's hold space for a task is often a signal that:**

- A) The sed installation is outdated and needs updating
- B) The task may have outgrown sed, and awk might be a clearer fit if it involves field awareness or arithmetic
- C) The file is too large for sed to process normally
- D) A different shell (zsh instead of bash) is required

<details>
<summary>Answer</summary>

**B** — This is the practical guidance from §1.6 — hold space is a real capability, but reaching for it is worth treating as a prompt to consider whether `awk` fits the actual need better. Trap: A, C, and D all invent unrelated causes.

</details>

---

**Q7. Why might you write `sed 's#/#-#g'` instead of `sed 's/\/​/-/g'` for the same file-path substitution?**

- A) `#` is required syntax for path substitutions specifically
- B) Choosing a delimiter that doesn't collide with the pattern's content avoids escaping every literal `/`
- C) `#` makes the substitution case-insensitive automatically
- D) `/` cannot be used as a sed delimiter at all

<details>
<summary>Answer</summary>

**B** — Any character can serve as sed's delimiter; picking one absent from the pattern/replacement avoids visual escaping noise. Trap: A and D both wrongly treat `/` as forbidden or `#` as required; C invents an unrelated effect.

</details>

---

**Q8. What regex flavor does plain `sed` (without `-E`) use by default?**

- A) ERE
- B) BRE
- C) PCRE
- D) A sed-specific flavor unrelated to POSIX

<details>
<summary>Answer</summary>

**B** — `sed` defaults to BRE, the same default as `grep` without `-E`, consistent with Demo 17's flavor table. Trap: A, C, and D all misstate the actual default.

</details>