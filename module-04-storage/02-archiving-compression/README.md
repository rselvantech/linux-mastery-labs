# Demo M04-02 — Archiving & Compression

## Overview

Two operations get treated as one, and that confusion is the root cause of most archiving mistakes: **archiving** bundles many files into a single stream (`tar`), and **compression** shrinks that stream (`gzip`, `bzip2`, `xz`). `tar` by itself doesn't compress anything — it's a shipping container, not a vacuum bag. Demo M04-01 covered provisioning storage; this demo covers moving data off it safely — backups, log shipping, and the exact commands a NovaTech deploy script runs every night.

By the end of this demo you'll know why `tar czf` isn't one command but three ideas glued together, which compressor to reach for and why, and how to avoid the two mistakes that account for almost every real-world archiving incident: a flag-order slip that silently writes the wrong file, and a "tarbomb" that scatters files across whatever directory you happened to be standing in when you extracted it.

## Demo Objectives

1. Explain why archiving and compression are separate operations, and compose them explicitly before using `tar`'s combined flags
2. Create, inspect, and extract `tar` archives safely — including `-C`, `--strip-components`, and why extracting into a fresh directory matters
3. Choose between `gzip`, `bzip2`, and `xz` based on their actual ratio/speed trade-off, not habit
4. Verify archive integrity with `gzip -t` and `sha256sum`, and recognize what a corrupted archive's failure actually looks like
5. Build archives safely from filenames with spaces, and selectively extract or exclude members
6. Use `zip`/`unzip` for cross-platform interoperability, and know when it's the right choice over `tar`

## Directory Structure

```
02-archiving-compression/
├── README.md                              # this file
├── m04-02-archiving-compression-anki.csv  # Anki flashcard deck
└── m04-02-archiving-compression-quiz.md   # standalone quiz
```

## Environments

| Task | Ubuntu 24.04 | Rocky Linux 9.8 | EC2 Amazon Linux |
|---|---|---|---|
| All lab tasks | ✅ verified in this session (tar 1.35, gzip 1.12, bzip2 1.0.8, xz 5.4.5) | ⚠️ [VERIFY] — not run this session; all four tools ship by default on RHEL9/Rocky9 as base packages, but exact versions and behavior weren't independently confirmed here | Not required |

## Recall Check — 01-storage-fundamentals

Answer these from memory before continuing — no looking back at Demo M04-01.

1. `df` shows a filesystem at 95% used; `du -sh` on the same path only accounts for half that space. What's the most likely cause?

2. A filesystem is mounted with a plain `mount` command but has no `/etc/fstab` entry. What happens after a reboot, and why?

3. What should you always run before partitioning a disk, and why?

<details>
<summary>Answers</summary>

1. A deleted file still held open by a running process — `df` counts it as used (the filesystem hasn't reclaimed the blocks); `du` only sees what's currently visible in the directory tree and can't account for a deleted file at all.

2. It disappears — a mount without a corresponding fstab entry is temporary. Nothing tells the system to remount it automatically at boot.

3. `lsblk` (ideally `lsblk -f`) — it's the cheapest way to confirm you're about to operate on the intended disk, not the root filesystem, where a mistake can mean destroyed data.

</details>

---

## Part 1 — Concepts

### 1.1 Archiving and compression are two different jobs

```bash
du -sh src/app
# 228K   src/app

tar -cf app.tar -C src app
ls -lh app.tar
# -rw-r--r-- 1 root root 220K app.tar
```
> **Observation:** the archive (220K) is *smaller* than the source (228K) — but only because `tar` doesn't preserve the filesystem's block-rounding overhead, not because it compressed anything. `tar` alone concatenates file contents plus a small amount of header metadata per file; it does not shrink the data itself.

`tar` = **t**ape **ar**chive — the name is a fossil from when the primary target was literally a tape drive, and it stuck. Its whole job is bundling many files (and their names, permissions, and directory structure) into one stream. Compression is a completely separate concern, bolted on afterward:

```bash
tar -czf app.tar.gz -C src app
ls -lh app.tar app.tar.gz
# -rw-r--r-- 1 root root 220K app.tar
# -rw-r--r-- 1 root root  26K app.tar.gz
```
> **Observation:** adding `-z` shrank the archive from 220K to 26K. The `-z` flag doesn't do anything `tar`-specific — it pipes the archive stream through `gzip` internally. Prove this by composing the same result manually:

```bash
tar -cf - -C src app | gzip > app-manual.tar.gz
sha256sum <(gzip -dc app.tar.gz) <(gzip -dc app-manual.tar.gz)
# f85e1d599777abbe7e2f5740ef9b66706866b373d10c1f895469ff4653e3840a  ...
# f85e1d599777abbe7e2f5740ef9b66706866b373d10c1f895469ff4653e3840a  ...
```
> **Observation:** identical checksums once decompressed — `tar -czf` and `tar -cf - | gzip` produce byte-identical results. `-czf` is convenience, not magic; it's `tar`'s combined-flag shorthand for a pipe you could write yourself. Understanding it as two steps is what makes the rest of this demo's flag combinations predictable instead of memorized.

**`tar`'s three mode flags** — exactly one of these is required, and it says what `tar` is doing, not what compression is in play:

| Flag | Letter meaning | What it does |
|---|---|---|
| `-c` | create | Build a new archive |
| `-x` | extract | Unpack an existing archive |
| `-t` | test/list | List an archive's contents without extracting |

**Compression selector flags** — combine with a mode flag, never used alone:

| Flag | Letter meaning | What it does |
|---|---|---|
| `-z` | gzip | Pipe through gzip |
| `-j` | bzip2 | Pipe through bzip2 |
| `-J` | xz | Pipe through xz |
| `-f` | file | The next argument is the archive filename — **almost always required**; without it, `tar` reads/writes the archive as a raw stream on stdin/stdout instead of a file |
| `-v` | verbose | Print each filename as it's processed |

### 1.2 Inspecting before extracting — never trust an archive blind

```bash
tar -tvf app.tar.gz
```
```
drwxr-xr-x root/root         0 2026-09-17 21:28 app/
drwxr-xr-x root/root         0 2026-09-17 21:28 app/config/
-rw-r--r-- root/root        31 2026-09-17 21:28 app/config/app.conf
-rw-r--r-- root/root        15 2026-09-17 21:28 app/README.md
drwxr-xr-x root/root         0 2026-09-17 21:28 app/logs/
-rw-r--r-- root/root     28000 2026-09-17 21:28 app/logs/error.log
-rw-r--r-- root/root    183575 2026-09-17 21:28 app/logs/access.log
```
> **Observation:** every path listed starts with `app/` — a single common top-level directory. This is the signature of a *well-formed* archive. §1.4 shows what it looks like when this signature is missing, and why that matters before you ever run `-x`.

```bash
file app.tar.gz
# app.tar.gz: gzip compressed data, from Unix, original size modulo 2^32 225280
file app.tar
# app.tar: POSIX tar archive (GNU)
```
`file` (introduced in Demo M01-07) identifies the archive by content, not extension — useful when you've received a file with no reliable name, or a `.tar` that's secretly something else.

**`-C` — extracting into a specific directory without `cd`-ing there first:**

```bash
mkdir -p restore1 restore2
tar -xzf app.tar.gz -C restore1
find restore1 -maxdepth 3
# restore1/app
# restore1/app/config
# restore1/app/config/app.conf
# restore1/app/README.md
# restore1/app/logs
# restore1/app/logs/error.log
# restore1/app/logs/access.log
```
> **Observation:** the full `app/` directory landed inside `restore1/`. `-C` changes `tar`'s working directory for the operation — equivalent to `cd restore1 && tar -xzf ../app.tar.gz`, but without leaving your shell's actual directory.

**`--strip-components` — dropping leading path segments on extraction:**

```bash
tar -xzf app.tar.gz -C restore2 --strip-components=1
find restore2 -maxdepth 3
# restore2/config
# restore2/config/app.conf
# restore2/README.md
# restore2/logs
# restore2/logs/error.log
# restore2/logs/access.log
```
> **Observation:** the `app/` prefix is gone — `restore2/config/app.conf`, not `restore2/app/config/app.conf`. `--strip-components=1` removes exactly one leading path segment from every member during extraction. This is the standard fix when a vendor's release tarball wraps everything in a version-specific directory (`myapp-2.4.1/`) you don't want to keep.

**Absolute paths get stripped automatically, with a warning:**

```bash
tar -cf abs-test.tar /home/claude/m04-02-lab/src/app/README.md
# tar: Removing leading `/' from member names
tar -tvf abs-test.tar
# -rw-r--r-- root/root        15 2026-09-17 21:28 home/claude/m04-02-lab/src/app/README.md
```
> **Observation:** `tar` refuses to store a member with a leading `/` — it strips it and warns, storing `home/claude/...` (relative) instead of `/home/claude/...` (absolute). This is a deliberate safety behavior: an archive that could write to `/etc/passwd` on extraction, just because that's where the source file happened to live, would be a serious security hazard. §7 in Part 2's Break-Fix section shows the related but distinct hazard this does *not* protect against.

### 1.3 Choosing a compressor — gzip, bzip2, xz, and zstd

These three ship on every mainstream distro and solve the same problem with different trade-offs:

```bash
tar -cf - -C src app > /tmp/plain.tar   # 220K uncompressed baseline
gzip  -9 -c /tmp/plain.tar > /tmp/plain.tar.gz    # 24K, 0.015s
bzip2 -9 -c /tmp/plain.tar > /tmp/plain.tar.bz2   # 16K, 0.012s
xz    -9 -c /tmp/plain.tar > /tmp/plain.tar.xz    #  8.0K, 0.120s
```
> **Observation:** on this specific 220K/mostly-text input, `xz` produced the smallest file by a wide margin (8.0K vs. gzip's 24K), `bzip2` landed in between (16K), and gzip finished fastest. ⚠️ [VERIFY — timing claim, docs only]: at this small file size, the timing differences shown are within noise (single-digit-to-low-double-digit milliseconds) and shouldn't be read as a general performance ranking — the widely-documented general relationship (xz has the best ratio and highest CPU cost; gzip is fastest and has the worst ratio; bzip2 sits between them) is consistent with what's shown here but wasn't independently confirmed against a production-scale dataset in this session.

| Tool | Name | Ratio (this test) | Relative speed | When to reach for it |
|---|---|---|---|---|
| `gzip` | GNU zip | Worst of the three (24K) | Fastest | Default choice — logs, general backups, anywhere CPU time matters more than a few extra saved bytes |
| `bzip2` | Block-sort ("bzip") compressor | Middle (16K) | Middle | Less common today; occasionally still specified by tooling that predates wide xz adoption |
| `xz` | (not an acronym — the container format's own name) | Best of the three (8.0K) | Slowest, most CPU-intensive | Cold storage, long-term archives, anywhere compression ratio matters more than CPU time and the data won't be touched again soon |

**A note on `zstd` — not installed in this environment, no lab steps run against it here:**
⚠️ [VERIFY] `zstd` (Facebook's Zstandard) is increasingly the default choice in production pipelines because it approaches `xz`-level ratios at speeds closer to `gzip` — but that specific ratio/speed claim was not independently verified in this session; `zstd` isn't installed on this Ubuntu 24.04 image and installing it requires network access this sandbox doesn't have. If you want it taught with a real lab section rather than this placeholder, run `sudo apt install zstd` on your own machine, then `zstd -19 -c /tmp/plain.tar | wc -c` and `time zstd -19 -c /tmp/plain.tar > /dev/null`, and share the output — it can be folded in as verified content rather than a flagged claim.

### 1.4 Integrity verification — before you trust a backup

An archive that looks fine can be silently corrupted. Two independent checks catch this:

```bash
gzip -t app.tar.gz
echo "exit: $?"
# exit: 0   (no output on success)
sha256sum app.tar.gz > app.tar.gz.sha256
cat app.tar.gz.sha256
# 37ea3ddf1dd6df32151467d148a4b7ec525fb1f419a86ea479eada84f689cb74  app.tar.gz
sha256sum -c app.tar.gz.sha256
# app.tar.gz: OK
```
`gzip -t` verifies the gzip stream's internal checksum — it catches corruption within the compressed data structure itself. `sha256sum` verifies the *whole file* hasn't changed since you recorded its hash — it catches corruption `gzip -t` might miss (e.g. truncation exactly at a chunk boundary) and is the only way to detect tampering or transfer corruption on an uncompressed `.tar`, which has no built-in integrity check at all.

**What real corruption looks like — one byte flipped:**

```bash
cp app.tar.gz app-corrupt.tar.gz
dd if=/dev/zero bs=1 count=1 seek=100 conv=notrunc of=app-corrupt.tar.gz
gzip -t app-corrupt.tar.gz
echo "exit: $?"
```
```
gzip: app-corrupt.tar.gz: invalid compressed data--crc error
gzip: app-corrupt.tar.gz: invalid compressed data--length error
exit: 1
```
> **Observation:** a single flipped byte anywhere in a compressed stream produces `invalid compressed data--crc error` — not a partial or degraded result. This is the nature of compressed formats: corruption doesn't localize to "one file inside the archive is bad," it can break the entire stream from that point forward. This is exactly why `gzip -t` (or `sha256sum -c` against a hash recorded at backup time) belongs in any real backup verification step — "the backup job completed" and "the backup is restorable" are different claims.

### 1.5 Selective extraction and exclusion

```bash
tar -xzf app.tar.gz -C restore3 app/config/app.conf
find restore3
# restore3/app
# restore3/app/config
# restore3/app/config/app.conf
```
> **Observation:** naming a specific member path after the archive filename extracts only that member — `restore3` never got `README.md` or `logs/`. The path given must match exactly what `-tvf` shows, including the `app/` prefix.

```bash
tar -czf app-nologs.tar.gz -C src --exclude='logs' app
tar -tzf app-nologs.tar.gz
# app/
# app/config/
# app/config/app.conf
# app/README.md
```
> **Observation:** `--exclude` takes a pattern matched against member paths during archive *creation* — the `logs/` directory and everything under it never made it into the archive at all, not filtered out afterward. `--exclude` must appear before the source path/files it's meant to apply to.

### 1.6 Building archives safely from filenames with spaces

Demo M05-06 covered why unquoted `xargs` breaks on filenames containing spaces, and the `find -print0` / `xargs -0` fix. The same hazard applies to building an archive from a dynamically generated file list:

```bash
find /tmp/mixed -name '*.txt' -print0 | tar --null -czf mixed.tar.gz --files-from=-
tar -tzf mixed.tar.gz
# tmp/mixed/report two.txt
# tmp/mixed/report one.txt
# tmp/mixed/normal.txt
```
> **Observation:** `report two.txt` is preserved as one member, not split into `report` and `two.txt`. `--files-from=-` tells `tar` to read the list of files to archive from stdin instead of the command line; `--null` tells it that list is null-delimited, matching `find -print0`'s output — the same null-safety pattern from Demo M05-06, applied to `tar` instead of `xargs`.

### 1.7 zip and unzip — cross-platform interoperability

```bash
zip -r -q app.zip src/app
unzip -l app.zip
```
```
Archive:  app.zip
  Length      Date    Time    Name
---------  ---------- -----   ----
        0  2026-09-17 21:28   src/app/
        0  2026-09-17 21:28   src/app/config/
       31  2026-09-17 21:28   src/app/config/app.conf
       15  2026-09-17 21:28   src/app/README.md
        0  2026-09-17 21:28   src/app/logs/
    28000  2026-09-17 21:28   src/app/logs/error.log
   183575  2026-09-17 21:28   src/app/logs/access.log
---------                     -------
   211621                     7 files
```
`zip` (the name is self-evident) bundles *and* compresses in a single step, unlike `tar` — there's no separate "plain zip, no compression" concept in normal use. `-r`: recursive, include subdirectories. `-q`: quiet, suppress the per-file listing during creation. `unzip -l`: list contents without extracting, `tar -tvf`'s equivalent.

**Why reach for `zip` at all, when `tar.gz` exists:** `.zip` is natively readable by Windows Explorer and macOS Finder with no extra tool — `.tar.gz` isn't, without something like 7-Zip installed. Choose `zip` specifically when the recipient's environment is unknown or is Windows; choose `tar` + a compressor for anything staying inside Linux/Unix infrastructure, where `tar`'s superior handling of permissions, symlinks, and ownership metadata matters more.

| | `tar` + compressor | `zip` |
|---|---|---|
| Preserves Unix permissions/ownership | Yes, faithfully | Partially — historically weaker, platform-dependent |
| Compress per-file or whole-stream | Whole-stream (can't extract one member without processing from the start in some cases) | Per-file — can extract one member instantly, without touching the rest |
| Native readability | Linux/Unix only, without extra tooling | Windows, macOS, Linux all read it natively |
| Typical use in this series | Backups, log shipping, Linux-to-Linux transfer | Deliverables crossing into non-Unix environments |

## Part 2 — Hands-On Lab

### Commands used in this demo

| Command | Flag meaning | Purpose |
|---|---|---|
| `tar -c/-x/-t` | create/extract/test | tar's three operating modes |
| `tar -z/-j/-J` | gzip/bzip2/xz | Compression selector, piped internally |
| `tar -f` | file | Archive filename follows — nearly always required |
| `tar -C DIR` | — | Change directory for this operation only |
| `tar --strip-components=N` | — | Drop N leading path segments on extraction |
| `tar --exclude=PATTERN` | — | Skip matching paths during archive creation |
| `tar --null --files-from=-` | — | Read a null-delimited file list from stdin |
| `gzip -t` | test | Verify a gzip stream's internal checksum |
| `gzip/bzip2/xz -9` | max compression | Highest ratio, most CPU time |
| `sha256sum` / `sha256sum -c` | — | Generate / verify a file's checksum |
| `zip -r -q` | recursive, quiet | Create a .zip, including subdirectories, no per-file output |
| `unzip -l` | list | Show .zip contents without extracting |

### Task 1 — Archive without compressing

```bash
mkdir -p ~/m04-02-lab/src && cd ~/m04-02-lab/src
mkdir -p app/config app/logs
echo "server_name=novatech" > app/config/app.conf
echo "port=8080" >> app/config/app.conf
echo "readme content" > app/README.md
# populate app/logs/access.log and app/logs/error.log with a few thousand
# lines of sample text — real content matters for Task 4's ratio comparison
cd ~/m04-02-lab
du -sh src/app
tar -cf app.tar -C src app
ls -lh app.tar
tar -tf app.tar
```

### Task 2 — Add compression, then prove the manual pipe is equivalent

```bash
tar -czf app.tar.gz -C src app
ls -lh app.tar app.tar.gz
tar -cf - -C src app | gzip > app-manual.tar.gz
sha256sum <(gzip -dc app.tar.gz) <(gzip -dc app-manual.tar.gz)
# Confirm the two checksums match — -czf and the manual pipe are identical
```

### Task 3 — Inspect before extracting

```bash
tar -tvf app.tar.gz
file app.tar.gz
mkdir -p restore1 restore2
tar -xzf app.tar.gz -C restore1
find restore1 -maxdepth 3
tar -xzf app.tar.gz -C restore2 --strip-components=1
find restore2 -maxdepth 3
# Confirm restore2 has no app/ prefix; restore1 does
```

### Task 4 — Compare compressors

```bash
tar -cf - -C src app > /tmp/plain.tar
for tool in gzip bzip2 xz; do
  time $tool -9 -c /tmp/plain.tar > /tmp/plain.tar.$tool
done
ls -lh /tmp/plain.tar /tmp/plain.tar.gzip /tmp/plain.tar.bzip2 /tmp/plain.tar.xz
# Compare sizes and elapsed time across all three
```

### Task 5 — Integrity verification, then a real corruption

```bash
gzip -t app.tar.gz
sha256sum app.tar.gz > app.tar.gz.sha256
sha256sum -c app.tar.gz.sha256

cp app.tar.gz app-corrupt.tar.gz
dd if=/dev/zero bs=1 count=1 seek=100 conv=notrunc of=app-corrupt.tar.gz
gzip -t app-corrupt.tar.gz
echo "exit code: $?"
# Confirm the exact "invalid compressed data--crc error" message appears
```

### Task 6 — Selective extraction, exclusion, null-safe building, and zip interop

```bash
mkdir -p restore3
tar -xzf app.tar.gz -C restore3 app/config/app.conf
find restore3

tar -czf app-nologs.tar.gz -C src --exclude='logs' app
tar -tzf app-nologs.tar.gz

mkdir -p /tmp/mixed && cd /tmp/mixed
touch "report one.txt" "report two.txt" normal.txt
find /tmp/mixed -name '*.txt' -print0 | tar --null -czf ~/m04-02-lab/mixed.tar.gz --files-from=-
tar -tzf ~/m04-02-lab/mixed.tar.gz
cd ~/m04-02-lab

zip -r -q app.zip src/app
unzip -l app.zip
ls -lh app.tar.gz app.zip
```

## What You Learned

1. ✅ Explained why archiving and compression are separate operations, and composed them explicitly before relying on `tar`'s combined flags
2. ✅ Created, inspected, and extracted `tar` archives safely — using `-C`, `--strip-components`, and confirming absolute-path stripping behavior
3. ✅ Compared `gzip`, `bzip2`, and `xz` on real ratio and timing data, and know when each is the right choice
4. ✅ Verified archive integrity with `gzip -t` and `sha256sum`, and saw exactly what a corrupted archive's failure looks like
5. ✅ Built an archive safely from filenames with spaces using `find -print0 | tar --null --files-from=-`, and used `--exclude` and selective extraction
6. ✅ Used `zip`/`unzip` for cross-platform interoperability and can state when it's preferable to `tar`

## Part 3 — Break-Fix Scenario

### Scenario — NovaTech Ops: two archiving incidents

> **Incident report — NovaTech Ops, ticket #4701**
>
> Alert 1: "the nightly backup script ran with no errors reported, but when we went to restore from it during a real outage, the expected backup file doesn't exist — instead there's a mysterious file just named `-C` sitting in the backup directory." Alert 2: "a teammate downloaded a vendor's `config-tools.tar.gz`, extracted it directly into `/tmp/downloads`, and now `/tmp/downloads` has `settings.conf`, `readme.txt`, and a `lib/` directory mixed in with three other unrelated files that were already there."

**Incident 1 — the mysterious file named `-C`**

```bash
$ tar -czf -C src app
tar: app: Cannot stat: No such file or directory
tar: Exiting with failure status due to previous errors
$ ls -la
-rw-r--r-- 1 root root 26420 ... -C
```

Tasks:
1. Given §1.1's flag table, what did `-f` actually consume as its argument here, and why?
2. Why did `tar` still produce a real (if wrongly-named) 26KB gzip file, rather than failing outright?
3. What's the corrected command, and what's the general rule about flag/argument order this incident teaches?

**Incident 2 — files scattered into `/tmp/downloads`**

```bash
$ tar -tzf config-tools.tar.gz | head -3
./
./readme.txt
./lib/
```

Tasks:
1. What's the signature in this `-tzf` output that would have warned you *before* extracting?
2. Contrast this with what §1.2 showed about `tar`'s absolute-path stripping — why doesn't that protection help here?
3. What's the safe extraction pattern that would have prevented this, regardless of whether the archive turned out to be well-formed or not?

<details>
<summary>Answers</summary>

**Incident 1:** `-f` always consumes the *very next token* as the archive filename, with no exception for tokens that look like other flags. In `tar -czf -C src app`, that next token is literally `-C` — so `tar` created a real gzip-compressed archive and named it `-C`. The remaining tokens (`src`, `app`) were then treated as ordinary files to add to the archive, evaluated relative to the current directory (since `-C` was consumed as a filename, it never took effect as the change-directory flag) — `src` exists there, but `app` doesn't, producing the `Cannot stat` error. The archive still got created (with whatever it successfully processed before the error) because `tar` doesn't abort mid-write just because a later file in the list is missing — hence a real, if wrongly-named and incomplete, file appears. Corrected command: `tar -czf app.tar.gz -C src app` — the archive filename must immediately follow `-f`, before any other flag. The general rule: never let an option letter be the token immediately after `-f`; when combining `-C` with `-f`, always double-check by reading the command left to right as "what does `-f`'s very next word resolve to."

**Incident 2:** The `-tzf` output shows no common leading directory — every path starts with `./` directly (`./readme.txt`, `./lib/`), not `app/readme.txt` or `config-tools/readme.txt`. This is the tarbomb signature: the archive was created by `cd`-ing into the source directory and archiving `.` rather than archiving the named parent directory itself. §1.2's absolute-path stripping only protects against paths starting with `/` (e.g. `/etc/passwd`) — it does nothing for a *relative* archive with no common prefix, since every one of those paths is technically valid and non-absolute; `tar` has no way to know the archive "should" have had a wrapping directory. The safe extraction pattern: always extract into a fresh, purpose-made subdirectory (`mkdir -p /tmp/extract-$$` or similar, then `tar -xzf archive.tar.gz -C that-dir`) rather than the current working directory, regardless of what `-tzf` shows beforehand — and run `-tzf` first as a habit, since a missing common prefix is the visible warning sign whenever it's present.

</details>

## Interview Prep

**Q1. A junior engineer writes `tar -czf backup.tar.gz /var/log`, and it works. They then write `tar -czf -C /var log`, expecting the same result with the directory-change flag, and it fails. Why?**

`-f` always claims the token immediately following it as the archive filename — no exception for tokens that look like other flags. In `-czf -C /var log`, `-f`'s argument becomes the literal string `-C`, so `tar` tries to create an archive named `-C` and then treats `/var` and `log` as files to add, evaluated from the current directory rather than `/var`. The fix is ordering: the archive filename must directly follow `-f`, so it should be written `tar -czf backup.tar.gz -C /var log`. This is one of the most common real-world `tar` mistakes precisely because bundled flags (`-czf`) make it easy to lose track of which token pairs with which flag.

**Q2. You're asked to choose a compression tool for a nightly database dump that needs to complete within a tight backup window, versus a compliance archive that's written once and almost never read again. Do you use the same tool for both?**

No — these have opposite priorities. The nightly dump is time-constrained, so `gzip` (fastest, worst ratio of the three) is the right default; CPU time saved matters more than the extra disk space a better ratio would save. The compliance archive is written once and rarely touched, so `xz` (slowest, best ratio) is the right choice — you pay the CPU cost once at write time and get the smallest possible long-term storage footprint. Using `xz` for the nightly dump would risk blowing the backup window; using `gzip` for the compliance archive would waste storage indefinitely for a one-time CPU savings that doesn't matter.

**Q3. A colleague says "I ran `gzip -t` on our backup and it passed, so the backup is definitely restorable." Do you agree?**

Not entirely. `gzip -t` verifies the gzip stream's own internal checksum — it catches corruption in the compressed data structure. It does not verify that the *tar contents inside* are the files you actually expected, uncorrupted at the application level, or that a full restore actually produces working data. A more complete verification also does a test extraction (`tar -tzf`, or better, an actual extraction to a scratch location) and checks the file listing and a few file contents against what's expected — "the compression layer is intact" and "the backup is restorable" are related but distinct claims.

## Key Takeaways

---

1. **Archiving and compression are two separate operations that `tar`'s combined flags (`-czf`) hide, but don't eliminate.** `tar -czf` is exactly equivalent to `tar -cf - | gzip` — understanding this makes every flag combination in this demo predictable instead of memorized.
2. **`-f` always consumes the very next token as the archive filename, with no exception for tokens that look like flags.** Writing `-czf -C src app` instead of `-czf archive.tar.gz -C src app` silently creates a real archive named `-C` and fails on the remaining arguments — this is the single most common real-world `tar` mistake.
3. **`tar` strips a leading `/` from member paths automatically (with a warning) to prevent an archive from writing outside the extraction directory — but this does not protect against a "tarbomb."** A relative archive built without a common wrapping directory (created via `tar -czf x.tar.gz .` from inside the source directory) scatters its contents into whatever directory you extract into. Always check `-tzf` for a common prefix, and always extract into a fresh subdirectory regardless.
4. **Choose the compressor by trade-off, not habit: `gzip` for speed, `xz` for ratio, `bzip2` sits between them.** On this demo's test data, `xz` produced a file roughly a third the size of `gzip`'s output at the cost of meaningfully more CPU time — the right choice depends on whether the bottleneck is a backup window or long-term storage cost.
5. **`gzip -t` verifies the compressed stream's internal checksum; `sha256sum` verifies the whole file hasn't changed since a known-good point.** They catch different failure modes — a single flipped byte anywhere in a compressed archive produces a full `invalid compressed data--crc error`, not a partial or localized failure, which is why integrity checks belong in any real backup verification step, not just "the job completed."
6. **The `find -print0` / null-delimited pattern from Demo M05-06 applies to building `tar` archives from dynamic file lists, not just `xargs`.** `find ... -print0 | tar --null -czf out.tar.gz --files-from=-` handles filenames with spaces correctly; a plain newline-delimited list does not.
7. **`zip` compresses per-file and is natively readable on Windows/macOS without extra tooling; `tar`+compressor preserves Unix permissions and ownership more faithfully.** Choose `zip` specifically when the recipient's environment is unknown or non-Unix; choose `tar` for anything staying inside Linux/Unix infrastructure.

> **Demo scope:** Primary concept: the archive/compression separation and its safe composition.
> Supporting concepts: tar mode and compression flags, integrity verification, safe extraction
> patterns, compressor trade-offs, zip interoperability.
> Estimated completion time: 38 minutes (reading + hands-on + verification).
> Checkpoints: 6 natural stopping points (after Task 1 through Task 6).

## Quick Commands Reference

### Archiving

| Command | Purpose |
|---|---|
| `tar -cf archive.tar -C dir path` | Create an uncompressed archive |
| `tar -tvf archive.tar` | List contents with details, without extracting |
| `tar -xf archive.tar -C dir` | Extract into a specific directory |
| `tar --strip-components=N` | Drop N leading path segments on extraction |
| `tar --exclude='pattern'` | Skip matching paths when creating an archive |

### Compression

| Command | Purpose |
|---|---|
| `tar -czf` / `-xzf` | Create/extract, gzip-compressed |
| `tar -cjf` / `-xjf` | Create/extract, bzip2-compressed |
| `tar -cJf` / `-xJf` | Create/extract, xz-compressed |
| `gzip -9 file` | Compress standalone, max ratio |
| `gzip -t file.gz` | Test integrity of a gzip stream |

### Safety and verification

| Command | Purpose |
|---|---|
| `sha256sum file > file.sha256` | Record a checksum |
| `sha256sum -c file.sha256` | Verify against a recorded checksum |
| `find DIR -print0 \| tar --null -czf out.tar.gz --files-from=-` | Null-safe archive building |

### Interop

| Command | Purpose |
|---|---|
| `zip -r -q archive.zip dir` | Create a zip, recursive, quiet |
| `unzip -l archive.zip` | List a zip's contents without extracting |

## Appendix A — Anki Cards

**`m04-02-archiving-compression-anki.csv`:**

```
#deck:Linux Mastery::Module 4 - Storage::Demo M04-02 - Archiving & Compression
#separator:Comma
#columns:Front,Back,Tags
"You run tar -cf archive.tar dir and the resulting file is smaller than the source directory's total size shown by du. Did tar compress the data?","No — tar alone never compresses anything. The size difference comes from tar not preserving the filesystem's per-file block-rounding overhead, not from any compression. Compression requires an explicit flag (-z/-j/-J) or piping through a separate compressor.","demo-m04-02,tar,compression,fundamentals"
"tar -czf app.tar.gz -C src app and tar -cf - -C src app | gzip > app.tar.gz — are these equivalent?","Yes, byte-for-byte once decompressed. -czf is tar's combined-flag shorthand for exactly this pipe: -c creates the archive stream, and -z pipes it through gzip internally. Understanding this composition is what makes tar's flag combinations predictable.","demo-m04-02,tar,gzip,fundamentals"
"A command is written as tar -czf -C src app (archive filename omitted). What actually happens?","-f always consumes the very next token as the archive filename, regardless of whether it looks like another flag. Here it consumes '-C' literally, creating a real gzip file named -C, then fails trying to stat 'app' as a file in the current directory since -C never took effect as the change-directory flag.","demo-m04-02,tar,break-fix,common-mistake"
"An archive's tar -tzf output shows every member starting with ./ directly, with no common wrapping directory. What does this signature indicate, and what's the risk?","This is a tarbomb signature — the archive was created by archiving the contents of a directory (via cd + tar . ) rather than the directory itself. Extracting it scatters every file directly into whatever directory you're standing in, rather than into one clean subdirectory.","demo-m04-02,tar,tarbomb,break-fix"
"tar strips a leading / from member paths with a warning. Does this protect against a tarbomb?","No — that protection only applies to absolute paths (e.g. /etc/passwd). A tarbomb's paths are all relative (./readme.txt, ./lib/) with no common prefix, which is technically valid and not caught by absolute-path stripping. Always extract into a fresh subdirectory regardless of what -tzf shows.","demo-m04-02,tar,tarbomb,security"
"When would you choose xz over gzip for compressing a backup?","When compression ratio matters more than CPU time — a compliance archive written once and rarely read benefits from xz's smaller footprint despite its higher CPU cost at creation. gzip is the better default when the backup window itself is the constraint.","demo-m04-02,compression,gzip,xz,tradeoffs"
"gzip -t on a backup file returns exit code 0 (success). Does this guarantee the backup is fully restorable?","Not entirely. gzip -t only verifies the compressed stream's internal checksum — it doesn't confirm the tar contents inside are the expected files or that a real extraction produces correct data. A fuller check also runs tar -tzf (or a real test extraction) against the decompressed contents.","demo-m04-02,integrity,verification,gzip"
"A single byte is flipped inside a .tar.gz file. What does gzip -t report, and why isn't the damage localized to one file inside the archive?","gzip: invalid compressed data--crc error — compressed streams don't degrade gracefully; corruption anywhere in the stream can invalidate everything after that point, not just the one logical file whose bytes were altered.","demo-m04-02,integrity,corruption,gzip"
"How do you safely build a tar archive from a dynamically generated list of filenames that might contain spaces?","find DIR -print0 | tar --null -czf out.tar.gz --files-from=- — the same null-delimited pattern from Demo M05-06's find -print0 | xargs -0, applied to tar's --files-from instead of xargs.","demo-m04-02,tar,find,cross-demo,safety"
"--exclude='logs' is passed to tar during archive CREATION, not extraction. What's the practical implication?","The excluded content never enters the archive at all — it's not a filter applied when reading the archive back out. This matters because you can't recover excluded content from that archive later; the decision to exclude is permanent for that archive.","demo-m04-02,tar,exclude,fundamentals"
"Why might you choose zip over tar+gzip for a specific deliverable?","When the recipient's environment is unknown or non-Unix (Windows, macOS) — .zip is natively readable there without extra tooling, unlike .tar.gz. tar+compressor remains preferable for Linux-to-Linux transfer where preserving Unix permissions/ownership matters more.","demo-m04-02,zip,tar,comparison"
"What's the difference between tar -tvf and unzip -l?","They're the direct equivalent of each other for their respective formats — both list an archive's contents with details (size, date, name) without extracting anything. tar -tvf works on tar archives (any compression); unzip -l works on .zip files specifically.","demo-m04-02,tar,zip,comparison"
```

## Appendix B — Quiz

**`m04-02-archiving-compression-quiz.md`:**

````markdown
# Quiz — Demo M04-02: Archiving & Compression

> One correct answer per question unless stated otherwise.
> Target: 80% or above before moving to the next demo.

---

**Q1.** `tar -cf archive.tar somedir` produces a file smaller than `somedir`'s total size shown by `du -sh`. What's the correct interpretation?

- A) `tar` compressed the data automatically, even without `-z`
- B) `tar` alone never compresses data — the smaller size comes from not preserving filesystem block-rounding overhead, not compression
- C) This indicates the archive is corrupted
- D) `du -sh` was measuring the wrong directory

<details>
<summary>Answer</summary>

**B** — `tar` without a compression flag (`-z`/`-j`/`-J`) never shrinks data; it only bundles files and metadata into one stream. Any size difference from the source comes from how filesystem block allocation is reported by `du`, not compression.

Trap: A wrongly assumes automatic compression; C and D invent unrelated explanations for a completely normal and expected size difference.

</details>

---

**Q2.** `tar -czf -C src app` is run, intending to create `app.tar.gz` from the `src/app` directory. What actually happens?

- A) It works correctly — `-C` is recognized as the directory-change flag regardless of position
- B) `-f` consumes `-C` as the literal archive filename, creating a file named `-C`; the command then fails trying to find `app` relative to the current directory
- C) `tar` prompts interactively for the missing archive filename
- D) The command fails immediately with a syntax error before creating anything

<details>
<summary>Answer</summary>

**B** — `-f` always claims the very next token as the archive filename, with no exception for tokens resembling other flags. This creates a real (if wrongly named) archive file called `-C`, and the subsequent failure comes from `app` not existing relative to the current directory, since `-C`'s directory-change behavior never actually took effect.

Trap: A wrongly assumes flag order doesn't matter; C and D both invent failure/prompt behaviors that don't match how `tar` actually handles this.

</details>

---

**Q3.** `tar -tzf vendor-package.tar.gz` shows every member path starting with `./` directly (`./readme.txt`, `./lib/`), with no shared top-level directory. What does this indicate?

- A) The archive is definitely corrupted and shouldn't be extracted
- B) This is a "tarbomb" signature — extracting it will scatter files directly into the current directory rather than into one clean subdirectory
- C) This is normal and safe — all archives extract this way by default
- D) The archive was created with the wrong compression flag

<details>
<summary>Answer</summary>

**B** — A missing common leading directory means the archive was created by archiving the *contents* of a directory rather than the directory itself, and extraction will place every member directly wherever you're standing rather than inside one wrapper folder — the reason to always `-tzf` first and extract into a fresh subdirectory regardless.

Trap: A invents corruption that isn't indicated by this output; C wrongly normalizes a real hazard; D invents an unrelated compression-flag explanation.

</details>

---

**Q4.** `tar` automatically strips a leading `/` from an absolute-path member name, with a warning. Does this protect against a tarbomb scattering files into the extraction directory?

- A) Yes — both are the same underlying protection mechanism
- B) No — absolute-path stripping only prevents writing outside the extraction directory via an absolute path; a tarbomb's paths are relative with no common prefix, which this mechanism doesn't address at all
- C) Yes, but only for archives created with GNU tar specifically
- D) No — `tar` has no protections against either scenario

<details>
<summary>Answer</summary>

**B** — These are two distinct hazards with two distinct (non-overlapping) causes. Absolute-path stripping addresses a member path trying to escape to `/etc/passwd`-style locations; it does nothing for a tarbomb, whose paths are all valid relative paths that simply lack a shared wrapping directory.

Trap: A and C both wrongly conflate two unrelated protections; D understates — absolute-path stripping is a real, functioning protection, just for a different problem.

</details>

---

**Q5.** On a given dataset, `xz -9` produces a noticeably smaller file than `gzip -9`, at the cost of more CPU time. For which scenario is `gzip` still the better choice?

- A) A compliance archive written once and almost never read again
- B) A nightly backup that must complete within a tight, time-constrained backup window
- C) Any scenario — `gzip` is always superior to `xz`
- D) Long-term cold storage where disk space is the primary cost driver

<details>
<summary>Answer</summary>

**B** — When the constraint is time (a backup window), `gzip`'s speed advantage matters more than the extra disk space `xz` would save. A and D both describe scenarios where ratio matters more than speed — exactly where `xz` is the better choice, not `gzip`.

Trap: C overgeneralizes a trade-off into an absolute superiority claim that isn't true in either direction.

</details>

---

**Q6.** `gzip -t backup.tar.gz` completes with exit code 0. A colleague concludes the backup is "definitely restorable." Is this conclusion fully justified?

- A) Yes — `gzip -t`'s success is a complete guarantee of restorability
- B) Not entirely — `gzip -t` only verifies the compressed stream's internal checksum, not that the tar contents inside are the expected files or that a real extraction produces correct data
- C) No — `gzip -t` provides no useful information at all about the file
- D) Yes, but only if the file was also checked with `sha256sum` beforehand

<details>
<summary>Answer</summary>

**B** — `gzip -t` catches corruption at the compression-stream level, which is genuinely useful, but it's a narrower claim than "the backup is restorable." A fuller check also verifies the tar contents (`tar -tzf`, or an actual test extraction).

Trap: A overstates what a single check proves; C understates it — `gzip -t` is a real, meaningful check, just not a complete one; D inverts the relationship — `sha256sum` verifies unchanged-since-recorded, a different and complementary check, not a prerequisite for `gzip -t` meaning anything.

</details>

---

**Q7.** `find /data -name '*.log' -print0 | tar --null -czf logs.tar.gz --files-from=-` is used instead of a plain newline-piped file list. Why?

- A) `--null`/`-print0` makes the archive compress better
- B) It safely handles filenames containing spaces or other whitespace, which a newline-delimited list would incorrectly split into multiple arguments
- C) `--files-from=-` is required for any archive larger than a few hundred files
- D) This pattern is only needed when compressing with `xz`, not `gzip`

<details>
<summary>Answer</summary>

**B** — This is the same null-safety pattern from Demo M05-06's `find -print0 | xargs -0`, applied to `tar`'s `--files-from` instead. Null bytes can't appear in filenames, making them the only delimiter safe for arbitrary names, including ones with spaces.

Trap: A invents an unrelated compression-ratio effect; C and D both invent scale/tool-specific requirements that don't apply — the safety concern is about filename content, not archive size or compressor choice.

</details>

---

**Q8.** Why might you choose `zip` over `tar` + `gzip` for a specific file you need to send to someone?

- A) `zip` always produces smaller files than `tar.gz`
- B) The recipient's environment is unknown or non-Unix (Windows/macOS), where `.zip` is natively readable without installing extra tools, unlike `.tar.gz`
- C) `zip` preserves Unix permissions and ownership more faithfully than `tar`
- D) `tar` cannot be used for files smaller than 1MB

<details>
<summary>Answer</summary>

**B** — Native cross-platform readability is `zip`'s real practical advantage — Windows Explorer and macOS Finder open `.zip` files with no extra software, unlike `.tar.gz`. Choose `tar`+compressor instead for Linux-to-Linux transfer, where its more faithful permission/ownership preservation matters more.

Trap: A is not reliably true — ratio depends on the compressor and content, not the container format; C reverses the actual comparison (`tar` is the stronger one for Unix metadata); D invents a nonexistent size restriction.

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