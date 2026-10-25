# Demo 16 — Remote Storage & Troubleshooting

## Overview

This demo closes out the storage phase with two things: accessing storage
that lives on another machine entirely, and diagnosing storage problems when
something's already gone wrong. NFS and network block devices cover the
"remote" half; the virtual filesystem, `fsck`, and a structured
troubleshooting decision tree cover the "something's wrong, now what" half.

## Demo Objectives

1. Explain what `/proc` and `/sys` actually are, and why they're not real disk storage
2. Mount an NFS share, both temporarily and persistently
3. Explain what a network block device is and how it differs conceptually
   from NFS, and configure one hands-on to make that difference concrete
4. Configure `autofs` to mount storage on demand rather than at boot
5. Run `fsck` correctly, including why it must not run on a mounted filesystem
6. Apply a systematic decision tree to diagnose "can't access this storage" issues

## Directory Structure

```
16-remote-storage-troubleshooting/
├── README.md                                          # this file
├── 16-remote-storage-troubleshooting-anki.csv         # Anki flashcard deck
└── 16-remote-storage-troubleshooting-quiz.md          # standalone quiz
```

## Environments

| Task | Ubuntu 24.04 | Rocky Linux 9.8 | EC2 Amazon Linux |
|---|---|---|---|
| NFS server/client | `nfs-kernel-server` / `nfs-common` | `nfs-utils` (both) | Not required |
| NBD server/client | `sudo apt install nbd-server` | `sudo dnf install nbd` (enable EPEL first if not found in base repos) | Not required |
| `autofs` | `sudo apt install autofs` | `sudo dnf install autofs` | Not required |

## Recall Check — 15-lvm-management

Answer these from memory before continuing — no looking back at Demo 15.

1. `lvextend` grows an LV, but `df` still shows the old size. What was
   missed?

2. What does a traditional LVM snapshot actually store, and what
   happens if changes to the origin exceed its allocated size?

3. Why must a filesystem be shrunk before the LV when reducing — the
   opposite order from extending?

<details>
<summary>Answers</summary>

1. The filesystem resize step (`resize2fs`/`xfs_growfs`) — `lvextend`
   only grows the LV itself; the filesystem needs to be told separately to
   expand into the new space, unless `lvextend -r` was used to do both at once.

2. Only the changes made to the origin after the snapshot was taken
   (copy-on-write), not a full copy. If changes exceed the snapshot's allocated
   size, it's automatically invalidated and dropped.

3. The filesystem needs its full original space available to safely
   relocate data out of blocks that are about to be removed — reversing the
   order risks destroying data still living in those blocks.

</details>

## Part 1 — Concepts

### 1.1 /proc and /sys — not real storage

```bash
df -h /proc
```

```
Filesystem     Size  Used Avail Use% Mounted on
proc              0     0     0    - /proc
```

**`/proc` and `/sys` are virtual filesystems — they don't occupy disk
space at all.** Every file under them is generated on-demand by the kernel
when read, reflecting live kernel/process state — there's nothing on disk
backing `/proc/loadavg` (Demo 09) or `/proc/meminfo` (also Demo 09); the
kernel constructs that content the instant something reads it. `/sys`
exposes device and driver state in a similar spirit, organized around the
kernel's internal device model rather than process information.

```bash
cat /sys/class/net/eth0/operstate
# up
cat /sys/block/sda/size
# size in 512-byte sectors — a hardware-level number no other tool this
# series has shown directly.
```

The practical relevance to storage specifically: `/sys/block/` is where the
kernel's view of every block device lives, useful for confirming a disk is
actually recognized by the kernel at all before troubleshooting anything
higher up the stack (LVM, filesystem, mount).

### 1.2 NFS — mounting remote storage

```bash
# Server side (exporting a directory):
echo '/export/shared  10.0.4.0/24(rw,sync,no_subtree_check)' | sudo tee -a /etc/exports
sudo exportfs -ra
# -r: re-export all, -a: all entries — applies /etc/exports changes without
# restarting the NFS service entirely.
```

`sync` (as opposed to `async`) means the server confirms a write request
only after it's actually been committed to stable storage, not just
accepted into memory — slower, but the write survives a server crash
immediately after being acknowledged. `async` acknowledges writes faster but
risks losing recently-written data if the server crashes before actually
flushing it to disk, even though the client was already told the write
succeeded. `sync` is the safer default and what's used throughout this demo.

```bash
# Client side:
showmount -e nfs-server.novatech.internal
# Confirms what's actually being exported, from the client's perspective —
# a mismatch here (nothing shown, or a different path than expected) rules
# out client-side mount configuration as the problem before you even try
# to mount.

sudo mount -t nfs nfs-server.novatech.internal:/export/shared /mnt/shared
```

Persisting it — the same `/etc/fstab` pattern as Demo 14, with NFS-specific
options:

```bash
echo 'nfs-server.novatech.internal:/export/shared  /mnt/shared  nfs  defaults,_netdev  0  0' | sudo tee -a /etc/fstab
```

**`_netdev` is not optional for network-backed mounts.** It tells the boot
process this mount depends on the network being up first — without it, the
system may attempt the mount before networking has initialized, fail, and
either hang or silently skip it, which looks identical to an NFS server
being unreachable and can send troubleshooting in the wrong direction
entirely.

### 1.3 Network block devices — a different remote-storage model

NFS exports a *directory* — the remote side already has a filesystem on it,
and the client sees files and folders. A **network block device** (NBD)
exports a *raw block device* instead — the client sees what looks like a
local disk (`/dev/nbd0`), with no filesystem imposed by the remote side at
all; the client formats and manages it exactly like a local disk (including
everything from Demo 14 — partition it, or hand it straight to LVM as a PV
from Demo 15).

```bash
sudo modprobe nbd
sudo nbd-client nbd-server.novatech.internal 10809 /dev/nbd0
lsblk /dev/nbd0
# Once connected, /dev/nbd0 behaves like any other block device — mkfs,
# mount, or pvcreate all apply exactly as they would to a local disk.
```

**The conceptual distinction matters for troubleshooting:** an NFS problem
is often a permissions/export/filesystem-level issue on the remote side; an
NBD problem is closer to a network connectivity issue to a raw device, with
filesystem concerns living entirely on the client, since the client owns the
filesystem layer that NFS's server side would otherwise own. Task 4 makes
this concrete: you'll run `mkfs` on an NBD device yourself, something you
never do on an NFS mount.

### 1.4 autofs — mounting on demand

Every mount covered so far is either always-on (via `/etc/fstab`) or
manual. `autofs` adds a third option: mount automatically the moment
something accesses the path, and unmount automatically after a period of
inactivity — useful for storage that's occasionally needed but shouldn't sit
mounted (and consuming a live NFS connection) permanently.

```bash
echo '/mnt/auto  /etc/auto.shares' | sudo tee -a /etc/auto.master
echo 'shared  -rw,soft  nfs-server.novatech.internal:/export/shared' | sudo tee /etc/auto.shares
sudo systemctl restart autofs

ls /mnt/auto/shared
# The access itself triggers the mount — nothing was mounted before this
# command ran.
mount | grep shared
# Confirms it's now actually mounted, triggered by the access above.
```

The `/etc/auto.master` line maps a base directory (`/mnt/auto`) to a **map
file** (`/etc/auto.shares`) that lists what can be mounted under it and how.
Each line in the map file is `subdirectory-name  mount-options
remote-path` — here, accessing `/mnt/auto/shared` looks up the `shared`
entry in the map file and mounts exactly what that line specifies.

`-rw,soft` here: `soft` (as opposed to `hard`, NFS's default) means an NFS
operation that times out returns an error to the calling application rather
than hanging indefinitely — a real tradeoff, since `soft` mounts risk silent
data loss on a timeout for write operations, versus `hard`'s guarantee of
eventually completing (or hanging forever) rather than silently failing.
Reads are generally safe to retry from scratch after a `soft` timeout;
writes are the genuinely risky case, because a write already in flight may
be partially applied by the time the timeout fires, with no guarantee the
application even knows to retry or verify what actually landed on disk.

### 1.5 fsck — checking and repairing filesystems

```bash
sudo fsck /dev/sdb1
```

**`fsck` must never be run on a mounted filesystem** — the filesystem's live
in-memory state and its on-disk state can differ while mounted, and `fsck`
operating against a mounted filesystem risks the exact kind of corruption
it's meant to detect and fix, rather than genuinely useless-but-harmless
behavior. This isn't a soft guideline; modern `fsck` implementations
typically refuse to run destructively on a mounted filesystem by default,
but relying on that refusal rather than unmounting first is not the correct
habit to build.

```bash
sudo umount /mnt/data
sudo fsck -f /dev/sdb1
# -f: force a check even if the filesystem appears clean per its own
# metadata — useful when you suspect corruption the filesystem's own
# "clean" flag might not have caught.
```

`fsck` on the root filesystem specifically can't be run this way at all
(you can't unmount `/` while the system is running from it) — that's what
the automatic fsck-on-boot pass (Demo 14 §1.4's fstab pass-number field) and
rescue/single-user mode boot targets exist for.

### 1.6 A storage troubleshooting decision tree

When storage is inaccessible, work through these in order:

1. **Does the kernel see the device at all?** (`lsblk`, or `/sys/block/` per
   §1.1 — if the device isn't there, nothing above this layer matters yet.)
2. **For remote storage, is the network path actually working?** (Demo 12's
   `ss`/`tcpdump` toolkit — is the NFS server or NBD host even reachable?)
3. **Is the mount actually active?** (`mount | grep PATH`, or `df -h` — a
   mount that failed silently at boot due to `_netdev` being missing looks
   identical to "storage is fine but empty.")
4. **Is the filesystem itself healthy?** (`fsck`, only after unmounting —
   the last resort, not the first thing to reach for.)

This order matters for the same reason Demo 12's decision tree did: each
step rules out an entire category before the next, and jumping straight to
`fsck` on a filesystem that was simply never mounted (or a device the kernel
never even recognized) wastes time diagnosing the wrong layer.

## Part 2 — Hands-On Lab

### Commands used in this demo

| Command | Flag meaning | Purpose |
|---|---|---|
| `cat /sys/block/sdX/size` | — | Confirm the kernel recognizes a block device |
| `showmount -e SERVER` | `-e`: list exports | Confirm what an NFS server actually exports |
| `mount -t nfs SERVER:/path /mnt/x` | `-t nfs`: filesystem type | Mount an NFS share |
| `exportfs -ra` | `-r`: re-export, `-a`: all | Apply `/etc/exports` changes without restarting NFS |
| `modprobe nbd` | — | Load the kernel's network block device module |
| `nbd-client SERVER PORT /dev/nbdN` | — | Connect a network block device |
| `nbd-client -d /dev/nbdN` | `-d`: disconnect | Disconnect a network block device |
| `fsck -f /dev/sdX1` | `-f`: force check | Check/repair a filesystem (unmounted only) |

### Task 1 — Confirm kernel-level device visibility

```bash
lsblk
cat /sys/block/sda/size
cat /sys/class/net/*/operstate
# Confirm you can find both a block device and a network interface's state
# directly through /sys, without any higher-level tool.
```

### Task 2 — NFS export and mount (single-host demo, server and client roles on the same lab machine)

```bash
sudo mkdir -p /export/shared
echo "shared test file" | sudo tee /export/shared/hello.txt
echo '/export/shared  127.0.0.1(rw,sync,no_subtree_check)' | sudo tee -a /etc/exports
sudo exportfs -ra

showmount -e localhost
sudo mkdir -p /mnt/nfs-shared
sudo mount -t nfs localhost:/export/shared /mnt/nfs-shared
cat /mnt/nfs-shared/hello.txt
```

### Task 3 — Persist the NFS mount with _netdev

```bash
echo 'localhost:/export/shared  /mnt/nfs-shared  nfs  defaults,_netdev  0  0' | sudo tee -a /etc/fstab
sudo umount /mnt/nfs-shared
sudo mount -a
mount | grep nfs-shared
```

### Task 4 — Network block device: server and client on the same machine

```bash
# Server side — export a local file as a raw block device
sudo apt install -y nbd-server            # Ubuntu
# sudo dnf install -y nbd                 # Rocky9 (enable EPEL first if needed)

sudo dd if=/dev/zero of=/export/nbd-backing-file bs=1M count=200
echo '[demo-export]
    exportname = /export/nbd-backing-file
    port = 10809' | sudo tee -a /etc/nbd-server/config
sudo systemctl restart nbd-server

# Client side — connect, then format and mount exactly like a local disk
sudo modprobe nbd
sudo nbd-client 127.0.0.1 10809 /dev/nbd0
lsblk /dev/nbd0
sudo mkfs.ext4 /dev/nbd0
sudo mkdir -p /mnt/nbd-demo
sudo mount /dev/nbd0 /mnt/nbd-demo
echo "hello via nbd" | sudo tee /mnt/nbd-demo/test.txt
cat /mnt/nbd-demo/test.txt

# Contrast with Task 2: here YOU ran mkfs and chose the filesystem — with
# NFS, the remote side already had a filesystem and you never touched mkfs
# at all. This is §1.3's conceptual distinction, made concrete.

# Clean up
sudo umount /mnt/nbd-demo
sudo nbd-client -d /dev/nbd0
sudo systemctl stop nbd-server
```

### Task 5 — autofs on-demand mounting

```bash
sudo mkdir -p /mnt/auto
echo '/mnt/auto  /etc/auto.demo' | sudo tee -a /etc/auto.master
echo 'shared  -rw,soft  localhost:/export/shared' | sudo tee /etc/auto.demo
sudo systemctl restart autofs
mount | grep auto   # not yet mounted
ls /mnt/auto/shared
mount | grep auto   # now mounted, triggered by the access
```

### Task 6 — fsck: observe the mounted-filesystem refusal, then check for real

```bash
# First, deliberately try fsck WITHOUT unmounting first:
sudo fsck -f /dev/sdb1
# On most systems fsck detects /dev/sdb1 is mounted and refuses to run
# destructively, printing a warning instead of proceeding — but never rely
# on this refusal as your safety net; always unmount first as the correct
# habit regardless of what any given implementation happens to do.

sudo umount /mnt/nfs-shared 2>/dev/null
# Using a local scratch partition instead, since NFS filesystems aren't
# fsck'd this way:
sudo umount /mnt/labdata 2>/dev/null   # from Demo 14's lab volume, if present
sudo fsck -f /dev/sdb1
# Now it actually runs the check, correctly, on an unmounted filesystem.
```

### Task 7 — Work the decision tree on a broken remote mount

```bash
sudo systemctl stop nfs-server 2>/dev/null || sudo systemctl stop nfs-kernel-server 2>/dev/null
mount | grep nfs-shared    # still shows mounted from earlier, stale
ls /mnt/nfs-shared          # observe what happens accessing a mount whose
                             # server just went away — work through §1.6's
                             # decision tree to correctly identify this as
                             # a server/network issue, not a local one
sudo systemctl start nfs-server 2>/dev/null || sudo systemctl start nfs-kernel-server 2>/dev/null
```

## What You Learned

1. ✅ Explained what `/proc` and `/sys` actually are, and why they're not disk storage
2. ✅ Mounted an NFS share, both temporarily and persistently with `_netdev`
3. ✅ Explained network block devices and how they differ conceptually from
   NFS, and demonstrated the difference hands-on by connecting, formatting,
   and mounting one
4. ✅ Configured `autofs` for on-demand mounting
5. ✅ Ran `fsck` correctly, only on an unmounted filesystem, and observed its
   refusal to run destructively on a mounted one
6. ✅ Applied a systematic decision tree to diagnose storage access issues

## Part 3 — Break-Fix Scenario

### Scenario — NovaTech Ops: two remote storage incidents

> **Incident report — NovaTech Ops, ticket #4601**
>
> Alert 1: "app-server-07 hangs on boot after adding an NFS mount to
> `/etc/fstab`." Alert 2: "`/mnt/shared-data` shows as mounted, but every
> file access hangs indefinitely."

**Incident 1 — hangs on boot after adding an NFS fstab entry**

```bash
$ cat /etc/fstab | tail -1
nfs-server:/export/data  /mnt/shared  nfs  defaults  0  0
```

Tasks:
1. Compare this fstab line against §1.2's example. What's missing, and why
   would that specifically cause a boot-time hang rather than a normal
   mount failure?
2. Fix the entry, and explain in one sentence what the fix actually
   guarantees about mount ordering during boot.

**Incident 2 — mounted but every file access hangs**

```bash
$ mount | grep shared-data
nfs-server:/export/data on /mnt/shared-data type nfs (rw,hard,...)
$ ls /mnt/shared-data
# hangs, no output, no error
```

Tasks:
1. The mount options show `hard` (not `soft`). Given §1.4's explanation of
   the `hard`/`soft` tradeoff, is this hanging behavior a malfunction or
   expected behavior for a `hard` mount when the server is unreachable?
2. Work through §1.6's decision tree — which step would have caught this
   before someone got stuck waiting on a hung `ls`?
3. What would changing this mount to `soft` actually trade away in exchange
   for not hanging?

<details>
<summary>Answers</summary>

**Incident 1:** The `_netdev` option is missing. Without it, the boot
process doesn't know this mount depends on the network being available
first, and may attempt to mount it before networking has initialized —
because NFS mounts (especially with default `hard` behavior) can block
waiting for an unreachable server rather than failing fast, this can hang
the entire boot sequence waiting on a mount that has no realistic chance of
succeeding yet. Fix: `nfs-server:/export/data /mnt/shared nfs
defaults,_netdev 0 0`. The fix guarantees the mount is only attempted after
network dependencies are satisfied during boot, rather than racing ahead of
them.

**Incident 2:** This is expected behavior for a `hard` mount, not a
malfunction — per §1.4, `hard` mounts guarantee an NFS operation eventually
completes rather than silently failing, which means blocking/hanging
indefinitely while the server is unreachable, by design. Working the
decision tree: step 2 (is the network path actually working — is the NFS
server reachable at all, via Demo 12's `ss`/connectivity tools) would have
identified the server-side unavailability *before* anyone ran a file
operation and got stuck waiting on it. Changing to `soft` would trade away
`hard`'s guarantee that operations eventually complete (or wait indefinitely
rather than silently fail) — a `soft` mount instead returns an error to the
calling application on timeout, which avoids the hang but risks silent data
loss or corruption for in-progress writes that get cut off by a timeout
rather than genuinely completing.

</details>

## Interview Prep

**Q1. Why is `_netdev` a required option for any network-backed `/etc/fstab` entry, not just a best practice?**

Without it, the boot process has no way to know this specific mount depends
on networking being available first, and may attempt it too early — since
network mounts (especially `hard` NFS mounts) can block indefinitely rather
than failing fast when unreachable, this risks hanging the entire boot
sequence rather than just that one mount failing cleanly.

---

**Q2. A file access on an NFS mount hangs indefinitely with no error. Is this necessarily a bug?**

Not necessarily — check the mount options for `hard` versus `soft`. `hard`
(NFS's default) is designed to block and eventually complete an operation
rather than fail it, which manifests as an indefinite hang when the server
is unreachable. This is expected behavior for that setting, not a
malfunction, though it may not be the right setting for every use case.

---

**Q3. Why should `fsck` never be run on a mounted filesystem?**

The filesystem's live in-memory state and its on-disk state can diverge
while mounted; running `fsck` against a mounted filesystem risks causing the
exact kind of corruption it's meant to detect and repair, rather than being
merely ineffective. Modern implementations typically refuse to run
destructively on a mounted filesystem, but relying on that refusal instead
of unmounting first isn't the correct habit.

---

**Q4. What's the conceptual difference between NFS and a network block device, and why does it matter for troubleshooting?**

NFS exports a directory — the remote side owns the filesystem, and the
client sees files/folders. A network block device exports a raw block
device — the client owns the filesystem layer entirely and manages it like
any local disk. This matters for troubleshooting because an NFS problem is
often about exports, permissions, or the remote filesystem; an NBD problem
is closer to pure network connectivity to a raw device, with filesystem
concerns being entirely the client's own responsibility.

---

## Key Takeaways

1. **`/proc` and `/sys` are virtual, kernel-generated, not real disk
   storage.** Nothing backs them on disk — every file is constructed on
   read, reflecting live kernel state.
2. **`_netdev` is mandatory, not optional, for network-backed `/etc/fstab`
   entries.** Without it, boot can attempt a network mount before
   networking is ready, risking a hang rather than a clean failure.
3. **NFS exports a filesystem; a network block device exports raw storage.**
   The client owns the filesystem layer with NBD but not with NFS — a real
   architectural difference that changes where you look when troubleshooting,
   and one this demo's lab makes concrete: you ran `mkfs` on the NBD device
   yourself, but never touched the NFS mount's filesystem at all.
4. **`hard` NFS mounts block indefinitely on an unreachable server by
   design; `soft` mounts fail fast but risk silent data loss.** A hanging
   file access on a `hard` mount during a server outage is expected
   behavior, not a bug — and writes are the genuinely risky case under
   `soft`, since a write already in flight may be only partially applied
   when the timeout fires.
5. **`fsck` must run only on an unmounted filesystem.** Running it against
   a mounted filesystem risks causing corruption rather than fixing it —
   never rely on the tool's own safety refusal instead of unmounting first,
   even though most implementations will indeed refuse and warn rather than
   silently proceeding if you forget.
6. **A systematic decision tree — kernel visibility, network reachability,
   mount state, filesystem health, in that order — beats guessing.** Each
   step rules out an entire category of cause before the next, the same
   principle as Demo 12's firewall decision tree applied to storage.
7. **NFS's `sync` export option confirms a write is on stable storage
   before acknowledging it; `async` acknowledges faster but can lose a
   recently-acknowledged write if the server crashes before actually
   flushing it.** `sync` is the safer default for exactly this reason.

> **Demo scope:** Primary concept: accessing remote storage and diagnosing
> storage problems. Supporting concepts: /proc and /sys, NFS, network block
> devices, autofs, fsck, a structured troubleshooting decision tree.
> Estimated completion time: 40 minutes (reading + hands-on + verification).
> Checkpoints: 7 natural stopping points (after Task 1 through Task 7).

## Quick Commands Reference

### Virtual filesystem inspection

| Command | Purpose |
|---|---|
| `cat /sys/block/sdX/size` | Confirm the kernel recognizes a block device |
| `cat /sys/class/net/IFACE/operstate` | Interface state directly from the kernel |

### NFS

| Command | Purpose |
|---|---|
| `showmount -e SERVER` | List what a server actually exports |
| `mount -t nfs SERVER:/path /mnt/x` | Mount an NFS share |
| `exportfs -ra` | Apply `/etc/exports` changes without restarting NFS |

### Network block devices

| Command | Purpose |
|---|---|
| `modprobe nbd` | Load the kernel's NBD client module |
| `nbd-client SERVER PORT /dev/nbdN` | Connect a network block device |
| `nbd-client -d /dev/nbdN` | Disconnect a network block device |

### autofs and fsck

| Command | Purpose |
|---|---|
| `systemctl restart autofs` | Apply autofs map changes |
| `fsck -f /dev/sdX1` | Force-check an unmounted filesystem |

## Appendix A — Anki Cards

**`16-remote-storage-troubleshooting-anki.csv`:**

```
#deck:Linux Mastery::Module 4 - Storage::Demo 16 - Remote Storage & Troubleshooting
#separator:Comma
#columns:Front,Back,Tags
"Why don't /proc and /sys occupy any disk space, and what does df show for them?","They are virtual filesystems — every file is generated on-demand by the kernel when read, reflecting live state. df shows 0 size/used/available for them since there's nothing backing them on disk.","demo16,proc,sys,fundamentals"
"Why is the _netdev option mandatory for a network-backed /etc/fstab entry, not just a best practice?","Without it, the boot process doesn't know this mount depends on networking being ready first, and may attempt it too early. Since network mounts (especially hard NFS mounts) can block indefinitely rather than fail fast, this risks hanging the entire boot sequence.","demo16,nfs,fstab,netdev"
"What's the conceptual difference between what NFS exports and what a network block device exports?","NFS exports a directory — the remote side already has a filesystem, and the client sees files/folders. A network block device exports a raw block device — the client owns the filesystem layer entirely, managing it exactly like a local disk.","demo16,nfs,nbd,comparison"
"An NFS mount uses hard (not soft) and a file access hangs indefinitely when the server is unreachable. Is this a bug?","No — this is expected behavior for a hard mount, which is designed to block and eventually complete an operation rather than fail it. soft mounts trade that guarantee for failing fast, at the risk of silent data loss on write timeouts.","demo16,nfs,hard-soft,troubleshooting"
"Why must fsck never be run on a mounted filesystem?","The filesystem's live in-memory state and on-disk state can diverge while mounted; running fsck against a mounted filesystem risks causing the exact corruption it's meant to detect and fix, not just being ineffective.","demo16,fsck,safety"
"What does autofs actually do differently from a normal /etc/fstab entry?","It mounts storage automatically the moment something accesses the path, and unmounts after a period of inactivity — useful for storage needed occasionally rather than sitting permanently mounted (and consuming a live connection).","demo16,autofs,fundamentals"
"What is the correct order for a storage troubleshooting decision tree, and why does it matter?","1) Does the kernel see the device at all? 2) Is the network path working (for remote storage)? 3) Is the mount actually active? 4) Is the filesystem itself healthy (fsck, last resort)? Each step rules out a category of cause before the next — jumping to fsck on a device the kernel never recognized wastes time on the wrong layer.","demo16,troubleshooting,decision-tree"
"Where in /sys can you confirm the kernel recognizes a specific block device, before troubleshooting anything higher up the storage stack?","/sys/block/ — e.g. cat /sys/block/sda/size confirms the kernel sees the device at the hardware level, before LVM, filesystem, or mount concerns are even relevant.","demo16,sys,troubleshooting"
"What does showmount -e SERVER confirm, and why check it before attempting an NFS mount?","What the server is actually exporting, from the client's perspective. Checking it first rules out client-side mount configuration as the problem if nothing (or the wrong path) shows up — the issue is on the server/export side, not the mount command.","demo16,nfs,showmount"
"Why does the client run mkfs on an NBD device, but never on an NFS mount?","With NBD, the remote side exports a raw block device with no filesystem imposed — the client owns the entire filesystem layer and must format it like any local disk. With NFS, the remote side already has a filesystem in place; the client only ever accesses files through it, never formatting anything itself.","demo16,nfs,nbd,comparison"
```

## Appendix B — Quiz

**`16-remote-storage-troubleshooting-quiz.md`:**

````markdown
# Quiz — Demo 16: Remote Storage & Troubleshooting

> One correct answer per question unless stated otherwise.
> Target: 80% or above before moving to Demo 17.

---

**Q1.** `cat /sys/block/sda/size` returns a number. What unit is this number in, and what does successfully reading it confirm?

- A) Gigabytes — confirms the filesystem on the disk is healthy
- B) 512-byte sectors — confirms the kernel recognizes this block device at the hardware level, before any LVM, filesystem, or mount concerns are even relevant
- C) Megabytes — confirms the disk is currently mounted
- D) Milliseconds — confirms the disk's average read latency

<details>
<summary>Answer</summary>

**B** — `/sys/block/*/size` reports device size in 512-byte sectors, a hardware-level number. Successfully reading it at all confirms the kernel sees this device, independent of anything about partitions, filesystems, or mounts layered on top later.

Trap: A, C, and D all invent incorrect units and unrelated meanings for what this file reports.

</details>

---

**Q2.** After adding a new line to `/etc/exports`, you run `sudo exportfs -ra` instead of restarting the NFS service entirely. What do the `-r` and `-a` flags each do, and why is this the better choice?

- A) `-r` restarts the service; `-a` applies changes only after a reboot
- B) `-r` re-exports everything; `-a` means all entries — together they apply `/etc/exports` changes to already-running NFS without dropping existing client connections a full service restart could disrupt
- C) `-r` and `-a` are aliases for the same flag; only one is needed
- D) `-r` requires root; `-a` requires the client to reconnect manually

<details>
<summary>Answer</summary>

**B** — `exportfs -ra` re-applies the full `/etc/exports` file to the running NFS service without a disruptive restart, making it the correct way to pick up new or changed export lines.

Trap: A invents an incorrect delayed-effect behavior; C wrongly claims redundancy; D invents unrelated permission/reconnection requirements.

</details>

---

**Q3.** `/etc/auto.master` contains `/mnt/auto /etc/auto.demo`, and `/etc/auto.demo` contains `shared -rw,soft localhost:/export/shared`. What does accessing `/mnt/auto/shared` actually do?

- A) Nothing — `auto.master` only takes effect after a reboot
- B) `autofs` looks up the `shared` entry in the map file (`/etc/auto.demo`) referenced by `auto.master`, and mounts `localhost:/export/shared` with the specified options at that moment
- C) It creates a new NFS export on the server automatically
- D) It fails, since `/mnt/auto/shared` doesn't exist as a directory beforehand

<details>
<summary>Answer</summary>

**B** — `auto.master` maps a base directory to a map file; the map file lists subdirectory names and what to mount there. Accessing the subdirectory triggers `autofs` to look up that specific entry and mount it on demand.

Trap: A invents a reboot requirement that contradicts autofs's entire on-demand design; C confuses client-side mounting with server-side exporting; D is wrong — autofs creates the mount point dynamically as needed.

</details>

---

**Q4.** To connect to a network block device, you run `sudo modprobe nbd` followed by `sudo nbd-client 127.0.0.1 10809 /dev/nbd0`. What does each part of the `nbd-client` command represent?

- A) `127.0.0.1` is the client's own address; `10809` is a timeout in seconds; `/dev/nbd0` is ignored
- B) `127.0.0.1` is the NBD server to connect to; `10809` is the port it's listening on; `/dev/nbd0` is the local device node the connection will be attached to
- C) All three arguments must be identical across every `nbd-client` invocation
- D) `modprobe nbd` is unnecessary; `nbd-client` alone handles kernel module loading

<details>
<summary>Answer</summary>

**B** — `nbd-client SERVER PORT DEVICE` connects to the specified NBD server on the specified port, attaching the connection to the given local block device node. `modprobe nbd` is a separate, necessary step to load the kernel module that provides `/dev/nbdN` devices in the first place.

Trap: A misassigns what each argument means; C invents a constraint that doesn't exist (different connections legitimately use different servers/ports/devices); D wrongly claims `nbd-client` handles module loading itself.

</details>

---

**Q5.** You run `sudo fsck -f /dev/sdb1` while `/dev/sdb1` is still mounted, without unmounting it first. What actually happens on most systems?

- A) `fsck` proceeds anyway and silently corrupts the filesystem with no warning
- B) `fsck` detects the filesystem is mounted and refuses to run destructively, printing a warning instead of proceeding
- C) The system immediately reboots
- D) `fsck` automatically unmounts the filesystem for you before proceeding

<details>
<summary>Answer</summary>

**B** — Most modern `fsck` implementations detect a mounted target and refuse to run destructively, warning instead. This refusal is a helpful safety net, but it should never be the reason you skip unmounting first — always treat unmounting first as the correct habit regardless of what a given implementation happens to do.

Trap: A describes the exact danger this refusal exists to prevent, not what actually (typically) happens; C and D both invent behaviors `fsck` doesn't have.

</details>

---

**Q6.** An `autofs` map uses `soft` for an NFS mount. Why are write operations specifically riskier than read operations under a `soft` timeout?

- A) Writes are not actually riskier — reads and writes are equally affected
- B) A write already in progress may be only partially applied by the time the timeout fires, with no guarantee the application knows to retry or verify what actually landed on disk — a read can simply be retried from scratch with no such ambiguity
- C) `soft` mounts don't support write operations at all
- D) Writes under `soft` mounts are queued indefinitely until the server returns, identical to `hard` mount behavior

<details>
<summary>Answer</summary>

**B** — The asymmetry is about partial completion: an interrupted read has nothing to lose (just retry it), but an interrupted write may have partially landed, and the timeout gives the application no clear signal about exactly what state the data is actually in.

Trap: A dismisses a real, specific asymmetry; C is factually wrong — `soft` mounts do support writes, just with this risk; D describes `hard` behavior, the opposite of what `soft` actually does.

</details>

---

**Q7.** Why can't you run `fsck` on the root filesystem (`/`) the same way you would on `/mnt/labdata` — by simply unmounting it first?

- A) The root filesystem doesn't support `fsck` at all
- B) You can't unmount `/` while the system is running from it — checking the root filesystem instead relies on the automatic fsck-on-boot pass (via the fstab pass-number field) or booting into rescue/single-user mode
- C) `fsck` on the root filesystem requires a special `--root` flag not needed elsewhere
- D) The root filesystem is always `xfs`, which cannot be checked with `fsck` under any circumstances

<details>
<summary>Answer</summary>

**B** — A running system can't unmount the filesystem it's actively running from, which is exactly why root filesystem checks rely on different mechanisms entirely: the automatic check at boot time (before the system fully starts using it) or a rescue/single-user environment where it isn't the active root.

Trap: A is wrong — the root filesystem is checked, just via different mechanisms; C invents a flag that doesn't exist; D wrongly assumes a fixed filesystem type and an absolute restriction that isn't accurate (xfs has its own separate repair tool, but that's a different point than what's being asked here).

</details>

---

**Q8.** An `/etc/exports` line uses `sync` (the default emphasized in this demo). What would change if it used `async` instead?

- A) `async` mounts are read-only; `sync` allows writes
- B) `async` acknowledges a write faster, before it's necessarily committed to stable storage — risking loss of a recently-acknowledged write if the server crashes before actually flushing it; `sync` waits for the commit before acknowledging, which is slower but safer
- C) `async` and `sync` only affect how quickly `showmount -e` returns results
- D) There is no functional difference; `async` is simply the newer name for the same behavior

<details>
<summary>Answer</summary>

**B** — `sync` confirms a write only after it's actually durable on disk; `async` confirms it sooner, purely for speed, at the cost of a window where an acknowledged write could still be lost to a server crash before it's truly flushed. This is exactly why `sync` is the safer default used throughout this demo.

Trap: A invents an unrelated read/write restriction; C misapplies the option to an unrelated command; D wrongly claims equivalence between two options with a real, documented behavioral difference.

</details>

---

Score guide:

| Score | Action |
|---|---|
| 7/8 or 8/8 | Import Anki cards, move to Demo 17 |
| 6/8 | Review the wrong answers, then proceed |
| 5/8 | Re-read the relevant sections, retry those questions |
| Below 5/8 | Re-read the full demo and redo the walkthrough before proceeding |
````