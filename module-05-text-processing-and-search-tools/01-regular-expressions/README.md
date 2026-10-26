# Demo M05-01 — Regular Expressions

## Overview

This demo is deliberately standalone, taught before grep, sed, or awk —
because all three depend on regular expressions, and each has its own
quirks about which regex flavor it speaks by default. Getting comfortable
with the underlying pattern language first means the next three demos can
focus on what each tool *does* with a match, not re-explain regex syntax
three times. By the end, you'll read and write patterns confidently and
know exactly which flavor a given tool expects.

## Demo Objectives

1. Explain what a regular expression actually is, and that matching is
   unanchored (substring) by default in every tool this series uses
2. Explain the difference between Basic and Extended Regular Expressions
   (BRE/ERE), including what grouping actually does and alternation's
   surprisingly low precedence
3. Use anchors to match position, not just content
4. Use character classes and their POSIX named forms correctly, and
   escape metacharacters to match them literally when that's what's needed
5. Use quantifiers to control repetition, including open-ended intervals
   and the greedy-by-default trap
6. Use backreferences to match repeated content within a single pattern,
   and know precisely which flavor(s) actually support them natively
7. Identify which regex flavor `grep`, `sed`, and `awk` each use by default,
   and recognize which conveniences are POSIX-standard versus GNU extensions

## Directory Structure

```
01-regular-expressions/
├── README.md                            # this file
├── m05-01-regular-expressions-anki.csv  # Anki flashcard deck
└── m05-01-regular-expressions-quiz.md   # standalone quiz
```

## Environments

| Task | Ubuntu 24.04 | Rocky Linux 9.8 | EC2 Amazon Linux |
|---|---|---|---|
| All lab tasks | ✅ | ✅ | Not required |

## Recall Check — 07-remote-storage-troubleshooting

Answer these from memory before continuing — no looking back at Demo M04-07.

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

### 1.1 What a regular expression actually is — and what it matches by default

A regular expression is a **pattern language for describing sets of
strings** — not a specific string itself, but a rule that some strings
satisfy and others don't. `cat` as a plain string means exactly those
three characters; `cat` as a regex pattern means "any string containing
these three characters in this order, anywhere within it," which is a
meaningfully different, broader thing.

```bash
printf 'cat\ncategory\nconcatenate\ndog\n' > /tmp/regex-lab0.txt
grep 'cat' /tmp/regex-lab0.txt
```
```
cat
category
concatenate
```
> Live-verified. **This is the critical default to internalize before
> anything else in this demo:** `grep`, `sed`, and `awk` all search for a
> pattern *anywhere within* a line by default — substring matching, not
> whole-line matching. `cat` matched not just the line that's exactly
> "cat," but also "category" and "concatenate," because the three letters
> appear somewhere in each. §1.3's anchors (`^`, `$`) exist specifically
> to *override* this default — they only make sense once you know what
> they're overriding.

**The `.` metacharacter** — any single character, except (in most modes)
a newline:

```bash
printf 'cat\ncot\nc t\nc9t\ncoot\n' > /tmp/regex-lab0b.txt
grep 'c.t' /tmp/regex-lab0b.txt
```
```
cat
cot
c t
c9t
```
> Live-verified. `.` matched a letter, a space, and a digit equally —
> "any single character" really does mean any character, not "any
> letter." Note `coot` does **not** match: `c.t` is exactly three
> characters wide (c, any-one-character, t), and `coot` has two
> characters between `c` and `t`, not one.

**Escaping a metacharacter to match it literally** — the flip side of
`.` meaning "any character" is that matching an *actual* dot requires
escaping it:

```bash
printf '192.168.1.1\n192x168x1x1\n' > /tmp/regex-lab0c.txt
echo "-- unescaped dot: matches BOTH, since . means any character --"
grep '192.168' /tmp/regex-lab0c.txt
echo "-- escaped dot: matches only the real IP --"
grep '192\.168' /tmp/regex-lab0c.txt
```
```
-- unescaped dot: matches BOTH, since . means any character --
192.168.1.1
192x168x1x1
-- escaped dot: matches only the real IP --
192.168.1.1
```
> Live-verified. This is a genuinely common real-world bug — an
> unescaped `.` in an IP address or filename pattern silently matches
> more than intended, with no error, because `x` (or any character) is
> just as valid a match for `.` as an actual dot is. The same escaping
> pattern applies to every other metacharacter when you want it literal:
> `\*` for a literal asterisk, `\[` for a literal bracket, `\\` for a
> literal backslash. Getting into the habit of asking "do I mean this
> character literally, or as a wildcard?" for every `.`, `*`, and `[` in
> a pattern is what prevents this class of silent overmatch.

### 1.2 BRE vs ERE — two different pattern languages, not two "modes" of one

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

**What grouping actually does** — the table above lists it as a syntax
row, but two genuinely different things happen when you group:

```bash
printf 'ab\nabb\nabab\nababab\na\n' > /tmp/regex-lab1.txt
echo "-- ab+ : one 'a', then one-or-more 'b's --"
grep -E 'ab+' /tmp/regex-lab1.txt
echo "-- (ab)+ : the whole two-character unit 'ab', repeated --"
grep -E '^(ab)+$' /tmp/regex-lab1.txt
```
```
-- ab+ : one 'a', then one-or-more 'b's --
ab
abb
abab
ababab
-- (ab)+ : the whole two-character unit 'ab', repeated --
ab
abab
ababab
```
> Live-verified. `abb` matches `ab+` (one `a`, two `b`s) but *not*
> `^(ab)+$` (it isn't whole repetitions of the two-character unit "ab").
> Grouping does two things at once: it **scopes a quantifier to more than
> one character** (`+` after `(ab)` repeats the whole pair, not just the
> last character before it), and — as §1.6 covers — it **captures** the
> matched text so a later backreference can refer back to it. Both
> effects come from the same parentheses; which one matters depends on
> whether anything downstream actually uses the capture.

**Alternation's precedence is lower than you'd probably guess** — a
frequent, genuine gotcha:

```bash
printf 'cats are great\nI have a dog\nbirds fly\n' > /tmp/regex-lab2.txt
echo "-- ^cat|dog\$ : does this mean lines STARTING with cat OR dog, or something else? --"
grep -E '^cat|dog$' /tmp/regex-lab2.txt
echo "-- correctly grouped: ^(cat|dog) --"
grep -E '^(cat|dog)' /tmp/regex-lab2.txt
```
```
-- ^cat|dog$ : does this mean lines STARTING with cat OR dog, or something else? --
cats are great
I have a dog
-- correctly grouped: ^(cat|dog) --
cats are great
```
> Live-verified. `^cat|dog$` parses as `(^cat)|(dog$)` — "starts with
> cat, OR ends with dog" — **not** "starts with cat or dog," which is
> probably what was intended. `|` binds more loosely than concatenation
> and anchors, so an anchor written on one side of `|` applies only to
> that side, not to the whole alternation. "I have a dog" matched
> because it ends in "dog," even though it doesn't start with either
> word — exactly the surprising case that makes this worth knowing
> explicitly rather than discovering it in production. When you mean
> "anchor applies to the whole alternation," group it explicitly:
> `^(cat|dog)`.

### 1.3 Anchors — matching position, not content

```bash
grep '^root' /etc/passwd        # ^ : start of line
grep 'bash$' /etc/passwd        # $ : end of line
grep '^root:.*bash$' /etc/passwd  # combine: whole-line shape
```

**Anchors match a *position*, not a character** — `^` doesn't consume any
input, it asserts "we are at the start of the line right now," overriding
§1.1's unanchored-by-default behavior for whatever side of the pattern
it's on. This distinction matters once you start combining anchors with
quantifiers: `^$` matches an empty line exactly (start immediately
followed by end), which is a genuinely useful and common pattern for
"find blank lines," not a construct that would make sense if `^`/`$` were
ordinary characters.

```bash
grep -w 'root' /etc/passwd
# -w: word boundary — matches "root" as a whole word, not as a substring of
# "rootkit" or "underroot". A different kind of position-matching from
# ^/$, anchored to word edges rather than line edges.
```

### 1.4 Character classes

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

**A character class is also where `.` stops being special, worth noting
given §1.1's escaping discussion:** inside `[...]`, a literal `.` doesn't
need escaping at all — `[0-9.]` means "a digit or a literal dot," with no
backslash required, since inside brackets most metacharacters (including
`.` and `*`) simply lose their special meaning and become literal by
default. This is the opposite convention from outside a character class,
and it's a small but real source of confusion if you carry the "always
escape a literal dot" habit into bracket expressions where it isn't needed.

**Case-insensitive matching**, while not a character-class feature itself,
solves a related class of "match more variants of this" problem without
writing `[Rr][Oo][Oo][Tt]`-style patterns by hand:

```bash
grep -i 'root' /etc/passwd
# -i: case-insensitive — matches "root", "Root", "ROOT", etc. Supported
# by grep, sed (as a trailing 'I' flag on the s/// command), and awk
# (via IGNORECASE in gawk specifically, not universal awk).
```

### 1.5 Quantifiers — open-ended intervals, and the greedy-by-default trap

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

### 1.6 Backreferences — matching repeated content

A backreference lets a pattern refer back to something an earlier group in
the *same* pattern actually matched — not a fixed value, but whatever that
group captured on this specific match:

```bash
grep -E '(\w+) \1' file.txt
# Matches a repeated word: "the the", "is is" — \1 refers to whatever the
# first group captured, and requires that exact same text to appear again.
```

**A precision worth getting exactly right, since it's easy to state
backwards:** backreferences are actually **native to POSIX BRE** — they're
part of the standard there, not an extension. What's *not* standard is
their availability **in ERE**: POSIX ERE's grammar doesn't define
backreferences at all. GNU tools supporting `\1` in ERE mode (as the
`grep -E` example above does) are extending ERE beyond strict POSIX — a
second, separate GNU extension stacked on top of ERE, distinct from `\w`
also being a GNU extension in the same pattern. The BRE form of the same
pattern is standard-compliant in a way the ERE form isn't:

```bash
grep '\(\w\+\) \1' file.txt
# The BRE spelling of the same pattern — \( \) for grouping, \+ for
# one-or-more (both requiring backslash-escaping in BRE, per §1.2), and
# \1 for the backreference. This backreference usage IS strict POSIX BRE;
# the earlier ERE version's \1 support is the GNU-extension part.
```

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

### 1.7 Which flavor does each tool actually use?

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
| `grep -i 'PATTERN'` | `-i`: ignore case | Case-insensitive match |
| `grep -o 'PATTERN'` | `-o`: only matching | Print only the matched text, not the whole line |
| `grep -P 'PATTERN'` | `-P`: Perl-compatible | PCRE features (GNU grep only) |

### Task 0 — Default matching, the dot, and literal escaping

```bash
printf 'cat\ncategory\nconcatenate\ndog\n' > /tmp/regex-lab.txt
grep 'cat' /tmp/regex-lab.txt
# Confirm all three cat-containing lines match — substring, not whole-line

printf 'cat\ncot\nc t\nc9t\ncoot\n' > /tmp/regex-lab0b.txt
grep 'c.t' /tmp/regex-lab0b.txt
# Confirm . matches a letter, a space, AND a digit — but not "coot"
# (two characters between c and t, not one)

printf '192.168.1.1\n192x168x1x1\n' > /tmp/regex-lab0c.txt
grep '192.168' /tmp/regex-lab0c.txt      # matches BOTH lines — bug
grep '192\.168' /tmp/regex-lab0c.txt     # matches only the real IP — fixed
```

### Task 1 — BRE vs ERE, the same pattern two ways

```bash
printf 'color\ncolour\n' > /tmp/regex-lab1.txt
grep 'colou\?r' /tmp/regex-lab1.txt      # BRE — backslash makes ? special
grep -E 'colou?r' /tmp/regex-lab1.txt    # ERE — ? is already special
# Confirm both return the same two matches, via different escaping rules.

printf 'ab\nabb\nabab\nababab\na\n' > /tmp/regex-lab1b.txt
grep -E 'ab+' /tmp/regex-lab1b.txt
grep -E '^(ab)+$' /tmp/regex-lab1b.txt
# Confirm these return DIFFERENT results — ab+ matches "abb" (grouping
# not scoping the +), (ab)+ doesn't (it requires whole "ab" repetitions)

printf 'cats are great\nI have a dog\nbirds fly\n' > /tmp/regex-lab1c.txt
grep -E '^cat|dog$' /tmp/regex-lab1c.txt
grep -E '^(cat|dog)' /tmp/regex-lab1c.txt
# Confirm the first matches "I have a dog" (surprising, via the dog$
# branch) while the properly-grouped second version does not
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

printf 'price: 9.99\nprice: 9x99\n' > /tmp/regex-lab3b.txt
grep '[0-9.]' /tmp/regex-lab3b.txt
# Confirm . inside [...] needs no escaping — both lines match, since [0-9.]
# means "a digit or a literal dot," unlike outside a bracket expression
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

### Task 5 — Backreferences: BRE and ERE forms, find repeated words

```bash
printf 'this is is a test\nthis is a a test\nthis is a fine test\n' > /tmp/regex-lab5.txt
grep -E '(\w+) \1' /tmp/regex-lab5.txt          # ERE form — GNU extension
grep '\(\w\+\) \1' /tmp/regex-lab5.txt          # BRE form — POSIX-standard
# Confirm both produce identical results — only the escaping differs,
# but the BRE version's backreference usage is the POSIX-compliant one
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

1. ✅ Explained what a regular expression is, and confirmed matching is
   unanchored (substring) by default before anchors override it
2. ✅ Explained BRE vs ERE, including what grouping actually does
   (capturing and quantifier-scoping) and alternation's low precedence
3. ✅ Used anchors to match position — line-level (`^`/`$`) and word-level (`-w`)
4. ✅ Used character classes, escaped metacharacters to match them
   literally, and confirmed `.` needs no escaping inside `[...]`
5. ✅ Used quantifiers, including open-ended intervals, and demonstrated
   the greedy-by-default trap
6. ✅ Used backreferences in both their POSIX-standard BRE form and their
   GNU-extension ERE form, and can state precisely which is which
7. ✅ Identified each tool's default regex flavor, confirmed the same
   pattern behaves differently across them, and recognized which regex
   conveniences are GNU extensions rather than POSIX-guaranteed

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
1. Spot the exact difference between the two commands. Reference §1.2 and
   §1.7 — why does dropping one flag change the result from 14 matches to 0?
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
   "DUPLICATE"; the `awk` version does nothing. Reference §1.7 and §1.6 —
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
`awk`. Per §1.7, `awk`'s regex engine is ERE-like by default, and standard
POSIX ERE does not support backreferences (`\1` referring back to a
previously matched group within the same pattern) — that's a feature GNU
`sed -E` happens to support as an extension beyond strict POSIX ERE (per
§1.6, backreferences are actually native to POSIX *BRE*, just not ERE),
but it's not a capability POSIX ERE (or `awk`'s regex engine) guarantees
or generally provides. To achieve the same "find repeated words" behavior
in `awk`, the approach needs to change entirely — rather than relying on a
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
is a GNU extension beyond strict POSIX ERE (backreferences are actually
native to POSIX BRE, just not ERE), not a capability POSIX ERE guarantees
universally. This isn't a bug in `awk`; it's a genuine capability boundary
that requires a different, non-regex-only approach (comparing adjacent
tokens programmatically) to achieve the same result.

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

**Q5. A pattern `^cat|dog$` is meant to match lines starting with either "cat" or "dog," but it also matches a line that only ends with "dog" partway through the text. Why?**

Alternation (`|`) has lower precedence than concatenation and anchors, so
each side of `|` is evaluated independently: `^cat|dog$` parses as
`(^cat)|(dog$)` — "starts with cat, OR ends with dog" — not "starts with
cat or dog" as the visual layout might suggest. A line like "I have a dog"
matches the second branch (`dog$`) even though it doesn't start with
either word. The fix is explicit grouping: `^(cat|dog)` applies the anchor
to the whole alternation, matching only lines that actually start with one
of the two words.

---

## Key Takeaways

1. **A regex describes a set of strings, and matching is unanchored (substring) by default in `grep`/`sed`/`awk`.** `grep 'cat'` matches "category" and "concatenate," not just an exact "cat" line — anchors exist specifically to override this default, which is why they only make sense once this baseline is understood.
2. **`.` matches any single character, and matching a literal dot (or any other metacharacter) requires escaping it — except inside a character class, where most metacharacters are already literal by default.** `192.168` unescaped matches "192x168" just as validly as the real IP; `[0-9.]` needs no backslash for its dot at all — two different, easy-to-conflate escaping rules.
3. **BRE and ERE invert which characters need escaping to be special.** `+`, `?`, `(`, `)`, `|` are literal in BRE unless escaped, and special in ERE unless escaped — mixing up which flavor a pattern was written for produces a valid-but-wrong pattern, not an error.
4. **Grouping does two things: it scopes a quantifier to more than one character, and it captures the matched text for backreferences.** `(ab)+` (repeats the whole two-character unit) and `ab+` (repeats only the `b`) are genuinely different patterns, not stylistic variants of each other.
5. **Alternation (`|`) has surprisingly low precedence — an anchor on one side of `|` doesn't automatically apply to the other side.** `^cat|dog$` means "(starts with cat) OR (ends with dog)," not "starts with cat or dog" — explicit grouping (`^(cat|dog)`) is required to get the intended meaning.
6. **Anchors (`^`, `$`, `-w`) match position, not content.** `^$` matching an empty line only makes sense once you internalize that anchors assert a position rather than consuming a character.
7. **Quantifiers are greedy by default in BRE/ERE, with no non-greedy option available.** `<.+>` overmatching across multiple tags is the classic symptom — fixed with a more specific character class (`[^>]` instead of `.`), not a non-greedy flag that doesn't exist in POSIX regex.
8. **Backreferences match previously-matched text, not repeated structure, and are POSIX-standard in BRE but a GNU extension in ERE.** `(\w+) \1` means "this exact word, then that exact same word again" — the BRE spelling (`\(\w\+\) \1`) is the strictly standard-compliant version; the ERE spelling works reliably in GNU tools but isn't guaranteed by POSIX ERE itself.
9. **Know each tool's default flavor before writing a pattern for it.** `grep`/`sed` default to BRE; `awk` defaults to ERE — the single most common source of "this pattern worked somewhere else" bugs in this entire topic.

> **Demo scope:** Primary concept: regular expression syntax and matching
> logic, including the foundational default-matching behavior and
> metacharacter escaping that everything else builds on.
> Supporting concepts: BRE vs ERE, grouping semantics, alternation
> precedence, anchors, character classes, quantifiers (including
> open-ended intervals), backreferences (BRE-native vs. ERE-extension),
> per-tool default flavor differences, POSIX-standard vs. GNU-extension
> features.
> Estimated completion time: 42 minutes (reading + hands-on + verification).
> Checkpoints: 7 natural stopping points (after Task 0 through Task 6).

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

### Fundamentals, anchors, and classes

| Syntax | Meaning |
|---|---|
| `.` | Any single character (escape as `\.` to match a literal dot) |
| `^` / `$` | Start / end of line |
| `-w` (grep) | Whole word boundary |
| `-i` (grep) | Case-insensitive match |
| `[0-9]` / `[[:digit:]]` | Digit (range vs. POSIX class) |
| `[^...]` | Negation, inside a character class |
| `[0-9.]` | Literal dot inside a class needs no escaping |

### Quantifiers and grouping

| Syntax (ERE) | Meaning |
|---|---|
| `*` | Zero or more |
| `+` | One or more |
| `?` | Zero or one |
| `{n,m}` | Between n and m |
| `{n}` | Exactly n |
| `{n,}` | At least n (comma required, no upper bound) |
| `(ab)+` vs `ab+` | Groups scope a quantifier to the whole group, not just the last character |

## Appendix A — Anki Cards

**`m05-01-regular-expressions-anki.csv`:**

```
#deck:Linux Mastery::Module 5 - Text Processing & Search Tools::Demo M05-01 - Regular Expressions
#separator:Comma
#columns:Front,Back,Tags
"grep 'cat' file.txt is run. Does it match ONLY lines that are exactly 'cat'?","No — matching is unanchored (substring) by default. It matches any line CONTAINING 'cat' anywhere, including 'category' and 'concatenate'. Anchors (^, $) exist specifically to override this default.","demo-m05-01,fundamentals,default-matching"
"What does the . metacharacter match, and what does matching a LITERAL dot require?",". matches any single character (letter, digit, space, etc.), not specifically a dot. Matching an actual literal dot requires escaping it as \\. — unescaped, 192.168 would also match '192x168'.","demo-m05-01,fundamentals,escaping"
"Does a literal dot need escaping INSIDE a character class like [0-9.]?","No — inside [...], most metacharacters including . are already literal by default, no backslash needed. This is the opposite convention from outside a character class, where . means 'any character' unless escaped.","demo-m05-01,character-classes,escaping"
"grep 'error|warning' file.log returns 0 matches, but grep -E 'error|warning' file.log returns 14. Why?","In BRE (grep's default without -E), | is a literal character, not alternation — the pattern searches for the literal string 'error|warning'. -E enables ERE, where | means alternation (match either word).","demo-m05-01,bre,ere,common-mistake"
"What's the escaping relationship between BRE and ERE for characters like +, ?, (, ), |?","It's inverted: in BRE these are literal unless backslash-escaped (\\+, \\?, etc.) to become special; in ERE they're already special unless backslash-escaped to become literal.","demo-m05-01,bre,ere,fundamentals"
"What TWO things does grouping with parentheses actually do in a regex?","It scopes a quantifier to the whole group rather than just the preceding character (e.g. (ab)+ repeats the pair 'ab', not just 'b'), AND it captures the matched text so a later backreference can refer back to it.","demo-m05-01,grouping,fundamentals"
"^cat|dog$ is intended to match lines starting with 'cat' or 'dog'. Does it actually do that?","No — alternation has low precedence, so this parses as (^cat)|(dog$): 'starts with cat OR ends with dog.' A line like 'I have a dog' matches via the dog$ branch even though it doesn't start with either word. Fix: ^(cat|dog).","demo-m05-01,alternation,precedence,common-mistake"
"Why does ^$  match an empty line, given that ^ and $ don't consume any characters?","Anchors match position, not content — ^ asserts 'start of line' and $ asserts 'end of line'. ^$ means those two positions are adjacent with nothing between them, which is exactly what an empty line is.","demo-m05-01,anchors,fundamentals"
"Why do POSIX named classes like [[:digit:]] exist when [0-9] usually works for the same purpose?","Locale correctness — character ordering/range behavior for constructs like [a-z] isn't guaranteed to mean exactly what you expect under every locale. POSIX named classes are defined to be locale-correct and portable.","demo-m05-01,character-classes,posix"
"grep -oE '<.+>' against '<a><b>' matches the entire string, not just '<a>'. Why?","Quantifiers are greedy by default — .+ consumes as much as possible while still letting the overall pattern match, extending across the middle '>' and '<' since a final '>' still exists to satisfy the pattern.","demo-m05-01,quantifiers,greedy,common-mistake"
"How do you fix a greedy '<.+>' overmatch without non-greedy quantifier support (which POSIX BRE/ERE doesn't have)?","Use a more specific character class instead of . — e.g. <[^>]+> — which structurally cannot cross a > boundary, achieving the same effect without needing non-greedy matching.","demo-m05-01,quantifiers,greedy,fix"
"What does the pattern (\\w+) \\1 actually match, and what's a common misreading of it?","It matches a word immediately followed by that exact same word again (a repeated word) — \\1 refers to whatever text the first group actually captured on this match, not just 'any two words' or 'a repeated structure'.","demo-m05-01,backreferences,fundamentals"
"Are backreferences (\\1) part of standard POSIX BRE? Part of standard POSIX ERE?","Backreferences ARE native to standard POSIX BRE. They are NOT part of standard POSIX ERE at all — GNU tools supporting \\1 in ERE mode (like sed -E) are extending ERE beyond POSIX, a separate extension from \\w also being non-POSIX.","demo-m05-01,backreferences,posix,precision"
"Does awk's default regex engine support backreferences the way sed -E does?","No — awk's regex engine follows POSIX ERE, which does not include backreferences. sed -E's backreference support is a GNU extension beyond strict POSIX ERE, not a universal capability.","demo-m05-01,backreferences,awk,sed,comparison"
"What are the default regex flavors for grep, sed, and awk respectively?","grep: BRE (use -E for ERE, -P for PCRE). sed: BRE (use -E or -r for ERE). awk: ERE by default, with no BRE mode.","demo-m05-01,tool-defaults,fundamentals"
"A pattern is tested and works correctly with one tool, then silently produces a different result when pasted unmodified into another tool. What's the most likely cause?","A mismatch between the regex flavor the pattern was written for and the flavor the second tool defaults to (BRE vs ERE) — both flavors treat the same pattern string as syntactically valid, just with different meanings, so there's no error to flag the mismatch.","demo-m05-01,tool-defaults,troubleshooting"
```

## Appendix B — Quiz

**`m05-01-regular-expressions-quiz.md`:**

````markdown
# Quiz — Demo M05-01: Regular Expressions

> One correct answer per question unless stated otherwise.
> Target: 80% or above before moving to Demo M05-05.

---

**Q1.** `grep 'cat' file.txt` is run against a file containing the lines "cat", "category", and "dog". Which lines match, and why?

- A) Only the exact "cat" line — `grep` matches whole lines by default
- B) "cat" and "category" — matching is unanchored (substring) by default, so "cat" appearing anywhere in the line is enough
- C) All three lines, since `grep` ignores content by default
- D) None, since a bare pattern with no flags never matches anything

<details>
<summary>Answer</summary>

**B** — This is the foundational default: `grep`/`sed`/`awk` search for a pattern anywhere within a line unless anchors restrict the position. "category" contains "cat" as a substring, so it matches just as validly as the exact "cat" line.

Trap: A wrongly assumes whole-line matching is the default; C and D both invent behaviors unrelated to how unanchored substring matching actually works.

</details>

---

**Q2.** `grep '192.168' file.txt` (unescaped dot) is run against a file containing `192.168.1.1` and `192x168x1x1`. Which lines match?

- A) Only `192.168.1.1`, since `.` in a pattern only matches an actual dot
- B) Both lines — unescaped, `.` matches ANY single character, so `x` satisfies it just as validly as a literal dot
- C) Neither line, since the pattern is malformed without escaping
- D) Only `192x168x1x1`, since unescaped `.` specifically excludes literal dots

<details>
<summary>Answer</summary>

**B** — `.` means "any single character" by default; a literal dot is just one of many characters that satisfies it. Escaping (`\.`) is required to match specifically a literal dot and exclude other characters like `x`.

Trap: A assumes the opposite of how `.` actually behaves; C invents an error condition that doesn't exist — the pattern is valid, just broader than likely intended; D gets the matching backwards.

</details>

---

**Q3.** `(ab)+` and `ab+` are both run against a line containing "abb". Do they produce the same result?

- A) Yes — parentheses are purely cosmetic and don't change matching behavior
- B) No — `ab+` matches "abb" (one `a`, two `b`s), while `(ab)+` does not, since it requires whole repetitions of the two-character unit "ab", and "abb" isn't `a` followed by `b` followed by `a` followed by `b`
- C) No — `(ab)+` matches "abb" but `ab+` does not
- D) Yes, both fail to match "abb"

<details>
<summary>Answer</summary>

**B** — Grouping scopes the `+` to the entire parenthesized unit. `ab+` only requires the `+` to apply to the final `b`; `(ab)+` requires the whole two-character sequence to repeat, which "abb" (a single `a` followed by two `b`s) doesn't satisfy.

Trap: A wrongly dismisses grouping's real semantic effect; C reverses which pattern actually matches; D wrongly claims neither matches.

</details>

---

**Q4.** `grep -E '^cat|dog$' file.txt` is run, intending to match lines starting with either "cat" or "dog". A line "I have a dog" (which doesn't start with either word) matches. Why?

- A) This is a `grep` bug
- B) Alternation has lower precedence than anchors and concatenation — the pattern actually parses as `(^cat)|(dog$)`, so a line ending in "dog" satisfies the second branch regardless of how it starts
- C) `$` in this context means "anywhere in the line," not specifically "end of line"
- D) `grep -E` ignores anchors entirely when `|` is present

<details>
<summary>Answer</summary>

**B** — This is the alternation-precedence gotcha: each side of `|` is evaluated independently, so the anchor on one side doesn't extend to the other. Explicit grouping (`^(cat|dog)`) is required to anchor the whole alternation.

Trap: A misattributes correct, standard regex precedence behavior as a bug; C and D both invent incorrect anchor semantics that aren't what's actually happening.

</details>

---

**Q5.** `grep -w 'root' /etc/passwd` is run against a file containing both a `root` line and a `rooter` line. What matches?

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

**Q6.** `grep -E 'ab{2,}c'` is run against a file containing `ac`, `abc`, `abbc`, and `abbbc`. Which lines match?

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

**Q7.** Are backreferences (`\1`) part of standard POSIX BRE, standard POSIX ERE, both, or neither?

- A) Both — backreferences are identically standard in BRE and ERE
- B) Native to standard POSIX BRE; NOT part of standard POSIX ERE at all — GNU tools supporting `\1` in ERE mode are extending beyond POSIX
- C) Neither — backreferences are a GNU extension in both flavors
- D) Standard only in ERE, not BRE

<details>
<summary>Answer</summary>

**B** — This is a precise, easy-to-get-backwards distinction: backreferences are genuinely part of the POSIX BRE standard, while POSIX ERE's grammar doesn't define them at all — their appearance in `sed -E`/`grep -E` is a GNU extension to ERE specifically.

Trap: A overstates ERE's standard support; C understates BRE's actual standard support; D reverses the correct answer entirely.

</details>

---

**Q8.** Why does the demo emphasize distinguishing "POSIX-standard" from "GNU extension" for features like BRE's `\|` and ERE's backreference support, given that both work fine on Ubuntu and Rocky9?

- A) It's a purely academic distinction with no practical consequence on any real system
- B) Because a script written assuming these extensions may behave differently or fail on a strictly POSIX-only or non-GNU regex implementation, even though it works reliably on the GNU-based systems this series targets
- C) Because these extensions are deprecated and will stop working in future GNU tool versions
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
| 7/8 or 8/8 | Import Anki cards, move to Demo M05-05 |
| 6/8 | Review the wrong answers, then proceed |
| 5/8 | Re-read the relevant sections, retry those questions |
| Below 5/8 | Re-read the full demo and redo the walkthrough before proceeding |
````