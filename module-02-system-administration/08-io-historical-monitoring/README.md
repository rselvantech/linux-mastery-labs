# Demo M02-08 — I/O & Historical Performance Monitoring

## Overview

Demo M02-07 covered what a machine is doing *right now*. This demo covers two
things live tools can't give you: per-device disk throughput detail, and what
happened an hour — or a week — ago. `iostat` and `iotop` show you which disk
and which process are responsible for I/O load; `sar`, backed by the
`sysstat` package, quietly records historical performance data so "was it
like this yesterday too?" has an actual answer instead of a guess.

---

## Demo Objectives

By the end of this demo you will be able to:

1. Read `iostat` output to identify a specific overloaded disk device
2. Use `iotop` to identify which process is generating disk I/O
3. Install and enable `sysstat`, and explain what `sa1`/`sa2` actually do
4. Query historical data with `sar -u`, `sar -r`, `sar -d`, and `sar -n`
5. Configure `sysstat` data retention and explain why the default often isn't enough
6. Know what `sadf` is for, at a high level

---

## Directory Structure

```
08-io-historical-monitoring/
├── README.md                                 # this file
├── m02-08-io-historical-monitoring-anki.csv  # Anki flashcard deck
└── m02-08-io-historical-monitoring-quiz.md   # standalone quiz
```

---

## Environments

| Task | Ubuntu 24.04 | Rocky Linux 9.8 | EC2 Amazon Linux |
|---|---|---|---|
| All lab tasks | ✅ | ✅ | Not required |
| `sysstat` install | `sudo apt install sysstat` | `sudo dnf install sysstat` | Not required |
| `iotop` install | `sudo apt install iotop` | `sudo dnf install iotop` | Not required |

---

## Recall Check

> Answer from memory before reading further. From Demo M02-07.

1. `free -h` shows low `free` but high `available`. Is the machine
   actually low on memory? Why or why not?

2. In `vmstat`, which two columns are the clearest signal of genuine
   memory pressure (active swapping), as opposed to normal cache usage?

3. A machine shows a high load average but `top` reports the CPU is
   mostly idle with a nonzero `wa`. What does that combination indicate?

<details>
<summary>Answers</summary>

1. No, not necessarily — `available` already accounts for reclaimable
   page cache (`buff/cache`); low `free` with high `available` is the normal,
   expected pattern on a healthy system, not a warning sign.

2. `si` and `so` (swap in/swap out) — sustained non-zero values mean the
   kernel is genuinely moving pages to and from disk due to real memory
   pressure.

3. An I/O bottleneck, not a CPU one — load average includes processes in
   uninterruptible I/O wait, so high load with idle CPU and nonzero `wa`
   points at disk/network-bound processes.

</details>

---

## Part 1 — Concepts

### 1.1 iostat — per-device I/O throughput

```bash
iostat -xz 2 5
# -x: extended statistics (utilization, queue depth, latency)
# -z: omit devices with zero activity, to cut noise
# 2 5: 2-second interval, 5 samples
```

```
Device            r/s     w/s     rkB/s     wkB/s   rareq-sz  wareq-sz  aqu-sz  %util
nvme0n1          12.50   84.00    512.00   3892.00     41.0      46.3     0.42   38.20
```

These columns answer two different questions, and a real investigation needs
both:

```
"How much work is happening?"        "Can the device keep up with it?"
r/s, w/s      reads/writes per sec    %util    percentage of time the device
rkB/s, wkB/s  throughput               was busy servicing requests
rareq-sz,                             aqu-sz   average queue size — requests
wareq-sz      average size of each             actually waiting, not just
              read/write request               the device being momentarily busy
```

The single most useful column for "is this disk the bottleneck" is **`%util`**
— sustained `%util` near 100% means the device is saturated. But `%util` high
with `aqu-sz` near zero is a different story than both being high together:
a fast device can show high `%util` while still keeping its queue essentially
empty — meaning it's busy but keeping up. `aqu-sz` rising alongside high
`%util` is what confirms requests are actually backing up, not just that the
disk is continuously active.

`rareq-sz`/`wareq-sz` matter for a different reason: the same throughput
number (`rkB/s`) can come from many small requests or few large ones, and
that distinction affects how a device performs — many tiny requests stress
seek/IOPS capacity even at modest total throughput, especially on spinning
disks.

**`iostat` alone doesn't tell you which process is responsible** — it reports
per-device, not per-process, activity. That's what `iotop` is for.

### 1.2 iotop — per-process I/O

```bash
sudo iotop -oPa
# -o: only show processes actually doing I/O (hides idle noise)
# -P: show processes, not threads (cleaner in most cases)
# -a: accumulated I/O since iotop started, not just current rate — better
#     for spotting a process that's done a lot of I/O over the session even
#     if its instantaneous rate looks modest
```

`iotop` is to `iostat` roughly what `top` is to `vmstat` — `iostat` gives the
device-level aggregate, `iotop` gives the process-level breakdown. When
`iostat` shows a saturated device, `iotop` is the next command, not a
replacement for it.

### 1.3 sysstat, sar, and what sa1/sa2 actually do

`sar` (System Activity Reporter) is part of the `sysstat` package, and unlike
every tool in Demo M02-07, it doesn't just show you the present — it queries
*historical* data collected automatically in the background.

```bash
sudo apt install sysstat   # Ubuntu
sudo dnf install sysstat   # Rocky9
```

That historical data doesn't collect itself — two small programs, `sa1` and
`sa2`, do the work, and how they're scheduled is a direct callback to Demo M02-05:

```bash
# Ubuntu:
cat /etc/cron.d/sysstat
# Rocky9 (or any modern systemd-first distro): sysstat commonly ships its own
# .timer units instead of a cron.d file:
systemctl list-timers | grep sysstat
```

`sa1` runs frequently (traditionally every 10 minutes) and appends a compact
binary sample to the day's data file; `sa2` runs once daily and produces a
human-readable summary report. Whether your distro wires this up via cron or
via a systemd timer, the underlying mechanism is exactly what Demo M02-05 covered
— a scheduled job invoking a oneshot collector. Enabling `sysstat`'s own
collection service is the same `enable --now` pattern:

```bash
sudo systemctl enable --now sysstat
```

### 1.4 sar -u / -r / -d / -n — querying historical data

Once collection has been running (even a few minutes gives you something to
query), `sar` reads it back:

```bash
sar -u 1 3          # CPU usage — same fields as top's %Cpu(s), historical
sar -r 1 3          # Memory usage over time
sar -d 1 3          # Per-device I/O over time (the historical counterpart to iostat)
sar -n DEV 1 3      # Network interface throughput over time
```

**Why reach for `sar -d` instead of just running `iostat` on a loop and
piping it to a log file yourself?** Two reasons beyond convenience: `sar`'s
collection already happened before you knew you'd need it — you can query
*yesterday's* data right now, with no prior setup, because `sa1`/`sa2` were
already quietly running. A self-rolled `iostat`-to-a-file loop only has data
from the moment you started it, which is no help for an incident that already
happened before you thought to look.

```bash
sar -u -f /var/log/sysstat/saXX   # Replace XX with the day-of-month file name
# Without -f, sar reads today's live-collected data; -f points it at a
# specific saved day's file — this is how you answer "what did CPU usage
# look like yesterday at 3 AM," which none of Demo M02-07's live tools can do.
```

This is the entire point of `sar` relative to `top`/`vmstat`/`free`: those
tools only ever show *right now*. `sar` is the only tool in this series so far
that can answer a question about the past without you having had a monitoring
dashboard already running and watching at the time.

### 1.5 Retention — /etc/sysstat/sysstat

```bash
cat /etc/sysstat/sysstat | grep -E '^(HISTORY|COMPRESSAFTER)'
```

```
HISTORY=28
COMPRESSAFTER=10
```

`HISTORY` sets how many days of data files are kept before being deleted.
The default (commonly 7–28 days depending on distro packaging) is often not
enough in production — if an incident gets investigated two weeks after it
happened, and retention was set to 7 days, the data is already gone. Setting
`HISTORY` deliberately, rather than trusting the packaged default, is a real
production decision, not a cosmetic one.

`COMPRESSAFTER` is a separate, smaller concern: it sets how many days'
worth of *uncompressed* data files are kept before older ones get compressed
in place to save disk space. It doesn't affect whether data still exists —
only whether it's compressed — so a compressed file older than
`COMPRESSAFTER` but still within `HISTORY` is still fully queryable with
`sar -f`, it just takes marginally longer to read.

### 1.6 sadf — sar data in other formats

```bash
sadf -d -- -u /var/log/sysstat/saXX > cpu_history.csv
# -d: output in CSV-like semicolon-delimited format instead of sar's default
# human-readable text — for feeding into a spreadsheet or another tool.
```

`sadf` isn't something you reach for daily — it exists for the moment you need
`sar`'s historical data somewhere other than a terminal (a chart, a report, an
external monitoring system's import). Knowing it exists is enough; this demo
doesn't go deeper than that.

---

## Part 2 — Hands-On Lab

### Commands used in this demo

| Command | Flag meaning | Purpose |
|---|---|---|
| `iostat -xz N C` | `-x`: extended, `-z`: hide idle devices | Per-device I/O throughput and utilization |
| `iotop -oPa` | `-o`: active only, `-P`: processes, `-a`: accumulated | Per-process I/O attribution |
| `systemctl enable --now sysstat` | — | Start historical data collection |
| `sar -u/-r/-d/-n` | flag selects CPU/memory/disk/network | Query historical performance data |
| `sar -f FILE` | `-f`: read a specific saved day | Query a past day's data, not today's |
| `sadf -d` | `-d`: CSV-like output | Export sar data for external tools |

### Task 1 — iostat: find the busy device

```bash
sudo apt install sysstat iotop -y   # or: sudo dnf install sysstat iotop -y

# Generate disk I/O in the background:
dd if=/dev/zero of=/tmp/iotest bs=1M count=2000 oflag=direct &

iostat -xz 2 5
# Watch %util and aqu-sz climb on the device backing /tmp while dd runs.
wait
rm -f /tmp/iotest
```

### Task 2 — iotop: attribute the I/O to a process

```bash
dd if=/dev/zero of=/tmp/iotest bs=1M count=2000 oflag=direct &
sudo iotop -oPa
# Confirm the dd process is the one accounting for the I/O iostat showed.
wait
rm -f /tmp/iotest
```

### Task 3 — Enable sysstat and inspect its collection schedule

```bash
sudo systemctl enable --now sysstat
systemctl status sysstat

# Ubuntu:
cat /etc/cron.d/sysstat 2>/dev/null
# Rocky9 / systemd-timer-based setups:
systemctl list-timers | grep -i sysstat
```

### Task 4 — Query live-collected data with sar

```bash
sar -u 1 3
sar -r 1 3
sar -d 1 3
```

### Task 5 — Retention and a saved-day query

```bash
cat /etc/sysstat/sysstat | grep -E '^(HISTORY|COMPRESSAFTER)'

ls /var/log/sysstat/ | head -5
# Note the saXX filenames — XX is day-of-month.
sar -u -f /var/log/sysstat/sa$(date +%d)
# Reads today's file explicitly via -f, demonstrating the same query you'd
# run against yesterday's or last week's file by changing the date.
```

---

## What You Learned

- ✅ Read `iostat` output to identify a specific overloaded disk device
- ✅ Used `iotop` to identify which process is generating disk I/O
- ✅ Installed and enabled `sysstat`, and explained what `sa1`/`sa2` do
- ✅ Queried historical data with `sar -u`, `sar -r`, `sar -d`
- ✅ Configured and inspected `sysstat` data retention
- ✅ Learned what `sadf` is for, at a high level

**Key Takeaway:** Every tool in this demo splits into "what's happening" versus "what already happened" — `iostat`/`iotop` are the live, per-device/per-process view Demo M02-07's tools didn't cover; `sar` is the only tool so far that can answer a question about the past, but only because `sa1`/`sa2` were already quietly collecting before you needed them. None of these tools fix anything themselves — they identify where to look next.

---

## Part 3 — Break-Fix Scenario

### Scenario — NovaTech Ops: two I/O and historical monitoring incidents

> **Incident report — NovaTech Ops, ticket #4518**
>
> Alert 1: "db-server-02 disk saturated, app timeouts." Alert 2: a postmortem
> request — "management wants to know what CPU usage looked like last
> Tuesday during the outage, three days ago." You're on call.

**Incident 1 — disk saturated, app timeouts**

```bash
$ iostat -xz 2 3
Device            r/s     w/s     rkB/s     wkB/s   rareq-sz  wareq-sz  aqu-sz  %util
nvme0n1          210.0   890.0   8400.0   35600.0     40.0      40.0     8.75   98.60
```

Tasks:
1. Is this device the bottleneck? Point to the specific two columns that
   justify your answer.
2. Run `iotop -oPa`. Two processes show significant accumulated I/O — a
   database process and a log-shipping process. Which piece of `iotop`'s
   output (accumulated vs. current rate) is more useful here, and why, given
   this is an ongoing incident rather than a one-off spike?
3. Once you've identified the responsible process, what's the actual next
   step — is `iostat`/`iotop` itself the fix, or a diagnostic step toward one?

**Incident 2 — "what did CPU usage look like last Tuesday?"**

```bash
$ sar -u 1 3
# Shows today's live data only — not what's needed.
$ ls /var/log/sysstat/
sa03  sa04  sa05  sa06  sa07  sa08  sa09  sa10
$ cat /etc/sysstat/sysstat | grep HISTORY
HISTORY=7
```

Tasks:
1. Today is the 10th. The incident was "three days ago" (the 7th). Is that
   data still available given the file listing shown? Construct the correct
   `sar -f` command to query it.
2. Now suppose the incident had been 10 days ago instead of 3. Given
   `HISTORY=7`, is that data available? What does this tell you about setting
   `HISTORY` deliberately rather than trusting the packaged default?
3. Management wants the CPU history as a spreadsheet attachment, not a
   terminal readout. Which tool from this demo produces that, and what does
   the `-d` flag do?

<details>
<summary>Answers</summary>

**Incident 1:** Yes — `%util` at 98.60% means the device is essentially
saturated (busy servicing requests almost 100% of the time), and `aqu-sz` at
8.75 confirms requests are actively queuing rather than being serviced
instantly, which is what turns into application-visible latency. Accumulated
I/O (`-a`) is more useful here than instantaneous rate, because an ongoing
incident is better diagnosed by "which process has done the most total I/O
over the window" than by a single noisy snapshot that could catch either
process mid-burst or mid-idle. `iostat`/`iotop` are diagnostic — they identify
the responsible device and process, but the actual fix (throttling the
log-shipping process, moving it to a different disk, adding I/O capacity,
`ionice`-ing it) is a separate action taken based on what they showed, not
something either tool does itself.

**Incident 2:** Yes — with `HISTORY=7` and today being the 10th, the 7th is
still within the retention window (the file listing confirms `sa07` exists).
Correct query: `sar -u -f /var/log/sysstat/sa07`. If the incident had been 10
days ago, that data would already be gone under `HISTORY=7` — this is exactly
why relying on the packaged default retention is risky for production
postmortems: incidents often aren't investigated same-week, and a retention
window shorter than your typical investigation lag means the data simply
won't exist when someone asks for it. `sadf -d` produces the
spreadsheet-friendly export — `-d` outputs sar's historical data in a
semicolon-delimited CSV-like format instead of its default human-readable
text.

</details>

---

## Interview Prep

**Q1. `iostat` shows a device at 98% utilization. Does that alone tell you which process to investigate?**

No — `iostat` reports per-device statistics, not per-process. It tells you
*that* a device is saturated and gives throughput/latency detail, but
identifying *which process* is responsible for that load requires `iotop`,
which attributes I/O to individual processes the way `iostat` attributes it
to devices.

---

**Q2. Why does `sar` exist when `top`/`vmstat`/`free` already show CPU, memory, and I/O?**

Those tools only ever show the current instant — there's no way to ask them
"what did this look like an hour ago" unless you happened to be watching and
recording at the time. `sar`, backed by `sysstat`'s background collection
(`sa1`/`sa2`), is the only tool in this series that can answer a question
about the past on demand, which matters enormously for postmortems and
"was this normal or new" comparisons.

---

**Q3. A team's `sysstat` retention is left at the packaged default. What's the practical risk, and how would you explain it to someone who thinks "the collection is running, so we're covered"?**

The risk is that "collection is running" only guarantees data exists *right
now* — retention (`HISTORY=`) controls how long it stays before being
deleted. If an incident isn't investigated until after the retention window
has passed (common — postmortems often happen days or weeks later, not
same-day), the historical data needed to actually investigate it is already
gone, even though collection was working the entire time. The fix is setting
`HISTORY` deliberately based on realistic investigation lag, not the
packaged default.

---

## Key Takeaways

1. **`iostat` reports per-device; `iotop` reports per-process.** They answer
   different questions — "is a device saturated" versus "which process is
   responsible" — and a real I/O investigation usually needs both, in that
   order.
2. **`%util` and `aqu-sz` together are the clearest signal of disk
   saturation.** High `%util` alone shows the device is busy; rising `aqu-sz`
   alongside it confirms requests are actually queuing, which is what
   translates into application-visible latency. High `%util` with `aqu-sz`
   near zero means busy but keeping up — a different, healthier situation.
3. **`sar` is the only tool covered so far that answers questions about the
   past, not just right now.** `top`/`vmstat`/`free` (Demo M02-07) are
   fundamentally live-only; `sar`'s value is entirely in querying data that
   was collected while nobody was watching.
4. **`sa1`/`sa2` are scheduled the same way any periodic job is — cron or a
   systemd timer, depending on the distro.** This is a direct application of
   Demo M02-05's timer concepts, not a new scheduling mechanism to learn.
5. **Default `sysstat` retention is often too short for real postmortems.**
   `HISTORY=` in `/etc/sysstat/sysstat` should be set based on how long
   incidents realistically take to get investigated, not left at the
   packaged default. `COMPRESSAFTER=` is a separate, smaller concern — it
   only affects whether older files are compressed, not whether they still
   exist.
6. **`sadf` exists for exporting `sar` data outside a terminal.** It's not a
   daily-use tool — it's there for the moment historical data needs to go
   into a spreadsheet, chart, or external system.

---

## Quick Commands Reference

### Point-in-time device/process I/O

| Command | Purpose |
|---|---|
| `iostat -xz N C` | Per-device I/O, extended stats, N-second interval, C samples |
| `iotop -oPa` | Per-process I/O, active only, accumulated totals |

### Historical data collection and query

| Command | Purpose |
|---|---|
| `systemctl enable --now sysstat` | Start background historical data collection |
| `sar -u N C` | Historical CPU usage |
| `sar -r N C` | Historical memory usage |
| `sar -d N C` | Historical per-device I/O |
| `sar -n DEV N C` | Historical network interface throughput |
| `sar -u -f /var/log/sysstat/saXX` | Query a specific saved day (XX = day-of-month) |

### Retention and export

| Command | Purpose |
|---|---|
| `HISTORY=` in `/etc/sysstat/sysstat` | Days of historical data retained |
| `COMPRESSAFTER=` in `/etc/sysstat/sysstat` | Days before uncompressed files are compressed (doesn't affect retention) |
| `sadf -d -- -u FILE` | Export sar data as CSV-like output |

---

## Appendix A — Anki Cards

**`m02-08-io-historical-monitoring-anki.csv`:**

```
#deck:Linux Mastery::Module 2 - System Administration::Demo M02-08 - I/O and Historical Performance Monitoring
#separator:Comma
#columns:Front,Back,Tags

"iostat shows a device at 98% util with rising aqu-sz. What does iostat alone NOT tell you, and what tool fills that gap?","iostat reports per-device statistics, not per-process — it doesn't tell you which process is responsible for the load. iotop fills that gap by attributing I/O to individual processes.","demo-m02-08,iostat,iotop"

"What do the %util and aqu-sz columns in iostat -x together indicate when both are high?","Disk saturation with actual request queuing — %util shows the device is busy nearly all the time, and rising aqu-sz confirms requests are backing up rather than being serviced instantly, which is what causes application-visible latency.","demo-m02-08,iostat,troubleshooting"

"Why does sar exist when top/vmstat/free already cover CPU, memory, and I/O?","Those tools only show the current instant with no history. sar, backed by sysstat's background collection (sa1/sa2), is the only tool covered so far that can answer a question about the past on demand.","demo-m02-08,sar,fundamentals"

"What do sa1 and sa2 actually do, and how are they scheduled?","sa1 runs frequently (traditionally every 10 min) and appends a compact data sample; sa2 runs once daily and produces a summary report. They're scheduled via cron or a systemd timer depending on the distro — the same scheduling mechanisms covered in Demo M02-05, not a new one.","demo-m02-08,sar,sysstat,cross-demo"

"A team's sysstat retention (HISTORY=) is left at the packaged default. What's the practical risk?","If an incident isn't investigated until after the retention window passes (common, since postmortems often happen days or weeks later), the historical data needed to investigate it will already be deleted, even though collection was running the whole time.","demo-m02-08,sysstat,retention,common-mistake"

"How do you query sar data for a specific past day rather than today's live-collected data?","sar -f /var/log/sysstat/saXX, where XX is the day-of-month of the saved data file you want to read, e.g. sar -u -f /var/log/sysstat/sa07 for the 7th.","demo-m02-08,sar,historical-query"

"What's the difference in purpose between iotop's accumulated mode (-a) and its default current-rate view?","Accumulated mode totals I/O since iotop started, better for identifying which process has done the most total I/O over an ongoing incident window; current-rate view shows only the instantaneous snapshot, which can catch a process mid-burst or mid-idle.","demo-m02-08,iotop,troubleshooting"

"What is sadf used for, and what does the -d flag do?","sadf exports sar's historical data outside the terminal — into a spreadsheet, chart, or external system. -d outputs it in a semicolon-delimited CSV-like format instead of sar's default human-readable text.","demo-m02-08,sadf,export"

"Which sar flag queries historical memory usage, and which queries historical per-device I/O?","sar -r for memory, sar -d for per-device I/O (the historical counterpart to iostat). sar -u covers CPU, sar -n DEV covers network.","demo-m02-08,sar,basics"

"iostat and top both show system activity — what's the key structural difference in what each one can report?","top (Demo M02-07) is a live-only, primarily process/CPU/memory-oriented view. iostat is specifically device-level I/O detail (per-disk throughput, utilization, queue depth) — the two are complementary, not overlapping, since top doesn't break down per-device I/O this way.","demo-m02-08,iostat,cross-demo"
```

---

## Appendix B — Quiz

**`m02-08-io-historical-monitoring-quiz.md`:**

````markdown
# Quiz — Demo M02-08: I/O & Historical Performance Monitoring

> One correct answer per question unless stated otherwise.
> Target: 80% or above before moving to Demo M03-01.

---

**Q1.** `iostat -x` reports both throughput columns (`r/s`, `w/s`, `rkB/s`,
`wkB/s`) and utilization columns (`%util`, `aqu-sz`). Why do you need both
categories rather than just `%util`?

- A) Throughput columns are redundant with `%util` and can be ignored
- B) Throughput columns answer "how much work is happening"; utilization
  columns answer "can the device keep up with it" — a device can show high
  throughput with low utilization (fast device, easy workload) or high
  utilization with modest throughput (slow device, same workload), and only
  looking at one tells an incomplete story
- C) `%util` is deprecated in favor of throughput columns on modern kernels
- D) Utilization columns only matter for network devices, not disks

<details>
<summary>Answer</summary>

**B** — These measure genuinely different things. A fast NVMe device might
show low `%util` at a throughput level that would saturate a slower disk;
reading only one category can mask what's actually happening.

</details>

---

**Q2.** `/etc/sysstat/sysstat` has both `HISTORY=28` and `COMPRESSAFTER=10`.
What does `COMPRESSAFTER` actually control, as distinct from `HISTORY`?

- A) The number of days before files are compressed in place — it doesn't
  affect whether the data still exists or is queryable, only whether it's
  stored compressed
- B) The same thing as `HISTORY`, just under a different name for compatibility
- C) How many days of data `sar` will compress into a single summary before deleting the rest
- D) The number of days before `sar` refuses to accept new incoming data

<details>
<summary>Answer</summary>

**A** — `HISTORY` controls actual deletion/retention; `COMPRESSAFTER`
only controls when older-but-still-retained files get compressed to save
disk space. A file older than `COMPRESSAFTER` but within `HISTORY` is still
fully queryable with `sar -f`, just compressed on disk.

</details>

---

**Q3.** What does `sar -n DEV` show that `sar -u`, `sar -r`, and `sar -d`
do not?

- A) Historical network interface throughput — none of the other three
  flags cover network activity at all
- B) The same CPU data as `sar -u`, just formatted differently
- C) DNS resolution history
- D) It requires root and the other three don't

<details>
<summary>Answer</summary>

**A** — `-u` is CPU, `-r` is memory, `-d` is per-device I/O, and `-n DEV`
is specifically network interface throughput — a fourth, distinct category
none of the others cover.

</details>

---

**Q4.** `iostat` and `iotop` together identify that a specific process on a
specific device is causing disk saturation during an ongoing incident. Have
you fixed the problem at this point?

- A) Yes — running these commands and identifying the cause resolves the
  incident automatically
- B) No — `iostat`/`iotop` are diagnostic tools that identify where to look;
  the actual remediation (throttling the process, moving it to different
  storage, adding capacity, `ionice`-ing it) is a separate action taken
  based on what they showed
- C) Yes, but only if run with the `--fix` flag
- D) No — you would need to restart the entire server regardless of what these tools show

<details>
<summary>Answer</summary>

**B** — Identification and remediation are separate steps. Neither `iostat`
nor `iotop` changes anything on the system; they only report what's
happening so a human can decide and take the actual fix.

</details>

---

**Q5.** Why would you use `sar -d` to investigate an I/O incident from
yesterday, rather than just running `iostat` repeatedly and saving the
output to a log file yourself?

- A) `sar -d` is more accurate than `iostat` for the exact same moment in time
- B) `sar`'s collection (via `sa1`/`sa2`) was already running in the
  background before you knew you'd need the data — a self-rolled `iostat`
  logging loop only has data from whenever you happened to start it, which
  is no help for an incident that already happened
- C) `iostat` cannot be redirected to a file; only `sar` supports output redirection
- D) `sar -d` and a logged `iostat` loop produce identical data, so there's no real reason to prefer either

<details>
<summary>Answer</summary>

**B** — The value of `sar` is entirely in having collection already running
before the need arises. A manual logging loop you start today can never
answer a question about yesterday.

</details>

---

**Q6.** `sar` historical files are named `saXX` (e.g. `sa07`, `sa10`). What
does `XX` represent, and why does this matter when constructing a `-f` query?

- A) `XX` is a random identifier assigned at creation and has no predictable meaning
- B) `XX` is the day-of-month the data was collected — to query a specific
  past date you construct the filename directly from that date
  (`sa07` for the 7th), not from a generic "yesterday" or "last week" label
- C) `XX` is a sequential counter starting from when `sysstat` was first installed
- D) `XX` represents the hour of the day, not the date

<details>
<summary>Answer</summary>

**B** — The naming is date-based and predictable, which is exactly what
makes constructing a `sar -f` query for "last Tuesday" or "10 days ago" a
matter of simple date arithmetic rather than searching for the right file.

</details>

---

**Q7.** A device shows `%util` at 95% but `aqu-sz` staying near 0 throughout.
How does this differ from a device showing both `%util` at 95% AND `aqu-sz`
climbing steadily?

- A) They mean the same thing — both indicate saturation requiring immediate action
- B) High `%util` with near-zero `aqu-sz` means the device is busy but
  keeping up with demand (requests aren't backing up); high `%util` with
  rising `aqu-sz` means requests are actively queuing, which is the real
  saturation signal that leads to application-visible latency
- C) Near-zero `aqu-sz` means the `%util` reading is unreliable and should be ignored
- D) `aqu-sz` only matters for network devices, not for local disks

<details>
<summary>Answer</summary>

**B** — `%util` alone tells you the device is active; `aqu-sz` tells you
whether that activity is translating into a backlog. A busy-but-keeping-up
device is a fundamentally different situation from one where requests are
piling up.

</details>

---

**Q8.** `rareq-sz`/`wareq-sz` in `iostat -x` output show the average size of
each read/write request. Why does this matter beyond just knowing total
throughput (`rkB/s`/`wkB/s`)?

- A) It doesn't matter — throughput alone fully describes disk performance
- B) The same throughput can come from many small requests or few large
  ones, and that distinction affects how a device performs — many tiny
  requests stress a device's IOPS/seek capacity even at modest total
  throughput, especially on spinning disks
- C) `rareq-sz`/`wareq-sz` are only relevant for SSDs, never for HDDs
- D) These fields replace `%util` entirely on modern kernels

<details>
<summary>Answer</summary>

**B** — Request size is a distinct dimension from total throughput. A
device handling many small requests can be under real stress (particularly
seek-bound spinning disks) even when the raw throughput number looks
unremarkable.

</details>

---

| Score | Action |
|---|---|
| 8/8 | Import Anki cards, move to Demo M03-01 |
| 6-7/8 | Review the wrong answers, then proceed |
| 4-5/8 | Re-read the relevant sections, retry those questions |
| Below 4/8 | Re-read the full demo and redo the walkthrough before proceeding |
````