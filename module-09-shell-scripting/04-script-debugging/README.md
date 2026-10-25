# Demo 34c — Script Debugging

## Overview

Every demo in this phase has built scripts correctly from the start.
This demo covers what to do once one still doesn't behave as expected —
`set -x` and a customized `PS4` for execution tracing, `trap DEBUG` for
tracing beyond what `set -x` shows on its own, and structured logging so
a problem that happened at 3 AM can be diagnosed without needing to
reproduce it live. `bashdb` is covered conceptually as the next step up
when tracing alone isn't enough.

## Demo Objectives

1. Use `set -x` to trace command execution, and customize `PS4` to add
   line numbers and function context to that trace
2. Use `trap DEBUG` for custom, per-command tracing beyond `set -x`'s
   default output
3. Build a structured, leveled, timestamped logging function for
   post-mortem debugging without needing to reproduce the issue live
4. Explain what `bashdb` adds over `set -x`, as a forward reference for
   when tracing alone isn't enough
5. Diagnose a real bug using a `set -x` trace alone, by reading exactly
   where actual behavior diverges from expectation

## Directory Structure

```
34c-script-debugging/
├── README.md                          # this file
├── 34c-script-debugging-anki.csv      # Anki flashcard deck
└── 34c-script-debugging-quiz.md       # standalone quiz
```

## Environments

| Task | Ubuntu 24.04 | Rocky Linux 9.8 | EC2 Amazon Linux |
|---|---|---|---|
| `set -x` / `PS4` / `trap DEBUG` | ✅ (built into bash) | ✅ (built into bash) | Not required |
| `bashdb` | `sudo apt install bashdb` | `sudo dnf install bashdb` (may need EPEL) | Not required |

> **Verification note:** §1.1–1.3 (`set -x`, custom `PS4`, `trap DEBUG`,
> structured logging) were all run live in the authoring sandbox — every
> trace line and log line shown is real output. `bashdb` isn't installed
> here (no network) and §1.4 is doc-sourced; recommend confirming its
> exact command set on a real machine.

## Recall Check — 34b-signal-trap-handling

Answer these from memory before continuing — no looking back at Demo 34b.

**Q1.** Why is `/tmp/myapp-$$` a genuine security risk as a temp
filename, and what does `mktemp` do differently?

**Q2.** Why doesn't an `flock`-based lock need separate stale-lock
cleanup logic, unlike a hand-rolled PID file?

**Q3.** A cron job runs every 5 minutes but occasionally takes 11
minutes. What actually happens without any locking?

<details>
<summary>Answers</summary>

**A1.** A predictable path lets another process create something there
first (e.g. a symlink) before the script does, which the script then
writes through unknowingly. `mktemp` creates the file atomically with a
confirmed-unique name and `0600` permissions from the moment of creation.

**A2.** The lock releases automatically the instant the holding
process's file descriptor closes, for any reason including a crash — no
separate "is that PID still alive" detection needed.

**A3.** The next one or two scheduled triggers fire while the previous
run is still active, creating overlapping instances racing against the
same shared state.

</details>

## Part 1 — Concepts

### 1.1 set -x — tracing execution, and why the default output isn't enough

```bash
bash -x script.sh
```

Real output, for a trivial script:

```
+ x=5
+ y=8
+ echo 'result: 8'
result: 8
```

Each line prefixed with `+` is a command bash actually executed, shown
*after* variable expansion — note `y=8`, not `y=$((x + 3))`: `set -x`
shows what the shell actually did, not the literal source line, which is
exactly why it's more useful than reading the script than for catching a
substitution that evaluated differently than expected.

**The default trace gets hard to read fast in anything beyond a trivial
script** — once a script has functions, loops, or runs on a shared
system where several scripts might be traced at once, a bare `+` tells
you almost nothing about *where* in the script, or *which* script, a line
came from.

### 1.2 Customizing PS4 — making the trace actually useful

```bash
export PS4='+ [${BASH_SOURCE}:${LINENO}] '
set -x
```

Real output, for a script with a function call:

```
+ [/tmp/dbg2.sh:8] greet wadmin
+ [/tmp/dbg2.sh:5] local name=wadmin
+ [/tmp/dbg2.sh:6] echo 'hello wadmin'
```

`PS4` is the prompt string `set -x` uses for every traced line — by
default just `+ `. `$BASH_SOURCE` and `$LINENO` are built-in variables
bash updates automatically as execution proceeds; putting them in `PS4`
means every trace line tells you exactly which file and line number
produced it, including correctly following execution *into* a function
body and back out — in the output above, line 8 (the call site) is
followed immediately by lines 5 and 6 (inside `greet`'s own body), which
is precisely the control flow that happened.

**This is the single highest-value change to make before reaching for
`set -x` on anything non-trivial** — the default `+`-only trace on a
multi-function script tells you *that* something ran, this version tells
you *exactly where*.

### 1.3 trap DEBUG — custom tracing beyond what set -x shows

```bash
trap 'echo "[TRACE line $LINENO]: $BASH_COMMAND"' DEBUG
```

Real output:

```
[TRACE line 3]: a=1
[TRACE line 4]: b=2
[TRACE line 5]: echo "sum: $((a+b))"
```

A `DEBUG` trap runs *before* every simple command — unlike `set -x`,
which only prints, `trap DEBUG` lets you run arbitrary code at that
point: log to a specific file instead of stderr, check a condition and
conditionally stop (`trap '[[ $x -gt 100 ]] && { echo "x exceeded limit
at line $LINENO"; exit 1; }' DEBUG`), or capture state that `set -x`'s
fixed output format can't express. **`$BASH_COMMAND` here shows the
literal, unexpanded source line** — the opposite of `set -x`'s
post-expansion output — which is genuinely useful when you want to know
*what the script intended* at that point, not just what it evaluated to.

### 1.4 Structured logging — debugging without reproducing live

```bash
log() {
    local level="$1"; shift
    printf "%s [%s] %s\n" "$(date "+%Y-%m-%d %H:%M:%S")" "$level" "$*" >> /tmp/app.log
}
log INFO "deploy started"
log WARN "retrying step 2"
log ERROR "step 3 failed: connection refused"
```

Real output:

```
2026-10-02 22:20:06 [INFO] deploy started
2026-10-02 22:20:06 [WARN] retrying step 2
2026-10-02 22:20:06 [ERROR] step 3 failed: connection refused
```

**This is the debugging tool for problems you can't reproduce on
demand** — `set -x` and `trap DEBUG` both require the script to actually
be running while you're watching; a timestamped, leveled log that's
*always on* at an appropriate verbosity means a 3 AM failure already has
a trail by the time anyone looks at it the next morning. A level prefix
(`INFO`/`WARN`/`ERROR`) lets a post-mortem `grep ERROR app.log` jump
straight to what actually went wrong, rather than reading every line in
order.

### 1.5 bashdb — stepping through, not just tracing

`set -x` and `trap DEBUG` both show you execution as it streams past.
`bashdb` is bash's interactive step-debugger: set a breakpoint, run until
it's hit, then step one command at a time, inspect any variable's current
value on demand, and continue — a fundamentally different interaction
model than reading a trace after the fact.

```bash
bashdb script.sh
```

`[Unverified — test before use]`

```
(bashdb) break 12
(bashdb) run
(bashdb) print $some_var
(bashdb) next
(bashdb) continue
```

**Reach for `bashdb` when a trace alone isn't enough to understand *why*
— when you need to inspect a variable's value at an exact moment, not
just see that a line executed.** For most day-to-day debugging, a
well-placed `PS4`-customized `set -x` or structured logging answers the
question faster; `bashdb`'s interactive stepping earns its overhead on
genuinely confusing control flow.

## Part 2 — Hands-On Lab

### Commands used in this demo

| Command | Purpose |
|---|---|
| `bash -x script.sh` | Trace execution |
| `export PS4='...'` | Customize the trace prefix (file, line number, etc.) |
| `trap 'cmd' DEBUG` | Run custom code before every simple command |
| `$BASH_COMMAND` | The literal, unexpanded source line, inside a DEBUG trap |
| `$LINENO` / `$BASH_SOURCE` | Current line number / current script file |

### Task 1 — Basic set -x trace

```bash
cat > /tmp/dbg1.sh << 'SCRIPT'
#!/bin/bash
x=5
y=$((x + 3))
echo "result: $y"
SCRIPT
bash -x /tmp/dbg1.sh
```

Confirm the trace shows `y=8` (the evaluated result), not the literal
source expression.

### Task 2 — Customized PS4 through a function call

```bash
cat > /tmp/dbg2.sh << 'SCRIPT'
#!/bin/bash
export PS4='+ [${BASH_SOURCE}:${LINENO}] '
set -x
greet() {
    local name="$1"
    echo "hello $name"
}
greet "wadmin"
SCRIPT
bash /tmp/dbg2.sh
```

Confirm the trace shows the call-site line number, then the line numbers
*inside* `greet`'s own body, in the actual order of execution.

### Task 3 — trap DEBUG with the literal source line

```bash
cat > /tmp/dbg3.sh << 'SCRIPT'
#!/bin/bash
trap 'echo "[TRACE line $LINENO]: $BASH_COMMAND"' DEBUG
a=1
b=2
echo "sum: $((a+b))"
SCRIPT
bash /tmp/dbg3.sh
```

Confirm the last trace line shows the literal `echo "sum:
$((a+b))"` — unexpanded — contrasting directly against Task 1's
post-expansion `set -x` output.

### Task 4 — A structured logging function

```bash
rm -f /tmp/app-postmortem.log
cat > /tmp/dbg4.sh << 'SCRIPT'
#!/bin/bash
log() {
    local level="$1"; shift
    printf "%s [%s] %s\n" "$(date "+%Y-%m-%d %H:%M:%S")" "$level" "$*" >> /tmp/app-postmortem.log
}
log INFO "deploy started"
log WARN "retrying step 2"
log ERROR "step 3 failed: connection refused"
SCRIPT
bash /tmp/dbg4.sh
cat /tmp/app-postmortem.log
grep ERROR /tmp/app-postmortem.log
```

### Task 5 — Diagnose a bug from a trace alone

```bash
cat > /tmp/buggy.sh << 'SCRIPT'
#!/bin/bash
export PS4='+ [${LINENO}] '
set -x
total=0
items=("10" "20" "3O")
for i in "${items[@]}"; do
    total=$((total + i))
done
echo "Total: $total"
SCRIPT
bash /tmp/buggy.sh 2>&1
```

Read the trace output and the error it produces. Identify exactly which
array element and which line caused the failure, without being told in
advance.

<details>
<summary>Answer</summary>

The third element is `"3O"` (the letter O, not the digit zero) — a typo.
The trace shows the loop reaching `total=$((total + i))` with `i=3O`,
and `$((...))` fails with a syntax error (`3O: value too great for
base`, or similar — arithmetic context requires a genuinely numeric
value). The trace pinpoints the exact iteration where things went wrong,
rather than leaving you to guess which of the three array elements was
the problem.

</details>

## What You Learned

- ✅ Used `set -x` to trace execution and customized `PS4` for line
  number and function context
- ✅ Used `trap DEBUG` for custom tracing with the literal, unexpanded
  source line
- ✅ Built a structured, leveled, timestamped logging function for
  post-mortem debugging
- ✅ Explained what `bashdb` adds over tracing alone
- ✅ Diagnosed a real bug from a `set -x` trace, pinpointing exactly
  where behavior diverged from expectation

## Part 3 — Break-Fix Scenario

### Scenario — NovaTech Ops: the failure nobody can reproduce

> **Incident report — NovaTech Ops, ticket #5330**
>
> "A nightly batch script failed once, three nights ago. It's run
> successfully every night since. The on-call engineer wants to add
> `set -x` to catch it next time, but can't predict which night it will
> fail again — and the script produces no output at all when it
> succeeds, so there's nothing to compare against from the good runs
> either."

Tasks:
1. Reference §1.3/§1.4 — is `set -x`, added now and left in place every
   night, actually a reasonable fix for "catch it next time"? What's a
   real downside to running a full trace permanently on a production
   script?
2. What approach would let the team investigate *this specific* failure
   without needing it to recur, and without a verbose trace cluttering
   every single successful run?
3. Given the script currently produces zero output on success, what's
   missing structurally, independent of this specific incident?

<details>
<summary>Answer</summary>

Leaving `set -x` permanently on is a real option but has a real cost: a
full execution trace on every run, every night, forever, produces a
large volume of low-value output on all the nights nothing goes wrong,
which itself becomes a cost to store and a haystack to search through
when something eventually does fail. A better approach, per §1.4: add
structured logging (leveled, timestamped) that's always on regardless of
whether anything fails — unlike a `set -x` trace, this is cheap enough
to run every night without flooding anything, and when the next failure
happens, the log already has a trail leading up to it, without needing
the failure to be actively watched or reproduced live. What's missing
structurally, independent of this specific incident: the script has no
logging at all on success, which is itself the deeper problem — a script
that produces nothing when it works and nothing useful when it doesn't
leaves a post-mortem with no evidence either way. Adding `log INFO`
lines at each major step, always, is what turns the *next* failure into
something diagnosable instead of another unreproducible mystery.

</details>

## Interview Prep

**Q1. Why is customizing `PS4` considered the highest-value change to make before using `set -x` on a non-trivial script?**
The default `+`-only trace tells you that a line executed but not where —
on a script with functions or loops, that's close to useless for
locating a specific problem. Including `$BASH_SOURCE` and `$LINENO` in
`PS4` means every trace line identifies exactly which file and line
produced it, correctly following execution into and out of function
calls.

**Q2. What's the difference between what `set -x` shows and what `$BASH_COMMAND` inside a `trap DEBUG` shows for the same line?**
`set -x` shows the command *after* variable expansion and arithmetic
evaluation — what the shell actually did. `$BASH_COMMAND` shows the
*literal, unexpanded* source line — what the script's author actually
wrote. Both are useful for different questions: "what value did this
evaluate to" versus "what did the script intend at this point."

**Q3. A production script fails intermittently and can't be reliably reproduced on demand. Why is `set -x`, left permanently enabled, not necessarily the right long-term fix?**
A full execution trace on every single run produces a large volume of
output on every night nothing goes wrong, which becomes its own storage
and searchability cost. Structured, leveled logging that's always on at
a lighter verbosity gives a usable trail leading up to the next failure
without flooding every successful run — the better fit for a problem you
can't reproduce on demand.

**Q4. When does `bashdb`'s interactive stepping actually earn its overhead over a `PS4`-customized `set -x` trace?**
When the question isn't just "did this line run" but "what was this
specific variable's value at this exact moment, and what happens if I
step one command at a time from here" — genuinely confusing control
flow where reading a trace after the fact isn't enough and you need to
interactively inspect state as execution happens, not reconstruct it
from output.

---

## Key Takeaways

1. **`set -x` shows commands after expansion; a `DEBUG` trap's
   `$BASH_COMMAND` shows the literal source line.** They answer different
   questions — what actually happened versus what was written — and both
   are useful.
2. **Customizing `PS4` with `$BASH_SOURCE`/`$LINENO` is the highest-value
   change before tracing anything beyond a trivial script** — it turns a
   bare `+` into "exactly which file and line," correctly following
   execution through function calls.
3. **`trap DEBUG` runs arbitrary code before every command, not just a
   print statement** — conditional breakpoint-like logic, custom log
   destinations, or state capture `set -x`'s fixed format can't express.
4. **Structured, leveled, timestamped logging is the tool for problems
   you can't reproduce on demand.** `set -x`/`trap DEBUG` need the script
   running while you watch; an always-on log already has a trail by the
   time anyone looks.
5. **Leaving a full trace permanently enabled has a real cost** — volume
   on every successful run — which is exactly why structured logging,
   not `set -x`, is the right fix for an intermittent, unreproducible
   failure.
6. **`bashdb`'s interactive stepping earns its overhead on genuinely
   confusing control flow** — inspecting a variable's exact value at an
   exact moment — not as the default first tool for routine debugging.

> **Demo scope:** Primary concept: debugging a bash script, in real time
> and after the fact. Supporting concepts: `set -x`/`PS4` customization,
> `trap DEBUG`, structured logging, `bashdb` as a forward reference.
> Estimated completion time: 32 minutes (reading + hands-on + verification).
> Checkpoints: 5 natural stopping points (after Task 1 through Task 5).

## Quick Commands Reference

| Command | Purpose |
|---|---|
| `bash -x script.sh` | Trace execution (post-expansion) |
| `export PS4='+ [${BASH_SOURCE}:${LINENO}] '` | Add file/line context to the trace |
| `trap 'cmd' DEBUG` | Run custom code before every simple command |
| `$BASH_COMMAND` | Literal, unexpanded source line (inside a DEBUG trap) |
| `bashdb script.sh` | Interactive step-debugger |

## Appendix A — Anki Cards

**`34c-script-debugging-anki.csv`:**

```
#deck:Linux Mastery::Module 9 - Shell Scripting::Demo 34c - Script Debugging
#separator:Comma
#columns:Front,Back,Tags
"Does set -x show a line's literal source text or its value after expansion?","After expansion/evaluation — e.g. y=$((x+3)) shows as y=8 in the trace, not the literal arithmetic expression. It shows what the shell actually did.","demo34c,set-x,fundamentals"
"Why is customizing PS4 with $BASH_SOURCE/$LINENO the highest-value change before using set -x on a non-trivial script?","The default trace is just a bare '+' per line, telling you nothing about WHERE. Adding file/line context to PS4 means every trace line identifies exactly which file and line produced it, including correctly through function calls.","demo34c,ps4,best-practice"
"What does $BASH_COMMAND show inside a trap DEBUG handler, and how does that differ from set -x's output for the same line?","The literal, UNEXPANDED source line — the opposite of set -x, which shows the post-expansion result. Useful for knowing what the script's author intended, not just what it evaluated to.","demo34c,trap-debug,bash-command"
"What can a trap DEBUG handler do that plain set -x cannot?","Run arbitrary code before every command — conditional logic (e.g. stop only if a variable exceeds a threshold), custom log destinations, or state capture — not just print a fixed-format trace line.","demo34c,trap-debug,fundamentals"
"Why is structured, leveled logging the right tool for a bug that can't be reliably reproduced on demand?","set -x and trap DEBUG both require the script to be actively running while you watch. An always-on, timestamped, leveled log already has a trail leading up to the next failure by the time anyone looks, without needing the failure to recur while being watched.","demo34c,logging,best-practice"
"What's a real downside to leaving set -x permanently enabled on a production script that runs regularly?","It produces a large volume of trace output on every single run, including every night nothing goes wrong — a real storage and searchability cost, not just noise.","demo34c,set-x,common-mistake"
"When does bashdb's interactive stepping actually earn its overhead over a PS4-customized set -x trace?","When you need to inspect a specific variable's exact value at an exact moment and step through genuinely confusing control flow interactively — not for routine debugging, where a trace or structured log usually answers the question faster.","demo34c,bashdb,fundamentals"
```

## Appendix B — Quiz

**`34c-script-debugging-quiz.md`:**

````markdown
# Quiz — Demo 34c: Script Debugging

> One correct answer per question unless stated otherwise.
> Target: 80% or above before moving to Demo 34d.

**Q1. Does `set -x`'s trace output show a command's literal source text or its value after shell expansion?**

- A) Literal source text, unexpanded
- B) After expansion/evaluation — e.g. `y=$((x+3))` appears as `y=8`
- C) Both, on separate lines
- D) Neither — `set -x` only reports exit codes

<details>
<summary>Answer</summary>

**B** — `set -x` shows what the shell actually did, which is why it
catches a substitution evaluating to something unexpected.

</details>

---

**Q2. Why is customizing `PS4` considered the highest-value change before using `set -x` on anything beyond a trivial script?**

- A) It makes the script execute faster
- B) The default trace is just a bare `+` per line with no location information; adding `$BASH_SOURCE`/`$LINENO` identifies exactly which file and line produced each trace line
- C) `set -x` doesn't work at all without a custom `PS4`
- D) `PS4` is required syntax, not optional

<details>
<summary>Answer</summary>

**B** — This is the real, practical value — locating exactly where in a
multi-function script a trace line came from.

</details>

---

**Q3. What does `$BASH_COMMAND` show inside a `trap DEBUG` handler?**

- A) The post-expansion, evaluated version of the command — identical to `set -x`'s output
- B) The literal, unexpanded source line, as written in the script
- C) The exit code of the previous command
- D) The full command history of the entire session

<details>
<summary>Answer</summary>

**B** — This is the opposite of `set -x`'s post-expansion output —
useful for knowing what the script's author actually wrote.

</details>

---

**Q4. What can a `trap DEBUG` handler do that plain `set -x` cannot?**

- A) Trace faster
- B) Run arbitrary custom code before every command — conditional logic, custom logging destinations, or state capture beyond a fixed trace format
- C) `trap DEBUG` only works in interactive shells, not scripts
- D) Nothing — they are functionally identical

<details>
<summary>Answer</summary>

**B** — `set -x` only prints in a fixed format; `trap DEBUG` runs
whatever code you give it.

</details>

---

**Q5. A bug can't be reliably reproduced on demand. Why is structured, leveled logging a better fit than relying on `set -x`?**

- A) `set -x` doesn't work on intermittent bugs at all
- B) `set -x`/`trap DEBUG` both require the script to be actively running while watched; an always-on log already has a trail by the time anyone investigates, without needing the failure to recur under observation
- C) Structured logging automatically fixes the bug once detected
- D) There is no real difference between the two approaches

<details>
<summary>Answer</summary>

**B** — This is the actual distinguishing factor — being present when
the failure happens versus having already recorded a trail beforehand.

</details>

---

**Q6. What's a real downside to leaving `set -x` permanently enabled on a production script that runs on a regular schedule?**

- A) It causes the script to fail more often
- B) It produces a large volume of trace output on every single run, including every time nothing goes wrong — a real storage and searchability cost
- C) `set -x` cannot be combined with `set -e`
- D) There is no real downside; this is always the correct approach

<details>
<summary>Answer</summary>

**B** — Volume on every successful run is the practical cost, which is
exactly why structured logging is the better long-term fit for this
situation.

</details>

---

**Q7. When does `bashdb`'s interactive stepping genuinely earn its overhead over a `PS4`-customized `set -x` trace?**

- A) For every debugging session, as the universal first choice
- B) When you need to inspect a specific variable's exact value at an exact moment and step through genuinely confusing control flow interactively
- C) Only when the script has no functions at all
- D) `bashdb` and `set -x` are interchangeable with no difference in use case

<details>
<summary>Answer</summary>

**B** — For routine debugging, a trace or structured log usually answers
the question faster; `bashdb` earns its place on genuinely confusing
cases.

</details>

---

Score guide:

| Score | Action |
|---|---|
| 7/7 | Import Anki cards, move to Demo 34d |
| 6/7 | Review the wrong answer, then proceed |
| 5/7 | Re-read the relevant section, retry those questions |
| Below 5/7 | Re-read the full demo and redo the walkthrough before proceeding |
````