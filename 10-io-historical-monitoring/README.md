# Demo 10 — I/O & Historical Performance Monitoring

## Overview

Demo 09 covered what a machine is doing *right now*. This demo covers two
things live tools can't give you: per-device disk throughput detail, and what
happened an hour — or a week — ago. `iostat` and `iotop` show you which disk
and which process are responsible for I/O load; `sar`, backed by the
`sysstat` package, quietly records historical performance data so "was it
like this yesterday too?" has an actual answer instead of a guess.

## Demo Objectives

1. Read `iostat` output to identify a specific overloaded disk device
2. Use `iotop` to identify which process is generating disk I/O
3. Install and enable `sysstat`, and explain what `sa1`/`sa2` actually do
4. Query historical data with `sar -u`, `sar -r`, `sar -d`, and `sar -n`
5. Configure `sysstat` data retention and explain why the default often isn't enough
6. Know what `sadf` is for, at a high level

## Directory Structure

```
10-io-historical-monitoring/
└── README.md   # This file — full demo content, including Anki (Appendix A)
                 # and Quiz (Appendix B). Standalone .csv/.md files are
                 # generated later via the series extraction script, not
                 # created manually per demo from this point forward.
```

## Environments

| Task | Ubuntu 24.04 | Rocky Linux 9.8 | EC2 Amazon Linux |
|---|---|---|---|
| All lab tasks | ✅ | ✅ | Not required |
| `sysstat` install | `sudo apt install sysstat` | `sudo dnf install sysstat` | Not required |
| `iotop` install | `sudo apt install iotop` | `sudo dnf install iotop` | Not required |

## Recall Check — 09-live-system-monitoring

Answer these from memory before continuing — no looking back at Demo 09.

**Q1.** `free -h` shows low `free` but high `available`. Is the machine
actually low on memory? Why or why not?

**Q2.** In `vmstat`, which two columns are the clearest signal of genuine
memory pressure (active swapping), as opposed to normal cache usage?

**Q3.** A machine shows a high load average but `top` reports the CPU is
mostly idle with a nonzero `wa`. What does that combination indicate?

<details>
<summary>Answers</summary>

**A1.** No, not necessarily — `available` already accounts for reclaimable
page cache (`buff/cache`); low `free` with high `available` is the normal,
expected pattern on a healthy system, not a warning sign.

**A2.** `si` and `so` (swap in/swap out) — sustained non-zero values mean the
kernel is genuinely moving pages to and from disk due to real memory pressure.

**A3.** An I/O bottleneck, not a CPU one — load average includes processes in
uninterruptible I/O wait, so high load with idle CPU and nonzero `wa` points
at disk/network-bound processes.

</details>

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

The single most useful column for "is this disk the bottleneck" is **`%util`**
— the percentage of time the device was busy servicing requests. Sustained
`%util` near 100% on a device means it's saturated; other processes waiting on
that disk will queue. `aqu-sz` (average queue size) rising alongside high
`%util` confirms requests are backing up, not just that the disk is busy for
an instant. `r/s`/`w/s` (reads/writes per second) and `rkB/s`/`wkB/s`
(throughput) tell you *how much* work is happening; `%util` and `aqu-sz` tell
you whether the device can keep up with it.

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
every tool in Demo 09, it doesn't just show you the present — it queries
*historical* data collected automatically in the background.

```bash
sudo apt install sysstat   # Ubuntu
sudo dnf install sysstat   # Rocky9
```

That historical data doesn't collect itself — two small programs, `sa1` and
`sa2`, do the work, and how they're scheduled is a direct callback to Demo 08:

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
via a systemd timer, the underlying mechanism is exactly what Demo 08 covered
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

```bash
sar -u -f /var/log/sysstat/saXX   # Replace XX with the day-of-month file name
# Without -f, sar reads today's live-collected data; -f points it at a
# specific saved day's file — this is how you answer "what did CPU usage
# look like yesterday at 3 AM," which none of Demo 09's live tools can do.
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

## What You Learned

1. ✅ Read `iostat` output to identify a specific overloaded disk device
2. ✅ Used `iotop` to identify which process is generating disk I/O
3. ✅ Installed and enabled `sysstat`, and explained what `sa1`/`sa2` do
4. ✅ Queried historical data with `sar -u`, `sar -r`, `sar -d`
5. ✅ Configured and inspected `sysstat` data retention
6. ✅ Learned what `sadf` is for, at a high level

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

## Interview Prep

**Q1. `iostat` shows a device at 98% utilization. Does that alone tell you which process to investigate?**
No — `iostat` reports per-device statistics, not per-process. It tells you
*that* a device is saturated and gives throughput/latency detail, but
identifying *which process* is responsible for that load requires `iotop`,
which attributes I/O to individual processes the way `iostat` attributes it
to devices.

**Q2. Why does `sar` exist when `top`/`vmstat`/`free` already show CPU, memory, and I/O?**
Those tools only ever show the current instant — there's no way to ask them
"what did this look like an hour ago" unless you happened to be watching and
recording at the time. `sar`, backed by `sysstat`'s background collection
(`sa1`/`sa2`), is the only tool in this series that can answer a question
about the past on demand, which matters enormously for postmortems and
"was this normal or new" comparisons.

**Q3. A team's `sysstat` retention is left at the packaged default. What's the practical risk, and how would you explain it to someone who thinks "the collection is running, so we're covered"?**
The risk is that "collection is running" only guarantees data exists *right
now* — retention (`HISTORY=`) controls how long it stays before being
deleted. If an incident isn't investigated until after the retention window
has passed (common — postmortems often happen days or weeks later, not
same-day), the historical data needed to actually investigate it is already
gone, even though collection was working the entire time. The fix is setting
`HISTORY` deliberately based on realistic investigation lag, not the
packaged default.

## Key Takeaways

1. **`iostat` reports per-device; `iotop` reports per-process.** They answer
   different questions — "is a device saturated" versus "which process is
   responsible" — and a real I/O investigation usually needs both, in that
   order.
2. **`%util` and `aqu-sz` together are the clearest signal of disk
   saturation.** High `%util` alone shows the device is busy; rising `aqu-sz`
   alongside it confirms requests are actually queuing, which is what
   translates into application-visible latency.
3. **`sar` is the only tool covered so far that answers questions about the
   past, not just right now.** `top`/`vmstat`/`free` (Demo 09) are
   fundamentally live-only; `sar`'s value is entirely in querying data that
   was collected while nobody was watching.
4. **`sa1`/`sa2` are scheduled the same way any periodic job is — cron or a
   systemd timer, depending on the distro.** This is a direct application of
   Demo 08's timer concepts, not a new scheduling mechanism to learn.
5. **Default `sysstat` retention is often too short for real postmortems.**
   `HISTORY=` in `/etc/sysstat/sysstat` should be set based on how long
   incidents realistically take to get investigated, not left at the
   packaged default.
6. **`sadf` exists for exporting `sar` data outside a terminal.** It's not a
   daily-use tool — it's there for the moment historical data needs to go
   into a spreadsheet, chart, or external system.

> **Demo scope:** Primary concept: I/O throughput and historical performance
> trend monitoring. Supporting concepts: iostat, iotop, sysstat setup and
> sa1/sa2 collection, sar querying (-u/-r/-d), retention configuration, sadf.
> Estimated completion time: 34 minutes (reading + hands-on + verification).
> Checkpoints: 5 natural stopping points (after Task 1 through Task 5).

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
| `sadf -d -- -u FILE` | Export sar data as CSV-like output |

## Appendix A — Anki Cards

**Note:** extracted into a standalone `10-io-historical-monitoring-anki.csv`
by the series extraction script — not created manually per demo from this
point forward.

```
#deck:Linux Mastery::Module 2 - System Administration::10-io-historical-monitoring
#separator:Comma
#columns:Front,Back,Tags
"iostat shows a device at 98% util with rising aqu-sz. What does iostat alone NOT tell you, and what tool fills that gap?","iostat reports per-device statistics, not per-process — it doesn't tell you which process is responsible for the load. iotop fills that gap by attributing I/O to individual processes.","demo10,iostat,iotop"
"What do the %util and aqu-sz columns in iostat -x together indicate when both are high?","Disk saturation with actual request queuing — %util shows the device is busy nearly all the time, and rising aqu-sz confirms requests are backing up rather than being serviced instantly, which is what causes application-visible latency.","demo10,iostat,troubleshooting"
"Why does sar exist when top/vmstat/free already cover CPU, memory, and I/O?","Those tools only show the current instant with no history. sar, backed by sysstat's background collection (sa1/sa2), is the only tool covered so far that can answer a question about the past on demand.","demo10,sar,fundamentals"
"What do sa1 and sa2 actually do, and how are they scheduled?","sa1 runs frequently (traditionally every 10 min) and appends a compact data sample; sa2 runs once daily and produces a summary report. They're scheduled via cron or a systemd timer depending on the distro — the same scheduling mechanisms covered in Demo 08, not a new one.","demo10,sar,sysstat,cross-demo"
"A team's sysstat retention (HISTORY=) is left at the packaged default. What's the practical risk?","If an incident isn't investigated until after the retention window passes (common, since postmortems often happen days or weeks later), the historical data needed to investigate it will already be deleted, even though collection was running the whole time.","demo10,sysstat,retention,common-mistake"
"How do you query sar data for a specific past day rather than today's live-collected data?","sar -f /var/log/sysstat/saXX, where XX is the day-of-month of the saved data file you want to read, e.g. sar -u -f /var/log/sysstat/sa07 for the 7th.","demo10,sar,historical-query"
"What's the difference in purpose between iotop's accumulated mode (-a) and its default current-rate view?","Accumulated mode totals I/O since iotop started, better for identifying which process has done the most total I/O over an ongoing incident window; current-rate view shows only the instantaneous snapshot, which can catch a process mid-burst or mid-idle.","demo10,iotop,troubleshooting"
"What is sadf used for, and what does the -d flag do?","sadf exports sar's historical data outside the terminal — into a spreadsheet, chart, or external system. -d outputs it in a semicolon-delimited CSV-like format instead of sar's default human-readable text.","demo10,sadf,export"
"Which sar flag queries historical memory usage, and which queries historical per-device I/O?","sar -r for memory, sar -d for per-device I/O (the historical counterpart to iostat). sar -u covers CPU, sar -n DEV covers network.","demo10,sar,basics"
"iostat and top both show system activity — what's the key structural difference in what each one can report?","top (Demo 09) is a live-only, primarily process/CPU/memory-oriented view. iostat is specifically device-level I/O detail (per-disk throughput, utilization, queue depth) — the two are complementary, not overlapping, since top doesn't break down per-device I/O this way.","demo10,iostat,cross-demo"
```

## Appendix B — Quiz

**Note:** extracted into a standalone `10-io-historical-monitoring-quiz.md`
by the series extraction script — not created manually per demo from this
point forward.

> One correct answer per question unless stated otherwise.
> Target: 80% or above before moving to Demo 11.

**Q1. `iostat -xz` shows a device at 98% `%util` with a high `aqu-sz`. What does this indicate?**

- A) The device is idle and available for more work
- B) The device is saturated, and requests are actively queuing
- C) A process is about to crash
- D) The reported values are a measurement error — utilization can't exceed 90% in practice

<details>
<summary>Answer</summary>

**B** — High `%util` means the device is busy nearly all the time; rising `aqu-sz` confirms requests are queuing rather than being serviced instantly. Trap: A misreads a saturation indicator as the opposite; C and D invent unrelated explanations.

</details>

---

**Q2. `iostat` shows a saturated device. You need to know which specific process is responsible. What's the correct next tool?**

- A) `vmstat`
- B) `free`
- C) `iotop`
- D) `top`

<details>
<summary>Answer</summary>

**C** — `iostat` reports per-device statistics only; `iotop` is the tool that attributes I/O to individual processes. Trap: A, B, and D are all real Demo 09 tools, but none of them break down I/O by process the way `iotop` does.

</details>

---

**Q3. What's the fundamental difference between `sar` and `top`/`vmstat`/`free`?**

- A) `sar` is faster to run
- B) `sar` requires root; the others don't
- C) `sar` can query historical data collected in the past; the others only show the current instant
- D) `sar` only works on Rocky9, not Ubuntu

<details>
<summary>Answer</summary>

**C** — This is `sar`'s entire reason for existing relative to the live tools from Demo 09. Trap: A and B make unrelated or false claims; D is factually wrong, `sysstat`/`sar` is available on both.

</details>

---

**Q4. What do `sa1` and `sa2` actually do?**

- A) They are the two required arguments to every `sar` command
- B) `sa1` collects a data sample frequently; `sa2` produces a daily summary report — both are scheduled via cron or a systemd timer
- C) They are alternative names for `iostat` and `iotop`
- D) They control `sar`'s color output settings

<details>
<summary>Answer</summary>

**B** — `sa1` (frequent sampling) and `sa2` (daily summary) are the background collection mechanism behind `sar`'s historical data, scheduled the same way any periodic job is — the Demo 08 timer/cron pattern applied here, not a new mechanism. Trap: A, C, and D all invent unrelated meanings for these names.

</details>

---

**Q5. `HISTORY=7` is set in `/etc/sysstat/sysstat`. An incident from 10 days ago needs investigating. What happens?**

- A) The data is still available — `HISTORY` only limits live queries, not `-f` queries
- B) The data has already been deleted — it's outside the 7-day retention window
- C) `sar` will automatically extend retention when queried for older data
- D) `HISTORY` only affects CPU data, not other `sar` categories

<details>
<summary>Answer</summary>

**B** — `HISTORY=7` means data files older than 7 days are removed; a 10-day-old incident's data would already be gone. This is exactly why setting retention deliberately (based on realistic investigation lag) matters. Trap: A, C, and D all invent behavior that would make retention configuration not matter, which defeats the point of the setting.

</details>

---

**Q6. Which `sar` flag would you use to check historical memory usage over the last few minutes?**

- A) `sar -u`
- B) `sar -r`
- C) `sar -d`
- D) `sar -n DEV`

<details>
<summary>Answer</summary>

**B** — `-r` is memory. `-u` is CPU, `-d` is per-device I/O, `-n DEV` is network interface throughput. Trap: all four are real `sar` flags, but only one is memory.

</details>

---

**Q7. Why is `iotop`'s accumulated mode (`-a`) often more useful than its default current-rate view during an ongoing incident investigation?**

- A) Accumulated mode uses less CPU to run
- B) Accumulated mode totals I/O since it started, better identifying the process responsible for sustained load rather than a single noisy instantaneous snapshot
- C) Accumulated mode is required — `iotop` doesn't have a current-rate view
- D) Accumulated mode shows network I/O in addition to disk I/O

<details>
<summary>Answer</summary>

**B** — For an ongoing incident, total accumulated I/O per process gives a more reliable picture than one instant, which could catch a process mid-burst or mid-idle. Trap: A and D invent unrelated capabilities; C is factually wrong — the current-rate view is the default.

</details>

---

**Q8. What is `sadf -d` for?**

- A) Deleting old `sar` data files to save disk space
- B) Disabling `sar`'s data collection
- C) Exporting `sar`'s historical data in a CSV-like format for use outside the terminal
- D) Running `sar` in debug mode

<details>
<summary>Answer</summary>

**C** — `sadf -d` outputs `sar`'s historical data in a semicolon-delimited, CSV-like format, useful for spreadsheets, charts, or other external tools. Trap: A, B, and D all invent unrelated meanings for the `-d` flag.

</details>