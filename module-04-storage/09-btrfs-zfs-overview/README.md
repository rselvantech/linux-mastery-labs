# Demo M04-09 — Btrfs/ZFS Conceptual Overview

> **A note on this demo's structure:** unlike every other demo in this
> module, this one is explicitly conceptual per the curriculum's own
> scoping — no hands-on Lab, no command-based Break-Fix. Part 2 below is
> a decision-framework exercise (reasoning through scenarios), not a
> shell walkthrough, and Part 3 is scenario-based reasoning rather than
> a live-command incident. This is a deliberate, flagged deviation from
> this module's usual skeleton, not an omission. Facts about current
> (2026) distro defaults, licensing status, and RAID stability below
> were verified via web search rather than pulled from memory, since
> this is exactly the kind of landscape that shifts year to year.

## Overview

Every filesystem covered so far in this module — ext4, xfs (Demo M04-01),
LVM on top of either (Demo M04-04) — shares one foundational trait: writes
happen **in place**. Modifying a file means overwriting the actual disk
blocks that already hold its data. Btrfs and ZFS take a fundamentally
different approach: **copy-on-write (CoW)**. A write never overwrites
existing data in place — it writes to a new location, then atomically
updates a pointer to reference the new location instead of the old one.
This one design decision is the root cause of almost every practical
difference between these filesystems and everything covered so far:
instant snapshots, built-in checksumming, and a genuinely different
recovery story after data corruption.

This demo covers what CoW actually changes, mechanically, and gives you
a real, current (2026) decision framework for when either filesystem is
worth reaching for over the ext4/xfs+LVM stack this series has built on
so far — including the licensing and support-status realities that
matter as much as the technical trade-offs.

## Demo Objectives

1. Explain what copy-on-write actually changes about how a write
   happens, in contrast to ext4/xfs's in-place overwrite model
2. Explain why CoW makes snapshots effectively instant and cheap, in
   contrast to LVM's snapshot mechanism (Demo M04-04)
3. Explain built-in checksumming and self-healing, and how that differs
   from relying on RAID (Demo M04-05) or hardware alone to catch corruption
4. State the Btrfs vs. ZFS licensing situation precisely, and its real,
   current practical consequence for which distros ship which by default
5. State the current production-readiness status of Btrfs's parity RAID
   modes (5/6) vs. ZFS's RAID-Z, and why this specific gap has persisted
6. Choose the right filesystem — ext4/xfs+LVM, Btrfs, or ZFS — for a
   given scenario, based on the actual constraints in play

## Directory Structure

```
09-btrfs-zfs-overview/
├── README.md                           # this file
├── m04-09-btrfs-zfs-overview-anki.csv  # Anki flashcard deck
└── m04-09-btrfs-zfs-overview-quiz.md   # standalone quiz
```

## Environments

| Task | Ubuntu 24.04 | Rocky Linux 9.8 | EC2 Amazon Linux |
|---|---|---|---|
| Conceptual only — no hands-on lab | Btrfs is in-kernel and installable (`btrfs-progs`); ZFS available via Ubuntu's installer, marked experimental since 24.04 LTS | **Btrfs is not supported at all** — removed from RHEL since version 7.4, and Rocky Linux tracks RHEL 1:1 on this; ZFS available via a separate OpenZFS-compatible repository, not in the base distro | Not applicable |

## Recall Check — 08-disk-quotas

Answer these from memory before continuing — no looking back at Demo M04-08.

1. What's the most common real-world mistake when enabling quotas —
   the step people skip after editing `/etc/fstab`?

2. A user's usage is well under their hard limit, but every write fails.
   What's the most likely explanation?

3. What's the difference between a block limit and an inode limit?

<details>
<summary>Answers</summary>

1. Running `quotacheck` — the fstab edit and remount alone don't create
   the `aquota.user`/`aquota.group` accounting files that tracking and
   enforcement actually depend on.

2. Their grace period on the *soft* limit has expired — once that
   happens, the soft limit is enforced as if it were a hard limit,
   regardless of remaining headroom before the actual hard limit.

3. Block limits cap total disk space; inode limits cap the number of
   files, regardless of individual file size — a user can exhaust one
   without approaching the other.

</details>

---

## Part 1 — Concepts

### 1.1 Copy-on-write — what actually changes

In ext4 or xfs (Demo M04-01), modifying a file's contents means the
filesystem locates the existing blocks on disk and overwrites them
directly. This is fast and simple, but it has a direct consequence:
there's no free, built-in way to "go back" to the pre-write state — the
old data is gone the moment the new data lands, unless a separate
snapshot mechanism (like LVM's, Demo M04-04) was already in place to
preserve it beforehand.

Btrfs and ZFS never overwrite in place. A write to any existing block
allocates a *new* block, writes the new data there, and then atomically
updates the metadata that points to "the current version" of that data
— the old block isn't touched or freed until nothing references it
anymore. This single mechanical difference is what makes both
filesystems' most distinctive features possible, and it's worth tracing
through deliberately rather than treating "CoW" as a buzzword.

### 1.2 Snapshots — why CoW makes them nearly free

Demo M04-04 covered LVM snapshots: when you create one, LVM starts tracking
which blocks change afterward, copying the *original* block to the
snapshot's reserved space the moment the live volume's data changes —
copy-on-write happening at the block layer, bolted onto a filesystem
that itself has no concept of it.

Btrfs and ZFS don't need this bolted-on mechanism, because CoW is
already how they write, always. A snapshot is just a pointer to the
current metadata tree at that exact moment — creating one is close to
instantaneous and consumes no extra space upfront, because nothing was
actually copied. Space is only consumed later, incrementally, as the
live filesystem and the snapshot diverge (new writes go to new blocks,
per §1.1, so the snapshot's old blocks simply aren't freed yet). This is
why Btrfs subvolume snapshots and ZFS dataset snapshots are cheap enough
to take automatically and frequently (before every package upgrade, for
instance) in a way that would be impractical with LVM's reserved-space
model from Demo M04-04.

### 1.3 Built-in checksumming and self-healing

Neither ext4/xfs nor plain LVM/RAID (Demo M04-05) checksums file data
itself — RAID protects against a *disk failing entirely*, but silent
data corruption on a disk that's still nominally "working" (a bit flip,
a firmware bug) can go completely undetected, since nothing is verifying
that the data read back actually matches what was written.

Btrfs and ZFS both checksum data (and metadata) at write time and verify
that checksum on every read. If a mismatch is found and the filesystem
has redundancy available (a mirrored or parity RAID configuration within
Btrfs/ZFS itself, not a separate mdadm array), it automatically repairs
the bad copy from a good one — this is what "self-healing" means in this
context, and it's a genuinely different capability from anything covered
in Demo M04-05's RAID discussion, which detects a disk being *entirely*
gone, not a single silently corrupted block on a disk that's still
functioning.

### 1.4 Licensing — why this is a practical concern, not just legal trivia

This is the factor that most determines which of these two filesystems
you'll actually encounter in practice, and it's worth understanding
precisely rather than as vague "legal stuff."

Btrfs is licensed under the GPL, the same license as the Linux kernel
itself, and lives directly in the kernel source tree (`fs/btrfs/`). It
gets built, tested, and released as part of the normal kernel release
cycle, on every distro that enables it.

ZFS (specifically OpenZFS, the actively maintained open-source
continuation) is licensed under the CDDL, a license the Free Software
Foundation has stated cannot be linked into the same binary as GPL code.
This is the specific, stated reason ZFS cannot ship as part of the Linux
kernel itself — it must always be built and distributed as a separate,
out-of-tree kernel module (via DKMS, rebuilt against each new kernel
version).

**The real, current (2026) practical consequence:**
- Fedora, openSUSE, and Ubuntu Desktop ship Btrfs as their **default**
  filesystem on a fresh install.
- **Rocky Linux and RHEL do not support Btrfs at all** — it was removed
  from RHEL starting at version 7.4, and Rocky Linux tracks RHEL
  compatibility exactly on this point; there's no supported path to
  Btrfs on this series' second reference distro.
- Ubuntu ships ZFS support in its installer, but marked **experimental**
  since 24.04 LTS — Canonical's position is that shipping ZFS as a
  loadable module sidesteps the CDDL/GPL linking concern, a legal
  position that (as of the sources checked for this demo) has never
  been tested in court.
- Fedora refuses to ship ZFS at all, and Red Hat has taken the same
  position.
- A February 2025 proposal to dual-license OpenZFS under CDDL+GPLv2
  stalled — Oracle (which controls the original ZFS license) never
  responded, and there's been no indication of interest in mainlining it.
- The operational cost of ZFS's out-of-tree status: every kernel upgrade
  is a potential breakage point, since the DKMS-built module has to be
  rebuilt against the new kernel before ZFS works again — a real,
  recurring maintenance burden that Btrfs, living in-tree, doesn't have.

### 1.5 RAID modes — a genuine, persistent gap

Both filesystems can provide RAID-like redundancy natively (distinct
from, and layered independently of, the mdadm RAID covered in Demo M04-05):

| | Btrfs | ZFS |
|---|---|---|
| Mirroring / striping (RAID1/0/10-equivalent) | Solid, production-ready | Solid, production-ready |
| Parity RAID (RAID5/6-equivalent) | **Still not considered production-ready as of 2026** — the "write hole" problem (a crash mid-write to parity can leave data inconsistent in a way the filesystem can't detect or repair) has not been fully resolved | RAID-Z1/Z2/Z3 — mature, trusted, one of the most well-regarded software RAID implementations available on Linux |

This is a specific, persistent, well-documented gap, not a passing
concern — if parity RAID (maximizing usable capacity while tolerating a
disk failure, the RAID5/6 trade-off from Demo M04-05's comparison table)
matters for a given use case, that's a genuine point in ZFS's favor, or
an argument for layering Btrfs on top of a separate mdadm RAID5/6 array
instead of using Btrfs's own parity RAID mode.

### 1.6 Other practical differences worth knowing

| | Btrfs | ZFS |
|---|---|---|
| RAM footprint | Light — fine on modest hardware | Heavy — ZFS's ARC (Adaptive Replacement Cache) uses as much RAM as it's given for caching |
| Encryption | No native encryption — layer LUKS (Demo M04-06) underneath, same as ext4/xfs | Native, per-dataset encryption built in |
| Send/receive (replicating snapshots to another host) | Supported | Supported, and generally considered more polished/mature |

### 1.7 When to reach for which — a decision framework

- **Stay with ext4/xfs + LVM (Demos M04-01/15)** when: the workload doesn't
  need frequent snapshots or built-in corruption detection, the host is
  Rocky/RHEL-family (where Btrfs isn't an option at all), or simplicity
  and broad tooling compatibility matter more than CoW's specific benefits.
- **Reach for Btrfs** when: on a distro where it's already the default
  (Fedora, openSUSE, Ubuntu) and in-kernel simplicity matters, frequent
  cheap snapshots are valuable (pre-upgrade system snapshots are a
  common real use case), and parity RAID isn't a requirement — or Btrfs
  is layered on top of a separate mdadm array if it is.
- **Reach for ZFS** when: parity-RAID-level redundancy at scale is a
  real requirement, the operational overhead of DKMS rebuilds on kernel
  upgrades is acceptable (or the host is something like TrueNAS/Proxmox,
  where ZFS is a first-class, well-supported citizen rather than an
  afterthought), and the RAM cost of the ARC cache is affordable for the
  hardware in play.

## Part 2 — Decision Framework Exercises

**No hands-on lab for this demo** — per this demo's conceptual scoping,
there's no command sequence to run. Instead, work through these
scenarios using §1.1–§1.7 before checking the reasoning below each one.

### Scenario 1
A team runs Rocky Linux exclusively across their fleet and wants
instant, space-efficient snapshots before every configuration change.
Is Btrfs on the table?

<details>
<summary>Reasoning</summary>

No — per §1.4, Btrfs is not supported on Rocky Linux/RHEL at all,
removed since RHEL 7.4. This isn't a "harder to install" situation, it's
"not an available option" on this distro family. LVM snapshots (Demo M04-04)
remain the available mechanism on Rocky, with the trade-offs already
covered there (reserved space allocated upfront, copy-on-write happening
at the block layer rather than being inherent to the filesystem).
ZFS via a separate OpenZFS-compatible repository is the CoW-filesystem
option that IS available on Rocky, if the team specifically wants CoW's
properties and can accept the DKMS/out-of-tree maintenance overhead.

</details>

### Scenario 2
A storage team needs maximum usable capacity from an 8-disk array while
tolerating up to two simultaneous disk failures — the RAID6-equivalent
trade-off from Demo M04-05's comparison table. They're deciding between
Btrfs's native RAID6 mode and ZFS's RAID-Z2.

<details>
<summary>Reasoning</summary>

Per §1.5, this leans clearly toward ZFS's RAID-Z2 — Btrfs's parity RAID
modes are still not considered production-ready as of 2026 due to the
unresolved write-hole problem, while ZFS's RAID-Z implementations are
mature and well-regarded specifically for this use case. If Btrfs is
still preferred for other reasons (in-kernel simplicity, an existing
Fedora/openSUSE/Ubuntu deployment), the alternative is layering Btrfs on
top of a separate mdadm RAID6 array (Demo M04-05) rather than using Btrfs's
own native parity RAID mode.

</details>

### Scenario 3
A single developer workstation running Fedora needs frequent, cheap
snapshots before risky package upgrades, with no need for enterprise
RAID modes, running on a laptop with limited RAM.

<details>
<summary>Reasoning</summary>

Btrfs is the clear fit here, per §1.4, §1.6, and §1.7: it's Fedora's
actual default, already in-kernel with no extra installation or DKMS
maintenance burden, snapshots are cheap exactly for the pre-upgrade use
case described (per §1.2), no parity RAID is needed (sidestepping §1.5's
gap entirely), and its lighter RAM footprint suits a resource-constrained
laptop better than ZFS's ARC cache would.

</details>

## Part 3 — Reasoning Scenario

### Scenario — NovaTech Ops: a filesystem-choice retrospective

> A team deployed ZFS on a fleet of Ubuntu 24.04 servers for its native
> encryption and mature RAID-Z. Six months later, a routine kernel
> security update broke every server's ZFS mount simultaneously — the
> DKMS-built ZFS module failed to rebuild against the new kernel, and
> the servers came back up without their ZFS pools mounted at all.

Tasks:
1. Referencing §1.4, was this a ZFS bug, or a predictable consequence of
   a known, documented characteristic of how ZFS ships on Linux?
2. What would have made this outage foreseeable rather than surprising?
3. Does this incident argue for avoiding ZFS entirely, or for a more
   specific operational practice? Reference §1.7's decision framework.

<details>
<summary>Reasoning</summary>

**1.** This is a predictable, documented consequence, not a ZFS bug in
the traditional sense. Per §1.4, ZFS's CDDL licensing is the specific,
stated reason it can't be built into the Linux kernel itself — it must
ship as a separate DKMS module, rebuilt against every new kernel. A
kernel security update is exactly the kind of event that triggers this
rebuild requirement, and a rebuild failure (a kernel API change the DKMS
module hasn't caught up with yet) is a known, recurring failure mode for
out-of-tree kernel modules generally, not specific to some defect in
this particular ZFS release.

**2.** Understanding §1.4's licensing situation *before* deployment —
specifically, that ZFS's out-of-tree status makes every kernel upgrade a
potential breakage point — would have made this a known operational risk
to plan around, rather than a surprise. Ubuntu's own installer marks ZFS
support as "experimental" as of 24.04 LTS, which is a direct signal
about exactly this kind of risk.

**3.** This argues for a more specific operational practice, not
blanket avoidance — §1.7's framework still holds ZFS as the right choice
when parity-RAID-level redundancy and native encryption are genuine
requirements. What changes is the deployment discipline: test kernel
updates against the DKMS rebuild in a staging environment before
applying them fleet-wide, hold kernel upgrades until the ZFS module's
compatibility is confirmed for that specific kernel version, and
monitor for DKMS build failures explicitly rather than assuming a kernel
update is safe by default the way it would be on a host with no
out-of-tree modules in play. Choosing ZFS means accepting this
maintenance overhead as a real, ongoing cost — not a one-time setup
detail.

</details>

## What You Learned

1. ✅ Explained what copy-on-write actually changes about how a write
   happens, in contrast to ext4/xfs's in-place overwrite model
2. ✅ Explained why CoW makes snapshots effectively instant and cheap,
   in contrast to LVM's snapshot mechanism (Demo M04-04)
3. ✅ Explained built-in checksumming and self-healing, and how it
   differs from relying on RAID (Demo M04-05) or hardware alone
4. ✅ Stated the Btrfs vs. ZFS licensing situation precisely, including
   its current, real practical consequence for distro defaults and support
5. ✅ Stated the current production-readiness status of Btrfs parity
   RAID vs. ZFS RAID-Z, and why the gap has persisted
6. ✅ Chose the right filesystem for several concrete scenarios, using a
   real decision framework rather than by reputation alone

## Interview Prep

**Q1. A stakeholder asks "should we just switch everything to Btrfs or ZFS for the snapshot capability?" How do you respond?**

Not without checking two things first: which distro family is actually
in play, and whether parity RAID is a real requirement. Btrfs isn't even
available on Rocky/RHEL-family systems — it was removed from RHEL at
7.4, and that's not a "harder to configure" situation, it's simply not
supported. And if the workload needs RAID5/6-equivalent redundancy,
Btrfs's own parity RAID modes are still not considered production-ready
as of 2026, which would point toward ZFS instead, or Btrfs layered on a
separate mdadm array. The snapshot capability itself is real and
valuable, but which filesystem gets you there depends heavily on the
actual environment, not a blanket "CoW is better" decision.

**Q2. Why does a Linux kernel security update carry more operational risk for a ZFS-based host than a Btrfs-based one?**

Because of the licensing difference covered in this demo: Btrfs is GPL
and lives in-tree, built and released alongside the kernel itself — a
kernel update and Btrfs are never out of sync. ZFS's CDDL license can't
be linked into the same binary as GPL kernel code, so it always ships as
a separate, out-of-tree module via DKMS, rebuilt against each new
kernel. A kernel update is exactly the kind of event that can break that
rebuild, taking ZFS pools offline until the module catches up — a real,
recurring maintenance cost that's specific to how ZFS has to be
distributed on Linux, not a flaw in ZFS's own engineering.

**Q3. What's the actual mechanical difference that makes a Btrfs or ZFS snapshot cheaper than an LVM snapshot?**

LVM (Demo M04-04) bolts copy-on-write onto a filesystem that has no native
concept of it — creating a snapshot means reserving space upfront and
then copying original blocks into that reserved space the moment the
live volume changes. Btrfs and ZFS are copy-on-write by design for every
write, always — a snapshot is just a pointer to the current metadata
tree at that moment, consuming essentially no extra space until the live
filesystem and the snapshot actually diverge afterward. The saving isn't
a clever optimization bolted onto snapshots specifically; it's a direct
consequence of how these filesystems already handle every write.

## Key Takeaways

---

1. **Copy-on-write means a write never overwrites data in place — it writes to a new block and atomically repoints to it, leaving the old block untouched until nothing references it.** This single mechanical difference is the root cause of both filesystems' most distinctive capabilities, not a separate feature bolted on top.
2. **Btrfs/ZFS snapshots are nearly free because CoW is how these filesystems always write — a snapshot is just a pointer to a metadata state.** LVM snapshots (Demo M04-04) achieve something similar by bolting copy-on-write onto a filesystem that doesn't have it natively, requiring reserved space allocated upfront.
3. **Both filesystems checksum data at write time and verify it on every read, self-healing from a redundant copy if a mismatch is found — a capability RAID alone (Demo M04-05) doesn't provide.** RAID protects against a disk failing entirely; CoW filesystem checksumming catches silent corruption on a disk that's still nominally working.
4. **Btrfs is GPL and in-tree; ZFS is CDDL and must ship as an out-of-tree DKMS module, because the FSF has stated CDDL and GPL code can't be linked in the same binary.** This is the specific reason every ZFS-based host carries ongoing kernel-upgrade risk that a Btrfs-based host doesn't.
5. **Btrfs is not supported on Rocky Linux/RHEL at all — removed since RHEL 7.4 — while it's the default filesystem on Fedora, openSUSE, and Ubuntu Desktop as of 2026.** This is a hard availability constraint, not a difficulty-of-setup consideration, and it should be checked before any Btrfs-based plan for a Rocky/RHEL fleet.
6. **Btrfs's parity RAID (5/6) is still not considered production-ready as of 2026 due to the unresolved write-hole problem; ZFS's RAID-Z1/2/3 is mature and trusted for exactly this use case.** This is a specific, persistent gap — not a general "Btrfs is less mature" claim — and matters specifically when parity-level redundancy is a real requirement.
7. **Neither filesystem is a strict upgrade over ext4/xfs+LVM — the right choice depends on distro family, RAID requirements, RAM budget, and how much out-of-tree-module maintenance overhead is acceptable.** ext4/xfs+LVM remains the right default when CoW's specific benefits (frequent snapshots, built-in checksumming) aren't actually needed.

> **Demo scope:** Primary concept: copy-on-write as the mechanism behind
> Btrfs/ZFS's snapshot and self-healing capabilities, contrasted with the
> ext4/xfs+LVM stack from Demos M04-01/15.
> Supporting concepts: checksumming/self-healing, licensing and distro
> support status, RAID mode maturity, a concrete decision framework.
> Estimated completion time: 20 minutes (reading + decision-framework
> exercises) — no hands-on lab, per this demo's conceptual scoping.
> Checkpoints: 3 natural stopping points (after Part 1, Part 2, Part 3).
> **Verification status:** facts checked via web search for currency
> (distro defaults, licensing status, RAID stability), not live-executed
> — this demo has no commands to execute by design.

## Quick Commands Reference

> No commands were run for this demo — it's conceptual by design. For
> reference, the tools you'd use if pursuing this hands-on later:

| Command | Purpose |
|---|---|
| `mkfs.btrfs` | Create a Btrfs filesystem |
| `btrfs subvolume snapshot` | Create a Btrfs snapshot |
| `zpool create` | Create a ZFS storage pool |
| `zfs snapshot` | Create a ZFS dataset snapshot |
| `btrfs scrub start` / `zpool scrub` | Manually trigger a checksum verification pass |

## Appendix A — Anki Cards

**`m04-09-btrfs-zfs-overview-anki.csv`:**

```
#deck:Linux Mastery::Module 4 - Storage::Demo M04-09 - Btrfs/ZFS Conceptual Overview
#separator:Comma
#columns:Front,Back,Tags
"What does copy-on-write (CoW) actually change about how a write happens, compared to ext4/xfs?","A write never overwrites existing data in place. It writes to a NEW block, then atomically updates a pointer to reference the new location. The old block isn't touched or freed until nothing references it anymore.","demo16b,cow,fundamentals"
"Why are Btrfs/ZFS snapshots nearly free, compared to LVM snapshots (Demo M04-04)?","Because CoW is how these filesystems ALWAYS write, a snapshot is just a pointer to the current metadata tree at that moment — no data is copied upfront. LVM bolts copy-on-write onto a filesystem that doesn't have it natively, requiring reserved space allocated in advance.","demo16b,snapshots,cow,cross-demo"
"Does RAID (Demo M04-05) alone protect against silent data corruption on a disk that's still working?","No — RAID protects against a disk failing ENTIRELY. It doesn't checksum data or detect a bit flip / firmware bug on a disk that's still nominally functioning. Btrfs and ZFS both checksum data at write time and verify on every read, catching this class of corruption RAID alone misses.","demo16b,checksumming,raid,cross-demo"
"Why can't ZFS ship as part of the Linux kernel itself, the way Btrfs does?","ZFS (OpenZFS) is licensed under the CDDL; the FSF has stated CDDL and GPL code cannot be linked in the same binary. Since the Linux kernel is GPLv2, ZFS must always ship as a separate, out-of-tree module (via DKMS), rebuilt against each new kernel — unlike Btrfs, which lives directly in the kernel tree.","demo16b,zfs,licensing,fundamentals"
"Is Btrfs supported on Rocky Linux / RHEL?","No — Btrfs was removed from RHEL starting at version 7.4, and Rocky Linux tracks RHEL compatibility exactly on this point. There is no supported path to Btrfs on Rocky/RHEL, unlike Fedora, openSUSE, and Ubuntu Desktop, where it's the default filesystem as of 2026.","demo16b,btrfs,rocky,distros"
"Is Btrfs's native RAID5/6 mode considered production-ready as of 2026?","No — Btrfs's parity RAID modes (5/6) are still not considered production-ready due to the unresolved 'write hole' problem. Btrfs's RAID0/1/10 modes are solid. ZFS's RAID-Z1/Z2/Z3 is mature and well-regarded for exactly this parity-RAID use case.","demo16b,btrfs,raid,write-hole"
"Why did a routine kernel security update break ZFS mounts on a Ubuntu server fleet, and was this a ZFS bug?","This is a predictable consequence of ZFS's out-of-tree, DKMS-based distribution (per its CDDL licensing) — the DKMS module has to rebuild against each new kernel, and a kernel API change can break that rebuild. This is a known, recurring risk category for out-of-tree kernel modules generally, not a defect specific to that ZFS release.","demo16b,zfs,dkms,break-fix"
"A workstation on Fedora needs frequent cheap snapshots before package upgrades, with limited RAM. Which filesystem fits better, Btrfs or ZFS, and why?","Btrfs — it's Fedora's actual default (already in-kernel, no DKMS overhead), its snapshots are cheap for exactly this pre-upgrade use case, and its RAM footprint is much lighter than ZFS's ARC cache, which suits resource-constrained hardware better.","demo16b,btrfs,decision,scenario"
"A team needs RAID6-equivalent redundancy (tolerate 2 simultaneous disk failures) at maximum usable capacity. Should they use Btrfs's native RAID6 or ZFS's RAID-Z2?","ZFS's RAID-Z2 — Btrfs's native parity RAID modes are still not production-ready as of 2026. If Btrfs is still preferred for other reasons, the alternative is layering Btrfs on top of a separate mdadm RAID6 array (Demo M04-05) rather than using Btrfs's own parity RAID mode.","demo16b,raid,decision,scenario"
"Does ZFS have native encryption? Does Btrfs?","ZFS has native, per-dataset encryption built in. Btrfs does not — it needs LUKS (Demo M04-06) layered underneath, the same approach used for ext4/xfs.","demo16b,zfs,btrfs,encryption,cross-demo"
```

## Appendix B — Quiz

**`m04-09-btrfs-zfs-overview-quiz.md`:**

````markdown
# Quiz — Demo M04-09: Btrfs/ZFS Conceptual Overview

> One correct answer per question unless stated otherwise.
> Target: 80% or above before moving to the next demo.

---

**Q1.** What does copy-on-write (CoW) actually change about how a write happens, compared to ext4/xfs's model?

- A) CoW compresses data before writing it, saving space
- B) A write never overwrites existing data in place — it writes to a new block and atomically updates a pointer to reference it, leaving the old block untouched until nothing references it
- C) CoW writes data to two disks simultaneously for redundancy
- D) CoW delays writes in memory until the system is idle, then commits them

<details>
<summary>Answer</summary>

**B** — This is the foundational mechanical difference this entire demo builds on. ext4/xfs overwrite data in place; Btrfs/ZFS always write to a new location and repoint, never touching the old data until it's no longer referenced.

Trap: A invents an unrelated compression behavior; C confuses CoW with RAID mirroring, a separate concept; D invents a write-caching behavior that isn't what CoW means.

</details>

---

**Q2.** Why are Btrfs/ZFS snapshots dramatically cheaper to create than LVM snapshots (Demo M04-04)?

- A) Btrfs/ZFS snapshots only capture metadata, never the actual file contents
- B) Because CoW is how these filesystems always write, a snapshot is just a pointer to the current metadata state — no data is copied upfront, unlike LVM's reserved-space, copy-on-first-write approach
- C) LVM snapshots are actually cheaper; this is a common misconception
- D) Btrfs/ZFS snapshots use a separate, faster storage tier automatically

<details>
<summary>Answer</summary>

**B** — The cost saving is a direct consequence of CoW already being the filesystem's normal write behavior — there's nothing extra to set up or reserve. LVM achieves something similar by bolting CoW onto a filesystem that lacks it natively, which requires upfront reserved space.

Trap: A wrongly claims snapshots never reflect actual data, which would make them useless; C reverses the actual comparison; D invents an automatic storage-tiering behavior that isn't part of how snapshots work.

</details>

---

**Q3.** Does RAID alone (Demo M04-05) protect against a bit flip or firmware bug that silently corrupts data on a disk that's still functioning normally?

- A) Yes — any RAID level detects and corrects silent corruption automatically
- B) No — RAID protects against a disk failing entirely; it doesn't checksum data, so silent corruption on a still-functioning disk can go completely undetected. Btrfs/ZFS's built-in checksumming addresses exactly this gap
- C) Only RAID6 provides this protection; RAID1 does not
- D) Yes, but only if the array includes a hot spare

<details>
<summary>Answer</summary>

**B** — This is the specific gap Btrfs/ZFS's checksumming-and-self-healing feature addresses, distinct from what RAID (Demo M04-05) does. RAID's failure detection is about a disk being gone, not about verifying the correctness of data on a disk that's still responding normally.

Trap: A and C both wrongly attribute checksumming capability to RAID levels that don't have it; D invents an unrelated hot-spare requirement.

</details>

---

**Q4.** Why can't ZFS be built directly into the Linux kernel the way Btrfs is?

- A) ZFS is simply too large to fit in the kernel source tree
- B) ZFS (OpenZFS) is licensed under the CDDL, which the FSF has stated cannot be linked into the same binary as GPL code (the Linux kernel's license) — so it must ship as a separate, out-of-tree DKMS module instead
- C) Oracle has a patent that legally prevents kernel inclusion
- D) The Linux kernel maintainers have technical objections to ZFS's code quality

<details>
<summary>Answer</summary>

**B** — This is a licensing incompatibility, not a technical or size limitation. CDDL/GPL linking is the specific, stated obstacle, which is why ZFS must always be distributed as a separate module rebuilt against each kernel via DKMS.

Trap: A invents an unrelated size constraint; C invents a patent issue not identified as the actual cause; D invents a code-quality objection that isn't the documented reason.

</details>

---

**Q5.** Is Btrfs available on Rocky Linux?

- A) Yes, and it's the default filesystem
- B) Yes, but only via a third-party DKMS module, similar to ZFS on other distros
- C) No — Btrfs was removed from RHEL starting at version 7.4, and Rocky Linux tracks RHEL compatibility exactly on this point; there is no supported path to Btrfs on Rocky/RHEL
- D) Yes, but only for read-only access to existing Btrfs filesystems, not for creating new ones

<details>
<summary>Answer</summary>

**C** — This is a hard availability constraint specific to the RHEL/Rocky family, not a matter of configuration difficulty. It's the opposite situation from Fedora, openSUSE, and Ubuntu Desktop, where Btrfs is the actual default.

Trap: A and D both wrongly claim some form of Btrfs availability that doesn't exist on Rocky; B confuses Btrfs's situation with ZFS's DKMS-based distribution model, which doesn't apply here since Btrfs isn't available on Rocky at all, DKMS or otherwise.

</details>

---

**Q6.** Is Btrfs's native RAID6 mode considered production-ready as of 2026?

- A) Yes, and has been for several years
- B) No — the parity RAID modes (5/6) are still not considered production-ready due to the unresolved "write hole" problem, while Btrfs's RAID0/1/10 modes are solid
- C) No — none of Btrfs's RAID modes, including RAID1, are considered production-ready
- D) Yes, but only when combined with ECC RAM

<details>
<summary>Answer</summary>

**B** — This is a specific, persistent gap limited to Btrfs's parity RAID modes — its mirroring/striping modes (0/1/10) are solid and widely used in production. ZFS's RAID-Z is the mature alternative for parity-RAID-level redundancy.

Trap: A overstates Btrfs's parity RAID maturity, which is specifically the unresolved case; C overstates the problem to modes that are actually fine; D invents an ECC RAM condition that isn't the documented fix for this issue.

</details>

---

**Q7.** A fleet of ZFS-based Ubuntu servers goes down after a routine kernel security update, because the DKMS-built ZFS module failed to rebuild against the new kernel. Was this a ZFS software defect?

- A) Yes — this indicates a bug specific to that ZFS release
- B) Not really — this is a predictable, documented consequence of ZFS's CDDL licensing forcing it to ship as an out-of-tree DKMS module, which must rebuild against every new kernel and can fail when a kernel API changes
- C) No — this can only happen if the servers were also running Btrfs simultaneously
- D) Yes — Canonical guarantees ZFS stability in Ubuntu's installer, so any breakage is a packaging bug

<details>
<summary>Answer</summary>

**B** — This is a known, recurring risk category for any out-of-tree kernel module, directly traceable to the licensing situation covered in this demo — not a one-off defect in a specific ZFS release.

Trap: A misattributes a category-level risk to a specific bug; C invents an unrelated Btrfs-coexistence condition; D overstates Ubuntu's actual stated position — ZFS support is marked experimental precisely because this kind of risk exists, not guaranteed stable.

</details>

---

**Q8.** A team needs RAID6-equivalent redundancy (tolerate 2 simultaneous disk failures) at maximum usable capacity, and is deciding between Btrfs's native RAID6 and ZFS's RAID-Z2. Which is the better-supported choice as of 2026, and why?

- A) Btrfs's RAID6 — it's simpler to set up
- B) ZFS's RAID-Z2 — Btrfs's native parity RAID modes are still not considered production-ready as of 2026, while ZFS's RAID-Z implementations are mature and well-regarded for exactly this use case
- C) Neither — parity RAID is deprecated technology on Linux entirely
- D) They're equally mature; the choice should be based on licensing preference alone

<details>
<summary>Answer</summary>

**B** — This directly reflects the persistent gap covered in §1.5: Btrfs's parity RAID modes remain not production-ready, making ZFS's RAID-Z2 (or Btrfs layered on a separate mdadm array) the better-supported path for this specific requirement.

Trap: A recommends the less mature option based on an irrelevant simplicity claim; C invents a deprecation that hasn't happened — parity RAID remains a real, actively used option via ZFS or mdadm; D ignores the real, documented maturity gap in favor of an unrelated licensing-only framing.

</details>

---

Score guide:

| Score | Action |
|---|---|
| 7/8 or 8/8 | Import Anki cards — this closes out Module 4 |
| 6/8 | Review the wrong answers, then proceed |
| 5/8 | Re-read the relevant sections, retry those questions |
| Below 5/8 | Re-read the full demo and redo the walkthrough before proceeding |
````