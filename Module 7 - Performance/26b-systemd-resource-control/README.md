# Demo 26b — systemd Resource Control

## Overview

Demo 08 mentioned that systemd unit files support real resource control
(`MemoryMax=`, `CPUQuota=`) but never built it. Demo 26 gave you
process-level tracing tools for *why* something is slow — this demo gives
you the tool for *preventing* a service from taking down a host in the
first place, entirely through systemd unit directives, no kernel-level
cgroup mechanics required. The classic symptom this demo exists to
resolve: "a service keeps getting killed under load, but the host shows
free memory the whole time" — the limit is enforced per-service, not
per-host, and that distinction is usually where the confusion starts.

## Demo Objectives

1. Explain what `MemoryMax=`/`MemoryHigh=`/`CPUQuota=` actually do at the
   systemd-directive level, and why a per-service limit can kill a
   service even while the host itself shows free memory
2. Cap a systemd service's memory with `MemoryMax=` and explain why swap
   must be capped alongside it to actually enforce that limit
3. Read `journalctl -k` (the kernel's own OOM report) alongside
   `systemctl status` to confirm a kill was caused by the configured
   limit, not something else
4. Sample a running service's actual memory usage against its configured
   limit with `systemctl show`
5. Distinguish `MemoryMax=` (hard limit, kills) from `MemoryHigh=` (soft
   limit, intended to throttle) and explain what each one guarantees
6. Apply `CPUQuota=` to cap a service's CPU usage

## Directory Structure

```
26b-systemd-resource-control/
├── README.md                                      # this file
├── 26b-systemd-resource-control-anki.csv          # Anki flashcard deck
└── 26b-systemd-resource-control-quiz.md           # standalone quiz
```

## Environments

| Task | Ubuntu 24.04 | Rocky Linux 9.8 | EC2 Amazon Linux |
|---|---|---|---|
| All lab tasks | ✅ (`sudo apt install stress-ng`) | ✅ (`sudo dnf install stress-ng` — same tool, same flags, on both distros) | Not required |

> **Verification note:** This sandbox's `systemctl` binary exists but
> systemd is not running as PID 1 here ("Host is down" when contacting
> the bus), and it uses a cgroup v1 layout rather than v2 — so a real
> unit file couldn't be created and OOM-triggered live in this
> environment. `MemoryMax=`/`CPUQuota=` as directives are long-stable,
> documented systemd behavior, but this demo's specific claims about
> exact `systemctl status`/`journalctl`/`systemctl show` output — which
> fields appear, in what form, at what point in the unit's lifecycle —
> are built from documentation, not a live run, and should be confirmed
> on a real Ubuntu/Rocky9 machine before being treated as settled. Two
> corrections already made to an earlier draft of this demo are noted
> inline where they matter (§1.2, §1.4): the earlier version omitted
> `MemorySwapMax=0` and fabricated a specific `systemctl show` output line
> that was never actually observed.

## Recall Check — 26-performance-troubleshooting

Answer these from memory before continuing — no looking back at Demo 26.

1. Why doesn't a high CPU% in `top` necessarily mean the CPU is doing
   proportionally useful work?
2. What's the difference between what `perf stat` and `perf top` each
   tell you?
3. What is `/proc/PID/` considered relative to `ps`/`top`/`lsof`?

<details>
<summary>Answers</summary>

1. `top`'s CPU% measures time scheduled on a core, not how efficiently
   that time is used — `perf stat`'s instructions-per-cycle (IPC) is the
   actual efficiency signal.

2. `perf stat` measures how efficient a run was overall (cycles,
   instructions, IPC); `perf top` shows, live, which specific function is
   consuming CPU time right now — efficiency versus location.

3. The ground truth every process-inspection tool ultimately reads its
   data from — when a formatted tool's output is ambiguous, reading
   `/proc/PID/status`/`/limits`/`/fd/` directly confirms what's true.

</details>

## Part 1 — Concepts

### 1.1 Why this stays at the systemd-directive level

The kernel's cgroup mechanism is the actual enforcement engine behind
everything in this demo — but this demo deliberately stays at the level
of systemd unit-file directives (`MemoryMax=`, `CPUQuota=`), not
hand-written cgroup filesystem paths. systemd translates these directives
into the underlying cgroup limits for you; you get real, kernel-enforced
resource control without needing to manage `/sys/fs/cgroup/` entries
directly. This is the practical level almost all day-to-day service
resource-limiting actually happens at.

### 1.2 MemoryMax= — the hard limit, and the swap escape route

```ini
[Service]
ExecStart=/usr/bin/stress-ng --vm 1 --vm-bytes 150M --vm-hang 0
MemoryMax=100M
MemorySwapMax=0
```

When a service's memory usage exceeds `MemoryMax=`, the kernel's cgroup
OOM killer targets **that service's own processes specifically** — not
the host in general, and not selected by the host-wide OOM heuristics
that might otherwise pick a completely different, unrelated process to
kill. **This is exactly why a service can be killed while the host itself
shows plenty of free memory**: the limit is enforced per-cgroup
(per-service), entirely independent of host-level memory headroom.

**`MemoryMax=` alone only bounds anonymous+page-cache memory — it does
not bound swap.** On a host with swap enabled, a process pushed against
`MemoryMax=` can be paged out to swap instead of killed, which defeats the
point of setting a hard limit for a memory-hungry service: instead of a
clean, fast failure, you get severe, silent slowdown as the service thrashes
against swap. `MemorySwapMax=0` closes that escape route, so the intended
kill actually happens once `MemoryMax=` is exceeded. Whether this specific
service actually gets killed, or slows down instead, depends on your
host's swap configuration — check `swapon --show`/`free -h` before relying
on `MemoryMax=` alone to produce a kill.

### 1.3 CPUQuota= — capping CPU usage

```ini
[Service]
CPUQuota=50%
```

`CPUQuota=` caps the percentage of CPU time this service's cgroup may
use — `50%` means half of one core's worth of time; a value like `200%`
scales to two full cores' worth on a multi-core host. Enforced by the
kernel's CPU bandwidth controller via systemd's translation.

### 1.4 Reading the aftermath — journalctl -k, status, and show

```
journalctl -k -n 50
```

**This is the primary evidence for an OOM-kill, not `systemctl show`.**
The kernel itself logs the OOM event — which process, which cgroup, why —
to the kernel ring buffer, readable via `journalctl -k` (the kernel-message
equivalent of `dmesg`). Look for a line naming the killed process and,
for a cgroup-triggered kill specifically, wording that identifies the
cgroup/memory-controller as the trigger rather than a general host-wide
OOM condition.

```
systemctl status memtest.service
```

The service's own status should show it exited via a kill signal, which
is corroborating evidence — but the kernel's own OOM report in
`journalctl -k` is what actually confirms *why*.

```
systemctl show memtest.service -p MemoryMax,MemoryCurrent
```

This command is genuinely useful **while the service is still running**
— it lets you sample current usage against the configured limit before
anything gets killed, which is the way to catch a service trending toward
its limit early. **It is not reliable evidence after the unit has already
died**: once a unit exits, `MemoryCurrent` reflects a cgroup that may no
longer exist, and the value you get back should not be assumed to be the
last usage at the moment of the kill without checking on your own system
first — an earlier draft of this demo asserted a specific output value
here that was never actually observed live, which was a mistake. Sample
`MemoryCurrent` *while running*, close to the limit, if you want a
verified before-and-after picture; rely on `journalctl -k` for the kill
event itself.

### 1.5 MemoryHigh= — the soft limit, and what it actually guarantees

```ini
[Service]
MemoryHigh=80M
MemoryMax=100M
```

`MemoryHigh=` is documented to apply memory reclaim pressure once
exceeded, without killing the service outright — the intent is to let a
workload that can tolerate some backpressure slow down and shed load
before ever reaching `MemoryMax=`'s hard kill point. **What this demo can't
promise you is a specific, easily-observed signal in `systemctl
status`/`journalctl` confirming throttling actually happened** — reclaim
pressure is a kernel-internal memory-management response, and whether it
surfaces as anything visible in the unit's own status output is worth
confirming directly on a real system rather than assumed.
`[Unverified — test before use]`
What you can rely on directly: pairing `MemoryHigh=` meaningfully below
`MemoryMax=` is the documented, correct way to give a workload a
throttling stage before a hard kill, whether or not you can easily see the
throttling itself in the logs.

## Part 2 — Hands-On Lab

### Commands used in this demo

| Command | Purpose |
|---|---|
| `MemoryMax=` / `MemorySwapMax=` / `MemoryHigh=` / `CPUQuota=` | Unit-file directives capping memory (hard/swap/soft) and CPU |
| `systemctl daemon-reload` | Load a new/edited unit file |
| `systemctl start/stop/disable --now` | Activate / stop and remove persistence |
| `systemctl status name.service` | Check failure/kill state |
| `journalctl -k -n 50` | Read the kernel's own OOM report |
| `journalctl -u name.service -n 50` | Read the service's own recent log |
| `systemctl show name.service -p KEY1,KEY2` | Sample specific properties while the unit is running |

### Task 1 — Create and cap a test service

```bash
# Ubuntu and Rocky9 — same package, same flags
sudo apt install stress-ng      # Ubuntu
sudo dnf install stress-ng      # Rocky9
```

```bash
sudo tee /etc/systemd/system/memtest.service > /dev/null << 'EOF'
[Unit]
Description=Memory limit test

[Service]
ExecStart=/usr/bin/stress-ng --vm 1 --vm-bytes 150M --vm-hang 0
MemoryMax=100M
MemorySwapMax=0
EOF

sudo systemctl daemon-reload
```

Before starting it, check whether swap is even a factor on this host:

```bash
swapon --show
free -h
```

```bash
sudo systemctl start memtest.service
```

The service is deliberately asked to allocate 150M while capped at a
100M hard limit, with `MemorySwapMax=0` removing the swap escape route
from §1.2 — expect the cgroup OOM killer to intervene.

### Task 2 — Confirm the kill via the kernel's own report

```bash
journalctl -k -n 50
systemctl status memtest.service
```

Confirm `journalctl -k` shows a kernel OOM event naming the process, and
that its timestamp lines up with when you started the service (Task 1).
Confirm `systemctl status` corroborates with a killed/failed state.

### Task 3 — Sample usage against the limit while running, then check host headroom

Repeat Task 1 with a workload that stays just under the limit
(`--vm-bytes 90M` instead of `150M`), so the service keeps running long
enough to sample:

```bash
sudo systemctl start memtest.service
sleep 2
systemctl show memtest.service -p MemoryMax,MemoryCurrent
free -h
sudo systemctl stop memtest.service
sudo systemctl disable memtest.service
```

Confirm `MemoryCurrent` is a real, specific number well under
`MemoryMax=100M` while running — this is the reliable way to use
`systemctl show`, per §1.4. Confirm `free -h` shows meaningful host
headroom regardless of this service's own usage — the concrete
demonstration that host-level and per-service memory accounting are
separate.

### Task 4 — MemoryHigh= paired with MemoryMax=

```bash
sudo tee /etc/systemd/system/memtest.service > /dev/null << 'EOF'
[Unit]
Description=Memory limit test with soft throttle

[Service]
ExecStart=/usr/bin/stress-ng --vm 1 --vm-bytes 90M --vm-hang 0
MemoryHigh=80M
MemoryMax=150M
MemorySwapMax=0
EOF

sudo systemctl daemon-reload
sudo systemctl start memtest.service
systemctl status memtest.service
journalctl -u memtest.service -n 20
```

This workload (90M) sits above `MemoryHigh=` (80M) but below `MemoryMax=`
(150M). Per §1.5, confirm at minimum that the service is **not killed**
(a `MemoryMax=` violation would be) — whether you can observe anything
specific indicating throttling is the open question flagged there.

```bash
sudo systemctl disable --now memtest.service
```

### Task 5 — CPUQuota=

```bash
sudo tee /etc/systemd/system/cputest.service > /dev/null << 'EOF'
[Unit]
Description=CPU quota test

[Service]
ExecStart=/usr/bin/stress-ng --cpu 1
CPUQuota=50%
EOF

sudo systemctl daemon-reload
sudo systemctl start cputest.service
top -b -n 1 | grep stress-ng
sudo systemctl disable --now cputest.service
```

Confirm the `stress-ng` process's CPU usage in `top`'s output stays capped
around 50% of one core, rather than consuming a full core the way an
unconstrained `stress-ng --cpu 1` normally would.

### Task 6 — Cleanup

```bash
sudo rm -f /etc/systemd/system/memtest.service /etc/systemd/system/cputest.service
sudo systemctl daemon-reload
```

## What You Learned

- ✅ Explained what `MemoryMax=`/`MemoryHigh=`/`CPUQuota=` do at the
  systemd-directive level, and why a per-service limit can kill a service
  even while the host shows free memory
- ✅ Capped a systemd service's memory with `MemoryMax=`, and closed the
  swap escape route with `MemorySwapMax=0`
- ✅ Read `journalctl -k` as the primary evidence for a cgroup OOM-kill,
  corroborated by `systemctl status`
- ✅ Sampled a running service's usage against its configured limit with
  `systemctl show`
- ✅ Distinguished `MemoryMax=` (hard, kills) from `MemoryHigh=` (soft,
  intended to throttle) and what each actually guarantees
- ✅ Applied `CPUQuota=` to cap a service's CPU usage

## Part 3 — Break-Fix Scenario

### Scenario — NovaTech Ops: "the service died, but the host had memory"

> **Incident report — NovaTech Ops, ticket #4960**
>
> "app-worker-04 keeps getting killed under load. `free -h` on the host
> shows 6GB free the entire time. The on-call engineer is convinced this
> means the OOM-kill logs are wrong, or something else killed it."

```bash
$ journalctl -k -n 20
... Memory cgroup out of memory: Killed process 8842 (app-worker) ...
$ systemctl status app-worker.service
● app-worker.service - App Worker
     Active: failed (Result: oom-kill)
$ free -h
              total        used        free
Mem:            16G         10G        6.0G
```

Tasks:
1. Reference §1.2/§1.4 — is the on-call engineer's assumption ("6GB free
   means this can't be a real OOM-kill") correct? Why or why not, and
   which piece of evidence above is the actual confirmation?
2. Given the service was likely near its limit for a while before dying,
   what should have been sampled earlier to catch this trending toward
   the limit, rather than only discovering it after the kill?
3. What's the actual fix here, given the workload apparently needs more
   than its current limit — and what directive from §1.5 would give the
   service a chance to signal distress before the next hard kill, even
   though this demo can't promise you'll see that signal clearly in the
   logs?

<details>
<summary>Answers</summary>

The on-call engineer's assumption is incorrect — this is exactly the
confusion this demo exists to resolve. `MemoryMax=` is enforced
per-service (per-cgroup), completely independent of host-level memory
headroom; a service can be killed for exceeding its own configured limit
while the host has abundant free memory, because the two are separate
accounting scopes entirely. `free -h`'s 6GB free figure is real and not in
conflict with the kill — it simply isn't the relevant number for
diagnosing a per-service limit. The actual confirmation is
`journalctl -k`'s kernel-level report naming the process and the memory
cgroup as the trigger — that is the authoritative evidence, not `free -h`
and not, after the fact, `systemctl show`'s `MemoryCurrent` (§1.4).
`systemctl show ... -p MemoryMax,MemoryCurrent`, sampled periodically
*while the service was still running and healthy*, would have shown
usage trending upward toward the limit ahead of time — the earlier the
trend is visible, the more room there is to act before a kill happens.
The fix is raising `MemoryMax=` (and `MemorySwapMax=` if swap is in play)
to accommodate the workload's real peak usage, informed by that kind of
periodic sampling rather than a guess; pairing the new `MemoryMax=` with
`MemoryHigh=` set meaningfully below it is the documented way to give the
service a throttling stage before the next hard kill — even though, per
§1.5, this demo can't guarantee you'll see an obvious log line confirming
the throttling itself happened.

</details>

## Interview Prep

**Q1. A service is repeatedly OOM-killed, but the host itself shows plenty of free memory. Is this a contradiction, and what's the authoritative evidence either way?**

Not a contradiction — `MemoryMax=` is enforced per-service (per-cgroup),
entirely independent of host-level memory headroom. `free -h` is simply
not the relevant figure for a per-service limit. The authoritative
evidence is `journalctl -k` — the kernel's own OOM report, which names the
process and the memory cgroup as the trigger — corroborated by
`systemctl status` showing a killed/failed state.

---

**Q2. Why does `MemoryMax=` alone sometimes fail to produce a clean kill on a host with swap enabled?**

`MemoryMax=` bounds anonymous and page-cache memory, but not swap — a
process pushed against the limit can be paged out to swap instead of
killed, producing severe slowdown rather than the clean, fast failure the
limit was meant to cause. `MemorySwapMax=0` removes that escape route, so
the configured limit actually results in a kill rather than thrashing.

---

**Q3. What's the practical difference between `MemoryHigh=` and `MemoryMax=`, and what should you be careful not to overclaim about `MemoryHigh=`'s effects?**

`MemoryHigh=` is documented to trigger memory reclaim pressure without
killing; `MemoryMax=` is the hard limit that results in a kill. What's
worth being careful about: whether crossing `MemoryHigh=` produces any
specific, easily observed signal in ordinary service logs isn't something
to assume without checking on your actual system — the throttling itself
is a kernel-internal response, and its visibility in `systemctl
status`/`journalctl` shouldn't be taken for granted.

---

**Q4. When is `systemctl show name.service -p MemoryMax,MemoryCurrent` reliable evidence, and when is it not?**

Reliable while the service is still running — it lets you sample actual
usage against the configured limit and catch a trend before a kill
happens. Not reliable as after-the-fact evidence for what usage was *at
the moment of a kill*: once a unit has died, its cgroup may no longer
exist, and the value returned shouldn't be assumed to reflect the last
live usage without confirming on your own system. `journalctl -k` is the
dependable source for the kill event itself.

---

## Key Takeaways

1. **`MemoryMax=` is enforced per-service (per-cgroup), completely
   independent of host-level memory headroom.** A service can be killed
   for exceeding its own limit while the host itself shows abundant free
   memory — this is expected, correct behavior, not a contradiction, and
   `journalctl -k` is the authoritative evidence, not `free -h`.
2. **`MemoryMax=` alone doesn't bound swap — pair it with
   `MemorySwapMax=0`** if you want the limit to actually produce a kill
   rather than severe, silent slowdown as the service pages out to swap
   instead.
3. **`journalctl -k` is the primary evidence for a cgroup OOM-kill.**
   `systemctl show`'s `MemoryCurrent` is genuinely useful for sampling a
   *running* service's usage against its limit, but shouldn't be trusted
   as evidence of usage at the exact moment of a kill after the unit has
   already died.
4. **`MemoryHigh=` is documented to throttle, not kill — but this demo
   can't promise you'll see an obvious log signal confirming that
   throttling happened.** Pair it meaningfully below `MemoryMax=` anyway;
   it's the documented, correct mechanism even where its visibility in
   ordinary logs is unconfirmed.
5. **`CPUQuota=` caps a service's CPU time as a percentage of one core's
   worth** — `50%` is half a core, `200%` is two full cores' worth on a
   multi-core host.
6. **This demo deliberately stays at the systemd-directive level, not
   raw cgroup filesystem mechanics** — the practical level almost all
   real day-to-day resource-limiting work actually happens at, closing
   the loop Demo 08 left open by mentioning this capability without
   building it.

> **Demo scope:** Primary concept: systemd-directive-level resource
> control for services (`MemoryMax=`/`MemoryHigh=`/`CPUQuota=`).
> Supporting concepts: the swap escape route and `MemorySwapMax=`,
> reading OOM-kill evidence via `journalctl -k`/`systemctl status`/
> `systemctl show`, host-vs-per-service memory accounting.
> Estimated completion time: 34 minutes (reading + hands-on + verification).
> Checkpoints: 6 natural stopping points (after Task 1 through Task 6).

---

## Quick Commands Reference

### Unit-file directives

| Directive | Effect |
|---|---|
| `MemoryMax=` | Hard memory limit — exceeding it triggers a cgroup OOM-kill of the service |
| `MemorySwapMax=` | Bounds swap usage — set to `0` so `MemoryMax=` can't be escaped via swap |
| `MemoryHigh=` | Soft memory limit — documented to trigger reclaim/throttling, not a kill |
| `CPUQuota=` | Caps CPU time as a percentage of one core's worth (`200%` = 2 cores) |

### Diagnosing a resource-limit kill

| Command | Purpose |
|---|---|
| `journalctl -k -n 50` | The kernel's own OOM report — the primary evidence |
| `systemctl status name.service` | Corroborating failed/killed state |
| `systemctl show name.service -p MemoryMax,MemoryCurrent` | Sample usage vs. limit **while the service is running** |
| `free -h` | Host-level headroom, for contrast against the per-service limit |

## Appendix A — Anki Cards

**`26b-systemd-resource-control-anki.csv`:**

```
#deck:Linux Mastery::Module 7 - Performance::Demo 26b - systemd Resource Control
#separator:Comma
#columns:Front,Back,Tags
"What's the practical difference between MemoryHigh= and MemoryMax=?","MemoryHigh= is documented to trigger reclaim/throttling, without killing. MemoryMax= is the hard limit that results in a cgroup OOM-kill. Whether MemoryHigh='s throttling is visible in ordinary logs isn't guaranteed — confirm on your own system.","demo26b,memoryhigh,memorymax,fundamentals"
"Why can MemoryMax= alone fail to actually kill a service on a host with swap enabled?","MemoryMax= bounds anonymous/page-cache memory, not swap — a process can be paged out to swap instead of killed. MemorySwapMax=0 closes that escape route.","demo26b,memorymax,swap,common-mistake"
"What's the authoritative evidence that a service was killed by its own MemoryMax= limit, as opposed to something else?","journalctl -k — the kernel's own OOM report, which names the process and identifies the memory cgroup as the trigger. systemctl status corroborates with a killed/failed state.","demo26b,journalctl,oom-kill,diagnosis"
"When is systemctl show name.service -p MemoryMax,MemoryCurrent reliable evidence, and when is it not?","Reliable while the service is still running, for sampling usage against the limit. Not reliable as after-the-fact evidence of usage at the moment of a kill — once the unit has died its cgroup may no longer exist.","demo26b,systemctl-show,limitation"
"What does CPUQuota=50% cap, and what would CPUQuota=200% mean on a multi-core host?","50% caps the service's cgroup to half of one CPU core's worth of time. 200% scales to two full cores' worth of time on a multi-core host.","demo26b,cpuquota,fundamentals"
"Why does this demo stay at the systemd-directive level (MemoryMax=, CPUQuota=) rather than covering raw cgroup filesystem mechanics?","Because that's the practical level almost all day-to-day service resource-limiting actually happens at — systemd translates these directives into real kernel-enforced cgroup limits without needing to manage /sys/fs/cgroup/ paths by hand.","demo26b,scope,fundamentals"
"What loop does this demo close, and from which earlier demo?","Demo 08 (systemd services) mentioned that unit files support real resource control (MemoryMax=/CPUQuota=) but never built it — this demo builds it.","demo26b,cross-demo,fundamentals"
```

## Appendix B — Quiz

**`26b-systemd-resource-control-quiz.md`:**

````markdown
# Quiz — Demo 26b: systemd Resource Control

> One correct answer per question unless stated otherwise.
> Target: 80% or above before moving to Demo 26c.

---

**Q1.** A service configured with `MemoryMax=100M` on a host with swap
enabled is under heavy memory pressure, but instead of being killed it
just gets extremely slow. What's the most likely explanation?

- A) `MemoryMax=` doesn't work on this kernel version
- B) `MemoryMax=` bounds anonymous/page-cache memory but not swap — the process is being paged out to swap instead of killed, since `MemorySwapMax=` was never set
- C) The service must be running as root to be killed
- D) `MemoryMax=` only applies to CPU, not memory

<details>
<summary>Answer</summary>

**B** — This is exactly the swap escape route from §1.2. Adding
`MemorySwapMax=0` closes it, so the limit produces the intended kill
rather than silent thrashing.

</details>

---

**Q2.** A service is repeatedly OOM-killed, and `free -h` shows 6GB free
on the host the whole time. What's the authoritative evidence that
confirms this was actually caused by the service's own `MemoryMax=`
limit?

- A) `free -h`'s 6GB figure itself
- B) `journalctl -k` — the kernel's own OOM report naming the process and the memory cgroup as the trigger
- C) The service's exit code alone, with no further evidence needed
- D) There is no way to confirm this after the fact

<details>
<summary>Answer</summary>

**B** — `free -h` is the wrong figure entirely (host-level, not
per-service). `journalctl -k` is the primary, authoritative evidence.

</details>

---

**Q3.** Why is `systemctl show name.service -p MemoryMax,MemoryCurrent`
described as reliable *while running* but not fully reliable *after* a
kill?

- A) The command doesn't exist after a service has failed
- B) Once a unit has died, its cgroup may no longer exist, so the value returned for a dead unit shouldn't be assumed to reflect usage at the exact moment of the kill without confirming on your own system
- C) `MemoryCurrent` is only accurate for services running as root
- D) It's fully reliable in both cases; there's no meaningful difference

<details>
<summary>Answer</summary>

**B** — This is a real correction from an earlier draft of this demo,
which asserted a specific after-the-fact value that was never actually
observed live. Sample it while running for a trustworthy before-and-after
picture.

</details>

---

**Q4.** What does this demo NOT promise about `MemoryHigh=`'s effects?

- A) That it exists as a valid systemd directive
- B) That crossing it will produce an easily observed, specific signal in ordinary `systemctl status`/`journalctl` output — this is flagged as unconfirmed, not assumed
- C) That it's documented to trigger reclaim pressure rather than a kill
- D) That it should be paired with `MemoryMax=`

<details>
<summary>Answer</summary>

**B** — The directive's documented *intent* (throttle, don't kill) is not
in question; what's flagged as unconfirmed is specifically whether you'll
see clear evidence of that throttling in ordinary service logs.

</details>

---

**Q5.** What does `CPUQuota=200%` mean on a multi-core host?

- A) The service can use 200 CPU cores
- B) The service's cgroup is capped at the equivalent of two full CPU cores' worth of time
- C) The service gets double priority over all other processes
- D) This is invalid syntax — `CPUQuota=` cannot exceed 100%

<details>
<summary>Answer</summary>

**B** — `CPUQuota=` percentages scale beyond 100% to express multiple
cores' worth of allowed time on a multi-core host.

</details>

---

**Q6.** Why does this demo stay at the systemd-directive level rather than
covering raw cgroup filesystem paths directly?

- A) Because cgroups don't actually enforce these limits — systemd does it independently
- B) Because the systemd-directive level is the practical level almost all day-to-day service resource-limiting actually happens at; systemd translates directives into real kernel-enforced cgroup limits for you
- C) Because cgroup filesystem mechanics are deprecated in modern kernels
- D) Because `MemoryMax=`/`CPUQuota=` only work when SELinux is disabled

<details>
<summary>Answer</summary>

**B** — The kernel cgroup mechanism is the real enforcement engine
underneath; engineers interact with it through systemd directives in the
vast majority of real operational work.

</details>

---

**Q7.** What loop does this demo explicitly close, and from which earlier demo?

- A) Demo 26's strace/ltrace tracing, which this demo replaces
- B) Demo 08, which mentioned systemd resource control directives without ever building them
- C) Demo 09/10's monitoring tools, which this demo deprecates
- D) No earlier demo referenced this topic before now

<details>
<summary>Answer</summary>

**B** — Demo 08 (systemd services) specifically mentioned `MemoryMax=`/
`CPUQuota=` as real unit-file capabilities but never built a working
example — this demo is that follow-through.

</details>

---

Score guide:

| Score | Action |
|---|---|
| 7/7 | Import Anki cards, move to Demo 26c |
| 6/7 | Review the wrong answer, then proceed |
| 5/7 | Re-read the relevant section, retry those questions |
| Below 5/7 | Re-read the full demo and redo the walkthrough before proceeding |
````