# Demo 14 — Storage Fundamentals

## Overview

Every server eventually needs more than its default disk layout — a data
volume, a mount point for shared storage, swap space sized correctly for its
workload. This demo covers the full path from raw block device to usable,
persistent storage: identifying disks, partitioning them, formatting them
with a filesystem, mounting them (both temporarily and permanently), and
reading disk usage correctly. Demo 15 builds on this with LVM, remote
filesystems, and storage troubleshooting — this demo is the "provision and
mount" half.

## Demo Objectives

1. Identify block devices and their partition layout with `lsblk`
2. Partition a disk with `fdisk` and `parted`, and explain when to use each
3. Create a filesystem on a partition with `mkfs`
4. Mount a filesystem temporarily and persistently via `/etc/fstab`
5. Read `df` and `du` output correctly, including where they can disagree
6. Configure and activate swap space correctly

## Directory Structure

```
14-storage-fundamentals/
└── README.md   # This file — full demo content, including Anki (Appendix A)
                 # and Quiz (Appendix B), extracted later via script.
```

## Environments

| Task | Ubuntu 24.04 | Rocky Linux 9.8 | EC2 Amazon Linux |
|---|---|---|---|
| All lab tasks | ✅ (use a lab/scratch disk, never the root disk) | ✅ (same caution) | Not required |

## Recall Check — 13-network-topology-traffic-distribution

Answer these from memory before continuing — no looking back at Demo 13.

**Q1.** A bond is configured in `active-backup` mode. A team expects doubled
throughput. Is this expectation correct? Why or why not?

**Q2.** An `ip_hash`-configured load balancer sends all traffic to one
backend, and every client request originates from the same NAT gateway IP.
Is this a malfunction?

**Q3.** Two overlapping static routes exist — a `/16` and a more specific
`/24` — for an address that matches both. Which route does the kernel use,
and why?

<details>
<summary>Answers</summary>

**A1.** No — `active-backup` provides redundancy only (one active link, one
standby), not throughput improvement. Only LACP (`mode=4`), with matching
switch-side configuration, actually improves throughput.

**A2.** No — `ip_hash` is working correctly. With effectively one source IP
(the shared NAT gateway), it has only one input to hash against and
consistently routes to the same backend as a direct, expected consequence.

**A3.** The `/24` route — the kernel always uses longest-prefix match,
picking the most specific matching route regardless of which was added
first.

</details>

## Part 1 — Concepts

### 1.1 lsblk — the map before you touch anything

```bash
lsblk -f
```

```
NAME   FSTYPE   LABEL   UUID                                 MOUNTPOINT
sda
├─sda1 ext4             a1b2c3d4-...                          /
└─sda2 swap             e5f6g7h8-...                          [SWAP]
sdb
```

`-f` adds filesystem type, label, and UUID — the information you need
*before* partitioning or formatting anything. `sdb` here has no partitions
and no filesystem — an unformatted disk, the starting point for this demo's
lab. **Always run `lsblk` before any partitioning command** — it's the
single cheapest way to avoid partitioning the wrong disk, which on a real
server can mean destroying data on the root filesystem by mistake.

### 1.2 fdisk and parted — partitioning, and when to use each

```bash
sudo fdisk -l /dev/sdb
```

`fdisk` is interactive, MBR- and GPT-capable (modern versions), and the more
commonly reached-for tool for straightforward partitioning:

```bash
sudo fdisk /dev/sdb
# n (new partition), p (primary), accept defaults for partition number and
# first sector, specify size (e.g. +10G), w (write and exit)
```

`parted` is scriptable (non-interactive one-liners) and has better native
support for GPT and partitions larger than 2TB, which is where `fdisk`'s
older MBR-era assumptions can become limiting:

```bash
sudo parted /dev/sdb mklabel gpt
sudo parted /dev/sdb mkpart primary ext4 0% 100%
```

**Practical rule of thumb:** `fdisk` for quick interactive work on a single
disk under 2TB; `parted` when scripting partition creation, or when GPT and
larger disks are involved. Both ultimately write a partition table — neither
creates a filesystem, which is a separate step (§1.3).

### 1.3 mkfs — creating a filesystem

A partition with no filesystem is unusable — `mkfs` writes the filesystem
structure onto it:

```bash
sudo mkfs.ext4 /dev/sdb1
# or:
sudo mkfs.xfs /dev/sdb1
```

`ext4` is the traditional, well-understood general-purpose default on both
distros. `xfs` (Rocky9's actual root filesystem default) handles very large
files and high-concurrency workloads better, at the cost of being somewhat
harder to shrink after creation (`ext4` supports online shrink in more
scenarios than `xfs`, which generally can't be shrunk at all without
recreating it) — a real factor when choosing a filesystem for a volume that
might need resizing down the line, not just up.

```bash
sudo mkfs.ext4 -L data-volume /dev/sdb1
# -L: label the filesystem — makes it identifiable in lsblk -f and usable in
# fstab as LABEL=data-volume instead of a raw device path.
```

### 1.4 mount, umount, and /etc/fstab — temporary vs. persistent

```bash
sudo mkdir -p /mnt/data
sudo mount /dev/sdb1 /mnt/data
mount | grep sdb1
```

**This mount is temporary** — exactly the same persistence trap as `ip addr
add` (Demo 11) and firewalld rules without `--permanent` (Demo 12). It
disappears on reboot unless written into `/etc/fstab`:

```bash
sudo blkid /dev/sdb1
# UUID="a1b2c3d4-..." TYPE="ext4" LABEL="data-volume"

echo 'UUID=a1b2c3d4-...  /mnt/data  ext4  defaults  0  2' | sudo tee -a /etc/fstab
```

**Using UUID instead of the raw device path (`/dev/sdb1`) is deliberate, not
stylistic.** Device names like `/dev/sdb` aren't guaranteed stable across
reboots — disk enumeration order can shift, especially with multiple disks
or after hardware changes, silently mounting the wrong device at the wrong
path. UUIDs are generated per-filesystem and don't change.

```bash
sudo mount -a
# Test the fstab entry without rebooting — mounts everything listed that
# isn't already mounted, and will surface a syntax error immediately rather
# than waiting to discover it at the next actual reboot, when a bad fstab
# entry can leave a server failing to boot cleanly.
```

The two trailing fstab fields matter: the dump field (`0`/`1`, legacy backup
tool flag, almost always `0` today) and the fsck pass number (`0` = never
check, `1` = check first/root filesystem, `2` = check after pass 1) — `2` is
the standard choice for a non-root data volume.

### 1.5 df and du — reading disk usage, and where they disagree

```bash
df -h
```

```
Filesystem      Size  Used Avail Use% Mounted on
/dev/sda1        50G   12G   36G  25% /
/dev/sdb1        20G  5.0G   14G  27% /mnt/data
```

`df` reports filesystem-level usage — fast, because it reads metadata the
filesystem already tracks, not actual file contents.

```bash
du -sh /mnt/data/*
```

`du` walks the directory tree and sums actual file sizes — slower, but
answers "what's actually taking up space and where," which `df` can't.

**These two commands can legitimately disagree**, and the most common cause
is a deleted-but-still-open file (the same mechanism Demo 02 covered with
`lsof`): `df` still counts that space as used (the filesystem hasn't
reclaimed it — a process still holds the file open), while `du`, which only
walks what's currently visible in the directory tree, doesn't see the
deleted file at all and won't count it. A disk showing "full" per `df`, with
`du` unable to account for where that space went, is close to always this
exact situation — resolved by finding and restarting (or otherwise closing)
whatever process is holding the deleted file open, at which point the space
is actually freed.

### 1.6 Swap — configuration, not just "extra RAM"

```bash
sudo fallocate -l 2G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
swapon --show
```

Persisting it, same fstab pattern as any mount:

```bash
echo '/swapfile  none  swap  sw  0  0' | sudo tee -a /etc/fstab
```

Swap isn't simply "extra RAM" — accessing swapped data means disk I/O
instead of memory access, orders of magnitude slower. Demo 09's `vmstat`
`si`/`so` columns are exactly how you'd notice a system actually relying on
swap under load, which is a performance problem to investigate, not a
capacity feature to lean on. Swap's practical value is providing a buffer
against sudden memory spikes and enabling clean handling of out-of-memory
conditions (via the kernel's OOM killer having somewhere to page from)
rather than serving as routine extra capacity.

## Part 2 — Hands-On Lab

### Commands used in this demo

| Command | Flag meaning | Purpose |
|---|---|---|
| `lsblk -f` | `-f`: filesystem info | Map block devices before touching anything |
| `fdisk /dev/sdX` | — | Interactive partitioning |
| `parted /dev/sdX mkpart ...` | — | Scriptable partitioning, better GPT/large-disk support |
| `mkfs.ext4` / `mkfs.xfs` | — | Create a filesystem on a partition |
| `blkid` | — | Show UUID/label/type for fstab entries |
| `mount -a` | `-a`: all fstab entries | Test fstab without rebooting |
| `df -h` | `-h`: human-readable | Filesystem-level usage |
| `du -sh PATH` | `-s`: summary, `-h`: human-readable | Actual file-tree usage |
| `mkswap` / `swapon` | — | Create and activate swap |

### Task 1 — Identify and inspect a scratch disk

```bash
lsblk -f
# Identify an unformatted scratch/lab disk (never the root disk) for the
# rest of this demo — confirm it has no existing filesystem or mountpoint.
```

### Task 2 — Partition with fdisk

```bash
sudo fdisk /dev/sdb
# n, p, 1, <enter> (default first sector), +5G, w
lsblk -f /dev/sdb
```

### Task 3 — Create a filesystem and label it

```bash
sudo mkfs.ext4 -L lab-data /dev/sdb1
lsblk -f /dev/sdb1
# Confirm the FSTYPE and LABEL columns now show ext4 / lab-data
```

### Task 4 — Mount temporarily, then persist via fstab

```bash
sudo mkdir -p /mnt/labdata
sudo mount /dev/sdb1 /mnt/labdata
df -h | grep labdata

sudo blkid /dev/sdb1
echo 'LABEL=lab-data  /mnt/labdata  ext4  defaults  0  2' | sudo tee -a /etc/fstab
sudo umount /mnt/labdata
sudo mount -a
df -h | grep labdata
# Confirm mount -a brought it back using the fstab entry, not the earlier
# manual mount command.
```

### Task 5 — df vs. du disagreement, demonstrated

```bash
dd if=/dev/zero of=/mnt/labdata/bigfile bs=1M count=500
df -h /mnt/labdata
du -sh /mnt/labdata

# Now delete it while a process still holds it open:
tail -f /mnt/labdata/bigfile > /dev/null &
TAIL_PID=$!
rm /mnt/labdata/bigfile
df -h /mnt/labdata    # still shows the space as used
du -sh /mnt/labdata    # no longer shows bigfile at all — mismatch
kill $TAIL_PID
df -h /mnt/labdata    # space now actually freed
```

### Task 6 — Configure swap

```bash
sudo fallocate -l 512M /swapfile-lab
sudo chmod 600 /swapfile-lab
sudo mkswap /swapfile-lab
sudo swapon /swapfile-lab
swapon --show
free -h
# Clean up (lab only — don't leave a stray swapfile on a real system):
sudo swapoff /swapfile-lab
sudo rm /swapfile-lab
```

## What You Learned

1. ✅ Identified block devices and their partition layout with `lsblk`
2. ✅ Partitioned a disk with `fdisk` and `parted`, and when to use each
3. ✅ Created a filesystem on a partition with `mkfs`
4. ✅ Mounted a filesystem temporarily and persistently via `/etc/fstab`
5. ✅ Read `df` and `du` output correctly, including where they can disagree
6. ✅ Configured and activated swap space correctly

## Part 3 — Break-Fix Scenario

### Scenario — NovaTech Ops: two storage incidents

> **Incident report — NovaTech Ops, ticket #4572**
>
> Alert 1: "`/var/log` shows 95% full per `df`, but `du -sh /var/log`
> only accounts for 20GB of a 40GB partition — where's the rest?" Alert 2: a
> server rebooted after maintenance and a data volume that was definitely
> mounted before the reboot is now missing.

**Incident 1 — df and du disagree by 20GB**

```bash
$ df -h /var/log
Filesystem      Size  Used Avail Use% Mounted on
/dev/sdb1        40G   38G  2.0G  95% /var/log
$ du -sh /var/log
20G     /var/log
```

Tasks:
1. Given §1.5, what's the most likely explanation for `df` and `du`
   disagreeing by roughly 20GB?
2. What command from an earlier demo would help you find the specific
   process responsible, and what would you actually run?
3. What's the fix, and why does simply deleting more files in `/var/log`
   not help if this diagnosis is correct?

**Incident 2 — mounted volume missing after reboot**

```bash
$ df -h | grep data
$ # no output — the volume isn't mounted at all
$ cat /etc/fstab | grep data
$ # no output either — no entry exists
$ mount | grep sdb1
$ # also no output
```

Tasks:
1. Given that `/etc/fstab` has no entry for this volume at all, what does
   that tell you about how it was mounted before the reboot?
2. What's the fix, referencing §1.4's UUID-vs-device-path guidance
   specifically?
3. Why is `mount -a` (rather than just rebooting again) the right way to
   verify the fix before trusting it?

<details>
<summary>Answers</summary>

**Incident 1:** The most likely explanation is a deleted-but-still-open file
— `df` reports filesystem-level used space, which still counts a deleted
file's blocks as long as some process holds it open, while `du` only walks
what's currently visible in the directory tree and never sees a deleted
file at all. The tool to find the responsible process is `lsof` (Demo 12,
built on the pattern from Demo 02): `sudo lsof +L1 /var/log` (or `lsof
/var/log | grep deleted`) to find open file handles with a link count of
zero — deleted files still held open. The fix is restarting or otherwise
closing whatever process is holding the deleted file(s) open (commonly a
log-writing daemon that had a log file rotated/deleted out from under it
without being told to reopen its handle) — at which point the kernel
actually releases the space. Deleting more files in `/var/log` doesn't help
because the discrepancy isn't about files that still exist in the tree; it's
about space `df` correctly counts as used that `du` structurally cannot see.

**Incident 2:** No `/etc/fstab` entry means this volume was mounted manually
with a bare `mount` command at some point — exactly the persistence trap
§1.4 describes, the same pattern as `ip addr add` (Demo 11) and firewalld
rules without `--permanent` (Demo 12): it worked until the next reboot, then
vanished, because nothing told the system to remount it automatically. Fix:
identify the volume's UUID with `blkid`, add a proper fstab entry using that
UUID (not the device path, since `/dev/sdb` naming isn't guaranteed stable
across reboots — using a UUID avoids ever landing in this exact situation
again for a different reason), then mount it. `mount -a` is the right
verification step rather than rebooting again because it applies the fstab
entry immediately and will surface a syntax error in the entry right away —
rebooting to test is slower and, if the entry is wrong, risks a boot that
hangs waiting on a bad mount, which is a worse way to discover the mistake
than `mount -a` failing cleanly in a live session.

</details>

## Interview Prep

**Q1. `df` and `du` report significantly different used space for the same partition. What's the most common cause, and how would you confirm it?**
A deleted file still held open by a running process — `df` counts it as used
space (the filesystem hasn't reclaimed the blocks) while `du` only sees
files currently visible in the directory tree and can't account for a
deleted one at all. Confirm with `lsof`, looking for open file handles
pointing at deleted files (`lsof | grep deleted`, or `lsof +L1` on the
mount) — the fix is closing or restarting whatever process is holding the
handle open.

**Q2. Why is using a UUID in `/etc/fstab` preferred over a raw device path like `/dev/sdb1`?**
Device names aren't guaranteed stable across reboots — disk enumeration
order can shift, especially with multiple disks attached or after hardware
changes, which can silently mount the wrong device at a given path. A
filesystem's UUID is generated once and doesn't change, so an fstab entry
referencing it correctly identifies the intended filesystem regardless of
what device name the kernel happens to assign it this boot.

**Q3. When would you reach for `parted` instead of `fdisk`?**
When scripting partition creation non-interactively, or when working with
GPT partition tables and disks larger than 2TB, where `fdisk`'s older
MBR-era assumptions become limiting. For quick interactive partitioning of a
single disk under 2TB, `fdisk` remains the more commonly reached-for tool.

**Q4. Why would you choose `ext4` over `xfs` for a volume you expect might need to shrink later?**
`ext4` supports online shrinking in more scenarios; `xfs` generally cannot
be shrunk at all without recreating the filesystem entirely. `xfs` is often
the better default for very large files or high-concurrency workloads, but
that resize limitation is a real, practical factor to weigh against those
benefits specifically when a volume's eventual size is expected to
decrease, not just grow.

## Key Takeaways

1. **Always `lsblk` before partitioning anything.** It's the cheapest way to
   confirm you're about to operate on the intended disk, not the root
   filesystem — a mistake here can mean destroyed data on a live system.
2. **A mount without an `/etc/fstab` entry is temporary, same as `ip addr
   add` or a firewalld rule without `--permanent`.** It vanishes on reboot
   unless persisted — verify persistence with `mount -a`, not by trusting
   the mount worked once.
3. **Use UUID, not device path, in `/etc/fstab`.** Device names like
   `/dev/sdb` aren't guaranteed stable across reboots; UUIDs are generated
   per-filesystem and don't change.
4. **`df` and `du` disagreeing almost always means a deleted file is still
   held open by a process.** `df` counts it as used (filesystem hasn't
   reclaimed it); `du` can't see it at all since it only walks the visible
   directory tree — `lsof` finds the responsible process.
5. **`fdisk` for quick interactive work under 2TB; `parted` for scripting or
   GPT/large-disk scenarios.** Neither creates a filesystem — `mkfs` is
   always a separate, required step after partitioning.
6. **Swap is a safety buffer, not routine extra RAM.** Active swap usage
   (`vmstat`'s `si`/`so`, from Demo 09) means disk I/O standing in for
   memory access — a performance problem to investigate, not a capacity
   feature to lean on.

> **Demo scope:** Primary concept: provisioning and mounting storage.
> Supporting concepts: lsblk, fdisk/parted partitioning, mkfs, mount/umount/
> fstab persistence, df vs. du, swap configuration.
> Estimated completion time: 38 minutes (reading + hands-on + verification).
> Checkpoints: 6 natural stopping points (after Task 1 through Task 6).

## Quick Commands Reference

### Identification

| Command | Purpose |
|---|---|
| `lsblk -f` | Block devices with filesystem info — run before any partitioning |
| `blkid /dev/sdX` | UUID/label/type for a specific device |

### Partitioning and filesystems

| Command | Purpose |
|---|---|
| `fdisk /dev/sdX` | Interactive partitioning, MBR/GPT, under 2TB |
| `parted /dev/sdX mkpart ...` | Scriptable partitioning, GPT/large-disk friendly |
| `mkfs.ext4 -L LABEL /dev/sdX1` | Create ext4 filesystem with a label |
| `mkfs.xfs /dev/sdX1` | Create xfs filesystem |

### Mounting

| Command | Purpose |
|---|---|
| `mount /dev/sdX1 /mnt/path` | Temporary mount |
| `mount -a` | Apply all `/etc/fstab` entries — test without rebooting |
| `umount /mnt/path` | Unmount |

### Usage

| Command | Purpose |
|---|---|
| `df -h` | Filesystem-level usage |
| `du -sh PATH` | Actual file-tree usage |
| `lsof \| grep deleted` | Find deleted-but-open files explaining a df/du mismatch |

### Swap

| Command | Purpose |
|---|---|
| `fallocate -l SIZE /swapfile && chmod 600` | Create a swapfile |
| `mkswap /swapfile` | Format as swap |
| `swapon /swapfile` / `swapon --show` | Activate / verify |

## Appendix A — Anki Cards

**Note:** extracted into a standalone `.csv` by the series extraction script.

```
#deck:Linux Mastery::Module 3 - Networking & Storage::14-storage-fundamentals
#separator:Comma
#columns:Front,Back,Tags
"df shows a partition at 95% used, but du -sh on the same path only accounts for half that space. What's the most likely cause?","A deleted file still held open by a running process. df counts it as used (filesystem hasn't reclaimed the blocks); du only sees files currently visible in the directory tree and can't account for a deleted one at all.","demo14,df,du,troubleshooting"
"Why is UUID preferred over a raw device path like /dev/sdb1 in /etc/fstab?","Device names aren't guaranteed stable across reboots — disk enumeration order can shift. A filesystem's UUID is generated once and doesn't change, avoiding a silent wrong-device mount.","demo14,fstab,uuid"
"A filesystem is mounted with a plain mount command but has no /etc/fstab entry. What happens after a reboot?","It disappears — a mount without a corresponding fstab entry is temporary, the same persistence trap as ip addr add (Demo 11) or a firewalld rule without --permanent (Demo 12).","demo14,mount,persistence,cross-demo"
"What does mount -a do, and why run it instead of just rebooting to test a new fstab entry?","It applies every entry in /etc/fstab immediately, surfacing a syntax error right away in a live session. Rebooting to test is slower and, if the entry is wrong, risks a boot that hangs waiting on a bad mount.","demo14,fstab,mount-a"
"When would you choose parted over fdisk for partitioning?","When scripting partition creation non-interactively, or when working with GPT and disks larger than 2TB, where fdisk's older MBR-era assumptions become limiting.","demo14,fdisk,parted,comparison"
"Why might ext4 be preferred over xfs for a volume expected to need shrinking later?","ext4 supports online shrinking in more scenarios; xfs generally cannot be shrunk at all without recreating the filesystem entirely.","demo14,ext4,xfs,comparison"
"What's the correct tool to find which process is responsible for a df/du space discrepancy caused by a deleted-but-open file?","lsof — looking for open file handles pointing at deleted files (e.g. lsof | grep deleted).","demo14,lsof,df,du,cross-demo"
"Is swap best thought of as extra RAM capacity for routine use?","No — accessing swapped data means disk I/O instead of memory access, orders of magnitude slower. Swap's real value is as a buffer against sudden memory spikes and enabling clean OOM handling, not routine extra capacity.","demo14,swap,fundamentals"
"Which vmstat columns (from Demo 09) directly indicate a system is actively relying on swap under load?","si and so (swap in/swap out) — sustained non-zero values mean the kernel is genuinely moving pages to/from disk, a performance problem to investigate.","demo14,swap,vmstat,cross-demo"
"What should you always run before partitioning any disk, and why?","lsblk (ideally lsblk -f) — it's the cheapest way to confirm you're about to operate on the intended disk and not accidentally the root filesystem, where a mistake can mean destroyed data.","demo14,lsblk,safety"
```

## Appendix B — Quiz

**Note:** extracted into a standalone `.md` by the series extraction script.

> One correct answer per question unless stated otherwise.
> Target: 80% or above before moving to Demo 15.

**Q1. `df` reports a partition at 95% used; `du -sh` on the same path accounts for only half that space. What's the most likely explanation?**

- A) `du` has a bug and undercounts large files
- B) A deleted file is still held open by a running process, which `df` counts but `du` cannot see
- C) The filesystem is corrupted
- D) `df` and `du` measure different partitions by default

<details>
<summary>Answer</summary>

**B** — This is the classic df/du mismatch: `df` reflects filesystem-level accounting (still counting an open-but-deleted file's blocks), while `du` only walks the currently visible directory tree. Trap: A, C, and D all invent explanations unrelated to the actual, well-documented cause.

</details>

---

**Q2. Why is using a UUID preferred over a device path like `/dev/sdb1` in `/etc/fstab`?**

- A) UUIDs are shorter to type
- B) Device names aren't guaranteed stable across reboots; UUIDs are generated per-filesystem and don't change
- C) `/etc/fstab` doesn't accept device paths at all
- D) UUIDs are required for `ext4` but not other filesystems

<details>
<summary>Answer</summary>

**B** — Disk enumeration order can shift between boots, especially with multiple disks; UUID avoids silently mounting the wrong device. Trap: A is a non-functional claim; C is factually wrong — fstab does accept device paths, just less reliably; D invents a filesystem-specific requirement that doesn't exist.

</details>

---

**Q3. A filesystem was mounted with a plain `mount` command and has no `/etc/fstab` entry. What happens after a reboot?**

- A) It remains mounted, since the kernel remembers recent mounts
- B) It disappears — this is the same persistence trap as `ip addr add` without persistent network config
- C) It gets automatically added to `/etc/fstab`
- D) The system will not boot without an fstab entry for every mounted filesystem

<details>
<summary>Answer</summary>

**B** — A bare `mount` command is runtime-only, exactly like `ip addr add` (Demo 11) or a firewalld rule missing `--permanent` (Demo 12) — it doesn't survive reboot without a corresponding persistent configuration. Trap: A and C both invent auto-persistence behavior that doesn't exist; D overstates the consequence.

</details>

---

**Q4. Why run `mount -a` after adding a new `/etc/fstab` entry, rather than just rebooting to test it?**

- A) `mount -a` is required before any fstab entry takes effect at all
- B) It applies the entry immediately and surfaces syntax errors right away, in a recoverable live session
- C) Rebooting deletes fstab entries that haven't been tested with `mount -a` first
- D) `mount -a` is faster than a reboot, which is the only reason to prefer it

<details>
<summary>Answer</summary>

**B** — Testing with `mount -a` catches a bad entry immediately in a session you can still fix; a bad entry discovered only at boot risks a hung or degraded boot. Trap: A overstates a requirement; C invents a deletion behavior that doesn't exist; D understates the real reason (speed isn't the main point — safety of discovery is).

</details>

---

**Q5. When would `parted` be the better choice over `fdisk`?**

- A) Never — `fdisk` is strictly superior in every case
- B) When scripting partition creation, or working with GPT and disks larger than 2TB
- C) Only when working exclusively with USB drives
- D) `parted` is required for `ext4`, `fdisk` for `xfs`

<details>
<summary>Answer</summary>

**B** — `parted`'s scriptability and better GPT/large-disk support are its practical advantages over `fdisk`'s more interactive, MBR-era-oriented design. Trap: A overstates fdisk's universality; C and D invent constraints that don't exist.

</details>

---

**Q6. Why might `ext4` be preferred over `xfs` for a volume that might need to shrink later?**

- A) `xfs` cannot be created smaller than 1TB
- B) `ext4` supports online shrinking in more scenarios; `xfs` generally cannot be shrunk without recreating it
- C) `ext4` is faster for all workloads
- D) `xfs` doesn't support labels

<details>
<summary>Answer</summary>

**B** — This is the specific, real tradeoff between the two filesystems relevant to resize planning. Trap: A, C, and D all invent unrelated or false constraints.

</details>

---

**Q7. What tool would you use to find the specific process responsible for a df/du space discrepancy caused by a deleted-but-open file?**

- A) `du -a`
- B) `lsof`
- C) `mkfs`
- D) `blkid`

<details>
<summary>Answer</summary>

**B** — `lsof` reveals open file handles, including ones pointing at deleted files, directly identifying the responsible process. Trap: A is still `du`, which structurally cannot see deleted files regardless of flags; C and D are unrelated tools (filesystem creation and UUID/label lookup, respectively).

</details>

---

**Q8. Is swap best understood as routine extra RAM capacity?**

- A) Yes — swap and RAM are functionally interchangeable
- B) No — swapped data access means disk I/O instead of memory access, orders of magnitude slower; swap is a safety buffer, not routine capacity
- C) Yes, as long as the swap file is large enough
- D) No, because modern Linux systems disable swap by default

<details>
<summary>Answer</summary>

**B** — This is the core distinction from §1.6: swap exists as a buffer against spikes and for OOM handling, not as capacity to lean on routinely, given the performance cost of disk-speed access replacing memory-speed access. Trap: A and C both wrongly treat swap as equivalent to RAM; D is factually inaccurate as a blanket claim.

</details>