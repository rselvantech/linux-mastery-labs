# Demo 15 — LVM (Logical Volume Management)

## Overview

Demo 14 partitioned and mounted storage the direct way — one partition, one
filesystem, fixed size. LVM adds a flexible layer between raw disks and
filesystems: volumes that can grow, shrink, span multiple physical disks, and
be snapshotted, all without unmounting or reformatting. This is how
production storage actually gets managed when "just resize the partition"
isn't an option because the data's already there and the service can't go
down.

## Demo Objectives

1. Explain the PV → VG → LV hierarchy and why each layer exists
2. Create physical volumes, a volume group, and a logical volume
3. Extend a logical volume and its filesystem live, without unmounting
4. Create and use an LVM snapshot, and explain what it actually captures
5. Understand thin provisioning at a conceptual level and its risk tradeoff
6. Reduce a logical volume safely, and explain why this is riskier than extending

## Directory Structure

```
15-lvm/
└── README.md   # This file — full demo content, including Anki (Appendix A)
                 # and Quiz (Appendix B), extracted later via script.
```

## Environments

| Task | Ubuntu 24.04 | Rocky Linux 9.8 | EC2 Amazon Linux |
|---|---|---|---|
| All lab tasks | ✅ (use scratch/lab disks) | ✅ (Rocky9's default install already uses LVM for root — extra caution) | Not required |

## Recall Check — 14-storage-fundamentals

Answer these from memory before continuing — no looking back at Demo 14.

**Q1.** `df` shows a partition at 95% used; `du -sh` accounts for only half
that. What's the most likely cause, and what tool identifies the specific
process responsible?

**Q2.** Why is UUID preferred over a device path like `/dev/sdb1` in
`/etc/fstab`?

**Q3.** When would you choose `parted` over `fdisk`?

<details>
<summary>Answers</summary>

**A1.** A deleted file still held open by a running process — `df` counts it
as used since the filesystem hasn't reclaimed the space, while `du` only
sees files currently visible in the directory tree. `lsof` (looking for open
handles on deleted files) identifies the responsible process.

**A2.** Device names like `/dev/sdb` aren't guaranteed stable across
reboots — disk enumeration order can shift. A filesystem's UUID is generated
once and doesn't change.

**A3.** When scripting partition creation non-interactively, or working with
GPT and disks larger than 2TB, where `fdisk`'s older MBR-era assumptions
become limiting.

</details>

## Part 1 — Concepts

### 1.1 The PV → VG → LV hierarchy

LVM has three layers, each solving a different problem:

```
Physical Volume (PV)  →  Volume Group (VG)  →  Logical Volume (LV)
   raw disk/partition       pool of storage        the thing you format
                             from one or more            and mount
                             PVs
```

A **Physical Volume** is a raw disk or partition, marked for LVM use. A
**Volume Group** pools one or more PVs into a single space — this is the
layer that lets storage span multiple physical disks transparently. A
**Logical Volume** is a slice of a VG's pooled space, and it's the LV that
actually gets a filesystem and a mount point — everything from Demo 14
(`mkfs`, `mount`, `/etc/fstab`) applies to an LV exactly the same way it
applies to a raw partition.

The entire point of this indirection: an LV can be resized (within the VG's
available space) without caring which physical disk(s) back it, and a VG can
have a new PV added to grow its available space — neither operation requires
unmounting the filesystem or touching the partition table the way resizing a
plain partition would.

### 1.2 Creating the stack

```bash
sudo pvcreate /dev/sdb /dev/sdc
sudo pvs
# PV         VG   Fmt  Attr PSize   PFree
# /dev/sdb        lvm2 ---  20.00g  20.00g
# /dev/sdc        lvm2 ---  20.00g  20.00g

sudo vgcreate data-vg /dev/sdb /dev/sdc
sudo vgs
# VG      #PV #LV #SN Attr   VSize  VFree
# data-vg   2   0   0 wz--n- 40.00g 40.00g

sudo lvcreate -L 10G -n app-data data-vg
sudo lvs
# LV       VG      Attr       LSize
# app-data data-vg -wi-a----- 10.00g

sudo mkfs.ext4 /dev/data-vg/app-data
sudo mkdir -p /mnt/app-data
sudo mount /dev/data-vg/app-data /mnt/app-data
```

Note the VG already pooled both disks (`40.00g` from two `20.00g` PVs) before
the LV was even created — the LV creation step (`-L 10G`) just claims a slice
of that pooled space, with `30.00g` remaining free in the VG for future LVs
or extension.

### 1.3 Extending — live, no unmount required

```bash
sudo lvextend -L +5G /dev/data-vg/app-data
# +5G: add 5G to the current size (relative), rather than specifying an
# absolute new total size.

sudo resize2fs /dev/data-vg/app-data
# For ext4 — grows the filesystem to fill the LV's new size.
# For xfs: sudo xfs_growfs /mnt/app-data (operates on the mountpoint, not
# the device — an xfs-specific difference worth remembering).

df -h /mnt/app-data
# Confirm the filesystem itself, not just the LV, now reflects the new size.
```

**Two separate steps, and both matter:** `lvextend` grows the logical volume
itself; the filesystem sitting on top doesn't automatically know about the
new space until `resize2fs`/`xfs_growfs` tells it to grow into it. Running
only `lvextend` and stopping there is a common incomplete-fix mistake — `df`
still shows the old size until the filesystem resize step actually runs.

```bash
sudo lvextend -L +5G -r /dev/data-vg/app-data
# -r: automatically run the matching filesystem resize command as part of
# lvextend itself — combines both steps into one, and is the safer default
# for ext4/xfs since it removes the chance of forgetting the second step.
```

### 1.4 Snapshots — what they actually capture

```bash
sudo lvcreate -L 2G -s -n app-data-snap /dev/data-vg/app-data
# -s: create a snapshot instead of a regular LV
# -L 2G: the snapshot's own allocated space — NOT the size of the data being
# snapshotted, see below
```

**A traditional LVM snapshot is not a full copy.** It only stores the
*changes* made to the origin volume after the snapshot was taken (a
copy-on-write mechanism) — the `-L 2G` size is how much space is reserved to
hold those changes, not a copy of the entire 10-15GB origin volume. This has
a direct, sharp consequence: **if the origin volume changes more than the
snapshot's allocated space can hold, the snapshot becomes invalid and is
automatically dropped** — silently losing your rollback point exactly when a
lot of writes have happened, which is often precisely when you'd have wanted
it most (e.g., during a risky migration or bulk update).

```bash
sudo lvs
# Check the Snap% column periodically during any operation you're using a
# snapshot to protect against — approaching 100% means the snapshot is
# about to be invalidated.
```

```bash
# Rolling back to the snapshot (reverts the origin to the snapshot's state):
sudo umount /mnt/app-data
sudo lvconvert --merge /dev/data-vg/app-data-snap
# The merge completes on next activation (may require reactivating the LV
# or a reboot for the origin volume, depending on whether it's currently
# in use) — this operation destroys the changes made since the snapshot,
# by design, which is the entire point of a rollback.
```

### 1.5 Thin provisioning — a conceptual introduction

Everything above is "thick" provisioning — an LV's declared size is fully
reserved from the VG immediately, whether or not the filesystem is actually
using that space yet. **Thin provisioning** creates a pool where LVs can be
allocated *more total space than the pool physically has*, on the bet that
not every volume will actually use its full declared size at once:

```bash
sudo lvcreate -L 20G -T data-vg/thin-pool
sudo lvcreate -V 15G -T data-vg/thin-pool -n thin-vol1
sudo lvcreate -V 15G -T data-vg/thin-pool -n thin-vol2
# Both volumes declare 15G (30G total) against a pool that only has 20G
# physically backing it — this is the entire point of thin provisioning:
# overcommitting on the bet that actual usage stays below declared size.
```

**The risk this trades in return for flexibility:** if actual usage across
all thin volumes together approaches the pool's real physical size, the pool
fills up regardless of what each volume's *declared* size claims — and
running out of space in a thin pool is a more dangerous failure mode than a
regular full filesystem, since multiple volumes can be affected
simultaneously. Thin provisioning is a real production tool (common in
virtualization and container storage backends), but it requires active
monitoring of pool utilization specifically — this demo doesn't build a full
thin-provisioning lab, but recognizing what it trades away is the practical
takeaway.

### 1.6 Reducing — why it's riskier than extending

```bash
sudo umount /mnt/app-data
sudo e2fsck -f /dev/data-vg/app-data
sudo resize2fs /dev/data-vg/app-data 8G
sudo lvreduce -L 8G /dev/data-vg/app-data
sudo mount /dev/data-vg/app-data /mnt/app-data
```

**Notice the order is reversed from extending, and this is not
arbitrary.** When extending, growing the LV first and the filesystem second
is safe because the filesystem is simply told about newly available space it
wasn't previously using. When reducing, the filesystem must be shrunk
*first* — while it still has its original full space to safely relocate any
data currently living in the blocks that are about to be removed — and only
*then* can the LV itself be shrunk to match. Reducing the LV before the
filesystem, or reducing it to a size smaller than the filesystem was
actually shrunk to, risks destroying data that the filesystem hadn't
finished relocating out of the truncated space. `xfs` cannot be shrunk at
all (per Demo 14 §1.3) — reducing an `xfs`-backed LV requires backing up,
recreating the filesystem at the smaller size, and restoring, not an
in-place shrink.

## Part 2 — Hands-On Lab

### Commands used in this demo

| Command | Flag meaning | Purpose |
|---|---|---|
| `pvcreate` | — | Mark a disk/partition as an LVM physical volume |
| `vgcreate NAME PV...` | — | Create a volume group from one or more PVs |
| `lvcreate -L SIZE -n NAME VG` | `-L`: size, `-n`: name | Create a logical volume |
| `lvextend -L +SIZE -r LV` | `+SIZE`: relative, `-r`: also resize filesystem | Extend an LV and its filesystem in one step |
| `resize2fs` / `xfs_growfs` | — | Grow a filesystem to fill its LV (if not using `-r`) |
| `lvcreate -s -n NAME -L SIZE ORIGIN` | `-s`: snapshot | Create an LVM snapshot |
| `lvconvert --merge SNAP` | — | Roll back origin to a snapshot's state |
| `lvreduce -L SIZE LV` | — | Shrink an LV (only after shrinking the filesystem first) |

### Task 1 — Build the PV → VG → LV stack

```bash
sudo pvcreate /dev/sdb /dev/sdc
sudo pvs
sudo vgcreate demo-vg /dev/sdb /dev/sdc
sudo vgs
sudo lvcreate -L 5G -n demo-lv demo-vg
sudo lvs
sudo mkfs.ext4 /dev/demo-vg/demo-lv
sudo mkdir -p /mnt/demo-lv
sudo mount /dev/demo-vg/demo-lv /mnt/demo-lv
df -h /mnt/demo-lv
```

### Task 2 — Extend live, and observe the two-step trap

```bash
sudo lvextend -L +2G /dev/demo-vg/demo-lv
sudo lvs   # LV now shows 7G
df -h /mnt/demo-lv   # filesystem still shows ~5G — the trap in action
sudo resize2fs /dev/demo-vg/demo-lv
df -h /mnt/demo-lv   # now reflects 7G
```

### Task 3 — Snapshot, modify, and roll back

```bash
sudo lvcreate -L 1G -s -n demo-lv-snap /dev/demo-vg/demo-lv
echo "important data before changes" | sudo tee /mnt/demo-lv/testfile.txt
echo "accidental overwrite" | sudo tee /mnt/demo-lv/testfile.txt
cat /mnt/demo-lv/testfile.txt

sudo umount /mnt/demo-lv
sudo lvconvert --merge /dev/demo-vg/demo-lv-snap
sudo lvchange -an /dev/demo-vg/demo-lv
sudo lvchange -ay /dev/demo-vg/demo-lv
sudo mount /dev/demo-vg/demo-lv /mnt/demo-lv
cat /mnt/demo-lv/testfile.txt
# Should show the original content — the merge reverted the overwrite.
```

### Task 4 — Watch snapshot space usage

```bash
sudo lvcreate -L 200M -s -n demo-lv-snap2 /dev/demo-vg/demo-lv
sudo lvs   # note demo-lv-snap2's Snap% starts near 0

dd if=/dev/urandom of=/mnt/demo-lv/bigchange bs=1M count=250
sudo lvs   # check Snap% again — likely near or at 100%, given the snapshot
           # is smaller than the amount of change written
sudo lvremove -f /dev/demo-vg/demo-lv-snap2
rm /mnt/demo-lv/bigchange
```

### Task 5 — Reduce safely (unmount → fsck → shrink fs → shrink LV)

```bash
sudo umount /mnt/demo-lv
sudo e2fsck -f /dev/demo-vg/demo-lv
sudo resize2fs /dev/demo-vg/demo-lv 4G
sudo lvreduce -L 4G /dev/demo-vg/demo-lv
sudo mount /dev/demo-vg/demo-lv /mnt/demo-lv
df -h /mnt/demo-lv
```

### Task 6 — Thin pool creation (conceptual, minimal)

```bash
sudo lvcreate -L 3G -T demo-vg/thin-pool
sudo lvcreate -V 2G -T demo-vg/thin-pool -n thin-demo1
sudo lvs
# Note Data% on the pool — this is the number to watch, not each volume's
# declared -V size.
```

## What You Learned

1. ✅ Explained the PV → VG → LV hierarchy and why each layer exists
2. ✅ Created physical volumes, a volume group, and a logical volume
3. ✅ Extended a logical volume and its filesystem live, without unmounting
4. ✅ Created and used an LVM snapshot, and what it actually captures
5. ✅ Understood thin provisioning conceptually and its risk tradeoff
6. ✅ Reduced a logical volume safely, in the correct order

## Part 3 — Break-Fix Scenario

### Scenario — NovaTech Ops: two LVM incidents

> **Incident report — NovaTech Ops, ticket #4589**
>
> Alert 1: "extended the app-data LV before a migration, but `df` still
> shows the old size — migration script failed assuming more space was
> available." Alert 2: "pre-migration snapshot vanished mid-migration, and
> now we can't roll back."

**Incident 1 — extended the LV, df still shows old size**

```bash
$ sudo lvs
LV       VG      LSize
app-data data-vg 15.00g
$ df -h /mnt/app-data
Filesystem                  Size  Used Avail Use% Mounted on
/dev/data-vg/app-data        10G  9.5G   50M  95%  /mnt/app-data
```

Tasks:
1. `lvs` confirms the LV is 15G, but `df` still shows 10G. Reference §1.3 —
   what step was missed?
2. Why does this specific mistake look, at a glance, like the extend
   operation itself failed, when it actually succeeded?
3. Fix it, and state the single-command alternative from §1.3 that would
   have prevented this exact incident.

**Incident 2 — snapshot vanished mid-migration**

```bash
$ sudo lvs
LV              VG      Attr       LSize  Snap%
app-data        data-vg owi-aos---  15.00g
$ # app-data-snap is no longer listed at all
```

Tasks:
1. Given §1.4, what's the most likely explanation for a snapshot
   disappearing entirely during a migration (a period of heavy writes)?
2. Was the snapshot's allocated size a factor here, and if so, how should it
   have been sized differently before starting a migration?
3. What does this incident reveal about relying on a snapshot as a rollback
   plan for an operation you know will involve heavy writes?

<details>
<summary>Answers</summary>

**Incident 1:** The filesystem resize step (`resize2fs`) was never run —
`lvextend` grew the logical volume itself, but the filesystem sitting on top
doesn't automatically grow into that new space; it needs to be told to via
`resize2fs` (or `xfs_growfs` for xfs). This looks like the extend failed at
a glance because the person checking `df` (filesystem-level, per Demo 14)
rather than `lvs` (LV-level) sees no change at all — the two tools are
reporting on two different layers, and only one of them reflects what
actually happened. Fix: `sudo resize2fs /dev/data-vg/app-data`. The
preventive single-command alternative: `lvextend -L +5G -r
/dev/data-vg/app-data` — the `-r` flag runs the matching filesystem resize
automatically as part of the same command, removing the chance of
forgetting the second step entirely.

**Incident 2:** The snapshot was very likely undersized relative to the
volume of changes written during the migration — per §1.4, a traditional
LVM snapshot only reserves space for the *changes* made after it was taken,
and if those changes exceed the snapshot's allocated size, it's
automatically invalidated and dropped, silently losing the rollback point.
Yes, sizing was almost certainly the factor: a snapshot intended to protect
a heavy-write operation like a migration needs to be sized generously
relative to the expected volume of changes, not just a small fraction of the
origin's size as might be reasonable for a lighter-touch operation. This
incident reveals a real limitation of traditional LVM snapshots as a
rollback plan for heavy-write operations specifically — they need active
`Snap%` monitoring during the operation itself (§1.4), and for very
write-heavy migrations, a full backup or a differently-sized/thin-pool-based
approach may be a more reliable rollback mechanism than a thinly-sized
traditional snapshot.

</details>

## Interview Prep

**Q1. Why does LVM insert PV and VG layers instead of just letting a logical volume map directly to a disk?**
The indirection is what makes resizing and multi-disk pooling possible
without touching partition tables or unmounting filesystems. A VG pools one
or more PVs into a single space, so an LV drawn from that VG can be resized
within available VG space, or the VG itself can grow by adding a new PV —
neither requires the kind of destructive, offline operation that resizing a
raw partition typically does.

**Q2. An LV was extended, but the application still reports the volume as full. What's the most common cause, and what's the safer way to avoid it next time?**
The filesystem resize step was skipped — `lvextend` grows the logical volume
itself, but the filesystem doesn't automatically expand into that new space
without `resize2fs`/`xfs_growfs` explicitly telling it to. The safer
approach is `lvextend -L +SIZE -r`, which combines both steps into one
command and removes the chance of forgetting the filesystem resize.

**Q3. Why can a snapshot disappear entirely during a heavy-write operation, and what does that mean for using snapshots as a rollback safety net?**
A traditional LVM snapshot only reserves space for the changes made to the
origin after it was taken, not a full copy — if those changes exceed the
snapshot's allocated size, it's automatically invalidated and dropped. This
means a snapshot's reliability as a rollback plan is directly tied to
correctly estimating (generously) how much change an operation will produce
before starting it, and actively monitoring `Snap%` during the operation —
not a "set it and forget it" safety net for genuinely heavy-write scenarios.

## Key Takeaways

1. **PV → VG → LV exists specifically to decouple logical volumes from the
   physical disks backing them.** This is what enables live resizing and
   multi-disk pooling without the destructive operations raw partition
   resizing usually requires.
2. **Extending an LV is two steps, and skipping the second is the most
   common LVM mistake.** `lvextend` grows the LV; the filesystem needs its
   own resize (`resize2fs`/`xfs_growfs`) — or use `lvextend -r` to do both
   at once.
3. **A traditional LVM snapshot stores only changes since it was taken, not
   a full copy.** If changes exceed its allocated size, it's automatically
   and silently invalidated — size snapshots generously for any operation
   involving heavy writes, and monitor `Snap%` during it.
4. **Reducing an LV requires the opposite order from extending: shrink the
   filesystem first, then the LV.** Reversing this order, or under-shrinking
   the filesystem relative to the LV's new size, risks data loss.
5. **Thin provisioning trades guaranteed space for flexibility.** Declaring
   more total volume size than a pool physically has works as long as actual
   usage stays below the pool's real capacity — but a full thin pool is a
   more dangerous failure mode than a single full filesystem, since it can
   affect multiple volumes simultaneously.
6. **`xfs` cannot be shrunk in place at all**, consistent with Demo 14's
   comparison — reducing an xfs-backed LV means backup, recreate smaller,
   restore, not a live shrink like ext4 supports.

> **Demo scope:** Primary concept: flexible logical volume management with
> LVM. Supporting concepts: PV/VG/LV hierarchy, live extension, snapshots and
> their copy-on-write limitation, thin provisioning tradeoffs, safe reduction
> ordering.
> Estimated completion time: 29 minutes (reading + hands-on + verification).
> Checkpoints: 6 natural stopping points (after Task 1 through Task 6).

## Quick Commands Reference

### Building the stack

| Command | Purpose |
|---|---|
| `pvcreate /dev/sdX` | Mark a device as an LVM physical volume |
| `vgcreate VG PV...` | Create a volume group |
| `lvcreate -L SIZE -n NAME VG` | Create a logical volume |

### Resizing

| Command | Purpose |
|---|---|
| `lvextend -L +SIZE -r LV` | Extend LV and filesystem together (safer default) |
| `resize2fs LV` | Grow ext4 filesystem to fill LV (if not using `-r`) |
| `xfs_growfs MOUNTPOINT` | Grow xfs filesystem (operates on mountpoint, not device) |
| `lvreduce -L SIZE LV` | Shrink LV — only after shrinking the filesystem first |

### Snapshots and thin provisioning

| Command | Purpose |
|---|---|
| `lvcreate -s -n NAME -L SIZE ORIGIN` | Create a snapshot |
| `lvconvert --merge SNAP` | Roll back origin to snapshot state |
| `lvs` (check `Snap%`) | Monitor snapshot space usage before it's invalidated |
| `lvcreate -L SIZE -T VG/POOL` | Create a thin pool |
| `lvcreate -V SIZE -T VG/POOL -n NAME` | Create a thin volume from the pool |

## Appendix A — Anki Cards

**Note:** extracted into a standalone `.csv` by the series extraction script.

```
#deck:Linux Mastery::Module 3 - Networking & Storage::15-lvm
#separator:Comma
#columns:Front,Back,Tags
"lvextend grows an LV, but df on the mounted filesystem still shows the old size. What was missed?","The filesystem resize step. lvextend only grows the logical volume itself — the filesystem needs resize2fs (ext4) or xfs_growfs (xfs) to actually expand into the new space, unless lvextend was run with -r to do both at once.","demo15,lvm,lvextend,common-mistake"
"What does a traditional LVM snapshot actually store — a full copy of the origin, or something else?","Only the changes made to the origin volume after the snapshot was taken (copy-on-write). The -L size at creation reserves space for those changes, not a copy of the entire origin.","demo15,lvm,snapshot,fundamentals"
"What happens if changes to an LVM snapshot's origin exceed the snapshot's allocated size?","The snapshot is automatically invalidated and dropped, silently losing the rollback point — often during exactly the kind of heavy-write operation you'd have wanted it most for.","demo15,lvm,snapshot,common-mistake"
"Why does reducing an LV require shrinking the filesystem BEFORE the LV, the opposite order from extending?","Extending is safe growing-LV-first because the filesystem is just told about new unused space. Reducing must shrink the filesystem first while it still has room to safely relocate data out of the blocks about to be removed — reversing this order risks destroying data.","demo15,lvm,lvreduce,safety"
"What does the PV to VG to LV hierarchy actually enable that a raw partition can't?","Live resizing and multi-disk pooling without unmounting the filesystem or touching the partition table — an LV can be resized within the VG's available space, and a VG can grow by adding a new PV.","demo15,lvm,fundamentals"
"What is thin provisioning, and what risk does it trade for its flexibility?","Creating a pool where LVs can be allocated more total declared space than the pool physically has, betting that actual usage stays below declared size. The risk: if real usage across all thin volumes approaches the pool's actual capacity, the pool fills up regardless of declared sizes, and multiple volumes can be affected simultaneously — a more dangerous failure mode than a single full filesystem.","demo15,lvm,thin-provisioning"
"Can an xfs-backed logical volume be shrunk in place the way an ext4-backed one can?","No — xfs cannot be shrunk in place at all. Reducing an xfs-backed LV requires backing up the data, recreating the filesystem at the smaller size, and restoring.","demo15,lvm,xfs,cross-demo"
"What does lvextend -r do differently from a plain lvextend -L +SIZE?","-r automatically runs the matching filesystem resize command (resize2fs or xfs_growfs) as part of the same lvextend operation, combining both required steps into one and removing the chance of forgetting the filesystem resize.","demo15,lvm,lvextend"
"During a migration expected to involve heavy writes, why might a snapshot sized at a small fraction of the origin volume's size be a poor rollback plan?","Because the snapshot only reserves space for changes, not a full copy — if the migration's writes exceed that reserved space, the snapshot is automatically invalidated mid-operation, exactly when it would be needed most.","demo15,lvm,snapshot,troubleshooting"
"What column in lvs output should be actively monitored during an operation a snapshot is meant to protect against?","Snap% — approaching 100% means the snapshot is close to exceeding its allocated space and being automatically invalidated.","demo15,lvm,snapshot,monitoring"
```

## Appendix B — Quiz

**Note:** extracted into a standalone `.md` by the series extraction script.

> One correct answer per question unless stated otherwise.
> Target: 80% or above before moving to Demo 16.

**Q1. `lvextend` successfully grows an LV, but `df` on the mounted filesystem still shows the old size. What was missed?**

- A) The extend operation actually failed silently
- B) The filesystem resize step (`resize2fs`/`xfs_growfs`), which doesn't happen automatically
- C) A reboot is required after any `lvextend`
- D) `df` caches its output and needs `df --refresh`

<details>
<summary>Answer</summary>

**B** — `lvextend` grows the LV itself; the filesystem needs a separate resize step (or `lvextend -r` to combine both) to actually use the new space. Trap: A misreads the evidence — `lvs` would confirm the LV did grow; C and D invent requirements/flags that don't exist.

</details>

---

**Q2. What does a traditional LVM snapshot actually store?**

- A) A complete, full copy of the origin volume at creation time
- B) Only the changes made to the origin after the snapshot was taken (copy-on-write)
- C) A compressed archive of the entire origin volume
- D) Metadata only, with no actual data storage

<details>
<summary>Answer</summary>

**B** — This is the core mechanism, and its direct consequence is that undersizing a snapshot relative to expected changes risks its automatic invalidation. Trap: A and C both overstate what's actually captured; D understates it — real change data is stored, not just metadata.

</details>

---

**Q3. Changes to an LVM snapshot's origin exceed the snapshot's allocated space during a migration. What happens?**

- A) The snapshot automatically grows to accommodate the extra changes
- B) The snapshot is automatically invalidated and dropped
- C) The migration is automatically paused until the snapshot is resized
- D) Nothing — snapshots have unlimited capacity for changes by default

<details>
<summary>Answer</summary>

**B** — This silent invalidation is exactly why snapshots need generous sizing and active monitoring (`Snap%`) for any heavy-write operation. Trap: A, C, and D all invent safety behaviors that traditional LVM snapshots don't actually have.

</details>

---

**Q4. Why must a filesystem be shrunk before the LV when reducing, the opposite order from extending?**

- A) It's an arbitrary tool limitation with no underlying reason
- B) The filesystem needs its full original space available to safely relocate data out of blocks that are about to be removed
- C) LVM physically requires filesystems to be smaller than their LV at all times
- D) Reducing the LV first is actually fine; the order doesn't matter

<details>
<summary>Answer</summary>

**B** — Reversing the order (or under-shrinking the filesystem relative to the new LV size) risks destroying data still living in blocks that get removed before the filesystem finished relocating it. Trap: A dismisses a real safety reason as arbitrary; C invents a permanent constraint; D directly contradicts the correct, safety-critical ordering.

</details>

---

**Q5. What does the PV → VG → LV hierarchy actually enable that a raw partition setup doesn't?**

- A) Faster read/write speeds in all cases
- B) Live resizing and multi-disk pooling without unmounting or touching the partition table
- C) Automatic RAID redundancy by default
- D) Elimination of the need for a filesystem entirely

<details>
<summary>Answer</summary>

**B** — This is LVM's actual value proposition: the VG pooling layer decouples logical volumes from specific physical disks, enabling resize and multi-disk operations that would otherwise require destructive, offline steps. Trap: A, C, and D all invent capabilities LVM doesn't provide by default.

</details>

---

**Q6. What risk does thin provisioning trade for its flexibility?**

- A) Thin volumes are always slower than thick volumes
- B) If real usage across thin volumes approaches the pool's actual physical capacity, the pool fills regardless of each volume's declared size, potentially affecting multiple volumes at once
- C) Thin provisioning requires a commercial LVM license
- D) Thin volumes cannot be snapshotted

<details>
<summary>Answer</summary>

**B** — This overcommit risk, and its multi-volume blast radius when it materializes, is the core tradeoff thin provisioning makes for its space-flexibility benefit. Trap: A, C, and D all invent unrelated constraints or costs.

</details>

---

**Q7. Can an `xfs`-backed logical volume be shrunk in place?**

- A) Yes, using `xfs_growfs -s`
- B) No — `xfs` cannot be shrunk in place at all; shrinking requires backup, recreate smaller, restore
- C) Yes, but only by an amount equal to the original size
- D) Only if the LV was created with a special `--shrinkable` flag

<details>
<summary>Answer</summary>

**B** — This is a hard limitation of `xfs` consistent with Demo 14's ext4-vs-xfs comparison. Trap: A invents a flag that doesn't exist for shrinking; C and D invent constraints that aren't real.

</details>

---

**Q8. What does `lvextend -L +5G -r /dev/vg/lv` do differently from `lvextend -L +5G /dev/vg/lv` alone?**

- A) `-r` makes the extend operation reversible
- B) `-r` automatically runs the matching filesystem resize as part of the same command
- C) `-r` restricts the extension to read-only volumes
- D) `-r` is required for any LV backed by more than one PV

<details>
<summary>Answer</summary>

**B** — `-r` combines the LV extension and the filesystem resize into one command, removing the chance of forgetting the filesystem step — the single most common LVM extension mistake. Trap: A, C, and D all invent meanings for the flag that aren't correct.

</details>