# Demo 09 — Live System Monitoring

## Overview

Every diagnosis of a slow or misbehaving server starts with the same question:
what is this machine doing right now? This demo covers the live, interactive
tools you reach for first — `top`, `htop`, `vmstat`, `free` — plus reading the
kernel's own live counters directly through `/proc`, and turning any one-shot
command into a running dashboard with `watch`. By the end, you'll be able to
read a load average number and know immediately whether it's fine or a
problem, and act on a runaway process without leaving `top`.

---

## Demo Objectives

By the end of this demo you will be able to:

1. Read and interpret load average (1/5/15 min) correctly against core count
2. Navigate `top` and `htop`'s live views: CPU, memory, per-core breakdown, I/O wait
3. Read `vmstat` output to spot memory pressure and CPU contention at a glance
4. Read `free` output correctly, including why "available" is not "free"
5. Read `/proc/loadavg` and `/proc/meminfo` directly, and know when to bother
6. Use `watch` to turn any command into a live-refreshing view
7. Sort, renice, and kill a process from inside `top` without leaving it

---

## Directory Structure

```
09-live-system-monitoring/
├── README.md                          # this file
├── 09-live-system-monitoring-anki.csv # Anki flashcard deck
└── 09-live-system-monitoring-quiz.md  # standalone quiz
```

---

## Environments

| Task | Ubuntu 24.04 | Rocky Linux 9.8 | EC2 Amazon Linux |
|---|---|---|---|
| All lab tasks | ✅ | ✅ | Not required |
| `htop` install | `sudo apt install htop` | `sudo dnf install htop` | Not required |

---

## Recall Check

> Answer from memory before reading further. From Demo 08.

1. A `.timer` unit has no `Unit=` line under `[Timer]`. Which service does
   it activate when it fires?

2. You set `Persistent=true` on a daily timer. The machine is off for
   exactly one missed run, then powered back on. How many catch-up runs
   happen?

3. You run `systemctl list-timers` with no flags and don't see a timer you
   know exists. What's the first thing you check, before assuming it's
   broken?

<details>
<summary>Answers</summary>

1. The identically-named `.service` unit — a `.timer` activates the
   `.service` of the same base name by default unless `Unit=` explicitly
   points elsewhere.

2. One — `Persistent=true` fires a single catch-up run for the most recent
   missed occurrence, not one run per missed day.

3. Re-run with `--all` — by default `list-timers` only shows currently
   active timers; a disabled or inactive timer won't appear without the flag.

</details>

---

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

Reading the CPU line, field by field:

```
us   User processes — time spent running normal (non-niced) user-space code
sy   System — time spent in the kernel servicing system calls on behalf of
     processes (not the kernel's own background work, but work done for a caller)
ni   Niced — time spent running processes that have a positive nice value
     (Demo 06); separated out so you can see how much CPU low-priority work
     is actually consuming
id   Idle — CPU genuinely has nothing to do
wa   I/O wait — CPU idle SPECIFICALLY because it's waiting on disk/network
     I/O, not truly free. A high wa with low us/sy points at a storage or
     network bottleneck, not a CPU one — a common misdiagnosis is treating
     high wa as "CPU is busy" and looking for a runaway process, when the
     real fix is on the I/O side.
hi   Hardware interrupts — time spent servicing hardware interrupt requests
si   Software interrupts — time spent servicing software-triggered interrupts
st   Stolen — time a virtualized guest wanted to run but its hypervisor gave
     the physical CPU to a different guest instead. Only meaningful on VMs/
     cloud instances; on bare metal this is always 0. A rising st with
     otherwise-normal us/sy/id means the noisy-neighbor problem is on the
     hypervisor's side, not something fixable from inside this guest at all.
```

`htop` shows the same underlying data with a friendlier layout: **per-core
bars across the top** — so you see imbalance across cores at a glance, which
plain `top`'s single aggregate `%Cpu(s)` line hides entirely (a process
pegging one core to 100% on an otherwise-idle 8-core machine shows as a modest
~12% in `top`'s aggregate line, but jumps out immediately as one maxed-out bar
in `htop`) — plus a **tree view** (`F5`) showing live parent/child process
relationships, similar to `pstree` but continuously updating rather than a
single snapshot, and mouse-clickable actions.

```bash
sudo apt install htop   # Ubuntu
sudo dnf install htop   # Rocky9
htop
```

**Worth knowing it exists:** `glances` is a newer, more ambitious alternative
that adds network throughput, disk I/O, and container awareness to the same
live-view concept in one dashboard — information that would otherwise mean
running several separate tools side by side. It's not covered hands-on here —
`top`/`htop` cover the fundamentals every environment has available — but it's
common enough in modern DevOps toolchains that recognizing the name is
worthwhile.

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
disk).

**`swpd` vs `si`/`so` — a point-in-time amount versus an active rate:**
`swpd` is how much memory is *currently* sitting swapped out to disk right
now — a non-zero `swpd` on its own isn't alarming; Linux can swap out a
long-idle process's memory once and leave it there indefinitely with no
ongoing performance cost, since nothing is actively touching it. `si`/`so`
(swap in/swap out) are the *rate* — pages actively moving to/from disk right
now. A large `swpd` with `si`/`so` at zero means "some memory was swapped out
at some point, and it's just sitting there, harmless." A small `swpd` with
active nonzero `si`/`so` means pages are being actively fought over right
now — that's the real, ongoing performance problem, not the `swpd` total by
itself.

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

---

## Part 2 — Hands-On Lab

### Commands used in this demo

| Command | Letter meaning | Purpose |
|---|---|---|
| `uptime` | — | Quick load average + uptime summary |
| `nproc` | number of processors | Core count, for normalizing load average |
| `top` | table of processes | Live process/resource view |
| `htop` | — | Friendlier live view with per-core bars and tree mode |
| `vmstat N C` | `N`: interval seconds, `C`: count | Fast r/b/swap/CPU snapshot over time |
| `free -h` | `h` = human-readable | Memory summary, correctly read via `available` |
| `watch -n N 'cmd'` | `n` = interval | Turn any command into a live view |
| `watch -d 'cmd'` | `d` = highlight diffs | Spot what changed between refreshes |

### Task 1 — Load average, normalized against core count

```bash
nproc
uptime
# Manually compute: load average ÷ core count for each of the three numbers.
# Is this machine's current demand comfortable, borderline, or oversubscribed?
```

### Task 2 — top and htop: read the CPU line under synthetic load

```bash
# Generate CPU load in the background:
yes > /dev/null &
yes > /dev/null &

top
# Watch %Cpu(s): us should rise sharply, id should drop. Note which PIDs
# correspond to the yes processes (they'll be at/near the top, sorted by CPU).
```

```bash
# Now install and open htop to see the same load rendered differently:
sudo apt install -y htop   # Ubuntu
sudo dnf install -y htop   # Rocky9
htop
# Observe the per-core bars at the top — on a multi-core machine, the two
# yes processes should each peg roughly one core near 100% while other
# cores stay comparatively idle. This is exactly the imbalance top's single
# aggregate %Cpu(s) line would hide. Press F5 for tree view and confirm both
# yes processes appear as children of this shell, live-updating — compare
# this against a one-shot pstree from Demo 06.

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

---

## What You Learned

- ✅ Read and interpreted load average (1/5/15 min) correctly against core count
- ✅ Navigated `top` and `htop`'s live views: CPU, memory, per-core breakdown, I/O wait
- ✅ Read `vmstat` output to spot memory pressure and CPU contention at a glance
- ✅ Read `free` output correctly, distinguishing `available` from `free`
- ✅ Read `/proc/loadavg` and `/proc/meminfo` directly
- ✅ Used `watch` to turn a one-shot command into a live-refreshing view
- ✅ Sorted, reniced, and killed a process from inside `top` without leaving it

**Key Takeaway:** Every monitoring question in this demo comes down to reading the *right* column instead of trusting the headline number — load average needs `%Cpu(s)`'s `wa`/`id` for context, `free` needs `available` instead of `free`, and `vmstat`'s `si`/`so` (not `swpd` alone) is the only reliable tell for real memory pressure. These tools deliberately overlap; when one number looks alarming, a second tool's column is usually what confirms or rules it out.

---

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

---

## Interview Prep

**Q1. A colleague says "load average is high, so the CPU must be the bottleneck." When is this reasoning wrong, and what would you check instead?**

It's wrong whenever the elevated load average is driven by processes in
uninterruptible I/O wait rather than CPU contention — Linux includes both in
the load average calculation. The check is `top`'s `%Cpu(s)` line: high `id`
(idle) combined with nonzero `wa` (I/O wait) means the CPUs aren't busy at
all; the load is coming from disk or network-bound processes. `vmstat`'s `b`
column (blocked, uninterruptible) versus `r` column (runnable, CPU-contending)
gives the same distinction more directly.

---

**Q2. `free -h` shows very little "free" memory but plenty of "available." Is this machine actually low on memory?**

No, not necessarily — this is one of the most common false alarms in Linux
memory monitoring. Linux deliberately uses otherwise-idle RAM for page cache
and other reclaimable caches (`buff/cache`), because unused RAM has no value
sitting idle. `available` is the number that already accounts for how much of
that cache the kernel could hand back to a new process on demand — that's the
figure that answers "how much memory can I actually use," not `free`.

---

**Q3. How would you tell the difference between "the system is using cache normally" and "the system is genuinely out of memory and swapping"?**

Check `vmstat`'s `si`/`so` columns — sustained non-zero swap-in/swap-out
activity is the unambiguous signal of real memory pressure, since it means the
kernel is moving pages to and from disk to make room, something it avoids
unless it has to. Normal cache usage shows up as high `buff/cache` with `si`/
`so` at or near zero. Also check whether `buff/cache` itself is unusually
small for the machine's total RAM — a thin cache alongside active swapping
means the system has already given up what it could reclaim and still needs
more memory than it has.

---

**Q4. Why would you reach for `vmstat` instead of `top` when triaging a suspected performance issue over SSH on a slow connection?**

`vmstat`'s output is compact, line-based, and easy to capture with `>>` into a
log or pipe into another tool — `top`'s default full-screen redraw is harder
to work with over high-latency connections and awkward to script. `vmstat N
C` (interval and count) also gives a clean time series in one command,
whereas getting the same trend from `top` means watching it live or using its
batch mode (`top -b -n`) with extra parsing.

---

## Key Takeaways

1. **Load average includes I/O wait, not just CPU contention.** A high load
   average with low CPU usage and nonzero `wa` in `top` almost always means
   an I/O bottleneck, not a CPU one — normalize against `nproc` first, then
   check `%Cpu(s)` before assuming the CPU is the problem.
2. **`free`'s "available" column is the number that matters, not "free."**
   Linux uses spare RAM for reclaimable page cache; low `free` with high
   `available` is completely normal, not a warning sign.
3. **`vmstat`'s `si`/`so` columns are the definitive swap-activity signal —
   `swpd` alone is not.** `swpd` is a point-in-time amount that can sit
   harmlessly non-zero; `si`/`so` are the active rate that signals genuine,
   ongoing memory pressure.
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
   mid-diagnosis. `htop`'s per-core bars also reveal imbalance across cores
   that `top`'s single aggregate CPU line hides entirely.

---

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

---

## Appendix A — Anki Cards

**`09-live-system-monitoring-anki.csv`:**

```
#deck:Linux Mastery::Module 2 - System Administration::Demo 09 - Live System Monitoring
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

---

## Appendix B — Quiz

**`09-live-system-monitoring-quiz.md`:**

````markdown
# Quiz — Demo 09: Live System Monitoring

> One correct answer per question unless stated otherwise.
> Target: 80% or above before moving to Demo 10.

---

**Q1.** A process pegs one core to 100% on an otherwise-idle 8-core machine.
In plain `top`'s aggregate `%Cpu(s)` line, this shows as roughly 12% usage.
Why does `htop` make this situation obvious at a glance where `top` doesn't?

- A) `htop` refreshes faster than `top`, catching spikes `top` misses
- B) `htop` displays per-core bars across the top of the screen, so one
  maxed-out core is immediately visible as one full bar among seven idle
  ones — `top`'s single aggregate line averages this imbalance away
- C) `htop` runs the process at a different priority, making the spike more visible
- D) `top` cannot display CPU usage above 100% at all, so it's simply wrong here

<details>
<summary>Answer</summary>

**B** — Aggregating across all cores into one percentage is exactly what
hides a single-core bottleneck; `htop`'s per-core bars show the imbalance
directly instead of averaging it away.

</details>

---

**Q2.** A cloud VM's `top` output shows `st: 6.2` in the CPU line while
`us`/`sy`/`id` otherwise look normal. What does this indicate, and where
would you look to actually address it?

- A) 6.2% of local disk I/O is being stolen by another process — check `iotop`
- B) "Stolen" time — the hypervisor gave the physical CPU to a different
  guest when this VM wanted it. This isn't fixable from inside the guest;
  it points at contention on the host/hypervisor side
- C) A misconfigured nice value is stealing CPU from other processes on this VM
- D) `st` only appears due to a kernel bug and can be safely ignored

<details>
<summary>Answer</summary>

**B** — `st` (stolen) only has meaning in virtualized environments and
reflects contention for the physical CPU at the hypervisor level, entirely
outside what's visible or fixable from inside the guest OS itself.

</details>

---

**Q3.** `top`'s CPU line shows a nonzero `ni` value. What is this measuring?

- A) Time spent on processes with a positive nice value (Demo 06) — lower
  scheduling priority than default
- B) Network I/O time
- C) The percentage of processes currently in a "new" (just-created) state
- D) Time spent on kernel-internal maintenance tasks

<details>
<summary>Answer</summary>

**A** — `ni` isolates CPU time spent on niced (deprioritized) processes,
separately from ordinary `us` (user) time — letting you see how much CPU
low-priority background work is actually consuming.

</details>

---

**Q4.** `vmstat` shows `swpd: 1200000` but `si`/`so` both at 0, consistently
across several samples. Is this machine under active memory pressure right
now?

- A) Yes — any nonzero `swpd` means the system is actively swapping
- B) Not necessarily — `swpd` is a point-in-time amount already sitting
  swapped out, which can be harmless if nothing is currently touching it;
  `si`/`so` at 0 means no active swap traffic is happening right now
- C) No — `swpd` only becomes meaningful when `si`/`so` are also nonzero,
  and a zero `si`/`so` means the `swpd` value itself is stale or invalid
- D) This combination is impossible; `swpd` cannot be nonzero while `si`/`so` are zero

<details>
<summary>Answer</summary>

**B** — `swpd` and `si`/`so` measure different things: an amount versus a
rate. Memory swapped out once and left alone (because nothing needs it back)
shows exactly this pattern — nonzero `swpd`, zero `si`/`so` — and isn't an
active problem. Trap: C invents a dependency between the two that doesn't
exist; `swpd` is a perfectly valid standalone snapshot value.

</details>

---

**Q5.** Why would you reach for `vmstat` rather than `top` when triaging a
performance issue over a slow SSH connection?

- A) `vmstat` requires less CPU overhead to run than `top`
- B) `vmstat`'s output is compact and line-based — easy to capture, log, or
  pipe — while `top`'s full-screen redraw is harder to work with over high
  latency and awkward to script; `vmstat N C` also gives a clean time series
  in one command
- C) `top` cannot be run over SSH at all without X11 forwarding
- D) `vmstat` and `top` show identical information, so the choice is arbitrary

<details>
<summary>Answer</summary>

**B** — The practical reasons are about output format and scriptability over
a constrained connection, not raw resource cost or missing functionality in
`top`.

</details>

---

**Q6.** Inside `top`, which single key toggles between an aggregate CPU line
and a per-core breakdown?

- A) `M`
- B) `P`
- C) `1`
- D) `r`

<details>
<summary>Answer</summary>

**C** — `1` toggles the per-core CPU view in `top`. Trap: A sorts by memory,
B sorts by CPU (the default), D renices a process — all real `top` keys,
just not the one this question asks about.

</details>

---

**Q7.** What does `glances` add on top of what `top`/`htop` already provide,
per this demo's brief mention of it?

- A) Nothing meaningfully different — it's a purely cosmetic reskin of `top`
- B) Network throughput, disk I/O, and container awareness combined into one
  dashboard — information that otherwise means running several separate
  tools side by side
- C) It replaces `vmstat` entirely and is required for any memory diagnosis
- D) It only works inside Docker containers, not on bare-metal hosts

<details>
<summary>Answer</summary>

**B** — `glances` is a broader, single-dashboard tool that folds in network,
disk, and container data alongside process/CPU/memory — genuinely more scope
than `top`/`htop`, not just a different skin on the same data.

</details>

---

**Q8.** `htop`'s tree view (`F5`) is described as similar to `pstree`. What's
the actual difference between the two for this purpose?

- A) `htop`'s tree view updates live and continuously; `pstree` is a single
  point-in-time snapshot each time you run it
- B) `pstree` shows more detail than `htop`'s tree view ever can
- C) They are functionally identical; `htop`'s version is just prettier
- D) `htop`'s tree view only works for processes owned by the current user

<details>
<summary>Answer</summary>

**A** — The parent/child relationship information is conceptually the same
as `pstree`, but `htop`'s tree view refreshes continuously as part of the
live monitoring session, rather than requiring you to re-run a command for
an updated snapshot.

</details>

---

| Score | Action |
|---|---|
| 8/8 | Import Anki cards, move to Demo 10 |
| 6-7/8 | Review the wrong answers, then proceed |
| 4-5/8 | Re-read the relevant sections, retry those questions |
| Below 4/8 | Re-read the full demo and redo the walkthrough before proceeding |
````