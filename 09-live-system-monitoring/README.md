# Demo 09 — Live System Monitoring

## Overview

Every diagnosis of a slow or misbehaving server starts with the same question:
what is this machine doing right now? This demo covers the live, interactive
tools you reach for first — `top`, `htop`, `vmstat`, `free` — plus reading the
kernel's own live counters directly through `/proc`, and turning any one-shot
command into a running dashboard with `watch`. By the end, you'll be able to
read a load average number and know immediately whether it's fine or a
problem, and act on a runaway process without leaving `top`.

## Demo Objectives

1. Read and interpret load average (1/5/15 min) correctly against core count
2. Navigate `top` and `htop`'s live views: CPU, memory, per-core breakdown, I/O wait
3. Read `vmstat` output to spot memory pressure and CPU contention at a glance
4. Read `free` output correctly, including why "available" is not "free"
5. Read `/proc/loadavg` and `/proc/meminfo` directly, and know when to bother
6. Use `watch` to turn any command into a live-refreshing view
7. Sort, renice, and kill a process from inside `top` without leaving it

## Directory Structure

```
09-live-system-monitoring/
├── README.md                          # This file — full demo content
├── 09-live-system-monitoring-anki.csv # Standalone Anki deck (Appendix A, extracted)
└── 09-live-system-monitoring-quiz.md  # Standalone quiz (Appendix B, extracted)
```

## Environments

| Task | Ubuntu 24.04 | Rocky Linux 9.8 | EC2 Amazon Linux |
|---|---|---|---|
| All lab tasks | ✅ | ✅ | Not required |
| `htop` install | `sudo apt install htop` | `sudo dnf install htop` | Not required |

## Recall Check — 08-systemd-timers

Answer these from memory before continuing — no looking back at Demo 08.

**Q1.** A `.timer` unit has no `Unit=` line under `[Timer]`. Which service does
it activate when it fires?

**Q2.** You set `Persistent=true` on a daily timer. The machine is off for
exactly one missed run, then powered back on. How many catch-up runs happen?

**Q3.** You run `systemctl list-timers` with no flags and don't see a timer you
know exists. What's the first thing you check, before assuming it's broken?

<details>
<summary>Answers</summary>

**A1.** The identically-named `.service` unit — a `.timer` activates the
`.service` of the same base name by default unless `Unit=` explicitly points
elsewhere.

**A2.** One — `Persistent=true` fires a single catch-up run for the most
recent missed occurrence, not one run per missed day.

**A3.** Re-run with `--all` — by default `list-timers` only shows currently
active timers; a disabled or inactive timer won't appear without the flag.

</details>

## Part 1 — Concepts

### 1.1 Load average — what the three numbers actually mean

`uptime` and the top-left corner of `top` both show three numbers:

```bash
$ uptime
 14:32:07 up 12 days,  3:14,  2 users,  load average: 2.15, 1.87, 1.42
```

These are the 1-minute, 5-minute, and 15-minute exponentially-weighted moving
averages of the number of processes either running on a CPU or waiting for one
(and on Linux, also processes in uninterruptible I/O wait — this is why load
average can rise even when CPUs look idle).

**The number is meaningless without knowing the core count.** A load average
of 4.0 is trivial on a 16-core machine and a serious overload on a 2-core one.
Always normalize:

```bash
nproc
# On a 4-core machine, load average 2.15 → 2.15/4 ≈ 0.54 (54% average demand
# over the last minute) — comfortable. The same 2.15 on a 2-core machine is
# ≈107% — already oversubscribed.
```

Reading the trend across the three numbers matters as much as the numbers
themselves: `2.15, 1.87, 1.42` (rising toward the present) means load is
*increasing* right now; the reverse pattern means it's easing off. This is
often more actionable than any single number.

### 1.2 top and htop — the live view

```bash
top
```

```
top - 14:35:02 up 12 days,  3:17,  2 users,  load average: 1.05, 1.20, 1.15
Tasks: 187 total,   1 running, 186 sleeping,   0 stopped,   0 zombie
%Cpu(s):  8.2 us,  2.1 sy,  0.0 ni, 89.0 id,  0.5 wa,  0.0 hi,  0.2 si,  0.0 st
MiB Mem :   7943.1 total,   1204.3 free,   2891.6 used,   3847.2 buff/cache
MiB Swap:   2048.0 total,   2048.0 free,      0.0 used.   4623.8 avail Mem
```

Reading the CPU line: `us` (user processes), `sy` (kernel/system calls),
`ni` (niced processes), `id` (idle), **`wa` (I/O wait — CPU idle specifically
because it's waiting on disk/network I/O, not truly free)**, `hi`/`si`
(hardware/software interrupts), `st` (stolen — a virtualized guest waiting for
its host's CPU). A high `wa` with low `us`/`sy` points at a storage or network
bottleneck, not a CPU one — a common misdiagnosis is treating high `wa` as
"CPU is busy" and looking for a runaway process, when the real fix is on the
I/O side.

`htop` shows the same underlying data with a friendlier layout: per-core bars
across the top (so you see imbalance across cores at a glance, which plain
`top`'s aggregate `%Cpu(s)` line hides), a tree view of process parent/child
relationships, and mouse-clickable actions.

```bash
sudo apt install htop   # Ubuntu
sudo dnf install htop   # Rocky9
htop
```

**Worth knowing it exists:** `glances` is a newer, more ambitious alternative
that adds network, disk I/O, and container awareness to the same live-view
concept in one dashboard. It's not covered hands-on here — `top`/`htop`
cover the fundamentals every environment has available — but it's common
enough in modern DevOps toolchains that recognizing the name is worthwhile.

### 1.3 vmstat — the fast diagnostic snapshot

```bash
vmstat 2 5
# 2: refresh every 2 seconds. 5: stop after 5 samples.
```

```
procs -----------memory---------- ---swap-- -----io---- -system-- ------cpu-----
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
 1  0      0 1204312  98432 3748864    0    0    12    45  201  412  8  2 89  1  0
```

Two columns matter most for a quick triage: **`r`** (processes actively
runnable, waiting for CPU — sustained high `r` relative to core count means
CPU contention) and **`b`** (processes blocked in uninterruptible sleep,
almost always I/O — sustained non-zero `b` means something is waiting on
disk). `si`/`so` (swap in/out) should be at or near zero on a healthy system;
non-zero and sustained means the machine is actively swapping, which is a
serious performance problem, not a minor one.

### 1.4 free — reading memory correctly

```bash
free -h
```

```
               total        used        free      shared  buff/cache   available
Mem:           7.8Gi       2.8Gi       1.2Gi        45Mi       3.8Gi       4.5Gi
Swap:          2.0Gi          0B        2.0Gi
```

**The most common misreading of this output is treating `free` as "how much
memory is actually available."** Linux aggressively uses spare RAM for
`buff/cache` (page cache, dentries, inode caches) because unused RAM is wasted
RAM — that cache is reclaimed instantly under memory pressure. `available` is
the column that answers "how much memory could a new process actually get
right now, accounting for reclaimable cache" — that's the number to watch, not
`free`. A system showing 1.2Gi `free` and 4.5Gi `available` is *not* low on
memory; a system showing low `available` regardless of `free`/`buff/cache` is.

### 1.5 Reading /proc directly — when the tools aren't enough

Every number `top`, `vmstat`, and `free` display ultimately comes from files
under `/proc` — the kernel's live, virtual view of its own state:

```bash
cat /proc/loadavg
# 2.15 1.87 1.42 1/187 48213
# Field 4 (1/187): 1 process currently runnable out of 187 total.
# Field 5: PID of the most recently created process.

cat /proc/meminfo | head -6
# MemTotal, MemFree, MemAvailable (the same "available" free shows), Buffers,
# Cached, SwapCached — the raw source data behind every memory tool.
```

Reading `/proc` directly matters in two situations: scripting (parsing `top`'s
formatted output is fragile; `/proc/loadavg` is one clean line) and when you
need a number no formatted tool surfaces directly.

### 1.6 watch — turning any command into a live view

```bash
watch -n 2 'free -h'
# -n 2: refresh every 2 seconds (default is 2 already, shown for clarity)
watch -d 'ss -tunlp'
# -d: highlight what changed between refreshes — useful for spotting new
# connections appearing rather than re-reading the whole table each time
```

`watch` is the general-purpose tool for anything that doesn't already have its
own live mode — most commands (`df -h`, `ss`, `ls -la /proc/*/fd`) don't, and
wrapping them in `watch` is simpler than writing a loop.

### 1.7 top's interactive controls — acting without leaving

Inside `top`, keystrokes change what you see and can act directly on
processes:

| Key | Effect |
|---|---|
| `1` | Toggle per-core CPU breakdown (vs. aggregate) |
| `M` | Sort by memory usage |
| `P` | Sort by CPU usage (default) |
| `k` | Kill a process — prompts for PID, then signal (default `SIGTERM`) |
| `r` | Renice a process — prompts for PID, then new niceness value |
| `q` | Quit |

Being able to spot the offending process, renice or kill it, and confirm the
effect — all without leaving `top` — is the difference between a fast
diagnosis and a slow one. `htop` exposes the same actions via function-key
labels at the bottom of the screen instead of memorized letters.

## Part 2 — Hands-On Lab

### Commands used in this demo

| Command | Flag meaning | Purpose |
|---|---|---|
| `uptime` | — | Quick load average + uptime summary |
| `nproc` | — | Core count, for normalizing load average |
| `top` | — | Live process/resource view |
| `htop` | — | Friendlier live view with per-core bars and tree mode |
| `vmstat N C` | `N`: interval seconds, `C`: count | Fast r/b/swap/CPU snapshot over time |
| `free -h` | `-h`: human-readable sizes | Memory summary, correctly read via `available` |
| `watch -n N 'cmd'` | `-n N`: refresh interval | Turn any command into a live view |
| `watch -d 'cmd'` | `-d`: highlight diffs | Spot what changed between refreshes |

### Task 1 — Load average, normalized against core count

```bash
nproc
uptime
# Manually compute: load average ÷ core count for each of the three numbers.
# Is this machine's current demand comfortable, borderline, or oversubscribed?
```

### Task 2 — top: read the CPU line under synthetic load

```bash
# Generate CPU load in the background:
yes > /dev/null &
yes > /dev/null &

top
# Watch %Cpu(s): us should rise sharply, id should drop. Note which PIDs
# correspond to the yes processes (they'll be at/near the top, sorted by CPU).

# Clean up:
kill $(jobs -p)
```

### Task 3 — vmstat: spot the r and b columns

```bash
vmstat 1 5
# Run this idle first, note r and b are near 0.
yes > /dev/null &
vmstat 1 5
# r should rise while the yes process is runnable and contending for CPU.
kill %1
```

### Task 4 — free: available vs. free after generating cache

```bash
free -h
# Note the buff/cache and available columns.
find / -type f -exec cat {} + > /dev/null 2>&1 &
sleep 5; kill %1 2>/dev/null
free -h
# buff/cache should have grown noticeably (page cache from reading many
# files) while available barely changes — that cache is reclaimable, so it
# doesn't threaten the available figure.
```

### Task 5 — /proc/loadavg and /proc/meminfo directly

```bash
cat /proc/loadavg
cat /proc/meminfo | grep -E '^(MemTotal|MemFree|MemAvailable):'
# Compare MemAvailable here against the "available" column free -h showed —
# same underlying number, different presentation.
```

### Task 6 — watch, and acting from inside top

```bash
watch -n 1 'cat /proc/loadavg'
# Ctrl+C to exit.

# In a second terminal, generate load again:
yes > /dev/null &

# Back in top: press 1 to toggle per-core view, P to confirm CPU-sort,
# then r to renice the yes process to 19 (lowest priority), and observe
# its %CPU share drop relative to other processes. Then k to kill it,
# selecting SIGTERM (15) at the prompt.
```

## What You Learned

1. ✅ Read and interpreted load average (1/5/15 min) correctly against core count
2. ✅ Navigated `top` and `htop`'s live views: CPU, memory, per-core breakdown, I/O wait
3. ✅ Read `vmstat` output to spot memory pressure and CPU contention at a glance
4. ✅ Read `free` output correctly, distinguishing `available` from `free`
5. ✅ Read `/proc/loadavg` and `/proc/meminfo` directly
6. ✅ Used `watch` to turn a one-shot command into a live-refreshing view
7. ✅ Sorted, reniced, and killed a process from inside `top` without leaving it

## Part 3 — Break-Fix Scenario

### Scenario — NovaTech Ops: two live-monitoring incidents

> **Incident report — NovaTech Ops, ticket #4502**
>
> Two alerts fired overnight. Alert 1: "app-server-03 load average critical."
> Alert 2: "app-server-04 appears to be swapping." You're on call — triage
> both using only the live tools from this demo.

**Incident 1 — "load average critical" on a machine that looks idle**

```bash
$ uptime
 03:14:22 up 40 days,  8:02,  1 user,  load average: 6.82, 6.40, 5.95
$ nproc
4
$ top
top - 03:14:40 up 40 days,  8:02,  1 user,  load average: 6.82, 6.40, 5.95
Tasks: 142 total,   1 running, 132 sleeping,   0 stopped,   9 zombie
%Cpu(s):  2.1 us,  1.0 sy,  0.0 ni, 92.4 id,  4.3 wa,  0.0 hi,  0.2 si,  0.0 st
```

Tasks:
1. Normalize 6.82 against 4 cores. On CPU alone, does this look like a CPU
   overload? Now look at `%Cpu(s)` — `id` is 92.4%. What does a high load
   average combined with mostly-idle CPU and a nonzero `wa` actually indicate?
2. What does "9 zombie" in the Tasks line contribute to load average, and what
   does it not contribute to? (Hint: zombies aren't running or waiting for CPU
   — they're dead processes waiting for their parent to reap them via `wait()`.)
3. Run `vmstat 1 5` to confirm which column supports your diagnosis — is it
   `r` (CPU-bound) or `b` (I/O-blocked) that's elevated?

**Incident 2 — "appears to be swapping"**

```bash
$ free -h
               total        used        free      shared  buff/cache   available
Mem:           3.8Gi       3.6Gi       89Mi        12Mi       120Mi        95Mi
Swap:          2.0Gi       1.4Gi       612Mi
$ vmstat 1 5
procs -----------memory---------- ---swap-- -----io----
 r  b   swpd   free   buff  cache   si   so    bi    bo
 2  1  1433600   91136  12288 122880  842  310   890   340
```

Tasks:
1. Look at `available` in `free -h`. Is 95Mi available on a 3.8Gi machine a
   real problem, or could this still be healthy given how Linux uses
   `buff/cache`? Justify using what §1.4 says `available` actually measures.
2. Now look at `swpd` (1.4Gi) and `si`/`so` (842/310) in `vmstat`. Does this
   change your answer? Which of these two pieces of evidence — low
   `available`, or active `si`/`so` — is the real signal that this machine
   has an actual memory problem, not just normal cache usage?
3. `buff/cache` here is only 120Mi — much smaller than a healthy system
   usually shows. Why does that matter, given what §1.4 said the kernel does
   with spare RAM?

<details>
<summary>Answers</summary>

**Incident 1:** Normalized, 6.82 ÷ 4 ≈ 1.7 — that would be a real CPU overload
if the CPU were actually busy. But `%Cpu(s)` shows 92.4% idle, meaning the CPUs
are not the bottleneck. Combined with `wa: 4.3`, this points at processes
stuck waiting on I/O — Linux counts uninterruptible I/O wait toward load
average, so a storage-bound workload can produce a high load average with
CPUs sitting mostly idle. Zombies contribute their count to the Tasks line and
technically each held a load-average-eligible slot briefly at some point, but
a zombie itself is not running or blocked on I/O — it consumes no CPU and
isn't the source of an *ongoing* elevated load average; 9 zombies is a
process-reaping hygiene issue (check the parent process), not the explanation
for a load average of 6.82. `vmstat`'s `b` column (processes in uninterruptible
sleep) should be elevated here, not `r` — confirming I/O-bound, not CPU-bound.

**Incident 2:** 95Mi `available` on a 3.8Gi machine, on its own, would be
concerning but not automatically a crisis — if `buff/cache` were large, most
of it could be instantly reclaimed. But here `buff/cache` is only 120Mi —
already small, meaning there's little reclaimable cache left to fall back on;
the low `available` is real, not a cache artifact. The clinching evidence is
`vmstat`'s `si`/`so` at 842/310 — non-zero and sustained swap activity is
never "normal cache usage," it's the kernel actively moving pages to and from
disk because RAM is genuinely insufficient. Small `buff/cache` matters exactly
because the kernel prefers to keep page cache over swapping when it can — a
system that's swapping while cache is already thin has already given up what
it could reclaim and still needs more memory than it has.

</details>

## Interview Prep

**Q1. A colleague says "load average is high, so the CPU must be the bottleneck." When is this reasoning wrong, and what would you check instead?**
It's wrong whenever the elevated load average is driven by processes in
uninterruptible I/O wait rather than CPU contention — Linux includes both in
the load average calculation. The check is `top`'s `%Cpu(s)` line: high `id`
(idle) combined with nonzero `wa` (I/O wait) means the CPUs aren't busy at
all; the load is coming from disk or network-bound processes. `vmstat`'s `b`
column (blocked, uninterruptible) versus `r` column (runnable, CPU-contending)
gives the same distinction more directly.

**Q2. `free -h` shows very little "free" memory but plenty of "available." Is this machine actually low on memory?**
No, not necessarily — this is one of the most common false alarms in Linux
memory monitoring. Linux deliberately uses otherwise-idle RAM for page cache
and other reclaimable caches (`buff/cache`), because unused RAM has no value
sitting idle. `available` is the number that already accounts for how much of
that cache the kernel could hand back to a new process on demand — that's the
figure that answers "how much memory can I actually use," not `free`.

**Q3. How would you tell the difference between "the system is using cache normally" and "the system is genuinely out of memory and swapping"?**
Check `vmstat`'s `si`/`so` columns — sustained non-zero swap-in/swap-out
activity is the unambiguous signal of real memory pressure, since it means the
kernel is moving pages to and from disk to make room, something it avoids
unless it has to. Normal cache usage shows up as high `buff/cache` with `si`/
`so` at or near zero. Also check whether `buff/cache` itself is unusually
small for the machine's total RAM — a thin cache alongside active swapping
means the system has already given up what it could reclaim and still needs
more memory than it has.

**Q4. Why would you reach for `vmstat` instead of `top` when triaging a suspected performance issue over SSH on a slow connection?**
`vmstat`'s output is compact, line-based, and easy to capture with `>>` into a
log or pipe into another tool — `top`'s default full-screen redraw is harder
to work with over high-latency connections and awkward to script. `vmstat N
C` (interval and count) also gives a clean time series in one command,
whereas getting the same trend from `top` means watching it live or using its
batch mode (`top -b -n`) with extra parsing.

## Key Takeaways

1. **Load average includes I/O wait, not just CPU contention.** A high load
   average with low CPU usage and nonzero `wa` in `top` almost always means
   an I/O bottleneck, not a CPU one — normalize against `nproc` first, then
   check `%Cpu(s)` before assuming the CPU is the problem.
2. **`free`'s "available" column is the number that matters, not "free."**
   Linux uses spare RAM for reclaimable page cache; low `free` with high
   `available` is completely normal, not a warning sign.
3. **`vmstat`'s `si`/`so` columns are the definitive swap-activity signal.**
   Non-zero and sustained swap-in/swap-out means real memory pressure;
   low `buff/cache` alongside it means the system has already exhausted its
   reclaimable cache and still needs more memory.
4. **`vmstat`'s `r` and `b` columns distinguish CPU-bound from I/O-bound
   contention directly.** `r` (runnable, waiting for CPU) points at CPU
   contention; `b` (blocked, uninterruptible) points at I/O.
5. **Every number these tools show ultimately comes from `/proc`.** Reading
   `/proc/loadavg` or `/proc/meminfo` directly is more robust than parsing
   formatted tool output when scripting, and occasionally surfaces a number
   no formatted tool shows on its own.
6. **`watch` turns any one-shot command into a live view without needing a
   dedicated monitoring tool.** `-n` sets the refresh interval; `-d`
   highlights what changed between refreshes.
7. **`top` and `htop` let you act on what you find without leaving the
   view.** Sorting, renicing, and killing from inside the tool is faster and
   less error-prone than switching to separate `ps`/`renice`/`kill` commands
   mid-diagnosis.

> **Demo scope:** Primary concept: live/interactive system resource
> monitoring. Supporting concepts: load average interpretation, top/htop
> navigation, vmstat, free and the available-vs-free distinction, /proc
> direct reads, watch, top's interactive process controls.
> Estimated completion time: 40 minutes (reading + hands-on + verification).
> Checkpoints: 6 natural stopping points (after Task 1 through Task 6).

## Quick Commands Reference

### Live view tools

| Command | Purpose |
|---|---|
| `top` | Live process/resource view, aggregate CPU line |
| `htop` | Friendlier live view, per-core bars, tree mode |
| `uptime` | Quick load average + uptime summary |
| `nproc` | Core count — always normalize load average against this |

### Point-in-time / short-interval tools

| Command | Purpose |
|---|---|
| `vmstat N C` | N-second interval, C samples — r/b/swap/CPU snapshot |
| `free -h` | Memory summary — read `available`, not `free` |

### Direct kernel reads

| Command | Purpose |
|---|---|
| `cat /proc/loadavg` | Raw load average + runnable/total task count |
| `cat /proc/meminfo` | Raw memory counters behind every memory tool |

### Live-ifying any command

| Command | Purpose |
|---|---|
| `watch -n N 'cmd'` | Refresh `cmd` every N seconds |
| `watch -d 'cmd'` | Highlight what changed between refreshes |

### top interactive keys

| Key | Effect |
|---|---|
| `1` | Toggle per-core CPU view |
| `M` / `P` | Sort by memory / CPU |
| `r` | Renice a process |
| `k` | Kill a process |

## Appendix A — Anki Cards

**09-live-system-monitoring-anki.csv:**

```
#deck:Linux Mastery::Module 2 - System Administration::09-live-system-monitoring
#separator:Comma
#columns:Front,Back,Tags
"A machine shows load average 6.82 on 4 cores, but top shows 92% idle CPU and wa: 4.3. What does this actually indicate?","An I/O bottleneck, not a CPU bottleneck. Linux counts processes in uninterruptible I/O wait toward load average, so high load with idle CPU and nonzero wa points at disk/network-bound processes, not CPU contention.","demo09,load-average,troubleshooting"
"free -h shows low free but high available. Is the machine low on memory?","No. Linux uses spare RAM for reclaimable page cache (buff/cache); available already accounts for how much of that cache could be reclaimed on demand. Available, not free, is the number that matters.","demo09,free,common-mistake"
"What's the definitive vmstat signal that a machine is genuinely out of memory and swapping, as opposed to just using cache normally?","Non-zero, sustained si/so (swap in/swap out) columns. Normal cache usage shows high buff/cache with si/so near zero; real memory pressure shows active si/so.","demo09,vmstat,swap"
"In vmstat, what's the difference between the r and b columns?","r counts processes runnable and waiting for CPU (CPU contention signal); b counts processes blocked in uninterruptible sleep, almost always I/O (I/O contention signal).","demo09,vmstat,basics"
"Why should you always normalize load average against nproc before judging it?","A raw load average number is meaningless without core count context — 4.0 is trivial on 16 cores and a serious overload on 2 cores.","demo09,load-average,basics"
"What does a high count of zombie processes in top's Tasks line contribute to load average?","Effectively nothing ongoing — a zombie process isn't running or waiting for CPU/IO, it's a dead process waiting to be reaped by its parent. High zombie count is a process-reaping hygiene issue, not an explanation for elevated load average.","demo09,top,zombies"
"Where does every number that top, vmstat, and free display ultimately come from?","/proc — the kernel's live virtual filesystem view of its own state, e.g. /proc/loadavg and /proc/meminfo.","demo09,proc,fundamentals"
"What does watch -d add compared to plain watch -n 2 'cmd'?","-d highlights what changed between refreshes, making it easier to spot new entries (e.g. new connections in ss output) rather than re-reading the whole table each cycle.","demo09,watch,basics"
"Inside top, which key renices a process, and which kills one?","r renices (prompts for PID then new niceness value); k kills (prompts for PID then signal, default SIGTERM).","demo09,top,interactive-controls"
"A system has thin buff/cache (120Mi) alongside active vmstat si/so. Why does the thin cache make the swapping more concerning, not less?","Because the kernel prefers reclaiming cache over swapping whenever it can. A system that's already swapping while cache is thin has already exhausted what it could reclaim and still needs more memory than it has.","demo09,vmstat,free,troubleshooting"
```

## Appendix B — Quiz

**09-live-system-monitoring-quiz.md:**

````markdown
# Quiz — Demo 09: Live System Monitoring

> One correct answer per question unless stated otherwise.
> Target: 80% or above before moving to Demo 10.

**Q1. A 4-core machine shows load average 6.82, 6.40, 5.95, but `top` shows 92% idle CPU and `wa: 4.3`. What's the correct diagnosis?**

- A) Severe CPU overload — the load average alone confirms it
- B) An I/O bottleneck — CPU is mostly idle, and load average includes uninterruptible I/O wait
- C) A memory leak — high load average always indicates memory pressure
- D) The load average figure must be a monitoring bug, since CPU is idle

<details>
<summary>Answer</summary>

**B** — Load average counts uninterruptible I/O wait, not just CPU contention. High load with idle CPU and nonzero `wa` points squarely at I/O-bound processes. Trap: A ignores the idle CPU evidence entirely; C conflates load average with a specific cause it doesn't measure; D wrongly assumes idle CPU rules out a real load average value.

</details>

---

**Q2. `free -h` shows 89Mi free, 4.5Gi available. What's the correct read?**

- A) The system is nearly out of memory — 89Mi is dangerously low
- B) The system is fine — available accounts for reclaimable cache, and it's high
- C) This output is contradictory and indicates a monitoring error
- D) Swap will imminently be needed since free is so low

<details>
<summary>Answer</summary>

**B** — `available` already factors in reclaimable `buff/cache`; a high `available` alongside low `free` is the normal, expected pattern on a healthy Linux system, not a warning sign. Trap: A and D both treat `free` as the meaningful figure, which is the most common misreading of this output; C assumes a discrepancy that isn't one.

</details>

---

**Q3. Which `vmstat` columns are the clearest signal of genuine memory pressure (not just normal cache usage)?**

- A) `buff` and `cache`
- B) `us` and `sy`
- C) `si` and `so`
- D) `in` and `cs`

<details>
<summary>Answer</summary>

**C** — Sustained non-zero `si`/`so` (swap in/out) is the unambiguous sign the kernel is actively moving pages to/from disk due to genuine memory pressure. Trap: A is high during totally normal cache usage, not a pressure signal on its own; B and D are CPU/interrupt-related, not memory-related.

</details>

---

**Q4. In `vmstat`, a machine shows elevated `b` and near-zero `r`. What does this indicate?**

- A) CPU contention — many processes are waiting for a CPU slot
- B) I/O contention — processes are blocked in uninterruptible sleep, not waiting on CPU
- C) The machine is idle
- D) Swap is active

<details>
<summary>Answer</summary>

**B** — `b` counts processes blocked in uninterruptible sleep, almost always I/O-related; `r` (near zero here) would indicate CPU contention if elevated instead. Trap: A misreads which column signals CPU contention; C ignores the elevated `b`; D requires checking `si`/`so`, not `b`, which measures something different.

</details>

---

**Q5. Why should load average always be normalized against `nproc` before judging it?**

- A) It's a stylistic preference, not functionally necessary
- B) Because the raw number's severity depends entirely on how many cores the machine has
- C) Because load average is reported in different units on different distros
- D) nproc converts load average from a percentage to an absolute count

<details>
<summary>Answer</summary>

**B** — The same raw load average number can be trivial or severe depending on core count; a load average of 4.0 is fine on 16 cores and an overload on 2. Trap: A understates how load-bearing this step actually is for a correct read; C and D invent mechanics that aren't real.

</details>

---

**Q6. `top`'s Tasks line shows "9 zombie" alongside a high load average. What's the correct interpretation?**

- A) The 9 zombie processes are the direct cause of the high load average
- B) Zombies consume significant CPU while waiting to be reaped
- C) Zombies aren't running or I/O-blocked — high zombie count is a reaping hygiene issue, not the source of an elevated load average
- D) Zombie processes count double toward load average

<details>
<summary>Answer</summary>

**C** — A zombie process is dead, waiting only for its parent to call `wait()` — it consumes no CPU and isn't running or blocked on I/O, so it isn't an ongoing contributor to load average. High zombie counts point at a separate problem (a parent not reaping children). Trap: A, B, and D all invent a direct load-average mechanism zombies don't have.

</details>

---

**Q7. What's the main practical reason to reach for `/proc/loadavg` directly instead of parsing `top`'s formatted output in a script?**

- A) `/proc/loadavg` updates faster than `top`
- B) `top`'s formatted output is fragile to parse and can change between versions; `/proc/loadavg` is one stable, clean line
- C) `top` requires root privileges to read; `/proc/loadavg` does not
- D) `/proc/loadavg` includes historical data that `top` discards

<details>
<summary>Answer</summary>

**B** — Scripting against formatted, human-oriented tool output is fragile; `/proc/loadavg` is a stable, minimal, machine-friendly source for the same data. Trap: A and D invent capabilities `/proc/loadavg` doesn't have beyond being a plain data source; C is factually wrong — neither requires root for basic reads.

</details>

---

**Q8. Inside `top`, which single key lets you toggle between an aggregate CPU line and a per-core breakdown?**

- A) `M`
- B) `P`
- C) `1`
- D) `r`

<details>
<summary>Answer</summary>

**C** — `1` toggles the per-core CPU view in `top`. Trap: A sorts by memory, B sorts by CPU (the default), D renices a process — all real `top` keys, just not the one this question asks about.

</details>
````