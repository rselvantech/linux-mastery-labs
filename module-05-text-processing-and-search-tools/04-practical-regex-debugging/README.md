# Demo M05-04 — Practical Regex Debugging

## Overview

Demo M05-01 taught regex syntax, 17a added PCRE, and 17b showed that the same
pattern means different things in different tools. This demo is about the
moment after that: the pattern is written, and it is wrong. Regex bugs are
unusual because they almost never produce an error. A bad pattern matches
too much, too little, or nothing at all, and every one of those looks like
a normal result. This demo teaches a repeatable way to build patterns that
you can trust, the four mistakes that cause most real bugs, and the one
failure mode (catastrophic backtracking) that can take a whole pipeline
down.

---

## Demo Objectives

By the end of this demo you will be able to:

1. Build a pattern incrementally against real sample data, verifying each step before adding the next
2. Explain greedy matching and fix it with a negated character class, and with a lazy quantifier where the tool supports one
3. Recognize that lazy quantifiers do nothing in ERE and `sed`, and that no error tells you so
4. Diagnose unescaped-metacharacter and anchoring mistakes, including the "zero matches looks like zero errors" trap
5. Show why a loose pattern accepts invalid data (an IP address with an octet of 300) and tighten it
6. Recognize catastrophic backtracking, reproduce it, and explain how it surfaces in `grep -P` versus other engines
7. Handle `grep`'s three exit codes so an engine error is never mistaken for "no match"

---

## Directory Structure

```
04-practical-regex-debugging/
├── README.md                                  # this file
├── m05-04-practical-regex-debugging-anki.csv  # Anki flashcard deck
└── m05-04-practical-regex-debugging-quiz.md   # standalone quiz
```

---

## Environments

| Task | Ubuntu 24.04 | Rocky Linux 9.8 | EC2 Amazon Linux |
|---|---|---|---|
| All lab tasks | ✅ | ✅ | Not required |
| Install | Nothing: `grep`, `sed`, and `python3` are preinstalled | Nothing: preinstalled | Not required |

> ⚠️ [VERIFY] Confirmed directly on GNU grep 3.11 and GNU sed 4.9 (the
> versions Ubuntu 24.04 ships): the outputs shown in Tasks 1–7, and the
> `grep`/`sed` evidence in the break-fix scenario, come from live runs. Only
> the Incident 2 script transcript (`check-labels.sh` and its output) is
> illustrative and was not executed as written. **Not run on Rocky 9.8**, which ships an older
> grep (3.6). The behavior should be the same, but confirm two things
> there: the exact wording of the `-P` backtracking-limit error in Task 6,
> and the timings, which will differ on any machine. Treat the timings as
> showing the *shape* of the growth, not the numbers.

---

## Recall Check

> Answer from memory before reading further. From Demos M05-01, 17a, and 17b.

1. `sed -n '/user_id=[0-9]+/p' file` prints nothing, but `grep -E
   'user_id=[0-9]+' file` matches. Why, and what are two ways to fix the
   `sed` command?

2. Which flavor does each of these use by default: `grep`, `grep -E`,
   `awk`, `sed`, `grep -P`?

3. Name two things PCRE (`grep -P`) can do that POSIX ERE cannot.

<details>
<summary>Answers</summary>

1. `sed` defaults to BRE, where `+` is a literal plus sign, not a
   quantifier. Fix it with `sed -nE` (switch to ERE) or escape the
   operator as `\+` (a GNU extension). Neither tool reports an error, which
   is why this class of bug ships.

2. `grep`: BRE. `grep -E`: ERE. `awk`: ERE. `sed`: BRE (ERE with `-E`).
   `grep -P`: PCRE.

3. Lookahead and lookbehind assertions, and lazy (non-greedy) quantifiers
   such as `.*?`. Also the `\d`, `\w`, `\s` shorthand classes. This demo
   leans on the lazy quantifiers and on PCRE's backtracking behavior.

</details>

---

## Part 1 — Concepts

### 1.1 The debugging workflow

A regex that fails does not raise an exception. It returns a count of zero,
or too many lines, or a plausible-looking wrong substring, and the only way
to notice is to look at what actually matched. So the workflow is the skill:

1. **Collect real sample data, including lines that must not match.** A
   pattern tested only against lines it should match can only prove it
   isn't too strict. It says nothing about whether it is too loose.
2. **Build incrementally.** Start with the smallest piece that matches
   something, confirm it, add one element, confirm again. When a long
   pattern fails, you don't know which of ten pieces is wrong. When you add
   one piece at a time, you always do.
3. **Look at the match itself, not the whole line.** `grep -o` prints only
   the matched text. `grep -n` adds line numbers. `grep -c` counts *lines*,
   not matches. To count matches, use `grep -o ... | wc -l`.
4. **Check the count against what you expect.** If you expect 7 lines and
   get 6, a pattern is silently too strict. If you expect 2 and get 7, it
   is too loose.
5. **Write down the flavor** (BRE, ERE, PCRE) next to the pattern. Half of
   all "it worked in my terminal" bugs are flavor bugs (Demo M05-03).

**A useful visibility trick** works in any `sed`: wrap the whole match in
markers using `&`, which stands for "whatever the pattern matched". It
shows the exact boundaries of the match, including where a greedy pattern
overshoots:

```bash
sed -nE '4s/msg="[^"]*"/<<&>>/p' app.log
```

### 1.2 Mistake 1: greedy matching

Quantifiers like `*` and `+` are **greedy**: they take as much text as
possible, then give characters back one at a time only if the rest of the
pattern fails to match. On a line with two quoted fields, `".*"` does not
stop at the first closing quote. It runs to the last one.

```
msg="file a.txt saved" note="ok"
    ^^^^^^^^^^^^^^^^^^^^^^^^^^^^
    msg=".*"  matches all of this, swallowing note="ok"
```

There are two fixes, and they are not equal:

| Fix | Pattern | Works in | Notes |
|---|---|---|---|
| **Negated character class** | `"[^"]*"` | BRE, ERE, PCRE, `sed`, `awk`, everywhere | Portable. Says exactly what you mean: "anything except a quote". Also faster, because the engine never has to backtrack. |
| **Lazy quantifier** | `".*?"` | **PCRE only** (`grep -P`) | Takes as little as possible. Convenient, but it only exists in one flavor. |

**Prefer the negated class by default.** Reserve `.*?` for cases where
there is no single character to exclude.

**The silent trap:** `.*?` in `grep -E` or `sed -E` produces no error. It
runs, and returns the greedy result. In POSIX ERE there is no lazy
quantifier, so the `?` does not make the `*` lazy. This is the Demo M05-03
lesson again: the pattern is valid in both flavors, with different
meaning.

### 1.3 Mistake 2: unescaped metacharacters

`.` matches **any** character, so `v1.2` matches `v1x2`. In a pattern
you want a literal dot, so you escape it (`\.`) or use a bracket
expression (`[.]`). The same applies to `*`, `+`, `?`, `[`, `(`, `{`, `|`,
`$`, and `^` whenever you mean them literally.

If you are searching for a literal string, especially one that arrives in
a shell variable, don't use a regex at all. `grep -F` treats the pattern as
fixed text, so there is nothing to escape and nothing a stray metacharacter
can do.

### 1.4 Mistake 3: anchoring

An unanchored pattern matches anywhere in the line. Two opposite bugs come
from this:

- **Too loose.** `user=` also matches inside `superuser=root`. Anchor the
  start with a word boundary (`\buser=`, a GNU extension that works in
  `grep -E` and `grep -P`) or an explicit delimiter (`(^| )user=`). `grep
  -w` does the same for whole words.
- **Too strict, and silent.** `^ERROR` looks right, but if every line
  starts with a timestamp, it matches *nothing*. The result is `0`, which
  reads exactly like "there were no errors today". This is the most
  dangerous form because the output is plausible. The defense is the
  positive control from §1.1: run the pattern against a line you *know*
  should match before trusting a zero.

### 1.5 Mistake 4: a pattern that "looks right" but accepts bad data

Matching an IPv4 address with the obvious pattern:

```
[0-9]+\.[0-9]+\.[0-9]+\.[0-9]+
```

also matches `300.1.1.1`, because `[0-9]+` accepts any number of digits. A
valid octet is 0–255, which needs alternation:

```
(25[0-5]|2[0-4][0-9]|1?[0-9]?[0-9])
```

Spelled out, this is: `250–255`, or `200–249`, or an optional leading `1`
plus up to two more digits (`0–199`). Repeating it with the dots gives a
strict pattern, with `\b` at both ends so it can't match a fragment of a
longer number. The point is not that you should always write this monster.
It is that "matches the shape" and "is valid" are different claims, and a
regex only checks the first unless you make it check the second. For log
filtering the loose form is usually fine; for anything that makes a
security or routing decision, tighten it, or validate in a real language.

### 1.6 Catastrophic backtracking

This is the failure mode that turns a regex into an outage.

Engines in PCRE, Python, Java, JavaScript, and most scripting languages
are **backtracking** engines. When a match fails partway, they back up and
try the other ways the pattern could have matched. Usually there are few
alternatives. But a **nested quantifier** such as `(a+)+` gives the engine
an exponential number of ways to divide the same run of characters between
the inner and outer `+`. On input that *almost* matches, the engine
explores all of them before giving up.

```
Pattern:  ^(a+)+b$
Subject:  aaaaaaaaaaaaaaaaaaaaaaaab!      (matches the shape, then fails at "!")
```

Measured on Python's `re` module, one extra character roughly doubles the
time:

| `a` count | Time (this session) |
|---|---|
| 20 | 0.07 s |
| 22 | 0.20 s |
| 24 | 0.75 s |
| 26 | 3.01 s |

Each two extra characters multiplies the time by about 4. Extrapolating
(not measured), 30 characters would take close to a minute, and 40 would
effectively never finish. A single log line, user-supplied string, or
filename of the wrong shape can pin a CPU. This is the basis of the
"ReDoS" class of denial-of-service bugs.

**How it shows up depends on the tool**, which is what makes it tricky:

| Engine | What happens on the bad input |
|---|---|
| Python `re`, many other libraries | Hangs. CPU at 100% until it finishes or you kill it. |
| `grep -P` (GNU grep 3.11) | Gives up quickly and prints `exceeded PCRE's backtracking limit`, then **exits with status 2**. |
| `grep -E` (DFA-based) | Unaffected. It does not backtrack, so the same pattern returns in milliseconds. |

The `grep -P` behavior sounds like a relief, but it has a sharp edge: exit
status 2 is neither "match" (0) nor "no match" (1), and a script that only
asks `if grep -qP ...` treats 2 exactly like 1. The line was never actually
evaluated, and the script carries on as though it was clean. See Task 7.

**Prevention, in order of preference:**

1. **Don't nest quantifiers over overlapping sets.** `(\w+\s?)+` is the
   textbook example: the optional `\s?` lets the engine split a single long
   word every possible way. Make the separator mandatory and unambiguous:
   `\w+( \w+)*`.
2. **Use negated classes instead of `.*`** where you can. They give the
   engine nowhere to backtrack to.
3. **Anchor** the pattern, so failing candidates are rejected early.
4. **In PCRE, use possessive quantifiers or atomic groups** (`a++`,
   `(?>a+)`), which forbid backtracking into the group.
5. **Use ERE (`grep -E`) when you don't need PCRE features**, especially on
   untrusted or very large input, because it cannot be driven into
   exponential time this way.

**A trap for anyone testing this:** modern PCRE2 does a cheap pre-check for
any literal character the pattern *requires*. If the test string doesn't
contain it, the match fails instantly and the slowdown never appears. For
`^(a+)+b$`, a subject of `aaaa...c` (no `b` anywhere) returns in
milliseconds. To reproduce the problem, the subject must contain the
required character but still fail, such as `aaaa...b!`.

---

## Part 2 — Hands-On Lab

### Commands used in this demo

| Command | Flag meaning | Purpose |
|---|---|---|
| `grep -o` | `-o`: print only the matched part | See exactly what a pattern matched |
| `grep -c` | `-c`: count matching **lines** | Quick count (not a count of matches) |
| `grep -F` | `-F`: fixed string, no regex | Search for literal text safely |
| `grep -w` | `-w`: whole word | Avoid matching inside longer words |
| `grep -P` | `-P`: PCRE | Lazy quantifiers, lookarounds, `\b`, `\w` |
| `sed -E 's/PAT/<<&>>/'` | `&`: the whole match | Show match boundaries |
| `echo $?` | — | Read the exit status of the previous command |
| `timeout N CMD` | `N`: seconds | Kill a command that runs too long |

### Task 1 — Create the sample log

The log has seven lines. Line 4 has two quoted fields (`msg` and `note`),
line 5 has an invalid IP, and one line has a `latency=` field the others
lack.

```bash
mkdir -p /tmp/regex-lab && cd /tmp/regex-lab
cat > app.log <<'EOF'
2026-10-14 09:12:01 INFO  user=alice action=login ip=10.0.4.21 msg="login ok"
2026-10-14 09:12:07 WARN  user=bob action=login ip=10.0.4.22 msg="bad password"
2026-10-14 09:12:09 ERROR user=bob action=login ip=10.0.4.22 msg="account locked"
2026-10-14 09:13:15 INFO  user=carol action=upload ip=192.168.1.7 msg="file a.txt saved" note="ok"
2026-10-14 09:13:40 ERROR user=dave action=upload ip=300.1.1.1 msg="invalid source"
2026-10-14 09:14:02 INFO  user=alice action=logout ip=10.0.4.21 msg="bye"
2026-10-14 09:14:30 DEBUG user=svc-1 action=healthcheck ip=127.0.0.1 msg="ok" latency=12ms
EOF

wc -l app.log
# Confirm: 7 lines.
grep -c 'ERROR' app.log
grep -ci 'error' app.log
# Confirm: both print 2. These are your reference counts for later tasks.
```

### Task 2 — Build a pattern incrementally

Goal: list the source IP of every **ERROR** line. Don't write it in one
go. Build it up and check each step.

```bash
cd /tmp/regex-lab

# Step 1: the smallest thing that matches. Is the field there at all?
grep -c 'ip=' app.log
# Expected: 7 (every line has an ip= field).

# Step 2: print only what matched, not the line
grep -oE 'ip=[0-9.]+' app.log | head -3
# Expected: ip=10.0.4.21, ip=10.0.4.22, ip=10.0.4.22

# Step 3: make it stricter (four dotted groups) and confirm the count
grep -oE 'ip=[0-9]+\.[0-9]+\.[0-9]+\.[0-9]+' app.log | wc -l
# Expected: 7 (the stricter pattern still matches every line).

# Step 4: restrict to ERROR lines with a second, separate filter
grep -E '^[^ ]+ [^ ]+ ERROR' app.log | grep -oE 'ip=[0-9.]+'
# Expected: ip=10.0.4.22 and ip=300.1.1.1
```

Each step is one small change and one check. If Step 4 had returned
nothing, you would know the bug is in the `ERROR` filter, because Steps 1–3
already passed. Note that the answer already contains a suspicious value
(`300.1.1.1`), which Task 5 follows up.

Count matches versus lines, since the two differ:

```bash
grep -c '="' app.log            # lines containing ="
grep -o '="' app.log | wc -l    # total occurrences of ="
# Expected: 7, then 8. Line 4 has two (msg= and note=), so there are more
# matches than matching lines.
```

### Task 3 — Greedy versus non-greedy

Line 4 is the interesting one. Compare six commands against it:

```bash
cd /tmp/regex-lab

# 1. Greedy .* in ERE: overshoots into note="ok"
grep -oE 'msg=".*"' app.log | sed -n 4p
# Expected: msg="file a.txt saved" note="ok"

# 2. Negated class: stops at the first closing quote
grep -oE 'msg="[^"]*"' app.log | sed -n 4p
# Expected: msg="file a.txt saved"

# 3. Lazy quantifier in PCRE: also stops correctly
grep -oP 'msg=".*?"' app.log | sed -n 4p
# Expected: msg="file a.txt saved"

# 4. The same lazy quantifier in ERE: no error, but NOT lazy
grep -oE 'msg=".*?"' app.log | sed -n 4p
# Expected: msg="file a.txt saved" note="ok"   (identical to #1)
```

Results #1 and #4 are identical. The `?` in #4 did nothing, and `grep`
raised no complaint. That is the silent flavor trap from Demo M05-03 in a new
costume.

Now the same bug in a **substitution**, where it destroys data instead of
just returning too much:

```bash
# 5. sed, greedy: swallows note="ok" along with msg
sed -E 's/msg=".*"/MSG/' app.log | sed -n 4p
# Expected: ...ip=192.168.1.7 MSG        (note="ok" is gone)

# 6. sed, negated class: replaces only the msg field
sed -E 's/msg="[^"]*"/MSG/' app.log | sed -n 4p
# Expected: ...ip=192.168.1.7 MSG note="ok"
```

`sed` has no lazy quantifier at all, so the negated class is not just
preferred here, it is the only option. Now see the boundaries directly
with the `&` trick from §1.1:

```bash
sed -nE '4s/msg=".*"/<<&>>/p' app.log
# Expected: ... <<msg="file a.txt saved" note="ok">>
sed -nE '4s/msg="[^"]*"/<<&>>/p' app.log
# Expected: ... <<msg="file a.txt saved">> note="ok"
```

### Task 4 — Metacharacters and anchoring

```bash
cd /tmp/regex-lab

# Unescaped dot: matches a string it shouldn't
echo 'file-v1x2' | grep -E 'v1.2'
# Expected: file-v1x2          (wrongly matched: . means "any character")
echo 'file-v1.2' | grep -oE 'v1\.2'
# Expected: v1.2               (escaped, so only a literal dot matches)
printf 'v1.2\nv1x2\n' | grep -F 'v1.2'
# Expected: v1.2               (-F: literal text, no escaping needed)

# The "zero looks like success" anchoring trap
grep -c 'ERROR' app.log                       # 2
grep -c '^ERROR' app.log                      # 0   <- silently wrong
grep -cE '^[^ ]+ [^ ]+ ERROR' app.log         # 2   <- anchored correctly
# Expected: 2, 0, 2. The middle answer is a lie: lines start with a
# timestamp, so ^ERROR can never match, and "0" reads like "no errors".

# Too loose: user= also matches inside superuser=
printf 'user=alice\nsuperuser=root\n' | grep -c 'user='
printf 'user=alice\nsuperuser=root\n' | grep -cE '(^| )user='
printf 'user=alice\nsuperuser=root\n' | grep -cP '\buser='
# Expected: 2, 1, 1.

# BRE vs ERE reminder: + is literal in BRE
grep -c 'user=[a-z]+' app.log       # 0   (literal plus sign)
grep -cE 'user=[a-z]+' app.log      # 7
# Expected: 0 then 7. Same data, silently different answer.
```

### Task 5 — A loose pattern accepts bad data

```bash
cd /tmp/regex-lab

# Loose: matches the shape of an IP
grep -oE '[0-9]+\.[0-9]+\.[0-9]+\.[0-9]+' app.log | sort -u
# Expected:
#   10.0.4.21
#   10.0.4.22
#   127.0.0.1
#   192.168.1.7
#   300.1.1.1            <- not a valid IP

# Strict: each octet limited to 0-255
grep -oE '\b((25[0-5]|2[0-4][0-9]|1?[0-9]?[0-9])\.){3}(25[0-5]|2[0-4][0-9]|1?[0-9]?[0-9])\b' app.log | sort -u
# Expected: the same list without 300.1.1.1

# Use the strict pattern as a filter: which ERROR lines have an invalid IP?
grep -E '^[^ ]+ [^ ]+ ERROR' app.log \
  | grep -vE 'ip=((25[0-5]|2[0-4][0-9]|1?[0-9]?[0-9])\.){3}(25[0-5]|2[0-4][0-9]|1?[0-9]?[0-9]) '
# Expected: only the dave/300.1.1.1 line.
```

The trailing space in the last pattern is deliberate: it ends the IP so a
valid-looking prefix of a longer value can't pass.

### Task 6 — Reproduce catastrophic backtracking

> Run these inside `timeout` as shown. Do not raise the counts past the
> values given. The Python loop gets about four times slower with each two
> extra characters.

First, the engine that hangs. Python's `re` is a backtracking engine:

```bash
python3 -I - <<'EOF'
import re, time
for n in (20, 22, 24, 26):
    s = "a" * n + "b!"
    t = time.perf_counter()
    re.match(r'^(a+)+b$', s)
    print(n, f"{time.perf_counter() - t:.2f}s")
EOF
# Measured here: 0.07s, 0.20s, 0.75s, 3.01s. Your numbers will differ,
# but each step should multiply the time, not add to it.
```

Now the same pattern through `grep`, in both flavors:

```bash
for n in 20 24 28; do
  s="$(printf 'a%.0s' $(seq 1 $n))b!"
  echo "n=$n"
  timeout 20 grep -cP '^(a+)+b$' <<<"$s"; echo "  PCRE rc=$?"
  timeout 20 grep -cE '^(a+)+b$' <<<"$s"; echo "  ERE  rc=$?"
done
# Expected:
#   n=20  PCRE: 0, rc=1     ERE: 0, rc=1
#   n=24  PCRE: "exceeded PCRE's backtracking limit", rc=2     ERE: 0, rc=1
#   n=28  PCRE: "exceeded PCRE's backtracking limit", rc=2     ERE: 0, rc=1
```

Three things to read from that output:

- The ERE column never changes. A DFA-based engine has no exponential case.
- `grep -P` doesn't hang. It hits a built-in limit and reports an error,
  with **exit status 2**.
- At n=20 the PCRE result looks fine, which shows that the same pattern
  can pass in testing and fail in production purely on input length.

Show the safe versions of the same intent:

```bash
s="$(printf 'a%.0s' $(seq 1 28))b!"
grep -cP '^a+b$' <<<"$s"; echo "rc=$?"       # no nesting: fast, rc=1
grep -cP '^(a++)+b$' <<<"$s"; echo "rc=$?"   # possessive: fast, rc=1
grep -cP '^(?>a+)+b$' <<<"$s"; echo "rc=$?"  # atomic group: fast, rc=1
# Expected: 0 and rc=1 each time. Fast, and no engine error.
```

And the trap for testers. With no `b` in the subject, PCRE2 rejects the
subject instantly via its required-character pre-check, so the slowdown
vanishes and your test "passes":

```bash
s="$(printf 'a%.0s' $(seq 1 30))c"
grep -cP '^(a+)+b$' <<<"$s"; echo "rc=$?"
# Expected: 0, rc=1, instantly. A "pass" that proves nothing.
```

### Task 7 — The exit code that gets swallowed

```bash
s="$(printf 'a%.0s' $(seq 1 28))b!"

if grep -qP '^(a+)+b$' <<<"$s" 2>/dev/null; then
  echo MATCH
else
  echo "treated as no match"
fi
# Expected: treated as no match

grep -qP '^(a+)+b$' <<<"$s" 2>/dev/null; echo "actual rc=$?"
# Expected: actual rc=2
```

The `if` reports "no match", but grep never decided that. It gave up. To
tell the three outcomes apart, read the status explicitly:

```bash
grep -qP '^(a+)+b$' <<<"$s"; rc=$?
case $rc in
  0) echo "match" ;;
  1) echo "no match" ;;
  *) echo "grep failed (rc=$rc), result unknown" >&2 ;;
esac
# Expected: grep prints its error, then "grep failed (rc=2), result unknown".
```

Reference: `grep` exits `0` when it found a match, `1` when it ran fine and
found none, and `2` on an error. Demo M01-02 introduced `$?`; this is where
getting it wrong costs something.

### Task 8 — Clean up

```bash
cd /tmp && rm -rf /tmp/regex-lab
ls /tmp/regex-lab 2>&1
# Expected: "No such file or directory".
```

---

## What You Learned

- ✅ Built a pattern one verified step at a time, and checked counts against expectations
- ✅ Explained greedy matching and fixed it with a negated class (portable) and a lazy quantifier (PCRE only)
- ✅ Saw that `.*?` in ERE and `sed` silently does nothing
- ✅ Diagnosed unescaped metacharacters and anchoring bugs, including the silent zero
- ✅ Showed that a loose IP pattern accepts `300.1.1.1`, and tightened it
- ✅ Reproduced catastrophic backtracking and compared how Python, `grep -P`, and `grep -E` respond
- ✅ Handled `grep`'s exit codes so an engine failure is not read as "no match"

**Key Takeaway:** A regex bug doesn't announce itself. It returns a number
or a string that looks reasonable. Every technique in this demo exists to
make the result visible: build incrementally, print only the match, count
against an expectation, test against lines that must *not* match, and
never read a bare `0` or a non-zero exit status without asking what it
actually means.

---

## Part 3 — Break-Fix Scenario

### Scenario — NovaTech Ops: two regex incidents

> **Incident report — NovaTech Ops, ticket #4562**
>
> Alert 1: "The nightly audit report shows truncated and garbled action
> messages since last Tuesday's release, but only for some rows." Alert 2:
> "The build-log checker reported all clean for three days, but QA says
> labelled sections are missing from the output."

**Incident 1 — garbled message field**

The extractor script has not changed. Last Tuesday the application began
appending a `note="..."` field to some log lines.

```bash
$ cat extract-msg.sh
sed -nE 's/.*msg="(.*)".*/\1/p' "$1"

$ ./extract-msg.sh app.log | sed -n 1,3p
login ok
bad password
account locked
$ ./extract-msg.sh app.log | sed -n 4p
file a.txt saved" note="ok
```

Tasks:
1. Why do rows 1–3 look right while row 4 is wrong, given that the script
   did not change?
2. Explain exactly how `(.*)` produced `file a.txt saved" note="ok`.
3. Give a fix, and say why it is better than switching the pattern to
   `(.*?)`.

**Incident 2 — checker reports "all clean"**

```bash
$ cat check-labels.sh
while IFS= read -r line; do
    if grep -qP '^(\w+\s?)+:$' <<<"$line" 2>/dev/null; then
        echo "label: $line"
    fi
done < build.log

$ ./check-labels.sh
label: build step ok:
label: label:
$ # three days of "all clean", but QA found labelled lines that were skipped

$ line='aaaaaaaaaaaaaaaaaaaaaaaaaaaaaa!:'
$ echo "$line" | grep -P '^(\w+\s?)+:$'; echo "rc=$?"
grep: (standard input): exceeded PCRE's backtracking limit
rc=2
```

Tasks:
1. What is wrong with the pattern, and why does it fail only on some lines?
2. Why did the script report success instead of failing, even though grep
   printed an error?
3. Give a safe replacement pattern with the same intent, and a change to
   the script so this failure cannot be silent again.

<details>
<summary>Answers</summary>

**Incident 1:** Rows 1–3 have only one quoted field, so the greedy `(.*)`
and the correct answer happen to coincide. Row 4 has two, and the release
that added `note="..."` is the trigger, not any change to the script. In
`.*msg="(.*)".*`, the middle `(.*)` is greedy: it first runs to the end of
the line, then backs up only as far as needed for a `"` to match the
literal quote that follows. The *last* quote on the line is the one that
closes `note="ok"`, so the capture ends there, taking `file a.txt saved"
note="ok` with it. The fix is a negated class: `s/.*msg="([^"]*)".*/\1/p`,
which prints `file a.txt saved`. This is better than `(.*?)` because `sed`
has no lazy quantifier at all (it would be a silent no-op in ERE, like
Task 3 #4), whereas `[^"]*` works in every tool and says exactly what is
meant: everything up to the next quote.

**Incident 2:** The pattern `^(\w+\s?)+:$` nests a quantifier over a group
whose separator (`\s?`) is optional. On a long run of word characters
followed by something that can't match, the engine can divide the same
characters among the inner and outer `+` in exponentially many ways, and
tries them all before failing. Short or well-formed lines match or fail
quickly, so most lines are evaluated correctly and the problem hides
until a line of the wrong shape arrives. The script reported success
because `if grep ...` only tests whether the status is zero. Status `2`
(an engine error) is non-zero, so it takes the same branch as `1` (no
match). The script also discards stderr (`2>/dev/null`), so the error
message was never seen. A safe
replacement with a mandatory, unambiguous separator is
`^\w+( \w+)*:$`, which matches `build step ok:` and `label:`, rejects the
bad line instantly, and cannot backtrack exponentially. (`^[\w ]+:$` is
also safe but accepts runs of spaces.) To make failures loud, capture the
status and treat anything other than 0 or 1 as an error, as in Task 7:
`grep -qP "$pat" <<<"$line"; rc=$?; [ "$rc" -gt 1 ] && { echo "grep failed
rc=$rc" >&2; exit 2; }`. Setting `set -e` is not enough here, since a
failing command inside an `if` condition does not trigger it.

</details>

---

## Interview Prep

**Q1. A pattern works on your sample data but returns wrong results in production. What is your approach?**

First check the flavor: is the production tool the same engine I tested in
(BRE versus ERE versus PCRE, Demo M05-03)? Then reproduce with the actual
production line, not a cleaned-up sample, and print only the matched text
with `grep -o` or a `sed 's/PAT/<<&>>/'` wrapper to see where the match
begins and ends. The usual culprits are greedy quantifiers swallowing a
field that didn't exist in my sample, a missing anchor, and an unescaped
dot. I would shrink the pattern to the smallest part that still shows the
problem, then add pieces back one at a time.

---

**Q2. What is catastrophic backtracking, and how would you prevent it in a log-processing pipeline?**

It is what happens when a backtracking regex engine tries exponentially
many ways to match input that almost matches, usually because of nested
quantifiers such as `(a+)+` or `(\w+\s?)+`. One badly shaped line can then
pin a CPU or stall the job. To prevent it, I avoid nested quantifiers over
overlapping sets, use negated character classes instead of `.*`, anchor
patterns, use possessive quantifiers or atomic groups in PCRE, and use
`grep -E` where I don't need PCRE features, because its DFA-based matching
has no exponential case. I also read `grep`'s exit status, since
`grep -P` reports an overrun as status 2, which a plain `if` silently
treats as "no match".

---

**Q3. Why prefer `[^"]*` over `.*?` for matching a quoted string?**

It is portable: it works in BRE, ERE, PCRE, `sed`, and `awk`, while a lazy
quantifier exists only in PCRE and in other flavors silently behaves like
the greedy version with no error. It is also more precise, since it states
the actual rule (everything up to the next quote), and cheaper, since the
engine never has to give characters back.

---

## Key Takeaways

1. **Regex bugs are silent by design.** A wrong pattern returns a count or
   a string that looks plausible, so the defense is making the match
   visible: `grep -o`, `sed 's/PAT/<<&>>/'`, and counts compared to an
   expectation.
2. **Build incrementally and test negatives.** Add one element, check it,
   repeat. A pattern tested only on lines it should match proves nothing
   about whether it is too loose.
3. **Greedy quantifiers overshoot.** `".*"` runs to the last quote on the
   line. The portable fix is a negated class (`"[^"]*"`); lazy `.*?` works
   only in PCRE and does nothing, silently, in ERE and `sed`.
4. **A zero from an anchored pattern may be a lie.** `^ERROR` on lines that
   start with a timestamp returns `0`, which reads as "no errors". Always
   run a positive control before trusting an empty result.
5. **Escape literal metacharacters, or use `grep -F`.** `v1.2` matches
   `v1x2`. `\b` or `-w` stops `user=` matching inside `superuser=`.
6. **Matching the shape is not validating the value.** The loose IP
   pattern accepts `300.1.1.1`. Tighten it, or validate elsewhere, when a
   decision depends on the result.
7. **Nested quantifiers can cause exponential time.** Python hangs,
   `grep -P` stops with an error and exit status 2, and `grep -E` is
   unaffected. Test with a subject that contains the required characters,
   or the check passes for the wrong reason.
8. **Read grep's exit status: 0 match, 1 no match, 2 error.** A plain `if`
   treats 2 as "no match", so an engine failure looks like clean data.

---

## Quick Commands Reference

### Seeing what matched

| Command | Purpose |
|---|---|
| `grep -o PATTERN FILE` | Print only the matched text |
| `grep -n PATTERN FILE` | Add line numbers |
| `grep -c PATTERN FILE` | Count matching **lines** |
| `grep -o PATTERN FILE \| wc -l` | Count total matches |
| `sed -nE 's/PAT/<<&>>/p' FILE` | Show exact match boundaries |

### Safer matching

| Command | Purpose |
|---|---|
| `grep -F STRING FILE` | Literal match, no regex |
| `grep -w WORD FILE` | Whole-word match |
| `grep -P '\bWORD'` / `grep -E '\bWORD'` | Word boundary (GNU) |
| `[^"]*` | Portable "everything up to the next quote" |
| `.*?` | Lazy quantifier (PCRE only) |
| `a++` / `(?>...)` | Possessive / atomic (PCRE), forbid backtracking |

### Exit status

| Status | Meaning |
|---|---|
| `0` | At least one match |
| `1` | Ran fine, no match |
| `2` | Error (bad pattern, missing file, PCRE backtracking limit) |

---

## Appendix A — Anki Cards

**`m05-04-practical-regex-debugging-anki.csv`:**

```
#deck:Linux Mastery::Module 5 - Text Processing & Search Tools::Demo M05-04 - Practical Regex Debugging
#separator:Comma
#columns:Front,Back,Tags
"Does grep -c count matches or lines? How do you count total matches?","Lines. A line with two matches counts once. Use grep -o PATTERN FILE | wc -l to count every match.","demo17c,grep,counting"
"What sed trick shows exactly where a pattern's match begins and ends?","Wrap the match using &, which stands for the whole match: sed -nE 's/PATTERN/<<&>>/p'. A greedy overshoot becomes visible inside the markers.","demo17c,sed,debugging"
"When should you use grep -F instead of a regex?","When searching for literal text, especially text from a shell variable. Nothing is interpreted as a metacharacter, so there is nothing to escape.","demo17c,grep,fixed-string"
"What is the difference between \buser= and a plain user= pattern?","The plain pattern also matches inside longer words such as superuser=. \b (a GNU extension that works in grep -E and grep -P) requires a word boundary before it. grep -w does the same for whole words.","demo17c,anchoring,word-boundary"
"Why is a negated class like [^""]* faster than a lazy .*? even in PCRE?","The engine never has to give characters back: the class can only match non-quote characters, so there is nothing to backtrack over.","demo17c,performance,negated-class"
"What do grep exit statuses 0, 1, and 2 mean?","0: at least one match. 1: ran fine, no match. 2: an error (bad pattern, missing file, PCRE backtracking limit).","demo17c,grep,exit-codes"
"You test a nested-quantifier pattern against aaaa...c and it returns instantly. Is the pattern safe?","Not necessarily. PCRE2 pre-checks for required literal characters; if the subject has no b for ^(a+)+b$, it fails immediately. Test with a subject that contains the required character but still fails, such as aaaa...b!.","demo17c,backtracking,testing"
"What do possessive quantifiers and atomic groups do in PCRE?","They forbid backtracking into the quantified part once it has matched. a++ and (?>a+) cannot be re-divided, which removes the exponential case.","demo17c,pcre,backtracking"
"Which grep mode is immune to catastrophic backtracking, and why?","grep -E (and plain grep): GNU grep uses a DFA-based matcher for these, which does not backtrack, so nested quantifiers have no exponential case.","demo17c,backtracking,engines"
"Why must a regex be tested against lines that should NOT match?","A pattern tested only on matching lines can only show that it is not too strict. Negative samples are the only way to find out that it is too loose.","demo17c,workflow,testing"
"Why is the ERE pattern 'msg="".*?""' in grep -E dangerous?","It runs without an error but ? does not make * lazy in ERE, so it returns the same greedy result as .*. Lazy quantifiers exist only in PCRE.","demo17c,erre,lazy-quantifier"
"Why can a script using set -e still miss a grep -P engine error inside an if?","A failing command used as an if condition does not trigger set -e. Status 2 is simply non-zero, so the if takes the else branch exactly as it would for no match.","demo17c,scripting,exit-codes"
```

---

## Appendix B — Quiz

**`m05-04-practical-regex-debugging-quiz.md`:**

````markdown
# Quiz — Demo M05-04: Practical Regex Debugging

> One correct answer per question unless stated otherwise.
> Target: 80% or above before moving to Demo M05-05.

---

**Q1.** A log line contains `msg="file saved" note="ok"`. `sed -E 's/msg=".*"/MSG/'` produces `MSG`, losing the `note` field. What is the correct, portable fix?

- A) `sed -E 's/msg=".*?"/MSG/'`
- B) `sed -E 's/msg="[^"]*"/MSG/'`
- C) `sed 's/msg=".*"/MSG/'`
- D) `sed -E 's/msg="."/MSG/'`

<details>
<summary>Answer</summary>

**B** — The negated class `[^"]*` matches everything up to the next quote, so the match stops at the end of the `msg` field. It works in every tool.

Trap: A has no effect in `sed`, which has no lazy quantifier. C is the same greedy pattern in BRE. D matches only a single character.

</details>

---

**Q2.** `grep -oE 'msg=".*?"' app.log` is run on the line from Q1. What happens?

- A) It returns only `msg="file saved"` because `?` makes `*` lazy
- B) It prints a syntax error
- C) It runs without error and returns the same greedy result as `.*`
- D) It matches nothing, because `?` is a literal character in ERE

<details>
<summary>Answer</summary>

**C** — In ERE there is no lazy quantifier, so the `?` doesn't change how `*` behaves, and `grep` accepts the pattern without complaint. Lazy quantifiers exist only in PCRE (`grep -P`).

Trap: A assumes PCRE behavior. B and D assume an error or literal match that doesn't occur.

</details>

---

**Q3.** `grep -c '^ERROR' app.log` prints `0`, yet `grep -c 'ERROR' app.log` prints `2`. Every line begins with a timestamp. What is the most likely explanation?

- A) The file has Windows line endings
- B) The `^` anchor requires `ERROR` at the start of the line, which never happens
- C) `grep -c` does not support anchors
- D) The `ERROR` lines contain invisible characters

<details>
<summary>Answer</summary>

**B** — `^` anchors to the start of the line, and the line starts with a timestamp, so the pattern can never match. The result `0` looks like "no errors" but is a pattern bug.

Trap: A, C, and D invent causes. Confirming against the unanchored count of 2 is the positive control that exposes it.

</details>

---

**Q4.** `echo 'file-v1x2' | grep -E 'v1.2'` prints a match. Why?

- A) `grep -E` treats the digits as wildcards
- B) The unescaped `.` matches any single character, including `x`
- C) The pattern is matching case-insensitively
- D) `grep` always matches partial words

<details>
<summary>Answer</summary>

**B** — `.` is a metacharacter meaning "any character". Escape it as `\.`, put it in a bracket expression `[.]`, or search with `grep -F 'v1.2'`.

</details>

---

**Q5.** The pattern `[0-9]+\.[0-9]+\.[0-9]+\.[0-9]+` is used to find IP addresses and returns `300.1.1.1`. Why?

- A) It is a bug in `grep -E`
- B) `[0-9]+` accepts any number of digits, so it checks the shape of an address, not that each octet is 0–255
- C) `300.1.1.1` is a valid IP address
- D) The `\.` escapes are matching any character

<details>
<summary>Answer</summary>

**B** — The pattern verifies four dotted groups of digits, nothing more. Limiting each octet to 0–255 needs alternation such as `(25[0-5]|2[0-4][0-9]|1?[0-9]?[0-9])`.

Trap: C is factually wrong (octets max out at 255), and D misreads `\.`, which matches a literal dot.

</details>

---

**Q6.** What makes `^(a+)+b$` dangerous on a backtracking engine?

- A) The `$` anchor forces a full-file scan
- B) The nested quantifier lets the engine divide a run of `a` characters between the inner and outer `+` in exponentially many ways, and it tries all of them when the match fails
- C) `+` always consumes the whole input
- D) The pattern is invalid in PCRE

<details>
<summary>Answer</summary>

**B** — Nested quantifiers over the same character set create an exponential number of possible splits. On input that almost matches, a backtracking engine explores them all before giving up, and each extra character roughly doubles the time.

Trap: A and C describe behavior that doesn't exist, and D is wrong: the pattern is valid, which is why it ships.

</details>

---

**Q7.** A script runs `if grep -qP "$pat" <<<"$line"; then ...; fi`. For one pathological line, `grep` prints `exceeded PCRE's backtracking limit`. What does the `if` do, and why does it matter?

- A) The script aborts, because grep failed
- B) The `if` takes the "no match" branch, because exit status 2 is non-zero, so an unevaluated line is treated as clean
- C) The `if` takes the "match" branch
- D) `grep` retries with a larger limit and returns a normal result

<details>
<summary>Answer</summary>

**B** — `if` only distinguishes zero from non-zero. Status 2 (error) lands in the same branch as status 1 (no match), so the line is silently treated as "no match". Read `$?` explicitly and treat anything above 1 as a failure.

Trap: A assumes `set -e`-style behavior, which doesn't apply to an `if` condition. C and D are not what happens.

</details>

---

**Q8.** A long regex returns nothing. What is the best first debugging step?

- A) Rewrite it from scratch in PCRE
- B) Shrink it to the smallest piece that matches something, then add one element at a time, checking the output after each
- C) Add `.*` at the start and end to make it more permissive
- D) Switch from `grep` to `awk`

<details>
<summary>Answer</summary>

**B** — Incremental building isolates which element introduced the failure. Rewriting (A) or switching tools (D) discards that information, and padding with `.*` (C) tends to make a pattern match too much instead of finding the real fault.

</details>

---

| Score | Action |
|---|---|
| 7/8 or 8/8 | Import Anki cards, move to Demo M05-05 |
| 6/8 | Review the wrong answers, then proceed |
| 5/8 | Re-read the relevant sections, retry those questions |
| Below 5/8 | Re-read the full demo and redo the walkthrough before proceeding |
````