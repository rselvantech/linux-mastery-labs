# Demo 06 — Process Management

## Overview

Every running program on a Linux system is a process — from the
kernel's own init process down to the `curl` command you just ran.
Understanding how processes relate to each other, what state
they're in, how to communicate with them via signals, and how to
control their priority and lifecycle is core to running anything
in production: a hung deployment, a runaway process eating CPU, a
background job that silently died when your SSH session dropped —
all of these are process management problems with process
management solutions.

This demo covers the process tree, process states, signals (the
correct way to stop a process, and why `kill -9` is a last resort
not a default), foreground/background job control, process
priority with `nice`/`renice`, and real-time monitoring with `top`
and `htop`.

---

## Demo Objectives

By the end of this demo you will be able to:

1. Explain what a process is, read PID/PPID relationships, and
   navigate the process tree
2. Identify every process state (`R`, `S`, `D`, `T`, `Z`) from
   `ps` output and explain what a zombie process actually is
3. Send the correct signal for the situation — explain why
   `SIGTERM` is the default and `SIGKILL` is a last resort
4. Find and kill processes correctly with `kill`, `killall`,
   and `pkill`
5. Manage foreground and background jobs — `&`, `jobs`, `fg`,
   `bg` — and explain why a background job dies when an SSH
   session disconnects unless `nohup` or `disown` is used
6. Set and change process priority with `nice` and `renice`,
   and explain why a lower nice value means higher priority
7. Diagnose a runaway process by CPU/memory using `top` and `htop`
8. Connect process-held file handles back to the `lsof` pattern
   from Demo 02, and explain why a service account's process
   still appears in `ps` even though the account itself cannot
   log in interactively (Demo 05)

---

## Directory Structure

```
06-process-management/
├── README.md                          # this file
├── 06-process-management-anki.csv     # Anki flashcard deck
└── 06-process-management-quiz.md      # standalone quiz
```

---

## Environments

| Task | Ubuntu 24.04 | Rocky Linux 9.8 | EC2 Amazon Linux 2023 |
|---|---|---|---|
| All lab tasks | ✅ | ✅ | Not required |
| `htop` | Requires `apt install htop` | Requires `dnf install htop` | Same as Rocky9 |

`htop` is not installed by default on minimal images on either
distro — Task 6 covers the one-line install on each.

---

## Recall Check

> Answer from memory before reading further. From Demo 05.

1. An admin runs `usermod -G backup deployer` intending to add
   deployer to the `backup` group. `id deployer` now shows only
   `backup` — `sudo` is gone. What happened and what's the fix?

2. A user is added to the `sudo` group on Rocky Linux 9, but sudo
   still doesn't work. Why?

3. What two settings are essential for a correct service account
   that should never allow interactive login?

<details>
<summary>Answers</summary>

1. Bare `-G` (without `-a`) replaces the entire supplementary
   group list rather than adding to it — every prior group except
   the primary group is wiped. Fix: `usermod -aG sudo,backup
   deployer` — `-a` appends instead of replacing.

2. `sudo` is the Debian/Ubuntu group name. RHEL-family distros
   (Rocky, RHEL, CentOS, Amazon Linux, Fedora) use `wheel` instead
   — that's the group with the actual sudoers rule.

3. Shell set to `/usr/sbin/nologin` (rejects interactive login
   while still letting systemd start the process as that user),
   and a locked/no usable password (`passwd -S` showing `L` or
   `NP`).

</details>

---

## Part 1 — Concepts

### 1.1 What is a process

A **process** is a running instance of a program — the kernel
allocates it memory, a process ID, and tracks its state from the
moment it starts until it exits. Every process is the child of
exactly one parent process (except PID 1, the very first process
the kernel starts at boot), forming a tree that traces back to
PID 1 at the root.

**Key identifiers every process has:**

```
PID    Process ID — a unique number identifying this process
PPID   Parent Process ID — the PID of the process that started it
UID    User ID the process runs as — determines its permissions
       (this is the same UID concept from Demo 05's /etc/passwd)
```

```bash
ps -ef | head -5
# UID    PID  PPID  C STIME TTY          TIME CMD
# root     1     0  0 09:00 ?        00:00:02 /sbin/init
# root   412     1  0 09:00 ?        00:00:00 /usr/lib/systemd/systemd-journald
# wadmin 1823  1801 0 09:15 pts/0    00:00:00 bash
# wadmin 1901  1823 0 09:20 pts/0    00:00:00 ps -ef
```

Reading this: PID 1 (`init`/`systemd`) has PPID 0 — it has no
parent, it's the root of the tree. Every other process traces its
ancestry back to PID 1 eventually. `bash` (PID 1823) is the parent
of the `ps` command itself (PID 1901, PPID 1823) — typing a
command in your shell makes your shell the parent of whatever you
just ran.

**When a parent process exits before its child:** the child is
**orphaned** and gets re-parented to PID 1 (or to a subreaper
process on modern systemd systems) — it doesn't simply die. This
is exactly the mechanism that makes background jobs survive an
SSH disconnect when started correctly (covered in section 1.5).

### 1.2 Viewing processes — ps and pstree

**`ps`** = **p**rocess **s**tatus. Shows a snapshot of running
processes at the moment you run it — not live, not auto-updating.

**Two command styles you'll see, both common:**

```bash
ps aux        # BSD-style flags (no dash) — most common in practice
ps -ef        # UNIX-style flags (with dash) — POSIX standard

# Both show similar information, slightly different columns:
ps aux
# USER   PID  %CPU %MEM    VSZ   RSS TTY   STAT START   TIME COMMAND
ps -ef
# UID    PID  PPID  C STIME TTY          TIME CMD
```

`ps aux` includes `%CPU`/`%MEM` directly — useful for spotting
resource hogs. `ps -ef` includes `PPID` directly — useful for
tracing parent/child relationships. Use whichever columns you
need; both are installed everywhere by default.

```bash
ps aux | grep nginx          # find a specific process by name
ps -ef | grep -v grep | grep nginx   # exclude grep's own match
pgrep nginx                  # cleaner — just prints matching PIDs
pgrep -l nginx               # PIDs with process name
```

**`pstree`** — shows the parent/child tree visually, instead of a
flat list:

```bash
pstree -p
# systemd(1)─┬─systemd-journald(412)
#            ├─sshd(1650)───sshd(1801)───bash(1823)───pstree(1905)
#            └─nginx(2001)─┬─nginx(2002)
#                          └─nginx(2003)
# -p shows PIDs in parentheses
```

This makes parent/child relationships immediately visible — nginx
worker processes (2002, 2003) are children of the nginx master
(2001), which is exactly how nginx is designed to run.

### 1.3 Process states — what each letter means

Every process is in exactly one state at any moment. `ps`/`top`
show this as a single letter in the `STAT`/`S` column.

```
R   Running or runnable — actively using CPU, or waiting for CPU time
S   Sleeping (interruptible) — waiting for an event (input, a timer,
    a lock) — the vast majority of idle processes sit here
D   Uninterruptible sleep — waiting on I/O (usually disk), cannot
    be interrupted even by a signal. Normal briefly; concerning if
    stuck here for a long time (often a sign of disk/storage trouble)
T   Stopped — paused by a signal (SIGSTOP) or job control, not running
    at all until resumed (covered in section 1.5)
Z   Zombie — process has exited, but its parent hasn't yet read its
    exit status. The process itself is gone; only an entry in the
    process table remains.
```

You may also see a `+` after the letter (e.g., `S+`) meaning the
process is in the foreground process group of its terminal — not
a different state, just additional detail.

**What a zombie actually is, and why you can't `kill` it:**

A zombie process has already finished executing — there's no
running program left, no memory in use, no CPU consumption. What
remains is a single entry in the kernel's process table holding
the exit status, waiting for the parent to call `wait()` and
collect it. You cannot kill something that's already dead — `kill`
sends a signal to a running process, and a zombie has nothing left
to receive or act on a signal.

```bash
ps aux | grep 'Z'
# wadmin   2150  0.0  0.0      0     0 ?        Z    10:02   0:00 [myapp] <defunct>
#                                                ↑
#                                                Z state, <defunct> tag confirms zombie
```

**The actual fix for a zombie:** you fix the **parent**, not the
zombie. Either the parent process needs to call `wait()` (a code
fix, if it's your own application not reaping its children
correctly), or if the parent itself has exited or is unresponsive,
killing/restarting the parent causes the kernel to re-parent the
zombie to PID 1, which reaps it immediately. A small number of
zombies that clear quickly is normal; zombies that accumulate
without clearing indicates a real bug in the parent process.

### 1.4 Signals — communicating with a process

A **signal** is a message sent to a process by the kernel, by
another process, or by a user — asking it to do something:
terminate, pause, reload its configuration, etc. The process can
choose how to respond to most signals (this is called "handling"
or "trapping" the signal) — with two important exceptions.

**The signals you actually need to know:**

```
SIGTERM   15   "Please terminate." The default, polite signal.
                The process can catch this and shut down cleanly —
                close files, finish in-flight requests, save state.
                This is what plain `kill PID` sends by default.

SIGKILL    9   "Terminate immediately, no choice." Cannot be caught,
                blocked, or ignored by the process — the kernel
                forcibly removes it. No cleanup happens. Use only
                when SIGTERM has failed or the process is hung.

SIGHUP     1   Historically "the terminal hung up" (closed). Many
                daemons interpret this as "reload your config file
                without restarting" — e.g. nginx, sshd.

SIGINT     2   "Interrupt." What Ctrl+C sends to the foreground
                process. Same default effect as SIGTERM unless the
                program handles it differently.

SIGSTOP   19   Pause the process completely (cannot be caught or
                ignored). Resume with SIGCONT. Used for job control
                (section 1.5).

SIGCONT   18   Resume a stopped process.
```

**Why `SIGTERM` is the default and `kill -9` is a last resort:**

`SIGTERM` gives the process a chance to clean up — close open file
handles, flush buffered writes to disk, finish an in-flight
database transaction, remove a lock file, deregister from a load
balancer. `SIGKILL` gives it none of that — the process is removed
mid-instruction with no chance to react. Killing a database process
with `SIGKILL` can leave data files in a corrupted, half-written
state that `SIGTERM`'s clean shutdown would have avoided entirely.

The correct escalation path in practice:

```bash
kill PID           # sends SIGTERM by default — try this first
sleep 5            # give the process a few seconds to exit cleanly
kill -9 PID        # only if it's still running after a reasonable wait
```

### 1.5 Finding and killing processes — kill, killall, pkill

```bash
kill PID                  # send SIGTERM (default) to a specific PID
kill -9 PID               # send SIGKILL — last resort
kill -SIGHUP PID          # send by name
kill -1 PID               # send by number — 1 = SIGHUP
kill -l                   # list all signal names and numbers

killall nginx             # kill ALL processes matching this exact name
killall -9 nginx          # force-kill all matching processes

pkill -f "python app.py"  # kill by matching the full command line,
                          # not just the process name — useful when
                          # multiple different scripts run as "python"
```

**Why `pkill -f` matters:** `killall python` would kill every
Python process on the system, regardless of which script it's
running — dangerous on a shared server. `pkill -f "python app.py"`
matches against the full command line, only killing processes that
were actually started with that exact script.

**Always confirm what you're about to kill before killing it:**

```bash
pgrep -fl "python app.py"   # list matches WITHOUT killing — verify first
pkill -f "python app.py"    # then kill, once you've confirmed the match
```

### 1.6 Foreground vs background jobs

When you run a command normally, it runs in the **foreground** —
it has your terminal's input/output, and your shell waits for it
to finish before giving you back the prompt.

```bash
sleep 100          # foreground — terminal is blocked until this finishes
                   # Ctrl+C sends SIGINT to stop it
```

**Running in the background — `&`:**

```bash
sleep 100 &
# [1] 2150        ← job number [1], PID 2150
# Prompt returns immediately — you can keep working

jobs               # list background jobs in this shell session
# [1]+  Running    sleep 100 &

fg %1              # bring job 1 back to the foreground
bg %1              # resume a stopped job in the background
```

**The critical problem: background jobs and SSH disconnects.**

A background job started with plain `&` is still a **child of your
shell**. If your shell exits — including when your SSH session
disconnects, intentionally or not — the kernel sends `SIGHUP` to
every job still attached to that shell, and the job dies with it.
This is the single most common cause of "I started a deployment,
my connection dropped, and now it's half-finished."

**`nohup` — the fix, started correctly from the beginning:**

```bash
nohup ./deploy.sh &
# nohup: ignoring input and appending output to 'nohup.out'
```

`nohup` makes the process **ignore** SIGHUP specifically — when
your shell exits, the kernel still sends the signal, but the
process is now set up to disregard it and keeps running, getting
re-parented to PID 1 as covered in section 1.1.

**`disown` — the fix for a job already running:**

If you forgot to use `nohup` and the job is already running in the
background of your current session:

```bash
./deploy.sh &
disown -h %1
# Removes the job from your shell's job table without killing it —
# it will no longer receive SIGHUP from this shell on exit
```

`nohup` must be used at the moment you start the command. `disown`
is the fix for a job you already started without `nohup` — it
can't retroactively redirect output the way `nohup` does, but it
does protect the running process from dying on disconnect.

### 1.7 Process priority — nice and renice

The kernel's scheduler decides which process gets CPU time and
when. You can influence — not dictate — this with a **niceness**
value.

**The scale, and why lower means higher priority:**

```
-20   highest priority — most CPU time relative to other processes
  0   default priority — what every process starts at, normally
+19   lowest priority — most willing to wait, "nice" to other processes
```

The name comes from the process being "nice" to others — a high
nice value means "I'm being considerate, let other processes go
first." A low (or negative) nice value means the opposite — the
process insists on more CPU time than default. This naming is the
single most common point of confusion in this topic: a *lower*
number is a *higher* priority.

```bash
nice -n 10 ./backup-script.sh        # start a NEW process at nice +10
                                     # (lower priority — good for batch
                                     # jobs you don't want competing
                                     # with interactive work)

renice -n 5 -p 2150                  # change the priority of an
                                     # ALREADY RUNNING process (PID 2150)

renice -n -10 -p 2150                # raise priority (needs root —
                                     # only root can set negative values)
```

**Permission rule:** any user can lower their own process's
priority (raise the nice value, e.g. 0 → 10). Only root can raise
priority into negative values. This prevents a regular user from
starving other users' processes of CPU time.

```bash
ps -el | grep 2150          # PRI and NI columns show current priority/niceness
top                          # NI column shown live, per process
```

### 1.8 Monitoring with top and htop

**`top`** — installed by default on essentially every Linux
system. Shows a live, auto-refreshing view of processes sorted by
resource usage.

```bash
top
# Press 'q' to quit
# Press 'P' to sort by %CPU (often the default)
# Press 'M' to sort by %MEM
# Press 'k' then enter a PID to kill a process from within top
# Press 'r' then enter a PID and a nice value to renice from within top
```

**Reading `top`'s header:**

```
top - 14:32:10 up 3 days,  2:15,  1 user,  load average: 0.45, 0.52, 0.38
                                            ↑
                                            load average over 1, 5, 15 minutes
                                            — roughly, how many processes were
                                            wanting CPU time on average; a
                                            number consistently above your
                                            CPU core count means the system
                                            is CPU-saturated
```

**`htop`** — not installed by default; a significant usability
upgrade over `top` — color output, mouse support, easier
sorting/filtering, and a built-in process tree view.

```bash
# Ubuntu
sudo apt install -y htop

# Rocky9
sudo dnf install -y htop

htop
# F3 to search, F4 to filter, F5 to toggle tree view (shows
# parent/child relationships directly, like pstree but live)
# F9 to kill a selected process, choosing the signal explicitly
```

### 1.9 Connecting back — Demo 02 and Demo 05

**Demo 02's `lsof | grep deleted` pattern is fundamentally a
process-management fact:** a deleted file stays allocated on disk
specifically because some **process** still holds it open. `lsof`
ties file handles back to the PID holding them:

```bash
lsof | grep deleted
# myapp     2150  wadmin   4u   REG   8,1   104857600   524291 /var/log/app.log (deleted)
#  ↑          ↑
#  process    PID holding the file open — this is the process you
#  name       need to restart to actually free the disk space
```

Now that this demo has covered `kill`/`pkill`, the Demo 02 fix
("restart the process holding the file") is a concrete action you
can perform: identify the PID from `lsof`, then `kill` it
correctly (SIGTERM first) to release the handle.

**Demo 05's service accounts still run as real processes:** a
service account created with `/usr/sbin/nologin` cannot be used
for interactive login — but the **process** systemd starts as that
account is a completely normal process, visible in `ps`/`top`,
killable, reniceable, and subject to every concept in this demo
exactly the same as a process started by a human-logged-in user.
`nologin` restricts the *account's login capability*, not the
*process's* behavior once running.

```bash
ps -ef | grep myapp-svc
# myapp-svc  2301     1  0  09:00 ?  00:01:12 /opt/myapp/bin/myapp
#                     ↑
#                     PPID 1 — started directly by systemd (PID 1),
#                     not by any login shell, exactly as expected
#                     for a correctly configured service account
```

---

## Part 2 — Hands-On Lab

### Commands used in this demo

| Command | Letter meaning | Purpose |
|---|---|---|
| `ps` | process status | Snapshot of running processes |
| `pstree` | process tree | Visual parent/child process tree |
| `pgrep` | process grep | Find PIDs by name/pattern |
| `kill` | — | Send a signal to a process by PID |
| `killall` | — | Send a signal to all processes matching a name |
| `pkill` | process kill | Send a signal by matching name/command line |
| `nice` | — | Start a new process at a given priority |
| `renice` | — | Change priority of a running process |
| `nohup` | no hang up | Make a process ignore SIGHUP |
| `disown` | — | Detach a running job from the current shell |
| `top` | — | Live process monitor |
| `htop` | — | Enhanced live process monitor |

---

### Task 1 — Explore the process tree

**Run on Ubuntu, then Rocky9:**

```bash
# Flat view, both common styles
ps aux | head -10
ps -ef | head -10

# Visual tree
pstree -p | head -20

# Find your own shell's PID and PPID
echo "My shell PID: $$"
ps -ef | grep $$
# Note the PPID — what process started your shell? (likely sshd)

# Find a specific process and its parent
pgrep -l sshd
ps -ef | grep sshd
```

**Paste your `pstree -p` output — identify your own shell in the
tree and what its parent process is.**

---

### Task 2 — Process states, including a zombie

**Run on Ubuntu, then Rocky9:**

```bash
# Observe a normal sleeping process
sleep 60 &
ps aux | grep sleep
# Note the STAT column — should show S (sleeping)

# Create a zombie deliberately, for observation only
cat << 'EOF' > /tmp/zombie-demo.sh
#!/bin/bash
sleep 5 &
CHILD_PID=$!
echo "Child PID: $CHILD_PID"
sleep 10
# Parent never calls wait — child becomes a zombie after it exits
# until this script itself exits
EOF
chmod +x /tmp/zombie-demo.sh

/tmp/zombie-demo.sh &
sleep 6
ps aux | grep -i defunct
# Should show a Z state, <defunct> process — the zombie

# Wait for the parent script to finish — zombie should clear
sleep 5
ps aux | grep -i defunct
# Should now be empty — parent exited, zombie was reaped
```

**Paste the `ps aux | grep -i defunct` output showing the zombie,
and confirm it clears after the parent exits.**

---

### Task 3 — Signals: SIGTERM vs SIGKILL

**Run on Ubuntu, then Rocky9:**

```bash
# Start a process that ignores SIGTERM, to observe the difference
cat << 'EOF' > /tmp/stubborn.sh
#!/bin/bash
trap '' SIGTERM
echo "PID $$ ignoring SIGTERM, running..."
while true; do sleep 1; done
EOF
chmod +x /tmp/stubborn.sh

/tmp/stubborn.sh &
STUBBORN_PID=$!
echo "Stubborn process PID: $STUBBORN_PID"

# Try SIGTERM first — it will NOT stop this process (it traps SIGTERM)
kill $STUBBORN_PID
sleep 2
ps -p $STUBBORN_PID
# Still running — SIGTERM was caught and ignored

# Escalate to SIGKILL — cannot be caught or ignored
kill -9 $STUBBORN_PID
sleep 1
ps -p $STUBBORN_PID
# Process is gone — SIGKILL cannot be trapped
```

**Paste the `ps -p` output before and after the `kill -9`,
confirming SIGTERM failed and SIGKILL succeeded.**

---

### Task 4 — Background jobs and the SSH disconnect problem

**Run on Ubuntu, then Rocky9** (simulate the SSH disconnect by
closing your terminal/session, not by actually losing your real
connection):

```bash
# WITHOUT nohup — this job dies when the shell exits
sleep 120 &
echo "Started without nohup, PID: $!"
jobs
# Close this terminal session now, then reconnect and check:
ps -ef | grep sleep
# The sleep process should be GONE — it died with the shell

# WITH nohup — this job survives shell exit
nohup sleep 120 &
echo "Started with nohup, PID: $!"
# Close this terminal session now, then reconnect and check:
ps -ef | grep sleep
# The sleep process should STILL be running, now re-parented to PID 1
ps -ef | grep $(pgrep sleep) 
# Confirm PPID is now 1
```

**Paste your `ps -ef | grep sleep` output from both tests —
showing the process gone without `nohup` and still running, with
PPID 1, when started with `nohup`.**

---

### Task 5 — Priority with nice and renice

**Run on Ubuntu, then Rocky9:**

```bash
# Start a process at default priority
sleep 100 &
DEFAULT_PID=$!
ps -el | grep $DEFAULT_PID
# Note the NI column — should be 0

# Start a NEW process at lower priority
nice -n 10 sleep 100 &
NICE_PID=$!
ps -el | grep $NICE_PID
# NI column should show 10

# Change the priority of the ALREADY RUNNING default process
renice -n 15 -p $DEFAULT_PID
ps -el | grep $DEFAULT_PID
# NI column should now show 15

# Attempt to raise priority into negative — requires root
renice -n -5 -p $DEFAULT_PID
# Without sudo: should fail with a permission error
sudo renice -n -5 -p $DEFAULT_PID
# With sudo: should succeed
ps -el | grep $DEFAULT_PID
```

**Paste the `ps -el` output at each stage, and the permission
error from the non-root `renice -n -5` attempt.**

---

### Task 6 — Diagnosing a runaway process with top and htop

**Run on Ubuntu, then Rocky9:**

```bash
# Install htop
sudo apt install -y htop      # Ubuntu
sudo dnf install -y htop      # Rocky9

# Create a deliberate CPU-burning process to observe
cat << 'EOF' > /tmp/cpu-burn.sh
#!/bin/bash
while true; do :; done
EOF
chmod +x /tmp/cpu-burn.sh
/tmp/cpu-burn.sh &
BURN_PID=$!
echo "CPU-burning process PID: $BURN_PID"

# Observe it in top — sort by CPU (press P), find the PID
top
# Note its %CPU — should be near 100% of one core

# Observe it in htop — try the tree view (F5)
htop

# Clean up — kill the runaway process
kill $BURN_PID
sleep 1
ps -p $BURN_PID
# Should be gone
```

**Paste a screenshot or copy of the `top` line showing the
CPU-burning process, noting its PID and %CPU.**

---

## What You Learned

- ✅ Explained what a process is, read PID/PPID relationships, and navigated the process tree with `ps` and `pstree`
- ✅ Identified every process state (`R`, `S`, `D`, `T`, `Z`) and explained what a zombie process actually is and how it's cleared
- ✅ Sent the correct signal for the situation, and explained why `SIGTERM` is the default and `SIGKILL` is a last resort
- ✅ Found and killed processes correctly with `kill`, `killall`, and `pkill -f`
- ✅ Managed foreground/background jobs and fixed the SSH-disconnect problem with `nohup` and `disown`
- ✅ Set and changed process priority with `nice`/`renice`, and understood why lower means higher priority
- ✅ Diagnosed a runaway process by CPU/memory using `top` and `htop`
- ✅ Connected process-held file handles back to Demo 02's `lsof` pattern, and confirmed a service account's process behaves identically to any other process despite `nologin` restricting only interactive login (Demo 05)

**Key Takeaway:** Almost every process-management incident is really one of three questions: is this process in a state it should be in, did it receive the signal you think it received, and does it have the parent/priority relationship you expect. `ps`, `pstree`, and `top`/`htop` answer all three — learn to reach for them before guessing.

---

## Part 3 — Break-Fix Scenario

### Scenario — NovaTech Ops: two process management incidents

> *"Incident 1: Someone ran a 40-minute deployment script over SSH
> last night. Their laptop went to sleep and the connection dropped
> around the 15-minute mark. This morning the deployment is
> half-finished and the process is gone."*

> *"Incident 2: A batch job is pinned at 100% CPU and has been for
> 20 minutes — way longer than it should take. We tried `kill` on
> it twice. It's still running."*

**Investigation — Incident 1:**

```bash
# Checking what was run and how
history | grep deploy
# ssh ops@server "cd /opt/app && ./deploy.sh"
# No nohup, no & even — this was a plain foreground SSH command

# Confirm the process is gone
ps -ef | grep deploy
# (nothing — confirmed gone)
```

**Investigation — Incident 2:**

```bash
ps -ef | grep batch-job
# wadmin  3301  1  98  09:40 ?  00:20:15 /opt/app/batch-job

kill 3301
sleep 3
ps -p 3301
# Still running — kill (SIGTERM) had no effect

kill 3301
sleep 3
ps -p 3301
# Still running after second attempt too
```

**Your tasks:**

1. For Incident 1: why did the deployment die with the SSH
   disconnect, and what should have been done differently when
   starting it?

2. For Incident 2: SIGTERM was sent twice with no effect. What
   are the possible explanations, and what is the correct next
   step?

3. After fixing Incident 2, what command would confirm the
   process is actually gone, not just unresponsive to `ps` for a
   moment?

<details>
<summary>Resolution — open after attempting</summary>

**Incident 1 — root cause:**

The deployment script was run as a plain foreground SSH command —
no `&`, no `nohup`. The process was a direct child of the SSH
session's shell. When the laptop slept and the SSH connection
dropped, the kernel sent `SIGHUP` to every process attached to
that now-dead session, including the deployment script — and it
had no protection against that signal, so it terminated
immediately, mid-deployment.

**What should have been done:**
```bash
# Correct from the start:
nohup ./deploy.sh > deploy.log 2>&1 &
# This survives a dropped connection entirely
```

Or, for long-running interactive work, a terminal multiplexer
(`tmux`/`screen`) is the more complete production answer — the
session itself survives a disconnect, not just one command — but
that's beyond this demo's scope; `nohup` is the minimum correct
fix for a single script.

**Incident 2 — why SIGTERM failed twice, and the correct
escalation:**

Two possible explanations, both real:
1. The process is **trapping SIGTERM** and either ignoring it or
   stuck in a handler that never completes (as demonstrated in
   Task 3 of this demo)
2. The process is in **uninterruptible sleep (D state)**, usually
   waiting on disk I/O — in this state, a process cannot respond
   to *any* signal, including SIGTERM, until the I/O operation
   completes or times out

```bash
ps -o pid,stat,cmd -p 3301
# Check the STAT column — if it shows D, this is an I/O-blocked
# process, not one ignoring the signal on purpose
```

**Correct next step — escalate to SIGKILL, but verify first:**

```bash
# If STAT is D: SIGKILL also won't work immediately — you may need
# to investigate the underlying I/O issue (disk, NFS mount, etc.)
# rather than just escalating signals

# If STAT is R/S and it's simply trapping SIGTERM:
kill -9 3301
sleep 2
ps -p 3301
```

**Question 3 — confirming the process is actually gone:**

```bash
ps -p 3301
# No output / "no such process" confirms it
# Alternative, more thorough check:
kill -0 3301
echo $?
# kill -0 sends no signal, just checks if the PID exists
# Exit code 0 = process still exists, non-zero = it's gone
```

</details>

---

## Interview Prep

**Q1. A process shows `Z` state in `ps` and `<defunct>` in the
command column. The team wants to `kill -9` it to clean it up.
Will that work, and what's the actual fix?**

No — `kill -9` will not work on a zombie, and trying it accomplishes
nothing. A zombie process has already finished executing; there's
no running program to signal, only a leftover entry in the process
table holding its exit status, waiting for its parent to call
`wait()`. The fix targets the **parent**, not the zombie: either
the parent's code needs to correctly reap its children, or if the
parent is gone/unresponsive, killing or restarting the parent
causes the kernel to re-parent the zombie to PID 1, which reaps it
immediately.

---

**Q2. Why does a background job started with `command &` die when
an SSH session disconnects, and how do you prevent it?**

A background job started with plain `&` is still a child of the
shell. When the shell exits — including when SSH disconnects — the
kernel sends `SIGHUP` to every process still attached to that
shell, and the job dies unless it's specifically protected against
that signal. `nohup command &` makes the process ignore SIGHUP from
the start. If the job is already running without `nohup`, `disown
-h %1` detaches it from the current shell's job table so it no
longer receives SIGHUP when the shell exits — though by that point
output redirection can't be added retroactively the way `nohup`
provides it.

---

**Q3. Why is `kill -9` (SIGKILL) considered a last resort rather
than a default way to stop a process?**

`SIGTERM` (the default signal `kill` sends) gives a process the
chance to shut down cleanly — close open files, flush buffered
writes, finish an in-flight transaction, deregister from a load
balancer. `SIGKILL` removes the process immediately with zero
chance to clean up, which can leave data files half-written or
locks unreleased. The correct pattern is to send `SIGTERM` first,
wait a reasonable amount of time, and only escalate to `SIGKILL`
if the process is still running — which usually means it's hung,
ignoring/trapping the signal, or stuck in an uninterruptible state.

---

**Q4. You renice a process to `-10` as a regular (non-root) user
and it fails. Why, and what's the fix?**

Only root can set a process's niceness into negative values
(higher priority than default). Any user can lower their own
process's priority (raise the nice value toward +19), but raising
it requires elevated privileges — this prevents a regular user
from starving other users' processes of CPU time by simply
prioritizing their own work above everyone else's. Fix: `sudo
renice -n -10 -p PID`.

---

## Key Takeaways

1. **Every process has exactly one parent, tracing back to PID 1 — `ps -ef` shows PPID directly, and `pstree -p` shows the whole tree visually.** When a parent exits before its child, the child is orphaned and re-parented to PID 1, not killed.

2. **A zombie (`Z` state, `<defunct>`) is already dead — there's nothing left to signal.** Fix the parent (make it call `wait()`, or kill/restart it) — never try to `kill` the zombie itself.

3. **`SIGTERM` (the default) asks a process to clean up and exit; `SIGKILL` (`-9`) removes it instantly with zero chance to clean up.** Always try `SIGTERM` first and escalate to `SIGKILL` only when the process is confirmed hung or unresponsive — an uninterruptible-sleep (`D` state) process won't respond to either until its I/O completes.

4. **A background job (`&`) is still a child of your shell — it receives `SIGHUP` and dies when the shell exits, including on SSH disconnect.** Use `nohup command &` from the start, or `disown -h %1` to protect a job already running.

5. **Niceness runs from -20 (highest priority) to +19 (lowest) — lower number means higher priority, the opposite of what the scale "feels" like.** Any user can lower their own process's priority; only root can raise it into negative values.

6. **`top` and `htop` show live, sorted process data — `htop` adds color, a tree view, and in-app signal/renice actions, but isn't installed by default on either distro.** Reach for these first when diagnosing a runaway process, before guessing at a PID.

7. **A deleted-but-still-allocated file (Demo 02) and a service account's running process (Demo 05) are both fundamentally process-management facts** — `lsof` ties a held-open file back to the PID that must be restarted to free it; `nologin` restricts an account's login capability, not its process's normal behavior once systemd starts it.

---

## Quick Commands Reference

### Viewing processes

| Command | Purpose |
|---|---|
| `ps aux` | All processes, BSD-style columns (`%CPU`, `%MEM`) |
| `ps -ef` | All processes, UNIX-style columns (`PPID`) |
| `pstree -p` | Visual parent/child tree with PIDs |
| `pgrep name` | Find PIDs matching a name |
| `pgrep -fl "cmd"` | Find PIDs matching full command line, with name |

### Signals and killing

| Command | Purpose |
|---|---|
| `kill PID` | Send `SIGTERM` (default, graceful) |
| `kill -9 PID` | Send `SIGKILL` (forceful, last resort) |
| `kill -SIGHUP PID` | Send a named signal |
| `kill -l` | List all signal names/numbers |
| `kill -0 PID` | Check if a PID still exists (no signal sent) |
| `killall name` | Kill all processes matching exact name |
| `pkill -f "cmd"` | Kill by matching full command line |

### Jobs and backgrounding

| Command | Purpose |
|---|---|
| `command &` | Run in background (child of current shell) |
| `jobs` | List background jobs in this shell |
| `fg %N` | Bring job N to foreground |
| `bg %N` | Resume stopped job N in background |
| `nohup command &` | Run in background, immune to `SIGHUP` |
| `disown -h %N` | Detach already-running job N from shell |

### Priority

| Command | Purpose |
|---|---|
| `nice -n N command` | Start a new process at niceness N |
| `renice -n N -p PID` | Change niceness of a running process |

### Monitoring

| Command | Purpose |
|---|---|
| `top` | Live process monitor (default-installed) |
| `htop` | Enhanced live process monitor (requires install) |

---

## Appendix A — Anki Cards

**`06-process-management-anki.csv`:**

```
#deck:Linux Mastery::Module 2 - System Administration::Demo 06 - Process Management
#separator:Comma
#columns:Front,Back,Tags

"ps aux shows a process in Z state with <defunct> in the command column. The team wants to kill -9 it. Will that work?","No. A zombie process has already finished executing - there is no running program to signal, only a leftover process-table entry holding its exit status, waiting for its parent to call wait(). kill -9 has nothing to act on. The fix targets the PARENT: either it needs to correctly reap children, or if it's gone/unresponsive, killing/restarting it causes the kernel to re-parent the zombie to PID 1, which reaps it immediately.","demo06,zombie,signals"

"Why does kill PID try SIGTERM first instead of going straight to kill -9?","SIGTERM (the default) gives the process a chance to shut down cleanly - close open files, flush buffered writes, finish an in-flight transaction, deregister from a load balancer. SIGKILL (-9) removes the process instantly with zero chance to clean up, which can leave data files half-written or locks unreleased. Correct pattern: SIGTERM first, wait a few seconds, escalate to SIGKILL only if still running.","demo06,signals,sigterm,sigkill"

"A background job (command &) dies the moment an SSH session disconnects. Why, and how do you prevent it?","A background job started with & is still a child of the shell. When the shell exits - including on SSH disconnect - the kernel sends SIGHUP to every process still attached to it, and the job dies unless protected. Fix: nohup command & from the start (ignores SIGHUP), or disown -h %1 to detach an already-running job from the shell's job table.","demo06,nohup,disown,sighup"

"What does niceness -20 vs +19 mean, and who can set which values?","-20 = highest priority (most CPU time). +19 = lowest priority ('nice' to other processes, willing to wait). Lower number = higher priority - the opposite of what the scale feels like. Any user can LOWER their own process's priority (raise nice value toward +19). Only root can RAISE priority into negative values - this stops a regular user from starving others of CPU.","demo06,nice,renice,priority"

"kill PID is sent twice on a runaway process and it's still running both times. What are the two most likely explanations?","(1) The process is trapping/ignoring SIGTERM in its own code - confirm with ps -o pid,stat,cmd and consider escalating to kill -9, which cannot be caught or ignored. (2) The process is in uninterruptible sleep (D state), usually waiting on disk I/O - in this state it cannot respond to ANY signal, including SIGKILL, until the I/O completes. Check STAT column before assuming it's just ignoring you.","demo06,signals,troubleshooting,d-state"

"What is the difference between killall nginx and pkill -f \"python app.py\"?","killall matches by exact process NAME only - killall python would kill every Python process on a shared server regardless of which script it runs, which is dangerous. pkill -f matches against the FULL COMMAND LINE, so pkill -f \"python app.py\" only kills processes that were started with that exact script, leaving other Python processes alone.","demo06,pkill,killall,safety"

"A deleted log file is still consuming disk space according to df, but doesn't show up in du. What process-management fact explains this, and what's the fix?","A deleted file's directory entry is gone (invisible to du), but if a process still has the file open, the kernel keeps the inode and data blocks allocated (still counted by df). lsof | grep deleted shows exactly which PID is holding it open. Fix: kill (SIGTERM first) the process holding the file to release the handle and actually free the disk space.","demo06,lsof,deleted-files,signals"

"A service account created with /usr/sbin/nologin is running a process under systemd. Does nologin prevent ps/kill/renice from working on that process?","No. /usr/sbin/nologin restricts the ACCOUNT's ability to be used for interactive login (ssh, su, console) - it has no effect on a process's behavior once it's running. A service account's process started by systemd (PPID 1) is a completely normal process: visible in ps/top, killable, reniceable, subject to every process-management concept exactly the same as a process started by a logged-in human user.","demo06,service-account,nologin,ps"
```

---

## Appendix B — Quiz

**`06-process-management-quiz.md`:**

````markdown
# Quiz — Demo 06: Process Management

> One correct answer per question unless stated otherwise.
> Target: 80% or above before moving to Demo 07.

---

**Q1.** `ps aux` shows a process with `STAT` = `Z` and `<defunct>`
in the command. Your teammate runs `kill -9` on it twice with no
effect. Why doesn't it work, and what's the actual fix?

- A) `kill -9` requires root for zombie processes specifically
- B) The zombie has already finished executing — there's nothing
  left to signal. Fix the parent process so it calls `wait()`,
  or kill/restart the parent so the kernel re-parents and reaps
  the zombie via PID 1.
- C) Zombies must be killed with `SIGTERM`, not `SIGKILL`
- D) The PID number must be wrong — zombies don't have real PIDs

<details>
<summary>Answer</summary>

**B** — A zombie is already dead; only a process-table entry with
its exit status remains, waiting for the parent to collect it via
`wait()`. No signal can affect a process with nothing left to
receive it. The fix always targets the parent, never the zombie
itself.

</details>

---

**Q2.** Why is `SIGTERM` the default signal `kill` sends, rather
than `SIGKILL`?

- A) `SIGTERM` is faster to process by the kernel
- B) `SIGTERM` lets the process clean up — close files, flush
  writes, finish transactions — before exiting. `SIGKILL` gives
  it no chance to do any of that.
- C) `SIGKILL` is deprecated on modern Linux kernels
- D) There is no real difference; `kill` and `kill -9` behave identically

<details>
<summary>Answer</summary>

**B** — `SIGTERM` is catchable, allowing graceful shutdown.
`SIGKILL` cannot be caught, blocked, or ignored — the kernel
removes the process immediately with zero cleanup, which can
corrupt in-progress writes or leave locks held. Always try
`SIGTERM` first; escalate to `SIGKILL` only when confirmed hung.

</details>

---

**Q3.** A deployment script is run as a plain foreground SSH
command with no `&` and no `nohup`. The SSH connection drops
mid-deployment. What happens, and why?

- A) Nothing — the script keeps running on the server regardless
  of the SSH connection
- B) The script is paused (`SIGSTOP`) until reconnection
- C) The kernel sends `SIGHUP` to every process attached to the
  now-dead shell, including the script, which terminates
  immediately unless it specifically ignores that signal
- D) The script restarts automatically from the beginning

<details>
<summary>Answer</summary>

**C** — The script is a child of the SSH session's shell. When the
shell exits (including on disconnect), the kernel sends `SIGHUP`
to its children. Without `nohup` (which makes a process ignore
`SIGHUP`) or `disown`, the process dies along with the shell.

</details>

---

**Q4.** You `renice -n -10 -p 3301` as a regular (non-root) user
and it fails with a permission error. Why?

- A) `renice` cannot target running processes, only new ones
- B) Negative nice values (higher priority) require root — any
  user can lower their own process's priority, but raising it
  into negative territory needs elevated privileges
- C) PID 3301 does not exist
- D) `renice` only works on processes you have not started yourself

<details>
<summary>Answer</summary>

**B** — Any user can raise their own process's nice value (lower
priority). Only root can set a negative nice value (raise
priority above default), preventing a regular user from starving
other users' processes of CPU time. Fix: `sudo renice -n -10 -p 3301`.

</details>

---

**Q5.** `kill PID` is sent twice on a runaway process with no
effect — it's still running both times. `ps -o pid,stat,cmd -p PID`
shows `STAT` = `D`. What does this tell you?

- A) The process is sleeping normally and will respond shortly
- B) The process is in uninterruptible sleep, usually waiting on
  disk I/O — it cannot respond to ANY signal, including SIGKILL,
  until the I/O operation completes
- C) `D` means the process is already dead
- D) `D` means debug mode is enabled on the process

<details>
<summary>Answer</summary>

**B** — `D` (uninterruptible sleep) means the process is blocked
on I/O and cannot process signals at all in this state — not even
`SIGKILL`. Escalating signals won't help; the underlying I/O issue
(disk, network mount, etc.) needs investigation.

</details>

---

**Q6.** What is the key difference between `killall python` and
`pkill -f "python app.py"`?

- A) They are functionally identical
- B) `killall` only works on Ubuntu, `pkill` only on Rocky9
- C) `killall` matches by exact process name only — it would kill
  every Python process on the system. `pkill -f` matches the full
  command line, only killing processes started with that exact script.
- D) `pkill` requires root, `killall` does not

<details>
<summary>Answer</summary>

**C** — `killall python` is dangerous on a shared system because
it kills all processes named `python`, regardless of what they're
running. `pkill -f "python app.py"` matches against the full
command line, targeting only that specific script.

</details>

---

**Q7.** A service account was created with `/usr/sbin/nologin` per
Demo 05's correct pattern. Does this prevent its running process
from appearing in `ps` or being killed/reniced normally?

- A) Yes — `nologin` accounts are invisible to `ps`
- B) Yes — `nologin` processes cannot be killed, only systemd can stop them
- C) No — `nologin` only restricts interactive login (ssh, su,
  console). The process itself, once started by systemd, behaves
  exactly like any other process: visible, killable, reniceable.
- D) No — but `kill` requires the `-f` flag for `nologin` accounts

<details>
<summary>Answer</summary>

**C** — `/usr/sbin/nologin` is a login-shell restriction, not a
process-behavior restriction. A service account's process started
by systemd (typically PPID 1) is fully visible and manageable with
every standard process-management tool, identical to a process
started by an interactively logged-in user.

</details>

---

| Score | Action |
|---|---|
| 7/7 | Import Anki cards, move to Demo 07 |
| 6/7 | Review the wrong answer, then proceed |
| 5/7 | Re-read the relevant section, retry those questions |
| Below 5/7 | Re-read the full demo and redo the walkthrough before proceeding |
````