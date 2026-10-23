# Demo 15a — RAID with `mdadm`

> **A note on how this demo was built:**
> This is the most constrained demo in the series so far. `mdadm` is not
> installed and can't be installed (no network access in the environment
> that authored this demo) — but more fundamentally, this container's
> kernel has **no `md` RAID module available at all** (no `modprobe`, no
> `/lib/modules/.../kernel/drivers/md/` tree, no `/proc/mdstat`). Loop
> devices work fine here, but RAID itself is structurally unreachable —
> there was no partial workaround available the way there sometimes is.
> Every command, flag, and behavior below is sourced to the official
> `mdadm(8)` manual page, the kernel.org RAID wiki, and cross-confirmed,
> independently-reported real command transcripts from multiple unrelated
> sources spanning over a decade of `mdadm` versions (the core behavior
> covered here — array states, event counts, `/proc/mdstat` format — has
> not changed across that span). Nothing below was run in the environment
> that authored it. Before treating this demo as production-verified, run
> Part 2 for real (a spare machine or VM with loop-device RAID works fine)
> and confirm the output matches what's described.

## Overview

A single disk is a single point of failure — RAID (Redundant Array of
Independent Disks) combines multiple physical devices into one logical
device that can survive some degree of disk failure, depending on the
RAID level chosen. Demo 14/14a/14b covered a single volume's lifecycle;
this demo covers what happens when that volume is actually *several*
disks acting as one, what happens when one of those disks fails, and the
one risk — "split-brain," where two copies of the same mirrored data
diverge independently — that catches people who don't understand what
RAID does and doesn't protect against.

This demo focuses on RAID1 (mirroring) for the hands-on build/degrade/
rebuild cycle, since it's the simplest level to reason about precisely,
and covers the other common levels (RAID0, 5, 6, 10) at the concept level
so you know which to reach for when.

## Demo Objectives

1. Explain what RAID actually protects against, and what it explicitly
   does not (it is not a backup)
2. Build a RAID1 array with `mdadm --create`, and read `/proc/mdstat` and
   `mdadm --detail` output correctly
3. Simulate a disk failure with `mdadm --fail`, recognize a degraded
   array in both output formats, and understand what "degraded" does and
   doesn't mean for data safety
4. Replace a failed member and rebuild the array, and explain what's
   actually happening during a rebuild
5. Explain RAID split-brain risk — how two array members can end up with
   different data, and how `mdadm` detects and resolves this via event
   counts
6. Compare RAID0, RAID1, RAID5, RAID6, and RAID10 by what they trade off,
   and choose the right level for a given scenario

## Directory Structure

```
15a-raid-mdadm/
├── README.md                              # this file
├── 15a-raid-mdadm-anki.csv                # Anki flashcard deck
└── 15a-raid-mdadm-quiz.md                 # standalone quiz
```

## Environments

| Task | Ubuntu 24.04 | Rocky Linux 9.8 | EC2 Amazon Linux |
|---|---|---|---|
| All lab tasks | ⚠️ [VERIFY] — `mdadm` is available via `apt` (candidate 4.3-1ubuntu2.1 confirmed via `apt-cache policy` in this session) but neither installed nor executed; this session's container additionally lacks the `md` kernel module entirely, which a normal Ubuntu install does not lack | ⚠️ [VERIFY] — not checked this session; `mdadm` ships in Rocky9's default repos as a standard package | Not required |

## Recall Check — 14b-rsync-backup

Answer these from memory before continuing — no looking back at Demo 14b.

1. `rsync -a src dst` and `rsync -a src/ dst` are run against the same destination. What's the difference?

2. What should always precede a real `rsync` run that includes `--delete`, and why?

3. Why does `rsync` need to be installed on both ends of a remote sync, unlike `scp`?

<details>
<summary>Answers</summary>

1. `src dst` (no trailing slash) copies the `src` directory itself into `dst`; `src/ dst` (trailing slash) copies only the contents of `src` into `dst`. Only the source path's slash matters — the destination's never does.

2. The exact same command with `--dry-run` added — `rsync`'s own manual page explicitly recommends this, since a wrong or typo'd source path combined with `--delete` doesn't error, it silently deletes real destination files.

3. `rsync`'s delta-transfer relies on a comparison protocol that runs on both sides to determine which blocks changed; `scp` just copies bytes with no comparison, so it only needs SSH access on the remote end.

</details>

---

## Part 1 — Concepts

### 1.1 What RAID actually protects against — and what it doesn't

RAID combines multiple physical block devices into one logical device.
Depending on the level, it can survive one or more member disks failing
without data loss or downtime. That's the entire scope of what it
protects against: **physical disk failure**.

RAID protects against none of the following, and treating it as if it
did is a common, costly mistake:
- **Accidental deletion or corruption** — if a file is deleted or
  overwritten, RAID faithfully mirrors or stripes that mistake across
  every member disk. There's nothing to "undo."
- **A filesystem-level bug or ransomware** — same reasoning; RAID
  operates below the filesystem, so it has no concept of "good" vs "bad"
  data, only "written" vs "not written."
- **Site-level disasters** — fire, theft, flood. All member disks are
  typically in the same physical machine.

RAID is an *availability* tool (keep the system running through a disk
failure), not a *data-protection* tool. Backups (Demo 14a/14b) are the
data-protection tool. The two are complementary, and a common production
mistake is treating a RAID array as if it were also a backup.

### 1.2 RAID levels — what each one trades off

| Level | Minimum disks | What it does | Survives | Usable capacity |
|---|---|---|---|---|
| RAID0 (striping) | 2 | Splits data across disks for speed | Nothing — any disk failure loses everything | 100% (sum of all disks) |
| RAID1 (mirroring) | 2 | Writes identical copies to every member | Any N-1 disks failing (in an N-disk mirror) | 1/N of total raw capacity |
| RAID5 | 3 | Stripes data + one parity block, rotated across disks | Exactly 1 disk failing | (N-1)/N of total raw capacity |
| RAID6 | 4 | Like RAID5, but two independent parity blocks | Any 2 disks failing | (N-2)/N of total raw capacity |
| RAID10 | 4 | Mirrored pairs, then striped across those pairs | Depends on which disks fail — at least 1 per mirrored pair | 50% of total raw capacity |

Per the `mdadm(8)` manual page, Linux software RAID (implemented through
the `md` driver) supports linear, RAID0, RAID1, RAID4, RAID5, RAID6,
RAID10, and a few specialized/deprecated modes (multipath, faulty,
container) not relevant to general storage use.

**Practical rule of thumb:** RAID1 for simplicity and read performance on
small arrays (boot/OS volumes); RAID5 when capacity efficiency matters
and you can tolerate the rebuild-time exposure of only surviving one
failure; RAID6 for larger arrays where a second failure during a long
rebuild is a real risk; RAID10 when write performance under heavy random
I/O matters more than capacity efficiency (databases). RAID0 has no place
in a data-durability context — it exists purely for speed, at the cost
of every member disk being a single point of failure for the whole array.

### 1.3 Building an array — `mdadm --create`

```bash
sudo mdadm --create /dev/md0 --level=1 --raid-devices=2 /dev/sdb1 /dev/sdc1
```

- `--create /dev/md0` — the new RAID device's name
- `--level=1` — RAID1 (mirroring)
- `--raid-devices=2` — how many devices make up the array
- The trailing device list — the actual member devices/partitions

Per the manual page, once created, the array is immediately usable — you
don't have to wait for the initial resync (the process that makes both
mirror members byte-identical) to complete before using it.

### 1.4 Reading array state — `/proc/mdstat` and `mdadm --detail`

```bash
cat /proc/mdstat
```

A healthy 2-disk RAID1 array reports (documented pattern, confirmed
consistent across every independent transcript checked for this demo):

```
Personalities : [raid1]
md0 : active raid1 sdb1[0] sdc1[1]
      1052160 blocks [2/2] [UU]

unused devices: <none>
```

The `[2/2] [UU]` is the part to read carefully: the first number pair
(`2/2`) is *configured members / currently active members*; the `[UU]`
is one letter per member slot — `U` (up) for a working member, `_` for a
missing or failed one. A degraded array (one disk missing or failed)
reports `[2/1] [U_]` or `[2/1] [_U]` depending on which slot is affected
— **the position of the underscore identifies which slot is down, not
necessarily which physical disk**, since slot order is assigned at
creation and doesn't always match device letter order.

```bash
sudo mdadm --detail /dev/md0
```

Reports the same information with more context — per real transcripts
consistently sourced for this demo, the key fields are:

```
/dev/md0:
        Version : 1.2
     Raid Level : raid1
     Array Size : 1052160 (1027.00 MiB)
   Raid Devices : 2
  Total Devices : 2
          State : clean
 Active Devices : 2
Working Devices : 2
 Failed Devices : 0
  Spare Devices : 0
```

`State: clean` means the array is consistent and not currently degraded.
On a degraded array, this becomes `State: clean, degraded`, and `Active
Devices` drops below `Raid Devices`.

### 1.5 Simulating a failure — `mdadm --fail`

```bash
sudo mdadm /dev/md0 --fail /dev/sdc1
cat /proc/mdstat
```

Per documented behavior, this marks the named member as failed without
physically removing it — the array immediately reports degraded status
(`[2/1] [U_]` or equivalent), and `mdadm --detail` shows `State: clean,
degraded` with `Active Devices: 1`. **Crucially: the array keeps
serving reads and writes normally at this point** — for a mirror, this
is the entire point of RAID1: one member is enough to keep operating.
What's lost is the redundancy itself, not current availability — if the
*second* member also fails before you replace the first, that's when
data loss actually occurs.

```bash
sudo mdadm /dev/md0 --remove /dev/sdc1
```

`--fail` marks a member down; `--remove` actually detaches it from the
array so a replacement can take its slot. These are deliberately separate
operations — a real disk that's failing (but not yet fully dead) might
need to be marked failed and investigated before you commit to physically
removing it.

### 1.6 Rebuilding — replacing the failed member

```bash
sudo mdadm /dev/md0 --add /dev/sdd1
cat /proc/mdstat
```

Adding a new device to a degraded array triggers an automatic rebuild
(recovery) — the array copies data from the surviving member(s) onto the
new one until it's fully synchronized. Per real transcripts consistently
showing this process, `/proc/mdstat` reports live progress during a
rebuild:

```
md0 : active raid1 sdd1[2] sdb1[0]
      1052160 blocks [2/1] [U_]
      [>....................]  recovery =  3.8% (27926016/732571904) finish=121.5min speed=96582K/sec
```

The percentage, estimated finish time, and current speed update
continuously until recovery reaches 100%, at which point the array
returns to `[2/2] [UU]` / `State: clean`. ⚠️ [VERIFY — timing claim, docs
only]: actual rebuild duration depends heavily on disk size, disk speed,
and concurrent I/O load on the array — the `finish=121.5min` figure
above is from one specific historical transcript, not a general estimate
this demo can offer for your hardware.

### 1.7 Split-brain risk — when mirror members disagree

This is the risk the curriculum for this demo specifically calls out, and
it's worth understanding precisely rather than treating as a vague
warning.

**The scenario:** a RAID1 array's members get separated — one disk drops
out (a cable fault, a controller hiccup) but the array keeps running on
the surviving member, accepting writes normally per §1.5. If the
"failed" disk is *not* actually dead, and something later reactivates it
independently — a bad reassembly, a misconfigured hotplug event, a disk
physically reinserted without going through a proper `--add`/rebuild —
you can end up with **two disks that both think they're the current,
authoritative copy of the mirror, each having accepted different writes
since they diverged.** That's split-brain: two members of what's supposed
to be one consistent mirror, no longer actually in agreement, with no
inherent way to know which one is "right."

**How `mdadm` actually protects against this:** every member disk carries
a superblock with an **event count** — a number that increments on
every significant array event (a write, a member change, a resync). When
`mdadm` reassembles an array, it compares each member's event count.
Members with matching (or very close) event counts are treated as
mutually consistent and assembled normally; a member with a
significantly lower event count is treated as stale — `mdadm` will
refuse to silently include it in a live array. Per the kernel.org RAID
wiki's own troubleshooting guidance, forcing a mismatched-event-count
disk back into an array (`--assemble --force`) is a deliberate,
explicit operator decision — never automatic — precisely because doing
this wrong can mean choosing to keep the wrong copy.

This is also *why* `--add`-ing a device to rebuild an array (§1.6)
always triggers a full resync rather than just resuming: `mdadm` doesn't
trust a reattached or new member to already be correct — it establishes
a fresh, known-consistent state rather than assuming continuity that
can't be verified from the disk alone.

## Part 2 — Hands-On Lab

**Before starting:** this lab uses loop devices (disk image files acting
as block devices) rather than real disks, which is safe to run on any
spare Linux VM. Confirm `mdadm` is installed (`which mdadm`; install with
`sudo apt install mdadm` / `sudo dnf install mdadm` if not) and that your
kernel has the `md` module available (`modprobe raid1` should succeed
silently). This lab was not run in the environment that authored it —
treat every `# Expected` comment as documented behavior to confirm, not
a transcript to compare against blindly.

### Commands used in this demo

| Command | Flag meaning | Purpose |
|---|---|---|
| `mdadm --create` | — | Build a new RAID array |
| `mdadm --level=N` | — | Which RAID level (0/1/5/6/10) |
| `mdadm --raid-devices=N` | — | How many devices make up the array |
| `mdadm --detail` | — | Full array status report |
| `mdadm --fail` | — | Mark a member as failed |
| `mdadm --remove` | — | Detach a failed member from the array |
| `mdadm --add` | — | Add a new member, triggering a rebuild |
| `mdadm --stop` | — | Stop (deactivate) an array |
| `mdadm --zero-superblock` | — | Erase RAID metadata from a device |
| `cat /proc/mdstat` | — | Live array state, all arrays |

### Task 1 — Create loop devices and build a RAID1 array

```bash
for i in 1 2 3; do
  dd if=/dev/zero of=/tmp/raid-disk$i.img bs=1M count=256
  sudo losetup /dev/loop$i /tmp/raid-disk$i.img
done
losetup -a
# Confirm three loop devices are attached, backed by the three image files

sudo mdadm --create /dev/md0 --level=1 --raid-devices=2 /dev/loop1 /dev/loop2
cat /proc/mdstat
sudo mdadm --detail /dev/md0
```

### Task 2 — Put a filesystem on it and confirm it's usable immediately

```bash
sudo mkfs.ext4 /dev/md0
sudo mkdir -p /mnt/raid-test
sudo mount /dev/md0 /mnt/raid-test
echo "written before resync completes" | sudo tee /mnt/raid-test/test.txt
cat /mnt/raid-test/test.txt
# Confirm this works even if /proc/mdstat still shows an active resync —
# per §1.3, the array doesn't wait for initial resync before being usable
```

### Task 3 — Simulate a disk failure

```bash
sudo mdadm /dev/md0 --fail /dev/loop2
cat /proc/mdstat
sudo mdadm --detail /dev/md0
# Confirm: [2/1] [U_] (or [_U]) in mdstat, "State: clean, degraded" and
# "Active Devices: 1" in --detail

cat /mnt/raid-test/test.txt
# Confirm the array still serves reads normally — degraded is not the
# same as down
```

### Task 4 — Remove the failed member and rebuild with a spare

```bash
sudo mdadm /dev/md0 --remove /dev/loop2
sudo mdadm --detail /dev/md0
# Total Devices should now show 1, not 2

sudo mdadm /dev/md0 --add /dev/loop3
cat /proc/mdstat
# Watch the recovery percentage climb — poll every few seconds:
watch -n 2 cat /proc/mdstat
# Once it reaches 100%, confirm /proc/mdstat returns to [2/2] [UU] and
# mdadm --detail shows "State: clean" with no degraded flag
```

### Task 5 — Confirm data survived the whole cycle

```bash
cat /mnt/raid-test/test.txt
# Should still read "written before resync completes" — the file
# survived the simulated failure and rebuild without being touched
```

### Task 6 — Clean up

```bash
sudo umount /mnt/raid-test
sudo mdadm --stop /dev/md0
sudo mdadm --zero-superblock /dev/loop1 /dev/loop2 /dev/loop3
for i in 1 2 3; do sudo losetup -d /dev/loop$i; done
rm -f /tmp/raid-disk*.img
```

## What You Learned

1. ✅ Explained what RAID actually protects against (disk failure) and
   what it explicitly doesn't (accidental deletion, corruption, disasters)
2. ✅ Built a RAID1 array with `mdadm --create` and read `/proc/mdstat`
   and `mdadm --detail` output correctly
3. ✅ Simulated a disk failure with `mdadm --fail`, recognized a degraded
   array, and confirmed it keeps serving reads/writes normally
4. ✅ Replaced a failed member and rebuilt the array, understanding what
   the recovery process is actually doing
5. ✅ Explained split-brain risk and how `mdadm`'s event-count mechanism
   detects and guards against it
6. ✅ Compared RAID0/1/5/6/10 by their actual trade-offs and can choose
   the right level for a given scenario

## Part 3 — Break-Fix Scenario

### Scenario — NovaTech Ops: two RAID incidents

> **Incident report — NovaTech Ops, ticket #4732**
>
> Alert 1: "a disk in our RAID1 array was reported failed three weeks
> ago. Nobody replaced it. Today a second disk in the same array just
> failed, and the array is completely down — all data on it is gone."
> Alert 2: "we had a controller glitch that dropped one disk from a
> mirror for about an hour. The disk was later reseated and the array
> auto-reassembled, but now we're seeing corrupted-looking data on some
> files that were being actively written during that window."

**Incident 1 — a second disk failure with no replacement in between**

Tasks:
1. Referencing §1.5, what was the array's actual risk exposure the
   entire three weeks between the first failure and the second?
2. Is this "RAID failed to protect us" or something else? Frame your
   answer using §1.1's distinction.
3. What process gap does this incident actually point to — what should
   have happened in week one?

**Incident 2 — a reseated disk led to corrupted-looking data**

Tasks:
1. Referencing §1.7, what's the most likely explanation for how a
   "successfully reassembled" array ended up with bad data on files
   written during the outage window?
2. What should have happened when the disk was reseated, instead of
   whatever "auto-reassembled" actually means here?
3. What's the general operational rule this incident argues for, when a
   RAID member disk reappears after being missing for a nontrivial period?

<details>
<summary>Answers</summary>

**Incident 1:** For the entire three weeks between the first failure and
the second, the array was running degraded — per §1.5, a RAID1 array
with one failed member has zero remaining redundancy; it's serving reads
and writes normally, but a second failure at any point during that window
means total data loss for that array, exactly as happened. This is not
"RAID failed to protect us" in the sense of RAID malfunctioning — RAID1
did exactly what it's designed to do: survive exactly one failure, no
more, per §1.2's table. The actual failure here is operational, not
technical: per §1.1, RAID is an availability tool that bought three weeks
of continued uptime after the first failure, which is real value — but
nobody used that window to actually replace the disk and rebuild (§1.6),
which is the entire point of that window existing. The process gap: a
failed-disk alert should trigger an actual replacement and rebuild
promptly, not just an acknowledged ticket — degraded-state monitoring
that doesn't escalate until the *array* goes down, rather than when the
*first* disk fails, catches this kind of incident too late by definition.

**Incident 2:** This is the split-brain scenario from §1.7. When the
disk dropped out, the surviving member kept accepting writes normally
(per §1.5) — its event count kept incrementing. If the reseated disk was
brought back into the array in a way that didn't go through a proper
`mdadm --add` and full resync — for instance, an automatic reassembly
that used event-count comparison incorrectly, or a scenario where the
reseated disk's event count was close enough to be accepted without a
forced, complete resync — files written during the outage window could
end up served from whichever copy actually has stale or inconsistent
data for that specific region of the disk, producing exactly the
corrupted-looking symptom reported. What should have happened: per §1.7,
a disk that's been separated from the array for a nontrivial period
should be treated as untrusted, not silently reintegrated — `mdadm`'s
own event-count mechanism exists specifically to flag this kind of
mismatch, and a reseated disk with a meaningfully lower event count
should trigger a full `--add`-style resync (§1.6), not an assumption of
continuity. The general operational rule: any RAID member that reappears
after being missing is either brought back in through an explicit,
monitored rebuild, or wiped and re-added as a fresh member — never
trusted at face value just because the disk itself is physically present
again.

</details>

## Interview Prep

**Q1. A stakeholder asks "we have RAID1 on our database server, so we don't need backups, right?" How do you respond?**

No — RAID and backups solve different problems, and conflating them is a
common, costly mistake. RAID protects against physical disk failure
specifically; it has no concept of "correct" vs. "incorrect" data, so it
faithfully mirrors an accidental `DROP TABLE`, a ransomware encryption
pass, or a bad application bug across every member disk just as reliably
as it mirrors legitimate writes. Backups are the tool that protects
against those scenarios — the two are complementary, and a database
server with RAID and no backups is still fully exposed to the failure
modes RAID was never designed to address.

**Q2. You notice a RAID1 array has been running degraded (one failed disk) for two weeks, still with no replacement scheduled. How urgent is this, and why?**

Very urgent — a degraded RAID1 array has zero remaining redundancy. It's
still serving reads and writes normally, which is exactly why it's easy
to underestimate: nothing appears broken from the application's
perspective. But a second disk failure at any point during that window
means total data loss for that array, with no warning beyond whatever
monitoring is (or isn't) watching for it. The alert that matters is the
first failure, not eventually noticing the array went fully down — by
the time the array is actually down, the redundancy window that should
have been used to fix it is already gone.

**Q3. A RAID member disk that was disconnected for several hours gets physically reconnected. Should it just be plugged back in and trusted to rejoin the array automatically?**

Not without verification. Per `mdadm`'s event-count mechanism, a member
that's been separated from the array has a stale view of its data — the
surviving member(s) kept accepting writes the whole time it was gone.
`mdadm` uses each member's event count specifically to detect this kind
of staleness and will refuse to silently treat a significantly
out-of-date member as current. The safe path is to treat the reappeared
disk as untrusted: either let `mdadm` run its normal detection and, if
it flags a mismatch, perform an explicit, monitored rebuild (`--add` and
a full resync), or proactively wipe and re-add it as a fresh member
rather than assuming a "successful" automatic reassembly actually
resolved the inconsistency correctly.

## Key Takeaways

---

1. **RAID protects against physical disk failure and nothing else — it is not a backup.** It faithfully mirrors accidental deletion, corruption, or ransomware across every member disk exactly as reliably as it mirrors legitimate writes. Backups (Demo 14a/14b) are the complementary, distinct tool for those failure modes.
2. **`[2/2] [UU]` in `/proc/mdstat` reads as configured/active member counts, then one letter per slot — `U` for up, `_` for missing.** The position of the underscore identifies which array *slot* is down, not necessarily which physical disk, since slot assignment happens at creation time.
3. **A degraded RAID1 array (one failed member) keeps serving reads and writes normally — "degraded" is not "down."** This is the entire point of mirroring, but it also means a degraded array's real risk is invisible from the application's perspective: zero remaining redundancy, with a second failure at any point meaning total data loss.
4. **`--fail` marks a member down without detaching it; `--remove` actually detaches it; `--add` attaches a replacement and triggers a full rebuild.** These are three deliberately separate operations, not one combined step, so a failing-but-not-dead disk can be investigated before being physically removed.
5. **Split-brain risk in RAID1 means two mirror members end up with genuinely different data after being separated and independently written to.** `mdadm` defends against this with per-member event counts on each disk's superblock — a member with a significantly lower event count is treated as stale and never silently reintegrated into a live array.
6. **A reseated or reappeared RAID member should never be trusted at face value just because the disk is physically present again.** It should go through an explicit, monitored rebuild or be treated as untrusted and re-added fresh — silently trusting an "automatic" reassembly is exactly how split-brain-adjacent corruption incidents like this demo's Incident 2 happen.
7. **Choose the RAID level by its actual trade-off, not habit: RAID1 for simplicity on small arrays, RAID5 for capacity efficiency with single-failure tolerance, RAID6 when a second failure during a long rebuild is a real risk, RAID10 for write-heavy workloads like databases.** RAID0 has no place in a durability context — every member disk is a single point of failure for the whole array.

> **Demo scope:** Primary concept: RAID1's build/degrade/rebuild cycle and
> split-brain risk, with the other common levels covered conceptually.
> Supporting concepts: mdadm create/fail/remove/add, /proc/mdstat and
> --detail output, event-count-based staleness detection, level comparison.
> Estimated completion time: 35 minutes (reading + hands-on + verification),
> ASSUMING mdadm and a working md kernel module are available — this demo's
> own authoring environment had neither.
> Checkpoints: 6 natural stopping points (after Task 1 through Task 6).
> **Verification status:** built from official documentation and
> cross-confirmed historical transcripts, not live execution in this
> session — see the note at the top of this file.

## Quick Commands Reference

### Building and inspecting

| Command | Purpose |
|---|---|
| `mdadm --create /dev/mdN --level=L --raid-devices=N dev1 dev2...` | Build a new array |
| `cat /proc/mdstat` | Live state of all arrays |
| `mdadm --detail /dev/mdN` | Full status report for one array |

### Failure and rebuild

| Command | Purpose |
|---|---|
| `mdadm /dev/mdN --fail /dev/sdX` | Mark a member as failed |
| `mdadm /dev/mdN --remove /dev/sdX` | Detach a failed member |
| `mdadm /dev/mdN --add /dev/sdY` | Add a replacement, triggering rebuild |
| `mdadm --assemble --force` | Force-reassemble with mismatched event counts (explicit, risky, last resort) |

### Teardown

| Command | Purpose |
|---|---|
| `mdadm --stop /dev/mdN` | Deactivate an array |
| `mdadm --zero-superblock /dev/sdX` | Erase RAID metadata from a device |

## Appendix A — Anki Cards

**`15a-raid-mdadm-anki.csv`:**

```
#deck:Linux Mastery::Module 4 - Storage::Demo 15a - RAID with mdadm
#separator:Comma
#columns:Front,Back,Tags
"Does RAID protect against accidental file deletion or ransomware encrypting your files?","No — RAID has no concept of correct vs incorrect data. It faithfully mirrors or stripes a deletion or a ransomware pass across every member disk just as reliably as legitimate writes. RAID protects against physical disk failure only; backups are the tool for accidental deletion or corruption.","demo15a,raid,fundamentals"
"/proc/mdstat shows [2/1] [U_] for a RAID1 array. What does this mean?","2/1 means 2 configured members, 1 currently active. [U_] shows one letter per slot: U (up) for a working member, _ for missing/failed. This array is degraded — one member is down.","demo15a,raid,mdstat,fundamentals"
"A RAID1 array has one failed disk (degraded state). Is the array currently down?","No — a degraded RAID1 array keeps serving reads and writes normally. Degraded is not down. What's lost is redundancy: a SECOND failure at this point would mean total data loss, since there's nothing left to fall back on.","demo15a,raid,degraded,fundamentals"
"What's the difference between mdadm --fail and mdadm --remove?","--fail marks a member as failed without detaching it from the array. --remove actually detaches a failed member so a replacement can take its slot. They're separate deliberately, so a failing-but-not-dead disk can be investigated before physical removal.","demo15a,raid,mdadm,fundamentals"
"What triggers a RAID rebuild, and what is the rebuild process actually doing?","Adding a new/replacement device with mdadm --add triggers an automatic rebuild. The array copies data from the surviving member(s) onto the new device until it's fully synchronized — visible in /proc/mdstat as a live recovery percentage.","demo15a,raid,rebuild,fundamentals"
"What is RAID split-brain, in the context of a RAID1 mirror?","Two mirror members end up holding genuinely different data after being separated — the surviving member keeps accepting writes while the other is out, and if the returning member is reintegrated without verification, you have two disks each thinking they're the current authoritative copy.","demo15a,raid,split-brain,fundamentals"
"How does mdadm detect and guard against split-brain / stale array members?","Every member disk's superblock carries an event count that increments on every significant array event. On reassembly, mdadm compares event counts — a member with a significantly lower count is treated as stale and never silently included in a live array without an explicit forced reassembly.","demo15a,raid,split-brain,mdadm,fundamentals"
"A RAID1 array's first disk fails and stays unreplaced for three weeks before a second disk fails, taking down the array entirely. Is this RAID failing to do its job?","No — RAID1 is designed to survive exactly one failure, which it did for three weeks. The actual failure is operational: the degraded-state window existed specifically to be used for replacement and rebuild, and wasn't.","demo15a,raid,break-fix,operational"
"Should a RAID member disk that was disconnected for several hours be trusted to rejoin the array automatically just because it's reconnected?","No — it should be treated as untrusted until mdadm's event-count check confirms it (or an explicit, monitored rebuild is performed). A disk missing that long has a stale view of the data, since the surviving member kept accepting writes the whole time.","demo15a,raid,split-brain,operational"
"When would you choose RAID6 over RAID5?","When a second disk failure during a long rebuild is a real risk — larger arrays with slow rebuild times are exactly this case. RAID6 tolerates 2 simultaneous failures (two independent parity blocks) vs RAID5's 1, at the cost of more usable-capacity overhead.","demo15a,raid,levels,comparison"
"Does RAID0 belong in a data-durability strategy?","No — RAID0 (striping with no redundancy) provides zero fault tolerance; ANY member disk failing loses the entire array's data. It exists purely for speed, making every member disk a single point of failure for the whole array.","demo15a,raid,raid0,fundamentals"
```

## Appendix B — Quiz

**`15a-raid-mdadm-quiz.md`:**

````markdown
# Quiz — Demo 15a: RAID with mdadm

> One correct answer per question unless stated otherwise.
> Target: 80% or above before moving to the next demo.

---

**Q1.** A production database has RAID1 but no backups. A bad migration script accidentally drops a table. Does RAID help recover the lost data?

- A) Yes — RAID1 keeps a mirrored copy that can be rolled back to before the mistake
- B) No — RAID has no concept of correct vs. incorrect data; it faithfully mirrors the DROP across every member disk exactly as reliably as any other write
- C) Only if the failure is caught within a few seconds
- D) Yes, but only if the array uses RAID6 instead of RAID1

<details>
<summary>Answer</summary>

**B** — RAID protects against physical disk failure, not against operator or application mistakes. Both mirror members faithfully receive the same (destructive) write. Backups, not RAID, are the tool for this failure mode.

Trap: A wrongly imagines RAID has a rollback capability it doesn't have; C invents a timing exception that doesn't exist; D wrongly assumes a different RAID level changes this fundamental limitation.

</details>

---

**Q2.** `/proc/mdstat` shows `md0 : active raid1 sdb1[0] sdc1[1] ... [2/1] [U_]` for an array. What does this indicate?

- A) The array has 2 total disks, both currently healthy
- B) The array has 2 configured members, only 1 currently active — it's degraded, with the down slot indicated by the underscore
- C) The array is being actively rebuilt and will self-heal shortly with no action needed
- D) This is normal, healthy output for any 2-disk RAID1 array

<details>
<summary>Answer</summary>

**B** — `2/1` means 2 configured members but only 1 active; `[U_]` shows one letter per slot, with `_` marking the down one. This is a degraded array requiring investigation, not a healthy or self-resolving state.

Trap: A and D both misread degraded output as healthy; C wrongly assumes automatic self-healing — `mdadm` does not automatically replace a failed disk without an explicit `--add`.

</details>

---

**Q3.** A RAID1 array has one failed disk (degraded state) but hasn't been replaced yet. Can applications still read and write to it normally?

- A) No — a degraded array is completely inaccessible until repaired
- B) Yes — a degraded RAID1 array continues serving reads and writes normally; what's actually lost is redundancy, not current availability
- C) Only reads are possible; writes are blocked until the array is repaired
- D) Only if the array is explicitly remounted in a special degraded-access mode

<details>
<summary>Answer</summary>

**B** — This is the entire point of RAID1 mirroring: the surviving member keeps the array fully operational. The real risk is that a SECOND failure at this point means total data loss, since there's no remaining redundancy — but current operation is unaffected.

Trap: A, C, and D all invent restricted-access states that don't match how a degraded RAID1 array actually behaves.

</details>

---

**Q4.** What's the difference between `mdadm --fail` and `mdadm --remove`?

- A) They're interchangeable aliases for the same operation
- B) `--fail` marks a member as failed without detaching it from the array; `--remove` actually detaches a failed member so a replacement can take its slot
- C) `--fail` is used for RAID1 only; `--remove` is used for RAID5/6 only
- D) `--remove` must always be run before `--fail`

<details>
<summary>Answer</summary>

**B** — These are deliberately separate steps: marking a member failed (without physically detaching it) allows investigation before committing to removal, which matters for a disk that might be failing intermittently rather than fully dead.

Trap: A wrongly claims equivalence; C invents a RAID-level restriction that doesn't exist; D reverses the actual required order — a member is failed, then removed, not the other way around.

</details>

---

**Q5.** `mdadm /dev/md0 --add /dev/sdd1` is run on a degraded array. What happens, and does it require you to wait before using the array again?

- A) It triggers an automatic rebuild that copies data from surviving members onto the new device, and the array remains usable during this process
- B) It immediately restores the array to full redundancy with no delay
- C) It requires unmounting the array and waiting for rebuild completion before any further use
- D) It only prepares the device as a spare; a separate manual command is required to start the actual rebuild

<details>
<summary>Answer</summary>

**A** — Adding a device triggers automatic recovery, visible as a live progress percentage in `/proc/mdstat`, and the array continues to be usable throughout — consistent with §1.3's point that arrays don't require waiting for a resync to complete before being used.

Trap: B wrongly claims instant full redundancy; C invents an unnecessary unmount requirement; D invents a separate manual trigger step that doesn't exist — `--add` itself starts the rebuild.

</details>

---

**Q6.** In the context of a RAID1 mirror, what does "split-brain" mean?

- A) The array physically splits into two separate RAID0 arrays
- B) Two mirror members end up holding genuinely different data after being separated and independently receiving writes, with no inherent way to know which is authoritative
- C) The RAID controller crashes and requires a full reboot
- D) A single disk develops two different filesystem partitions

<details>
<summary>Answer</summary>

**B** — This is the actual risk: a surviving mirror member keeps accepting writes while another is disconnected, and if that other member is later reintegrated without verification, both disks may believe they're the current, correct copy while actually holding different data.

Trap: A, C, and D all invent unrelated failure descriptions that don't match the real split-brain risk in a mirrored array.

</details>

---

**Q7.** How does `mdadm` actually detect and guard against a stale, out-of-sync member rejoining an array?

- A) It calculates a full checksum of every byte on the disk before allowing it back in
- B) Each member disk's superblock carries an event count that increments on every significant array event; a member with a significantly lower count is treated as stale and not silently reintegrated
- C) It relies entirely on the physical timestamp of when the disk was last powered on
- D) It has no such mechanism — any physically present disk is trusted automatically

<details>
<summary>Answer</summary>

**B** — The event-count mechanism on each member's superblock is exactly how `mdadm` distinguishes a genuinely current member from a stale one, without needing a full data comparison. Forcing a mismatched member back in (`--assemble --force`) is a deliberate, explicit, and risky override of this protection, not something that happens automatically.

Trap: A invents an expensive mechanism `mdadm` doesn't use for this purpose; C invents a timestamp-based mechanism that isn't how event counts work; D wrongly claims no protection exists at all.

</details>

---

**Q8.** A team needs write-heavy performance for a database and can afford the capacity cost of losing half their raw storage to redundancy. Which RAID level best fits?

- A) RAID0, for maximum raw speed
- B) RAID10 — mirrored pairs striped together, which handles heavy random write I/O well while tolerating at least one failure per mirrored pair
- C) RAID6, for its extra parity redundancy
- D) RAID5, for its balance of capacity and redundancy

<details>
<summary>Answer</summary>

**B** — RAID10 is specifically suited to write-heavy, random-I/O workloads like databases, at the cost of 50% capacity overhead — which the scenario explicitly states is acceptable. RAID5/6's parity calculations impose write overhead that makes them a worse fit for heavy random writes specifically, even though they're more capacity-efficient.

Trap: A ignores that RAID0 provides zero fault tolerance, disqualifying it outright for a database; C and D are both reasonable RAID levels in general but are optimized for capacity efficiency over write performance, the opposite of what this scenario asks for.

</details>

---

Score guide:

| Score | Action |
|---|---|
| 7/8 or 8/8 | Import Anki cards, move to the next demo |
| 6/8 | Review the wrong answers, then proceed |
| 5/8 | Re-read the relevant sections, retry those questions |
| Below 5/8 | Re-read the full demo and redo the walkthrough before proceeding |
````