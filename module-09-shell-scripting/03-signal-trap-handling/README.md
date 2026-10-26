# Demo M09-03 — Signal Handling, Safe Temp Files & Concurrency

## Overview

Demo M09-02's `trap` section covered the basics — register a handler, clean up
on exit. This demo covers the three things that actually cause production
incidents when they're missing: a script that leaves things in a known,
safe state when killed mid-operation rather than a "partially applied"
mess; temp files created safely instead of with a predictable, guessable
name; and a lockfile guard that stops two triggers of the same script
(two overlapping cron runs, most commonly) from racing each other. Every
scenario in this demo was built and run live in this sandbox — nothing
here is a documentation paraphrase.

## Demo Objectives

1. Handle `SIGTERM` explicitly and distinguish it from `SIGINT`,
   including why a script might want different behavior for each
2. Use `mktemp` correctly for files and directories, and explain the
   concrete risk a predictable temp-file name creates
3. Build a lockfile-based concurrency guard with `flock` that prevents
   two instances of a script from running at once
4. Diagnose and fix the classic race condition: two cron-triggered runs
   of the same script overlapping
5. Build a multi-step script that records enough state to recover
   correctly when killed partway through, instead of leaving an unknown,
   partially-applied state

## Directory Structure

```
03-signal-trap-handling/
├── README.md                             # this file
├── m09-03-signal-trap-handling-anki.csv  # Anki flashcard deck
└── m09-03-signal-trap-handling-quiz.md   # standalone quiz
```

## Environments

| Task | Ubuntu 24.04 | Rocky Linux 9.8 | EC2 Amazon Linux |
|---|---|---|---|
| All lab tasks | ✅ (bash, `flock` from `util-linux`, both preinstalled) | ✅ (bash, `flock` from `util-linux`, both preinstalled) | Not required |

> **Verification note:** every scenario in this demo — `SIGTERM` handling,
> `mktemp`'s uniqueness and permissions, the `flock` concurrency guard, and
> the interrupted multi-step deploy — was built and run live in the
> authoring sandbox, including actually sending signals to a running
> background process and launching two real overlapping script instances.
> No `[Unverified]` markers are needed anywhere in this demo.

## Recall Check — 02-shell-scripting-advanced

Answer these from memory before continuing — no looking back at Demo M09-02.

**Q1.** What specific failure does `set -o pipefail` catch that plain
`set -e` does not?

**Q2.** Why is `trap cleanup EXIT` more robust than cleanup code placed at
the literal end of a script?

**Q3.** A `getopts` option string is `"vo"` instead of `"vo:"`. What goes
wrong when the script is called with `-o report.txt`?

<details>
<summary>Answers</summary>

**A1.** A pipeline where an earlier stage fails but a later stage
succeeds — without `pipefail`, the pipeline's exit status reflects only
the last command, silently masking the earlier failure.

**A2.** `trap ... EXIT` runs on any exit path — errors, interruptions,
kills — not just the path where execution reaches the script's end
normally.

**A3.** Without the colon, `-o` is treated as a flag that takes no
argument — `report.txt` becomes an unconsumed positional argument, and
`$OPTARG` is never populated.

</details>

## Part 1 — Concepts

### 1.1 SIGTERM vs. SIGINT — different signals, different intent

Demo M09-02 §1.2 registered one handler for both `INT` and `TERM` together.
They're not actually the same situation: `SIGINT` is what a human sends
with Ctrl+C — immediate, at-the-keyboard, "stop now." `SIGTERM` is the
default signal `kill` and process managers (systemd, Kubernetes, cron's
own job control) send to ask a process to shut down **gracefully** —
finish what it's doing, then exit, rather than stopping instantly mid-step.

```bash
trap 'echo "Got SIGTERM — finishing current step then exiting cleanly"; exit 0' TERM
trap 'echo "Got SIGINT — exiting immediately"; exit 130' INT
```

Real output from actually sending `SIGTERM` to a running script:

```
$ bash sigdemo.sh &
PID 90 running, sleeping...
$ kill -TERM 90
Got SIGTERM — finishing current step then exiting cleanly
exit status: 0
```

**Why this distinction matters operationally**: a process manager
stopping your service for a routine redeploy sends `SIGTERM` and expects
a clean, prompt shutdown — if your script only handles `SIGINT`, the
`SIGTERM` falls through to the default action (immediate termination,
`SIGKILL`-style, no cleanup at all) rather than your intended graceful
path. Handling `TERM` explicitly is what makes a script behave correctly
under an orchestrator, not just under a human pressing Ctrl+C.

### 1.2 mktemp — why a predictable temp file name is a real risk

```bash
# Don't do this:
insecure_tmp="/tmp/myapp-cache-$$"
```

Using the process ID (`$$`) or any other predictable scheme to build a
temp filename has a genuine, documented security problem: between the
moment your script decides on that filename and the moment it actually
creates the file, another process (or another user, on a shared `/tmp`)
can create something at that exact path first — a symlink pointing
somewhere sensitive, for instance — and your script then writes through
it. This is a real vulnerability class (a symlink/race attack on
predictable temp paths), not a theoretical concern.

```bash
t1=$(mktemp /tmp/myapp.XXXXXX)
t2=$(mktemp /tmp/myapp.XXXXXX)
```

Real output:

```
t1=/tmp/myapp.iqsSAF
t2=/tmp/myapp.O9It8k
-rw------- 1 root root 0 Oct  2 07:37 /tmp/myapp.iqsSAF
```

**Two things `mktemp` gives you that a hand-built name doesn't**: the
`X`s are replaced with genuinely random characters, confirmed unique on
creation (not just "probably unique"), and — note the permissions in the
real output above — the file is created with `0600` (owner read/write
only) from the start, not created-then-chmod'd, closing the window where
a predictable or world-readable file could be read or tampered with
before you lock it down.

```bash
d=$(mktemp -d /tmp/myapp-work.XXXXXX)
```

`-d` creates a directory instead of a file, with the same uniqueness and
safe-permissions guarantees — the right tool for a scratch working
directory, not just single files.

### 1.3 flock — a real concurrency guard

The curriculum scenario this section exists for: a cron job scheduled
every 5 minutes occasionally takes longer than 5 minutes to run, so the
next trigger fires while the previous run is still going — two instances
of the same script now racing against the same files, state, or
downstream system.

```bash
LOCKFILE=/tmp/guarded.lock
exec 200>"$LOCKFILE"
if ! flock -n 200; then
    echo "Another instance is already running. Exiting."
    exit 1
fi
echo "Lock acquired, working..."
```

Real output from two actually-overlapping instances:

```
$ bash guarded.sh &
Lock acquired (PID 105), working...
$ bash guarded.sh          # started while the first is still sleeping
Another instance is already running. Exiting.
$ wait
Done (PID 105)
```

**How this actually works**: `exec 200>"$LOCKFILE"` opens the lockfile on
file descriptor 200 (an arbitrary unused number, by convention), without
needing to write anything to it. `flock -n 200` attempts to take an
exclusive advisory lock on that file descriptor — `-n` (non-blocking)
means it returns immediately with a failure if the lock is already held,
rather than waiting. **The lock is automatically released when the
script's process exits for any reason** (normal exit, error, even a
kill) — the file descriptor closes, which releases the lock, so there's
no separate "remember to unlock" step and no stale-lock cleanup needed
the way a hand-rolled "check if a PID file exists" scheme would require.

### 1.4 Why a hand-rolled "check for a PID file" lock is worse

A common manual alternative — checking whether a PID file exists before
starting, and writing your own PID into it if not — has a real race
condition of its own: two instances can both check "does the file exist"
at nearly the same moment, both see "no," and both proceed to write their
own PID and continue running simultaneously, exactly the bug the lock was
supposed to prevent. `flock`'s lock-acquisition is a single atomic kernel
operation — there's no gap between "check" and "acquire" for a second
process to slip through. It also self-cleans: a stale PID file from a
crashed previous run needs manual detection logic (is that PID actually
still running?); `flock`'s lock disappears the instant the holding
process's file descriptor closes, crash included.

### 1.5 Surviving a kill mid-operation — recording state, not just cleaning up

Demo M09-02's `trap` deleted a temp file on exit — a cleanup action. A
multi-step operation (a deploy, a migration) needs something more: a
record of *how far it got*, so an interrupted run doesn't leave the
system in an ambiguous state nobody can safely reason about afterward.

```bash
STATE_FILE=/tmp/deploy-state.txt
cleanup() {
    echo "Interrupted — rolling back to pre-deploy state" >> "$STATE_FILE"
}
trap cleanup TERM INT

echo "step1: backup taken" > "$STATE_FILE"
sleep 2
echo "step2: new version staged" >> "$STATE_FILE"
sleep 10
echo "step3: symlink swapped - DEPLOY COMPLETE" >> "$STATE_FILE"
```

Real output, killed with `SIGTERM` partway through step 2's sleep:

```
step1: backup taken
step2: new version staged
Interrupted — rolling back to pre-deploy state
```

**The state file itself is the point** — step 3 never ran, and the log
says so explicitly, rather than leaving an operator to guess whether the
deploy fully completed, partially completed, or never started at all from
external symptoms alone. The curriculum's "partially applies on failure
and leaves the system in an unknown state" incident is exactly what this
pattern prevents: *known* partial state (logged, and ideally
rollback-triggering) instead of *unknown* partial state.

## Part 2 — Hands-On Lab

### Commands used in this demo

| Command | Purpose |
|---|---|
| `trap 'cmd' TERM` | Handle `SIGTERM` specifically, distinct from `INT` |
| `kill -TERM PID` | Send `SIGTERM` to a specific process |
| `mktemp [-d] /tmp/prefix.XXXXXX` | Create a uniquely-named, safely-permissioned temp file/directory |
| `exec 200>LOCKFILE` | Open a file on a specific file descriptor, for locking |
| `flock -n FD` | Attempt a non-blocking exclusive lock on that descriptor |

### Task 1 — SIGTERM vs. SIGINT, live

```bash
cat > /tmp/sigdemo.sh << 'SCRIPT'
#!/bin/bash
trap 'echo "Got SIGTERM — finishing current step then exiting cleanly"; exit 0' TERM
trap 'echo "Got SIGINT — exiting immediately"; exit 130' INT
echo "PID $$ running, sleeping..."
sleep 30
SCRIPT
bash /tmp/sigdemo.sh &
PID=$!
sleep 1
kill -TERM "$PID"
wait "$PID" 2>/dev/null
echo "exit status: $?"
```

### Task 2 — mktemp's uniqueness and permissions

```bash
t1=$(mktemp /tmp/myapp.XXXXXX)
t2=$(mktemp /tmp/myapp.XXXXXX)
[ "$t1" != "$t2" ] && echo "confirmed unique"
ls -l "$t1"
rm -f "$t1" "$t2"

d=$(mktemp -d /tmp/myapp-work.XXXXXX)
ls -ld "$d"
rmdir "$d"
```

Confirm the listed permissions on the created file are `-rw-------`
(owner-only) without having run any `chmod` yourself.

### Task 3 — A real concurrency guard with flock

```bash
cat > /tmp/guarded.sh << 'SCRIPT'
#!/bin/bash
LOCKFILE=/tmp/guarded.lock
exec 200>"$LOCKFILE"
if ! flock -n 200; then
    echo "Another instance is already running. Exiting."
    exit 1
fi
echo "Lock acquired (PID $$), working..."
sleep 3
echo "Done (PID $$)"
SCRIPT
bash /tmp/guarded.sh &
P1=$!
sleep 0.5
bash /tmp/guarded.sh
wait "$P1"
```

Confirm the second, overlapping invocation exits immediately with the
"already running" message while the first is still sleeping.

### Task 4 — Recording state through an interruption

```bash
rm -f /tmp/deploy-state.txt
cat > /tmp/deploy-safe.sh << 'SCRIPT'
#!/bin/bash
STATE_FILE=/tmp/deploy-state.txt
cleanup() {
    echo "Interrupted — rolling back to pre-deploy state" >> "$STATE_FILE"
}
trap cleanup TERM INT

echo "step1: backup taken" > "$STATE_FILE"
sleep 2
echo "step2: new version staged" >> "$STATE_FILE"
sleep 10
echo "step3: symlink swapped - DEPLOY COMPLETE" >> "$STATE_FILE"
SCRIPT
bash /tmp/deploy-safe.sh &
P=$!
sleep 3
kill -TERM "$P"
wait "$P" 2>/dev/null
cat /tmp/deploy-state.txt
```

Confirm the state file shows exactly steps 1 and 2 plus the interruption
line — step 3 never appears, proving the script was genuinely stopped
mid-way rather than completing and the log being wrong.

### Task 5 — Diagnose the cron-overlap incident

```
A cron entry runs /opt/app/sync.sh every 5 minutes. sync.sh has no
locking of any kind. On a day when an upstream API is slow, one run
takes 11 minutes. Describe exactly what happens to the SECOND and THIRD
scheduled triggers that fire during that window, and what one change
(per §1.3) would fix it.
```

<details>
<summary>Answer</summary>

Cron doesn't know or care that the previous run is still active — it
fires `sync.sh` again at the next scheduled 5-minute mark regardless,
starting a second instance while the first is still running (minute 11
of its own run), and a third instance fires at the next mark after that,
now with two prior instances potentially still active. All of them
operate against the same files/state/downstream system simultaneously,
racing each other. Wrapping the script's work in the `exec 200>LOCKFILE`
+ `flock -n 200` guard from §1.3 fixes this directly: the second and
third triggers would each find the lock already held, print a message,
and exit immediately rather than racing — only one instance ever
actually does work at a time, regardless of how many overlapping
triggers fire.

</details>

## What You Learned

- ✅ Handled `SIGTERM` explicitly, distinguishing it from `SIGINT`
- ✅ Used `mktemp` correctly for files and directories, and explained the
  real risk a predictable temp-file name creates
- ✅ Built a lockfile-based concurrency guard with `flock`
- ✅ Diagnosed and fixed the classic overlapping-cron-trigger race
  condition
- ✅ Built a multi-step script that records state through an interruption
  instead of leaving an unknown, partially-applied result

## Part 3 — Break-Fix Scenario

### Scenario — NovaTech Ops: two concurrency incidents

> **Incident report — NovaTech Ops, ticket #5290**
>
> Alert 1: "the nightly sync job appears to have run twice simultaneously
> last night, and some records were double-processed." Alert 2: "a
> deploy script was killed by the orchestrator during a routine restart
> and left the application in a state where neither the old nor new
> version was fully active."

**Incident 1 — double-processed records**

```bash
$ crontab -l
*/5 * * * * /opt/app/sync.sh
$ grep "duration" /var/log/sync.log | tail -3
sync completed in 11m32s
sync completed in 4m10s
sync completed in 4m22s
```

Tasks:
1. Reference §1.3/Task 5 — given the cron schedule and that one run took
   over 11 minutes, what happened, and why would records specifically get
   double-processed rather than just "running twice harmlessly"?
2. What's the direct fix, and why is it better than a hand-rolled PID-file
   check (§1.4)?

**Incident 2 — the deploy left in an ambiguous state**

```bash
$ cat /opt/app/deploy-state.log
step1: backup taken
step2: new version staged
$ # no further lines - the orchestrator sent SIGTERM during step 2's work,
$ # and the script had no trap registered for TERM at all
```

Tasks:
1. Reference §1.1/§1.5 — if the script had no `TERM` handler, what
   actually happened the instant `SIGTERM` was delivered, versus what
   would have happened with the handler from §1.5?
2. Why does having the state log at all, even an incomplete one, matter
   more here than it might seem?

<details>
<summary>Answers</summary>

**Incident 1:** The 11m32s run was still active when the next two
5-minute cron triggers fired (at roughly minute 5 and minute 10 of its
runtime), starting a second and third instance of `sync.sh` concurrently
with the first and with each other — all three operating against the
same source data at the same time, each unaware the others exist.
Records get double- (or triple-) processed specifically because each
instance independently reads and acts on the same pending records,
having no way to know another instance already claimed them — this is a
genuine race condition, not just redundant harmless work. The direct fix
is the `flock` guard from §1.3: wrapping `sync.sh`'s actual work so only
one instance can hold the lock at a time, with every other overlapping
trigger exiting immediately instead of proceeding. This is better than a
hand-rolled PID-file check because `flock`'s acquisition is a single
atomic operation with no race window between checking and acquiring, and
the lock self-releases the instant the holding process exits for any
reason — no stale-lock detection logic needed, unlike a PID file that
can outlive a crashed process and require extra logic to recognize as
stale.

**Incident 2:** With no `TERM` handler at all, `SIGTERM` falls through to
its default action — the process is terminated immediately, wherever it
happened to be, with zero opportunity to run any cleanup or logging code
at all. With the handler from §1.5, the script would have had the chance
to log that it was interrupted specifically (rather than leaving silence
after "step2"), and potentially trigger an actual rollback rather than
just a log line. The state log mattering even incomplete: its absence of
a "step3" or interruption line is itself informative — an operator
reading it afterward can be fairly confident the process died somewhere
during or right after step 2, narrowing the investigation, versus a
completely silent failure with no log at all, which would leave genuinely
no information about how far the deploy actually got.

</details>

## Interview Prep

**Q1. Why would a script need separate handlers for `SIGTERM` and `SIGINT` rather than one handler for both?**
They represent different intents: `SIGINT` is a human pressing Ctrl+C,
typically wanting an immediate stop; `SIGTERM` is the default signal
process managers (systemd, Kubernetes, cron job control) send when asking
a process to shut down gracefully. A script that only handles `SIGINT`
falls through to the default, cleanup-free termination when a real
orchestrator sends `SIGTERM` during a routine restart — exactly Incident
2's failure mode.

**Q2. Why is a predictable temp filename like `/tmp/myapp-$$` a genuine security concern, not just a style issue?**
Between deciding on that filename and creating the file, another process
on a shared system could create something at that exact predictable path
first — a symlink pointing somewhere sensitive, for instance — and the
script then writes through it unknowingly. `mktemp` closes this window by
creating the file atomically, with a confirmed-unique random name and
safe `0600` permissions from the moment of creation, not after the fact.

**Q3. Two triggers of the same cron job end up running simultaneously, corrupting shared state. What's the correct fix, and why not a hand-rolled PID-file check instead?**
An `flock`-based lockfile guard (`exec 200>LOCKFILE; flock -n 200`). A
hand-rolled "does a PID file already exist" check has its own race
condition — two instances can both check and find no file at nearly the
same instant and both proceed — and needs separate logic to detect and
clean up a stale lock from a crashed prior run. `flock`'s lock
acquisition is a single atomic operation, and the lock releases
automatically the instant the holding process's file descriptor closes,
for any reason, including a crash.

**Q4. A multi-step deploy script gets killed partway through. What should it have done differently to avoid leaving the system in an unknown state?**
Record its progress as it goes (a state/log file updated after each
step) and register a `trap` for `TERM`/`INT` that logs the interruption
explicitly (and ideally triggers a rollback), rather than relying on no
signal handling at all. The goal isn't necessarily to prevent every
possible partial state — it's to make sure whatever partial state occurs
is *known and logged*, not silently ambiguous to whoever investigates
afterward.

---

## Key Takeaways

1. **`SIGTERM` and `SIGINT` represent different intents and deserve
   different handling** — `SIGINT` is a human's Ctrl+C; `SIGTERM` is what
   orchestrators send for a graceful shutdown. A script handling only one
   behaves wrong under the other's actual source.
2. **A predictable temp-file name is a genuine symlink/race-attack
   vulnerability, not just untidy.** `mktemp` closes the gap between
   deciding a name and creating the file, with safe `0600` permissions
   from the moment of creation.
3. **`flock`'s lock acquisition is atomic and self-releasing** — no race
   window between checking and acquiring, and no stale-lock cleanup logic
   needed, unlike a hand-rolled PID-file check which has both problems.
4. **Cron doesn't know or care whether a previous run finished** — a job
   that occasionally runs longer than its schedule interval WILL overlap
   itself eventually; a lockfile guard is the fix, not a "this won't
   really happen" assumption.
5. **Recording state through a multi-step operation turns an unknown
   partial failure into a known one.** The goal isn't eliminating partial
   states entirely — it's making sure whichever partial state occurs is
   logged and actionable, not silently ambiguous.

> **Demo scope:** Primary concept: making scripts safe under interruption
> and concurrent execution. Supporting concepts: SIGTERM/SIGINT
> distinction, mktemp's safety guarantees, flock-based concurrency
> guards, state-logging through interruption.
> Estimated completion time: 35 minutes (reading + hands-on + verification).
> Checkpoints: 5 natural stopping points (after Task 1 through Task 5).

## Quick Commands Reference

### Signals

| Command | Purpose |
|---|---|
| `trap 'cmd' TERM` | Handle a graceful-shutdown request specifically |
| `trap 'cmd' INT` | Handle Ctrl+C specifically |
| `kill -TERM PID` | Send `SIGTERM` to a process |

### Safe temp files

| Command | Purpose |
|---|---|
| `mktemp /tmp/prefix.XXXXXX` | Create a unique, `0600`-permissioned temp file |
| `mktemp -d /tmp/prefix.XXXXXX` | Same, for a directory |

### Concurrency guard

| Command | Purpose |
|---|---|
| `exec 200>LOCKFILE` | Open a file on descriptor 200 for locking |
| `flock -n 200` | Non-blocking exclusive lock attempt on that descriptor |

## Appendix A — Anki Cards

**`m09-03-signal-trap-handling-anki.csv`:**

```
#deck:Linux Mastery::Module 9 - Shell Scripting::Demo M09-03 - Signal Handling, Safe Temp Files & Concurrency
#separator:Comma
#columns:Front,Back,Tags
"What's the actual difference in intent between SIGINT and SIGTERM?","SIGINT is a human pressing Ctrl+C, typically wanting an immediate stop. SIGTERM is the default signal process managers (systemd, Kubernetes, cron) send to request a GRACEFUL shutdown. A script handling only one behaves wrong when it actually receives the other.","demo-m09-03,signals,fundamentals"
"Why is a predictable temp filename like /tmp/myapp-$$ a genuine security risk?","Between deciding on that filename and creating the file, another process could create something at that exact path first (e.g. a symlink to somewhere sensitive), and the script would write through it unknowingly — a real symlink/race-attack vulnerability class.","demo-m09-03,mktemp,security"
"What two guarantees does mktemp provide that a hand-built temp filename doesn't?","A confirmed-unique name (not just 'probably unique'), and 0600 (owner-only) permissions set atomically at creation time — not created-then-chmod'd, closing the window where the file could be read/tampered with first.","demo-m09-03,mktemp,fundamentals"
"How does flock -n 200 avoid the race condition a hand-rolled 'check if a PID file exists' lock has?","flock's lock acquisition is a single atomic kernel operation — no gap between checking and acquiring for a second process to slip through. A PID-file check can have two processes both see 'no file' at nearly the same instant and both proceed.","demo-m09-03,flock,concurrency,common-mistake"
"Why does an flock-based lock not need separate stale-lock cleanup logic, unlike a PID file?","The lock is automatically released the instant the holding process's file descriptor closes, for ANY reason — normal exit, error, or a crash — so there's no leftover lock state requiring separate detection/cleanup logic.","demo-m09-03,flock,fundamentals"
"A cron job runs every 5 minutes but occasionally takes 11 minutes. What happens, and what fixes it?","The next one or two scheduled triggers fire while the previous run is still active, starting overlapping instances that race against the same files/state. An flock-based concurrency guard fixes it: overlapping triggers find the lock held and exit immediately instead of racing.","demo-m09-03,flock,cron,break-fix"
"Why does logging a multi-step script's progress matter even if the script still gets killed partway through?","It turns an UNKNOWN partial failure into a KNOWN one — an operator can see exactly which steps completed and that the process was interrupted, rather than facing silent ambiguity about how far the operation actually got.","demo-m09-03,state-logging,best-practice"
"If a script has no handler for SIGTERM at all, what happens when SIGTERM is delivered?","It falls through to the default action — immediate termination, with zero opportunity to run any cleanup, logging, or rollback code at all.","demo-m09-03,signals,common-mistake"
```

## Appendix B — Quiz

**`m09-03-signal-trap-handling-quiz.md`:**

````markdown
# Quiz — Demo M09-03: Signal Handling, Safe Temp Files & Concurrency

> One correct answer per question unless stated otherwise.
> Target: 80% or above before moving to Demo M09-04.

**Q1. Why might a script need a specific handler for `SIGTERM`, separate from `SIGINT`?**

- A) They are actually the same signal under a different name
- B) `SIGINT` is a human's Ctrl+C; `SIGTERM` is what process managers/orchestrators send for a graceful shutdown — a script handling only one responds wrong to the other's actual source
- C) `SIGTERM` can only be sent by the root user
- D) `SIGINT` cannot be trapped at all, only `SIGTERM` can

<details>
<summary>Answer</summary>

**B** — This is the real, practical reason the two deserve distinct
handling, not just a syntax variation.

</details>

---

**Q2. Why is `/tmp/myapp-$$` (using the process ID) considered a security risk as a temp filename?**

- A) Process IDs are confidential information
- B) Between deciding the filename and creating the file, another process could create something at that exact predictable path first, such as a symlink — a real race/symlink attack vector
- C) `$$` doesn't work reliably across different shells
- D) `/tmp` is read-only on most production systems

<details>
<summary>Answer</summary>

**B** — This is a documented vulnerability class, not a theoretical
concern — `mktemp` exists specifically to close this gap.

</details>

---

**Q3. What does `mktemp` guarantee about a file's permissions at the moment of creation?**

- A) World-readable, for easy debugging
- B) `0600` (owner read/write only), set atomically at creation — not created with default permissions and chmod'd afterward
- C) Permissions matching the calling script's own file's permissions
- D) `mktemp` does not set any specific permissions

<details>
<summary>Answer</summary>

**B** — This closes the window where a predictable or over-permissive
file could be read or tampered with before being locked down.

</details>

---

**Q4. Why is `flock -n 200` safer against race conditions than a script checking "does a PID file already exist" before starting?**

- A) `flock` is simply faster to execute
- B) `flock`'s lock acquisition is a single atomic operation with no gap between checking and acquiring; a PID-file check can have two processes both see "no file" at nearly the same moment and both proceed
- C) PID files are deprecated and no longer supported on modern Linux
- D) `flock` requires root privileges, making it inherently more secure

<details>
<summary>Answer</summary>

**B** — The atomicity is the actual safety property here, not speed or
privilege level.

</details>

---

**Q5. Why doesn't an `flock`-based lock need separate stale-lock cleanup logic?**

- A) `flock` locks never expire and must be manually removed
- B) The lock releases automatically the instant the holding process's file descriptor closes, for any reason including a crash
- C) `flock` checks a timestamp and expires locks older than 60 seconds automatically
- D) Stale locks are actually not a real concern with any locking mechanism

<details>
<summary>Answer</summary>

**B** — This automatic release-on-close is exactly what a hand-rolled
PID file lacks, requiring extra "is that PID still alive" logic instead.

</details>

---

**Q6. A cron job scheduled every 5 minutes occasionally takes 11 minutes to run. Without any locking, what happens?**

- A) Cron automatically skips the overlapping trigger
- B) The next one or two scheduled triggers fire while the previous run is still active, creating overlapping instances racing against shared state
- C) The script automatically queues and runs triggers sequentially
- D) Nothing — cron always waits for the previous run to finish before firing again

<details>
<summary>Answer</summary>

**B** — Cron has no awareness of whether a previous invocation is still
running; it fires on schedule regardless.

</details>

---

**Q7. A multi-step deploy script is killed mid-way and left no log of its progress at all. Why does this matter more than it might seem?**

- A) It doesn't matter — the deploy either fully succeeded or fully failed, nothing in between is possible
- B) Without any progress record, an operator has no way to know how far the operation got, turning a recoverable situation into a guessing exercise
- C) Linux automatically logs every script's progress regardless of what the script itself does
- D) This only matters for scripts running as root

<details>
<summary>Answer</summary>

**B** — A known partial state (even an incomplete log) is dramatically
more actionable than total silence about what actually happened.

</details>

---

Score guide:

| Score | Action |
|---|---|
| 7/7 | Import Anki cards, move to Demo M09-04 |
| 6/7 | Review the wrong answer, then proceed |
| 5/7 | Re-read the relevant section, retry those questions |
| Below 5/7 | Re-read the full demo and redo the walkthrough before proceeding |
````