# Demo 34d — Script Testing

## Overview

Demo 34c gave you tools to debug a script after something already went
wrong. This demo closes Module 9 with the tools that catch a problem
*before* it ships — a lightweight assert-based testing pattern you can
write in pure bash with no framework, `bats` as the structured framework
version of the same idea, and **idempotency testing**: confirming a
script is genuinely safe to run more than once, which matters enormously
for anything that might actually run twice in production (a retried
deploy, a re-triggered automation, a human running a setup script again
"just to be sure").

## Demo Objectives

1. Write a lightweight, assert-based test for a bash function with no
   external framework
2. Explain what `bats` adds over a hand-rolled assert pattern, as a
   forward reference
3. Explain what idempotency means for a script, and why it matters
   specifically for infrastructure/automation scripts
4. Test a script for idempotency by running it twice and comparing
   results
5. Identify and fix the single most common idempotency bug: an append
   with no existence check first

## Directory Structure

```
34d-script-testing/
├── README.md                          # this file
├── 34d-script-testing-anki.csv        # Anki flashcard deck
└── 34d-script-testing-quiz.md         # standalone quiz
```

## Environments

| Task | Ubuntu 24.04 | Rocky Linux 9.8 | EC2 Amazon Linux |
|---|---|---|---|
| Manual assert testing / idempotency testing | ✅ (bash only) | ✅ (bash only) | Not required |
| `bats` | `sudo apt install bats` | `sudo dnf install bats` (may need EPEL) | Not required |

> **Verification note:** §1.1 and §1.3–1.4 (the assert-based test
> pattern, and both the non-idempotent and idempotent script examples)
> were run live in the authoring sandbox — all output shown is real,
> including a deliberately failing assertion. `bats` isn't installed here
> (no network) and §1.2 is doc-sourced; recommend confirming its exact
> syntax on a real machine.

## Recall Check — 34c-script-debugging

Answer these from memory before continuing — no looking back at Demo 34c.

**Q1.** Does `set -x` show a command's literal source text or its value
after expansion?

**Q2.** What does `$BASH_COMMAND` show inside a `trap DEBUG` handler,
and how does that differ from `set -x`'s output?

**Q3.** Why is structured, leveled logging a better fit than `set -x`
for a bug that can't be reliably reproduced on demand?

<details>
<summary>Answers</summary>

**A1.** After expansion/evaluation — `set -x` shows what the shell
actually did, not the literal source line.

**A2.** The literal, unexpanded source line — the opposite of `set -x`'s
post-expansion output.

**A3.** `set -x`/`trap DEBUG` both require the script to be actively
running while watched; an always-on log already has a trail leading up
to the next failure without needing it to recur under observation.

</details>

## Part 1 — Concepts

### 1.1 A lightweight assert pattern — no framework required

```bash
pass=0; fail=0
assert_eq() {
    local desc="$1" expected="$2" actual="$3"
    if [ "$expected" == "$actual" ]; then
        echo "PASS: $desc"; pass=$((pass+1))
    else
        echo "FAIL: $desc (expected [$expected], got [$actual])"; fail=$((fail+1))
    fi
}
```

Testing a small math library:

```bash
source mathutils.sh
assert_eq "add 2+3" "5" "$(add 2 3)"
assert_eq "add -1+1" "0" "$(add -1 1)"
assert_eq "add bug check" "10" "$(add 4 5)"
echo "$pass passed, $fail failed"
[ "$fail" -eq 0 ]
```

Real output, including a deliberately wrong expectation to show a real
failure:

```
PASS: add 2+3
PASS: add -1+1
FAIL: add bug check (expected [10], got [9])
---
2 passed, 1 failed
```

(exit code: 1)

**The exit code at the end matters as much as the printed output** — `[
"$fail" -eq 0 ]` as the script's final statement means the test script
itself exits non-zero when anything failed, which is what lets this be
wired into CI or a pre-deploy check (Demo 34's `set -e`-style "stop the
pipeline on failure" pattern) rather than needing a human to read the
output and notice a `FAIL` line.

### 1.2 bats — the structured version of the same idea

`bats` (Bash Automated Testing System) formalizes the assert pattern
above into a dedicated test-file format and runner, rather than
hand-rolling `pass`/`fail` counters yourself:

```bash
# mathutils.bats
#!/usr/bin/env bats

load 'mathutils.sh'

@test "add 2+3 equals 5" {
    result=$(add 2 3)
    [ "$result" -eq 5 ]
}

@test "is_even correctly identifies 4 as even" {
    is_even 4
}
```

```bash
bats mathutils.bats
```

`[Unverified — test before use]`

```
 ✓ add 2+3 equals 5
 ✓ is_even correctly identifies 4 as even

2 tests, 0 failures
```

**What `bats` adds over the hand-rolled pattern**: a standard,
recognizable test-file format other engineers (and CI systems) already
know how to run and interpret, built-in setup/teardown hooks for
per-test preparation and cleanup, and output formats CI systems can
parse directly (TAP format), rather than a bespoke `pass`/`fail` counter
every script author reinvents slightly differently. For a single quick
script, the Task 1 pattern is often enough; for a shared library of
functions multiple scripts depend on, `bats`'s standardization pays off.

### 1.3 Idempotency — safe to run more than once

A script is **idempotent** if running it multiple times produces the
same end result as running it once — no duplicate entries, no error on
the second run, no accumulating side effects. This matters specifically
for infrastructure and automation scripts because **"ran exactly once"
is not a guarantee you actually get** in practice: a deploy retried after
a flaky network blip, a cron job's previous run not fully confirmed
before the next trigger, a human re-running a setup script "just to be
sure" — all of these mean a script written assuming single-execution can
fail or corrupt state the second time, even though it worked perfectly
the first.

```bash
echo "listen_port=8080" >> /tmp/config.txt
```

Real output, running this exact line twice:

```
listen_port=8080
listen_port=8080
```

(2 lines in the file — a genuine bug, not a hypothetical one.)

### 1.4 Testing for idempotency, and the standard fix

**The test itself is simple**: run the script, capture the resulting
state; run it again; compare. Any difference between the two end states
(beyond something explicitly expected to change, like a timestamp) is an
idempotency bug.

```bash
grep -q "^listen_port=" /tmp/config.txt 2>/dev/null || echo "listen_port=8080" >> /tmp/config.txt
```

Real output, running this fixed version twice:

```
listen_port=8080
```

(1 line in the file, after both runs.)

**The fix pattern is almost always the same shape**: check whether the
desired end state already exists before taking the action that would
create or append it — `grep -q ... || echo ... >>`, `[ -f file ] ||
touch file`, `id user >/dev/null 2>&1 || useradd user`. The specific
check differs by what the script is actually doing, but the underlying
discipline — "check existing state before acting, not just act
unconditionally" — is the same pattern every time. This is also exactly
the Demo 26b lesson applied at script-authoring time instead of
infrastructure-config time: check first, then act, rather than acting
blindly and hoping it was the first time.

## Part 2 — Hands-On Lab

### Commands used in this demo

| Command | Purpose |
|---|---|
| `assert_eq "desc" "expected" "actual"` | Hand-rolled assertion helper (this demo's own pattern) |
| `bats testfile.bats` | Run a `bats` test file |
| `grep -q PATTERN file \|\| ACTION` | The standard "check before acting" idempotency guard |

### Task 1 — Write and run assert-based tests, including a real failure

```bash
cat > /tmp/mathutils.sh << 'LIB'
add() { echo $(( $1 + $2 )); }
is_even() { [ $(( $1 % 2 )) -eq 0 ]; }
LIB

cat > /tmp/test_mathutils.sh << 'TEST'
#!/bin/bash
source /tmp/mathutils.sh
pass=0; fail=0
assert_eq() {
    local desc="$1" expected="$2" actual="$3"
    if [ "$expected" == "$actual" ]; then
        echo "PASS: $desc"; pass=$((pass+1))
    else
        echo "FAIL: $desc (expected [$expected], got [$actual])"; fail=$((fail+1))
    fi
}
assert_eq "add 2+3" "5" "$(add 2 3)"
assert_eq "add -1+1" "0" "$(add -1 1)"
assert_eq "add bug check" "10" "$(add 4 5)"
echo "---"
echo "$pass passed, $fail failed"
[ "$fail" -eq 0 ]
TEST
bash /tmp/test_mathutils.sh
echo "exit code: $?"
```

Confirm the third assertion fails (it deliberately expects the wrong
value), and that the script's own exit code is non-zero because of it.

### Task 2 — Reproduce the non-idempotent bug

```bash
rm -f /tmp/config.txt
cat > /tmp/bad-setup.sh << 'SCRIPT'
#!/bin/bash
echo "listen_port=8080" >> /tmp/config.txt
SCRIPT
bash /tmp/bad-setup.sh
bash /tmp/bad-setup.sh
cat /tmp/config.txt
wc -l < /tmp/config.txt
```

Confirm the file has 2 lines after 2 runs — the bug, reproduced
deliberately.

### Task 3 — Fix it and confirm idempotency

```bash
rm -f /tmp/config2.txt
cat > /tmp/good-setup.sh << 'SCRIPT'
#!/bin/bash
grep -q "^listen_port=" /tmp/config2.txt 2>/dev/null || echo "listen_port=8080" >> /tmp/config2.txt
SCRIPT
bash /tmp/good-setup.sh
bash /tmp/good-setup.sh
cat /tmp/config2.txt
wc -l < /tmp/config2.txt
```

Confirm the file has exactly 1 line after 2 runs — the standard "check
before acting" fix pattern.

### Task 4 — Write an idempotency test as an actual test

```bash
cat > /tmp/test_idempotency.sh << 'TEST'
#!/bin/bash
rm -f /tmp/config2.txt
bash /tmp/good-setup.sh
first_run=$(cat /tmp/config2.txt)
bash /tmp/good-setup.sh
second_run=$(cat /tmp/config2.txt)
if [ "$first_run" == "$second_run" ]; then
    echo "PASS: script is idempotent"
else
    echo "FAIL: output changed between runs"
    echo "first:  $first_run"
    echo "second: $second_run"
fi
TEST
bash /tmp/test_idempotency.sh
```

This turns §1.4's manual "run twice and compare" check into an actual,
automatable test — the same `assert`-style pattern from Task 1, applied
to idempotency specifically rather than a single function's return value.

## What You Learned

- ✅ Wrote a lightweight, assert-based test for a bash function with no
  external framework
- ✅ Explained what `bats` adds over a hand-rolled assert pattern
- ✅ Explained what idempotency means and why it matters for
  infrastructure/automation scripts specifically
- ✅ Tested a script for idempotency by running it twice and comparing
  results
- ✅ Identified and fixed the standard idempotency bug — an unconditional
  append with no existence check

## Part 3 — Break-Fix Scenario

### Scenario — NovaTech Ops: the retried deploy that corrupted config

> **Incident report — NovaTech Ops, ticket #5360**
>
> "A deploy pipeline step timed out waiting for a response and the
> orchestrator automatically retried the same step. The retry succeeded,
> but the application came up with a corrupted config file — it looks
> like the same block of settings got appended to the file twice."

```bash
$ cat /opt/app/provision.sh
#!/bin/bash
cat >> /etc/app/app.conf << 'EOF'
max_connections=100
timeout=30
EOF
```

Tasks:
1. Reference §1.3 — the step "timed out," but the retry "succeeded." Is
   it actually certain the first attempt never ran the critical part of
   the script? Why does this matter for diagnosing what happened?
2. Reference §1.4 — what's the specific fix for this exact script,
   keeping the same here-doc content but making the operation safe to
   run more than once?
3. What test, per Task 4's pattern, would have caught this before it
   ever reached production?

<details>
<summary>Answers</summary>

A timeout on the *orchestrator's* side (waiting for a response) says
nothing certain about whether the script itself actually finished —
it's entirely possible the first attempt's `cat >> ... EOF` block
completed successfully and wrote the config, and only some later step
(or the orchestrator's own status check) was what timed out. This
matters because it means "the first attempt timed out" cannot be
assumed to mean "the first attempt did nothing" — exactly the trap that
caused this incident: the retry re-ran the same unconditional append,
assuming a clean slate that wasn't actually guaranteed. The specific fix:
wrap the append in an existence check for its content, the same pattern
as §1.4 — for example, checking whether `max_connections=` already
exists in the file before appending the block at all
(`grep -q "^max_connections=" /etc/app/app.conf || cat >> /etc/app/app.conf
<< 'EOF' ... EOF`), or more robustly, generating the whole config file
fresh each time with `>` instead of appending with `>>`, if the full
desired content is already known up front rather than built
incrementally. An idempotency test exactly like Task 4's — run
`provision.sh` once, capture `app.conf`'s content, run it again, compare
— would have caught this immediately in testing: the second run would
have produced a visibly different (doubled) file, long before any
orchestrator retry ever happened in production.

</details>

## Interview Prep

**Q1. Why does a test script's own exit code matter, not just the pass/fail text it prints?**
A non-zero exit code when any assertion fails is what lets the test be
wired into an automated pipeline (CI, a pre-deploy check) that stops on
failure — the same `set -e`-style "fail loudly, don't continue" pattern
from Demo 34, applied to testing specifically. Printed `PASS`/`FAIL`
text alone only helps a human reading it in real time; the exit code is
what a machine acts on.

**Q2. What does `bats` add over a hand-rolled `assert_eq` pattern, and when is the hand-rolled version still a reasonable choice?**
`bats` gives a standard, widely-recognized test-file format other
engineers and CI systems already know how to run and parse, plus
built-in setup/teardown hooks. For a single, small script tested once
during development, the lightweight hand-rolled pattern is often
sufficient; `bats`'s standardization earns its place for a shared
library of functions multiple scripts depend on.

**Q3. Why does idempotency matter specifically for infrastructure/automation scripts, more than for, say, a one-off report-generating script?**
Infrastructure and automation scripts are the ones most likely to
actually run more than once in practice — a retried deploy after a
timeout, a cron job whose previous run wasn't confirmed complete, a
human re-running setup "just to be sure." A script that assumes
single-execution and isn't idempotent can corrupt state or duplicate
side effects on exactly the kind of retry that's common in real
automated pipelines, not a rare edge case.

**Q4. What's the standard shape of the fix for a non-idempotent script, regardless of what the script specifically does?**
Check whether the desired end state already exists before taking the
action that would create it — `grep -q pattern file || append`, `[ -f
file ] || create it`, `id user || create the user`. The specific check
differs by what's being provisioned, but the underlying discipline —
verify current state before acting, rather than acting unconditionally —
is the same pattern every time.

---

## Key Takeaways

1. **A test script's own exit code, not just its printed output, is what
   makes it usable in an automated pipeline.** `[ "$fail" -eq 0 ]` as the
   final statement turns a human-readable report into something CI can
   act on.
2. **`bats` standardizes the same assert pattern into a recognized
   format with setup/teardown hooks and CI-parseable output** — valuable
   for a shared library, often unnecessary for a single small script.
3. **Idempotency means running a script more than once produces the same
   end result as running it once** — no duplicate entries, no
   accumulating side effects.
4. **"Ran exactly once" is not a guarantee automation actually gets** —
   retried deploys, uncertain cron overlaps, and manual re-runs are all
   realistic, which is why idempotency isn't an edge case to worry about
   later.
5. **The idempotency test itself is simple: run it, capture state, run
   it again, compare.** Any unexpected difference between the two end
   states is a bug, catchable before it ever reaches production.
6. **The standard idempotency fix is always "check existing state before
   acting"** — `grep -q ... || action`, applied to whatever the script is
   actually provisioning — the same discipline Demo 26b applied to
   infrastructure config, here applied to script authoring.

> **Demo scope:** Primary concept: testing bash scripts for correctness
> and safety under repeated execution. Supporting concepts: assert-based
> testing, `bats` as a forward reference, idempotency testing and the
> standard fix pattern.
> Estimated completion time: 32 minutes (reading + hands-on + verification).
> Checkpoints: 4 natural stopping points (after Task 1 through Task 4).

## Quick Commands Reference

| Command | Purpose |
|---|---|
| `assert_eq "desc" "expected" "actual"` | Hand-rolled assertion (compare and track pass/fail) |
| `[ "$fail" -eq 0 ]` (as the last line) | Make the test script's own exit code reflect pass/fail |
| `bats testfile.bats` | Run a `bats` test file |
| `grep -q PATTERN file \|\| ACTION` | Standard idempotency guard — act only if not already done |

## Appendix A — Anki Cards

**`34d-script-testing-anki.csv`:**

```
#deck:Linux Mastery::Module 9 - Shell Scripting::Demo 34d - Script Testing
#separator:Comma
#columns:Front,Back,Tags
"Why does a test script's own exit code matter, beyond just printing PASS/FAIL text?","A non-zero exit code when any assertion fails is what lets the test be wired into an automated pipeline (CI, pre-deploy check) that stops on failure — printed text alone only helps a human reading it live; the exit code is what a machine acts on.","demo34d,testing,exit-code,fundamentals"
"What does bats add over a hand-rolled assert_eq pattern?","A standard, widely-recognized test-file format other engineers/CI already know how to run and parse, built-in setup/teardown hooks, and CI-parseable output formats — instead of a bespoke pass/fail counter every script reinvents.","demo34d,bats,comparison"
"What does it mean for a script to be idempotent?","Running it multiple times produces the same end result as running it once — no duplicate entries, no error on a second run, no accumulating side effects.","demo34d,idempotency,fundamentals"
"Why does idempotency matter specifically for infrastructure/automation scripts?","'Ran exactly once' is not a guarantee actually available in practice — retried deploys after timeouts, uncertain cron overlaps, and manual re-runs are all realistic scenarios where a non-idempotent script can corrupt state or duplicate effects.","demo34d,idempotency,fundamentals"
"How do you actually test whether a script is idempotent?","Run it, capture the resulting state; run it again; compare the two end states. Any unexpected difference (beyond something explicitly expected to change) is an idempotency bug.","demo34d,idempotency,testing"
"echo 'listen_port=8080' >> config.txt is run twice. What happens, and why is this a real bug?","The line gets appended twice, producing two duplicate entries in the file — a genuine idempotency bug, since running the same provisioning step twice (e.g. via a retry) corrupts the config rather than leaving it unchanged.","demo34d,idempotency,common-mistake"
"What is the standard fix pattern for a non-idempotent append-style script?","Check whether the desired end state already exists before taking the action that would create it: grep -q PATTERN file || ACTION. The specific check varies by what's being provisioned, but the 'verify before acting' discipline is the same every time.","demo34d,idempotency,fix,best-practice"
"A deploy step times out but the system state afterward is uncertain. Why can't you assume the timed-out attempt definitely did nothing?","A timeout on the orchestrator's side (waiting for a response) says nothing certain about whether the underlying script actually finished its work — it may have completed successfully and only a later check or the response itself timed out.","demo34d,idempotency,break-fix"
```

## Appendix B — Quiz

**`34d-script-testing-quiz.md`:**

````markdown
# Quiz — Demo 34d: Script Testing

> One correct answer per question unless stated otherwise.
> Target: 80% or above — this is the final quiz in Module 9.

**Q1. Why does a test script's own exit code matter, not just the PASS/FAIL text it prints?**

- A) Exit codes are purely cosmetic and don't affect anything
- B) A non-zero exit code on failure is what lets the test be wired into an automated pipeline that stops on failure — a machine acts on the exit code, not printed text
- C) Bash scripts cannot print text and also have an exit code
- D) Exit codes only matter for compiled programs, not scripts

<details>
<summary>Answer</summary>

**B** — This is the practical reason to make `[ "$fail" -eq 0 ]` the
final statement of a test script.

</details>

---

**Q2. What does `bats` add over a hand-rolled `assert_eq` testing pattern?**

- A) `bats` makes scripts run faster
- B) A standard, recognized test-file format, built-in setup/teardown hooks, and CI-parseable output — instead of a bespoke counter every script author reinvents
- C) `bats` is required for any bash script to be considered valid
- D) Nothing — they are functionally identical with no real difference

<details>
<summary>Answer</summary>

**B** — Standardization and tooling integration are `bats`'s actual
value over a one-off hand-rolled pattern.

</details>

---

**Q3. What does it mean for a script to be idempotent?**

- A) It runs faster on subsequent executions
- B) Running it multiple times produces the same end result as running it once — no duplicate entries or accumulating side effects
- C) It can only be run exactly one time, ever
- D) It automatically detects and fixes its own bugs on a second run

<details>
<summary>Answer</summary>

**B** — This is the precise, correct definition — repeated execution is
safe and produces an unchanged end state.

</details>

---

**Q4. `echo "listen_port=8080" >> config.txt` is run twice. What happens?**

- A) Nothing — `>>` automatically prevents duplicate lines
- B) The line is appended twice, producing two duplicate entries — a genuine idempotency bug
- C) The second run fails with an error since the file already exists
- D) `bash` automatically deduplicates appended content

<details>
<summary>Answer</summary>

**B** — Plain `>>` has no awareness of existing content; it just adds
more, which is exactly the bug idempotency testing catches.

</details>

---

**Q5. What's the standard fix pattern for a non-idempotent append-style provisioning script?**

- A) Replace `>>` with `>` in every case, regardless of context
- B) Check whether the desired end state already exists before taking the action that would create it, e.g. `grep -q PATTERN file || ACTION`
- C) Add `sleep 5` before the append to avoid race conditions
- D) Run the script inside a loop that retries until it succeeds

<details>
<summary>Answer</summary>

**B** — "Verify existing state before acting" is the consistent shape of
the fix, regardless of what's specifically being provisioned.

</details>

---

**Q6. A deploy step times out and is automatically retried by the orchestrator. Why can't you assume the first attempt definitely did nothing?**

- A) Timeouts always mean the script fully completed successfully
- B) A timeout on the orchestrator's side (waiting for a response) says nothing certain about whether the underlying script actually finished its work — it may have completed and only the status check timed out
- C) Retries are only ever triggered after confirmed total failure
- D) This is never actually a real concern in practice

<details>
<summary>Answer</summary>

**B** — This uncertainty is exactly what makes idempotency essential for
anything that might be retried — you can't assume a clean slate.

</details>

---

**Q7. How do you actually test whether a script is idempotent?**

- A) Read the script's source code carefully and reason about it without running it
- B) Run it, capture the resulting state; run it again; compare the two end states — any unexpected difference is a bug
- C) Idempotency cannot be tested, only guaranteed by careful coding
- D) Run it exactly once and check that it doesn't produce any errors

<details>
<summary>Answer</summary>

**B** — This run-twice-and-compare approach is a concrete, automatable
test, not just a code-review judgment call.

</details>

---

Score guide:

| Score | Action |
|---|---|
| 7/7 | Import Anki cards — Module 9 complete |
| 6/7 | Review the wrong answer, then proceed |
| 5/7 | Re-read the relevant section, retry those questions |
| Below 5/7 | Re-read the full demo and redo the walkthrough before proceeding |
````