# Demo 26 — Performance Troubleshooting

## Overview

Demo 09/10 covered monitoring — knowing *that* something is slow. This
demo covers the next level down: knowing *why*, at the level of individual
system calls, library calls, and CPU cycles. `strace` and `ltrace` show
exactly what a process is asking the kernel and its libraries to do;
`perf stat` gives a CPU-level performance summary; `/proc` gives direct
access to a specific process's live state. By the end, you'll have a
concrete first move for "this process is slow and I don't know why,"
rather than guessing.

## Demo Objectives

1. Trace a process's system calls with `strace` and read the output
2. Trace a process's library calls with `ltrace`, and explain when it's the better choice over `strace`
3. Use `strace -c` for a syscall summary, and `strace -T` for per-call timing
4. Read basic `perf stat` output and identify what it measures
5. Read process-specific data directly from `/proc/PID/`
6. Know what eBPF/bpftrace is for, at a conceptual level, as a forward reference

## Directory Structure

```
26-performance-troubleshooting/
└── README.md   # This file — full demo content, including Anki (Appendix A)
                 # and Quiz (Appendix B), extracted later via script.
```

## Environments

| Task | Ubuntu 24.04 | Rocky Linux 9.8 | EC2 Amazon Linux |
|---|---|---|---|
| `strace`/`ltrace` | `sudo apt install strace ltrace` | `sudo dnf install strace ltrace` | Not required |
| `perf` | `sudo apt install linux-tools-generic` | `sudo dnf install perf` | Not required |

## Recall Check — 25-shell-scripting-advanced

Answer these from memory before continuing — no looking back at Demo 25.

**Q1.** What specific failure does `set -o pipefail` catch that plain
`set -e` alone does not?

**Q2.** A script with `set -e` continues running after a command used as an
`if` condition fails. Is this a bug?

**Q3.** Why is `"${array[@]}"` (quoted, `@`) the correct way to iterate a
bash array?

<details>
<summary>Answers</summary>

**A1.** An early stage of a pipeline failing while a later stage still
succeeds — without `pipefail`, the pipeline's exit status reflects only its
last command, silently masking the earlier failure.

**A2.** No — commands used as `if`/`while` conditions are expected to
potentially fail as part of what's being tested; `set -e` deliberately
doesn't treat that as script-ending.

**A3.** It preserves each element as its own distinct word, even if an
element contains spaces — the array equivalent of quoting a regular
variable to prevent word splitting.

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
giving a much lower-level picture than `top`/`vmstat` (Demo 09/10). **The
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

### 1.4 /proc/PID — process-specific live data

Demo 09 and Demo 16 covered `/proc/loadavg` and `/proc/meminfo` —
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
# authoritative source lsof (Demo 12) actually reads from; useful when you
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
Demo 16 established for `/proc`/`/sys` generally.

### 1.5 eBPF/bpftrace — a conceptual forward reference

`strace` and `ltrace` both work by intercepting calls, which adds real
overhead — sometimes enough to change the very timing behavior being
investigated ("the observer effect"). **eBPF** is a kernel technology that
lets small, safe programs run directly inside the kernel, triggered by
events (a syscall, a function call, a scheduling decision) — observing
system behavior with dramatically lower overhead than `strace`, and
without the same collect-then-analyze-afterward limitation. **`bpftrace`**
is a high-level scripting front end for writing eBPF programs without
writing kernel code directly.

This demo doesn't build a working eBPF/bpftrace example — it's a large
enough topic (a full language and execution model of its own) to warrant
dedicated treatment elsewhere. The practical takeaway here: know that
`strace`/`ltrace`'s overhead has a real ceiling for production,
performance-sensitive tracing, and that eBPF/bpftrace exists specifically
to trace with far less observer effect when that ceiling matters.

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

### Task 5 — /proc/PID direct inspection

```bash
sleep 60 &
SLEEP_PID=$!
cat /proc/$SLEEP_PID/status | head -10
cat /proc/$SLEEP_PID/limits | head -5
ls -la /proc/$SLEEP_PID/fd/
cat /proc/$SLEEP_PID/cmdline | tr '\0' ' '; echo
kill "$SLEEP_PID" 2>/dev/null
```

### Task 6 — Choosing strace vs. ltrace for a symptom

```bash
# Given a process reported as "slow, seems to be doing a lot of file I/O":
# which tool from this demo would you reach for FIRST, strace or ltrace?
# (No command to run — this is a judgment call to make explicitly before
# checking against §1.1/§1.2's guidance.)
```

## What You Learned

1. ✅ Traced a process's system calls with `strace` and read the output
2. ✅ Traced a process's library calls with `ltrace`, and when it's the better choice
3. ✅ Used `strace -c` for a syscall summary, and `-T` for per-call timing
4. ✅ Read basic `perf stat` output, including the IPC efficiency signal
5. ✅ Read process-specific data directly from `/proc/PID/`
6. ✅ Understood what eBPF/bpftrace is for, conceptually, as a forward reference

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
```

Tasks:
1. `top` shows 98% CPU utilization for this process. Reference §1.3 — does
   high CPU% necessarily mean the CPU is doing proportionally useful work?
2. What does an IPC of 0.25 suggest is actually happening during those CPU
   cycles?
3. What would be a reasonable next diagnostic step, given this finding
   points at memory access patterns rather than pure computation?

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
2. Reference §1.2 — which tool from this demo would be a more appropriate
   next step, given the evidence so far points away from syscall/kernel
   activity?
3. What kind of application behavior would `ltrace -c` be well-suited to
   reveal that `strace -c` structurally couldn't have shown at all?

<details>
<summary>Answers</summary>

**Incident 1:** No — `top`'s CPU% measures time the process was actively
scheduled on a core, not how efficiently that scheduled time was used;
`perf stat`'s IPC (instructions per cycle) is the metric that actually
measures efficiency. An IPC of 0.25 is well below the 1.0 baseline
suggested in §1.3, indicating the CPU is spending a large share of its
cycles stalled — most commonly waiting on memory access (cache misses,
memory latency) rather than executing useful instructions — while still
appearing "busy" (98%) from a pure scheduling perspective. A reasonable
next step given this points at memory access patterns specifically:
investigate the application's data access patterns (poor cache locality,
excessive pointer-chasing through large data structures, or unexpectedly
large working-set size relative to available cache) rather than assuming
the fix is "add more CPU cores" or "optimize the algorithm's instruction
count," since the bottleneck here isn't instruction volume, it's memory
stalls per instruction.

**Incident 2:** The time is going somewhere `strace` structurally cannot
see — pure CPU-bound computation happening entirely in user space,
without making system calls at all. `strace` only ever sees the
kernel-boundary-crossing activity (syscalls); a process doing intensive
computation, string processing, or library-internal work between syscalls
is invisible to `strace`'s summary, which is exactly why its total traced
time (0.000083s) is nowhere close to the job's actual 45-second runtime.
`ltrace -c` is the appropriate next step, since it traces library-function
calls happening in that same user-space gap `strace` can't see — `ltrace`
could reveal, for example, an enormous number of redundant `malloc`/`free`
calls, inefficient string operations, or a library function being called
far more often than expected, any of which would explain 45 seconds of
work that never touches the kernel boundary at all.

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
`ltrace -c` is the appropriate next step, since it traces library-function
calls in exactly that gap, potentially revealing excessive `malloc`/`free`
activity, inefficient string processing, or unexpectedly frequent
library-function calls.

**Q3. When would you reach for `strace` versus `ltrace` as the first diagnostic tool for a slow process?**
`strace` when the suspected bottleneck involves the kernel boundary — file
I/O, network calls, process/signal interaction, memory allocation
syscalls. `ltrace` when the suspected bottleneck is inside the
application's own logic or a library it depends on — string processing,
in-application memory management patterns, redundant library calls — work
that never crosses into the kernel and so is invisible to `strace`.

**Q4. Why does eBPF/bpftrace exist when `strace`/`ltrace` already provide tracing capability?**
`strace`/`ltrace` intercept calls, which adds real overhead — sometimes
enough to change the timing behavior actually being investigated. eBPF
runs small, safe programs directly inside the kernel triggered by events,
observing behavior with dramatically lower overhead, which matters
specifically for production, performance-sensitive tracing where
`strace`/`ltrace`'s interception cost would itself distort the
measurement.

## Key Takeaways

1. **`strace` sees kernel-boundary activity (syscalls); `ltrace` sees
   library-call activity within user space.** A slow process spending its
   time in pure computation between syscalls is invisible to `strace` —
   that's exactly when `ltrace` becomes the right next tool.
2. **High CPU% in `top` doesn't mean the CPU is doing proportionally
   useful work.** `perf stat`'s instructions-per-cycle (IPC) is the actual
   efficiency signal — a low IPC despite high CPU% points at memory stalls,
   not computation volume.
3. **`strace -c`'s summary — which syscalls dominate time — points at a
   category of problem before you look at anything else.** Heavy
   `read`/`write`/`openat` suggests I/O; heavy `futex`/`nanosleep`
   suggests contention/waiting; heavy `mmap`/`brk` suggests memory
   allocation activity.
4. **`/proc/PID/` is the ground truth every process-inspection tool
   ultimately reads from.** When `ps`/`top`/`lsof`'s formatted output is
   ambiguous, reading `/proc/PID/status`, `/limits`, or `/fd/` directly
   confirms what's actually true.
5. **`strace -p PID` attaches to an already-running process** — essential
   when a problem is already happening and restarting the process under
   trace from scratch would lose the exact conditions that caused it.
6. **eBPF/bpftrace exists specifically because `strace`/`ltrace`'s
   interception overhead has a real ceiling for production tracing.**
   Lower-overhead, in-kernel observation is the tradeoff eBPF makes —
   recognizing when that ceiling matters is the practical takeaway, not
   building eBPF programs in this demo.

> **Demo scope:** Primary concept: diagnosing performance problems at the
> syscall and CPU-profiling level. Supporting concepts: strace, ltrace,
> perf stat basics, /proc/PID deep dive, eBPF/bpftrace conceptual forward
> reference.
> Estimated completion time: 30 minutes (reading + hands-on + verification).
> Checkpoints: 6 natural stopping points (after Task 1 through Task 6).

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

### Direct process inspection

| Command | Purpose |
|---|---|
| `cat /proc/PID/status` | Detailed per-process state |
| `cat /proc/PID/limits` | Actual resource limits in effect |
| `ls -la /proc/PID/fd/` | Open file descriptors, raw view |
| `cat /proc/PID/cmdline \| tr '\0' ' '` | Exact launch command line |

## Appendix A — Anki Cards

**Note:** extracted into a standalone `.csv` by the series extraction script.

```
#deck:Linux Mastery::Module 5 - Performance & Security Capstone::26-performance-troubleshooting
#separator:Comma
#columns:Front,Back,Tags
"A process shows 98% CPU in top. Does that alone confirm the CPU is doing proportionally useful work?","No — top's CPU% measures time scheduled on a core, not efficiency of that time. perf stat's instructions-per-cycle (IPC) is the actual efficiency signal; a low IPC despite high CPU% indicates memory stalls, not productive computation.","demo26,perf,top,common-mistake"
"A job runs far longer than expected, but strace -c shows almost no time in any syscall. Where is the time going?","Pure user-space computation between syscalls — work happening entirely within the application/libraries that strace structurally cannot see, since strace only observes kernel-boundary crossings.","demo26,strace,troubleshooting"
"Given strace -c shows almost no syscall time despite a job clearly being slow, what tool is the appropriate next step?","ltrace -c — it traces library-function calls in the same user-space gap strace can't see, potentially revealing excessive malloc/free calls or inefficient library usage.","demo26,ltrace,strace,tool-boundary"
"What does strace -c's syscall time breakdown suggest when heavy time is spent in read/write/openat versus futex/nanosleep versus mmap/brk?","read/write/openat: I/O-bound. futex/nanosleep: contention or waiting. mmap/brk: memory allocation activity. The summary's shape points at a problem category before deeper investigation.","demo26,strace,diagnosis"
"When would you reach for ltrace instead of strace as the first diagnostic tool?","When the suspected bottleneck is inside the application's own logic or a library it depends on (string processing, memory allocation patterns, redundant library calls) — work that never crosses the kernel boundary and so is invisible to strace.","demo26,ltrace,strace,comparison"
"Why is strace -p PID (attaching to a running process) important, rather than always starting a fresh trace?","A problem already in progress can't always be reproduced by restarting the process — attaching preserves the exact conditions that caused it, rather than losing them by starting over.","demo26,strace,attach"
"What is /proc/PID/ considered the 'ground truth' for, relative to tools like ps/top/lsof?","Every process-inspection tool ultimately reads its data from /proc/PID/ — when a formatted tool's output is ambiguous or seems wrong, reading /proc/PID/status, /limits, or /fd/ directly confirms what's actually true.","demo26,proc,fundamentals,cross-demo"
"Why does eBPF/bpftrace exist when strace/ltrace already provide tracing?","strace/ltrace intercept calls, adding real overhead that can distort the timing behavior being investigated (the 'observer effect'). eBPF runs safe programs inside the kernel with dramatically lower overhead, mattering for production performance-sensitive tracing.","demo26,ebpf,bpftrace,fundamentals"
"What does perf stat's instructions-per-cycle (IPC) metric actually measure, and what does a low value (well under 1.0) suggest?","How many instructions execute per CPU cycle — a low IPC suggests the CPU is frequently stalled, most commonly waiting on memory access, rather than doing productive computational work.","demo26,perf,ipc,fundamentals"
"What's the difference in scope between strace and perf stat as diagnostic tools?","strace traces individual system calls (what the process asks the kernel to do); perf stat measures CPU-level hardware counters (cycles, instructions, cache behavior) — a lower-level, more efficiency-focused picture than either strace or top/vmstat provide.","demo26,strace,perf,comparison"
```

## Appendix B — Quiz

**Note:** extracted into a standalone `.md` by the series extraction script.

> One correct answer per question unless stated otherwise.
> Target: 80% or above before moving to Demo 27.

**Q1. A process shows 98% CPU in `top`. Does this confirm the CPU is doing proportionally useful work?**

- A) Yes, CPU% and efficiency are the same thing
- B) No — CPU% measures scheduled time, not efficiency; `perf stat`'s IPC is the actual efficiency signal
- C) Yes, but only if the process has been running for over a minute
- D) No, because `top`'s CPU% is always inaccurate

<details>
<summary>Answer</summary>

**B** — This is the exact distinction between "busy" (scheduling) and "efficient" (IPC) that `perf stat` reveals and `top` cannot. Trap: A conflates two different measurements; C invents an irrelevant time threshold; D overstates `top`'s inaccuracy rather than identifying what it actually measures.

</details>

---

**Q2. `strace -c` on a slow job shows almost no time spent in any syscall. What does this indicate?**

- A) `strace` is malfunctioning and needs to be reinstalled
- B) The time is being spent in pure user-space computation, which `strace` cannot observe since it only sees kernel-boundary crossings
- C) The job is actually running much faster than reported
- D) `strace -c` only works correctly on multi-threaded processes

<details>
<summary>Answer</summary>

**B** — `strace` structurally cannot see anything that doesn't cross into the kernel via a syscall. Trap: A and D both invent tool malfunctions; C contradicts the given evidence of a genuinely slow job.

</details>

---

**Q3. Given `strace -c` shows minimal syscall time despite a job being slow, what's the appropriate next diagnostic tool?**

- A) `perf stat`, exclusively
- B) `ltrace -c`, to trace library-function calls in the same user-space gap `strace` can't see
- C) Restarting the job with more CPU cores allocated
- D) `diff` between the job's expected and actual output

<details>
<summary>Answer</summary>

**B** — `ltrace` specifically observes the library-call layer that explains time invisible to `strace`. Trap: A isn't wrong to also consider, but doesn't specifically address the library-call layer this scenario points at; C and D are unrelated diagnostic non-sequiturs.

</details>

---

**Q4. `strace -c` shows heavy time spent in `futex` and `nanosleep`. What category of problem does this suggest?**

- A) Heavy disk I/O
- B) Contention or waiting, not I/O or computation
- C) A memory leak
- D) Network latency specifically

<details>
<summary>Answer</summary>

**B** — `futex`/`nanosleep` point at synchronization/waiting behavior, distinct from the I/O signature (`read`/`write`/`openat`) or memory-allocation signature (`mmap`/`brk`). Trap: A, C, and D all misassign this specific syscall pattern to a different category.

</details>

---

**Q5. When is `ltrace` the more appropriate first tool over `strace`?**

- A) When the suspected bottleneck is inside application logic or a library it depends on, not kernel-boundary activity
- B) Always — `ltrace` is strictly more informative than `strace`
- C) Only for statically-linked binaries
- D) Only when running as root

<details>
<summary>Answer</summary>

**A** — This is the actual, specific distinguishing use case between the two tools. Trap: B overstates `ltrace`'s universal superiority; C and D both invent false requirements.

</details>

---

**Q6. Why would you use `strace -p PID` instead of restarting a process under `strace` from scratch?**

- A) `strace -p` is always faster to execute
- B) A problem already in progress often can't be reproduced by restarting — attaching preserves the exact conditions that caused it
- C) `strace` cannot trace newly-started processes at all
- D) `-p` is required syntax for `strace` to work on Rocky9

<details>
<summary>Answer</summary>

**B** — This is the specific, practical reason to attach rather than restart. Trap: A invents a performance claim; C is factually wrong — `strace` traces fresh commands fine; D invents a platform-specific requirement.

</details>

---

**Q7. What is `/proc/PID/` considered relative to tools like `ps`, `top`, and `lsof`?**

- A) A deprecated legacy interface these tools no longer use
- B) The ground truth these tools ultimately read their data from
- C) A write-only interface for configuring process behavior
- D) Only relevant for kernel developers, not operational troubleshooting

<details>
<summary>Answer</summary>

**B** — This is the same "go to the source" principle established for `/proc`/`/sys` generally in Demo 16, applied here to per-process data. Trap: A is factually wrong; C misdescribes `/proc` as write-only; D understates its practical, everyday troubleshooting relevance.

</details>

---

**Q8. Why does eBPF/bpftrace exist as tracing tools alongside `strace`/`ltrace`?**

- A) They replace `strace`/`ltrace` entirely and should always be used instead
- B) `strace`/`ltrace`'s interception overhead can itself distort the timing behavior being investigated; eBPF observes with dramatically lower overhead
- C) eBPF is required for tracing any process running as a systemd service
- D) `strace`/`ltrace` don't work on Rocky9, only Ubuntu

<details>
<summary>Answer</summary>

**B** — This overhead/observer-effect tradeoff is the actual reason eBPF exists as a complementary, lower-overhead tracing approach. Trap: A overstates eBPF as a full replacement; C and D both invent false platform/service requirements.

</details>