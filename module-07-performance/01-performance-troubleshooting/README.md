# Demo M07-01 — Performance Troubleshooting

## Overview

Demo M02-07/10 covered monitoring — knowing *that* something is slow. This
demo covers the next level down: knowing *why*, at the level of individual
system calls, library calls, and CPU cycles. `strace` and `ltrace` show
exactly what a process is asking the kernel and its libraries to do;
`perf stat` and `perf top` give a CPU-level view of where time actually
goes; `/proc` gives direct access to a specific process's live state. By
the end, you'll have a concrete first move for "this process is slow and I
don't know why," rather than guessing.

## Demo Objectives

1. Trace a process's system calls with `strace` and read the output
2. Trace a process's library calls with `ltrace`, and explain when it's the better choice over `strace`
3. Use `strace -c` for a syscall summary, and `strace -T` for per-call timing
4. Read basic `perf stat` output and identify what it measures
5. Use `perf top` to find which function is actually consuming CPU time in a running process
6. Read process-specific data directly from `/proc/PID/`

## Directory Structure

```
01-performance-troubleshooting/
├── README.md                                    # this file
├── m07-01-performance-troubleshooting-anki.csv  # Anki flashcard deck
└── m07-01-performance-troubleshooting-quiz.md   # standalone quiz
```

## Environments

| Task | Ubuntu 24.04 | Rocky Linux 9.8 | EC2 Amazon Linux |
|---|---|---|---|
| `strace`/`ltrace` | `sudo apt install strace ltrace` | `sudo dnf install strace ltrace` | Not required |
| `perf` | `sudo apt install linux-tools-generic` | `sudo dnf install perf` | Not required |

> **Verification note:** `strace`/`ltrace`/`perf` could not be installed
> or run live in this sandbox (no network access — the same constraint
> already disclosed in `06-cron-log-rotation` and the Vim/Neovim demos).
> §1.1–1.3 and §1.5's sample outputs (syscall percentages, cycle counts,
> IPC figures, `perf top` rows) are illustrative and built from stable,
> well-documented tool behavior — recommend a real verification pass on
> an actual Ubuntu/Rocky9 machine before treating exact figures as
> guaranteed. **§1.6 (`/proc/PID/`) is the exception: its specific claims
> were live-tested directly in this sandbox** (a real backgrounded
> process's `status`, `limits`, `fd/`, and null-byte-separated `cmdline`
> were all confirmed exactly as described) and can be treated as
> confirmed.

## Recall Check — 05-neovim-lsp

Answer these from memory before continuing — no looking back at Demo M06-05.

**Q1.** What's the actual, defensible differentiator between Neovim and
Vim, and what common but incorrect argument does it rule out?

**Q2.** What are the two genuinely separate things that both need to be
true for `gd`/`K` to work at all?

**Q3.** What's the structured first command to run when an LSP setup
seems broken, rather than guessing from a silent failure?

<details>
<summary>Answers</summary>

**A1.** The built-in LSP client (native diagnostics, hover, and
go-to-definition with no plugin). This rules out `:terminal` as a valid
argument — Vim has had an integrated terminal since version 8, so it
isn't a genuine Neovim-exclusive feature.

**A2.** The LSP client needs to be configured to talk to a server (e.g.
`vim.lsp.enable('pylsp')` in `init.lua`), AND the actual language server
binary needs to be separately installed and running — two independent
requirements, not one.

**A3.** `:checkhealth` — it runs structured self-checks specifically
designed to surface setup mismatches like a configured client with a
missing server binary, rather than leaving you to guess from silence.

</details>

## Part 1 — Concepts

### 1.1 strace — tracing system calls

```bash
strace -c ls /tmp > /dev/null
```

```
% time     seconds  usecs/call     calls    errors syscall
------ ----------- ----------- --------- --------- ----------------
 35.20    0.000234          12        19           openat
 22.15    0.000147          15        10           read
 ...
------ ----------- ----------- --------- --------- ----------------
100.00    0.000665                    62         2 total
```

`strace` intercepts and logs every system call a process makes — the
interface between a process and the kernel: file operations, network
calls, memory allocation requests, process/signal operations. `-c`
produces a summary (call counts, time spent, per-syscall breakdown) rather
than a live line-by-line trace, which is usually what you want first —
"which syscall is this process spending the most time in" before drilling
into the specific calls themselves.

```bash
strace -T -e trace=open,read,write ls /tmp
# -T: show the TIME each individual syscall took, appended to each line
# -e trace=...: filter to only these specific syscalls, cutting noise from
# an otherwise very long full trace
```

```bash
strace -p 1234
# -p PID: attach to an ALREADY-RUNNING process, rather than starting a new
# one under strace — essential for diagnosing a process that's already
# stuck or misbehaving, since you can't always restart it under strace
# from scratch without losing the exact conditions that caused the problem.
```

**A process spending most of its time in `read`/`write`/`openat` points at
I/O; heavy time in `futex` or `nanosleep` points at contention or waiting;
heavy time in `mmap`/`brk` points at memory allocation activity.** Reading
the syscall summary's shape is often enough to point at the right category
of problem before looking at anything else.

### 1.2 ltrace — tracing library calls

```bash
ltrace -c ls /tmp > /dev/null
```

`ltrace` is `strace`'s sibling, but at a different layer: instead of
kernel syscalls, it traces calls to shared library functions (`malloc`,
`strlen`, `printf`, and application-specific library functions). **Reach
for `ltrace` when the suspected bottleneck is inside the application's own
logic or a library it depends on — string processing, memory allocation
patterns, repeated redundant library calls — rather than kernel-level I/O
or process interaction**, which is `strace`'s territory. A process making
enormous numbers of `malloc`/`free` calls (visible in `ltrace -c`'s
summary) suggests inefficient memory management in the application code
itself, something `strace`'s syscall-level view wouldn't directly surface.

```bash
ltrace -e malloc+free ./some_program
# Filter to specific library functions, same -e pattern as strace §1.1.
```

### 1.3 perf stat — CPU-level performance counters

```bash
perf stat ls /tmp
```

```
 Performance counter stats for 'ls /tmp':

              1.23 msec task-clock                #    0.412 CPUs utilized
                 2      context-switches          #    1.626 K/sec
                 0      cpu-migrations            #    0.000 K/sec
               105      page-faults               #   85.366 K/sec
         2,340,158      cycles                    #    1.902 GHz
         1,842,203      instructions              #    0.79  insn per cycle
```

`perf stat` reads hardware and kernel performance counters — actual CPU
cycles consumed, instructions executed, cache behavior, context switches —
giving a much lower-level picture than `top`/`vmstat` (Demo M02-07/10). **The
`instructions per cycle` (IPC) figure is a genuinely useful efficiency
signal**: a low IPC (well under 1.0) can indicate the CPU is frequently
stalled waiting on memory rather than doing useful work, even while
`%CPU` in `top` shows the process as "using" a full core — `top`'s CPU%
measures time scheduled on a core, not how efficiently that time is
actually being used.

```bash
perf stat -p 1234 sleep 10
# -p PID: attach to a running process for 10 seconds and report counters
# for that observation window, rather than running a fresh command.
```

`perf stat` tells you **how efficient** the CPU time was (via IPC) — it
doesn't tell you **which function** in the code is consuming that time.
That's what `perf top` (§1.5) is for.

### 1.4 Reading strace's summary as a first triage step

Before reaching for `perf` or `ltrace`, `strace -c`'s shape alone often
narrows the problem:

```
Heavy read/write/openat  → I/O
Heavy futex/nanosleep    → contention or waiting
Heavy mmap/brk           → memory allocation activity
Almost no syscall time   → the time is in pure user-space computation —
                            strace structurally cannot see it; reach for
                            ltrace (§1.2) or perf top (§1.5) instead
```

### 1.5 perf top — finding the hot function live

```bash
perf top -p 1234
```

```
  47.32%  libc.so.6        [.] __memmove_avx_unaligned
  18.05%  myapp            [.] parse_record
   9.41%  myapp            [.] validate_checksum
   6.88%  [kernel]         [.] copy_page
```

`perf top` samples the running process continuously and shows a live,
ranked view of which function is consuming CPU time right now — a
different question from `perf stat`'s "how efficient was this run
overall." Where `strace`/`ltrace` show *calls being made*, `perf top`
shows *time spent inside specific functions*, including ones you never
called directly (`__memmove_avx_unaligned` here is glibc's memory-copy
routine, likely invoked from inside `myapp` rather than named explicitly
anywhere in its own source).

```bash
perf top -p 1234 --sort comm,dso
# --sort: group the live view by command and by the library/binary
# (dso = "dynamic shared object") each sample landed in, useful for
# separating "time in my code" from "time in a library" at a glance.
```

**When `perf top` is the right next step, versus `ltrace`:** both look
inside a process rather than at the kernel boundary. `ltrace -c` gives you
call *counts* — how many times each library function was called, which is
the right question when the suspicion is "something is being called far
too often." `perf top` gives you a live, sampled view of where CPU time is
*actually spent right now*, which is the right question when the
suspicion is "something is slow but I don't know which function," without
needing to guess which library calls to filter for in advance.

### 1.6 /proc/PID — process-specific live data

Demo M02-07 and Demo M04-07 covered `/proc/loadavg` and `/proc/meminfo` —
system-wide data. Every running process also has its own directory under
`/proc/PID/`:

```bash
cat /proc/1234/status
# State, memory usage (VmRSS, VmSize), thread count, signal masks — a
# detailed per-process snapshot, more granular than what ps/top show by
# default.

cat /proc/1234/limits
# The actual resource limits (ulimit) in effect for THIS specific process
# — confirming what limits actually apply, rather than assuming based on
# shell configuration that may not match what the process inherited.

ls -la /proc/1234/fd/
# Every file descriptor the process currently has open — the direct,
# authoritative source lsof (Demo M03-02) actually reads from; useful when you
# want the raw view without lsof's formatting/filtering layer.

cat /proc/1234/cmdline | tr '\0' ' '
# The exact command line the process was started with, null-byte
# separated in the raw file — tr converts nulls to spaces for readability,
# since cmdline's raw format uses null bytes (not spaces) between
# arguments.
```

**`/proc/PID/` is the ground truth every process-inspection tool (`ps`,
`top`, `lsof`) ultimately reads from** — when a formatted tool's output is
ambiguous or seems wrong, reading the raw `/proc/PID/` files directly is
the way to confirm what's actually true, the same "go to the source" habit
Demo M04-07 established for `/proc`/`/sys` generally.

## Part 2 — Hands-On Lab

### Commands used in this demo

| Command | Flag meaning | Purpose |
|---|---|---|
| `strace -c CMD` | `-c`: summary | Syscall count/time breakdown |
| `strace -T -e trace=X,Y CMD` | `-T`: per-call timing, `-e`: filter | Detailed, scoped trace |
| `strace -p PID` | `-p`: attach | Trace an already-running process |
| `ltrace -c CMD` | `-c`: summary | Library-call count/time breakdown |
| `perf stat CMD` | — | CPU-level performance counters |
| `perf stat -p PID CMD` | `-p`: attach | Counters for a running process |
| `perf top -p PID` | `-p`: attach | Live, ranked view of which function consumes CPU time |
| `cat /proc/PID/status` | — | Detailed per-process state |
| `cat /proc/PID/limits` | — | Actual resource limits in effect |

### Task 1 — strace summary and a targeted trace

```bash
strace -c ls /tmp > /dev/null
strace -T -e trace=openat,read ls /tmp 2>&1 | head -10
```

### Task 2 — strace attaching to a running process

```bash
sleep 60 &
SLEEP_PID=$!
timeout 3 strace -p "$SLEEP_PID" 2>&1 | head -5
kill "$SLEEP_PID" 2>/dev/null
```

### Task 3 — ltrace and comparing against strace's focus

```bash
ltrace -c ls /tmp > /dev/null
echo "--- compare against strace -c for the same command ---"
strace -c ls /tmp > /dev/null
```

### Task 4 — perf stat and reading IPC

```bash
perf stat ls -la /tmp 2>&1 | tail -15
# Identify the instructions-per-cycle figure and note whether it's closer
# to 1.0 (efficient) or notably lower (memory-stalled).
```

### Task 5 — perf top on a CPU-bound process

```bash
yes > /dev/null &
YES_PID=$!
timeout 5 perf top -p "$YES_PID" --sort comm,dso 2>&1 | head -15
kill "$YES_PID" 2>/dev/null
```

Identify the top-ranked function. For `yes`, expect most time to land
inside the C library's `write`/output path rather than in `yes` itself —
consistent with §1.4's observation that a process making heavy,
repetitive syscalls spends much of its time in kernel-adjacent library
code, not its own logic.

### Task 6 — /proc/PID direct inspection

```bash
sleep 60 &
SLEEP_PID=$!
cat /proc/$SLEEP_PID/status | head -10
cat /proc/$SLEEP_PID/limits | head -5
ls -la /proc/$SLEEP_PID/fd/
cat /proc/$SLEEP_PID/cmdline | tr '\0' ' '; echo
kill "$SLEEP_PID" 2>/dev/null
```

### Task 7 — Choosing the right tool for a symptom

```bash
# Given a process reported as "slow, seems to be doing a lot of file I/O":
# which tool from this demo would you reach for FIRST?
# Given a process reported as "high CPU but I don't know which function":
# which tool from this demo would you reach for FIRST?
# (No command to run — a judgment call, then check against §1.1-§1.5.)
```

## What You Learned

- ✅ Traced a process's system calls with `strace` and read the output
- ✅ Traced a process's library calls with `ltrace`, and when it's the
  better choice
- ✅ Used `strace -c` for a syscall summary, and `-T` for per-call timing
- ✅ Read basic `perf stat` output, including the IPC efficiency signal
- ✅ Used `perf top` to find which function is actually consuming CPU
  time in a running process
- ✅ Read process-specific data directly from `/proc/PID/`

## Part 3 — Break-Fix Scenario

### Scenario — NovaTech Ops: two performance incidents

> **Incident report — NovaTech Ops, ticket #4761**
>
> Alert 1: "app-server-09's request handler is slow, `top` shows it at 98%
> CPU, but throughput is much lower than a healthy instance at the same
> CPU%." Alert 2: "a batch job is taking 10x longer than expected, and
> `strace -c` shows almost no time in any syscall at all."

**Incident 1 — high CPU%, but low throughput compared to a healthy instance**

```bash
$ perf stat -p 8842 sleep 5
         8,200,412      cycles
         2,050,103      instructions          #    0.25  insn per cycle
$ perf top -p 8842 --sort comm,dso
  61.20%  myapp            [.] recompute_hash
  14.03%  libc.so.6        [.] __memcpy_avx_unaligned
```

Tasks:
1. `top` shows 98% CPU utilization for this process. Reference §1.3 — does
   high CPU% necessarily mean the CPU is doing proportionally useful work?
2. What does an IPC of 0.25 suggest is actually happening during those CPU
   cycles?
3. Given `perf top` names `recompute_hash` as the dominant function, what
   would be a reasonable next diagnostic step?

**Incident 2 — batch job slow, but strace -c shows almost no syscall time**

```bash
$ strace -c ./batch-job.sh
% time     seconds  usecs/call     calls    errors syscall
------ ----------- ----------- --------- --------- ----------------
  2.10    0.000045           5         9           read
  1.80    0.000038           4         9           write
------ ----------- ----------- --------- --------- ----------------
100.00    0.000083                    18         0 total
$ # total traced time: 0.000083 seconds, but the job took 45 real seconds
```

Tasks:
1. Given `strace -c`'s own numbers show almost no time spent in syscalls,
   but the job clearly took 45 seconds, where is the time actually going?
2. Reference §1.4/§1.5 — name two tools from this demo that could each
   help narrow this down, and what each one would specifically tell you
   that the other wouldn't.

<details>
<summary>Answers</summary>

**Incident 1:** No — `top`'s CPU% measures time the process was actively
scheduled on a core, not how efficiently that scheduled time was used;
`perf stat`'s IPC (instructions per cycle) is the metric that actually
measures efficiency. An IPC of 0.25 is well below the 1.0 baseline
suggested in §1.3, indicating the CPU is spending a large share of its
cycles stalled — most commonly waiting on memory access (cache misses,
memory latency) rather than executing useful instructions — while still
appearing "busy" (98%) from a pure scheduling perspective. With `perf top`
already naming `recompute_hash` as the dominant function, the reasonable
next step is examining that specific function's memory access pattern
(poor cache locality, excessive pointer-chasing, an unexpectedly large
working set) rather than assuming the fix is "add more CPU cores" or
"reduce instruction count" — the bottleneck is memory stalls per
instruction inside a function you can now name specifically, not a vague,
whole-process guess.

**Incident 2:** The time is going somewhere `strace` structurally cannot
see — pure CPU-bound computation happening entirely in user space,
without making system calls at all. `strace` only ever sees the
kernel-boundary-crossing activity (syscalls); a process doing intensive
computation, string processing, or library-internal work between syscalls
is invisible to `strace`'s summary, which is exactly why its total traced
time (0.000083s) is nowhere close to the job's actual 45-second runtime.
Two reasonable next tools: `ltrace -c`, which would show *how many times*
library functions are being called — useful if the suspicion is
"something redundant is being called far too often" (an enormous
`malloc`/`free` count, for instance); and `perf top`, which would show
*which function, live*, is consuming the CPU time right now, without
needing to guess which library calls to filter for in advance — useful
when there's no specific suspicion yet about which function is the
problem.

</details>

## Interview Prep

**Q1. A process shows 98% CPU in `top`. Does that alone tell you the process is doing proportionally useful work?**
No — `top`'s CPU% measures time scheduled on a core, not how efficiently
that time is used. `perf stat`'s instructions-per-cycle (IPC) figure is the
actual efficiency signal; a low IPC despite high CPU% indicates the CPU is
frequently stalled (commonly on memory access) rather than executing
useful instructions, even though it's technically "busy" the whole time
from a scheduling perspective.

**Q2. A job runs far longer than expected, but `strace -c` shows almost no time spent in any syscall. Where's the time going, and what tool would you reach for next?**
The time is in pure user-space computation — work happening entirely
within the application and its libraries, between syscalls, which `strace`
structurally cannot observe since it only sees kernel-boundary crossings.
`ltrace -c` or `perf top` are the appropriate next steps, depending on
whether the suspicion is "something is called too often" (`ltrace`) or
"I don't know which function is slow" (`perf top`).

**Q3. When would you reach for `perf top` instead of `ltrace` once you've established the bottleneck is in user-space computation?**
`ltrace -c` answers "how many times is each library function being
called" — the right question when redundant calls are the suspicion.
`perf top` answers "which function, live, is actually consuming CPU time
right now" — the right question when you have no specific suspicion yet
about which function is slow, including functions you never called
directly (library internals invoked on your behalf, like a memory-copy
routine).

**Q4. Why does `strace -c`'s syscall-time breakdown matter as a first step, before reaching for `ltrace` or `perf`?**
Its shape narrows the category of problem before any deeper tool is
needed: heavy `read`/`write`/`openat` points at I/O, heavy
`futex`/`nanosleep` points at contention, heavy `mmap`/`brk` points at
memory allocation — and almost no syscall time at all is itself the
signal that the problem is pure user-space computation, which is exactly
when `ltrace`/`perf top` become the right next tools instead.

---

## Key Takeaways

1. **`strace` sees kernel-boundary activity (syscalls); `ltrace` sees
   library-call activity within user space.** A slow process spending its
   time in pure computation between syscalls is invisible to `strace` —
   that's exactly when `ltrace` or `perf top` become the right next tool.
2. **High CPU% in `top` doesn't mean the CPU is doing proportionally
   useful work.** `perf stat`'s instructions-per-cycle (IPC) is the actual
   efficiency signal — a low IPC despite high CPU% points at memory stalls,
   not computation volume.
3. **`strace -c`'s summary — which syscalls dominate time — points at a
   category of problem before you look at anything else.** Heavy
   `read`/`write`/`openat` suggests I/O; heavy `futex`/`nanosleep`
   suggests contention/waiting; heavy `mmap`/`brk` suggests memory
   allocation activity.
4. **`perf stat` measures how efficient CPU time was; `perf top` shows
   which function it's actually being spent in, live.** They answer
   different questions — efficiency versus location — and often get used
   together, stat first, top second.
5. **`ltrace -c` counts calls; `perf top` samples where time is spent.**
   Reach for `ltrace` when the suspicion is "something is called too
   often"; reach for `perf top` when there's no specific suspicion yet
   about which function is slow.
6. **`/proc/PID/` is the ground truth every process-inspection tool
   ultimately reads from.** When `ps`/`top`/`lsof`'s formatted output is
   ambiguous, reading `/proc/PID/status`, `/limits`, or `/fd/` directly
   confirms what's actually true.

> **Demo scope:** Primary concept: diagnosing performance problems at the
> syscall and CPU-profiling level. Supporting concepts: strace, ltrace,
> perf stat and perf top, /proc/PID deep dive.
> Estimated completion time: 32 minutes (reading + hands-on + verification).
> Checkpoints: 7 natural stopping points (after Task 1 through Task 7).

---

## Quick Commands Reference

### System call tracing

| Command | Purpose |
|---|---|
| `strace -c CMD` | Syscall count/time summary |
| `strace -T -e trace=X,Y CMD` | Scoped trace with per-call timing |
| `strace -p PID` | Attach to a running process |

### Library call tracing

| Command | Purpose |
|---|---|
| `ltrace -c CMD` | Library-call count/time summary |
| `ltrace -e FUNC1+FUNC2 CMD` | Scoped to specific functions |

### CPU-level profiling

| Command | Purpose |
|---|---|
| `perf stat CMD` | Cycles, instructions, IPC, context switches |
| `perf stat -p PID CMD` | Counters for a running process over a window |
| `perf top -p PID` | Live, ranked view of which function consumes CPU time |
| `perf top -p PID --sort comm,dso` | Same, grouped by command and library/binary |

### Direct process inspection

| Command | Purpose |
|---|---|
| `cat /proc/PID/status` | Detailed per-process state |
| `cat /proc/PID/limits` | Actual resource limits in effect |
| `ls -la /proc/PID/fd/` | Open file descriptors, raw view |
| `cat /proc/PID/cmdline \| tr '\0' ' '` | Exact launch command line |

## Appendix A — Anki Cards

**`m07-01-performance-troubleshooting-anki.csv`:**

```
#deck:Linux Mastery::Module 7 - Performance::Demo M07-01 - Performance Troubleshooting
#separator:Comma
#columns:Front,Back,Tags
"What does top's CPU% actually measure, and what does it NOT measure?","Time the process was scheduled on a core. It does not measure how efficiently that time was used — that's perf stat's instructions-per-cycle (IPC) figure.","demo-m07-01,perf,top,fundamentals"
"What does perf stat's instructions-per-cycle (IPC) metric measure, and what does a low value (well under 1.0) suggest?","How many instructions execute per CPU cycle — a low IPC suggests the CPU is frequently stalled, most commonly waiting on memory access, rather than doing productive computational work.","demo-m07-01,perf,ipc,fundamentals"
"What's the difference in scope between strace and ltrace?","strace sees kernel-boundary activity (syscalls) — file I/O, network, process/signal operations. ltrace sees library-call activity in user space (malloc, strlen, application library functions) — work that never crosses the kernel boundary.","demo-m07-01,strace,ltrace,scope"
"What's the difference between what perf stat and perf top each tell you?","perf stat measures how efficient a run was overall (cycles, instructions, IPC). perf top shows, live, which specific function is consuming CPU time right now — a different question (efficiency vs. location).","demo-m07-01,perf,perf-top,fundamentals"
"What's the difference between what ltrace -c and perf top each reveal about a slow user-space process?","ltrace -c counts how many times each library function was called — useful when the suspicion is 'something is called too often.' perf top samples where CPU time is actually spent live — useful when there's no specific suspicion yet about which function is slow.","demo-m07-01,ltrace,perf-top,tool-boundary"
"Why is strace -p PID (attaching to a running process) important, rather than always starting a fresh trace?","A problem already in progress can't always be reproduced by restarting — attaching preserves the exact conditions that caused it, rather than losing them by starting over.","demo-m07-01,strace,attach"
"What is /proc/PID/ considered the 'ground truth' for, relative to tools like ps/top/lsof?","Every process-inspection tool ultimately reads its data from /proc/PID/ — when a formatted tool's output is ambiguous or seems wrong, reading /proc/PID/status, /limits, or /fd/ directly confirms what's actually true.","demo-m07-01,proc,fundamentals,cross-demo"
"In perf top's output, a library function like __memmove_avx_unaligned shows up as the top consumer of CPU time even though the application never calls it by name. Why?","perf top samples where CPU time is spent, including library-internal functions invoked on the application's behalf (e.g. a memory-copy routine called from inside the app's own code) — not just functions the developer wrote or named directly.","demo-m07-01,perf-top,fundamentals"
```

## Appendix B — Quiz

**`m07-01-performance-troubleshooting-quiz.md`:**

````markdown
# Quiz — Demo M07-01: Performance Troubleshooting

> One correct answer per question unless stated otherwise.
> Target: 80% or above before moving to Demo M07-02.

---

**Q1. A batch job's `strace -c` shows almost no syscall time, and its `ltrace -c` shows a completely unremarkable number of library calls — nothing redundant. What tool from this demo is the logical next step, and why?**

- A) `strace -p`, run again with a longer timeout
- B) `perf top`, since neither the kernel boundary nor call-count analysis pointed at a specific cause — a live, sampled view of where CPU time is actually going is the remaining option
- C) `perf stat`, exclusively, since IPC alone will identify the exact function
- D) There is no further tool available; the investigation is exhausted

<details>
<summary>Answer</summary>

**B** — `strace` ruled out the kernel boundary, and `ltrace -c`'s call
counts didn't point anywhere unusual. `perf top` answers a different
question than either — not "what's called" but "where is time actually
being spent, live" — which is exactly the remaining unknown.

Trap: C confuses `perf stat`'s efficiency summary with `perf top`'s
location-finding; `perf stat` alone never names a specific function.

</details>

---

**Q2. `perf stat` on a slow process reports an IPC of 0.9, close to the 1.0 baseline. What does this suggest, compared to an IPC of 0.25?**

- A) The process is severely memory-stalled, just like a 0.25 reading
- B) The CPU is executing instructions relatively efficiently for the cycles it's using — the bottleneck, if any, is more likely elsewhere (I/O, waiting, syscall volume) than memory-stall-driven CPU inefficiency
- C) An IPC of 0.9 always means the process is I/O-bound
- D) IPC values above 0.5 are never meaningful and should be ignored

<details>
<summary>Answer</summary>

**B** — IPC close to 1.0 indicates the CPU cycles being spent are largely
going toward real instruction execution, not memory stalls — the opposite
signal from the 0.25 case described in §1.3. If the process is still slow,
the cause is more likely outside pure CPU efficiency.

</details>

---

**Q3. You suspect a process is calling `malloc`/`free` far more often than it should, but you don't yet know if that's actually true. Which tool directly answers that specific question?**

- A) `perf top`, since it always ranks `malloc` at the top when it's a problem
- B) `ltrace -c`, since it counts calls to specific library functions like `malloc`/`free` directly
- C) `strace -c`, since `malloc` is a syscall
- D) `/proc/PID/status`

<details>
<summary>Answer</summary>

**B** — `ltrace -c`'s summary directly answers "how many times was this
library function called." Trap: C incorrectly treats `malloc` as a
syscall — it's a library function, `strace`'s territory doesn't cover it
directly (though `malloc` may itself trigger syscalls like `brk`/`mmap`,
which is a different, indirect signal).

</details>

---

**Q4. `strace -c` shows heavy time spent in `futex` and `nanosleep`. What category of problem does this suggest?**

- A) Heavy disk I/O
- B) Contention or waiting, not I/O or computation
- C) A memory leak
- D) Network latency specifically

<details>
<summary>Answer</summary>

**B** — `futex`/`nanosleep` point at synchronization/waiting behavior,
distinct from the I/O signature (`read`/`write`/`openat`) or
memory-allocation signature (`mmap`/`brk`).

</details>

---

**Q5. Why would you use `strace -p PID` instead of restarting a process under `strace` from scratch?**

- A) `strace -p` is always faster to execute
- B) A problem already in progress often can't be reproduced by restarting — attaching preserves the exact conditions that caused it
- C) `strace` cannot trace newly-started processes at all
- D) `-p` is required syntax for `strace` to work on Rocky9

<details>
<summary>Answer</summary>

**B** — This is the specific, practical reason to attach rather than
restart.

</details>

---

**Q6. What is `/proc/PID/` considered relative to tools like `ps`, `top`, and `lsof`?**

- A) A deprecated legacy interface these tools no longer use
- B) The ground truth these tools ultimately read their data from
- C) A write-only interface for configuring process behavior
- D) Only relevant for kernel developers, not operational troubleshooting

<details>
<summary>Answer</summary>

**B** — This is the same "go to the source" principle established for
`/proc`/`/sys` generally in Demo M04-07, applied here to per-process data.

</details>

---

**Q7. `perf top -p PID` shows a library function you never call by name consuming 47% of CPU time. Is this a sign of a bug in `perf top` itself?**

- A) Yes — `perf top` should only ever show functions the application explicitly calls
- B) No — `perf top` samples all CPU time, including library-internal functions invoked on the application's behalf (e.g. a memory-copy routine), whether or not the application names them directly
- C) Yes, unless the application is statically linked
- D) No, but only for kernel-space functions, never library functions

<details>
<summary>Answer</summary>

**B** — This is exactly the kind of insight `perf top` provides that call
counts alone don't: time genuinely spent inside library internals your
own code triggers indirectly.

</details>

---

Score guide:

| Score | Action |
|---|---|
| 7/7 | Import Anki cards, move to Demo M07-02 |
| 6/7 | Review the wrong answer, then proceed |
| 5/7 | Re-read the relevant section, retry those questions |
| Below 5/7 | Re-read the full demo and redo the walkthrough before proceeding |
````