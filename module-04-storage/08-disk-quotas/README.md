# Demo 16a — Disk Quotas

> **A note on how this demo was built:**
> Same constraint as Demos 15a and 15b: the `quota` package is not
> installed and can't be installed (no network access), and this
> container has no quota kernel subsystem available either (no
> `/lib/modules/.../kernel/fs/quota/` tree). Every command and behavior
> below is sourced to the official `repquota(8)`/`edquota(8)` manual
> pages and Red Hat's quota documentation (cited inline), not to
> live-captured output. Nothing below was run in the environment that
> authored it. Before treating this demo as production-verified, run
> Part 2 for real (a spare VM with a dedicated ext4/xfs mount works fine)
> and confirm the output matches what's described.

## Overview

Demo 14 covered provisioning a volume; Demos 15a and 15b covered
protecting it from disk failure and unauthorized physical access. This
demo covers a different problem entirely: what happens when the people
or processes *using* a shared volume don't fail or misbehave, but simply
consume more than their fair share — one user's runaway log directory,
one team's forgotten test data, filling a `/home` partition everyone else
depends on. Disk quotas enforce per-user or per-group limits on a
filesystem, with a built-in grace period that gives someone a chance to
clean up before they're actually locked out.

## Demo Objectives

1. Explain what disk quotas actually solve (fair-share capacity
   enforcement) and how that's distinct from RAID (Demo 15a) or backups
   (Demo 14a/14b) — none of which prevent one user from filling a shared disk
2. Enable quotas on a filesystem via `/etc/fstab` and initialize them
   with `quotacheck`
3. Set user quota limits with `setquota`/`edquota`, understanding the
   distinction between block limits (space) and inode limits (file count)
4. Explain the soft limit / hard limit / grace period model precisely —
   what each one actually permits and blocks
5. Read `repquota` output and interpret its exceeded-limit indicators
6. Diagnose the classic "I added `usrquota` to fstab and nothing
   happened" mistake, and the "my usage is under the hard limit but I'm
   still blocked" grace-period confusion

## Directory Structure

```
16a-disk-quotas/
├── README.md                              # this file
├── 16a-disk-quotas-anki.csv               # Anki flashcard deck
└── 16a-disk-quotas-quiz.md                # standalone quiz
```

## Environments

| Task | Ubuntu 24.04 | Rocky Linux 9.8 | EC2 Amazon Linux |
|---|---|---|---|
| All lab tasks | ⚠️ [VERIFY] — the `quota` package is in Ubuntu 24.04's default `apt` repository (candidate 4.06-1build6 confirmed via `apt-cache policy` in this session) but neither installed nor executed; this session's container has no quota kernel subsystem available at all | ⚠️ [VERIFY] — not checked this session; `quota` ships in Rocky9's default repos as a standard package | Not required |

## Recall Check — 15b-luks-encryption

Answer these from memory before continuing — no looking back at Demo 15b.

1. Does LUKS protect data on a volume that's already unlocked and mounted on a running system?

2. What happens if a `cryptsetup open`'d volume has no `/etc/crypttab` entry, after a reboot?

3. Why is a LUKS header backup non-negotiable, in a way most storage backups aren't?

<details>
<summary>Answers</summary>

1. No — once unlocked and mounted, the volume behaves like any normal filesystem, readable in plaintext by anything with access to that running system. LUKS protects data at rest, not a compromised running system.

2. It stays locked — a manually opened volume with no crypttab entry has no way to automatically unlock at boot, the same persistence trap as an unpersisted mount or `ip addr add`.

3. Header damage means the passphrase, even if correct, has nothing to decrypt — the volume key lives in the header. Without an intact header or a backup of one, the data is permanently unrecoverable.

</details>

---

## Part 1 — Concepts

### 1.1 What quotas solve — and what they don't

Quotas are a *fair-share* mechanism, not a durability or security one.
None of Demo 14a/14b's backups, Demo 15a's RAID, or Demo 15b's LUKS
encryption prevent a single user or group from filling a shared
filesystem — a runaway log directory, a forgotten dataset, or simple
carelessness on a shared `/home` can take down every other user's
ability to write, with none of the previous demos' protections applying
to that specific failure mode. Quotas cap per-user or per-group
consumption on a given filesystem, so one account's excess doesn't
become an outage for everyone sharing that disk.

### 1.2 Enabling quotas — `/etc/fstab` and `quotacheck`

Per Red Hat's quota documentation, enabling quotas is a three-step
process: mark the filesystem in `/etc/fstab`, remount it, then build the
quota accounting database.

```bash
# /etc/fstab entry, adding quota options to an existing mount:
UUID=xxxx  /home  ext4  defaults,usrquota,grpquota  0  2
```

`usrquota` enables per-user quotas; `grpquota` enables per-group quotas;
both can be present together, as shown here. This is the same fstab
persistence pattern Demo 14 covered for mount options generally — the
option only takes effect once the filesystem is actually remounted (or
the system rebooted) with that line in place:

```bash
sudo mount -o remount /home
```

```bash
sudo quotacheck -avug
```

Per multiple independently-documented transcripts of this exact command,
`quotacheck` scans every quota-enabled filesystem and builds the
accounting database from scratch — `-a`: all quota-enabled filesystems,
`-v`: verbose, `-u`: user quotas, `-g`: group quotas. This creates two
files at the root of the quota-enabled filesystem: `aquota.user` and
`aquota.group` (the modern format; older systems use `quota.user`/
`quota.group`). **This step is not optional and not automatic** — simply
adding `usrquota`/`grpquota` to `/etc/fstab` and remounting does not, by
itself, create these accounting files or start enforcing anything; §3's
Break-Fix walks through exactly this gap.

### 1.3 Setting limits — `setquota` and `edquota`

```bash
sudo setquota -u jsmith 10485760 12582912 0 0 /home
```

Per the documented command format, `setquota -u USERNAME BLOCK_SOFT
BLOCK_HARD INODE_SOFT INODE_HARD FILESYSTEM` — block values are in
kilobytes (here: 10 GB soft, 12 GB hard), and inode values (0 0 in this
example) are left unlimited. `setquota` sets limits directly from the
command line, in one shot — useful for scripting.

```bash
sudo edquota jsmith
```

Per the manual page, `edquota` is the interactive alternative: it opens
a temporary file with the user's current quota state in `$EDITOR`,
showing existing usage and letting you edit the limits directly rather
than remembering `setquota`'s positional argument order:

```
Disk quotas for user jsmith (uid 1001):
  Filesystem           blocks       soft       hard     inodes     soft     hard
  /dev/mapper/vg-home  440436          0          0      37418        0        0
```

**Block limits vs. inode limits — two genuinely different constraints:**
block limits cap total disk space consumed; inode limits cap the *number
of files*, regardless of their individual size. A user with thousands of
tiny files (a build cache, a mail spool with one file per message) can
exhaust an inode limit while barely touching their block limit — this is
a real, distinct failure mode from "ran out of space," worth setting
both limits deliberately rather than only thinking in terms of gigabytes.

### 1.4 Soft limit, hard limit, grace period — the three-part model

This is the part of quotas most often misunderstood, and it's worth
having precisely rather than approximately:

- **Soft limit** — a warning threshold. A user can exceed it, but doing
  so starts a countdown (the grace period).
- **Hard limit** — an absolute ceiling. It can never be exceeded, full
  stop, regardless of grace period status.
- **Grace period** — the time window after crossing the soft limit
  during which writes still succeed. Once the grace period expires
  without the user dropping back under the soft limit, **the soft limit
  itself starts being enforced as if it were a hard limit** — writes are
  blocked even though the user is still under the actual hard limit.

**Worked example**, per a documented walkthrough: soft limit 10 GB, hard
limit 12 GB, grace period 7 days. A user writes up to 10.5 GB on Monday —
crossing the soft limit starts their 7-day countdown, but writes continue
succeeding. If they're still over 10 GB the following Monday, writes get
blocked — not because they hit 12 GB, but because their grace period on
the 10 GB soft limit ran out. Separately, and regardless of grace period
timing: if usage reaches 12 GB (the hard limit) at any point, writes are
blocked immediately, no countdown involved.

**This is exactly the mechanism behind a genuinely confusing support
ticket:** "my usage is well under my hard limit, but I can't write
anything." §3's second incident is precisely this scenario.

```bash
sudo edquota -t
```

Per the manual page, this edits the *filesystem-wide* grace period
(applied once a user's soft limit is exceeded) rather than any
individual user's limits — grace period is a per-filesystem setting, not
set per-user the way soft/hard limits are.

### 1.5 Reading `repquota` output

```bash
sudo repquota -a
```

Per Red Hat's documentation and the manual page, `repquota` summarizes
usage and limits for every user on a quota-enabled filesystem:

```
*** Report for user quotas on device /dev/mapper/VolGroup00-LogVol02
Block grace time: 7days; Inode grace time: 7days
                        Block limits                File limits
User      used    soft    hard   grace    used  soft  hard  grace
----------------------------------------------------------------------
root  --     36       0       0              4     0     0
kristin --  540       0       0            125     0     0
testuser +- 440400  500000  550000          37418     0     0
```

The two-character column immediately after the username is the part to
read carefully. Per the manual page: the first character marks block
(space) limit status — `-` means under the soft limit, `+` means the
soft limit is exceeded (or the hard limit reached, if no soft limit is
set). The second character does the same for inode (file count) status.
`testuser` above shows `+-`: over their block soft limit, but fine on
inodes — and the `grace` column for that row would show the remaining
countdown once populated. A row showing `--` for a user with usage well
under their limits, versus a row showing `+-`, is the fastest way to
distinguish "fine" from "in their grace period, about to be blocked" at
a glance across an entire filesystem's users.

## Part 2 — Hands-On Lab

**Before starting:** confirm the `quota` package is installed (`which
quotacheck`; install with `sudo apt install quota` / `sudo dnf install
quota` if not) and that you have a dedicated ext4 or xfs mount to test
against — never enable quotas on a shared production filesystem for a
first attempt. This lab was not run in the environment that authored it
— treat every `# Expected` comment as documented behavior to confirm.

### Commands used in this demo

| Command | Flag meaning | Purpose |
|---|---|---|
| `mount -o remount FS` | — | Apply new fstab options without a reboot |
| `quotacheck -avug` | all/verbose/user/group | Build the quota accounting database |
| `setquota -u USER SB HB SI HI FS` | soft/hard block, soft/hard inode | Set limits directly, one command |
| `edquota USER` | — | Interactively edit a user's limits |
| `edquota -t` | — | Edit the filesystem-wide grace period |
| `repquota -a` | all | Summarize usage/limits for all quota-enabled filesystems |
| `quotaon` / `quotaoff` | — | Enable/disable enforcement on a filesystem |

### Task 1 — Enable quotas on a test filesystem

```bash
# Assumes a dedicated test mount already exists, e.g. from Demo 14's lab
grep /mnt/labdata /etc/fstab
# Add usrquota,grpquota to the options field for that line, e.g.:
# LABEL=lab-data  /mnt/labdata  ext4  defaults,usrquota,grpquota  0  2

sudo mount -o remount /mnt/labdata
mount | grep labdata
# Expected: usrquota and grpquota now appear in the mount options
```

### Task 2 — Build the accounting database

```bash
sudo quotacheck -avug
ls -la /mnt/labdata/aquota.*
# Expected: aquota.user and aquota.group now present at the mount root
```

### Task 3 — Turn on enforcement and set a limit

```bash
sudo quotaon -avu
sudo setquota -u testuser 10240 12288 0 0 /mnt/labdata
# 10 MB soft, 12 MB hard, unlimited inodes — small values for a quick lab test

sudo repquota /mnt/labdata
# Expected: testuser now shows the 10240/12288 block limits
```

### Task 4 — Cross the soft limit and observe grace-period behavior

```bash
sudo -u testuser dd if=/dev/zero of=/mnt/labdata/bigfile bs=1M count=11 2>&1
# Expected (per §1.4): the write succeeds — 11MB is over the 10MB soft
# limit but under the 12MB hard limit; a warning may print, but the
# write is not blocked

sudo repquota /mnt/labdata
# Expected: testuser's row now shows a + in the block-status column,
# and a grace-period countdown in the grace column
```

### Task 5 — Hit the hard limit

```bash
sudo -u testuser dd if=/dev/zero of=/mnt/labdata/bigfile2 bs=1M count=5 2>&1
# Expected: this write is BLOCKED partway through — 11MB + 5MB exceeds
# the 12MB hard limit, and hard limits are never exceedable regardless
# of grace period
```

### Task 6 — Clean up

```bash
sudo -u testuser rm -f /mnt/labdata/bigfile /mnt/labdata/bigfile2
sudo setquota -u testuser 0 0 0 0 /mnt/labdata
sudo quotaoff -avu
# Remove usrquota,grpquota from the fstab line if this was a real test mount
```

## What You Learned

1. ✅ Explained what disk quotas solve (fair-share capacity enforcement)
   and how that's distinct from RAID (Demo 15a) and backups (Demo 14a/14b)
2. ✅ Enabled quotas via `/etc/fstab` and initialized accounting with
   `quotacheck`
3. ✅ Set user quota limits with `setquota`/`edquota`, distinguishing
   block limits from inode limits
4. ✅ Explained the soft limit / hard limit / grace period model
   precisely, including why a user can be blocked while under their hard limit
5. ✅ Read `repquota` output and interpreted its `+`/`-` exceeded-status
   indicators
6. ✅ Diagnosed the "quota added to fstab but not enforced" mistake and
   the grace-period-expired confusion

## Part 3 — Break-Fix Scenario

### Scenario — NovaTech Ops: two quota incidents

> **Incident report — NovaTech Ops, ticket #4759**
>
> Alert 1: "we added `usrquota,grpquota` to `/home`'s fstab entry weeks
> ago and remounted, but a user just filled the disk completely — quotas
> don't seem to be doing anything at all." Alert 2: "a user's `repquota`
> row shows they're using 8GB against a 10GB hard limit, well under —
> but they're reporting every write fails with a disk-quota-exceeded
> error. Nothing in monitoring shows the disk as full."

**Incident 1 — quota options added to fstab, but nothing was enforced**

Tasks:
1. Referencing §1.2, what's the most likely missing step, given that the
   fstab edit and remount were both done correctly?
2. What two files, specifically, would confirm whether this step was
   ever completed?
3. What's the fix, and why doesn't editing `/etc/fstab` alone ever
   create those files?

**Incident 2 — blocked despite being under the hard limit**

Tasks:
1. Referencing §1.4, what's the most likely explanation for writes being
   blocked when usage is reported well under the hard limit?
2. What specific field in `repquota`'s output would confirm this
   diagnosis, that the Alert 1 investigation might not have looked at?
3. What's the actual fix here — and is it a quota system malfunction?

<details>
<summary>Answers</summary>

**Incident 1:** Per §1.2, enabling quotas is a three-step process, and
the fstab edit plus remount only completes the first two — the missing
step is `quotacheck`, which actually builds the accounting database that
`setquota`/`edquota`/enforcement depend on. Without it, the kernel has
`usrquota`/`grpquota` mount options active, but no accounting data
exists for any user, and (per the documented three-step process) nothing
is actually being tracked or enforced. The two files that confirm this:
`aquota.user` and `aquota.group` at the root of the mounted filesystem —
their absence means `quotacheck` was never run. The fix: run `sudo
quotacheck -avug`, then `quotaon -avu` to start enforcement, then set
actual limits with `setquota`/`edquota` — none of which happens
automatically just from the fstab line and remount, because `/etc/fstab`
only controls *mount options*, not the separate quota accounting
database that has to be built and limits that have to be assigned
explicitly.

**Incident 2:** This is the grace-period-expired scenario from §1.4. A
user can be reported as using 8GB against a 10GB hard limit and still be
completely blocked from writing, if their *soft* limit (say, 5GB) was
exceeded long enough ago that their grace period has run out — at that
point, the soft limit is enforced as if it were the hard limit,
regardless of how much headroom remains before the actual hard limit.
The field that confirms this: `repquota`'s `grace` column for that
user's block-limits row — an expired grace period is exactly the kind of
detail Alert 1's investigation (which was checking whether enforcement
existed at all) wouldn't have had reason to look at, since it's about a
*different* failure mode entirely: enforcement working exactly as
designed, just not the way the user (or the on-call engineer, without
having read §1.4 carefully) expected. This is not a quota malfunction —
the system is behaving exactly as the soft/hard/grace model specifies.
The actual fix: either the user needs to reduce usage back under their
soft limit (which resets the grace period the next time they cross it),
or an administrator can raise the soft limit itself with `setquota`/
`edquota` if 5GB was too conservative for this user's legitimate needs.

</details>

## Interview Prep

**Q1. A team lead asks "we have RAID and backups on our home directory server — do we need quotas too?"**

Yes, and it's worth explaining why those aren't substitutes for each
other. RAID protects against disk failure; backups protect against data
loss; neither does anything to stop one user or process from filling
the shared disk in the first place, which is an availability problem for
every *other* user on that filesystem the moment it happens. Quotas are
the specific tool for that failure mode — capping per-user or per-group
consumption so one account's runaway usage doesn't become an outage for
everyone sharing the disk.

**Q2. A user reports their writes are failing with a quota error, but `repquota` shows their usage well under their hard limit. What's your first hypothesis?**

An expired grace period on their soft limit, not a hard-limit issue.
Per the three-part soft/hard/grace model, once someone exceeds their
soft limit and the grace period given to fix that runs out, the soft
limit itself starts being enforced as a hard stop — regardless of how
much room remains before the actual hard limit. I'd check the `grace`
column in `repquota`'s output specifically, which is easy to overlook if
you're only comparing usage against the hard limit number.

**Q3. Quotas were enabled on a filesystem — fstab updated, remounted — but nobody can find any evidence quotas are actually being tracked. What's the most likely missing step?**

`quotacheck` was never run. Enabling quotas is a three-step process:
mark the filesystem in fstab, remount it, and separately build the
accounting database with `quotacheck` — the fstab options alone don't
create the `aquota.user`/`aquota.group` files that tracking and
enforcement actually depend on. Checking for those two files at the
filesystem root is the fastest way to confirm whether this step was ever
completed.

## Key Takeaways

---

1. **Quotas are a fair-share mechanism, not a durability or security one — they solve a problem RAID (Demo 15a), backups (Demo 14a/14b), and LUKS (Demo 15b) don't touch.** None of those protections stop one user from filling a shared disk; quotas cap per-user/per-group consumption specifically to prevent that.
2. **Enabling quotas is a three-step process, and skipping the last step is the most common real-world mistake: mark `/etc/fstab` (`usrquota`/`grpquota`), remount, then separately run `quotacheck` to build the accounting database.** The fstab edit and remount alone create no `aquota.user`/`aquota.group` files and enforce nothing.
3. **Block limits (space) and inode limits (file count) are genuinely separate constraints.** A user with many small files can exhaust their inode limit while barely touching their block limit — a distinct failure mode from "ran out of space," worth setting deliberately rather than defaulting to unlimited.
4. **The soft/hard/grace model has a specific, easy-to-misread behavior: once a grace period on the soft limit expires, the soft limit is enforced as a hard stop — even while usage remains well under the actual hard limit.** This produces the exact confusing symptom of "I'm blocked but not even close to my hard limit," which is working as designed, not a malfunction.
5. **`repquota`'s two-character status column (`-`/`+` per user, for blocks then inodes) is the fast way to scan an entire filesystem's users for who's over a limit** — and the `grace` column specifically is what distinguishes "just crossed the soft limit, still fine" from "grace expired, now blocked," a distinction the block-status character alone doesn't show.
6. **`setquota` sets limits in one command (`-u USER BLOCK_SOFT BLOCK_HARD INODE_SOFT INODE_HARD FS`) for scripting; `edquota` opens an interactive editor showing current usage alongside the limits being set** — same underlying change, different workflow fit.
7. **Grace period is a per-filesystem setting (`edquota -t`), not set individually per user the way soft/hard limits are.** Every user sharing a filesystem gets the same grace window once they cross their own soft limit.

> **Demo scope:** Primary concept: quotas as fair-share enforcement,
> distinct from durability/security tools covered elsewhere in this module.
> Supporting concepts: fstab/quotacheck setup, setquota/edquota, block vs.
> inode limits, the soft/hard/grace model, repquota interpretation.
> Estimated completion time: 32 minutes (reading + hands-on + verification),
> ASSUMING the quota package and kernel subsystem are available — this
> demo's own authoring environment had neither.
> Checkpoints: 6 natural stopping points (after Task 1 through Task 6).
> **Verification status:** built from official documentation, not live
> execution in this session — see the note at the top of this file.

## Quick Commands Reference

### Setup

| Command | Purpose |
|---|---|
| `usrquota,grpquota` in `/etc/fstab` | Enable per-user/per-group quota tracking on a mount |
| `mount -o remount FS` | Apply the new fstab options without a reboot |
| `quotacheck -avug` | Build the quota accounting database |
| `quotaon -avu` / `quotaoff -avu` | Start/stop enforcement |

### Setting and viewing limits

| Command | Purpose |
|---|---|
| `setquota -u USER SB HB SI HI FS` | Set block/inode soft and hard limits directly |
| `edquota USER` | Interactively edit a user's limits |
| `edquota -t` | Edit the filesystem-wide grace period |
| `repquota -a` | Summarize usage and limit status for all quota-enabled filesystems |

## Appendix A — Anki Cards

**`16a-disk-quotas-anki.csv`:**

```
#deck:Linux Mastery::Module 4 - Storage::Demo 16a - Disk Quotas
#separator:Comma
#columns:Front,Back,Tags
"Do RAID and backups prevent one user from filling a shared disk?","No — RAID protects against disk failure and backups protect against data loss, but neither stops a single user's excessive usage from taking down write access for everyone else sharing that filesystem. Quotas are the specific tool for that fair-share problem.","demo16a,quota,fundamentals"
"You added usrquota,grpquota to /etc/fstab and remounted the filesystem. Is quota tracking now active?","Not necessarily — enabling quotas is a three-step process. The fstab edit and remount only complete steps one and two. quotacheck must be run separately to build the aquota.user/aquota.group accounting database that tracking and enforcement depend on.","demo16a,quota,break-fix,common-mistake"
"What two files does quotacheck create, and where?","aquota.user and aquota.group, at the root of the quota-enabled filesystem. Their absence means quotacheck was never run, even if usrquota/grpquota are active mount options.","demo16a,quota,quotacheck,fundamentals"
"What's the difference between a block limit and an inode limit in disk quotas?","Block limits cap total disk space consumed; inode limits cap the NUMBER of files, regardless of individual file size. A user with many small files can exhaust an inode limit while barely touching their block limit.","demo16a,quota,inodes,fundamentals"
"A user crosses their soft limit but stays under their hard limit. Are their writes blocked?","Not immediately — crossing the soft limit starts a grace period countdown, during which writes still succeed. Writes are only blocked once the grace period expires without usage dropping back under the soft limit, OR if usage reaches the hard limit directly.","demo16a,quota,soft-limit,grace-period,fundamentals"
"A user's grace period on their soft limit has expired, but they're still well under their hard limit. Can they write?","No — once the grace period expires, the soft limit itself is enforced as if it were a hard limit, regardless of remaining headroom before the actual hard limit. This produces a confusing 'blocked but under my hard limit' symptom that is working as designed, not a malfunction.","demo16a,quota,grace-period,break-fix"
"In repquota output, what does a + vs - mean in the two-character status column after a username?","- means under the limit; + means the soft limit is exceeded (or hard limit reached if no soft limit is set). The first character is for block (space) limits, the second for inode (file count) limits.","demo16a,quota,repquota,fundamentals"
"What's the difference between setquota and edquota for setting limits?","setquota sets limits directly in one command (positional: user, block soft, block hard, inode soft, inode hard, filesystem) — good for scripting. edquota opens an interactive editor showing current usage alongside the limits, letting you edit them directly.","demo16a,quota,setquota,edquota,comparison"
"Is the grace period set per-user or per-filesystem?","Per-filesystem, via edquota -t — not set individually per user the way soft and hard limits are. Every user sharing that filesystem gets the same grace window once they cross their own soft limit.","demo16a,quota,grace-period,fundamentals"
"A repquota report shows a user's grace column has a countdown value. What does this indicate?","The user has exceeded their soft limit and is currently within their grace period — writes are still succeeding, but will be blocked once the grace period shown expires unless usage drops back under the soft limit first.","demo16a,quota,repquota,grace-period"
```

## Appendix B — Quiz

**`16a-disk-quotas-quiz.md`:**

````markdown
# Quiz — Demo 16a: Disk Quotas

> One correct answer per question unless stated otherwise.
> Target: 80% or above before moving to the next demo.

---

**Q1.** A server has RAID and regular backups configured on its `/home` partition. Does this mean the partition is protected against a single user filling it completely?

- A) Yes — RAID and backups together cover all storage-related failure modes
- B) No — RAID protects against disk failure and backups protect against data loss, but neither prevents one user's excessive usage from consuming shared space; quotas are the specific tool for that
- C) Yes, but only if the RAID level is RAID6 or higher
- D) No — only LUKS encryption prevents this

<details>
<summary>Answer</summary>

**B** — This is a distinct failure mode from what RAID or backups address. Quotas exist specifically to cap per-user/per-group consumption so one account doesn't take down write access for everyone else on a shared filesystem.

Trap: A wrongly assumes comprehensive coverage from unrelated tools; C invents an irrelevant RAID-level distinction; D names the wrong tool entirely — LUKS addresses data-at-rest confidentiality, not capacity fairness.

</details>

---

**Q2.** `usrquota,grpquota` was added to `/etc/fstab` for `/home` and the filesystem was remounted. A week later, no quota enforcement appears to be happening at all. What's the most likely missing step?

- A) The system needs a full reboot; remounting alone is never sufficient
- B) `quotacheck` was never run — enabling quotas is a three-step process, and the fstab edit plus remount only complete the first two steps
- C) `usrquota` and `grpquota` are mutually exclusive and can't both be set
- D) Quota enforcement requires a separate license key to activate

<details>
<summary>Answer</summary>

**B** — `quotacheck` builds the actual accounting database (`aquota.user`/`aquota.group`) that tracking and enforcement depend on. Without it, the mount options are active but nothing is actually being measured or enforced.

Trap: A overstates the reboot requirement — remounting is documented as sufficient for the fstab options themselves; C wrongly claims incompatibility between two options that are commonly used together; D invents a licensing requirement that doesn't exist for standard Linux quota tools.

</details>

---

**Q3.** What's the practical difference between a block limit and an inode limit?

- A) They're two names for the same underlying limit, expressed in different units
- B) Block limits cap total disk space consumed; inode limits cap the number of files, regardless of individual file size — a user can exhaust one without approaching the other
- C) Block limits apply only to root-owned files; inode limits apply only to regular user files
- D) Inode limits are deprecated and no longer enforced on modern filesystems

<details>
<summary>Answer</summary>

**B** — These are genuinely independent constraints. A user storing thousands of tiny files (a build cache, individual mail files) can hit an inode limit while barely using any actual disk space, which block limits alone wouldn't catch.

Trap: A wrongly conflates two different measurement dimensions; C invents an ownership-based restriction that doesn't apply; D is false — inode limits remain a standard, actively enforced part of the quota system.

</details>

---

**Q4.** A user's usage crosses their soft limit but stays under their hard limit. What happens to their write access?

- A) Writes are blocked immediately upon crossing the soft limit
- B) Writes continue succeeding — crossing the soft limit starts a grace period countdown, and writes are only blocked once that grace period expires without usage dropping back under the soft limit (or if the hard limit is reached directly)
- C) Writes succeed but at a throttled, slower speed
- D) The user receives a warning email but nothing else changes

<details>
<summary>Answer</summary>

**B** — This is the core soft/hard/grace mechanism: the soft limit is a warning threshold with a countdown attached, not an immediate block. Only grace-period expiration or reaching the actual hard limit blocks writes.

Trap: A misreads the soft limit as an immediate hard stop, which defeats its purpose as a warning threshold; C invents a throttling behavior the quota system doesn't implement; D understates the real consequence — there is an actual enforcement mechanism (the grace countdown), not just a passive notification.

</details>

---

**Q5.** A `repquota` report shows a user at 8GB usage against a 10GB hard limit, but the user reports every write failing. Monitoring shows the disk isn't full. What's the most likely explanation?

- A) The repquota report is stale and needs to be regenerated
- B) The user's grace period on their SOFT limit has expired — the soft limit is now being enforced as a hard stop, regardless of remaining headroom before the actual 10GB hard limit
- C) A different user on the same filesystem has hit their own hard limit, blocking everyone
- D) The 8GB figure reported is incorrect due to a `repquota` bug

<details>
<summary>Answer</summary()>

**B** — This is exactly the grace-period-expired scenario: enforcement can kick in well before the hard limit is reached if the user's soft limit was crossed long enough ago. The `grace` column in `repquota`'s output specifically reveals this, distinct from the block-status character alone.

Trap: A, C, and D all invent explanations unrelated to the documented soft/hard/grace mechanism, when the reported symptom matches that mechanism precisely.

</details>

---

**Q6.** In `repquota` output, a user's row shows `+-` in the two-character status column. What does this indicate?

- A) Both block and inode limits are exceeded
- B) The block (space) limit is exceeded (`+`), but the inode (file count) limit is fine (`-`)
- C) Neither limit is exceeded; `+-` is a formatting artifact with no meaning
- D) The user's quota has been administratively suspended

<details>
<summary>Answer</summary>

**B** — The first character reflects block-limit status, the second reflects inode-limit status, independently. `+-` specifically means over on space, fine on file count — a real, distinct combination from `-+` or `++`.

Trap: A misreads both characters as exceeded when only the first is; C wrongly dismisses meaningful output as an artifact; D invents an administrative-suspension state that `repquota`'s status characters don't represent.

</details>

---

**Q7.** What's the functional difference between `setquota` and `edquota` for setting a user's limits?

- A) `setquota` only works for group quotas; `edquota` only works for user quotas
- B) `setquota` sets limits directly via command-line arguments in one step, useful for scripting; `edquota` opens an interactive editor showing current usage alongside the limits being set
- C) `edquota` is deprecated in favor of `setquota` on all modern systems
- D) They produce different, incompatible quota formats that can't coexist on the same filesystem

<details>
<summary>Answer</summary>

**B** — Both change the same underlying quota data; they differ only in workflow — `setquota` for one-shot/scripted changes, `edquota` for interactive review-and-edit against current usage context.

Trap: A invents a user/group restriction that doesn't exist for either tool; C wrongly claims deprecation; D invents an incompatibility between two tools that operate on the exact same quota data.

</details>

---

**Q8.** Is the grace period configured per individual user, the same way soft and hard limits are?

- A) Yes — each user has their own independently configurable grace period
- B) No — grace period is a per-filesystem setting (via `edquota -t`); every user sharing that filesystem gets the same grace window once they cross their own soft limit
- C) Grace period is set per-file, not per-user or per-filesystem
- D) Grace period cannot be configured at all; it's a fixed kernel constant

<details>
<summary>Answer</summary>

**B** — Soft and hard limits are set per-user (or per-group); grace period is a filesystem-wide setting applied uniformly whenever any user on that filesystem crosses their own soft limit.

Trap: A wrongly extends the per-user model to grace period specifically; C invents a per-file granularity that doesn't exist in the quota system; D wrongly claims grace period is fixed and unconfigurable, when `edquota -t` exists specifically to change it.

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