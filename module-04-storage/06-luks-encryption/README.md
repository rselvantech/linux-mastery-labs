# Demo M04-06 — Encryption at Rest (LUKS/dm-crypt)

> **A note on how this demo was built:**
> Same constraint as Demo M04-05: `cryptsetup` is not installed and can't be
> installed (no network access), and this container has no `dm-crypt`
> kernel module or `/dev/mapper` device-mapper support available at all.
> Every command and behavior below is sourced to the official
> `cryptsetup(8)` manual page and Red Hat's LUKS documentation (both
> cited inline), not to live-captured output. Nothing below was run in
> the environment that authored it. Before treating this demo as
> production-verified, run Part 2 for real (a spare VM works fine — no
> real disk is needed, a loop device is sufficient) and confirm the
> output matches what's described.

## Overview

Demo M04-05 covered surviving a disk failing. This demo covers a different
threat model entirely: what happens when a disk — working or not — ends
up somewhere it shouldn't. A decommissioned drive sold without being
wiped, a stolen laptop, a compliance auditor asking "is data at rest
encrypted on this host" — all failure modes that RAID, backups, and
filesystem permissions do nothing to address. LUKS (Linux Unified Key
Setup), built on the kernel's `dm-crypt` device-mapper target, is the
standard Linux answer: transparent block-device encryption that sits
between the raw disk and everything above it (partitioning, filesystem,
mounting) covered in Demo M04-01.

## Demo Objectives

1. Explain what LUKS/dm-crypt actually protects against (data at rest on
   a disk that leaves your control) and what it doesn't (data while the
   volume is unlocked and mounted, weak passphrases)
2. Initialize a LUKS volume with `luksFormat` and unlock it with
   `luksOpen`/`open`, understanding the key-slot passphrase model
3. Use the resulting `/dev/mapper` device exactly like any other block
   device — partition, format, and mount it per Demo M04-01's pattern
4. Read `luksDump` output and manage multiple key slots with `luksAddKey`
5. Persist automatic unlocking via `/etc/crypttab`, and diagnose the
   classic "volume locked after reboot" failure
6. Explain the LUKS1/LUKS2 distinction and state the header-backup
   discipline that prevents a self-inflicted, unrecoverable data loss

## Directory Structure

```
06-luks-encryption/
├── README.md                        # this file
├── m04-06-luks-encryption-anki.csv  # Anki flashcard deck
└── m04-06-luks-encryption-quiz.md   # standalone quiz
```

## Environments

| Task | Ubuntu 24.04 | Rocky Linux 9.8 | EC2 Amazon Linux |
|---|---|---|---|
| All lab tasks | ⚠️ [VERIFY] — `cryptsetup` is in Ubuntu 24.04's default `apt` repository (candidate 2:2.7.0-1ubuntu4.2 confirmed via `apt-cache policy` in this session) but neither installed nor executed; this session's container has no `dm-crypt` module or `/dev/mapper` support at all | ⚠️ [VERIFY] — not checked this session; `cryptsetup` ships in Rocky9's default repos as a standard package | Not required |

## Recall Check — 05-raid-mdadm

Answer these from memory before continuing — no looking back at Demo M04-05.

1. Does RAID protect against accidental file deletion or ransomware? Why or why not?

2. A RAID1 array shows `[2/1] [U_]` in `/proc/mdstat`. Is the array currently down?

3. What is RAID split-brain, and how does `mdadm` guard against it?

<details>
<summary>Answers</summary>

1. No — RAID has no concept of correct vs. incorrect data. It faithfully mirrors or stripes a deletion or a ransomware pass across every member disk exactly as reliably as legitimate writes. RAID protects against physical disk failure only.

2. No — a degraded RAID1 array keeps serving reads and writes normally. What's lost is redundancy, not current availability.

3. Two mirror members ending up with genuinely different data after being separated and independently written to. `mdadm` guards against this with per-member event counts on each disk's superblock — a member with a significantly lower count is treated as stale and never silently reintegrated.

</details>

---

## Part 1 — Concepts

### 1.1 What LUKS/dm-crypt actually protects — and what it doesn't

Per the `cryptsetup(8)` manual page, `cryptsetup` configures `dm-crypt`
managed device-mapper mappings — the kernel does the actual encryption
and decryption transparently, below the filesystem layer, once a volume
is unlocked. LUKS is an extension that adds a standardized on-disk header
and metadata format on top of raw `dm-crypt`, enabling multiple
passwords and secure key management (§1.6) that plain `dm-crypt` alone
doesn't provide.

**What this protects:** data at rest — the actual bytes on the physical
disk, unreadable without the correct key, protecting against a disk
that's lost, stolen, decommissioned without wiping, or physically
accessed by someone who shouldn't have it.

**What this does not protect:**
- **Data while the volume is unlocked and mounted.** Once you've
  `luksOpen`'d a volume and it's mounted, it behaves like any normal
  filesystem — anyone with access to the running, unlocked system reads
  it in plaintext. LUKS protects the disk at rest, not a compromised
  running system.
- **A weak or guessable passphrase.** The encryption algorithm can be
  cryptographically sound and the whole scheme still fail if the
  passphrase is trivial to guess — this is a key-management problem, not
  a `cryptsetup` problem.
- **Damage to the LUKS header itself.** Covered explicitly in §1.8 — this
  is a distinct, LUKS-specific risk with its own required discipline.

### 1.2 `luksFormat` — initializing a LUKS volume

```bash
sudo cryptsetup luksFormat /dev/sdb1
```

Per the manual page, this initializes a LUKS partition and sets the
initial passphrase, either via an interactive prompt or a key file.
**This is destructive** — per the manual page's own explicit warning,
running `luksFormat` on a device that already contains a LUKS container
(or any other data) makes the previous contents permanently
irretrievable. There is no confirmation beyond `cryptsetup`'s own
interactive "are you sure" prompt (skippable with `--batch-mode`, which
should never be used carelessly in a script touching a device you
haven't triple-checked).

### 1.3 `luksOpen`/`open` — unlocking a volume

```bash
sudo cryptsetup open /dev/sdb1 secure-data
# older syntax, equivalent: cryptsetup luksOpen /dev/sdb1 secure-data
```

Per the manual page, this opens the LUKS partition and, after verifying
the supplied passphrase (or key file) against one of the volume's key
slots (§1.5), sets up a mapping under the given name. The unlocked,
decrypted device then appears at `/dev/mapper/secure-data` — from this
point on, it's usable exactly like any other block device.

```bash
sudo cryptsetup close secure-data
# older syntax, equivalent: cryptsetup luksClose secure-data
```

`close` (or `luksClose`) removes the mapping — the underlying disk
reverts to being unreadable ciphertext until opened again.

### 1.4 Using the mapped device — Demo M04-01's pattern, one layer down

Once unlocked, `/dev/mapper/secure-data` is a normal block device.
Everything from Demo M04-01 applies directly:

```bash
sudo mkfs.ext4 /dev/mapper/secure-data
sudo mkdir -p /mnt/secure-data
sudo mount /dev/mapper/secure-data /mnt/secure-data
```

This is the core mental model worth having precisely: LUKS/`dm-crypt`
sits as one additional layer *below* the filesystem and *above* the raw
partition — `partition → LUKS/dm-crypt → filesystem → mount`, one link
added to the chain Demo M04-01 already taught, not a replacement for any of
those steps.

### 1.5 `luksDump` — reading header and key-slot information

```bash
sudo cryptsetup luksDump /dev/sdb1
```

Per Red Hat's LUKS documentation and the manual page, this reports the
LUKS header's contents — version (LUKS1 vs. LUKS2, §1.8), UUID, and each
key slot's state:

```
LUKS header information for /dev/sdb1
Version: 1
Cipher name: aes
Cipher mode: xts-plain64
Hash spec: sha256
UUID: 1da14492-aec4-4924-905d-e5aa28cbcff4
Key Slot 0: ENABLED
Key Slot 1: DISABLED
Key Slot 2: DISABLED
...
Key Slot 7: DISABLED
```

LUKS1 provides 8 key slots (LUKS2 provides more, and the exact number is
configurable). Each `ENABLED` slot holds an independently-set passphrase
or key file capable of unlocking the *same* underlying volume key — this
is what makes multi-passphrase management and clean revocation possible
(§1.6), without ever needing to re-encrypt the actual bulk data.

### 1.6 `luksAddKey` — multiple passphrases, and why that matters

```bash
sudo cryptsetup luksAddKey /dev/sdb1
# prompts for an EXISTING passphrase to authorize the change,
# then the NEW passphrase to add to the next available key slot
```

Per a real, independently-documented transcript of this exact workflow:
adding a key requires unlocking with one of the currently enabled slots
first (proving you already have legitimate access), after which the new
passphrase occupies the next free slot — `luksDump` afterward shows two
`ENABLED` slots instead of one.

**Why this matters operationally:** a team can each hold an individually
revocable passphrase to the same encrypted volume — when someone leaves
the team, their specific key slot is disabled (`cryptsetup luksKillSlot`)
without needing to re-encrypt the volume or distribute a new shared
passphrase to everyone else. This is the same "individually revocable
credentials" principle behind per-user SSH keys rather than one shared
key, applied to disk encryption.

### 1.7 Persistence — `/etc/crypttab`, and the "locked after reboot" trap

An unlocked LUKS volume from a manual `cryptsetup open` is exactly as
temporary as an unpersisted `mount` (Demo M04-01) or an `ip addr add` without
a config file (Demo M03-01) — it does not survive a reboot on its own.

Per Red Hat's LUKS documentation, persisting automatic unlocking requires
an entry in `/etc/crypttab`:

```bash
sudo cryptsetup luksUUID /dev/sdb1
# a52e2cc9-a5be-47b8-a95d-6bdf4f2d9325

echo 'secure-data UUID=a52e2cc9-a5be-47b8-a95d-6bdf4f2d9325 none' | sudo tee -a /etc/crypttab
```

The three fields are: the mapping name (matches what appears under
`/dev/mapper/`), the volume's UUID (from `luksUUID`, the same
stability rationale as Demo M04-01's `/etc/fstab` UUID guidance), and the
key source (`none` means prompt interactively at boot; a key-file path
enables unattended unlocking, with its own separate security
considerations about protecting that key file). The mapped device then
still needs its own `/etc/fstab` entry, exactly as in Demo M04-01, pointing
at `/dev/mapper/secure-data` or its own filesystem UUID.

**A volume that's `cryptsetup open`'d manually but has no `/etc/crypttab`
entry will not automatically unlock on the next boot** — this is
precisely the same category of persistence trap Demo M04-01 covered for
mounts and Demo M03-01 covered for network config, just one layer further
down the stack. Part 3's Break-Fix scenario walks through exactly this.

### 1.8 LUKS1 vs. LUKS2, and the header-backup discipline

Current `cryptsetup` versions (2.x, matching the candidate version
confirmed available in this session's Ubuntu 24.04 environment) default
to creating LUKS2 volumes. LUKS2 supports a configurable number of key
slots (beyond LUKS1's fixed 8), and per Red Hat's documentation, supports
online re-encryption for LUKS2 volumes — LUKS1 requires unmounting for
these operations.

**The discipline that matters regardless of version:** per the
`cryptsetup` manual page's own explicit warning, damaging the LUKS
header is something people manage to do "with surprising frequency" —
and unlike most storage failures, header damage on a LUKS volume is
catastrophic *by design*, not by accident of implementation. The header
holds the encrypted copies of the actual volume key; without an intact
header (or a backup of it), **the underlying data is permanently
unrecoverable, even with the correct passphrase** — there's no "your
data is technically still there, just needs recovery tools" fallback the
way there sometimes is for a damaged filesystem.

```bash
sudo cryptsetup luksHeaderBackup /dev/sdb1 --header-backup-file /root/sdb1-luks-header.bin
```

Store this backup somewhere physically separate from the disk it backs
up — a header backup stored only on the same disk it protects is
useless for exactly the failure mode it exists to guard against.

## Part 2 — Hands-On Lab

**Before starting:** confirm `cryptsetup` is installed (`which
cryptsetup`; install with `sudo apt install cryptsetup` / `sudo dnf
install cryptsetup-luks` if not) and that `/dev/mapper` and the
`dm-crypt` module are available (`sudo modprobe dm-crypt` should succeed
silently). This lab uses a loop device, so no real disk is required or
put at risk. This lab was not run in the environment that authored it —
treat every `# Expected` comment as documented behavior to confirm.

### Commands used in this demo

| Command | Flag meaning | Purpose |
|---|---|---|
| `cryptsetup luksFormat` | — | Initialize a LUKS volume (destructive) |
| `cryptsetup open` / `luksOpen` | — | Unlock a volume, create a `/dev/mapper` mapping |
| `cryptsetup close` / `luksClose` | — | Remove the mapping, re-lock the volume |
| `cryptsetup luksDump` | — | Show header info and key-slot states |
| `cryptsetup luksAddKey` | — | Add an additional passphrase to a free key slot |
| `cryptsetup luksKillSlot` | — | Disable/revoke one specific key slot |
| `cryptsetup luksUUID` | — | Print the volume's UUID, for `/etc/crypttab` |
| `cryptsetup luksHeaderBackup` | — | Back up the LUKS header to a separate file |

### Task 1 — Create a loop device and initialize LUKS

```bash
dd if=/dev/zero of=/tmp/luks-disk.img bs=1M count=256
sudo losetup /dev/loop4 /tmp/luks-disk.img

sudo cryptsetup luksFormat /dev/loop4
# Confirm the destructive-action prompt, type YES, set a test passphrase
```

### Task 2 — Unlock, format, and mount the decrypted volume

```bash
sudo cryptsetup open /dev/loop4 secure-data
ls /dev/mapper/
# Expected: secure-data present under /dev/mapper/

sudo mkfs.ext4 /dev/mapper/secure-data
sudo mkdir -p /mnt/secure-data
sudo mount /dev/mapper/secure-data /mnt/secure-data
echo "sensitive test content" | sudo tee /mnt/secure-data/secret.txt
cat /mnt/secure-data/secret.txt
```

### Task 3 — Lock it, and confirm the underlying disk is unreadable

```bash
sudo umount /mnt/secure-data
sudo cryptsetup close secure-data
ls /dev/mapper/
# Expected: secure-data no longer listed

file -s /dev/loop4
# Expected: reports the device as encrypted/random-looking data, not
# a recognizable ext4 filesystem — confirm the raw device gives no
# indication of its contents without unlocking it first
```

### Task 4 — Inspect the header and add a second key slot

```bash
sudo cryptsetup luksDump /dev/loop4
# Expected: Key Slot 0: ENABLED, all others DISABLED

sudo cryptsetup luksAddKey /dev/loop4
# Enter the EXISTING passphrase from Task 1, then a new second passphrase

sudo cryptsetup luksDump /dev/loop4 | grep "Key Slot"
# Expected: Key Slot 0 and Key Slot 1 both ENABLED now

sudo cryptsetup open /dev/loop4 secure-data-test
# Confirm EITHER passphrase successfully unlocks the volume
sudo cryptsetup close secure-data-test
```

### Task 5 — Persist unlocking via `/etc/crypttab`

```bash
sudo cryptsetup luksUUID /dev/loop4
# Note the UUID printed

echo 'secure-data UUID=<paste-the-uuid-here> none' | sudo tee -a /etc/crypttab
cat /etc/crypttab
# Confirm the entry is present, matching the format in §1.7
```

### Task 6 — Back up the header, then clean up

```bash
sudo cryptsetup luksHeaderBackup /dev/loop4 --header-backup-file /tmp/loop4-header-backup.bin
ls -lh /tmp/loop4-header-backup.bin
# Confirm the backup file was created — in a real scenario this file
# would be moved somewhere physically separate from this disk

# Clean up (lab only):
sudo losetup -d /dev/loop4
rm -f /tmp/luks-disk.img /tmp/loop4-header-backup.bin
sudo sed -i '/secure-data/d' /etc/crypttab
```

## What You Learned

1. ✅ Explained what LUKS/dm-crypt protects (data at rest) and what it
   doesn't (an unlocked running system, a weak passphrase, header damage)
2. ✅ Initialized a LUKS volume with `luksFormat` and unlocked it with
   `open`/`luksOpen`, understanding the key-slot passphrase model
3. ✅ Used the resulting `/dev/mapper` device exactly like any block
   device — partitioned, formatted, and mounted it per Demo M04-01's pattern
4. ✅ Read `luksDump` output and managed multiple key slots with
   `luksAddKey`
5. ✅ Persisted automatic unlocking via `/etc/crypttab` and understand
   why a manually-opened volume doesn't survive a reboot on its own
6. ✅ Explained the LUKS1/LUKS2 distinction and the header-backup
   discipline that prevents unrecoverable data loss

## Part 3 — Break-Fix Scenario

### Scenario — NovaTech Ops: two LUKS incidents

> **Incident report — NovaTech Ops, ticket #4745**
>
> Alert 1: "a server rebooted after scheduled maintenance and a data
> volume that was definitely working before the reboot is now completely
> inaccessible — the mount point is just empty, and there's no error in
> the boot log that looks obviously related." Alert 2: "we're trying to
> recover an old decommissioned server's encrypted drive using the
> correct, documented passphrase, and `cryptsetup` refuses to open it —
> `luksDump` reports the header looks corrupted, and there's no header
> backup on file anywhere."

**Incident 1 — volume inaccessible after reboot, correct passphrase not even prompted**

Tasks:
1. Referencing §1.7, what's the most likely explanation for a volume
   that was manually `cryptsetup open`'d and mounted, now showing no
   prompt for a passphrase at all after reboot?
2. What specific file, and what specific check, confirms this diagnosis?
3. What's the fix, and how does it parallel a mistake this series has
   already covered in a different context?

**Incident 2 — correct passphrase, but the volume won't open**

Tasks:
1. Referencing §1.8, why does having the *correct* passphrase not
   guarantee a LUKS volume can still be unlocked?
2. What should have existed before this became an active incident, and
   why does "the disk itself is physically fine" not help here?
3. What's the actual, honest answer to give the stakeholder asking for
   this data back, given no header backup exists?

<details>
<summary>Answers</summary>

**Incident 1:** This is the "locked after reboot" trap from §1.7. A
volume unlocked with a manual `cryptsetup open` (or `luksOpen`) command,
with no corresponding `/etc/crypttab` entry, has no way to automatically
re-unlock at boot — the system has no instruction to prompt for or apply
a passphrase to that device at all, so it stays as unreadable ciphertext,
and nothing mounts on top of it, producing exactly the reported
"mount point is just empty" symptom with no obviously related boot error
(there's nothing to error about — from the system's perspective, that
volume was simply never part of the boot sequence). The specific check:
`cat /etc/crypttab` — confirm whether an entry exists for this volume's
UUID at all. The fix: add the missing entry (§1.7's format), following
`cryptsetup luksUUID` to get the correct UUID, then re-run `cryptsetup
open` manually once now, and confirm the array unlocks cleanly on a test
reboot rather than assuming the crypttab entry alone is sufficient
without verification. This is the same category of persistence mistake
as Demo M04-01's unpersisted `mount` command and Demo M03-01's `ip addr add`
without a config file — "it worked when I ran it manually" and "it
survives a reboot" are different claims, one layer further down the
storage stack than Demo M04-01's version of this same lesson.

**Incident 2:** Per §1.8, LUKS's encryption model means the actual bulk
data is encrypted with a volume key, and that volume key itself is what's
protected by each passphrase-guarded key slot in the header — the
passphrase alone, without an intact header to decrypt with it, unlocks
nothing. If the header is damaged (a stray write, a partition tool
run against the wrong device, accidental `luksFormat` reuse, or any of
the "surprising frequency" causes the manual page warns about), there is
no path from "I know the correct passphrase" to "I can read the data,"
because the thing the passphrase actually unlocks no longer exists in a
usable form. What should have existed beforehand: a `luksHeaderBackup`
(§1.8), stored somewhere physically separate from the disk itself —
this is precisely the scenario that backup exists for, and its absence
here is the actual root cause of this being unrecoverable, not the header
damage itself, which is a known, expected-to-happen-eventually risk.
"The disk itself is physically fine" doesn't help because the failure
isn't physical — the data on the disk is intact ciphertext, but with no
intact header and no backup of one, there's no cryptographic path back
to the plaintext. The honest answer to the stakeholder: this data is not
recoverable through normal means; a header backup is what would have
prevented this, and going forward, header backups need to be a mandatory,
verified step at LUKS volume creation time (§1.8), not an
optional afterthought.

</details>

## Interview Prep

**Q1. A stakeholder asks "our disks are LUKS-encrypted, so we're covered if the server itself is compromised while running, right?"**

No — this is a common conflation of two different threat models. LUKS
protects data *at rest*: an unmounted, powered-off, or physically
separated disk. Once a volume is unlocked and mounted on a running
system, it behaves like any normal filesystem — anyone with access to
that running, unlocked system (a compromised account, an attacker with
shell access) reads the data in plaintext exactly as if it weren't
encrypted at all. LUKS is the right answer for "what happens if this
disk is stolen or decommissioned without wiping," not for "what happens
if this running server is compromised."

**Q2. A team wants shared access to an encrypted volume but also wants to be able to revoke one person's access without affecting everyone else. How does LUKS's key-slot model support this?**

Each LUKS key slot holds an independently-set passphrase or key file
capable of unlocking the same underlying volume key — per `luksDump`,
you can see exactly which slots are enabled. Each team member gets their
own passphrase added to their own slot via `luksAddKey`. When someone
leaves, `luksKillSlot` disables just their slot — everyone else's
passphrase keeps working unchanged, and the actual bulk data never needs
re-encrypting. This mirrors the same individually-revocable-credential
principle as per-user SSH keys instead of one shared key.

**Q3. Why is a LUKS header backup non-negotiable in a way that most storage backups aren't?**

Because header damage is a *total and permanent* loss of access to the
data, even with the correct passphrase — the header holds the encrypted
copy of the volume key that the passphrase actually unlocks; without an
intact header, the passphrase has nothing to decrypt. Per `cryptsetup`'s
own manual page, header damage happens "with surprising frequency," not
as a rare edge case, which makes `luksHeaderBackup` — stored physically
separate from the disk it protects — a mandatory step at volume creation
time, not an optional afterthought the way, say, an extra filesystem
snapshot might be for an unencrypted volume.

## Key Takeaways

---

1. **LUKS/dm-crypt protects data at rest — a disk that's lost, stolen, or decommissioned — not data on a running, unlocked system.** Once a volume is `open`'d and mounted, it's readable plaintext to anything with access to that running system; conflating "encrypted at rest" with "safe if the server is compromised" is a common, costly mistake.
2. **LUKS sits as one layer between the raw partition and the filesystem: `partition → LUKS/dm-crypt → filesystem → mount`.** Once unlocked, the resulting `/dev/mapper/name` device is a completely normal block device — every `mkfs`/`mount` step from Demo M04-01 applies to it unchanged.
3. **Key slots (8 on LUKS1, more on LUKS2) let multiple independent passphrases unlock the same underlying volume key.** This enables individually revocable access (`luksAddKey` to grant, `luksKillSlot` to revoke) without ever needing to re-encrypt the actual bulk data — the same principle as per-user SSH keys over one shared key.
4. **A manually `cryptsetup open`'d volume does not survive a reboot without an `/etc/crypttab` entry.** This is the identical persistence trap as an unpersisted Demo M04-01 `mount` command or a Demo M03-01 `ip addr add` — it works until the next reboot, then silently doesn't, with no obviously related error to point at the cause.
5. **LUKS header damage is catastrophic by design, not by implementation accident — the passphrase alone unlocks nothing without an intact header, even when the passphrase is correct and the underlying disk is physically fine.** Per `cryptsetup`'s own documentation, this happens with real, non-negligible frequency.
6. **`luksHeaderBackup`, stored physically separate from the disk it protects, is the only thing standing between header damage and total, permanent, unrecoverable data loss.** This makes it a mandatory step at volume creation, not an optional extra the way a filesystem snapshot might be for unencrypted storage.
7. **A weak or guessable passphrase defeats the entire scheme regardless of how sound the underlying cipher is.** LUKS's cryptographic strength says nothing about key-management discipline — those are separate concerns, and the second one is usually the actual weak point in practice.

> **Demo scope:** Primary concept: LUKS/dm-crypt as an added layer in the
> Demo M04-01 storage stack, and the header-backup discipline that prevents
> unrecoverable loss.
> Supporting concepts: luksFormat/open/close, key slots and luksAddKey,
> crypttab persistence, LUKS1 vs LUKS2, header backup.
> Estimated completion time: 35 minutes (reading + hands-on + verification),
> ASSUMING cryptsetup and dm-crypt are available — this demo's own
> authoring environment had neither.
> Checkpoints: 6 natural stopping points (after Task 1 through Task 6).
> **Verification status:** built from official documentation, not live
> execution in this session — see the note at the top of this file.

## Quick Commands Reference

### Setup and unlocking

| Command | Purpose |
|---|---|
| `cryptsetup luksFormat /dev/sdX` | Initialize a LUKS volume (destructive) |
| `cryptsetup open /dev/sdX name` | Unlock, create `/dev/mapper/name` |
| `cryptsetup close name` | Remove the mapping, re-lock |

### Inspection and key management

| Command | Purpose |
|---|---|
| `cryptsetup luksDump /dev/sdX` | Show header info and key-slot states |
| `cryptsetup luksAddKey /dev/sdX` | Add a passphrase to a free key slot |
| `cryptsetup luksKillSlot /dev/sdX N` | Disable/revoke one key slot |
| `cryptsetup luksUUID /dev/sdX` | Print the volume's UUID |

### Persistence and safety

| Command | Purpose |
|---|---|
| `/etc/crypttab` entry | `name UUID=xxx none` — persist auto-unlock at boot |
| `cryptsetup luksHeaderBackup /dev/sdX --header-backup-file FILE` | Back up the LUKS header |

## Appendix A — Anki Cards

**`m04-06-luks-encryption-anki.csv`:**

```
#deck:Linux Mastery::Module 4 - Storage::Demo M04-06 - Encryption at Rest (LUKS/dm-crypt)
#separator:Comma
#columns:Front,Back,Tags
"Does LUKS encryption protect data on a running system that has already been unlocked and mounted?","No — once a LUKS volume is opened and mounted, it behaves like any normal filesystem. Anyone with access to that running, unlocked system reads the data in plaintext. LUKS protects data AT REST (disk lost, stolen, decommissioned), not a compromised running system.","demo-m04-06,luks,fundamentals"
"Where does an unlocked LUKS volume appear once cryptsetup open succeeds?","At /dev/mapper/<name> — a completely normal block device from that point on. Every mkfs/mount step from Demo M04-01 applies to it unchanged.","demo-m04-06,luks,fundamentals,cross-demo"
"How many key slots does LUKS1 provide, and what does having multiple slots enable?","8 key slots on LUKS1 (LUKS2 supports more, configurable). Multiple independent passphrases can each unlock the same underlying volume key, enabling individually revocable access without needing to re-encrypt the actual bulk data.","demo-m04-06,luks,keyslots,fundamentals"
"A LUKS volume was manually unlocked with cryptsetup open and mounted, then the server rebooted. Is it still mounted and accessible?","No — a manually-opened volume with no /etc/crypttab entry does not survive a reboot. This is the same persistence trap as an unpersisted mount command (Demo M04-01) or ip addr add (Demo M03-01) — it works until the next reboot, then silently doesn't.","demo-m04-06,luks,crypttab,break-fix,cross-demo"
"What are the three fields in an /etc/crypttab entry?","The mapping name (matches /dev/mapper/<name>), the volume's UUID (from cryptsetup luksUUID), and the key source (none for an interactive prompt at boot, or a key-file path for unattended unlocking).","demo-m04-06,luks,crypttab,fundamentals"
"You have the CORRECT passphrase for a LUKS volume, but the header is corrupted. Can you still recover the data?","No — the passphrase unlocks the volume key stored IN the header. Without an intact header, there's nothing for the passphrase to decrypt, even though it's correct and the underlying disk is physically fine. This is a total, permanent loss without a header backup.","demo-m04-06,luks,header,break-fix"
"What does cryptsetup luksHeaderBackup do, and where should the backup be stored?","Backs up the LUKS header (which holds the encrypted volume key) to a separate file. It must be stored physically separate from the disk it protects — a backup stored only on the same disk is useless for the exact failure mode (header damage) it exists to guard against.","demo-m04-06,luks,header,backup,fundamentals"
"How does luksKillSlot let a team revoke one person's access to a shared encrypted volume without affecting anyone else?","Each team member has their own passphrase in their own key slot (added via luksAddKey). luksKillSlot disables just that one slot — everyone else's passphrase keeps working unchanged, and the actual bulk data never needs re-encrypting.","demo-m04-06,luks,keyslots,revocation"
"Does a cryptographically strong cipher guarantee a LUKS volume is actually secure?","No — a weak or guessable passphrase defeats the entire scheme regardless of cipher strength. Encryption algorithm soundness and key-management discipline (passphrase strength) are separate concerns, and the second is usually the actual weak point in practice.","demo-m04-06,luks,security,fundamentals"
"What layer does LUKS/dm-crypt occupy in the storage stack, relative to Demo M04-01's partition/filesystem/mount sequence?","partition -> LUKS/dm-crypt -> filesystem -> mount. LUKS is inserted as one additional layer between the raw partition and the filesystem, not a replacement for any of Demo M04-01's steps.","demo-m04-06,luks,fundamentals,cross-demo"
```

## Appendix B — Quiz

**`m04-06-luks-encryption-quiz.md`:**

````markdown
# Quiz — Demo M04-06: Encryption at Rest (LUKS/dm-crypt)

> One correct answer per question unless stated otherwise.
> Target: 80% or above before moving to the next demo.

---

**Q1.** A server with LUKS-encrypted disks is compromised while running, with the encrypted volume already unlocked and mounted. Does LUKS prevent the attacker from reading the data?

- A) Yes — LUKS encryption remains active even on a mounted, unlocked volume
- B) No — once unlocked and mounted, the volume behaves like any normal filesystem; LUKS protects data at rest (an unmounted, powered-off, or physically separated disk), not a compromised running system
- C) Only if the attacker doesn't have root access
- D) Yes, but only for files created after the compromise began

<details>
<summary>Answer</summary>

**B** — This is the core threat-model distinction: LUKS answers "what if this disk is stolen or decommissioned," not "what if this running server is compromised." An unlocked, mounted volume is plaintext to anything with access to that running system.

Trap: A wrongly assumes encryption persists through the unlock; C and D both invent conditions that don't actually change this fundamental limitation.

</details>

---

**Q2.** Where does a LUKS volume appear once `cryptsetup open /dev/sdb1 myvolume` succeeds, and what can be done with it?

- A) At `/dev/sdb1_decrypted` — it must be accessed with special LUKS-aware tools only
- B) At `/dev/mapper/myvolume` — a completely normal block device from that point on, usable with standard `mkfs`/`mount` exactly as covered in Demo M04-01
- C) Nowhere new — the original `/dev/sdb1` path is used directly, now transparently decrypted
- D) At `/mnt/myvolume` automatically, already mounted

<details>
<summary>Answer</summary>

**B** — The mapping appears under `/dev/mapper/`, named as specified, and behaves as an ordinary block device requiring the same `mkfs`/`mount` steps as any unencrypted partition.

Trap: A invents a nonexistent naming convention and unnecessary special tooling; C wrongly claims the original device path becomes usable directly; D wrongly assumes automatic mounting, which `open` alone doesn't do.

</details>

---

**Q3.** A LUKS1 volume shows `Key Slot 0: ENABLED` and all other slots `DISABLED` in `luksDump`. What does adding a second passphrase with `luksAddKey` require?

- A) Nothing beyond the new passphrase itself
- B) An existing, already-valid passphrase (proving legitimate access) must be supplied first, before the new passphrase is added to the next free slot
- C) The volume must be completely re-encrypted with a new cipher
- D) The disk must be unmounted and the array rebuilt from scratch

<details>
<summary>Answer</summary>

**B** — `luksAddKey` requires authenticating with a currently valid passphrase before it will add a new one — this prevents anyone without existing access from simply adding themselves a key.

Trap: A skips the authentication requirement; C and D both invent unnecessary, disruptive operations that `luksAddKey` doesn't require.

</details>

---

**Q4.** A volume was unlocked with `cryptsetup open` and mounted manually. The server reboots. What's the expected state afterward, assuming no `/etc/crypttab` entry exists for it?

- A) It remains mounted and accessible, since LUKS state persists automatically
- B) It's locked (not unlocked) and not mounted — nothing in the boot process knows to unlock or mount it without a crypttab entry
- C) The system will fail to boot entirely
- D) It automatically re-mounts read-only as a safety fallback

<details>
<summary>Answer</summary()>

**B** — This is the identical persistence trap covered elsewhere in the series for unpersisted mounts and network config — a manual action doesn't survive a reboot without a corresponding permanent configuration entry, here specifically `/etc/crypttab`.

Trap: A wrongly assumes persistence without configuration; C invents a boot failure that doesn't occur for an unrelated, non-boot-critical volume; D invents an automatic fallback behavior that doesn't exist.

</details>

---

**Q5.** You have the correct, verified passphrase for a LUKS volume, but its header has been damaged. Can the data be recovered?

- A) Yes, always — the correct passphrase guarantees recoverability regardless of header state
- B) No, not without a prior header backup — the passphrase unlocks the volume key stored in the header itself; without an intact header, there's nothing for the passphrase to decrypt
- C) Yes, but only using specialized data-recovery hardware
- D) No — LUKS-encrypted data can never be recovered under any circumstances, even with an intact header and correct passphrase

<details>
<summary>Answer</summary>

**B** — This is the specific, severe risk `luksHeaderBackup` exists to protect against. A correct passphrase is necessary but not sufficient — it needs an intact header (or a backup of one) to actually decrypt anything.

Trap: A wrongly assumes the passphrase alone is sufficient; C invents a hardware-based recovery path that doesn't apply to a cryptographic (not physical) failure; D overstates the impossibility — recovery IS possible with a header backup, which is exactly the point.

</details>

---

**Q6.** Why does the `cryptsetup` manual page specifically warn about protecting the LUKS header, rather than treating it as just another risk among many?

- A) The header is the largest file on the encrypted volume, so it's the most likely to get corrupted
- B) Header damage is described as happening "with surprising frequency," and unlike most storage failures, it produces total, permanent, unrecoverable data loss even with an otherwise-intact disk and the correct passphrase
- C) The header is stored on a separate, less reliable disk by default
- D) This warning applies only to LUKS1, not LUKS2

<details>
<summary>Answer</summary>

**B** — The combination of (a) a real, non-negligible frequency of accidental header damage and (b) that damage being catastrophic rather than merely inconvenient is exactly why this gets explicit, dedicated warning treatment rather than being folded into general storage-safety advice.

Trap: A invents a size-based explanation that isn't the actual reasoning; C invents a separate-disk-by-default claim that isn't accurate; D wrongly limits the warning's scope — header damage risk isn't LUKS1-specific.

</details>

---

**Q7.** A team wants shared access to an encrypted volume, with the ability to revoke one departing member's access without affecting anyone else or re-encrypting the data. What LUKS feature enables this?

- A) A single shared passphrase, changed whenever someone leaves
- B) Individual key slots per person — `luksAddKey` grants each person their own slot, and `luksKillSlot` revokes just one slot without affecting others or requiring re-encryption
- C) LUKS does not support this use case; a different encryption tool is required
- D) Separate, fully independent LUKS volumes per team member

<details>
<summary>Answer</summary()>

**B** — This is exactly what multiple key slots are for: independently revocable credentials to the same underlying volume key, mirroring the principle behind per-user SSH keys over one shared key.

Trap: A describes a workable but clumsy alternative that requires redistributing a new passphrase to everyone on every departure; C wrongly claims LUKS can't do this; D invents unnecessary volume duplication that isn't how key slots work.

</details>

---

**Q8.** Once unlocked, is a LUKS/`dm-crypt` volume's usage (partitioning, filesystem creation, mounting) different from an unencrypted block device's, per this demo and Demo M04-01?

- A) Yes — LUKS-specific tools replace `mkfs` and `mount` entirely
- B) No — once mapped under `/dev/mapper/`, the exact same `mkfs`/`mount` steps from Demo M04-01 apply unchanged; LUKS is an added layer below the filesystem, not a replacement for it
- C) Yes — LUKS volumes cannot use `ext4` and require a LUKS-specific filesystem
- D) Only the mount step differs; formatting is identical

<details>
<summary>Answer</summary>

**B** — LUKS/`dm-crypt` inserts itself as a layer between the raw partition and the filesystem (`partition → LUKS/dm-crypt → filesystem → mount`); once unlocked, everything above that layer is completely standard, unmodified Demo M04-01 territory.

Trap: A and C both invent LUKS-specific requirements that don't exist — any normal filesystem works fine on a mapped LUKS device; D wrongly claims a partial difference when there's actually none.

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