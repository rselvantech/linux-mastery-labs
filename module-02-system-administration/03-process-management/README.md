# Demo 06 — Process Management

## Overview

Every running program on a Linux system is a process — from the
kernel's own init process down to the `curl` command you just ran.
Understanding how processes relate to each other, what state they
are in, how to communicate with them via signals, and how to
control their priority and lifecycle is core to running anything
in production: a hung deployment, a runaway process eating CPU, a
background job that silently died when your SSH session dropped —
all of these are process management problems with process
management solutions.

This demo covers the process tree, process states, signals (the
correct way to stop a process and why `kill -9` is a last resort
not a default), controlling terminals and how they attach to
processes, foreground/background job control, process priority
with `nice`/`renice`, and real-time monitoring with `top` and
`htop`.

---

## Demo Objectives

By the end of this demo you will be able to:

1. Explain what a process is, read PID/PPID relationships, and
   navigate the process tree
2. Identify every process state (`R`, `S`, `D`, `T`, `Z`) and
   all modifier characters (`s`, `<`, `N`, `l`, `+`) from `ps`
   output, and explain what a zombie process actually is
3. Explain what a controlling terminal is, how processes attach
   to and detach from one, and why detachment is required for
   background services
4. Send the correct signal for the situation — explain why
   `SIGTERM` is the default and `SIGKILL` is a last resort, and
   what happens when a process has no signal handler registered
5. Find and kill processes correctly with `kill`, `killall`,
   and `pkill`
6. Manage foreground and background jobs — `&`, `jobs`, `fg`,
   `bg` — and explain why a background job dies when an SSH
   session disconnects unless `nohup` or `disown` is used
7. Set and change process priority with `nice` and `renice`,
   and read the `NI` and `PRI` columns in `ps -el` output
8. Diagnose a runaway process by CPU/memory using `top` and
   `htop`
9. Connect process-held file handles back to the `lsof` pattern
   from Demo 02, and explain why a service account's process
   still appears in `ps` even though the account cannot log in
   interactively (Demo 05)

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
PID    Process ID — a unique number identifying this process.
       Assigned sequentially, wraps around at the system maximum
       (typically 32768 or 4194304). Once a process exits its PID
       is freed and may be reused.

PPID   Parent Process ID — the PID of the process that started
       this one. Critical for tracing "who launched this?"

UID    User ID the process runs as — determines its permissions
       (same UID concept from Demo 05's /etc/passwd). A process
       running as UID 1000 has exactly the same file access as
       the user with UID 1000.
```

```bash
ps -ef | head -5
# UID    PID  PPID  C STIME TTY          TIME CMD
# root     1     0  0 09:00 ?        00:00:34 /sbin/init
# root   412     1  0 09:00 ?        00:04:02 /usr/lib/systemd/systemd-journald
# wadmin 1823  1801 0 09:15 pts/0    00:00:00 bash
# wadmin 1901  1823 0 09:20 pts/0    00:00:00 ps -ef
```

Reading this: PID 1 (`init`/`systemd`) has PPID 0 — the kernel
itself, not a real process. `bash` (PID 1823) is the parent of
the `ps` command itself (PID 1901, PPID 1823) — typing a command
in your shell makes your shell the parent of that command.

**Real-world scenario — "who started this mystery process?"**

```bash
# Find a suspicious high-CPU process
ps aux --sort=-%cpu | head -3
# wadmin  4521 40.1  0.3  /tmp/.hidden/miner

# Trace the full ancestry chain
pstree -s 4521
# systemd(1)───cron(163)───sh(4519)───miner(4521)
# Launched by cron — check the crontab immediately:
crontab -l -u wadmin
sudo crontab -l
```

**When a parent exits before its child:** the child is
**orphaned** and gets re-parented to PID 1 automatically — it
does not die. This is the mechanism that lets `nohup` jobs
survive a shell exit (section 1.6).

### 1.2 Viewing processes — ps, pgrep, pstree

**`ps`** = **p**rocess **s**tatus. Snapshot of running processes
at the moment you run it — not live, not auto-updating.

**Two styles you will use constantly, both universal:**

```
ps aux        BSD-style flags (no leading dash)
  a = show processes from ALL users, not just yours
  u = USER-oriented format — shows %CPU, %MEM, USER column
  x = include processes NOT attached to a terminal
      (without x: daemons — nginx, sshd, systemd — are all
       hidden because they have no controlling terminal)

ps -ef        UNIX-style flags (with leading dash)
  -e = every process (equivalent to aux combined)
  -f = full format — adds PPID column, critical for tracing
       parent/child relationships

ps -el        Long format — adds NI (nice/priority) columns
  -l = long — adds F (flags), PRI (kernel priority), NI (nice
       value), WCHAN (what the process is waiting on)
       Use this when diagnosing process priority (section 1.7)
```

**When to use which:**

```
ps aux       Resource usage — %CPU and %MEM shown directly.
             First choice for "what's eating my CPU/memory?"

ps -ef       Parent/child relationships — PPID shown directly.
             First choice for "what started this process?"

ps -el       Priority diagnosis — NI and PRI columns shown.
             Use when working with nice/renice (section 1.7)

ps aux --sort=-%cpu | head -10
             Production first response: top CPU consumers ranked
```

**Output columns explained — using your actual terminal output:**

```
ps aux columns:
USER    PID   %CPU  %MEM    VSZ    RSS   TTY   STAT  START   TIME  CMD
root      1    0.0   0.0  21884  13232   ?     Ss    Jun28  0:34  /sbin/init
root      2    0.0   0.0   3172   2148  hvc0   Sl+   Jun28  0:00  /init
root     39    0.0   0.1  83236  19696   ?     S<s   Jun28  4:02  /usr/lib/systemd/systemd-journald

VSZ    Virtual Size — all memory mapped including not-yet-loaded
       pages. Often much larger than what's actually in RAM.
RSS    Resident Set Size — RAM actually in use right now (KB).
       This is the real memory consumption number.
%CPU   Percentage of one CPU core. Can exceed 100% on multi-core
       systems if a process uses multiple cores simultaneously.
TTY    Controlling terminal. ? = no terminal (daemon). pts/N =
       pseudo-terminal from an SSH or local terminal session.
TIME   Total CPU time consumed since the process started — not
       wall-clock time. A process running for days but idle shows
       low TIME.

ps -ef additional columns:
PPID   Parent PID — use this to trace who started a process.
       Every daemon shows PPID 1 (started by systemd).
C      CPU scheduling counter — rough instantaneous CPU % (low
       precision, use %CPU from ps aux for accuracy).
```

**`pgrep`** = **p**rocess **grep**. Searches processes by name or
pattern, returns PIDs. Cleaner than `ps | grep` because it never
includes the grep process itself in results.

```
pgrep name              Print PIDs matching process name
  -l  list              Also print the process name
  -f  full command line Match against the full command string,
                        not just the process name
  -u  user              Match only processes owned by this user
  -P  parent PID        Find all direct children of a parent PID
  -o  oldest            Return only the oldest (first) matching PID
  -n  newest            Return only the newest matching PID
  -a  all info          Print full command line with PID
```

**Real-world example — finding and verifying before killing:**

```bash
# ALWAYS verify what pgrep matches BEFORE killing
pgrep -fl "python app.py"
# 3301 python app.py   ← confirm this is what you want
# 3302 python app.py   ← and how many instances

# Then kill with confidence
pkill -f "python app.py"
```

**`pstree`** = **p**rocess **tree**. Displays parent/child
relationships visually. Unfiltered output on a busy server is too
long to be useful — always filter to a specific PID or use case.

```
pstree -p              Show all with PIDs (use only for learning
                       the format — unfiltered is too long to read
                       on any real server)
pstree -p PID          Show only the subtree rooted at this PID
pstree -s PID          Show the ANCESTORS of PID — traces upward
                       to PID 1. Most useful production command.
pstree -u username     Show only processes owned by this user
pstree -p -u username  User's processes with PIDs
```

**Real-world use cases — pstree is only useful when filtered:**

```bash
# "Who started this process and through what chain?"
# Most useful pstree command in production:
pstree -s 3301
# systemd(1)───sshd(1650)───bash(1823)───batch-job(3301)
# Immediately tells you: SSH session → bash shell → launched manually

# "What did nginx spawn?"
pstree -p $(pgrep -o nginx)
# nginx(2001)─┬─nginx(2002)
#             └─nginx(2003)

# "What is this specific user running?"
pstree -p -u deployer
```

**Finding a parent process — four methods:**

```bash
# Method 1: ps -ef, read PPID column (most common)
ps -ef | grep 3301
# root  3301  1823  ...   ← PPID is 1823

# Method 2: pstree -s (most readable for full ancestry chain)
pstree -s 3301

# Method 3: read directly from the kernel's /proc filesystem
cat /proc/3301/status | grep PPid
# PPid:  1823

# Method 4: $PPID in bash (your current shell's parent only)
echo $PPID
```

### 1.3 Process states — what each letter means

Every process is in exactly one state at any moment. `ps` shows
this as a letter in the `STAT` or `S` column.

**Primary state letters:**

```
R   Running or runnable — actively using CPU or waiting for CPU
    time. A single-core system can only have one truly Running
    process at once; all others wanting CPU are also R (runnable).

S   Sleeping (interruptible) — waiting for an event: input, a
    timer, a network response, a lock. Most idle processes sit
    here. Can be woken by a signal.

D   Uninterruptible sleep — waiting on I/O (usually disk or
    network filesystem). Cannot be interrupted even by a signal.
    Normal briefly; concerning if stuck here for minutes
    (usually indicates disk or NFS storage trouble).

T   Stopped — paused by SIGSTOP or job control (Ctrl+Z), not
    running at all. Resumes with SIGCONT or `fg`.

Z   Zombie — process has exited but its parent hasn't called
    wait() to collect its exit status. No memory or CPU used —
    only a process table entry remains.

I   Idle kernel thread — only seen on kernel-internal processes.
```

**Modifier characters shown after the primary letter:**

Your real `ps aux` output showed states like `Ss`, `Sl+`, `S<s`,
`SN` — these combine the primary state with modifier characters:

```
s    Session leader — this process leads a process group/session.
     The main daemon process, not its workers.
     Your output: sshd, cron, systemd-journald all show Ss.

<    High priority — nice value is negative, getting MORE CPU
     time than default.
     Your output: S<s for systemd-journald (kernel elevated it).

N    Low priority — nice value is positive, getting LESS CPU
     time than default.
     Your output: SN for background sleep 60 started with &
     (shell deprioritizes background jobs slightly).

l    Multi-threaded — has multiple kernel threads.
     Your output: Ssl, Sl+ for systemd-timesyncd.

+    In the foreground process group of its terminal — actively
     connected to a terminal in the foreground.
     Your output: S+ for the grep you ran interactively.
```

**Reading your actual ps aux output:**

```
root     1   Ss   → systemd: sleeping, session leader
root     2   Sl+  → /init (WSL2 bridge): multi-threaded,
                    foreground of hvc0 (hypervisor console)
root    39   S<s  → systemd-journald: sleeping, HIGH PRIORITY,
                    session leader (kernel elevated log daemon)
systemd+154  Ssl  → systemd-timesyncd: sleeping, session leader,
                    multi-threaded
wadmin sleep SN   → your background sleep: low priority (N added
                    by shell for background jobs)
```

**What a zombie actually is, and why you cannot `kill` it:**

A zombie has already finished executing — no running program, no
memory in use, no CPU consumption. What remains is a single entry
in the kernel's process table holding the exit status, waiting for
the parent to call `wait()`. You cannot kill something already
dead — `kill` sends a signal to a running process; a zombie has
nothing to receive or act on it.

The fix always targets the **parent**: make it call `wait()`, or
kill/restart the parent so the kernel re-parents the zombie to
PID 1, which reaps it immediately. A small number of zombies that
clear quickly is normal; zombies that accumulate without clearing
indicate a real bug in the parent process.

### 1.4 Signals — communicating with a process

A **signal** is a message sent to a process by the kernel, by
another process, or by a user — asking it to do something:
terminate, pause, reload its configuration, etc.

**Default actions and signal handlers:**

Every signal has a **default action** built into the kernel that
applies if the process has NOT registered its own handler:

```
Signal    Num  Default action     Common handler override
──────────────────────────────────────────────────────────
SIGTERM   15   Terminate          Clean shutdown (close files,
                                  flush writes, finish transactions)
SIGKILL    9   Terminate          CANNOT be overridden — ever.
                                  Kernel enforces this directly.
SIGHUP     1   Terminate          Daemons (nginx, sshd) override
                                  this to reload config instead.
                                  Background jobs have no handler
                                  → they get the default: killed.
SIGINT     2   Terminate          Ctrl+C — programs may override
                                  to clean up before exiting.
SIGSTOP   19   Stop/pause         CANNOT be overridden — ever.
SIGCONT   18   Resume stopped     Resume after SIGSTOP.
SIGSEGV   11   Terminate +        Segmentation fault — program
               core dump          accessed invalid memory.
```

**Two signals can NEVER be caught, blocked, or handled:
`SIGKILL` (9) and `SIGSTOP` (19).** The kernel enforces these
directly, bypassing any handler. This is why `kill -9` always
works when `SIGTERM` doesn't.

**What happens if a program has no handler for a signal?**

The kernel's default action for that signal applies automatically.
Most programs only register handlers for `SIGTERM` (clean
shutdown) and `SIGHUP` (reload, if applicable). Everything else
gets the default. A plain shell script or background job you
start has no signal handlers registered — so every signal that
reaches it applies the default action.

**Why `SIGTERM` is the default and `kill -9` is a last resort:**

`SIGTERM` gives the process a chance to clean up — close open
file handles, flush buffered writes to disk, finish an in-flight
database transaction, deregister from a load balancer. `SIGKILL`
removes the process with zero cleanup. Killing a database process
with `SIGKILL` can leave data files half-written in a state that
`SIGTERM`'s clean shutdown would have avoided.

**Correct escalation path:**

```bash
kill PID           # SIGTERM — try this first, always
sleep 5            # give it time to shut down cleanly
kill -0 PID        # check if still alive (sends no signal)
echo $?            # 0 = still running, non-zero = gone
kill -9 PID        # SIGKILL — only if still running after wait
```

**Real-world troubleshooting — SIGTERM with no effect:**

Two distinct explanations, requiring different responses:

```
Explanation 1: Process is trapping SIGTERM
  → Its handler is either hung or running slowly
  → Check: ps -o pid,stat,cmd -p PID
  → If STAT shows S or R: handler is stuck → kill -9 is correct

Explanation 2: Process is in D state (uninterruptible sleep)
  → Waiting on I/O, cannot respond to ANY signal including SIGKILL
  → Check: ps -o pid,stat,cmd -p PID
  → If STAT shows D: killing won't help — investigate the I/O
     (disk failure, NFS mount hung, storage timeout)
  → SIGKILL will be queued but won't deliver until D state ends
```

### 1.5 Finding and killing processes — kill, killall, pkill

**`kill`** — sends a signal to a process by PID.

```
kill PID                  Send SIGTERM (15) — the default
kill -9 PID               Send SIGKILL — last resort
kill -SIGHUP PID          Send by name (more readable than number)
kill -1 PID               Send by number (1 = SIGHUP)
kill -l                   List all signal names and numbers
kill -0 PID               Check if PID exists — sends NO signal,
                          just tests existence. Exit code 0 = exists.
```

**`killall`** = **kill** **all**. Sends a signal to ALL processes
matching a name.

```
killall nginx             Kill all processes named exactly "nginx"
killall -9 nginx          Force-kill all matching processes
killall -u username       Kill all processes owned by this user
killall -i nginx          Interactive — prompt before each kill
```

**`pkill`** = **p**rocess **kill**. Like killall but matches
against pattern and can match the full command line.

```
pkill name                Kill by process name (like killall)
pkill -f "pattern"        Kill by matching FULL command line —
                          the critical flag for precision killing
pkill -u username         Kill all processes owned by this user
pkill -9 -f "pattern"     Force-kill by full command line match
```

**Why `pkill -f` matters — the precise kill:**

```bash
# DANGEROUS — kills EVERY python process on the system
killall python

# SAFE — kills only this specific script
pkill -f "python /opt/app/worker.py"

# ALWAYS verify the match before killing:
pgrep -fl "python /opt/app/worker.py"
# 3301 python /opt/app/worker.py   ← confirm exactly what matches
pkill -f "python /opt/app/worker.py"   # then kill
```

**Real-world scenario — safely stopping one of many workers:**

```bash
# 10 worker processes running, need to stop one specific one
ps aux | grep "worker"
# wadmin  3301  python /opt/app/worker.py --id=5
# wadmin  3302  python /opt/app/worker.py --id=6
# ...

# Stop only worker id=5
pkill -f "worker.py --id=5"

# Verify it's gone
pgrep -f "worker.py --id=5"
# (no output = confirmed gone)
```

### 1.6 Terminals, controlling TTY, and foreground vs background

**What is a controlling terminal, and how do processes attach to
one?**

Every interactive process has three standard file descriptors:

```
stdin   (fd 0)   Where the process reads input from
stdout  (fd 1)   Where the process writes normal output to
stderr  (fd 2)   Where the process writes error output to
```

When you run a command in a terminal, the terminal is the source
of stdin and destination of stdout/stderr. This connection is the
**controlling terminal** (TTY — TeleTYpewriter, the historical
name that stuck). You see it in `ps` output:

```
ps aux TTY column:
pts/7   = pseudo-terminal N (from SSH or a terminal emulator)
          this process is attached to your session
?       = NO controlling terminal
          this process is a daemon, detached from any session
          ALL properly running services should show ?
hvc0    = hypervisor virtual console (WSL2 or VM-specific)
```

**How a process gets attached:** when you open a terminal session,
the kernel creates a pseudo-terminal pair (`pts/N`). Your shell is
attached to it as the controlling terminal. Any command you run
from that shell **inherits** the same controlling terminal.

**How detachment works:**

```
Attached (default):        Process ←→ pts/7
                           Output appears in your terminal
                           SIGHUP sent if terminal closes
                           Process dies when terminal closes

Detached (nohup/daemon):   Process ←→ ? (no terminal)
                           Output redirected to file or /dev/null
                           No SIGHUP on terminal close
                           Survives terminal close, lives until
                           explicitly killed or exits naturally
```

**Foreground vs background:**

When you run a command normally, it runs in the **foreground** —
it holds your terminal's input/output and your shell waits for it
to finish before returning the prompt.

```bash
sleep 100          # foreground — terminal is blocked
                   # Ctrl+C sends SIGINT to interrupt it
                   # Ctrl+Z sends SIGSTOP to pause it (returns prompt)
```

**Running in the background — `&`:**

```bash
sleep 100 &
# [1] 2150        ← job number [1], PID 2150
# Prompt returns immediately

jobs               # list background jobs in this shell session
# [1]+  Running    sleep 100 &

fg %1              # bring job [1] back to foreground
bg %1              # resume a stopped job in the background
```

**The critical problem: background jobs and SSH disconnects.**

A background job started with plain `&` is still a **child of
your shell** and inherits the same controlling terminal. When your
shell exits — including when your SSH session disconnects — the
kernel sends SIGHUP to every process still attached to that
shell. SIGHUP's **default action** is to terminate the process,
and a plain shell script or background job has no SIGHUP handler
registered to override this. So the job dies.

This is NOT what SIGHUP does to daemons like nginx — nginx
registers a custom SIGHUP handler that reloads config instead of
terminating. Your background job has no such handler, so it gets
the default: killed.

**`nohup`** — the fix, started correctly from the beginning:

```
nohup     = no hang up
```

```bash
nohup ./deploy.sh > deploy.log 2>&1 &
# nohup registers a SIGHUP handler that IGNORES the signal
# > deploy.log 2>&1 redirects output (no terminal to write to
# after disconnect)
# & puts it in the background
```

`nohup` achieves detachment by registering a handler that ignores
SIGHUP. When your shell exits, the kernel still sends SIGHUP to
the process, but the process discards it and keeps running, then
gets re-parented to PID 1 (section 1.1).

**`disown`** — the fix for a job already running:

```bash
./deploy.sh &
# Forgot nohup — it's already running
disown -h %1
# -h = mark this job to not receive SIGHUP on shell exit
# Removes the job from the shell's job table without killing it
# Cannot retroactively redirect output — but protects from SIGHUP
```

**`nohup` vs `disown`:**

```
nohup command &   Use at START — ignores SIGHUP, redirects output
disown -h %N      Use AFTER start — protects from SIGHUP but cannot
                  redirect output that's already going to terminal
```

**Real-world scenario — the incomplete deployment:**

```bash
# WRONG — will die on disconnect
ssh ops@server "./deploy.sh"

# CORRECT — survives disconnect
ssh ops@server "nohup ./deploy.sh > /tmp/deploy.log 2>&1 &"

# Even better for long sessions:
ssh ops@server
tmux new-session -s deploy
./deploy.sh
# Ctrl+B then D to detach — tmux session survives any disconnect
```

### 1.7 Process priority — nice and renice

The kernel's scheduler decides which process gets CPU time. You
can influence — not dictate — this with a **niceness** value.

**The scale, and why lower means higher priority:**

```
-20   highest priority — most CPU time relative to others
  0   default — what every process starts at
+19   lowest priority — "nice" to other processes, yields CPU
```

The name comes from "being nice to others" — a high nice value
means willingness to wait. A low value means the process insists
on more CPU time. **Lower number = higher priority** — the most
common point of confusion here.

**`nice`** = start a new process at a given priority level.
**`renice`** = change the priority of an already running process.

```
nice -n N command         Start a NEW process at nice value N
  -n  number              The nice value (-20 to +19)
  Example: nice -n 10 ./backup.sh
  Use case: batch jobs, backups — lower priority so they don't
  compete with interactive work

renice -n N -p PID        Change priority of a RUNNING process
  -n  number              New nice value
  -p  pid                 Target process by PID
  -u  user                Change all processes owned by this user
  -g  group               Change all processes in this group
  Example: sudo renice -n -5 -p 2150
```

**Permission rules:**

```
Any user:    Can LOWER their own process priority (raise nice value
             from 0 toward +19)
Root only:   Can RAISE priority into negative values (-1 to -20)
             This prevents regular users from starving others of CPU
```

**Reading priority in `ps -el` output — your actual output:**

```
ps -el columns relevant to priority:
F   S   UID    PID   PPID  C   PRI  NI   ADDR  SZ    WCHAN    TTY    TIME  CMD
0   S   1000  8173   8145  0   80   0    -    1335  hrtime  pts/2  0:00  sleep
                               ↑    ↑
                               │    └─ NI (nice value): 0 = default priority
                               └─ PRI (kernel priority): 80 = standard default

NI = 0    Default priority — normal CPU share
NI = 10   Lower priority — gets less CPU than default
NI = -5   Higher priority — gets more CPU (requires root)

PRI = 80 + NI roughly (Linux uses a 0-139 internal scale;
80 = default, lower PRI number = higher actual priority in kernel)
You never set PRI directly — only NI through nice/renice.
```

**Real-world scenarios:**

```bash
# Backup script competing with production — lower its priority
nice -n 15 /opt/backup/run-backup.sh &

# Database query is time-critical — raise its priority
sudo renice -n -10 -p $(pgrep postgres | head -1)

# Lower all processes for a specific user
sudo renice -n 5 -u batch-user

# Confirm the change took effect
ps -el | grep PID
# NI column should show the new value
```

### 1.8 Monitoring with top and htop

**`top`** — installed by default on essentially every Linux
system. Live, auto-refreshing view of processes sorted by
resource usage.

```
top interactive keys:
  P    Sort by %CPU (often the default)
  M    Sort by %MEM
  k    Kill a process — prompts for PID and signal
  r    Renice — prompts for PID and new nice value
  u    Filter by user — prompts for username
  f    Fields management — add/remove columns
  1    Toggle per-CPU view (shows each core's usage separately)
  q    Quit
```

**Reading `top`'s header — the numbers that matter:**

```
top - 14:32:10 up 3 days, 2:15, 1 user,  load average: 0.45, 0.52, 0.38
                                          ↑             ↑     ↑     ↑
                                          │             │     │     └─ 15-min avg
                                          │             │     └─ 5-min avg
                                          │             └─ 1-min avg
                                          └─ load average = how many processes
                                             wanted CPU on average during the period

Load average interpretation:
  Number of CPU cores = 4
  Load avg 0.5 = system is at 12.5% capacity (well under)
  Load avg 4.0 = system is exactly at capacity (100%)
  Load avg 8.0 = system is overloaded (200% — processes queuing for CPU)
  Rule: load avg consistently > number of CPU cores = CPU bottleneck
```

**`htop`** — significant usability upgrade over `top`. Requires
install on most minimal server images.

```bash
sudo apt install -y htop      # Ubuntu
sudo dnf install -y htop      # Rocky9
```

```
htop key commands:
  F3 or /   Search — filter by process name instantly
  F4        Filter — narrow the display to matching processes
  F5        Tree view — shows parent/child relationships live
            (like pstree but updating in real time)
  F6        Sort by any column
  F9        Send signal — select from a list with arrow keys
            (no need to remember signal numbers)
  F10       Quit
```

**`top`/`htop` coverage note:** Full production-depth coverage
of `top` and `htop` — load average interpretation, per-core CPU
view, memory breakdown (buffers vs cache vs used), I/O wait
meaning, and `htop`'s advanced filtering — is deferred to
**Demo 09 — Live System Monitoring**, where they are
the primary tools alongside `vmstat`, `iostat`, and `free`. This
demo introduces them for the immediate need: diagnosing a runaway
process.

### 1.9 Connecting back — Demo 02 and Demo 05

**Demo 02's `lsof | grep deleted` pattern is a process
management fact:**

A deleted file stays allocated on disk specifically because some
**process** still holds it open. `lsof` ties file handles to the
PID holding them:

```bash
lsof | grep deleted
# myapp  2150  wadmin  4u  REG  8,1  104857600  /var/log/app.log (deleted)
# ↑       ↑
# process PID holding the file — restart this process to free space
```

Now that this demo has covered `kill`/`pkill`, the Demo 02 fix
("restart the process") is a concrete action: identify the PID
from `lsof`, then `kill -TERM PID` (then `kill -9` only if
needed) to release the handle and free the disk space.

**`lsof` will be covered in full detail in Demo 11** (Security
and Networking), where `lsof -i` (open network sockets), `lsof
-p PID` (all files a specific process has open), and `lsof -i
:8080` (what's listening on a port) are the primary diagnostic
tools.

**Demo 05's service accounts run as real processes:**

A service account created with `/usr/sbin/nologin` cannot be used
for interactive login. But the **process** systemd starts as that
account is a completely normal process — visible in `ps`/`top`,
killable, reniceable, subject to every concept in this demo.
`nologin` restricts the account's login capability, not the
process's behavior once running:

```bash
ps -ef | grep novatech-svc
# novatech-svc  2301  1  0  09:00  ?  00:01:12 /opt/novatech-app/bin/app
#                          ↑
#               PPID 1 — started directly by systemd, as expected
#               TTY ? — no controlling terminal, correct for a service
```

---

## Part 2 — Hands-On Lab

### Commands used in this demo

| Command | Flag meaning | Purpose |
|---|---|---|
| `ps aux` | `a`=all users, `u`=user format, `x`=no-terminal | All processes, BSD-style, %CPU/%MEM shown |
| `ps -ef` | `-e`=every, `-f`=full format | All processes, adds PPID column |
| `ps -el` | `-e`=every, `-l`=long format | All processes, adds PRI/NI/WCHAN columns |
| `ps aux --sort=-%cpu` | `--sort`=sort by column | Rank processes by CPU usage |
| `pgrep -fl "pattern"` | `-f`=full command line, `-l`=list name | Find PIDs matching full command, with names |
| `pstree -s PID` | `-s`=show ancestors | Trace a process's ancestry up to PID 1 |
| `pstree -p PID` | `-p`=show PIDs | Show a process's subtree with PIDs |
| `kill PID` | — | Send SIGTERM (default) |
| `kill -9 PID` | `-9`=SIGKILL | Force kill — cannot be caught or ignored |
| `kill -0 PID` | `-0`=zero signal | Check if PID exists without sending a signal |
| `killall name` | — | Send a signal to all processes matching a name |
| `pkill -f "pattern"` | `-f`=full command | Send a signal by matching the full command line |
| `nice -n N command` | `-n`=nice value | Start a new process at a given priority |
| `renice -n N -p PID` | `-n`=nice value, `-p`=target PID | Change priority of a running process |
| `nohup command &` | — | Make a process ignore SIGHUP on terminal close |
| `disown -h %N` | `-h`=no SIGHUP | Detach a running job from the shell without killing it |
| `top` | — | Live process monitor (default-installed) |
| `htop` | — | Enhanced live process monitor (requires install) |
| `lsof \| grep deleted` | — | Show files held open by processes after deletion |

---

### Task 1 — Explore the process tree

**Run on Ubuntu, then Rocky9:**

```bash
# Install pstree if not present
sudo apt install -y psmisc 2>/dev/null || sudo dnf install -y psmisc

# Flat view — both styles
ps aux | head -10
ps -ef | head -10
# Compare the columns: ps aux gives %CPU/%MEM, ps -ef gives PPID

# Sort by CPU to find the most active processes
ps aux --sort=-%cpu | head -5

# Visual tree — filtered to be readable
pstree -p | head -20

# Find your own shell's PID and trace its ancestry
echo "My shell PID: $$"
pstree -s $$
# Shows: systemd → sshd → bash (or similar chain)

# Find a specific service and its children
pstree -p $(pgrep -o sshd)

# Find a process's parent three ways
TARGET_PID=$(pgrep cron | head -1)
echo "Cron PID: $TARGET_PID"
ps -ef | grep $TARGET_PID          # Method 1: read PPID column
pstree -s $TARGET_PID              # Method 2: show full ancestry
cat /proc/$TARGET_PID/status | grep PPid  # Method 3: from /proc
```

**Reading your actual ps aux output — key observations:**

From your real output:
- `root 1 Ss ?` — systemd: sleeping, session leader, no terminal ✅
- `root 2 Sl+ hvc0` — WSL2/VM init: multi-threaded, foreground
  of hypervisor console — confirms you're running in a VM/WSL2
- `root 39 S<s ?` — systemd-journald: HIGH PRIORITY (< modifier),
  session leader — kernel elevated the log daemon's priority
- All system daemons show `?` for TTY and PPID 1 — correctly
  started by systemd with no controlling terminal

**Paste your `pstree -s $$` output — identify the full chain from
PID 1 to your current shell.**

---

### Task 2 — Process states, including a zombie

**Run on Ubuntu, then Rocky9:**

```bash
# Start a normal background sleep and observe its state
sleep 60 &
SLEEP_PID=$!
ps aux | grep $SLEEP_PID
# STAT column should show S (sleeping) or SN (sleeping, low priority)
# The N modifier is added by the shell for background jobs

# Create a zombie with a reliable observation window
cat > /tmp/zombie-demo.sh << 'SCRIPT'
#!/bin/bash
echo "Parent PID: $$"
bash -c 'sleep 2; exit 0' &
CHILD_PID=$!
echo "Child PID: $CHILD_PID — becomes zombie in ~2 seconds"
echo "Check now: ps aux | grep defunct"
echo "Parent sleeping 30 seconds without calling wait..."
sleep 30
echo "Parent exiting — zombie reaped by PID 1"
SCRIPT
chmod +x /tmp/zombie-demo.sh

# Run in background — your terminal stays free for observation
/tmp/zombie-demo.sh &
PARENT_PID=$!

# Wait for child to exit (~3 seconds)
sleep 3

# Check for zombie — the child has exited, parent hasn't called wait
ps aux | grep defunct
# Expected: [bash] <defunct>  with Z in STAT column

# Confirm Z state directly
ps -o pid,stat,cmd --ppid $PARENT_PID

# Also check via /proc if ps shows nothing (kernel may reap quickly)
ls /proc/$PARENT_PID/task/ 2>/dev/null && echo "Parent still running"

# Wait for parent to finish — zombie should clear
sleep 28
ps aux | grep defunct
# Should be empty — parent exited, PID 1 reaped the zombie
```

**Note if zombie window is too short:** modern kernels on fast
hardware may reap zombies so quickly they're gone before `ps`
runs. In that case, verify you understand the mechanism:
the `[bash] <defunct>` line would appear briefly between the
child exiting (after 2 seconds) and the parent exiting (after 30
seconds). The important observation is that `ps aux` output showed
no `defunct` lines after the parent exited.

**Paste the `ps aux | grep defunct` output showing the zombie,
and the final check confirming it cleared.**

---

### Task 3 — Signals: SIGTERM vs SIGKILL, via kill, pkill, and killall

**Run on Ubuntu, then Rocky9:**

```bash
# Create a process that explicitly ignores SIGTERM
cat > /tmp/stubborn.sh << 'SCRIPT'
#!/bin/bash
# Register a handler for SIGTERM that does nothing
trap '' SIGTERM
echo "PID $$ — ignoring SIGTERM (trap registered)"
echo "Try: kill $$ — it will have no effect"
echo "Then: kill -9 $$ — SIGKILL cannot be ignored"
while true; do sleep 1; done
SCRIPT
chmod +x /tmp/stubborn.sh

/tmp/stubborn.sh &
STUBBORN_PID=$!
sleep 1

# Check its state
ps -o pid,stat,cmd -p $STUBBORN_PID
# STAT should show S (sleeping) — normal, not D state

# Try SIGTERM — it will NOT stop this process
kill $STUBBORN_PID
sleep 2
ps -p $STUBBORN_PID
# Still running — SIGTERM was caught and ignored (empty trap handler)

# Check D state diagnosis — does ps show D or S?
ps -o pid,stat,cmd -p $STUBBORN_PID
# Shows S (sleeping) — this confirms: it's not D state, it's trapping SIGTERM
# Correct escalation: SIGKILL

# Escalate to SIGKILL — cannot be caught or ignored
kill -9 $STUBBORN_PID
sleep 1
ps -p $STUBBORN_PID 2>&1 || echo "Process is gone"

# Verify via kill -0 (no signal sent, just existence check)
kill -0 $STUBBORN_PID 2>/dev/null && echo "still alive" || echo "confirmed dead"
```

**Now repeat the same escalation using `pkill -f` instead of bare `kill`:**

```bash
# Start a second instance of the same stubborn script
/tmp/stubborn.sh &
STUBBORN_PID2=$!
sleep 1

# ALWAYS verify the match before using pkill — same discipline as any pattern kill
pgrep -fl stubborn.sh

# SIGTERM by pattern match — same "no effect" result as before, same reason
pkill -f stubborn.sh
sleep 2
ps -p $STUBBORN_PID2 2>&1 || echo "gone"
# Still alive — pkill sent SIGTERM, but the trap still ignores it

# Escalate via pkill this time
pkill -9 -f stubborn.sh
sleep 1
ps -p $STUBBORN_PID2 2>&1 || echo "confirmed dead via pkill"
```

**Now the same again with `killall` — exact process-name match:**

```bash
# Start a third instance
/tmp/stubborn.sh &
STUBBORN_PID3=$!
sleep 1

# killall matches by exact process name — verify what would match first
pgrep -l stubborn.sh

# killall -9 kills every process named exactly "stubborn.sh"
# Safe here because we know only one instance is running —
# on a shared system, killall's blast radius is exactly the danger
# described in section 1.5 ("DANGEROUS — kills EVERY python process")
killall -9 stubborn.sh
sleep 1
ps -p $STUBBORN_PID3 2>&1 || echo "confirmed dead via killall"
```

**Paste the `ps -p` output before and after each `kill -9`/`pkill
-9 -f`/`killall -9`, and the `kill -0` confirmation from the first
round.**

---

### Task 4 — Background jobs and the SSH disconnect problem

**Simulating a disconnect safely — using a nested SSH session:**

```bash
# Step 1: Start a nested SSH session (simulates a fresh connection)
ssh localhost
# Accept host key if prompted, enter your password
# You are now in a NESTED shell session

# Step 2: Inside the nested session — WITHOUT nohup
sleep 120 &
NO_NOHUP_PID=$!
echo "Without nohup PID: $NO_NOHUP_PID"
jobs

# Step 3: Exit the nested session (simulates SSH disconnect)
exit

# Step 4: Back in original session — check
ps -p $NO_NOHUP_PID 2>&1 || echo "Dead — died with the shell"

# Step 5: New nested session for the nohup test
ssh localhost

# Step 6: Inside nested session — WITH nohup
nohup sleep 120 > /dev/null 2>&1 &
NOHUP_PID=$!
echo "With nohup PID: $NOHUP_PID"

# Step 7: Exit again
exit

# Step 8: Check — should survive and show PPID 1
ps -f -p $NOHUP_PID
# PPID should be 1 (re-parented to systemd after shell exited)

# Cleanup
kill $NOHUP_PID 2>/dev/null
```

**If `ssh localhost` is unavailable (no SSH server/keys),
use a subshell simulation instead:**

```bash
# Alternative simulation without SSH
(
  # This subshell represents a "session"
  sleep 120 &
  NO_NOHUP_PID=$!
  echo "Without nohup PID: $NO_NOHUP_PID"
  # Subshell exits here — parent shell checks if job survived
)
sleep 1
ps -p $NO_NOHUP_PID 2>&1 || echo "Died with subshell exit"

(
  nohup sleep 120 > /dev/null 2>&1 &
  NOHUP_PID=$!
  echo "With nohup PID: $NOHUP_PID"
)
sleep 1
ps -p $NOHUP_PID && echo "Survived — nohup worked"
kill $NOHUP_PID 2>/dev/null
```

**Test `disown` on an already-running job:**

```bash
sleep 200 &
DISOWN_PID=$!
echo "PID: $DISOWN_PID"
jobs
# [1]+  Running    sleep 200 &

# Now disown it
disown -h %1
jobs
# (empty — job removed from shell's job table)

# The process is still running
ps -p $DISOWN_PID
# Still alive — just detached from this shell

kill $DISOWN_PID
```

**Paste the `ps -f -p` output showing the nohup process with
PPID 1 after the nested session exited.**

---

### Task 5 — Priority with nice and renice

**Run on Ubuntu, then Rocky9:**

```bash
# Start a process at DEFAULT priority and capture its PID
sleep 100 &
DEFAULT_PID=$!

# Read ps -el output — note PRI and NI columns
ps -el | grep $DEFAULT_PID
# Expected: PRI=80, NI=0

# Start a NEW process at explicitly LOWER priority (nice=10)
nice -n 10 sleep 100 &
NICE_PID=$!
ps -el | grep $NICE_PID
# Expected: PRI=90, NI=10 (lower priority than default)

# Change the ALREADY RUNNING default process's priority
renice -n 15 -p $DEFAULT_PID
ps -el | grep $DEFAULT_PID
# Expected: NI=15 (even lower priority now)

# Attempt to RAISE priority into negatives — requires root
renice -n -5 -p $DEFAULT_PID
# Expected: "Permission denied" for a regular user

sudo renice -n -5 -p $DEFAULT_PID
ps -el | grep $DEFAULT_PID
# Expected: NI=-5, PRI=75 (higher priority than default)

# Confirm all three processes and their priorities
ps -el | grep -E "$DEFAULT_PID|$NICE_PID"

# Cleanup
kill $DEFAULT_PID $NICE_PID 2>/dev/null
```

**Reading your actual Rocky9 output:**

```
0 S 1000  8173  8145  0  80  0  -  1335  hrtime  pts/2  0:00  sleep
         ↑              ↑   ↑
         UID=1000       PRI  NI=0 (default priority, no nice applied)
```

After `nice -n 10`: NI column shows 10, PRI shows 90.
After `sudo renice -n -5`: NI shows -5, PRI shows 75.

**Paste the `ps -el` output at each stage showing NI changes,
and the permission error from the non-root renice attempt.**

---

### Task 6 — Diagnosing a runaway process with top and htop

**Run on Ubuntu, then Rocky9:**

```bash
# Install htop
sudo apt install -y htop      # Ubuntu
sudo dnf install -y htop      # Rocky9

# Create a CPU-burning process
cat > /tmp/cpu-burn.sh << 'SCRIPT'
#!/bin/bash
echo "CPU burner PID: $$"
while true; do :; done
SCRIPT
chmod +x /tmp/cpu-burn.sh
/tmp/cpu-burn.sh &
BURN_PID=$!
echo "CPU-burning process PID: $BURN_PID"

# Observe in top (press P to sort by CPU, q to quit)
top

# Observe in htop (F5 for tree view, F9 to signal, F10 to quit)
htop

# From the command line — quick check without entering top
ps aux --sort=-%cpu | head -5
# Should show the burn process at or near 100% CPU

# Practice the correct kill escalation:
kill $BURN_PID           # SIGTERM first
sleep 2
kill -0 $BURN_PID 2>/dev/null && echo "still alive" || echo "gone"
# The burn script has no signal handler, so SIGTERM should work
# If still alive, escalate:
kill -9 $BURN_PID 2>/dev/null
ps -p $BURN_PID 2>&1 || echo "confirmed dead"
```

**In top/htop — what to observe:**
- The burn process should appear at the top of the CPU-sorted list
- Load average in top's header should increase (1-min avg climbing)
- In htop's tree view (F5), you can see the parent/child relationship
  with this demo's shell as the parent of the burn process

**Paste the `ps aux --sort=-%cpu | head -5` output showing the
CPU-burning process, and the kill escalation result.**

---

## What You Learned

- ✅ Explained what a process is, read PID/PPID relationships, and traced process ancestry with `ps`, `pstree -s`, and `/proc`
- ✅ Identified every process state (`R`, `S`, `D`, `T`, `Z`) and all modifier characters (`s`, `<`, `N`, `l`, `+`) from real ps output
- ✅ Explained what a controlling terminal is, how processes attach to and detach from one, and why SIGHUP kills background jobs but causes daemons to reload
- ✅ Sent the correct signal for the situation, and explained the two-step SIGTERM→SIGKILL escalation and when D state changes the diagnosis
- ✅ Found and killed processes correctly with `kill`, `pkill -f`, and `killall`, always verifying with `pgrep` before killing
- ✅ Managed foreground/background jobs and fixed the SSH-disconnect problem with `nohup` and `disown`
- ✅ Set and changed process priority with `nice`/`renice`, and read `NI`/`PRI` columns in `ps -el` output
- ✅ Diagnosed a runaway process by CPU using `top` and `htop`
- ✅ Connected process-held file handles back to Demo 02's `lsof` pattern, and confirmed a service account's process behaves identically to any other process despite `nologin` restricting only interactive login

**Key Takeaway:** Almost every process-management incident is really one of three questions: is this process in a state it should be in, did it receive the signal you think it received, and does it have the parent/priority relationship you expect. The answer to all three is in `ps -o pid,stat,ppid,ni,cmd -p PID` — one command that shows you state, parent, and priority together. Always run this before deciding what action to take.

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
# Confirm the process is gone
ps aux | grep deploy
# (nothing)

# Check what was run and how (command history if available)
history | grep deploy
# ssh ops@server "./deploy.sh"
# No nohup, no & — plain foreground SSH command
# When SSH dropped, SIGHUP was sent to the shell → default: terminate
# deploy.sh had no SIGHUP handler → died immediately
```

**Investigation — Incident 2:**

```bash
# Find the process
ps aux | grep batch-job
# wadmin  3301  98.7  0.2  /opt/app/batch-job

# Check its state BEFORE attempting to kill
ps -o pid,stat,cmd -p 3301
# 3301  S  /opt/app/batch-job
# STAT is S (sleeping) — NOT D — so signals will be received

kill 3301             # first attempt SIGTERM
sleep 3
ps -p 3301            # still running
kill 3301             # second attempt — same result
# SIGTERM is being received but the process is trapping it
# or has a SIGTERM handler stuck in a loop
```

**Your tasks:**

1. For Incident 1: what is the root cause in one sentence, and
   what exact command should have been used to start the
   deployment?

2. For Incident 2: SIGTERM was sent twice with no effect, but
   `ps -o stat` shows `S` not `D`. What does this confirm, and
   what is the correct next step?

3. After resolving Incident 2, what single command confirms the
   process is completely gone rather than just slow to appear in
   `ps`?

<details>
<summary>Resolution — open after attempting</summary>

**Incident 1 — root cause:**

The deployment script was run as a plain foreground SSH command
with no `nohup` and no `&`. It was a direct child of the SSH
session's shell. When the laptop slept and SSH disconnected, the
kernel sent SIGHUP to the shell, which propagated to the script.
The script had no SIGHUP handler, so it received the default
action: termination. Immediately, mid-deployment.

**Correct command:**
```bash
ssh ops@server "nohup ./deploy.sh > /tmp/deploy.log 2>&1 &"
# nohup: ignore SIGHUP
# > /tmp/deploy.log 2>&1: redirect output (no terminal after disconnect)
# &: background immediately so the SSH command returns
```

**Incident 2 — SIGTERM trapped, correct escalation:**

`STAT` showing `S` (not `D`) confirms the process is not in
uninterruptible sleep — it IS receiving signals. SIGTERM was
delivered twice but had no effect because the process has a
SIGTERM handler registered that is either ignoring it or stuck
in cleanup. Since D state is ruled out, SIGKILL is the correct
escalation:

```bash
kill -9 3301
sleep 1
kill -0 3301 2>/dev/null && echo "still alive" || echo "confirmed dead"
```

**Question 3 — confirming the process is truly gone:**

```bash
kill -0 3301 2>/dev/null
echo $?
# Non-zero exit code = PID no longer exists
# More reliable than ps because kill -0 queries the kernel directly
# rather than waiting for ps to refresh its process list
```

</details>

---

## Interview Prep

**Q1. A process shows `Z` state in `ps` and `<defunct>`. The
team wants to `kill -9` it. Will that work?**

No — a zombie process has already finished executing; there is
nothing left to signal. Only a process table entry holding its
exit status remains. The fix always targets the **parent**: make
it call `wait()` so it collects the exit status, or kill/restart
the parent so the kernel re-parents the zombie to PID 1, which
reaps it immediately. A small number of transient zombies is
normal; accumulating zombies indicate a bug in the parent.

---

**Q2. Why does a background job (`command &`) die when an SSH
session disconnects, but nginx does not die when you send it
SIGHUP?**

Signal handlers. When an SSH session ends, the kernel sends
SIGHUP to every process attached to that shell. SIGHUP's default
action is to terminate the process — a background job has no
SIGHUP handler registered, so it receives the default and dies.
nginx has registered a custom SIGHUP handler that overrides the
default and reloads its config file instead. `nohup` works by
registering a handler that ignores SIGHUP entirely. Same signal,
three different outcomes based entirely on whether the process
has a handler and what that handler does.

---

**Q3. Why is `kill -9` (SIGKILL) a last resort?**

`SIGTERM` gives the process a chance to clean up — flush buffered
writes, close file handles, finish in-flight database
transactions, deregister from a load balancer. `SIGKILL` removes
the process with zero cleanup, which can leave data files
half-written or locks permanently held. Additionally, `SIGKILL`
will not deliver immediately to a process in D state (waiting on
I/O) — so when SIGTERM is ineffective and you suspect D state,
escalating to SIGKILL is still not the right fix. Check `ps -o
stat` first: S or R → SIGKILL is correct; D → investigate the
I/O issue instead.

---

**Q4. What does `NI=0` mean in `ps -el` output, and who can
change it to a negative value?**

`NI` is the nice value — the user-visible priority input. `NI=0`
is the default: the process gets its normal share of CPU time.
Positive values (1–19) lower priority; negative values (−1 to
−20) raise it. Any user can lower their own process's priority
(raise the nice value toward +19). Only root can raise priority
into negative values — this prevents a regular user from starving
other users' processes of CPU. The kernel converts the nice value
into its internal `PRI` scheduling value, which you never set
directly.

---

## Key Takeaways

1. **Every process has exactly one parent, tracing back to PID 1 — `ps -ef` shows PPID directly, and `pstree -s PID` traces the full ancestry chain in one command.** When a parent exits before its child, the child is re-parented to PID 1, not killed — this is how `nohup` jobs survive a shell exit.

2. **`ps -o pid,stat,ppid,ni,cmd -p PID` answers the three most important process questions in one command: what state is it in, who started it, and what priority does it have.** Always run this before deciding whether to kill, wait, or investigate further.

3. **A zombie (`Z`/`<defunct>`) is already dead — there is nothing to signal.** Fix the parent process, not the zombie. A `Z` in STAT means `kill` of any kind will have no effect.

4. **SIGTERM's default action is terminate, but daemons like nginx override it to reload config instead. Background jobs have no handler, so they get the default: terminated.** `nohup` works by registering a "do nothing" SIGHUP handler. `SIGKILL` and `SIGSTOP` cannot be overridden by any handler, ever.

5. **Before killing anything, check `ps -o stat` — a process in `D` state cannot respond to signals, even `SIGKILL`.** D state means the kernel is the bottleneck (disk/NFS I/O), not the process logic. Escalating signals won't help; investigate the I/O layer instead.

6. **`pkill -f "full command pattern"` is the safe production kill — always verify the match with `pgrep -fl` first.** `killall processname` kills every process with that exact name, which is dangerous on shared servers running multiple instances.

7. **Niceness runs from -20 (highest priority) to +19 (lowest) — lower number means higher priority.** Any user can lower their own process priority; only root can raise it above default. The `NI` column in `ps -el` shows the current value.

8. **`pstree` unfiltered is too long to be useful on any real server — always filter with `-s PID` (ancestry), `-p PID` (subtree), or `-u username`.** Unfiltered `pstree -p` is for learning the format, not production use.

---

## Quick Commands Reference

### Viewing processes

| Command | Purpose |
|---|---|
| `ps aux` | All processes, BSD-style (%CPU, %MEM columns) |
| `ps -ef` | All processes, UNIX-style (PPID column) |
| `ps -el` | All processes, long format (NI, PRI columns) |
| `ps aux --sort=-%cpu \| head -10` | Top CPU consumers |
| `ps -o pid,stat,ppid,ni,cmd -p PID` | Focused view of one process |
| `pstree -s PID` | Full ancestry chain of a process |
| `pstree -p PID` | Subtree rooted at a process |
| `pgrep -fl "pattern"` | Find PIDs matching full command line |
| `cat /proc/PID/status \| grep PPid` | Parent PID from kernel directly |

### Signals and killing

| Command | Purpose |
|---|---|
| `kill PID` | Send SIGTERM (graceful, default) |
| `kill -9 PID` | Send SIGKILL (forceful, last resort) |
| `kill -SIGHUP PID` | Send by name |
| `kill -0 PID` | Check PID exists without sending a signal |
| `kill -l` | List all signal names and numbers |
| `pkill -f "pattern"` | Kill by full command line match |
| `pgrep -fl "pattern"` | Verify match before killing |
| `killall name` | Kill all processes with this exact name |

### Jobs and backgrounding

| Command | Purpose |
|---|---|
| `command &` | Run in background (child of current shell) |
| `jobs` | List background jobs in this shell |
| `fg %N` | Bring job N to foreground |
| `bg %N` | Resume stopped job N in background |
| `nohup command > log 2>&1 &` | Run immune to SIGHUP, output to log |
| `disown -h %N` | Detach running job N from shell's SIGHUP |

### Priority

| Command | Purpose |
|---|---|
| `nice -n N command` | Start new process at nice value N |
| `renice -n N -p PID` | Change nice value of running process |
| `sudo renice -n -N -p PID` | Raise priority (negative, requires root) |

### Monitoring

| Command | Purpose |
|---|---|
| `top` | Live monitor — P=CPU sort, M=MEM sort, q=quit |
| `htop` | Enhanced monitor — F5=tree, F9=kill, F3=search |
| `lsof \| grep deleted` | Files held open by processes (Demo 02 callback) |

---

## Appendix A — Anki Cards

**`06-process-management-anki.csv`:**

```
#deck:Linux Mastery::Module 2 - System Administration::Demo 06 - Process Management
#separator:Comma
#columns:Front,Back,Tags

"ps aux shows a process with STAT = Z and <defunct> in the command. The team wants to kill -9 it. Will that work?","No. A zombie process has already finished executing — there is nothing left to signal, only a process-table entry with its exit status waiting for the parent to call wait(). The fix always targets the PARENT: make it call wait(), or kill/restart the parent so the kernel re-parents the zombie to PID 1, which reaps it immediately.","demo06,zombie,signals"

"SIGTERM is sent twice to a process with no effect. ps -o stat -p PID shows S. What does this confirm, and what is the correct next step?","STAT S = sleeping, not D state. The process IS receiving signals but has a SIGTERM handler registered that is either ignoring the signal or stuck in cleanup. D state (uninterruptible sleep/IO wait) is ruled out. Correct next step: escalate to kill -9 (SIGKILL cannot be caught or ignored). If STAT had shown D, escalating signals would not help — the IO issue would need investigation instead.","demo06,signals,troubleshooting,d-state"

"Why does SIGHUP kill a background shell job but cause nginx to reload its config?","Signal default action: if a process has NOT registered a handler for SIGHUP, the kernel's default action is to TERMINATE it. Background jobs have no SIGHUP handler registered — they get the default: killed. nginx has registered a custom SIGHUP handler that overrides the default and reloads nginx.conf instead. nohup registers a handler that IGNORES SIGHUP entirely. Same signal, three different outcomes based entirely on whether a handler exists and what it does.","demo06,sighup,signal-handlers,nohup"

"What does pstree -s PID do, and why use it instead of unfiltered pstree -p?","pstree -s PID shows the ANCESTRY chain of a specific process from PID 1 down: systemd(1)───sshd(650)───bash(1823)───suspicious-process(3301). Immediately answers who started this and through what chain. Unfiltered pstree -p prints the entire system process tree which is unreadably long on any real server. Always filter: -s for ancestry, -p PID for subtree, -u for a specific user.","demo06,pstree,troubleshooting"

"What is the difference between a process in S state and D state? Why does this matter for signal delivery?","S (interruptible sleep): waiting for an event (input, timer, lock). CAN receive and respond to signals — SIGTERM and SIGKILL both work. D (uninterruptible sleep): waiting on I/O (disk, NFS). CANNOT receive or respond to any signal, including SIGKILL. The signal is queued but not delivered until the IO completes. Production implication: if kill -9 has no effect, check ps -o stat — if D, the problem is the IO layer, not the process.","demo06,process-states,signals,d-state"

"What does ? in the TTY column of ps aux mean, and why should ALL properly running services show it?","? = no controlling terminal. The process is detached from any interactive session. All properly running daemons and systemd services should show ? because they were started by systemd without a terminal. A service showing pts/N instead of ? was likely started interactively by mistake and will die when that terminal closes (unless nohup was used). TTY ? also means SIGHUP from any terminal close has no effect on this process.","demo06,tty,terminal,daemons"

"What does NI=0 mean in ps -el output? Who can set it to a negative value and why?","NI is the nice value — the user-visible priority input. NI=0 = default priority, normal CPU share. NI=+10 = lower priority. NI=-5 = higher priority (requires root). Any user can LOWER their own process priority (raise NI toward +19). Only root can RAISE priority into negative values — preventing a regular user from starving other processes of CPU time.","demo06,nice,renice,priority"

"A deleted log file is still consuming disk space per df but not visible in du. What process fact explains this, and what is the exact fix?","A deleted file's directory entry is gone (invisible to du which walks directory entries). But if a PROCESS still has the file open, the kernel keeps the inode and data blocks allocated (counted by df). lsof | grep deleted shows exactly which PID holds the file open. Fix: kill -TERM PID (restart the process) to release the file handle — only then does the kernel free the blocks and df reflects the freed space.","demo06,lsof,deleted-files,signals"

"ps aux shows STAT values like Ss, S<s, and SN. What do the s, <, and N modifier characters mean?","s = session leader — this process leads a process group/session (e.g. sshd, cron, systemd-journald). < = high priority — nice value is negative, getting MORE CPU than default (kernel-elevated daemons like systemd-journald). N = low priority — nice value is positive, getting LESS CPU than default (shells apply this automatically to background jobs started with &). These stack after the primary state letter (R/S/D/T/Z) and combine to describe a process precisely.","demo06,process-states,modifiers"

"What is the mechanical difference between starting a command with nohup vs running disown -h on a job already running?","nohup command &: used at the START. Registers a handler that IGNORES SIGHUP from the moment the process starts, AND you must separately redirect output (> file 2>&1) since there will be no terminal to write to after disconnect. disown -h %N: used AFTER a job is already running. Removes it from the shell's job table so it won't receive SIGHUP when the shell exits, but CANNOT retroactively redirect output already flowing to the terminal. Use nohup when you remember at the start; use disown to rescue a job you forgot to nohup.","demo06,nohup,disown,jobs"
```

---

## Appendix B — Quiz

**`06-process-management-quiz.md`:**

````markdown
# Quiz — Demo 06: Process Management

> One correct answer per question unless stated otherwise.
> Target: 80% or above before moving to Demo 07.

---

**Q1.** A shell starts a background job with `command &`, then the
shell itself exits before the job finishes (not via SSH disconnect
— just a normal parent exit). What happens to the child process?

- A) The child is automatically killed along with the parent
- B) The child becomes an orphan and is automatically re-parented
  to PID 1, which continues to manage it until it exits naturally —
  it does not die just because its original parent exited
- C) The child immediately becomes a zombie
- D) The child is paused (SIGSTOP) until a new parent claims it

<details>
<summary>Answer</summary>

**B** — Orphaned processes are re-parented to PID 1 automatically;
this re-parenting, not any special "survival" feature, is exactly
the mechanism that lets a `nohup` job keep running after its
original shell is gone.

Trap: C confuses orphaning with zombie state — those are opposite
situations: a zombie has exited but its parent hasn't reaped it; an
orphan is still running but its parent is gone.

</details>

---

**Q2.** You need to see (1) which process is using the most CPU/memory
and (2) which process started which. Which two `ps` invocations give
you each of these directly, without extra parsing?

- A) `ps aux` for resource usage (%CPU/%MEM columns shown directly);
  `ps -ef` for parent/child relationships (PPID column shown directly)
- B) `ps -ef` for resource usage; `ps aux` for parent/child — the two
  are reversed from what most people assume
- C) `ps -el` is the only variant needed for both purposes
- D) Both show identical columns; the difference is purely cosmetic

<details>
<summary>Answer</summary>

**A** — `ps aux` (BSD-style) surfaces `%CPU`/`%MEM` directly — first
choice for "what's eating resources?" `ps -ef` (UNIX-style) surfaces
`PPID` directly — first choice for "what started this?" `ps -el`
adds `PRI`/`NI` for priority work specifically, not a general-purpose
replacement for the other two.

</details>

---

**Q3.** What does the `N` modifier in `SN` in the STAT column of
`ps aux` mean, and why does a `sleep 60 &` background job show it?

- A) N = not responding (the process has stopped accepting input)
- B) N = low priority — the nice value is positive. The shell
  slightly deprioritizes background jobs when started with `&`,
  giving them less CPU time than foreground processes.
- C) N = new process that has not yet been scheduled
- D) N = running in a network namespace

<details>
<summary>Answer</summary>

**B** — `N` is the modifier for a positive nice value (lower
than default priority). Shells typically apply a slight nice
increase to background jobs started with `&`, reflected as `SN`
in the STAT column.

</details>

---

**Q4.** What is the safest sequence to kill a process you
identified by a command pattern?

- A) `kill -9 $(pgrep -f "pattern")` — use SIGKILL from the start
- B) `pgrep -fl "pattern"` to verify the match, then `pkill -f
  "pattern"` for SIGTERM, wait, then `pkill -9 -f` only if needed
- C) `killall pattern` — most efficient for pattern-based kills
- D) Send SIGHUP first, then SIGTERM, then SIGKILL in sequence

<details>
<summary>Answer</summary>

**B** — Always verify the match with `pgrep -fl` before killing
anything. Use `pkill -f` (SIGTERM) first to allow clean shutdown.
`killall` matches only exact process names, not patterns. SIGKILL
as a starting point risks data corruption and skips the chance
for a clean shutdown.

</details>

---

**Q5.** `renice -n -10 -p 3301` fails with "Permission denied"
for a regular user. Why, and what is the fix?

- A) `renice` cannot target running processes, only new ones
- B) Negative nice values (higher priority than default) require
  root — regular users can only lower their own process priority.
  Fix: `sudo renice -n -10 -p 3301`
- C) PID 3301 does not belong to this user
- D) `-10` is outside the valid range for `renice`

<details>
<summary>Answer</summary>

**B** — Any user can raise their nice value (lower priority). Only
root can lower the nice value into negatives (raise priority above
default). This prevents a regular user from starving other users'
processes. `sudo renice` applies the change with root privileges.

</details>

---

**Q6.** A 4-core server shows `load average: 8.2, 7.9, 6.1` in
`top`'s header. What does this tell you?

- A) The system is under-utilized — plenty of spare capacity
- B) The system is significantly overloaded — a load average
  roughly double the core count means processes are queuing for
  CPU time that isn't available
- C) Load average is unrelated to CPU cores; it only measures disk
  I/O wait
- D) This is normal and requires no action — load average always
  reads high on multi-core systems

<details>
<summary>Answer</summary>

**B** — The rule: load average consistently above the number of
CPU cores indicates a CPU bottleneck. On 4 cores, a load average
around 4.0 means the system is exactly at capacity; ~8.0 means
roughly double the demand the system can serve at once — processes
are actively waiting in the run queue.

Trap: C wrongly narrows load average to I/O wait only — it reflects
overall demand for CPU time, of which I/O wait is only one
contributing factor.

</details>

---

**Q7.** A service account was created with `/usr/sbin/nologin`.
Does the `nologin` shell prevent its running process from
appearing in `ps` or receiving signals normally?

- A) Yes — `nologin` processes are hidden from `ps` by default
- B) Yes — `nologin` processes can only be stopped by systemd
- C) No — `nologin` only blocks interactive login (SSH, su,
  console). Once started by systemd, the process is fully visible
  and manageable: it appears in `ps`, can receive signals, and
  can be reniced exactly like any other process.
- D) No — but signals require the `-f` flag when targeting
  `nologin` accounts

<details>
<summary>Answer</summary>

**C** — `/usr/sbin/nologin` is a login-shell restriction, not a
process-behavior restriction. The running process is completely
normal from the kernel's perspective. `nologin` only affects the
authentication/login path, not what the process can do once
started.

</details>

---

| Score | Action |
|---|---|
| 7/7 | Import Anki cards, move to Demo 07 |
| 6/7 | Review the wrong answer, then proceed |
| 5/7 | Re-read the relevant section, retry those questions |
| Below 5/7 | Re-read the full demo and redo the walkthrough before proceeding |
````