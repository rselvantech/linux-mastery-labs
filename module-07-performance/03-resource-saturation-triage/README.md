# Demo M07-03 — Resource Saturation Triage

## Overview

Demo M07-01 assumed you had already narrowed a slowdown to one process and
gave you tools to trace it. Demo M07-02 gave you limits to stop one service
from starving the rest. This demo covers the step that comes *before*
both: when a host is "slow" and you don't yet know which resource is the
problem. CPU, memory, disk, and — on virtual machines — the hypervisor
itself can each produce the same complaint ("everything is slow"), and
each needs a different fix. Reaching for `strace` or `perf` before you
know which resource is saturated is how you spend an hour tracing a
process that was never the bottleneck.

**Real-world scenario:** a service on a cloud VM gets slow every
afternoon. The application team profiles their code and finds nothing.
The host's CPU looks busy, so someone adds worker threads, and the
slowdown gets worse. A single `vmstat 1 5` would have shown the answer in
five seconds: the VM was being starved by its hypervisor, and no change
inside the guest could fix that.

## Demo Objectives

By the end of this demo you will be able to:

1. Apply a fixed order of checks to classify a slow host as CPU-bound,
   memory-bound, I/O-bound, or affected by CPU steal
2. Read the columns of `vmstat` and state what each one indicates
3. Explain what CPU steal time is, and why high `st` cannot be fixed from
   inside the guest
4. Distinguish a fully utilized CPU from a saturated one using the run
   queue (`r`) against the CPU count
5. Use per-CPU views (`mpstat -P ALL`, `top` with `1`) to catch a
   single-thread bottleneck that host-wide averages hide
6. Recognize what `iowait` does and does not tell you, and use
   `iostat -x` to check whether a local disk is actually busy
7. Decide when to hand off to Demo M07-01's tracing tools once the saturated
   resource is known

## Directory Structure

```
03-resource-saturation-triage/
├── README.md                                   # this file
├── m07-03-resource-saturation-triage-anki.csv  # Anki flashcard deck
└── m07-03-resource-saturation-triage-quiz.md   # standalone quiz
```

## Environments

| Task | Ubuntu 24.04 | Rocky Linux 9.8 | EC2 Amazon Linux |
|---|---|---|---|
| `vmstat`, `top`, `ps`, `/proc/stat` | preinstalled (procps) | preinstalled (procps-ng) | Not required |
| `mpstat`, `iostat` | `sudo apt install sysstat` | `sudo dnf install sysstat` | Not required |
| Load generation | `sudo apt install stress-ng` | `sudo dnf install stress-ng` (may need EPEL — confirm on your image) | Optional, for Task 4 |

> **Verification note:** every `vmstat`, `top`, `ps`, and `/proc/stat`
> output in this demo was captured live in the authoring sandbox (Ubuntu
> 24.04.4, procps-ng 4.0.4, **a single vCPU** — so run-queue and per-CPU
> numbers on your multi-core machine will look different, and that
> difference is the point of Task 5). `mpstat`, `iostat`, and `stress-ng`
> could not be installed there (no network access), so their output
> shapes come from the sysstat and stress-ng man pages and are marked
> `[Unverified — test before use]`. Steal time reads 0 on the authoring
> host (a dedicated guest), so every non-zero steal example below is
> marked `⚠️ Simulated scenario`.

## Recall Check — 02-systemd-resource-control

Answer these from memory before continuing — no looking back at Demo M07-02.

1. A service is killed for exceeding `MemoryMax=` while `free -h` shows
   gigabytes free on the host. Is that a contradiction?
2. On a host that has swap enabled, why might a service that exceeds
   `MemoryMax=` slow down instead of being killed, and which directive
   changes that?
3. Where do you look for evidence of an OOM kill after the unit has
   already died?

<details>
<summary>Answers</summary>

1. No — `MemoryMax=` is enforced per service, independent of host-level
   free memory; the two are separate accounting scopes.
2. `MemoryMax=` bounds memory, not swap; with swap available the kernel
   can push the service's excess memory to swap instead of killing it.
   `MemorySwapMax=0` removes that escape route.
3. The journal — systemd's own message for the unit plus the kernel's OOM
   report (`journalctl -k`) — not `MemoryCurrent`, which belongs to a
   cgroup that no longer exists.

</details>

---

## Part 1 — Concepts

### 1.1 Triage before tracing

Tracing tools (Demo M07-01) answer "what is this process doing?" Triage
answers a different question: "which resource is this whole machine short
of?" The order matters, because the answer decides which tool you reach
for next:

```
1. Is the CPU run queue longer than the number of CPUs?   → CPU saturation
2. Is the machine swapping?                                → memory pressure
3. Are processes blocked waiting on I/O?                   → disk / storage
4. Is the CPU busy but a chunk of time is "stolen"?        → hypervisor contention
```

This is a small application of a well-known checklist method, the **USE
method** (for every resource, check Utilization, Saturation, and Errors).
The distinction that matters most here is **utilization vs saturation**:
a resource can be 100% utilized yet not saturated (nothing is waiting for
it), and it only becomes a bottleneck when work starts *queueing* behind
it. `vmstat` is the fastest single command that shows queueing for CPU,
memory, and I/O at once.

### 1.2 vmstat — one line per second

```bash
vmstat 1 5
```

Real output, captured on an idle 1-vCPU host:

```
procs -----------memory---------- ---swap-- -----io---- -system-- -------cpu-------
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st gu
 1  0      0 3982380   3424  51744    0    0  6942     8  466   11  1 33 65  1  0  0
 0  0      0 3982352   3424  51792    0    0     0     0   45   67  0  0 100  0  0  0
 0  0      0 3982324   3424  51792    0    0     0     0   46   74  0  0 100  0  0  0
```

**The first row is not a live reading.** It reports averages since boot,
which is why it shows `bi 6942` and `sy 33` on a machine that was doing
nothing when the following rows were captured. Always discard row one and
read the rows after it.

| Column | Meaning (per `vmstat(8)`) |
|---|---|
| `r` | Processes runnable — running or waiting for a CPU |
| `b` | Processes in uninterruptible sleep, usually waiting on I/O |
| `swpd` `free` `buff` `cache` | Memory figures, in KiB |
| `si` `so` | Memory swapped in from / out to disk, per second |
| `bi` `bo` | Blocks received from / sent to block devices, per second |
| `in` `cs` | Interrupts and context switches per second |
| `us` `sy` `id` `wa` `st` `gu` | CPU time %: user, kernel, idle, waiting for I/O, stolen, guest |

### 1.3 Reading the CPU columns

**`us` high, `sy` low — the CPU is doing application work.** Real output
from a pure user-space loop on the 1-vCPU host (row 1 is the since-boot
row):

```
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st gu
 3  0      0 3947532  12028  74176    0    0   273  5150   71    0  5  6 89  1  0  0
 1  0      0 3947532  12028  74176    0    0     0     0  282   90 100  0  0  0  0  0
 1  0      0 3947532  12028  74176    0    0     0     0  266   53 100  0  0  0  0  0
```

`r` is 1 and the host has 1 CPU: the CPU is 100% **utilized** but not
**saturated** — nothing is waiting for it. Now two `yes > /dev/null`
processes on the same single CPU:

```
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st gu
 2  0      0 3952512  11868  71596    0    0     0     0  269  297 37 63  0  0  0  0
 2  0      0 3952512  11868  71596    0    0     0     0  272  297 47 53  0  0  0  0
 2  0      0 3952512  11868  71596    0    0     0     0  265  288 41 59  0  0  0  0
```

`r` is 2 on 1 CPU: one process is always waiting — the CPU is now
**saturated**. Notice also that `sy` (52–63%) is larger than `us`: `yes`
spends most of its time in the kernel because it makes a `write` system
call for every line. **A high `sy` means the time is going into system
calls — that is the signal to reach for `strace -c` from Demo M07-01.**

**`wa` — waiting for I/O.** Real output while writing 4 GB with
`dd ... oflag=direct`:

```
 r  b   swpd   free   buff  cache   si   so    bi     bo   in   cs us sy id wa st gu
 0  1      0 3947532  12028  74172    0    0     0 851968 1055 1798  0  9  0 91  0  0
 0  1      0 3947532  12028  74172    0    0     0 345088  460  746  1  6  0 93  0  0
 0  1      0 3947532  12028  74176    0    0     0 397312  505  842  0  6  0 94  0  0
```

`r` is 0, `b` is 1, `bo` is in the hundreds of thousands, and `wa` is
about 93%. The CPU is not busy — it has nothing to run because the only
process is blocked on disk. Be careful how far you trust `wa`: the kernel
documentation (`proc(5)`) notes that iowait is not a reliable measure,
because a waiting task is not running on any CPU and the value can even
decrease. Treat high `wa` together with `b > 0` as a hint to check the
disk, not as proof.

### 1.4 CPU steal time

On a virtual machine, a virtual CPU can be ready to run while the
hypervisor is running something else on the physical CPU. That time is
reported as **steal** (`st`). It appears in three places:

```bash
vmstat 1 3                       # st column
top -bn1 | sed -n 3p             # %st in the CPU summary line
grep '^cpu ' /proc/stat          # 8th value on the line
```

Real output on the authoring host, where steal is 0:

```
%Cpu(s):  0.0 us,  0.0 sy,  0.0 ni,100.0 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
cpu  4 0 90 375 3 0 0 0 0 0
```

`/proc/stat` lists the CPU time counters in this order, per `proc(5)`:
user, nice, system, idle, iowait, irq, softirq, **steal**, guest,
guest_nice. On dedicated hardware, and on most dedicated guests, steal
stays at 0. It becomes visible when the hypervisor is oversubscribed
("noisy neighbor"), and on some burstable cloud instance types when CPU
credits run out — how that is reported is provider-specific
`[Unverified — test before use]`.

`⚠️ Simulated scenario` — a 4-vCPU guest under contention (illustrative
numbers, not captured from a real host):

```
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st gu
 7  0      0 812340  90112 1204480    0    0     0    12 3120 4100 46  8  0  0 46  0
```

The guest looks saturated — `r` is 7 on 4 CPUs and idle is 0 — but 46% of
the time the vCPUs were waiting on the hypervisor. **Adding threads or
optimizing the application cannot recover time the hypervisor is not
giving you.** The fixes live outside the guest: a larger or non-burstable
instance type, a different host, or a word with the provider.

### 1.5 Per-CPU views — what averages hide

`vmstat`'s CPU columns are averages across all CPUs. A single-threaded
program pinned at 100% of one core on an 8-CPU host shows up as about
12.5% in the aggregate — easy to dismiss as "the host is mostly idle"
while the one job that matters is maxed out.

```bash
mpstat -P ALL 1 3       # sysstat: one row per CPU
```

`[Unverified — test before use]` `mpstat` prints one row per CPU with
`%usr %nice %sys %iowait %irq %soft %steal %guest %gnice %idle` columns.
Without sysstat, `top` gives the same per-CPU breakdown: press `1` while
it is running, or start it with `top -1`. Real output on the authoring
host (one CPU, so only `Cpu0` appears):

```
%Cpu0  : 42.7 us, 57.3 sy,  0.0 ni,  0.0 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
```

### 1.6 iostat — is the local disk actually busy?

```bash
iostat -x 1 3            # -x: extended per-device statistics
```

`[Unverified — test before use]` The columns to read are `r/s` and `w/s`
(requests per second), `r_await` and `w_await` (average milliseconds a
request takes, including queueing), `aqu-sz` (average queue length) and
`%util`. Rising `await` and a growing queue are the saturation signal.
`%util` needs care: per the `iostat(1)` man page, on devices that serve
requests in parallel — RAID arrays and modern SSDs — it does not reflect
the device's real performance limit, so 100% does not automatically mean
the device is saturated.

`-z` hides devices with no activity, which keeps the output readable on
hosts with many devices.

### 1.7 From symptom to next tool

| What you see | Resource | Next step |
|---|---|---|
| `r` well above the CPU count, `us` high | CPU saturation | `perf top`, `ltrace -c` (Demo M07-01) |
| `sy` larger than `us` | Kernel / system calls | `strace -c` (Demo M07-01) |
| `so` sustained above 0, `free` small | Memory pressure | find the largest consumers; check service limits (Demo M07-02) |
| `b` above 0, `wa` high, `iostat -x` shows high `await` | Local disk | identify the process doing the I/O |
| `b` above 0, `wa` high, local disk quiet | Something else blocking on I/O (for example a network filesystem) | list D-state processes; see Demo M04-07 |
| `st` sustained and non-trivial, `us` moderate | Hypervisor contention | not fixable in the guest — escalate |
| One CPU at 100%, aggregate low | Single-threaded bottleneck | `perf top -p PID` (Demo M07-01) |

Any numeric threshold for "sustained and non-trivial" is a rule of thumb
that depends on the workload — not a standard.

Triage names the resource, not the cause. Once you know it is CPU, or
disk, or steal, the tools from Demo M07-01 answer *why*.

---

## Part 2 — Hands-On Lab

### Commands used in this demo

| Command | Flag meaning | Purpose |
|---|---|---|
| `vmstat 1 5` | interval 1 s, 5 samples | Run queue, memory, swap, I/O, CPU split |
| `nproc` | — | Number of CPUs, to compare against `r` |
| `top -bn1 \| sed -n 3p` | `-b`: batch, `-n1`: one iteration | CPU summary line including `%st` |
| `top -b -n2 -d1 -1` | `-1`: per-CPU lines | Per-CPU breakdown without sysstat |
| `grep '^cpu ' /proc/stat` | — | Raw CPU time counters, steal is the 8th value |
| `mpstat -P ALL 1 3` | `-P ALL`: every CPU | Per-CPU utilization (sysstat) |
| `iostat -x 1 3` | `-x`: extended stats | Per-device `await`, queue, `%util` (sysstat) |
| `ps -eo pid,stat,wchan:20,comm` | `-o`: choose columns | Find processes in D (blocked) state |

### Task 1 — Baseline

```bash
nproc
vmstat 1 5
```

Record the CPU count. In the vmstat output, confirm the first row differs
from the rest, and note which columns are near zero on an idle machine.

### Task 2 — CPU-bound versus syscall-heavy

```bash
awk 'BEGIN{for(;;) x=x+1}' &
vmstat 1 4
kill %1
```

Expect `us` near 100, `sy` near 0, and `r` equal to about the number of
loops you started. Then repeat with a syscall-heavy loop:

```bash
yes > /dev/null &
vmstat 1 4
kill %1
```

Confirm `sy` now rivals or exceeds `us`. This is the same distinction
Demo M07-01 uses `strace -c` to explore.

### Task 3 — I/O-bound

Needs about 4 GB of free space in `/tmp`.

```bash
dd if=/dev/zero of=/tmp/ddtest bs=1M count=4000 oflag=direct &
vmstat 1 6
ps -eo pid,stat,wchan:20,comm | awk '$2 ~ /^D/'
wait
rm -f /tmp/ddtest
```

Confirm `b` rises to at least 1, `bo` becomes large, `wa` climbs, and `us`
stays near 0. The `ps` line may catch `dd` in the D (uninterruptible
sleep) state while it waits on the disk.

### Task 4 — Read steal time three ways

```bash
vmstat 1 3
top -bn1 | sed -n 3p
grep '^cpu ' /proc/stat
```

On a dedicated host expect `st` and `%st` to be 0 and the 8th value on the
`/proc/stat` line to be 0. Optional, cloud burstable instance only: run
`stress-ng --cpu $(nproc) --timeout 300s` and watch `vmstat 1` for `st`
rising as credits are exhausted `[Unverified — test before use]`.

### Task 5 — Per-CPU versus aggregate

Needs a host with more than one CPU.

```bash
awk 'BEGIN{for(;;) x=x+1}' &
vmstat 1 3
top -b -n2 -d1 -1 | grep '^%Cpu'
kill %1
```

On an N-CPU host, `vmstat` shows `us` of about 100/N while exactly one
`%CpuN` line in `top` shows about 100. With sysstat installed, compare
`mpstat -P ALL 1 3`.

### Task 6 — Classify three samples

Use the decision table in §1.7 to name the resource and the next tool for
each sample (all on a 4-CPU host):

```
A:  r=9  b=0   si=0 so=0   us=85 sy=10 id=5  wa=0  st=0
B:  r=0  b=8   si=0 so=0   us=2  sy=4  id=0  wa=94 st=0
C:  r=5  b=0   si=0 so=0   us=45 sy=7  id=0  wa=0  st=48   (⚠️ Simulated scenario)
```

<details>
<summary>Answers</summary>

**A** — CPU saturation: the run queue (9) is more than twice the CPU
count and the CPUs are busy with application work. Next: `perf top` /
`ltrace -c` to find where the time goes.

**B** — Disk / I/O: nothing is runnable, eight processes are blocked, and
the CPUs are waiting rather than working. Next: `iostat -x` to confirm the
device is busy, and find the process doing the I/O.

**C** — Steal: the guest is saturated on paper but 48% of the time is
taken by the hypervisor. Next: escalate — a bigger or non-burstable
instance, or a different host. Nothing inside the guest recovers it.

</details>

### Cleanup

```bash
kill %1 2>/dev/null; rm -f /tmp/ddtest
```

---

## What You Learned

- ✅ Applied a fixed order of checks to classify a slow host by the
  resource that is saturated
- ✅ Read the columns of `vmstat` and discarded the since-boot first row
- ✅ Explained CPU steal time and why it cannot be fixed from inside the
  guest
- ✅ Distinguished a fully utilized CPU from a saturated one using `r`
  against the CPU count
- ✅ Used per-CPU views to catch a single-thread bottleneck that averages
  hide
- ✅ Recognized the limits of `iowait` and used `iostat -x` to check the
  disk directly
- ✅ Chose the next tool from Demo M07-01 based on which resource was
  saturated

---

## Part 3 — Break-Fix Scenario

### Scenario — NovaTech Ops: two "the host is slow" incidents

> **Incident report — NovaTech Ops, ticket #5012**
>
> Alert 1: "api-node-07 gets slow every afternoon. The dev team added
> worker threads and it got worse." Alert 2: "batch-host-02 is slow, the
> load average is high, but the disk dashboard shows the local disk almost
> idle."

`⚠️ Simulated scenario` — both samples below are illustrative, not
captured from a real host.

**Incident 1 — slow every afternoon on a cloud VM (4 vCPUs)**

```
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st gu
 7  0      0 812340  90112 1204480    0    0     0    12 3120 4100 46  8  0  0 46  0
```

Tasks:
1. On the surface, what do `r=7` and `id=0` suggest, and what does `st=46`
   change about that reading?
2. Which of these will *not* recover the lost time: adding worker threads,
   optimizing the request handler, resizing the VM to a larger,
   non-burstable type?
3. How would you confirm the pattern is contention rather than the
   application, using only tools from this demo?

**Incident 2 — high `wa`, quiet local disk**

```
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st gu
 0  9      0 512000  40000  900000    0    0    10     4  300  500  2  3 15 80  0  0
```

`iostat -x` (`[Unverified — test before use]`) shows the local disk at
about 2% `%util` with negligible `await`.

Tasks:
1. Reference §1.3 — what does `wa=80` with `b=9` tell you, and what does
   it *not* prove?
2. If the local disk is quiet, what else can leave processes in
   uninterruptible sleep?
3. Which commands find the blocked processes and what they are waiting on?

<details>
<summary>Answers</summary>

**Incident 1:** On the surface the guest looks CPU-saturated (run queue
longer than the CPU count, no idle time). `st=46` means 46% of the time the
vCPUs were ready but the hypervisor ran something else, so barely half of
the "busy" time is the guest doing its own work. Adding threads makes the
run queue longer without adding capacity, and optimizing the handler cannot
recover time that was never given to the guest — both fail to help.
Resizing to a larger, non-burstable instance (or moving to another host)
addresses the actual cause. To confirm the pattern is contention: watch
`vmstat 1` or the `%st` figure in `top` across the slow window and compare
against a quiet window; steal that tracks the slowdown while the
application's own profile shows nothing unusual points at the hypervisor.

**Incident 2:** `wa=80` with `b=9` says most CPU time is idle-while-waiting
and nine processes are blocked in uninterruptible sleep — the machine is
waiting on I/O. It does not prove the *local disk* is the cause, and
`proc(5)` warns that iowait is not a reliable figure. A quiet local disk
points elsewhere: the usual suspect is a network filesystem (NFS, CIFS)
whose server is slow or unreachable, which is Demo M04-07's territory. Find the
blocked processes and what they wait on with
`ps -eo pid,stat,wchan:20,comm | awk '$2 ~ /^D/'` (the `wchan` column names
the kernel function the process is sleeping in), and check for network
mounts with `mount | grep -E 'nfs|cifs'`.

</details>

---

## Interview Prep

**Q1. A VM is slow, `top` shows the CPUs 100% busy, but the application team's profiling finds nothing. What do you check?**

Steal time. `vmstat`'s `st` column, or `%st` in `top`'s CPU line, shows
time the vCPUs were ready but the hypervisor ran something else. If it is
sustained and non-trivial, the guest is being starved and the fix is
outside the guest — a different instance type or host — not more threads
or code changes.

---

**Q2. What's the difference between a CPU being fully utilized and being saturated?**

Utilized means it is busy; saturated means work is queueing for it. A
single-threaded loop on a single-CPU machine shows `us` at 100 with `r` at
1 — fully utilized, nothing waiting. Add a second runnable process and `r`
becomes 2 on 1 CPU: something is always waiting, which is saturation. The
run queue compared against the CPU count is the saturation signal.

---

**Q3. Why do you discard the first line of `vmstat` output?**

It reports averages since boot rather than a live one-second reading, so it
can show large `bi`/`bo` or `sy` values on a machine that is idle right
now. The rows after it are the live samples.

---

**Q4. Aggregate CPU on an 8-core host is about 13% busy, but one batch job is far slower than expected. What is the likely explanation, and how do you confirm it?**

A single-threaded job pinned to one core: 100% of one core is about 12.5%
of eight, so the aggregate looks nearly idle. Confirm with a per-CPU view —
`mpstat -P ALL` or `top` with `1` — where one CPU sits at about 100% while
the others are idle. The fix is a design change (parallelize the job), not
more cores.

---

## Key Takeaways

1. **Triage before you trace.** Decide which resource is saturated —
   CPU, memory, disk, or stolen CPU — before choosing a tracing tool;
   `vmstat 1 5` shows queueing for all of them in one command.
2. **Discard `vmstat`'s first row.** It is an average since boot, not a
   live reading, and can show activity that is not happening now.
3. **Utilized is not saturated.** A CPU at 100% with `r` no larger than
   the CPU count has nothing waiting; `r` above the CPU count is the
   saturation signal.
4. **High `sy` means system calls; high `wa` with `b > 0` means blocked
   on I/O.** Neither is proof on its own: `sy` points you at `strace -c`,
   and `wa` needs the disk itself checked with `iostat -x` because iowait
   is an unreliable figure.
5. **Steal time cannot be fixed inside the guest.** A saturated-looking
   guest with sustained `st` is being starved by the hypervisor — the fix
   is a different instance or host, and more threads make it worse.
6. **Averages hide single-thread bottlenecks.** One pinned core on an
   N-CPU host shows as about 100/N percent in the aggregate; use a
   per-CPU view before concluding the host is idle.
7. **Triage names the resource, not the cause.** Once you know which
   resource is saturated, hand off to Demo M07-01's tools (or Demo M07-02's
   limits) to find out why.

> **Demo scope:** Primary concept: top-down triage of which resource is
> saturated on a slow host. Supporting concepts: `vmstat` column reading,
> utilization vs saturation, CPU steal time, per-CPU views, `iostat -x`.
> Estimated completion time: 35 minutes (reading + hands-on + verification).
> Checkpoints: 6 natural stopping points (after Task 1 through Task 6).

---

## Quick Commands Reference

### First look

| Command | Purpose |
|---|---|
| `vmstat 1 5` | Run queue, memory, swap, I/O, and CPU split; ignore row one |
| `nproc` | CPU count, to compare against `r` |
| `top -bn1 \| sed -n 3p` | CPU summary line including `%st` |

### Drilling down

| Command | Purpose |
|---|---|
| `mpstat -P ALL 1 3` | Per-CPU utilization (sysstat) |
| `top -b -n2 -d1 -1` | Per-CPU lines without sysstat |
| `iostat -x 1 3` | Per-device `await`, queue length, `%util` (sysstat) |
| `grep '^cpu ' /proc/stat` | Raw CPU counters; steal is the 8th value |
| `ps -eo pid,stat,wchan:20,comm \| awk '$2 ~ /^D/'` | Processes blocked in D state, and where |

---

## Appendix A — Anki Cards

**`m07-03-resource-saturation-triage-anki.csv`:**

```
#deck:Linux Mastery::Module 7 - Performance::Demo M07-03 - Resource Saturation Triage
#separator:Comma
#columns:Front,Back,Tags
"What does the FIRST row of vmstat output represent?","Averages since boot, not a live reading — ignore it and read the rows after it.","demo26c,vmstat"
"vmstat column r — what does it count?","Processes that are runnable: running or waiting for a CPU. Compare against the CPU count (nproc).","demo26c,vmstat"
"vmstat column b — what does it count?","Processes in uninterruptible sleep, usually blocked waiting on I/O.","demo26c,vmstat"
"vmstat columns si and so — what do they measure?","Memory swapped in from disk (si) and out to disk (so), per second. Sustained so above 0 indicates memory pressure.","demo26c,vmstat,memory"
"vmstat column wa — what does it measure?","Percentage of CPU time spent waiting on I/O. proc(5) warns iowait is not a reliable figure, so confirm with iostat -x.","demo26c,vmstat,io"
"vmstat column st — what does it measure?","Percentage of time a virtual CPU was ready to run but the hypervisor ran something else (stolen time).","demo26c,steal"
"Name two places besides vmstat where steal time is visible.","The %st field in top's CPU summary line, and the 8th value on the cpu line of /proc/stat.","demo26c,steal,proc"
"Which command shows utilization for each CPU separately, and which package provides it?","mpstat -P ALL 1 (sysstat). Without sysstat, top shows per-CPU lines when you press 1.","demo26c,mpstat,needs-verification"
"What does the -x flag add to iostat?","Extended per-device statistics, including await, average queue length (aqu-sz), and %util.","demo26c,iostat,needs-verification"
"What do the letters U, S, E stand for in the USE method?","Utilization, Saturation, Errors — checked for every resource.","demo26c,use-method"
```

---

## Appendix B — Quiz

**`m07-03-resource-saturation-triage-quiz.md`:**

````markdown
# Quiz — Demo M07-03: Resource Saturation Triage

> One correct answer per question unless stated otherwise.
> Target: 80% or above before moving to Demo M07-04.

---

**Q1.** A 4-CPU host is reported slow. `vmstat` (after the first row)
shows `r=9 b=0 us=85 sy=10 id=5 wa=0 st=0`. What is the most accurate
reading?

- A) The disk is saturated — `wa` needs investigating
- B) CPU saturation: the run queue is more than twice the CPU count and the CPUs are busy with work; profile where the CPU time goes next
- C) The hypervisor is starving the guest
- D) Memory pressure — the system is swapping

<details>
<summary>Answer</summary>

**B** — `r` far above the CPU count with almost no idle time is CPU
saturation. `wa=0` rules out I/O, `st=0` rules out steal, and swap columns
are not involved. Next tools: `perf top` and `ltrace -c` from Demo M07-01.

</details>

---

**Q2.** On a 4-vCPU cloud VM, `vmstat` shows `r=5 us=45 sy=7 id=0 wa=0
st=48`. The team proposes doubling the worker threads. Will that help?

- A) Yes — more threads use the idle CPU
- B) No — there is no idle CPU; 48% of the time is stolen by the hypervisor, and more threads lengthen the run queue without adding capacity
- C) Yes, but only if the threads are I/O-bound
- D) No, because `us` is too low for threading to matter

<details>
<summary>Answer</summary>

**B** — Steal time is capacity the guest never receives. The fix is
outside the guest: a larger or non-burstable instance, or another host.

</details>

---

**Q3.** `vmstat` shows `r=0 b=8 us=2 sy=4 id=0 wa=94`. A colleague says
"the CPU is at 100%, so it needs more cores." What is wrong with that?

- A) Nothing — `id=0` means the CPU is fully busy
- B) The CPU is not doing work: nothing is runnable and eight processes are blocked on I/O, so the CPU is waiting rather than busy; more cores would not help
- C) `wa` is always 0 on healthy systems, so the reading is a monitoring bug
- D) The processes are swapping

<details>
<summary>Answer</summary>

**B** — `id=0` here does not mean busy: the time is in `wa`, and `r=0`
means nothing is waiting for a CPU. The bottleneck is the storage path.

</details>

---

**Q4.** On an 8-CPU host `vmstat` shows `us=13 id=87`, yet one nightly job
is far slower than expected. Which check explains it fastest?

- A) `free -h`
- B) A per-CPU view (`mpstat -P ALL` or `top` with `1`) — a single-threaded job pinned to one core is about 12.5% of eight and hides in the aggregate
- C) `iostat -x`
- D) Restarting the host

<details>
<summary>Answer</summary>

**B** — 100% of one core out of eight averages to about 12.5%. The
aggregate looks idle while the one job that matters is maxed out.

</details>

---

**Q5.** `vmstat` shows `so` sustained in the thousands of KiB/s, `free`
very small, and `r` low. What is the host telling you, and where do you
look next?

- A) CPU saturation — check `perf top`
- B) Memory pressure: it is actively swapping out; find the largest memory consumers and check whether a per-service limit is involved (Demo M07-02)
- C) A disk fault — run `fsck`
- D) Steal time from the hypervisor

<details>
<summary>Answer</summary>

**B** — Sustained swap-out with little free memory is memory pressure.
CPU columns and `r` are not the signal here.

</details>

---

**Q6.** An engineer runs `vmstat` once, sees `bi=6942 sy=33` in the first
row of output on a quiet machine, and concludes there is heavy disk
reading right now. What went wrong?

- A) Nothing — `bi` is always reliable
- B) The first row is an average since boot, not a live reading; the rows after it show what is happening now
- C) `vmstat` counts in different units on the first row
- D) They should have used `top` instead

<details>
<summary>Answer</summary>

**B** — On the authoring host the first row showed `bi=6942 sy=33` while
the following rows were all zero.

</details>

---

**Q7.** `vmstat` shows `us=37 sy=63` on a busy process, with `r` equal to
the CPU count. Which tool from Demo M07-01 is the sensible next step?

- A) `ltrace -c`, because the time is in libraries
- B) `strace -c`, because most CPU time is in the kernel, meaning heavy system-call activity
- C) `perf stat`, because IPC is the only remaining question
- D) None — `sy` above `us` is normal for every workload

<details>
<summary>Answer</summary>

**B** — `sy` larger than `us` points at system calls, which is exactly
what `strace -c` summarizes.

</details>

---

Score guide:

| Score | Action |
|---|---|
| 7/7 | Import Anki cards, move to Demo M07-04 |
| 6/7 | Review the wrong answer, then proceed |
| 5/7 | Re-read the relevant section, retry those questions |
| Below 5/7 | Re-read the full demo and redo the walkthrough before proceeding |
````