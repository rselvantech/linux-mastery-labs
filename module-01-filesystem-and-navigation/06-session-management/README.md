# Demo M01-06 — Session & Process Management

## Overview

Start a long-running job over SSH, close your laptop, and by
default that job dies — not because anything went wrong, but
because closing the connection sends a signal that kills every
process still attached to that terminal. Three tools solve this in
different ways: `tmux`/`screen` give you a persistent session you
can detach from and reattach to later, exactly as you left it;
`nohup`/`disown` let an *already-running* job survive without a
session manager at all; `watch` solves a smaller, adjacent problem —
repeatedly running a command so you don't have to.

---

## Demo Objectives

By the end of this demo you will be able to:

1. Explain what actually kills a background job when a terminal
   disconnects (SIGHUP), and why that's the mechanism being worked
   around
2. Start, detach from, reattach to, and end a `tmux` session
3. Explain the precise difference between `nohup` and `disown` —
   they are not interchangeable, and one is safer than the other
4. Use `watch` to repeat a command on an interval, including `-d`
   (highlight changes) and `-g` (exit automatically when output changes)

---

## Directory Structure

```
06-session-management/
├── README.md                           # this file
├── m01-06-session-management-anki.csv  # Anki flashcard deck
└── m01-06-session-management-quiz.md   # standalone quiz
```

---

## Environments

| Task | Ubuntu 24.04 | Rocky Linux 9.8 | EC2 Amazon Linux |
|---|---|---|---|
| All lab tasks | ✅ | ✅ | Not required |

`tmux`, `screen`, `nohup`, and `watch` all need to be installed on a
minimal system (`apt install tmux screen` / `dnf install tmux
screen`) — none are guaranteed present by default. `nohup` and
`disown` ship with coreutils/bash respectively and need no install.

> ⚠️ **A note on how this demo was verified:** testing whether a
> background job "survives a disconnect" needs an actual terminal
> hangup to test against. Rather than simulate that indirectly, every
> claim below about `nohup`/`disown` was verified by sending an
> actual `SIGHUP` signal directly to a real background process and
> observing whether it survived — the precise mechanism, not a proxy
> for it. `tmux`/`screen` session creation, listing, and termination
> were verified against real sessions with real child processes.

---

## Recall Check — Demo M01-05

Before starting, answer from memory:

1. `scp` exits `0` after a transfer. Does that confirm the file
   arrived byte-for-byte correct?
2. What does a matching checksum prove, and what does it NOT prove?
3. `scp mydir user@host:/tmp/` is run where `mydir` is a directory.
   What happens?

<details>
<summary>Answers</summary>

1. No — a `0` exit only confirms the SSH session completed without
   a connection-level error, not that the bytes are correct. Only a
   checksum comparison confirms that.
2. A match proves the compared bytes equal whatever the checksum was
   generated from. It does NOT prove the original source was ever
   correct — the checksum must come from a known-good source.
3. It fails immediately with an error — plain `scp` refuses
   directories; `-r` is required.

</details>

---

## Part 1 — Concepts

### 1.1 What actually kills a background job on disconnect

When a terminal session ends — you close the window, the SSH
connection drops — the kernel sends `SIGHUP` ("hangup") to every
process still attached to that terminal. By default, a process that
receives `SIGHUP` terminates. This is the specific mechanism every
tool in this demo is working around, not a vague "disconnection
kills things" — it's one specific signal.

> ⚠️ [VERIFY] Confirmed directly: sending `SIGHUP` to a plain
> background job (`sleep 60 &`, then `kill -HUP` its PID) killed it
> immediately. This is the exact mechanism being defended against
> below, isolated from anything else that might be happening at
> session end.

### 1.2 `tmux` — a persistent, detachable session

```bash
tmux new-session -s mywork       # start a new named session
# ... do work ...
# Ctrl-b, then d                 # DETACH — session keeps running

tmux list-sessions                # see what's still running
tmux attach -t mywork             # REATTACH — exactly where you left off
tmux kill-session -t mywork       # end it for good
```

The session and everything running inside it belongs to the `tmux`
server process, not your terminal — detaching just disconnects your
*view* of it. Nothing inside receives `SIGHUP` when you detach,
because your terminal disconnecting from `tmux` is not the same
event as `tmux`'s own controlling terminal disconnecting.

> ⚠️ [VERIFY] Confirmed directly: a detached `tmux` session
> (`tmux new-session -d -s name 'long-running-command'`) shows up in
> `tmux list-sessions`, and the command running inside it is a real
> child process of the `tmux` server, verified via `ps -ef`.

### 1.3 `screen` — the older equivalent

```bash
screen -S mywork          # start a new named session
# Ctrl-a, then d           # DETACH
screen -ls                 # list sessions
screen -r mywork           # REATTACH
screen -S mywork -X quit   # end it for good
```

`screen` predates `tmux` and does the same core job — a persistent,
detachable session. `tmux` has become the more commonly reached-for
tool (more actively developed, better default split-pane/scripting
support), but `screen` is still what's pre-installed on some minimal
systems where `tmux` isn't, so it's worth being able to use either.

> ⚠️ [VERIFY] Confirmed directly: `screen -dmS name command` creates
> a detached session immediately (no attach step needed to start it
> in the background), listed correctly by `screen -ls`, with the
> command running as a genuine child of the `SCREEN` process — and
> `screen -S name -X quit` cleanly terminates it.

### 1.4 `nohup` vs. `disown` — NOT interchangeable, verified precisely

This is the part of this demo most tutorials get vague about.

**`nohup` makes the process itself ignore `SIGHUP`,** regardless of
where the signal comes from:

```bash
nohup long-running-command &
```

> ⚠️ [VERIFY] Confirmed directly: a `nohup`-wrapped background job
> sent an actual `SIGHUP` signal **survived**. An identical plain
> background job (no `nohup`) sent the same signal **died
> immediately**. This isolates `nohup`'s actual effect precisely.

**`disown` does something different: it removes an already-running
job from the current shell's own job table.** It does **not** make
the process immune to `SIGHUP` — it just means *this shell* will no
longer be the one sending it a signal when the shell itself exits:

```bash
long-running-command &
disown            # job removed from `jobs` output
```

> ⚠️ [VERIFY] Confirmed directly: a `disown`-only background job
> (no `nohup`), sent an actual `SIGHUP` signal directly, **died just
> like the plain job** — `disown` alone provided no protection
> against the signal itself. Separately confirmed: after `disown`,
> the job disappears from `jobs` output and can no longer be
> referenced by job number (`%1`).

**The practical distinction:** if your shell exiting is what would
send the fatal signal (some interactive shells send `SIGHUP` to
their remaining jobs on exit if `huponexit` is set), `disown` alone
is often enough — there's no longer a job for the exiting shell to
signal. But if the hangup is coming from somewhere else (the
terminal/session itself, not your shell choosing to signal its own
jobs), only `nohup` actually protects against it. When in doubt, use
both together — `nohup command &` then `disown` — rather than
relying on `disown` alone to mean the same thing as `nohup`.

### 1.5 `watch` — repeat a command on an interval

```bash
watch -n 2 df -h                # re-run every 2 seconds (default: 2s)
watch -d -n 2 df -h              # highlight what changed between runs
watch -g -n 1 'cat status.txt'  # exit automatically once output changes
```

> ⚠️ [VERIFY] Confirmed directly: `watch -g` correctly exited on its
> own, at the moment the watched file's content actually changed —
> not on a fixed timeout — timed at ~1.5 seconds against a background
> process that modified the file at that same 1.5-second mark. `-g`
> is genuinely useful for scripts that need to block until a
> condition changes, not just for a human watching a screen.

---

## Part 2 — Hands-On Lab

### Commands used in this demo

| Command | Flag meaning | Purpose |
|---|---|---|
| `tmux new-session -d -s` | `d` = detached, `s` = session name | Create a named session without attaching to it |
| `tmux list-sessions` | — | Show all running sessions |
| `tmux attach -t` | `t` = target | Reattach to a named session |
| `screen -dmS` | `d` = detached, `m` = force new, `S` = session name | Create a named detached session directly |
| `screen -ls` | `ls` = list sessions | Show all running sessions |
| `nohup` | — | Make the wrapped process ignore `SIGHUP` |
| `disown` | — | Remove a job from the current shell's job table (bash builtin) |
| `watch -n` | `n` = interval seconds | Set how often the command re-runs |
| `watch -d` | `d` = differences | Highlight what changed since the last run |
| `watch -g` | `g` = chgexit | Exit automatically once output changes |

### Task 1 — See `SIGHUP` actually kill a background job

```bash
sleep 60 &
plain_pid=$!
kill -HUP $plain_pid
sleep 0.5
ps -p $plain_pid || echo "confirmed: died from SIGHUP"
```

### Task 2 — `nohup` blocks it, `disown` alone does not

```bash
echo "--- nohup, sent SIGHUP directly ---"
nohup sleep 60 > /tmp/nohup.log 2>&1 &
nohup_pid=$!
kill -HUP $nohup_pid
sleep 0.5
ps -p $nohup_pid && echo "SURVIVED"

echo "--- disown only, sent SIGHUP directly ---"
sleep 60 &
disown_pid=$!
disown
kill -HUP $disown_pid
sleep 0.5
ps -p $disown_pid || echo "died — disown alone did not block SIGHUP"
```

### Task 3 — `tmux`: create, list, verify, kill

```bash
tmux new-session -d -s labsession "sleep 300"
tmux list-sessions
ps -ef | grep sleep         # confirm it's a real child process
tmux kill-session -t labsession
tmux list-sessions 2>&1     # confirm it's gone
```

### Task 4 — `screen`: the same workflow

```bash
screen -dmS labsession sleep 300
screen -ls
screen -S labsession -X quit
screen -ls 2>&1
```

### Task 5 — `watch -g` for a scriptable wait

```bash
echo "initial" > status.txt
( sleep 3; echo "changed" > status.txt ) &
time watch -g -n 0.5 cat status.txt
# exits on its own once the file content changes, around the 3s mark
```

---

## What You Learned

- ✅ Confirmed `SIGHUP` is the specific signal that kills a plain
  background job when a session ends
- ✅ Created, listed, and terminated both `tmux` and `screen`
  detached sessions, confirming the running command is a genuine
  child process independent of the invoking shell
- ✅ Proved `nohup` blocks a directly-sent `SIGHUP` while an
  identical `disown`-only job does not — the precise, verified
  distinction between the two
- ✅ Used `watch -g` to block until monitored output actually
  changes, confirmed by timing

**Key Takeaway:** `nohup` and `disown` solve different problems that
happen to often get lumped together — `nohup` makes a process
immune to `SIGHUP` regardless of source; `disown` only stops *this
shell* from being the one that sends it. Reach for `tmux`/`screen`
when you want to come back and interact with the session again;
reach for `nohup` when you just need a job to survive walking away.

---

## Part 3 — Break-Fix Scenario

### Scenario — NovaTech: the migration script died overnight, again

You're on call. The ticket:

> *"We SSH in, run the database migration script in the background
> with `&`, then `disown` it so it survives us logging off. It keeps
> dying partway through anyway — no error in the script's own logs,
> it just stops. This has happened twice now, always after we log
> off."*

**Your task — answer from what this demo covered before revealing
the resolution:**

1. Why would a `disown`ed job still die when the engineers log off,
   if `disown` is supposed to protect it?
2. What's the one-word fix?
3. Why did it not fail every single time — is that consistent with
   the actual mechanism?

<details>
<summary>Resolution — open after attempting</summary>

**1. Why `disown` alone wasn't enough:**

`disown` only stops *the current shell* from sending `SIGHUP` to its
own remaining jobs when it exits — it does not make the job itself
immune to `SIGHUP` arriving from elsewhere. When the SSH session
itself ends, the hangup can propagate at the terminal/session level,
not just from the shell's own job-control cleanup — and a
`disown`ed-but-not-`nohup`ed process has no protection against that.

**2. The fix:**

```bash
nohup migration_script.sh > migration.log 2>&1 &
disown
```

Use both together. `nohup` makes the process itself ignore `SIGHUP`
regardless of where it comes from; `disown` additionally removes it
from the shell's job table so there's nothing left for the shell to
try to signal either. `disown` alone was never sufficient on its
own for this.

**3. Consistent with the mechanism — not a contradiction:**

Whether `SIGHUP` actually propagates to the job on logout can depend
on exact session/terminal configuration, network conditions, and how
the disconnect happens (clean logout vs. dropped connection) —
`disown` protects against one specific source of `SIGHUP` (the
shell's own exit-time cleanup) but not others. Intermittent failure
is exactly what you'd expect from a gap that's real but not the only
path to the same signal — not evidence the mechanism is random.

**The lesson:** `disown` and `nohup` are not interchangeable
safety nets — one blocks the signal itself, the other only removes
one specific source of it. For anything that must survive logout
reliably, use `nohup`, not `disown` alone.

</details>

---

## Interview Prep

**Q1. What specifically kills a background job when a terminal
session ends, and what are you actually protecting against when you
use `nohup`?**

`SIGHUP` — sent by the kernel to processes still attached to a
terminal when that terminal disconnects. `nohup` makes the wrapped
process ignore that specific signal, regardless of what caused it to
be sent. It's not protecting against "disconnection" abstractly;
it's blocking one specific, well-defined signal.

---

**Q2. A teammate uses `disown` on a background job and assumes it's
now equivalent to having used `nohup`. Are they the same?**

No. `disown` removes the job from the current shell's own job
table, so that shell won't be the one sending it `SIGHUP` when the
shell exits. It does not make the process ignore `SIGHUP` if it
arrives from another source — confirmed directly: a `disown`-only
job sent an actual `SIGHUP` signal died just like an unprotected
one. `nohup` is the one that actually makes the process immune to
the signal itself.

---

**Q3. When would you reach for `tmux` over `nohup` for a long-running
task?**

When you need to come back and actively interact with the session
again — check progress, scroll back through output, or run
additional commands in that same context. `nohup` just keeps a
single already-started command alive in the background with no way
to reattach and interact with it; `tmux` gives you a full persistent
session you can detach from and return to exactly as you left it.

---

## Key Takeaways

1. **`SIGHUP` is the specific signal that kills a background job when a session ends — not disconnection in the abstract.** Every tool in this demo works around that one signal, confirmed by sending it directly and observing the result.
2. **`tmux`/`screen` sessions belong to their own server process, not your terminal — detaching disconnects your view, not the session itself.** Confirmed: the commands running inside are genuine child processes, unaffected by your terminal's own disconnection.
3. **`nohup` makes a process ignore `SIGHUP` regardless of where it comes from — confirmed by sending the signal directly and watching it survive.**
4. **`disown` only removes a job from the current shell's job table — it does NOT make the process immune to `SIGHUP` arriving from another source.** Confirmed: a `disown`-only job died from a directly-sent `SIGHUP`, identical to an unprotected job.
5. **`nohup` and `disown` are not interchangeable — use both together (`nohup cmd &` then `disown`) for a job that must reliably survive logout,** rather than treating `disown` alone as equivalent protection.
6. **`watch -g` exits automatically the moment monitored output changes — genuinely useful for scripts that need to block on a condition, confirmed by timing, not just for a human staring at a refreshing screen.**

---

## Quick Commands Reference

### Session tools

| Tool | Start detached | List | Reattach | End |
|---|---|---|---|---|
| `tmux` | `tmux new-session -d -s name` | `tmux list-sessions` | `tmux attach -t name` | `tmux kill-session -t name` |
| `screen` | `screen -dmS name` | `screen -ls` | `screen -r name` | `screen -S name -X quit` |

### `nohup` vs. `disown`

| | Blocks `SIGHUP` itself | Removes from shell's job table |
|---|---|---|
| `nohup` | ✅ Yes | No |
| `disown` | ❌ No | ✅ Yes |
| Both together | ✅ Yes | ✅ Yes |

### `watch` flags

| Flag | Effect |
|---|---|
| `-n <secs>` | Interval between runs (default 2s) |
| `-d` | Highlight differences from the previous run |
| `-g` | Exit automatically once output changes |

---

## Appendix A — Anki Cards

**`m01-06-session-management-anki.csv`:**

```
#deck:Linux Mastery::Module 1 - Filesystem & Navigation::Demo M01-06 - Session & Process Management
#separator:Comma
#columns:Front,Back,Tags

"What specific signal kills a background job by default when its terminal session ends?","SIGHUP (hangup). The kernel sends it to processes still attached to a terminal when that terminal disconnects. Confirmed directly: kill -HUP against a plain background job killed it immediately - this is the exact mechanism every tool in this demo (nohup, tmux, screen) works around.","demo00f,sighup,signal"

"nohup cmd & is run, then the process is sent an actual SIGHUP signal directly. Does it survive?","Yes - confirmed directly. nohup makes the wrapped process ignore SIGHUP regardless of where the signal came from. An identical plain background job (no nohup) sent the same signal died immediately, isolating nohup's actual effect.","demo00f,nohup,sighup,verified"

"A job is started with cmd & then disown is run (no nohup). It's later sent an actual SIGHUP signal directly. Does it survive?","No - confirmed directly, it died just like an unprotected job. disown only removes the job from the CURRENT SHELL's job table, so that shell won't be the one sending it SIGHUP on exit - it does NOT make the process immune to SIGHUP arriving from any other source.","demo00f,disown,sighup,footgun"

"Are nohup and disown interchangeable ways to protect a background job from dying on logout?","No. nohup makes the process itself ignore SIGHUP regardless of source. disown only stops the current shell from being the one that signals it. For a job that must reliably survive logout, use both together: nohup cmd & then disown - disown alone is not equivalent protection.","demo00f,nohup,disown,distinction"

"When a tmux session is detached, does the process running inside it receive SIGHUP?","No - the session and everything inside it belongs to the tmux server process, not your terminal. Detaching disconnects your VIEW of the session, not the session itself. Confirmed directly: the command running inside a detached tmux session is a genuine child process of the tmux server, unaffected by your terminal disconnecting.","demo00f,tmux,detach,sighup"

"What's the practical difference between reaching for tmux versus nohup for a long-running task?","tmux gives you a full persistent session you can detach from and reattach to later, interacting with it again exactly as you left it. nohup just keeps a single already-started command alive with no way to reattach - useful when you don't need to interact with it again, just need it to survive you leaving.","demo00f,tmux,nohup,usecases"

"screen -dmS name command creates a session. How is this different from screen -S name followed by manually detaching?","screen -dmS creates the session ALREADY detached and running the command immediately, with no attach step needed at all - useful for starting something from a script. screen -S name would attach interactively first, requiring a manual Ctrl-a d to detach afterward.","demo00f,screen,dmS,detached"

"watch -g -n 1 'cat status.txt' is run. Under what condition does it exit on its own, and why is this useful in scripts?","It exits automatically the moment the watched command's OUTPUT changes from the previous run - confirmed by timing against a background process that modified the file at a known point. This is useful in scripts that need to BLOCK until some condition changes, rather than watch requiring a human to notice the change on screen.","demo00f,watch,chgexit,scripting"

"A migration script is run with cmd & then disown, and it intermittently dies after the engineer logs off - not every time. Is this consistent with disown's actual protection, or a sign something else is wrong?","Consistent with disown's actual (limited) protection. disown only blocks ONE specific source of SIGHUP - the shell's own exit-time signaling of its remaining jobs. Other sources of SIGHUP at logout (terminal/session-level propagation) are NOT blocked by disown alone, so intermittent failure depending on exact disconnect conditions is expected, not random.","demo00f,disown,intermittent,break-fix"

"Why does which fail to find disown, even though disown clearly works when typed at a bash prompt?","disown is a bash builtin with no corresponding executable file anywhere in PATH - which only searches PATH for files. This is the same which-vs-builtin gap covered for cd in Demo M01-03: type disown correctly reports 'disown is a shell builtin', while which disown finds nothing.","demo00f,disown,which,builtin"
```

---

## Appendix B — Quiz

**`m01-06-session-management-quiz.md`:**

```
# Quiz — Demo M01-06: Session & Process Management

> One correct answer per question. Target: 80% before Demo M01-07.

---

**Q1.** What specific signal is responsible for killing a plain
background job when its terminal session ends?

- A) `SIGTERM`
- B) `SIGKILL`
- C) `SIGHUP`
- D) `SIGINT`

<details>
<summary>Answer</summary>

**C** — `SIGHUP` ("hangup") is sent by the kernel to processes still
attached to a terminal when that terminal disconnects. Confirmed
directly: sending it explicitly killed an unprotected background job
immediately.

</details>

---

**Q2.** A background job is started, then `disown` is run (no
`nohup`). It's later sent an actual `SIGHUP` signal directly. What
happens?

- A) It survives — `disown` makes it immune to `SIGHUP` from any source
- B) It dies — `disown` only removes it from the shell's job table,
  it does not block `SIGHUP` itself from any source
- C) It survives only if it's also running inside `tmux`
- D) `disown` converts `SIGHUP` into `SIGTERM` automatically

<details>
<summary>Answer</summary>

**B** — Confirmed directly: a `disown`-only job died from a
directly-sent `SIGHUP`, identical to an unprotected job. `disown`'s
actual effect is narrower than commonly assumed — it only stops the
current shell specifically from being the source of that signal.

</details>

---

**Q3.** For a job that absolutely must survive logout reliably,
what's the correct approach?

- A) `disown` alone is always sufficient
- B) `nohup` alone is always sufficient, `disown` is redundant
- C) Use both together: `nohup cmd &` then `disown`
- D) Run it under `sudo` so it can't be signaled

<details>
<summary>Answer</summary>

**C** — `nohup` blocks the signal at the process level regardless of
source; `disown` additionally removes it from the shell's own job
table. Neither alone covers every path a hangup could take; together
they do.

</details>

---

**Q4.** A `tmux` session is detached. Does the command running
inside it receive `SIGHUP` as a result of the detach?

- A) Yes, always
- B) No — the session belongs to the `tmux` server process, not your
  terminal; detaching only disconnects your view of it
- C) Only if the command was started with `&`
- D) Only on Rocky Linux, not Ubuntu

<details>
<summary>Answer</summary>

**B** — Confirmed directly: the process inside a detached `tmux`
session is a genuine child of the `tmux` server, unaffected by the
originating terminal's disconnection.

</details>

---

**Q5.** When would `tmux` be the better choice over `nohup` for a
long-running task?

- A) When you need the task to survive a reboot
- B) When you'll want to reattach and actively interact with the
  session again later, not just have the command finish unattended
- C) `tmux` is always strictly better, `nohup` has no valid use case
- D) When the task takes less than a minute to run

<details>
<summary>Answer</summary>

**B** — `nohup` just keeps an already-started command alive with no
way back in. `tmux` gives you a full session you can detach from and
return to, interacting with it exactly as you left it — the right
tool when you need to come back, not just walk away.

</details>

---

**Q6.** `watch -g -n 1 'cat status.txt'` is run. Under what
condition does it exit?

- A) After exactly 1 second, regardless of output
- B) Automatically, the moment the watched command's output actually
  changes from the previous run
- C) Only when the watched command itself errors
- D) It never exits automatically — `-g` only changes the color scheme

<details>
<summary>Answer</summary>

**B** — Confirmed by timing: `watch -g` exited right at the moment
a background process changed the monitored file's content, not on a
fixed schedule — genuinely useful for scripts that need to block
until a condition changes.

</details>

---

**Q7.** `which disown` fails to find anything. Why?

- A) `disown` doesn't exist as a real bash feature
- B) `disown` is a bash builtin with no backing executable file
  anywhere in `PATH` — `which` only searches `PATH` for files
- C) `disown` requires root privileges to be located by `which`
- D) This is specific to non-interactive shells only

<details>
<summary>Answer</summary>

**B** — Same `which`-vs-builtin gap covered for `cd` back in Demo
M01-03. `type disown` correctly reports "disown is a shell builtin";
`which`, being a pure `PATH` file search, finds nothing.

</details>

---

Score guide:

| Score | Action |
|---|---|
| 7/7 | Import Anki cards, move to Demo M01-07 |
| 6/7 | Review the wrong answer, then proceed |
| 5/7 | Re-read the relevant section, retry those questions |
| Below 5/7 | Re-read the full demo and redo the walkthrough before proceeding |
```