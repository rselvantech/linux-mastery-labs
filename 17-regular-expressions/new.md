# Demo 17 — Regular Expressions

## Overview

This demo is deliberately standalone, taught before grep, sed, or awk —
because all three depend on regular expressions, and each has its own
quirks about which regex flavor it speaks by default. Getting comfortable
with the underlying pattern language first means the next three demos can
focus on what each tool *does* with a match, not re-explain regex syntax
three times. By the end, you'll read and write patterns confidently and
know exactly which flavor a given tool expects.

## Demo Objectives

1. Explain the difference between Basic and Extended Regular Expressions (BRE/ERE)
2. Use anchors to match position, not just content
3. Use character classes and their POSIX named forms correctly
4. Use quantifiers to control repetition, including open-ended intervals
   and the greedy-by-default trap
5. Use backreferences to match repeated content within a single pattern
6. Identify which regex flavor `grep`, `sed`, and `awk` each use by default,
   and recognize which conveniences are POSIX-standard versus GNU extensions

## Directory Structure

```
17-regular-expressions/
├── README.md                          # this file
├── 17-regular-expressions-anki.csv    # Anki flashcard deck
└── 17-regular-expressions-quiz.md     # standalone quiz
```

## Environments

| Task | Ubuntu 24.04 | Rocky Linux 9.8 | EC2 Amazon Linux |
|---|---|---|---|
| All lab tasks | ✅ | ✅ | Not required |

## Recall Check — 16-remote-storage-troubleshooting

Answer these from memory before continuing — no looking back at Demo 16.

1. Why is `_netdev` mandatory for a network-backed `/etc/fstab` entry,
   not just a best practice?

2. An NFS mount uses `hard`, and a file access hangs indefinitely when
   the server is unreachable. Is this a bug?

3. Why must `fsck` never be run on a mounted filesystem?

<details>
<summary>Answers</summary>

1. Without it, the boot process doesn't know the mount depends on
   networking being ready first, and may attempt it too early — since network
   mounts can block indefinitely rather than fail fast, this risks hanging the
   whole boot sequence.

2. No — this is expected `hard`-mount behavior, designed to block and
   eventually complete an operation rather than fail it, when the server is
   unreachable.

3. The filesystem's live in-memory and on-disk state can diverge while
   mounted, and running `fsck` against it risks causing the exact corruption
   it's meant to detect and fix.

</details>

## Part 1 — Concepts

### 1.1 BRE vs ERE — two different pattern languages, not two "modes" of one

POSIX defines two related but distinct regex dialects. The practical
difference is which characters need a backslash to gain "special" meaning:

| Feature | BRE (Basic) | ERE (Extended) |
|---|---|---|
| One-or-more | `\+` | `+` |
| Zero-or-one | `\?` | `?` |
| Alternation (OR) | `\|` | `|` |
| Grouping | `\(...\)` | `(...)` |

In BRE, `+`, `?`, `(`, `)`, and `|` are literal characters unless
backslash-escaped; in ERE, they're special unless backslash-escaped to make
them literal — **the escaping is inverted between the two**, which is
exactly the kind of thing that produces a pattern that "should work" but
silently doesn't, because it was written for the wrong flavor.

```bash
echo "color" | grep 'colou\?r'      # BRE: \? makes u optional
echo "color" | grep -E 'colou?r'    # ERE: ? is already special, no backslash
```

**One more precision worth having:** `\|` for alternation in BRE isn't
actually part of the POSIX standard — strict POSIX BRE has no standard
alternation operator at all. `\|` working in `grep`/`sed` is a widely
supported GNU extension, reliable on the Linux systems this series targets
but not guaranteed on every BRE implementation you might encounter
elsewhere. ERE's plain `|`, by contrast, genuinely is POSIX-standard —
there's no portability asterisk on that one.

### 1.2 Anchors — matching position, not content

```bash
grep '^root' /etc/passwd        # ^ : start of line
grep 'bash$' /etc/passwd        # $ : end of line
grep '^root:.*bash$' /etc/passwd  # combine: whole-line shape
```

**Anchors match a *position*, not a character** — `^` doesn't consume any
input, it asserts "we are at the start of the line right now." This
distinction matters once you start combining anchors with quantifiers:
`^$` matches an empty line exactly (start immediately followed by end),
which is a genuinely useful and common pattern for "find blank lines," not
a construct that would make sense if `^`/`$` were ordinary characters.

```bash
grep -w 'root' /etc/passwd
# -w: word boundary — matches "root" as a whole word, not as a substring of
# "rootkit" or "underroot". A different kind of position-matching from
# ^/$, anchored to word edges rather than line edges.
```

### 1.3 Character classes

```bash
grep '[0-9]' file.txt          # any single digit
grep '[^0-9]' file.txt         # any single character that is NOT a digit —
                                # ^ inside [] means negation, a completely
                                # different meaning from ^ as a line anchor
grep '[[:digit:]]' file.txt    # POSIX named class, equivalent to [0-9]
grep '[[:alpha:]]' file.txt    # any letter
grep '[[:space:]]' file.txt    # any whitespace character
```

**POSIX named classes (`[[:digit:]]`, `[[:alpha:]]`, etc.) exist mainly for
locale and encoding correctness**, not just as longer synonyms —
`[a-z]` in some locales doesn't reliably mean "all lowercase letters" the
way `[[:lower:]]` does, because character ordering/ranges can behave
differently across locales in ways `[a-z]` was never designed to account
for. For portable scripts, POSIX classes are the more correct choice even
though `[0-9]`/`[a-z]` "usually work" in practice.

```bash
grep '[aeiou]' file.txt        # any single vowel
grep '[^aeiou]' file.txt       # any single non-vowel character
```

### 1.4 Quantifiers — open-ended intervals, and the greedy-by-default trap

```bash
grep -E 'ab+c' file.txt        # one or more 'b's between a and c
grep -E 'ab*c' file.txt        # zero or more 'b's
grep -E 'ab?c' file.txt        # zero or one 'b'
grep -E 'ab{2,4}c' file.txt    # between 2 and 4 'b's
grep -E 'ab{3}c' file.txt      # exactly 3 'b's
grep -E 'ab{2,}c' file.txt     # at least 2 'b's, no upper bound
```

`{n,}` (omitting the second number, but keeping the comma) means "at least
n" — the comma is required even with nothing after it; drop it entirely
(`{n}`, already shown above) and it instead means exactly n. There's no
equivalent "at most m" form with an omitted lower bound in standard
BRE/ERE — an upper-bound-only interval isn't part of the POSIX interval
syntax.

**Regex quantifiers are greedy by default — they match as much as possible,
not as little as possible.** This is easy to misjudge when a pattern spans
more than the specific characters you were thinking about:

```bash
echo '<a><b>' | grep -oE '<.+>'
# Matches the ENTIRE string '<a><b>', not just '<a>' — .+  greedily
# consumes as much as it can while still allowing the overall pattern to
# match, including the '>' in the middle that a naive reading might expect
# to stop it.
```

This is one of the most common sources of "my regex matched way more than I
expected" — the fix (non-greedy matching, or a more specific character
class instead of `.`) depends on the tool; plain POSIX BRE/ERE (which `grep`
uses) doesn't support non-greedy quantifiers at all — that's a PCRE
(`grep -P`) feature, not something available in standard BRE/ERE. PCRE mode
also brings other conveniences BRE/ERE simply don't have — `\d` as digit
shorthand, lookahead/lookbehind assertions — but it's GNU grep-specific,
not something every `grep` implementation supports.

### 1.5 Backreferences — matching repeated content

A backreference lets a pattern refer back to something an earlier group in
the *same* pattern actually matched — not a fixed value, but whatever that
group captured on this specific match:

```bash
grep -E '(\w+) \1' file.txt
# Matches a repeated word: "the the", "is is" — \1 refers to whatever the
# first group captured, and requires that exact same text to appear again.
```

`\w` (any word character) is itself worth flagging the same way as BRE's
`\|` in §1.1: it's a widely supported GNU extension, not something strict
POSIX BRE/ERE defines. It works reliably in GNU `grep`/`sed`/`gawk` — the
tools this entire series uses — but isn't guaranteed on a strictly
POSIX-only regex implementation.

```bash
echo "2026-2026" | grep -E '([0-9]{4})-\1'
# Matches only if the second number is identical to the first — not "any
# 4-digit number", but specifically the same one.
echo "2026-2027" | grep -E '([0-9]{4})-\1'
# No match — 2027 != 2026, even though both are valid 4-digit numbers on
# their own.
```

**Backreferences match previously-matched text, not a repeated pattern
structure.** `(\w+) \1` doesn't mean "two words" — it means "a word,
followed by that exact same word again," which is a meaningfully narrower
and different thing.

### 1.6 Which flavor does each tool actually use?

| Tool | Default flavor | How to get the other |
|---|---|---|
| `grep` | BRE | `grep -E` for ERE, `grep -P` for PCRE (Perl-compatible, GNU grep only) |
| `sed` | BRE | `sed -E` (GNU and BSD sed) or `sed -r` (GNU sed only, not BSD) for ERE |
| `awk` | ERE | No BRE mode — `awk` regex is ERE-like by default |

**This table is the single most practically important thing in this demo.**
A pattern written and tested with plain `grep` (BRE) that gets pasted
unmodified into `awk` (ERE) can behave differently — or vice versa — purely
because of which flavor each tool assumes by default, with no error message
indicating the mismatch, just a pattern that silently doesn't match what you
expected. `sed -E` vs. `sed -r` is its own smaller version of this same
"which spelling actually works here" trap: GNU `sed` accepts both as
synonyms, but only `-E` is portable to BSD/macOS `sed` as well — `-r` alone
is a GNU-only spelling of the same flag.

## Part 2 — Hands-On Lab

### Commands used in this demo

| Command | Flag meaning | Purpose |
|---|---|---|
| `grep 'PATTERN'` | — | BRE match |
| `grep -E 'PATTERN'` | `-E`: extended | ERE match |
| `grep -w 'WORD'` | `-w`: word boundary | Match whole words only |
| `grep -o 'PATTERN'` | `-o`: only matching | Print only the matched text, not the whole line |
| `grep -P 'PATTERN'` | `-P`: Perl-compatible | PCRE features (GNU grep only) |

### Task 1 — BRE vs ERE, the same pattern two ways

```bash
printf 'color\ncolour\n' > /tmp/regex-lab.txt
grep 'colou\?r' /tmp/regex-lab.txt      # BRE — backslash makes ? special
grep -E 'colou?r' /tmp/regex-lab.txt    # ERE — ? is already special
# Confirm both return the same two matches, via different escaping rules.
```

### Task 2 — Anchors: line-level and word-level

```bash
printf 'root:x:0:0::/root:/bin/bash\nrooter:x:1001::/home/rooter:/bin/bash\n' > /tmp/regex-lab2.txt
grep '^root:' /tmp/regex-lab2.txt      # matches only the first line
grep -w 'root' /tmp/regex-lab2.txt     # word-boundary — compare which
                                        # line(s) match vs. the ^root: test
```

### Task 3 — Character classes: ranges vs. POSIX classes

```bash
printf 'abc123\nXYZ789\n   \n' > /tmp/regex-lab3.txt
grep '[0-9]' /tmp/regex-lab3.txt
grep '[[:digit:]]' /tmp/regex-lab3.txt   # same result, POSIX form
grep '[[:space:]]' /tmp/regex-lab3.txt   # matches the whitespace-only line
```

### Task 4 — Quantifiers, open-ended intervals, and the greedy trap

```bash
printf 'ac\nabc\nabbc\nabbbc\nabbbbc\n' > /tmp/regex-lab4.txt
grep -E 'ab{2,}c' /tmp/regex-lab4.txt
# Confirm this matches "abbc", "abbbc", and "abbbbc" (2 or more b's), but
# not "ac" or "abc" (0 or 1 b's) — the open-ended {2,} form in action.

echo '<a><b>' | grep -oE '<.+>'
# Observe the full-string match — greedy behavior in action.
echo '<a><b>' | grep -oE '<[^>]+>'
# Compare against a more specific character class ([^>] instead of .) —
# this constrains the match to stop at the first '>', avoiding the
# greediness trap without needing non-greedy quantifier support.
```

### Task 5 — Backreferences: find repeated words

```bash
printf 'this is is a test\nthis is a a test\nthis is a fine test\n' > /tmp/regex-lab5.txt
grep -E '(\w+) \1' /tmp/regex-lab5.txt
# Confirm only the two lines with an actual repeated word match — the third
# line, with no repetition, correctly doesn't.
```

### Task 6 — Same pattern, three tools, three flavor assumptions

```bash
echo "test123value" > /tmp/regex-lab6.txt
grep -E '[0-9]+' /tmp/regex-lab6.txt          # ERE, explicit
sed -E 's/[0-9]+/NUM/' /tmp/regex-lab6.txt    # ERE, explicit via -E
awk '{gsub(/[0-9]+/, "NUM"); print}' /tmp/regex-lab6.txt   # ERE by default,
                                                             # no flag needed
# Confirm all three treat + as a quantifier without needing a backslash —
# because all three were told (or default to) ERE here. Now try grep
# without -E:
grep '[0-9]+' /tmp/regex-lab6.txt
# This treats + as a LITERAL character (BRE default) — matches lines
# containing a literal "+" character, which this file doesn't have, so it
# returns nothing. Same pattern string, different tool-default flavor,
# completely different result.
```

## What You Learned

1. ✅ Explained the difference between BRE and ERE, and why escaping is inverted between them
2. ✅ Used anchors to match position — line-level (`^`/`$`) and word-level (`-w`)
3. ✅ Used character classes, including why POSIX named classes matter for portability
4. ✅ Used quantifiers, including open-ended intervals, and demonstrated the greedy-by-default trap
5. ✅ Used backreferences to match previously-matched, not just structurally similar, content
6. ✅ Identified each tool's default regex flavor, confirmed the same pattern behaves differently across them, and recognized which regex conveniences are GNU extensions rather than POSIX-guaranteed

## Part 3 — Break-Fix Scenario

### Scenario — NovaTech Ops: two regex incidents

> **Incident report — NovaTech Ops, ticket #4618**
>
> Alert 1: "a log-parsing `grep` pattern that worked perfectly in testing
> matches nothing in production, no error." Alert 2: "an `awk` script
> ported from a working `sed` one-liner behaves differently on the same
> input."

**Incident 1 — grep pattern works in testing, matches nothing in production**

```bash
$ # Testing (developer's terminal):
$ grep -E 'error|warning' app.log | wc -l
14
$ # Production (deployed script):
$ grep 'error|warning' app.log | wc -l
0
```

Tasks:
1. Spot the exact difference between the two commands. Reference §1.1 and
   §1.6 — why does dropping one flag change the result from 14 matches to 0?
2. In BRE, what does `error|warning` actually mean, character by character,
   given that `|` is not special without escaping?
3. Fix the production script, and explain why this class of bug produces no
   error message at all — just a silently wrong result.

**Incident 2 — awk script ported from sed behaves differently**

```bash
$ sed -E 's/(\w+)\s\1/DUPLICATE/' input.txt
this is DUPLICATE a test
$ awk '{gsub(/(\w+)\s\1/, "DUPLICATE"); print}' input.txt
this is is a test
```

Tasks:
1. The `sed` version correctly collapses the repeated "is is" to
   "DUPLICATE"; the `awk` version does nothing. Reference §1.6 and §1.5 —
   what capability does `sed -E`'s ERE support here that plain POSIX ERE
   (which `awk` uses) does not?
2. Is this a bug in `awk`, or a genuine capability difference between what
   `sed -E` and `awk`'s regex engine each support?
3. What would need to change about the approach (not just the flag) to
   achieve the same "find repeated words" result in `awk`?

<details>
<summary>Answers</summary>

**Incident 1:** The production command dropped `-E`, meaning `grep` falls
back to its default flavor — BRE. In BRE, `|` is a literal character, not
alternation, unless escaped as `\|`. So `grep 'error|warning'` in BRE
literally searches for the exact string `error|warning` (the word "error",
a literal pipe character, the word "warning", all in sequence) — which
essentially never appears in a log file, hence 0 matches. The testing
command explicitly used `-E`, correctly enabling ERE where `|` means
alternation, matching either word — hence 14. Fix: add `-E` back to the
production script (or escape it as `error\|warning` to keep BRE, though
using `-E` is the clearer fix here). This class of bug produces no error
message because both BRE and ERE parse `error|warning`/`error\|warning`
as completely valid patterns — they're just valid patterns with different
*meanings*, so there's nothing for either regex engine to reject; it's not
a syntax error, it's a semantic mismatch between what was intended and what
the chosen flavor actually specifies.

**Incident 2:** This is a genuine capability difference, not a bug in
`awk`. Per §1.6, `awk`'s regex engine is ERE-like by default, and standard
POSIX ERE does not support backreferences (`\1` referring back to a
previously matched group within the same pattern) — that's a feature GNU
`sed -E` happens to support as an extension beyond strict POSIX ERE, but
it's not a capability POSIX ERE (or `awk`'s regex engine) guarantees or
generally provides. To achieve the same "find repeated words" behavior in
`awk`, the approach needs to change entirely — rather than relying on a
single backreference-based regex, this typically means splitting the line
into fields/words and comparing adjacent tokens programmatically (a loop
comparing `$i` to `$(i-1)` across fields), since `awk`'s regex engine alone
can't express "match whatever this earlier group captured."

</details>

## Interview Prep

**Q1. A regex pattern works when tested with one tool but silently fails with another, using the exact same pattern string. What's the first thing you'd check?**

Which regex flavor each tool defaults to — BRE, ERE, or PCRE. `grep` and
`sed` default to BRE (requiring backslash-escaping for `+`, `?`, `(`, `)`,
`|` to be special), while `awk` defaults to ERE (where those characters are
already special without escaping). A pattern written for one flavor can be
a syntactically valid but semantically different pattern in another,
producing a silently wrong result with no error to flag the mismatch.

---

**Q2. Why do POSIX named character classes like `[[:alpha:]]` exist when `[a-z]` "usually works" for the same purpose?**

Locale correctness. Character ordering and range behavior for constructs
like `[a-z]` aren't guaranteed to mean exactly "all lowercase letters" under
every locale — POSIX named classes are defined specifically to be
locale-correct and encoding-aware, making them the more portable and
technically correct choice for scripts that need to behave consistently
across environments, even though `[a-z]` frequently "works" in common
single-locale setups.

---

**Q3. Why can't `awk`'s default regex engine support the same backreference-based "find repeated words" pattern that `sed -E` supports?**

`awk`'s regex engine follows POSIX ERE, which does not include
backreferences as part of the standard — `sed -E`'s backreference support
is a GNU extension beyond strict POSIX ERE, not a capability POSIX ERE
guarantees universally. This isn't a bug in `awk`; it's a genuine capability
boundary that requires a different, non-regex-only approach (comparing
adjacent tokens programmatically) to achieve the same result.

---

**Q4. What does the "greedy by default" nature of quantifiers actually mean, and how would you avoid the classic `<.+>` overmatch trap in a tool without non-greedy quantifier support?**

Greedy means a quantifier consumes as much input as possible while still
letting the overall pattern succeed — `<.+>` against `<a><b>` matches the
entire string, not just `<a>`, because `.+` greedily extends across the
middle `>` and `<` as long as a final `>` still exists somewhere to satisfy
the pattern. In a POSIX BRE/ERE context without non-greedy quantifier
support (a PCRE-only feature via `grep -P`), the fix is using a more
specific character class instead of `.` — `<[^>]+>` — which structurally
can't cross a `>` boundary, achieving the same practical effect without
needing non-greedy matching at all.

---

## Key Takeaways

1. **BRE and ERE invert which characters need escaping to be special.**
   `+`, `?`, `(`, `)`, `|` are literal in BRE unless escaped, and special in
   ERE unless escaped — mixing up which flavor a pattern was written for
   produces a valid-but-wrong pattern, not an error.
2. **Anchors (`^`, `$`, `-w`) match position, not content.** `^$` matching
   an empty line only makes sense once you internalize that anchors assert
   a position rather than consuming a character.
3. **POSIX named character classes (`[[:digit:]]`, etc.) exist for locale
   correctness, not just as longer synonyms for `[0-9]`-style ranges.**
   They're the more portable choice for scripts that must behave
   consistently across environments.
4. **Quantifiers are greedy by default in BRE/ERE, with no non-greedy
   option available.** `<.+>` overmatching across multiple tags is the
   classic symptom — fixed with a more specific character class (`[^>]`
   instead of `.`), not a non-greedy flag that doesn't exist in POSIX
   regex. `{n,}` (comma required, no second number) extends this same
   interval syntax to "at least n," with no equivalent unbounded-below
   "at most m" form.
5. **Backreferences match previously-matched text, not repeated structure.**
   `(\w+) \1` means "this exact word, then that exact same word again" —
   and this capability is a GNU extension in tools like `sed -E`, not
   something standard POSIX ERE (and therefore `awk`) guarantees.
6. **Know each tool's default flavor before writing a pattern for it.**
   `grep`/`sed` default to BRE; `awk` defaults to ERE — the single most
   common source of "this pattern worked somewhere else" bugs in this
   entire topic.
7. **Some very common regex conveniences aren't strictly POSIX at all.**
   BRE's `\|` alternation and `\w`'s word-character shorthand are both
   widely supported GNU extensions, not guaranteed by the POSIX standard
   itself — reliable throughout this series' GNU-based Linux environments,
   but not something to assume on every regex implementation you might
   encounter elsewhere.

> **Demo scope:** Primary concept: regular expression syntax and matching
> logic. Supporting concepts: BRE vs ERE, anchors, character classes,
> quantifiers (including open-ended intervals), backreferences, per-tool
> default flavor differences, POSIX-standard vs. GNU-extension features.
> Estimated completion time: 35 minutes (reading + hands-on + verification).
> Checkpoints: 6 natural stopping points (after Task 1 through Task 6).

## Quick Commands Reference

### Flavor selection

| Command | Flavor |
|---|---|
| `grep 'PATTERN'` | BRE |
| `grep -E 'PATTERN'` | ERE |
| `grep -P 'PATTERN'` | PCRE (GNU grep only) |
| `sed 's/PATTERN/.../'` | BRE |
| `sed -E 's/PATTERN/.../'` | ERE (portable: GNU and BSD sed) |
| `sed -r 's/PATTERN/.../'` | ERE (GNU sed only, not BSD) |
| `awk '/PATTERN/'` | ERE (default, no flag) |

### Anchors and classes

| Syntax | Meaning |
|---|---|
| `^` / `$` | Start / end of line |
| `-w` (grep) | Whole word boundary |
| `[0-9]` / `[[:digit:]]` | Digit (range vs. POSIX class) |
| `[^...]` | Negation, inside a character class |

### Quantifiers

| Syntax (ERE) | Meaning |
|---|---|
| `*` | Zero or more |
| `+` | One or more |
| `?` | Zero or one |
| `{n,m}` | Between n and m |
| `{n}` | Exactly n |
| `{n,}` | At least n (comma required, no upper bound) |

## Appendix A — Anki Cards

**`17-regular-expressions-anki.csv`:**

```
#deck:Linux Mastery::Module 4 - Text Processing & Scripting for DevOps::Demo 17 - Regular Expressions
#separator:Comma
#columns:Front,Back,Tags
"grep 'error|warning' file.log returns 0 matches, but grep -E 'error|warning' file.log returns 14. Why?","In BRE (grep's default without -E), | is a literal character, not alternation — the pattern searches for the literal string 'error|warning'. -E enables ERE, where | means alternation (match either word).","demo17,bre,ere,common-mistake"
"What's the escaping relationship between BRE and ERE for characters like +, ?, (, ), |?","It's inverted: in BRE these are literal unless backslash-escaped (\\+, \\?, etc.) to become special; in ERE they're already special unless backslash-escaped to become literal.","demo17,bre,ere,fundamentals"
"Why does ^$  match an empty line, given that ^ and $ don't consume any characters?","Anchors match position, not content — ^ asserts 'start of line' and $ asserts 'end of line'. ^$ means those two positions are adjacent with nothing between them, which is exactly what an empty line is.","demo17,anchors,fundamentals"
"Why do POSIX named classes like [[:digit:]] exist when [0-9] usually works for the same purpose?","Locale correctness — character ordering/range behavior for constructs like [a-z] isn't guaranteed to mean exactly what you expect under every locale. POSIX named classes are defined to be locale-correct and portable.","demo17,character-classes,posix"
"grep -oE '<.+>' against '<a><b>' matches the entire string, not just '<a>'. Why?","Quantifiers are greedy by default — .+ consumes as much as possible while still letting the overall pattern match, extending across the middle '>' and '<' since a final '>' still exists to satisfy the pattern.","demo17,quantifiers,greedy,common-mistake"
"How do you fix a greedy '<.+>' overmatch without non-greedy quantifier support (which POSIX BRE/ERE doesn't have)?","Use a more specific character class instead of . — e.g. <[^>]+> — which structurally cannot cross a > boundary, achieving the same effect without needing non-greedy matching.","demo17,quantifiers,greedy,fix"
"What does the pattern (\\w+) \\1 actually match, and what's a common misreading of it?","It matches a word immediately followed by that exact same word again (a repeated word) — \\1 refers to whatever text the first group actually captured on this match, not just 'any two words' or 'a repeated structure'.","demo17,backreferences,fundamentals"
"Does awk's default regex engine support backreferences the way sed -E does?","No — awk's regex engine follows POSIX ERE, which does not include backreferences. sed -E's backreference support is a GNU extension beyond strict POSIX ERE, not a universal capability.","demo17,backreferences,awk,sed,comparison"
"What are the default regex flavors for grep, sed, and awk respectively?","grep: BRE (use -E for ERE, -P for PCRE). sed: BRE (use -E or -r for ERE). awk: ERE by default, with no BRE mode.","demo17,tool-defaults,fundamentals"
"A pattern is tested and works correctly with one tool, then silently produces a different result when pasted unmodified into another tool. What's the most likely cause?","A mismatch between the regex flavor the pattern was written for and the flavor the second tool defaults to (BRE vs ERE) — both flavors treat the same pattern string as syntactically valid, just with different meanings, so there's no error to flag the mismatch.","demo17,tool-defaults,troubleshooting"
```

## Appendix B — Quiz

**`17-regular-expressions-quiz.md`:**

````markdown
# Quiz — Demo 17: Regular Expressions

> One correct answer per question unless stated otherwise.
> Target: 80% or above before moving to Demo 18.

---

**Q1.** `grep -w 'root' /etc/passwd` is run against a file containing both a `root` line and a `rooter` line. What matches?

- A) Both lines, since `rooter` contains `root` as a substring
- B) Only the `root` line — `-w` requires `root` to appear as a whole word, not as part of a longer word like `rooter`
- C) Neither line, since `-w` requires an exact full-line match
- D) Only the `rooter` line

<details>
<summary>Answer</summary>

**B** — `-w` anchors the match to word boundaries, so `root` matches only when it isn't part of a larger word — `rooter` doesn't qualify since `root` there is immediately followed by more word characters (`er`), not a word boundary.

Trap: A ignores what `-w` specifically restricts; C confuses word-boundary matching with whole-line matching (that would be `^root$`); D gets the result backwards.

</details>

---

**Q2.** What does adding `-o` to `grep -oE '[0-9]+'` change about the output, compared to `grep -E '[0-9]+'` alone?

- A) `-o` makes the match case-insensitive
- B) `-o` prints only the matched text itself, not the entire line it was found on
- C) `-o` limits the search to the first match per file, ignoring the rest
- D) `-o` has no effect when combined with `-E`

<details>
<summary>Answer</summary>

**B** — Without `-o`, `grep` prints the whole line containing a match; with `-o`, it prints just the matched substring itself, one per line. This is why the demo's greedy-quantifier examples use `-oE` — to see exactly what the pattern matched, not the surrounding line.

Trap: A, C, and D all invent behavior `-o` doesn't have — it changes output granularity, not case sensitivity, match count, or flag compatibility.

</details>

---

**Q3.** `grep -E 'ab{2,}c'` is run against a file containing `ac`, `abc`, `abbc`, and `abbbc`. Which lines match?

- A) Only `abc`
- B) `abbc` and `abbbc` — `{2,}` means "at least 2," so `ac` (0 b's) and `abc` (1 b) don't qualify
- C) All four lines
- D) None — `{2,}` is invalid syntax without a specified upper bound

<details>
<summary>Answer</summary>

**B** — `{2,}` requires at least 2 repetitions with no upper limit, so only the lines with 2 or more `b`s between `a` and `c` match. The comma with no following number is valid, required syntax for "at least n," not an error.

Trap: A misreads the minimum as an exact count; C ignores the lower-bound requirement entirely; D wrongly claims the open-ended form is invalid.

</details>

---

**Q4.** Is `\w` (matching any word character) part of the strict POSIX ERE/BRE standard?

- A) Yes, `\w` is defined identically in both POSIX BRE and ERE
- B) No — `\w` is a widely supported GNU extension, reliable in GNU `grep`/`sed`/`gawk` (used throughout this series) but not guaranteed by the POSIX standard itself
- C) `\w` is POSIX-standard only in ERE, not BRE
- D) `\w` doesn't work in any Linux regex tool without `-P`

<details>
<summary>Answer</summary>

**B** — `\w` is a common, reliable GNU extension across the GNU toolchain this series uses, but it isn't part of the POSIX BRE/ERE specification itself — the same category of convenience as BRE's `\|` alternation.

Trap: A and C both invent a POSIX-standard status `\w` doesn't have; D is factually wrong — `\w` works in plain `grep -E`/`gawk` without needing `-P`.

</details>

---

**Q5.** In ERE (not BRE), what does the escaped form `\|` actually mean?

- A) Alternation (OR) — identical to plain `|`
- B) A literal pipe character, since `|` is already special in ERE and backslash-escaping it makes it literal instead
- C) `\|` is invalid syntax in ERE
- D) A line-continuation marker

<details>
<summary>Answer</summary>

**B** — In ERE, `|` is already the special alternation operator, so escaping it with a backslash flips it back to a literal character — the opposite of BRE, where `|` is literal by default and `\|` is what makes it special (as a GNU extension).

Trap: A wrongly imports BRE's `\|` meaning into ERE; C and D both invent incorrect behavior for a syntactically valid escape.

</details>

---

**Q6.** What's the relationship between `sed -E` and `sed -r` for enabling ERE mode?

- A) They are entirely unrelated flags with different purposes
- B) GNU `sed` accepts both as synonyms for the same ERE-enabling effect, but only `-E` is also portable to BSD/macOS `sed` — `-r` is a GNU-only spelling
- C) `-r` enables ERE; `-E` enables a completely different, incompatible regex mode
- D) `-E` only works in `sed` scripts, `-r` only works on the command line

<details>
<summary>Answer</summary>

**B** — Both flags do the same thing on GNU `sed`, but `-E` is the more portable choice since BSD/macOS `sed` recognizes `-E` but not `-r` — a smaller-scale version of the same "which spelling actually works here" caution that applies across regex tooling generally.

Trap: A and C both wrongly claim the flags do different things; D invents a context restriction that doesn't exist for either flag.

</details>

---

**Q7.** Beyond enabling non-greedy quantifiers, what else does `grep -P` (PCRE mode) provide that standard BRE/ERE don't?

- A) Nothing else — non-greedy quantifiers are PCRE's only additional feature
- B) Additional conveniences like `\d` digit shorthand and lookahead/lookbehind assertions — but these are GNU grep-specific, not guaranteed on every `grep` implementation
- C) PCRE mode is slower but otherwise functionally identical to ERE
- D) `-P` is required to use character classes like `[0-9]` at all

<details>
<summary>Answer</summary>

**B** — PCRE mode brings a broader feature set than plain BRE/ERE — `\d`, lookahead/lookbehind, and non-greedy quantifiers among them — but it's a GNU grep extension, not something to assume is available on every `grep` you might encounter.

Trap: A understates PCRE's actual feature set; C wrongly claims functional identity with ERE; D is false — standard character classes work fine without `-P`.

</details>

---

**Q8.** Why does the demo emphasize distinguishing "POSIX-standard" from "GNU extension" for features like BRE's `\|` and `\w`, given that both work fine on Ubuntu and Rocky9?

- A) It's a purely academic distinction with no practical consequence on any real system
- B) Because a script written assuming these extensions may behave differently or fail on a strictly POSIX-only or non-GNU regex implementation, even though it works reliably on the GNU-based systems this series targets
- C) Because `\|` and `\w` are actually deprecated and will stop working in future GNU tool versions
- D) Because Rocky9 uses a different regex engine than Ubuntu, and only one of them supports these extensions

<details>
<summary>Answer</summary>

**B** — The distinction matters for portability: code relying on GNU-specific regex extensions can break on other systems (older Unix variants, BSD-derived tools, some minimal/embedded environments) even though it works perfectly on the GNU-based Linux distros used throughout this series.

Trap: A dismisses a real portability concern; C invents a deprecation that isn't happening; D is factually wrong — both Ubuntu and Rocky9 use the same GNU tools and support these extensions identically.

</details>

---

Score guide:

| Score | Action |
|---|---|
| 7/8 or 8/8 | Import Anki cards, move to Demo 18 |
| 6/8 | Review the wrong answers, then proceed |
| 5/8 | Re-read the relevant sections, retry those questions |
| Below 5/8 | Re-read the full demo and redo the walkthrough before proceeding |
````