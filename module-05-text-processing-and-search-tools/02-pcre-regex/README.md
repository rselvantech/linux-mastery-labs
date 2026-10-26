# Demo M05-02 — PCRE and grep -P

## Overview

Demo M05-01 covered the regular-expression syntax every Linux tool shares.
This demo covers what `grep -P` (Perl-Compatible Regular Expressions,
PCRE) adds on top: shorthand classes like `\d` and `\w`, non-greedy
matching, and — the feature that changes how you write extraction
commands — **lookaround**, which lets a pattern match text based on what
surrounds it without including that surrounding text in the result.

It also covers the other half of the decision: PCRE is not available
everywhere. Knowing which `grep -P` features have a plain-POSIX
equivalent, and which ones genuinely don't, is what lets you choose
between a short PCRE one-liner and a more portable pipeline on purpose.

Demo M05-03 covers how the flavors differ across tools; this demo goes deep
on the richest one.

## Demo Objectives

1. Use the shorthand classes `\d`, `\w`, `\s`, and `\b`, and state what
   each matches
2. Use lazy quantifiers to match the shortest span instead of the longest
3. Use the four lookaround forms to match text by its context without
   including the context in the match
4. Use `\K` with `-o` to extract only a value, including when the text
   before it varies in length
5. Use inline flags `(?i)`, non-capturing groups, and named groups with
   back-references
6. Translate each PCRE feature to a POSIX-only equivalent, and identify
   the ones that have no direct equivalent
7. Choose between `grep -P` and a portable ERE pipeline on purpose

## Directory Structure

```
02-pcre-regex/
├── README.md                   # this file
├── m05-02-pcre-regex-anki.csv  # Anki flashcard deck
└── m05-02-pcre-regex-quiz.md   # standalone quiz
```

## Environments

| Task | Ubuntu 24.04 | Rocky Linux 9.8 | EC2 Amazon Linux |
|---|---|---|---|
| `grep -P` | ✅ GNU grep 3.11, PCRE support compiled in | ✅ GNU grep (`-P` expected present) `[Unverified — test before use]` | Not required |
| ERE equivalents (`grep -E`, `sed -E`, `awk`, `cut`) | ✅ | ✅ | Not required |

> **Verification note:** every pattern, output, exit code, and error
> message in this demo was run live in the authoring sandbox (Ubuntu
> 24.04, GNU grep 3.11, GNU sed 4.9, mawk 1.3.4). Not tested there:
> Rocky's `grep -P`, and builds of `grep` that omit PCRE support (such as
> minimal container images or BSD-derived `grep`) — those statements are
> marked `[Unverified — test before use]`.

## Recall Check — 01-regular-expressions

Answer these from memory before continuing — no looking back at Demo M05-01.

**Q1.** What is the difference between `.` and `\.` in a pattern?

**Q2.** What is the difference between `a*` and `a+`?

**Q3.** What does `(cat|dog)s?` match?

<details>
<summary>Answers</summary>

**A1.** `.` matches any single character; `\.` matches a literal dot only.

**A2.** `a*` matches zero or more `a` characters (so it also matches the
empty string); `a+` requires at least one.

**A3.** The word `cat` or `dog`, optionally followed by an `s` — so `cat`,
`cats`, `dog`, and `dogs`.

</details>

---

## Part 1 — Concepts

### 1.1 What -P adds

| Feature | Example | Needs PCRE? |
|---|---|---|
| Shorthand classes | `\d` `\w` `\s` `\b` | `\d` yes; `\s`/`\b` also GNU ERE |
| Lazy quantifiers | `.*?` `+?` | yes |
| Lookahead / lookbehind | `(?=...)` `(?<=...)` | yes |
| Reset match start | `\K` | yes |
| Inline flags | `(?i)` | yes |
| Non-capturing / named groups | `(?:...)` `(?<n>...)` | yes |
| Possessive quantifiers | `a++` | yes |

Everything in the rest of this demo was run against this sample file:

```
price: $45.99 USD
price: $7 USD
id=482 user=ann
id=17 user=bob
ERROR timeout at 10:42
WARN timeout at 10:43
foo_bar baz
2026-10-10 deploy ok
```

### 1.2 Shorthand classes

| Shorthand | Matches | Equivalent POSIX class |
|---|---|---|
| `\d` | a digit | `[0-9]` or `[[:digit:]]` |
| `\w` | a "word" character: letter, digit, or underscore | `[[:alnum:]_]` |
| `\s` | whitespace | `[[:space:]]` |
| `\b` | a word boundary (a position, not a character) | — |

```
$ grep -oP '\d+' p.txt | head -4
45
99
7
482
$ printf 'foo_bar baz\n' | grep -oP '^\w+'
foo_bar
```

The second command confirms `\w` includes the underscore: it matched all
of `foo_bar`. `-o` prints only the matched part, one match per line, which
is why every extraction in this demo uses it.

### 1.3 Lazy quantifiers

A normal quantifier is greedy: it takes as much as it can. Adding `?`
makes it lazy — it takes as little as it can.

```
$ printf '<a>x</a><a>y</a>\n' | grep -oP '<a>.*?</a>'
<a>x</a>
<a>y</a>
```

The greedy `.*` would return the whole line as a single match. The lazy
form returns the two pairs separately. (In `grep -E` the same `.*?` is
accepted but is *not* lazy — see Demo M05-03 §1.6.)

### 1.4 Lookaround — match by context, return only the value

A lookaround is a **zero-width assertion**: it checks what is next to the
current position without consuming those characters, so they are not part
of the match. There are four forms:

| Form | Syntax | Meaning |
|---|---|---|
| positive lookahead | `(?=...)` | what follows must match |
| negative lookahead | `(?!...)` | what follows must **not** match |
| positive lookbehind | `(?<=...)` | what precedes must match |
| negative lookbehind | `(?<!...)` | what precedes must **not** match |

```
$ grep -oP '(?<=id=)\d+' p.txt                # digits after "id="
482
17
$ grep -oP '\$[\d.]+(?= USD)' p.txt           # amount followed by " USD"
$45.99
$7
$ grep -P '^(?!WARN).*timeout' p.txt          # timeout lines NOT starting WARN
ERROR timeout at 10:42
```

The first command is the one that replaces a pipeline: the label `id=` is
checked but not printed, so one command returns just the numbers.

**A negative lookahead must be anchored.** Without the `^`, the pattern
can start matching one character later, past the text you meant to
exclude:

```
$ grep -P '(?!WARN).*timeout' p.txt           # unanchored — wrong
ERROR timeout at 10:42
WARN timeout at 10:43
$ grep -P '^(?!WARN).*timeout' p.txt          # anchored — correct
ERROR timeout at 10:42
```

The `WARN` line still matches the unanchored version because the engine
tries every starting position: at position 0 the lookahead fails, but at
position 1 the text is `ARN timeout...`, which does not start with
`WARN`, so the pattern succeeds.

**A lookbehind must have a fixed length.** PCRE needs to know how far back
to look:

```
$ grep -oP '(?<=id=\d+ user=)\w+' p.txt
grep: lookbehind assertion is not fixed length          (exit status 2)
$ grep -oP '(?<=id=|uid=)\d+' p.txt                      # two fixed lengths: OK
482
17
```

Alternatives of *different but individually fixed* lengths (`id=|uid=`)
are accepted; an open-ended quantifier like `\d+` inside the lookbehind is
not. The next section is the answer when you need variable-length left
context.

### 1.5 \K — reset the match start

`\K` tells the engine to throw away everything matched so far and start
the reported match here. It behaves like a lookbehind with no length
limit:

```
$ grep -oP 'id=\d+ user=\K\w+' p.txt
ann
bob
```

`id=\d+ user=` has a variable-length digit run, which §1.4 showed a
lookbehind rejects. `\K` accepts it, and `-o` prints only `ann` and `bob`.

### 1.6 Inline flags, groups, and back-references

```
$ grep -cP '(?i)error' p.txt                  # (?i): case-insensitive from here on
1
$ printf 'foobar\n' | grep -oP '(?:foo)bar'   # (?:...): group without capturing
foobar
$ printf 'abcabc\nabcabd\n' | grep -P '(?<g>abc)\k<g>'    # named group + back-reference
abcabc
$ printf 'abcabc\nabcabd\n' | grep -P '(abc)\1'            # numbered back-reference
abcabc
```

A back-reference matches the *same text* the group captured, so `abcabd`
is rejected.

**Possessive quantifiers** (`a++`) take as much as they can and never give
any back. That is what makes them useful for avoiding runaway matching
(Demo M05-04) and also what makes them match differently:

```
$ printf 'aaa\n' | grep -cP 'a+a'      # backtracks one "a" for the final "a"
1
$ printf 'aaa\n' | grep -cP 'a++a'     # keeps all three, nothing left for final "a"
0
```

### 1.7 When you are forced back to POSIX

`grep -P` is an optional feature of GNU grep, and not every `grep` has it
— some builds omit PCRE support entirely, and BusyBox- or BSD-derived
greps may not offer `-P` at all `[Unverified — test before use]`. Test for
it instead of assuming:

```
$ echo 1 | grep -qP '\d' 2>/dev/null && echo "-P supported"
-P supported
```

Each PCRE feature has a portable replacement — except lookaround and
possessive quantifiers. All replacements below were run against the same
sample file and produced the same output as the PCRE original:

| PCRE | Portable equivalent |
|---|---|
| `\d+` | `[0-9]+` |
| `\w+` | `[[:alnum:]_]+` |
| `\s` | `[[:space:]]` |
| `(?i)error` | `grep -i error` |
| `<a>.*?</a>` | `<a>[^<]*</a>` — a negated class stops at the first `<` |
| `(?<=id=)\d+` | `grep -oE 'id=[0-9]+' \| cut -d= -f2`, or `sed -nE 's/.*id=([0-9]+).*/\1/p'`, or awk's `match()` and `substr()` |
| `user=\K\w+` | `sed -nE 's/.*user=([a-z]+).*/\1/p'` |
| `^(?!WARN).*timeout` | `grep timeout \| grep -v '^WARN'` |
| `\$[\d.]+(?= USD)` | `grep -oE '\$[0-9.]+ USD' \| sed 's/ USD//'` |

**What has no direct replacement**: lookaround as a single zero-width test,
and possessive/atomic grouping. Lookaround's job is always done in two
steps instead — capture including the context, then strip the context, or
filter with a second command. That is the real cost of staying portable.

**Choosing:** use `grep -P` for interactive work and for scripts that run
on hosts you control and have verified; write the ERE pipeline when the
script must run on unknown or minimal hosts. If a script uses `-P`, say so
in a comment and feature-detect it (above) rather than letting it fail
with an obscure error.

### 1.8 Exit codes

```
$ grep -P 'zzz' p.txt; echo $?       # valid pattern, no match
1
$ grep -P '(' p.txt; echo $?         # invalid pattern
grep: missing closing parenthesis
2
```

`1` means "ran fine, found nothing"; `2` means the command itself failed
(here a bad pattern, or the lookbehind error from §1.4). A script that
tests only for non-zero cannot tell a genuine empty result from a broken
pattern.

## Part 2 — Hands-On Lab

### Commands used in this demo

| Command | Flag meaning | Purpose |
|---|---|---|
| `grep -P PATTERN file` | `-P`: Perl-compatible | PCRE search |
| `grep -oP PATTERN file` | `-o`: only matching | Print just the matched part |
| `grep -cP PATTERN file` | `-c`: count | Count matching lines |
| `grep -qP PATTERN` | `-q`: quiet | Test only — no output, exit status only |
| `grep -v PATTERN` | `-v`: invert | Keep lines that do not match |
| `cut -d= -f2` | `-d`: delimiter, `-f`: field | Take the part after `=` |

### Task 1 — Create the sample file and try the shorthands

```bash
cat > p.txt << 'EOF'
price: $45.99 USD
price: $7 USD
id=482 user=ann
id=17 user=bob
ERROR timeout at 10:42
WARN timeout at 10:43
foo_bar baz
2026-10-10 deploy ok
EOF
grep -oP '\d+' p.txt | head -4
printf 'foo_bar baz\n' | grep -oP '^\w+'
grep -cP 'foo_bar\sbaz' p.txt
grep -cE 'foo_bar[[:space:]]baz' p.txt
```

Confirm both final commands print `1`, and that `\w+` took `foo_bar`
including the underscore.

### Task 2 — Greedy versus lazy

```bash
printf '<a>x</a><a>y</a>\n' | grep -oE '<a>.*</a>'
printf '<a>x</a><a>y</a>\n' | grep -oP '<a>.*?</a>'
printf '<a>x</a><a>y</a>\n' | grep -oE '<a>[^<]*</a>'
```

Confirm the first returns one match and the second and third return two.

### Task 3 — All four lookarounds

```bash
grep -oP '(?<=id=)\d+' p.txt
grep -oP '\$[\d.]+(?= USD)' p.txt
grep -P '^(?!WARN).*timeout' p.txt
grep -oP '(?<!\$)\b\d+\b' p.txt | head -6
```

The last command uses a negative lookbehind to find digits not preceded
by a `$`. Look closely at the output and explain why `99` appears even
though `45.99` was on a `$` line.

<details>
<summary>Answer</summary>

`$45` is excluded because `45` is preceded by `$`, but the decimal part
`99` is preceded by a `.`, not a `$`, and `\b` still sees a boundary
between `.` and `9`, so `99` passes both checks. A lookaround tests only
the immediate neighbor; it does not know that `99` belongs to the same
number as `45`. Match whole numbers first (`\d+(?:\.\d+)?`) if the
decimal part should be treated as part of the amount.

</details>

### Task 4 — The anchoring and fixed-length traps

```bash
grep -P  '(?!WARN).*timeout' p.txt
grep -P '^(?!WARN).*timeout' p.txt
grep -oP '(?<=id=\d+ user=)\w+' p.txt; echo "exit=$?"
grep -oP '(?<=id=|uid=)\d+' p.txt
grep -oP 'id=\d+ user=\K\w+' p.txt
```

Confirm the unanchored form prints the `WARN` line too, the
variable-length lookbehind fails with exit status 2, and `\K` solves it.

### Task 5 — Flags, groups, back-references, and possessive quantifiers

```bash
grep -cP '(?i)error' p.txt
printf 'abcabc\nabcabd\n' | grep -P '(?<g>abc)\k<g>'
printf 'aaa\n' | grep -cP 'a+a'
printf 'aaa\n' | grep -cP 'a++a'
```

Explain why the last two commands differ.

### Task 6 — Translate to POSIX and prove the output matches

```bash
grep -oP '(?<=id=)\d+' p.txt > pcre.out
grep -oE 'id=[0-9]+' p.txt | cut -d= -f2 > ere.out
diff pcre.out ere.out && echo "identical"

grep -P '^(?!WARN).*timeout' p.txt > pcre2.out
grep timeout p.txt | grep -v '^WARN' > ere2.out
diff pcre2.out ere2.out && echo "identical"
```

### Task 7 — Feature-detect and fall back

```bash
cat > extract-ids.sh << 'EOF'
#!/bin/bash
# Prints the numbers after "id=" — uses grep -P if available, else ERE.
file="${1:?Usage: $0 FILE}"
if echo 1 | grep -qP '\d' 2>/dev/null; then
    grep -oP '(?<=id=)\d+' "$file"
else
    grep -oE 'id=[0-9]+' "$file" | cut -d= -f2
fi
EOF
bash extract-ids.sh p.txt
```

Confirm the output is `482` and `17`. Then edit the script to force the
`else` branch and confirm it prints the same.

## What You Learned

- ✅ Used `\d`, `\w`, `\s`, and `\b`, and stated what each matches
- ✅ Used lazy quantifiers to match the shortest span
- ✅ Used all four lookaround forms, including the anchoring and
  fixed-length rules
- ✅ Used `\K` with `-o` to extract a value after variable-length context
- ✅ Used inline flags, non-capturing and named groups, back-references,
  and possessive quantifiers
- ✅ Translated each PCRE feature to a POSIX equivalent and identified the
  ones with none
- ✅ Chose between `grep -P` and a portable pipeline, with feature
  detection

## Part 3 — Break-Fix Scenario

### Scenario — NovaTech Ops: two "grep -P" incidents

> **Incident report — NovaTech Ops, ticket #5560**
>
> Alert 1: "An extraction script exits with an unfamiliar `grep` error
> on every run. It worked yesterday, before someone extended the pattern
> to also capture the username." Alert 2: "The alert filter meant to ignore `WARN` lines
> keeps paging on them."

**Incident 1 — the crashing extractor**

```bash
$ grep -oP '(?<=id=\d+ user=)\w+' app.log
grep: lookbehind assertion is not fixed length
```

Tasks:
1. Reference §1.4 — what is the actual rule being violated?
2. Is this failure caused by the log data, or by the pattern? How does
   the exit status help you tell?
3. Give two fixes.

**Incident 2 — the filter that doesn't filter**

```bash
$ grep -P '(?!WARN).*timeout' app.log
ERROR timeout at 10:42
WARN timeout at 10:43
```

Tasks:
1. Reference §1.4 — why does the `WARN` line still match?
2. What is the one-character fix?
3. What non-PCRE command pair gives the same result?

<details>
<summary>Answers</summary>

**Incident 1:** PCRE lookbehinds must be fixed-length, and `\d+` inside
one is open-ended, so the extended pattern is invalid. The earlier version,
`(?<=id=)\d+`, had a fixed-length lookbehind (`id=`), which is why it
worked until the username was added. The failure comes from the pattern,
not the data: the error appears at compile time, on every run regardless
of what the log contains, and the exit status is `2` (the command failed)
rather than `1` (valid command, nothing found). Two fixes: use `\K`
(`id=\d+ user=\K\w+`), which has no length limit, or avoid `-P` for this
step with `grep -oE 'user=[a-z]+' | cut -d= -f2` if `-P` is not guaranteed
on the host.

**Incident 2:** The negative lookahead is unanchored, so the engine can
begin matching at the second character of `WARN timeout...`, where the
following text is `ARN` and does not start with `WARN`, so the lookahead
succeeds. Anchoring with `^` forces it to be checked at the start of the
line only: `grep -P '^(?!WARN).*timeout'`. The portable equivalent is
`grep timeout app.log | grep -v '^WARN'`.

</details>

## Interview Prep

**Q1. What does a lookahead or lookbehind do that an ordinary group cannot?**
It asserts what surrounds a position without consuming it, so the
surrounding text is not part of the match. That is what lets a single
`grep -oP '(?<=id=)\d+'` return only the numbers, where a normal pattern
would return `id=482` and need a second step to strip the label.

---

**Q2. Why must a lookbehind be fixed-length, and what do you use when the left context varies?**
PCRE has to know how many characters to step back before testing the
assertion, so an open-ended quantifier like `\d+` is rejected (`lookbehind
assertion is not fixed length`). `\K` has no such limit, so
`id=\d+ user=\K\w+` handles variable-length left context.

---

**Q3. A script uses `grep -P` and fails on a minimal container image. What are your options?**
Feature-detect it (`echo 1 | grep -qP '\d'`) and fall back to an ERE
pipeline, or avoid `-P` for that script. Most PCRE features have portable
equivalents (POSIX classes, a negated class instead of a lazy quantifier,
`sed -E` captures instead of `\K`); lookaround has none as a single step
and is done as capture-then-strip or a second filter.

---

**Q4. Why can't a script tell "no match" from "bad pattern" by testing for non-zero exit?**
`grep` returns `1` when the pattern is valid and nothing matched, and `2`
when the command failed — for example a malformed or unsupported pattern.
Both are non-zero, so only comparing the specific status separates a
legitimately empty result from a broken command.

---

## Key Takeaways

1. **`grep -P` adds shorthand classes, lazy quantifiers, lookaround, `\K`,
   inline flags, and named groups** — the PCRE features BRE and ERE do not
   have.
2. **Lookaround is zero-width: it checks context without including it in
   the match.** Combined with `-o`, one command returns a bare value
   instead of needing a second step to strip a label.
3. **A negative lookahead needs an anchor.** Without `^`, the engine can
   start one character later and the exclusion silently fails.
4. **A lookbehind must be fixed-length; `\K` has no such limit.** Use `\K`
   when the text before the value varies in length.
5. **Every PCRE feature except lookaround and possessive quantifiers has a
   portable equivalent** — POSIX classes, a negated class for laziness,
   `sed -E` captures, or a second `grep -v` — and lookaround is replaced
   by two steps.
6. **`grep -P` is optional.** Feature-detect it and fall back, and
   remember exit `1` means "no match" while `2` means the command itself
   failed.

> **Demo scope:** Primary concept: PCRE features available through
> `grep -P` and when to use them. Supporting concepts: shorthand classes,
> lazy quantifiers, lookaround, `\K`, inline flags and groups, POSIX
> equivalents, feature detection.
> Estimated completion time: 38 minutes (reading + hands-on + verification).
> Checkpoints: 7 natural stopping points (after Task 1 through Task 7).

---

## Quick Commands Reference

### PCRE syntax

| Syntax | Meaning |
|---|---|
| `\d` `\w` `\s` `\b` | Digit, word character, whitespace, word boundary |
| `.*?` `+?` | Lazy quantifiers |
| `(?=X)` `(?!X)` | Lookahead: must / must not follow |
| `(?<=X)` `(?<!X)` | Lookbehind: must / must not precede (fixed length) |
| `\K` | Reset the reported match start |
| `(?i)` | Case-insensitive from here |
| `(?:X)` | Non-capturing group |
| `(?<n>X)` `\k<n>` | Named group and back-reference |
| `X++` | Possessive quantifier |

### Commands

| Command | Purpose |
|---|---|
| `grep -oP PATTERN file` | Print only what matched |
| `echo 1 \| grep -qP '\d'` | Test whether `-P` is supported |
| `echo $?` | `0` match, `1` none, `2` error |

## Appendix A — Anki Cards

**`m05-02-pcre-regex-anki.csv`:**

```
#deck:Linux Mastery::Module 5 - Text Processing & Search Tools::Demo M05-02 - PCRE and grep -P
#separator:Comma
#columns:Front,Back,Tags
"What does \w match in PCRE?","A word character: a letter, a digit, or an underscore.","demo-m05-02,shorthand"
"What does \b match?","A word boundary — a position between a word character and a non-word character, not a character itself.","demo-m05-02,shorthand"
"What does ? after a quantifier (as in .*?) do?","Makes it lazy — it matches as little as possible instead of as much as possible.","demo-m05-02,lazy"
"What are the four lookaround forms?","(?=X) positive lookahead, (?!X) negative lookahead, (?<=X) positive lookbehind, (?<!X) negative lookbehind.","demo-m05-02,lookaround"
"What does zero-width mean for a lookaround?","It checks the surrounding text without consuming it, so that text is not part of the match.","demo-m05-02,lookaround"
"What does \K do?","Resets the start of the reported match to the current position, discarding everything matched before it.","demo-m05-02,K"
"Why does grep -oP '(?<=id=\d+ user=)\w+' fail, and what is the error's exit status?","PCRE lookbehinds must be fixed-length and \d+ is not. Error: lookbehind assertion is not fixed length; exit status 2.","demo-m05-02,lookbehind,errors"
"What does exit status 1 versus 2 mean for grep?","1: valid command, no match. 2: the command itself failed (e.g. a bad or unsupported pattern).","demo-m05-02,exit-status"
"What does the possessive quantifier a++ do?","Takes as many a's as possible and never gives any back, so a++a can never match.","demo-m05-02,possessive"
"Give the portable replacement for the lazy pattern <a>.*?</a>.","<a>[^<]*</a> — a negated class that stops at the first <.","demo-m05-02,portability"
```

## Appendix B — Quiz

**`m05-02-pcre-regex-quiz.md`:**

````markdown
# Quiz — Demo M05-02: PCRE and grep -P

> One correct answer per question unless stated otherwise.
> Target: 80% or above before moving to Demo M05-03.

---

**Q1.** A support engineer must print just the numeric order IDs from lines
like `order=8841 status=ok`, using one `grep` command with no pipe. Which
command does that?

- A) `grep -E 'order=[0-9]+' log`
- B) `grep -oP '(?<=order=)\d+' log`
- C) `grep -c 'order=' log`
- D) `grep -v 'order=' log`

<details>
<summary>Answer</summary>

**B** — A lookbehind checks `order=` without including it, and `-o`
prints only the digits. Option A would return `order=8841` and need a
second step.

</details>

---

**Q2.** An alert rule should ignore lines starting with `DEBUG` but match
those containing `refused`. The author wrote
`grep -P '(?!DEBUG).*refused' log` and `DEBUG` lines still come through.
What is the fix?

- A) Add `-i`
- B) Anchor it: `grep -P '^(?!DEBUG).*refused' log`
- C) Replace `.*` with `.+`
- D) Switch to `grep -E`

<details>
<summary>Answer</summary>

**B** — Unanchored, the engine can start one character into `DEBUG ...`
where the next text no longer starts with `DEBUG`, so the lookahead passes.
The anchor forces the check at the start of the line.

</details>

---

**Q3.** `grep -oP '(?<=uid=\d+ name=)\w+' users.log` aborts with exit
status 2. What does that status tell you, and what is the best fix?

- A) No lines matched; add `-i`
- B) The pattern itself is invalid (variable-length lookbehind); use `\K`: `grep -oP 'uid=\d+ name=\K\w+' users.log`
- C) The file is unreadable; run with `sudo`
- D) `-o` conflicts with `-P`; remove `-o`

<details>
<summary>Answer</summary>

**B** — Status 2 means the command failed, not that nothing matched
(that would be 1). The lookbehind contains `\d+`, which has no fixed
length; `\K` has no such limit.

</details>

---

**Q4.** A script written with `grep -P` fails on a stripped-down container
image. Which approach keeps it working on both that image and normal hosts?

- A) Assume `-P` exists everywhere
- B) Feature-detect with `echo 1 | grep -qP '\d'` and fall back to an ERE pipeline when it is unavailable
- C) Add `set -e`
- D) Rewrite every pattern to use `\d`

<details>
<summary>Answer</summary>

**B** — Detecting and falling back avoids an obscure failure. `set -e`
would just stop the script, and `\d` is itself PCRE-only.

</details>

---

**Q5.** Which statement about replacing PCRE features with POSIX tools is
correct?

- A) Every PCRE feature has a single-step POSIX equivalent
- B) Lazy quantifiers can use a negated class such as `<a>[^<]*</a>`, but lookaround has no single-step equivalent and is done as capture-then-strip or a second filter
- C) Lookaround is available in `grep -E`
- D) `\d` is available in ERE but `[0-9]` is not

<details>
<summary>Answer</summary>

**B** — A negated class stops at the first closing delimiter, giving
lazy-like behavior; lookaround's zero-width test needs two steps without
PCRE.

</details>

---

**Q6.** Matching `aaa`, which pattern fails to match, and why?

- A) `a+a`, because `+` is greedy
- B) `a++a`, because the possessive `a++` keeps all three `a`'s and leaves none for the final `a`
- C) Both fail
- D) Neither fails

<details>
<summary>Answer</summary>

**B** — The greedy `a+` gives one `a` back so the final `a` can match;
the possessive form never gives anything back.

</details>

---

**Q7.** Digits following `$` should be excluded, but
`grep -oP '(?<!\$)\b\d+\b'` still prints `99` for the text `$45.99`. Why?

- A) `\b` is broken
- B) `99` is preceded by a `.`, not a `$`, so it passes the check — a lookaround tests only its immediate neighbor
- C) `-o` reorders digits
- D) Lookbehinds only work on letters

<details>
<summary>Answer</summary>

**B** — The assertion looks at the single character before `99`, which is
the decimal point. Match whole numbers first if the decimal part should
belong to the amount.

</details>

---

Score guide:

| Score | Action |
|---|---|
| 7/7 | Import Anki cards, move to Demo M05-03 |
| 6/7 | Review the wrong answer, then proceed |
| 5/7 | Re-read the relevant section, retry those questions |
| Below 5/7 | Re-read the full demo and redo the walkthrough before proceeding |
````