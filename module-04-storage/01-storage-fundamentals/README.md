# Demo M04-01 — Storage Fundamentals

## Overview

Every server eventually needs more than its default disk layout — a data
volume, a mount point for shared storage, swap space sized correctly for its
workload. This demo covers the full path from raw block device to usable,
persistent storage: identifying disks, partitioning them, formatting them
with a filesystem, mounting them (both temporarily and permanently), and
reading disk usage correctly. Demo M04-04 builds on this with LVM, remote
filesystems, and storage troubleshooting — this demo is the "provision and
mount" half.

## Demo Objectives

1. Identify block devices and their partition layout with `lsblk`
2. Partition a disk with `fdisk` and `parted`, and explain when to use each
3. Create a filesystem on a partition with `mkfs`
4. Mount a filesystem temporarily and persistently via `/etc/fstab`
5. Read `df` and `du` output correctly, including where they can disagree
6. Configure and activate swap space correctly, including tuning how
   aggressively it's used

## Directory Structure

```
01-storage-fundamentals/
├── README.md                             # this file
├── m04-01-storage-fundamentals-anki.csv  # Anki flashcard deck
└── m04-01-storage-fundamentals-quiz.md   # standalone quiz
```

## Environments

| Task | Ubuntu 24.04 | Rocky Linux 9.8 | EC2 Amazon Linux |
|---|---|---|---|
| All lab tasks | ✅ (use a lab/scratch disk, never the root disk) | ✅ (same caution) | Not required |

## Recall Check — 04-topology-traffic-distribution

Answer these from memory before continuing — no looking back at Demo M03-04.

1. A bond is configured in `active-backup` mode. A team expects doubled
   throughput. Is this expectation correct? Why or why not?

2. An `ip_hash`-configured load balancer sends all traffic to one
   backend, and every client request originates from the same NAT gateway IP.
   Is this a malfunction?

3. Two overlapping static routes exist — a `/16` and a more specific
   `/24` — for an address that matches both. Which route does the kernel use,
   and why?

<details>
<summary>Answers</summary>

1. No — `active-backup` provides redundancy only (one active link, one
   standby), not throughput improvement. Only LACP (`mode=4`), with matching
   switch-side configuration, actually improves throughput.

2. No — `ip_hash` is working correctly. With effectively one source IP
   (the shared NAT gateway), it has only one input to hash against and
   consistently routes to the same backend as a direct, expected consequence.

3. The `/24` route — the kernel always uses longest-prefix match,
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

**A precision worth having:** the `ext4` in `parted ... mkpart primary ext4
0% 100%` is not `parted` secretly formatting the partition — it sets a
filesystem-type *hint* in the partition table metadata, informing tools
that read the partition table what's expected to be there. The actual
filesystem is still created by `mkfs` in §1.3, exactly as with `fdisk`;
`parted` never creates one on its own.

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
add` (Demo M03-01) and firewalld rules without `--permanent` (Demo M03-02). It
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
is a deleted-but-still-open file (the same mechanism Demo M01-08 covered with
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

**A caveat worth knowing before you rely on `fallocate` here:** on some
filesystems (Btrfs in particular, and occasionally others depending on
kernel/filesystem version) `fallocate` can produce a file with holes that
`mkswap` refuses to use as swap. On ext4 and xfs this is generally reliable,
but if `mkswap` complains or swap activation fails unexpectedly, fall back
to the slower-but-always-safe method:

```bash
sudo dd if=/dev/zero of=/swapfile bs=1M count=2048   # 2G, filled with real zero blocks
```

Persisting the swapfile itself, same fstab pattern as any mount:

```bash
echo '/swapfile  none  swap  sw  0  0' | sudo tee -a /etc/fstab
```

Swap isn't simply "extra RAM" — accessing swapped data means disk I/O
instead of memory access, orders of magnitude slower. Demo M02-07's `vmstat`
`si`/`so` columns are exactly how you'd notice a system actually relying on
swap under load, which is a performance problem to investigate, not a
capacity feature to lean on. Swap's practical value is providing a buffer
against sudden memory spikes and enabling clean handling of out-of-memory
conditions (via the kernel's OOM killer having somewhere to page from)
rather than serving as routine extra capacity.

**Swap also has a tunable aggressiveness, not just an on/off state.**
`vm.swappiness` (0–100, default usually 60) controls how strongly the
kernel prefers swapping out inactive memory pages versus reclaiming page
cache instead:

```bash
cat /proc/sys/vm/swappiness
sudo sysctl vm.swappiness=10          # temporary, until reboot
echo 'vm.swappiness=10' | sudo tee -a /etc/sysctl.conf   # persistent
```

A lower value (e.g. `10`) tells the kernel to strongly prefer keeping
processes resident in RAM and only swap as a last resort — common on
database servers, where any swap I/O directly costs query latency. Note the
same temporary-vs-persistent pattern as everything else in this demo:
`sysctl` alone is runtime-only; `/etc/sysctl.conf` (or a file under
`/etc/sysctl.d/`) is what survives a reboot.

## Part 2 — Hands-On Lab

### Commands used in this demo

| Command | Flag meaning | Purpose |
|---|---|---|
| `lsblk -f` | `-f`: filesystem info | Map block devices before touching anything |
| `fdisk /dev/sdX` | — | Interactive partitioning |
| `parted /dev/sdX mklabel gpt` | — | Create a fresh GPT partition table |
| `parted /dev/sdX mkpart ...` | — | Scriptable partitioning, better GPT/large-disk support |
| `mkfs.ext4` / `mkfs.xfs` | — | Create a filesystem on a partition |
| `blkid` | — | Show UUID/label/type for fstab entries |
| `mount -a` | `-a`: all fstab entries | Test fstab without rebooting |
| `df -h` | `-h`: human-readable | Filesystem-level usage |
| `du -sh PATH` | `-s`: summary, `-h`: human-readable | Actual file-tree usage |
| `mkswap` / `swapon` | — | Create and activate swap |
| `sysctl vm.swappiness=N` | — | Tune swap aggressiveness (temporary) |

### Task 1 — Identify and inspect a scratch disk

```bash
lsblk -f
# Identify an unformatted scratch/lab disk (never the root disk) for the
# rest of this demo — confirm it has no existing filesystem or mountpoint.
```

### Task 2 — Partition with fdisk, then recreate with parted for comparison

```bash
# Approach 1: fdisk (interactive)
sudo fdisk /dev/sdb
# n, p, 1, <enter> (default first sector), +5G, w
lsblk -f /dev/sdb

# Remove that partition, then recreate the same 5G partition with parted
# instead — directly comparing both tools' workflows on the same disk:
sudo fdisk /dev/sdb
# d, 1, w   (delete partition 1, write)

# Approach 2: parted (scriptable, no interactive menu)
sudo parted /dev/sdb mklabel gpt
# mklabel gpt creates a fresh partition table, discarding the one fdisk
# just created — safe here because this is your scratch disk only, never
# do this on a disk holding data you need.
sudo parted /dev/sdb mkpart primary ext4 0% 5GiB
lsblk -f /dev/sdb
# Confirm sdb1 exists again — this time created non-interactively via
# parted rather than fdisk's interactive prompts.
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

### Task 6 — Configure swap and tune its aggressiveness

```bash
sudo fallocate -l 512M /swapfile-lab
sudo chmod 600 /swapfile-lab
sudo mkswap /swapfile-lab
sudo swapon /swapfile-lab
swapon --show
free -h

# Check and tune swappiness — capture the original value so you can
# restore it exactly, rather than assuming a default:
ORIG_SWAPPINESS=$(cat /proc/sys/vm/swappiness)
echo "Original swappiness: $ORIG_SWAPPINESS"
sudo sysctl vm.swappiness=10
cat /proc/sys/vm/swappiness
sudo sysctl vm.swappiness=$ORIG_SWAPPINESS
cat /proc/sys/vm/swappiness
# Confirm it's back to the original value — this was a temporary runtime
# change either way, nothing was written to /etc/sysctl.conf.

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
6. ✅ Configured and activated swap space correctly, including tuning
   `vm.swappiness`

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
file at all. The tool to find the responsible process is `lsof` (Demo M03-02,
built on the pattern from Demo M01-08): `sudo lsof +L1 /var/log` (or `lsof
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
§1.4 describes, the same pattern as `ip addr add` (Demo M03-01) and firewalld
rules without `--permanent` (Demo M03-02): it worked until the next reboot, then
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

---

**Q2. Why is using a UUID in `/etc/fstab` preferred over a raw device path like `/dev/sdb1`?**

Device names aren't guaranteed stable across reboots — disk enumeration
order can shift, especially with multiple disks attached or after hardware
changes, which can silently mount the wrong device at a given path. A
filesystem's UUID is generated once and doesn't change, so an fstab entry
referencing it correctly identifies the intended filesystem regardless of
what device name the kernel happens to assign it this boot.

---

**Q3. When would you reach for `parted` instead of `fdisk`?**

When scripting partition creation non-interactively, or when working with
GPT partition tables and disks larger than 2TB, where `fdisk`'s older
MBR-era assumptions become limiting. For quick interactive partitioning of a
single disk under 2TB, `fdisk` remains the more commonly reached-for tool.

---

**Q4. Why would you choose `ext4` over `xfs` for a volume you expect might need to shrink later?**

`ext4` supports online shrinking in more scenarios; `xfs` generally cannot
be shrunk at all without recreating the filesystem entirely. `xfs` is often
the better default for very large files or high-concurrency workloads, but
that resize limitation is a real, practical factor to weigh against those
benefits specifically when a volume's eventual size is expected to
decrease, not just grow.

---

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
   always a separate, required step after partitioning, and `parted`'s
   filesystem-type argument in `mkpart` is only a metadata hint, not an
   actual format operation.
6. **Swap is a safety buffer, not routine extra RAM.** Active swap usage
   (`vmstat`'s `si`/`so`, from Demo M02-07) means disk I/O standing in for
   memory access — a performance problem to investigate, not a capacity
   feature to lean on.
7. **Swap has a tunable aggressiveness, not just an on/off state.**
   `vm.swappiness` controls how eagerly the kernel prefers swapping over
   reclaiming page cache — lower it on latency-sensitive workloads like
   databases. Like every other persistence pattern in this demo, `sysctl`
   alone is temporary; `/etc/sysctl.conf` (or `/etc/sysctl.d/`) is what
   survives a reboot.

> **Demo scope:** Primary concept: provisioning and mounting storage.
> Supporting concepts: lsblk, fdisk/parted partitioning, mkfs, mount/umount/
> fstab persistence, df vs. du, swap configuration and tuning.
> Estimated completion time: 42 minutes (reading + hands-on + verification).
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
| `parted /dev/sdX mklabel gpt` | Create a fresh GPT partition table (destroys existing one) |
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
| `fallocate -l SIZE /swapfile && chmod 600` | Create a swapfile (fast) |
| `dd if=/dev/zero of=/swapfile bs=1M count=N` | Create a swapfile (slower, always safe fallback) |
| `mkswap /swapfile` | Format as swap |
| `swapon /swapfile` / `swapon --show` | Activate / verify |
| `sysctl vm.swappiness=N` | Tune swap aggressiveness (temporary; persist via `/etc/sysctl.conf`) |

## Appendix A — Anki Cards

**`m04-01-storage-fundamentals-anki.csv`:**

```
#deck:Linux Mastery::Module 4 - Storage::Demo M04-01 - Storage Fundamentals
#separator:Comma
#columns:Front,Back,Tags
"df shows a partition at 95% used, but du -sh on the same path only accounts for half that space. What's the most likely cause?","A deleted file still held open by a running process. df counts it as used (filesystem hasn't reclaimed the blocks); du only sees files currently visible in the directory tree and can't account for a deleted one at all.","demo-m04-01,df,du,troubleshooting"
"Why is UUID preferred over a raw device path like /dev/sdb1 in /etc/fstab?","Device names aren't guaranteed stable across reboots — disk enumeration order can shift. A filesystem's UUID is generated once and doesn't change, avoiding a silent wrong-device mount.","demo-m04-01,fstab,uuid"
"A filesystem is mounted with a plain mount command but has no /etc/fstab entry. What happens after a reboot?","It disappears — a mount without a corresponding fstab entry is temporary, the same persistence trap as ip addr add (Demo M03-01) or a firewalld rule without --permanent (Demo M03-02).","demo-m04-01,mount,persistence,cross-demo"
"What does mount -a do, and why run it instead of just rebooting to test a new fstab entry?","It applies every entry in /etc/fstab immediately, surfacing a syntax error right away in a live session. Rebooting to test is slower and, if the entry is wrong, risks a boot that hangs waiting on a bad mount.","demo-m04-01,fstab,mount-a"
"When would you choose parted over fdisk for partitioning?","When scripting partition creation non-interactively, or when working with GPT and disks larger than 2TB, where fdisk's older MBR-era assumptions become limiting.","demo-m04-01,fdisk,parted,comparison"
"Why might ext4 be preferred over xfs for a volume expected to need shrinking later?","ext4 supports online shrinking in more scenarios; xfs generally cannot be shrunk at all without recreating the filesystem entirely.","demo-m04-01,ext4,xfs,comparison"
"What's the correct tool to find which process is responsible for a df/du space discrepancy caused by a deleted-but-open file?","lsof — looking for open file handles pointing at deleted files (e.g. lsof | grep deleted).","demo-m04-01,lsof,df,du,cross-demo"
"Is swap best thought of as extra RAM capacity for routine use?","No — accessing swapped data means disk I/O instead of memory access, orders of magnitude slower. Swap's real value is as a buffer against sudden memory spikes and enabling clean OOM handling, not routine extra capacity.","demo-m04-01,swap,fundamentals"
"Which vmstat columns (from Demo M02-07) directly indicate a system is actively relying on swap under load?","si and so (swap in/swap out) — sustained non-zero values mean the kernel is genuinely moving pages to/from disk, a performance problem to investigate.","demo-m04-01,swap,vmstat,cross-demo"
"What should you always run before partitioning any disk, and why?","lsblk (ideally lsblk -f) — it's the cheapest way to confirm you're about to operate on the intended disk and not accidentally the root filesystem, where a mistake can mean destroyed data.","demo-m04-01,lsblk,safety"
```

## Appendix B — Quiz

**`m04-01-storage-fundamentals-quiz.md`:**

````markdown
# Quiz — Demo M04-01: Storage Fundamentals

> One correct answer per question unless stated otherwise.
> Target: 80% or above before moving to Demo M04-04.

---

**Q1.** `lsblk -f` shows disk `sdb` with no entries at all under `FSTYPE`, `LABEL`, `UUID`, or `MOUNTPOINT` for any of its partitions (or the disk itself, if unpartitioned). What does this tell you?

- A) The disk is broken and needs replacing
- B) The disk (or that partition) has no filesystem on it and isn't mounted anywhere — exactly the starting state you want to confirm before partitioning or formatting
- C) The disk is currently in use by another process
- D) `lsblk` requires `-a` to show this information; `-f` alone is insufficient

<details>
<summary>Answer</summary>

**B** — Blank `FSTYPE`/`LABEL`/`UUID`/`MOUNTPOINT` means there's no filesystem present and nothing mounted from it — this is exactly the state you want to confirm on your intended scratch disk before running any partitioning or `mkfs` command, precisely so you don't accidentally target a disk that already has data.

Trap: A and C both invent unrelated problems; D is wrong — `-f` is exactly the flag that surfaces this information.

</details>

---

**Q2.** `sudo parted /dev/sdb mkpart primary ext4 0% 100%` is run. Does this format the partition as ext4?

- A) Yes — this is `parted`'s equivalent of `mkfs.ext4`
- B) No — `ext4` here only sets a filesystem-type hint in the partition table metadata; the actual filesystem still must be created separately with `mkfs`
- C) Yes, but only for partitions smaller than 2TB
- D) No — the `ext4` argument is ignored entirely by `parted`

<details>
<summary>Answer</summary>

**B** — `parted` never creates a filesystem on its own, regardless of what filesystem type is named in `mkpart`. The name is recorded as a hint in the partition table; `mkfs.ext4` (or whichever filesystem you actually want) is still a required, separate step.

Trap: A and C both wrongly assume `parted` performs formatting; D is wrong — the argument isn't ignored, it's recorded as metadata, just not acted upon as a format operation.

</details>

---

**Q3.** An `/etc/fstab` line reads: `UUID=abc-123  /mnt/data  ext4  defaults  0  2`. What do the final two fields (`0` and `2`) mean?

- A) `0` is the mount priority; `2` is the number of retry attempts
- B) `0` is the legacy dump-backup flag (almost always `0` today); `2` is the fsck pass number — checked after the root filesystem (which uses pass `1`)
- C) `0` and `2` together specify a 2-second mount timeout
- D) These fields are ignored by modern Linux and exist only for historical compatibility

<details>
<summary>Answer</summary>

**B** — The dump field is a legacy flag for the old `dump` backup utility, essentially always `0` on modern systems. The fsck pass field controls `fsck` ordering at boot: `0` = never check, `1` = check first (reserved for the root filesystem), `2` = check after pass 1 — the standard value for a non-root data volume.

Trap: A and C invent meanings unrelated to fstab's actual field definitions; D is wrong — the fsck pass field is still actively used to determine boot-time filesystem check order.

</details>

---

**Q4.** Why run `blkid /dev/sdb1` before writing an `/etc/fstab` entry for it?

- A) `blkid` formats the partition in preparation for mounting
- B) `blkid` shows the partition's UUID, filesystem type, and label — exactly the identifying information needed to write a correct, stable fstab entry
- C) `blkid` is required to unlock the partition before it can be referenced in fstab
- D) `blkid` and `mkfs` are interchangeable for creating a filesystem

<details>
<summary>Answer</summary>

**B** — `blkid` reads and reports the identifying metadata (UUID, TYPE, LABEL) already present on the filesystem, which is exactly what you need to reference it reliably (by UUID or LABEL) in `/etc/fstab`, rather than the unstable device path.

Trap: A, C, and D all invent functions `blkid` doesn't perform — it's a read-only reporting tool, not a formatting or unlocking one.

</details>

---

**Q5.** What does `mkfs.ext4 -L lab-data /dev/sdb1` accomplish that plain `mkfs.ext4 /dev/sdb1` does not?

- A) It creates the filesystem faster
- B) It assigns a human-readable label (`lab-data`) to the filesystem, making it identifiable in `lsblk -f` and usable in `/etc/fstab` as `LABEL=lab-data` instead of a raw device path or UUID
- C) It automatically adds the filesystem to `/etc/fstab`
- D) It reserves additional space for the label metadata, reducing usable capacity noticeably

<details>
<summary>Answer</summary>

**B** — The `-L` flag sets a filesystem label at creation time, which then becomes a stable, human-readable way to reference the filesystem — in `lsblk -f`'s LABEL column and as `LABEL=name` in fstab.

Trap: A invents a performance claim; C invents automatic fstab management that doesn't happen; D overstates the (negligible) metadata overhead as a meaningful capacity cost.

</details>

---

**Q6.** `fallocate -l 2G /swapfile` followed by `mkswap /swapfile` fails or behaves unexpectedly on a particular filesystem. What's the most likely explanation, and what's the safe fallback?

- A) `fallocate` never works for swap files under any circumstances; always use `dd`
- B) On some filesystems (Btrfs in particular), `fallocate` can produce a file with holes that `mkswap` rejects — `dd if=/dev/zero of=/swapfile bs=1M count=N` is slower but always reliably fills the file with real blocks
- C) The swapfile must be created on a separate physical disk from the root filesystem
- D) `fallocate` requires the file to already exist before it can allocate space

<details>
<summary>Answer</summary>

**B** — This is a known, filesystem-dependent caveat: `fallocate` is generally reliable on ext4/xfs but can create sparse/holed files on some filesystems that `mkswap` won't accept as swap. `dd` is slower (it actually writes every block) but always produces a fully-allocated file that works.

Trap: A overstates the problem — `fallocate` does work reliably on ext4/xfs, the common case; C and D invent unrelated requirements.

</details>

---

**Q7.** `cat /proc/sys/vm/swappiness` shows `60`. You lower it to `10` with `sudo sysctl vm.swappiness=10` on a database server. What have you actually changed, and does it survive a reboot?

- A) You've disabled swap entirely; it does survive reboot
- B) You've told the kernel to more strongly prefer keeping processes in RAM and reclaim page cache instead of swapping, unless truly necessary; this change is temporary and will NOT survive a reboot without also adding it to `/etc/sysctl.conf` (or `/etc/sysctl.d/`)
- C) You've increased the swap file's size by lowering swappiness
- D) `swappiness` only affects swap partitions, not swap files, so this change has no effect if using a swapfile

<details>
<summary>Answer</summary>

**B** — Lowering `vm.swappiness` makes the kernel more reluctant to swap, preferring to reclaim page cache first — valuable on latency-sensitive workloads like databases where swap I/O directly costs query time. Like every other `sysctl` change and every other runtime-only action in this demo, it's temporary unless also written to a persistent config file.

Trap: A misreads swappiness as a full disable switch (0 still permits swapping, just least aggressively); C invents an unrelated sizing effect; D is false — swappiness applies regardless of whether swap is backed by a partition or a file.

</details>

---

**Q8.** Your scratch disk already has an MBR partition table from an earlier `fdisk` session. You then run `sudo parted /dev/sdb mklabel gpt`. What happens?

- A) `parted` refuses to run since a partition table already exists
- B) The existing partition table is discarded and replaced with a fresh, empty GPT partition table — any partitions previously on the disk are gone
- C) The disk now has both an MBR and a GPT table simultaneously, selectable at boot
- D) `mklabel gpt` only affects new partitions created afterward; existing ones are untouched

<details>
<summary>Answer</summary>

**B** — `mklabel` creates a brand-new partition table of the specified type, discarding whatever was there before. This is safe to do deliberately on a scratch/lab disk with no data you need, but is exactly the kind of command that must never be run against a disk (or the wrong disk) holding real data.

Trap: A invents a refusal that doesn't happen; C invents a dual-table state that isn't how partition tables work; D wrongly suggests old partitions survive the operation.

</details>

---

Score guide:

| Score | Action |
|---|---|
| 7/8 or 8/8 | Import Anki cards, move to Demo M04-04 |
| 6/8 | Review the wrong answers, then proceed |
| 5/8 | Re-read the relevant sections, retry those questions |
| Below 5/8 | Re-read the full demo and redo the walkthrough before proceeding |
````