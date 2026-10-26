# Demo M05-03 — Regex Flavor Differences Across Tools

## Overview

A pattern that works perfectly when tested in `grep -E` during
development, then silently matches nothing once it's pasted into a `sed`
script in the deployed pipeline, is one of the quietest ways a log-parsing
job can break: no error, no warning, just empty output. The cause is that
"regular expression" is not one language. `grep`, `sed`, `awk`, `grep -P`,
and Vim each implement a *flavor* — a slightly different rule set for
which characters are operators and which are literals — and the same
characters mean different things depending on the tool and its flags.

This demo gives you one comparison table, built from real runs, so you
can predict what a pattern will do in each tool before it ships.

## Demo Objectives

1. Name the regex flavor each common tool uses by default, and the flag
   that switches it
2. Predict whether a given pattern will match in each tool, for the
   operators `+`, `?`, `|`, `( )`, and `{n}`
3. Reproduce and diagnose the "pattern silently matches nothing" bug, and
   explain why `set -e` cannot catch it
4. Explain what GNU's backslash-escaped operators (`\+`, `\?`, `\|`) do in
   BRE, and why they are not portable
5. Identify PCRE-only features (`\d`, lazy quantifiers, lookaround) and
   what each other tool does when handed one
6. Standardize a pipeline on one flavor and document which one is in use

## Directory Structure

```
03-regex-flavors/
├── README.md                      # this file
├── m05-03-regex-flavors-anki.csv  # Anki flashcard deck
└── m05-03-regex-flavors-quiz.md   # standalone quiz
```

## Environments

| Task | Ubuntu 24.04 | Rocky Linux 9.8 | EC2 Amazon Linux |
|---|---|---|---|
| `grep`, `sed`, `awk` | ✅ GNU grep 3.11, GNU sed 4.9, **mawk** 1.3.4 as `awk` | ✅ GNU grep/sed; `awk` is expected to be **gawk** `[Unverified — test before use]` | Not required |
| `grep -P` | ✅ (PCRE support compiled in) | ✅ `[Unverified — test before use]` | Not required |
| Vim regex | `vim` (see §1.7) | `vim` (see §1.7) | Not required |

> **Verification note:** every `grep`, `sed`, and `awk` result in §1.2–1.6
> and every Lab task was run live in the authoring sandbox (Ubuntu 24.04,
> GNU grep 3.11, GNU sed 4.9, mawk 1.3.4 20240123) and the outputs shown
> are real. Three things were not testable there: **gawk** (Rocky's
> default `awk` — behavior may differ from mawk, see §1.6), **Vim's regex
> dialect** (Vim isn't installed — §1.7 is documentation-sourced), and
> macOS/BSD tools (not covered by this demo). Those are marked
> `[Unverified — test before use]` where they appear.

## Recall Check — 02-pcre-regex

Answer these from memory before continuing — no looking back at Demo M05-02.

**Q1.** What does a lookahead or lookbehind do that an ordinary group
cannot?

**Q2.** Why must a lookbehind be fixed-length, and what do you use when
the text before the value varies in length?

**Q3.** Which PCRE features have no single-step portable replacement?

<details>
<summary>Answers</summary>

**A1.** It asserts what surrounds a position without consuming it, so the
surrounding text is not part of the match — with `-o`, one command returns
a bare value instead of needing a second step to strip a label.

**A2.** PCRE has to know how far back to look, so an open-ended quantifier
like `\d+` is rejected (`lookbehind assertion is not fixed length`, exit
status 2). `\K` has no length limit.

**A3.** Lookaround and possessive quantifiers. Lookaround is replaced by
two steps (capture then strip, or a second filter); the other PCRE
features have direct POSIX equivalents.

</details>

---

## Part 1 — Concepts

### 1.1 Why there is more than one flavor

POSIX defines two regular-expression syntaxes: **BRE** (Basic) and **ERE**
(Extended). **PCRE** (Perl-Compatible Regular Expressions) is a separate,
richer syntax implemented as a library. Each tool picks one of these as
its default, and a few tools can switch between them with a flag. The
patterns look similar enough that the differences stay invisible until one
of them silently produces the wrong result.

The single most important difference is **which characters are operators
by default**. In BRE, `+ ? | ( ) { }` are plain literal characters, and
you add a backslash to turn them into operators. In ERE it is the other
way around: they are operators by default, and you add a backslash to make
them literal.

### 1.2 Which tool uses which flavor

| Tool and flags | Flavor | Verified here |
|---|---|---|
| `grep` | BRE | ✅ |
| `grep -E` | ERE | ✅ |
| `grep -P` | PCRE | ✅ |
| `grep -F` | none — plain fixed string, no operators at all | ✅ |
| `sed` | BRE | ✅ |
| `sed -E` | ERE | ✅ |
| `awk` | ERE | ✅ (mawk 1.3.4) |
| Vim `/` and `:s` | Vim's own dialect (§1.7) | ❌ not installed |

### 1.3 The operator comparison table

One pattern *meaning* per row — the form you have to type differs by tool.
Every cell below was confirmed with a real run against a small test file.

```
Meaning        grep / sed (BRE)   grep -E / sed -E / awk (ERE)   grep -P (PCRE)
one or more    \+                 +                              +
zero or one    \?                 ?                              ?
alternation    \|                 |                              |
grouping       \( \)              ( )                            ( )
interval       \{3\}              {3}                            {3}
```

Real runs, against a file containing `color`, `colour`, `cat`, `dog`,
`aaa`, `abab`:

```
$ grep 'colou?r' words.txt            # BRE: ? is a literal question mark
(no output, exit 1)
$ grep 'colou\?r' words.txt           # BRE with the GNU escape
color
colour
$ grep -E 'colou?r' words.txt         # ERE
color
colour
$ grep 'cat|dog' words.txt            # BRE: | is a literal bar
(no output, exit 1)
$ grep 'cat\|dog' words.txt
cat
dog
$ grep 'a{3}' words.txt               # BRE: braces are literal
(no output, exit 1)
$ grep 'a\{3\}' words.txt
aaa
```

The same results held for `sed` (BRE with escapes, `sed -E` without),
`awk`, and `grep -P`.

**The backslash forms `\+`, `\?`, and `\|` in BRE are GNU extensions, not
POSIX.** They work in GNU `grep` and GNU `sed` (as confirmed above) but
are not guaranteed on other systems. A script that must run on a non-GNU
`sed` should use `sed -E` — or, better, avoid depending on either spelling
by standardizing on one flavor (§1.8).

### 1.4 The silent failure, reproduced

This is the exact incident the demo is built around. A sample log:

```
user_id=482
session+id=91
```

```
$ grep -E 'user_id=[0-9]+' sample.log
user_id=482                             # exit 0

$ sed -n '/user_id=[0-9]+/p' sample.log
                                        # no output, exit 0   <-- the bug

$ sed -nE '/user_id=[0-9]+/p' sample.log
user_id=482                             # exit 0

$ grep 'user_id=[0-9]+' sample.log
                                        # no output, exit 1
```

Plain `sed` treats `+` as a literal plus sign, so it is looking for the
text `user_id=` followed by one digit followed by a literal `+` — which
never appears. Nothing is wrong with the data, the pattern, or the tool;
the pattern was simply written in one flavor and run in another.

**Notice the exit codes.** Plain `grep` exits `1` when nothing matches,
which at least gives a script something to react to. `sed -n '/pat/p'`
exits `0` whether or not anything matched — and so does `awk '/pat/'` —
because both are *processing* tools that succeeded at processing, not
*search* tools reporting whether they found something. This has a direct
consequence for Demo M09-02: **`set -e` and `pipefail` cannot catch this bug**,
because from the shell's point of view nothing failed. Even the
`sed -n '/zzz/{p;q}'` form still exits `0` on no match.

### 1.5 The mismatch goes both ways

The failure above is "operators treated as literals." The reverse also
happens, when a pattern meant to match a literal character is run in a
flavor where that character is an operator:

```
$ grep 'session+id' sample.log           # BRE: + is literal
session+id=91

$ grep -E 'session+id' sample.log        # ERE: + means "one or more n"
                                         # no output, exit 1
```

In ERE, `session+id` means `sessio`, then one or more `n`, then `id` — it
no longer matches the literal text `session+id`. To match a literal `+` in
ERE you escape it (`session\+id`) or put it in a bracket expression
(`session[+]id`), and both were confirmed to match. If you simply want the
literal string and no regex at all, `grep -F 'session+id'` does it with no
escaping to get wrong.

**A mismatch does not always fail quietly.** In a `sed` substitution with
a back-reference, the wrong flavor produces a loud error, because BRE sees
no group for `\2` to refer to:

```
$ echo ab12 | sed 's/\(ab\)\(12\)/\2\1/'      # BRE, groups escaped
12ab
$ echo ab12 | sed -E 's/(ab)(12)/\2\1/'       # ERE, groups bare
12ab
$ echo ab12 | sed 's/(ab)(12)/\2\1/'          # BRE, groups bare = literals
sed: -e expression #1, char 16: invalid reference \2 on `s' command's RHS
```

So the dangerous case is specifically a **match or filter** with the wrong
flavor — no error, no output — while a substitution that uses `\1`/`\2`
tends to fail loudly.

### 1.6 PCRE-only features, and what the other tools do with them

`grep -P` adds shorthand classes, lazy quantifiers, and lookaround that
BRE and ERE simply do not have. The risk is what happens when you hand one
of those to a tool that does not understand it.

| Feature | `grep -P` | `grep -E` | `sed -E` | `awk` (mawk) |
|---|---|---|---|---|
| `\d` (digit) | ✅ matches | no match, silently | no match, silently | no match, silently |
| `\s` (whitespace) | ✅ | ✅ (GNU ext.) | ✅ (GNU ext.) | **no match** |
| `\b` (word boundary) | ✅ | ✅ (GNU ext.) | — not tested | — not tested |
| `\<` `\>` word edges | — | — not tested | — not tested | **no match** |
| `[[:space:]]`, `[[:digit:]]` | ✅ | ✅ | ✅ | ✅ |

```
$ grep -P 'ab\d+' words.txt          # works
ab12
$ grep -E 'ab\d+' words.txt          # no output, exit 1
$ sed -nE '/ab\d+/p' words.txt       # no output, exit 0
$ awk '/foo\sbar/' t.txt             # no output (mawk has no \s)
$ awk '/foo[[:space:]]bar/' t.txt    # works
foo bar
```

**The portable choice is the POSIX bracket classes** — `[[:digit:]]`,
`[[:space:]]`, `[[:alpha:]]` — or a plain `[0-9]`. They worked in every
tool tested.

**Lazy quantifiers and lookaround are PCRE-only.** Against the line
`<b>one</b><b>two</b>`:

```
$ grep -oE '<b>.*</b>' h.txt          # greedy: one big match
<b>one</b><b>two</b>
$ grep -oP '<b>.*?</b>' h.txt         # lazy: two separate matches
<b>one</b>
<b>two</b>
$ grep -oE '<b>.*?</b>' h.txt         # ERE: no error, but NOT lazy
<b>one</b><b>two</b>
$ grep -oP '(?<=<b>)[a-z]+(?=</b>)' h.txt    # lookaround, PCRE only
one
two
```

The third command is the instructive one: `grep -E` accepted `.*?` without
complaint and returned the *greedy* result. The syntax looks like it works
and silently does something different. Same lesson as §1.4, different
operator.

**"awk" is not a single flavor.** The sandbox's `awk` is **mawk**, which
supports `{n}` intervals (confirmed) and POSIX classes but not `\s`, `\<`,
or `\>`. gawk — the `awk` on RHEL-family systems such as Rocky — adds GNU
extensions that mawk lacks, so the same awk script can behave differently
between Ubuntu and Rocky `[Unverified — test before use]`. When an awk
pattern must run on both, use POSIX classes.

### 1.7 Vim's dialect

Vim's `/` search and `:s` substitute (Demo M06-03) use Vim's own regex
dialect, not `sed`'s. By default (Vim calls this "magic" mode) it behaves
like BRE: `+`, `?`, `|`, and `( )` need a backslash to be operators.
Prefixing the pattern with `\v` ("very magic") flips most of them to
ERE-style bare operators `[Unverified — test before use: Vim isn't
installed in the authoring sandbox]`.

The practical rule: a pattern you perfected in Vim is **not** safe to paste
into `sed` unchanged, and a `sed` script is not safe to paste into
`:%s///` unchanged — treat each as its own flavor and re-test.

### 1.8 Remediation and standards

- **Standardize on ERE** (`grep -E`, `sed -E`, `awk`) across a pipeline so
  one mental model applies everywhere. This also dodges the GNU-only
  backslash extensions.
- **If BRE is unavoidable**, escape the operators explicitly (`\+`, `\?`,
  `\|`, `\(...\)`) instead of relying on memory of which flavor is active.
- **Prefer POSIX classes** (`[[:digit:]]`, `[[:space:]]`) over `\d`/`\s`
  when the same pattern must run in more than one tool.
- **Write the flavor in a comment** above any non-trivial pattern, for
  example `# ERE (sed -E)`.
- **Test with the same tool and flags the script will use.** A pattern
  verified in `grep -E` has not been verified for `sed`.
- **Treat "matches nothing, no error" as a flavor red flag** before
  suspecting the data changed. This class of bug produces no error message
  by design.

## Part 2 — Hands-On Lab

### Commands used in this demo

| Command | Flag meaning | Purpose |
|---|---|---|
| `grep PATTERN file` | — | BRE search |
| `grep -E PATTERN file` | `-E`: Extended | ERE search |
| `grep -P PATTERN file` | `-P`: Perl-compatible | PCRE search |
| `grep -F STRING file` | `-F`: Fixed string | Literal match, no regex |
| `grep -o PATTERN file` | `-o`: only matching | Print just the matched part |
| `sed -n '/PAT/p' file` | `-n`: no auto-print | BRE filter |
| `sed -nE '/PAT/p' file` | `-E`: Extended | ERE filter |
| `awk '/PAT/' file` | — | ERE filter (flavor of your awk) |
| `awk --version` | — | Identify which awk you have |

### Task 1 — Reproduce the silent failure

```bash
printf 'user_id=482\nsession+id=91\n' > sample.log
grep -E 'user_id=[0-9]+' sample.log; echo "exit=$?"
sed -n '/user_id=[0-9]+/p' sample.log; echo "exit=$?"
sed -nE '/user_id=[0-9]+/p' sample.log; echo "exit=$?"
```

Confirm `grep -E` and `sed -E` match, plain `sed` prints nothing, and plain
`sed` still exits `0`.

### Task 2 — Build the operator table yourself

```bash
printf 'color\ncolour\ncat\ndog\naaa\nabab\n' > words.txt
grep 'colou?r'      words.txt; echo "exit=$?"
grep 'colou\?r'     words.txt
grep -E 'colou?r'   words.txt
sed -n '/colou?r/p' words.txt
sed -nE '/colou?r/p' words.txt
awk '/colou?r/'     words.txt
```

Repeat with `cat|dog`, `a{3}`, and `(ab)+`, writing the correct form for
each tool, then check yourself against §1.3.

### Task 3 — The mismatch in the other direction

```bash
grep    'session+id' sample.log; echo "exit=$?"
grep -E 'session+id' sample.log; echo "exit=$?"
grep -E 'session\+id' sample.log
grep -E 'session[+]id' sample.log
grep -F 'session+id' sample.log
```

Explain, for each, whether `+` was acting as a literal or an operator.

### Task 4 — A loud failure versus a silent one

```bash
echo ab12 | sed 's/\(ab\)\(12\)/\2\1/'
echo ab12 | sed -E 's/(ab)(12)/\2\1/'
echo ab12 | sed 's/(ab)(12)/\2\1/'
```

Confirm the third command errors out. Why does this mismatch fail
loudly when Task 1's did not?

### Task 5 — PCRE-only features across tools

```bash
printf 'ab12\nfoo bar\n' > t.txt
grep -P 'ab\d+' t.txt
grep -E 'ab\d+' t.txt; echo "exit=$?"
awk '/foo\sbar/' t.txt
awk '/foo[[:space:]]bar/' t.txt
printf '<b>one</b><b>two</b>\n' > h.txt
grep -oE '<b>.*</b>'  h.txt
grep -oP '<b>.*?</b>' h.txt
grep -oE '<b>.*?</b>' h.txt
```

Identify which command accepted syntax it does not implement, and what it
did instead.

### Task 6 — Identify your awk

```bash
awk --version 2>&1 | head -1
awk -W version 2>&1 | head -1
ls -l /etc/alternatives/awk
```

Record whether you have mawk or gawk, and test `awk '/foo\sbar/' t.txt`
against it.

### Task 7 — Audit a pipeline script

```bash
cat > /tmp/pipeline.sh << 'EOF'
#!/bin/bash
grep -E 'user_id=[0-9]+' app.log
sed -n '/code=(404|500)/p' app.log
sed -n '/retry[0-9]+/p' app.log
awk '/timeout\s+[0-9]+/' app.log
EOF
```

For each line, state which flavor it is written for, whether it will behave
as intended, and the one-line fix.

<details>
<summary>Answer</summary>

Line 1 is correct ERE for `grep -E`. Line 2 is ERE alternation and grouping
run under plain `sed` (BRE): the `|` and `( )` are literals, so it silently
matches nothing — fix with `sed -nE`. Line 3 uses `+` under plain `sed`:
the `+` is a literal, so it matches nothing — fix with `sed -nE` or
`retry[0-9]\+`. Line 4 uses `\s`, which mawk does not implement — fix by
using `[[:space:]]+`, which works in both mawk and gawk.

</details>

## What You Learned

- ✅ Named the regex flavor each common tool uses by default and the flag
  that switches it
- ✅ Predicted whether a pattern matches in each tool for `+`, `?`, `|`,
  `( )`, and `{n}`
- ✅ Reproduced the "silently matches nothing" bug and explained why
  `set -e` cannot catch it
- ✅ Explained GNU's backslash-escaped BRE operators and why they are not
  portable
- ✅ Identified PCRE-only features and what each other tool does when given
  one
- ✅ Standardized a pipeline on one flavor and documented which one

## Part 3 — Break-Fix Scenario

### Scenario — NovaTech Ops: two "the pattern worked in testing" incidents

> **Incident report — NovaTech Ops, ticket #5520**
>
> Alert 1: "The error-code extraction step of the log pipeline produces an
> empty report every night since last week's refactor. The same pattern
> returns results when tested by hand." Alert 2: "A search for the literal
> string `session+id` returns nothing, even though `grep` finds it in the
> same file when run without any flags."

**Incident 1 — the refactor that moved a pattern into sed**

```bash
# before the refactor (ran fine)
grep -E 'code=(404|500)' app.log

# after the refactor (deployed)
sed -n '/code=(404|500)/p' app.log
```

Tasks:
1. Reference §1.3/§1.4 — why does the deployed command produce empty
   output instead of an error?
2. The pipeline script uses `set -euo pipefail`. Why did it not stop the
   job?
3. Give two valid fixes and the one check that would have caught this
   before deployment.

**Incident 2 — the literal plus sign**

```bash
$ grep 'session+id' app.log
session+id=91
$ grep -E 'session+id' app.log
$
```

Tasks:
1. Reference §1.5 — why does adding `-E` make the match disappear?
2. What does `session+id` mean to `grep -E`?
3. Give three different ways to match the literal text under `-E`.

<details>
<summary>Answers</summary>

**Incident 1:** Plain `sed` is BRE, so `|`, `(`, and `)` are literal
characters and the pattern is looking for the literal text
`code=(404|500)`, which never occurs — and `sed` does not treat "no line
matched" as an error. Under `set -euo pipefail` nothing fails either: `sed`
exits `0` whether or not it printed anything (§1.4), so neither `-e` nor
`pipefail` has a failing status to react to. Valid fixes: switch to
`sed -nE '/code=(404|500)/p'`, or keep BRE and escape the operators
(`code=\(404\|500\)`). The check that would have caught it is running the
exact deployed command against a known-good sample file and verifying it
produces output — testing in `grep -E` verified a different tool.

**Incident 2:** In ERE, `+` is a quantifier, so `session+id` means
`sessio`, then one or more `n`, then `id`. That does not match the literal
text `session+id`, which contains an actual plus sign. Plain `grep` (BRE)
treated `+` as a literal, which is why it matched. Three ways to match the
literal under `-E`: escape it (`session\+id`), put it in a bracket
expression (`session[+]id`), or drop regex entirely with `grep -F
'session+id'`.

</details>

## Interview Prep

**Q1. A pattern works in `grep -E` but matches nothing in `sed`, with no error. What's the most likely cause?**
The two tools default to different flavors: `grep -E` is ERE, while plain
`sed` is BRE, where `+ ? | ( ) { }` are literals. The pattern isn't broken
— it's being read as a different language. Switching to `sed -E`, or
escaping the operators, fixes it.

---

**Q2. Why can't `set -e` catch a regex flavor mismatch?**
Because nothing fails from the shell's perspective. `sed` and `awk` exit
`0` whether or not a pattern matched, since they are processing tools, not
search tools — only `grep` uses exit status `1` to mean "nothing found."
With no failing exit status there is nothing for `-e` or `pipefail` to
react to; the only reliable guard is checking the output (or a match
count) against a known-good sample.

---

**Q3. Why prefer `[[:digit:]]` over `\d` in a script that must run in several tools?**
`\d` is PCRE shorthand: it works in `grep -P` but silently matches nothing
in `grep -E`, `sed -E`, and `awk`. The POSIX bracket classes worked in
every tool tested, so they are the portable spelling.

---

**Q4. Is "awk" one regex flavor?**
No. awk implementations differ: Ubuntu's default `awk` is mawk, which
supports POSIX classes and `{n}` intervals but not `\s` or word-boundary
escapes, while RHEL-family systems ship gawk, which adds GNU extensions.
A pattern that works under one may not under the other, so use POSIX
classes when it must run on both and check `awk --version` when something
behaves differently between hosts.

---

## Key Takeaways

1. **"Regex" is several languages.** `grep` and `sed` default to BRE,
   `grep -E`/`sed -E`/`awk` to ERE, `grep -P` to PCRE, and Vim to its own
   dialect — a pattern is only verified for the exact tool and flags it ran
   under.
2. **In BRE, `+ ? | ( ) { }` are literals; in ERE they are operators.** The
   same characters flip meaning, which is why a pattern can silently match
   nothing, or silently stop matching a literal, depending on the flag.
3. **`sed` and `awk` exit `0` on no match; only `grep` exits `1`.** A
   flavor mismatch in a filter produces empty output with a success status,
   so `set -e` and `pipefail` cannot catch it.
4. **Backslash-escaped BRE operators (`\+`, `\?`, `\|`) are GNU
   extensions**, not POSIX — standardizing on ERE avoids depending on them.
5. **PCRE-only syntax fails quietly elsewhere.** `\d` matches nothing in
   `grep -E`/`sed -E`/`awk`, and `.*?` in `grep -E` is accepted but is not
   lazy. POSIX classes like `[[:digit:]]` and `[[:space:]]` are portable.
6. **Standardize on one flavor, comment it, and test with the deployed
   tool.** Treat "matches nothing, no error" as a flavor red flag before
   suspecting the data.

> **Demo scope:** Primary concept: predicting regex behavior across tools
> by flavor. Supporting concepts: the BRE/ERE operator table, the silent
> no-match failure and its exit-status consequence, PCRE-only features,
> mawk vs. gawk, Vim's dialect as a forward reference.
> Estimated completion time: 35 minutes (reading + hands-on + verification).
> Checkpoints: 7 natural stopping points (after Task 1 through Task 7).

---

## Quick Commands Reference

### Choosing a flavor

| Command | Flavor |
|---|---|
| `grep`, `sed` | BRE |
| `grep -E`, `sed -E`, `awk` | ERE |
| `grep -P` | PCRE |
| `grep -F` | None — fixed string |

### Operator spellings

```
Meaning        BRE (GNU)    ERE     PCRE
one or more    \+           +       +
zero or one    \?           ?       ?
alternation    \|           |       |
group          \( \)        ( )     ( )
interval       \{n\}        {n}     {n}
```

### Portable classes

| Class | Matches |
|---|---|
| `[[:digit:]]` | a digit — use instead of `\d` |
| `[[:space:]]` | whitespace — use instead of `\s` |
| `[[:alpha:]]` | a letter |

### Diagnosing

| Command | Purpose |
|---|---|
| `awk --version` | Identify mawk vs. gawk |
| `echo $?` after the command | grep `1` = no match; sed/awk `0` regardless |

## Appendix A — Anki Cards

**`m05-03-regex-flavors-anki.csv`:**

```
#deck:Linux Mastery::Module 5 - Text Processing & Search Tools::Demo M05-03 - Regex Flavor Differences
#separator:Comma
#columns:Front,Back,Tags
"Which regex flavor does plain sed use by default?","BRE (Basic Regular Expressions). sed -E switches it to ERE.","demo-m05-03,flavors,sed"
"Which regex flavor does plain grep use by default, and which flag gives ERE and which gives PCRE?","BRE by default; grep -E for ERE; grep -P for PCRE.","demo-m05-03,flavors,grep"
"In BRE, which of + ? | ( ) { } are operators by default?","None — they are literal characters. A backslash turns each into an operator (GNU extension for \+ \? \|).","demo-m05-03,bre,operators"
"In ERE, how do you match a literal plus sign?","Escape it (\+) or put it in a bracket expression ([+]). Or use grep -F to skip regex entirely.","demo-m05-03,ere,literals"
"Which exit status does grep return when nothing matches, and what do sed and awk return?","grep returns 1; sed and awk return 0 regardless of whether anything matched.","demo-m05-03,exit-status"
"What does \d match, and which of grep -P, grep -E, sed -E, awk understand it?","A digit. Only grep -P; the others accept it but silently match nothing.","demo-m05-03,pcre,shorthand"
"What is the portable replacement for \d and \s?","POSIX bracket classes: [[:digit:]] and [[:space:]] (or [0-9] for digits).","demo-m05-03,portability,posix-classes"
"What is the difference between grep -F and grep with a pattern?","-F treats the pattern as a fixed literal string with no operators at all; otherwise the pattern is interpreted as BRE (or ERE/PCRE with -E/-P).","demo-m05-03,grep,fixed-string"
"Name two features that exist only in PCRE (grep -P) and not in BRE or ERE.","Lazy quantifiers (.*?) and lookahead/lookbehind assertions such as (?<=...) and (?=...).","demo-m05-03,pcre,features"
"Is awk one regex flavor? Name the two common implementations.","No. mawk (Ubuntu default) and gawk (RHEL-family default) differ — e.g. mawk has no \s or \< \>.","demo-m05-03,awk,mawk-gawk"
```

## Appendix B — Quiz

**`m05-03-regex-flavors-quiz.md`:**

````markdown
# Quiz — Demo M05-03: Regex Flavor Differences Across Tools

> One correct answer per question unless stated otherwise.
> Target: 80% or above before moving to Demo M05-05.

---

**Q1.** A nightly job's `sed -n '/id=[0-9]+/p' app.log` step returns
nothing, though the same text was matched by hand with `grep -E`. The job
runs under `set -euo pipefail` and reports success. What's the correct
explanation?

- A) `set -e` is broken on `sed`
- B) Plain `sed` is BRE, so `+` is a literal and nothing matches; `sed` exits `0` on no match, so there was no failure for `set -e` to catch
- C) `sed` needs `sudo` to read log files
- D) `grep -E` and `sed` use the same flavor, so the data must have changed

<details>
<summary>Answer</summary>

**B** — Two facts combine: BRE reads `+` as a literal, and `sed` reports
success even when nothing matched, leaving nothing for `-e` or `pipefail`
to react to.

</details>

---

**Q2.** Which single change makes `sed -n '/code=(404|500)/p'` do what its
author intended?

- A) Add `sudo`
- B) Change `sed` to `sed -nE` so `|` and `( )` are operators
- C) Replace `p` with `P`
- D) Remove the parentheses and leave the `|`

<details>
<summary>Answer</summary>

**B** — `-E` selects ERE. Option D leaves a literal `|` in BRE, still
matching nothing useful.

</details>

---

**Q3.** `grep 'session+id' app.log` finds the line, but
`grep -E 'session+id' app.log` finds nothing. Why?

- A) `-E` disables searching the file
- B) In ERE, `+` is a quantifier on `n`, so the pattern no longer means the literal text `session+id`
- C) `-E` only works on numbers
- D) Plain `grep` is case-insensitive

<details>
<summary>Answer</summary>

**B** — In BRE `+` is a literal character; in ERE it's "one or more of
the previous character."

</details>

---

**Q4.** Which of these would find the literal text `session+id` under `-E`
*without* relying on any regex escaping at all?

- A) `grep -E 'session\+id'`
- B) `grep -E 'session[+]id'`
- C) `grep -F 'session+id'`
- D) `grep -P 'session+id'`

<details>
<summary>Answer</summary>

**C** — `-F` is a fixed-string search with no operators. A and B escape the
plus; D is PCRE, where `+` is still a quantifier.

</details>

---

**Q5.** A script runs `grep -E 'ab\d+'` and `sed -nE '/ab\d+/p'` on a file
containing `ab12`. What happens?

- A) Both print `ab12`
- B) Both print nothing — `\d` is PCRE shorthand and is accepted but silently matches nothing in these tools
- C) Both fail with a syntax error
- D) Only `sed -nE` prints `ab12`

<details>
<summary>Answer</summary>

**B** — Only `grep -P` understands `\d`. The portable fix is `[0-9]` or
`[[:digit:]]`.

</details>

---

**Q6.** An engineer replaces `<b>.*</b>` with `<b>.*?</b>` in `grep -oE`
expecting each tag pair to be matched separately. What happens?

- A) Two separate matches are printed
- B) `grep -E` accepts the pattern but `.*?` is not lazy in ERE, so the greedy single match is still returned
- C) `grep` prints a syntax error
- D) The command hangs

<details>
<summary>Answer</summary>

**B** — Lazy quantifiers are PCRE. Under `-E` there's no error, just the
unchanged greedy result. Switch to `grep -oP`.

</details>

---

**Q7.** A shared awk script works on an Ubuntu host but silently matches
nothing on another host for the pattern `/timeout\s+[0-9]+/`. What's the
most likely cause and the portable fix?

- A) `awk` isn't installed on one host
- B) The two hosts run different awk implementations (mawk vs. gawk) and mawk has no `\s`; use `[[:space:]]+` instead
- C) `awk` can't match digits
- D) The script needs `set -e`

<details>
<summary>Answer</summary>

**B** — Implementations differ. The POSIX class works in both.

</details>

---

Score guide:

| Score | Action |
|---|---|
| 7/7 | Import Anki cards, move to Demo M05-05 |
| 6/7 | Review the wrong answer, then proceed |
| 5/7 | Re-read the relevant section, retry those questions |
| Below 5/7 | Re-read the full demo and redo the walkthrough before proceeding |
````