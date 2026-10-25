# Demo 14b — `rsync` for Sync & Backup

> **A note on how this demo was built, unlike the others in this series:**
> `rsync` is not installed in the environment used to build this demo, and
> installing it requires network access that wasn't available. Every
> behavioral claim below is sourced to the official `rsync(1)` manual page
> (cross-checked against current 3.x-era usage guides to confirm nothing
> has drifted since) rather than to live-captured output. Claims that
> genuinely can't be settled by documentation alone — exact `--bwlimit`
> timing, real delta-transfer byte savings on a specific re-sync, and the
> Rocky9 column of the Environments table — are marked `⚠️ [VERIFY]`.
> Before treating this demo as fully production-verified, run the tasks in
> Part 2 for real and confirm the output matches what's described.

## Overview

Demo 14a covered `tar`+compressor: bundle everything, ship the whole
archive, every time. That's fine for a one-off transfer or a cold backup,
but it's wasteful for the same reason re-sending a whole book to fix one
typo would be. `rsync` solves a different problem — keeping two directory
trees in sync — by transferring only the parts of files that actually
changed, using its own delta-transfer algorithm. This is what makes it the
standard tool for nightly backups, ongoing mirrors, and any deploy that
touches the same destination repeatedly rather than once.

This demo covers `rsync`'s core flags, the one safety habit that prevents
its most common real-world disasters (`--dry-run`, always run first), the
trailing-slash rule that trips up nearly everyone the first time, and when
to reach for `rsync` instead of Demo 14a's `tar`+`scp` pattern.

## Demo Objectives

1. Explain what makes `rsync`'s delta-transfer approach different from
   `tar`+`scp` (Demo 14a), and when that difference actually matters
2. Use `-a` for an archive-preserving sync and state precisely what its
   flag bundle does and does not preserve
3. Always run `--dry-run` before a real sync, and read its output correctly
4. State the trailing-slash source-path rule precisely and predict its
   effect on the destination layout
5. Build exclude patterns with `--exclude` / `--exclude-from`
6. Throttle transfers with `--bwlimit`, and choose between `rsync` and
   `tar`+`scp` for a given backup scenario

## Directory Structure

```
14b-rsync-backup/
├── README.md                              # this file
├── 14b-rsync-backup-anki.csv              # Anki flashcard deck
└── 14b-rsync-backup-quiz.md               # standalone quiz
```

## Environments

| Task | Ubuntu 24.04 | Rocky Linux 9.8 | EC2 Amazon Linux |
|---|---|---|---|
| All lab tasks | ⚠️ [VERIFY] — `rsync` is in Ubuntu 24.04's default `apt` repository (candidate version 3.2.7 confirmed via `apt-cache policy` in this session) but was not installed or executed here | ⚠️ [VERIFY] — not checked this session; `rsync` ships in Rocky9's default AppStream repos as a standard package | Not required |

## Recall Check — 14a-archiving-compression

Answer these from memory before continuing — no looking back at Demo 14a.

1. `tar -czf -C src app` is run, intending to create an archive from `src/app`. What actually goes wrong?

2. `tar -tzf` on an archive shows every member path starting with `./` directly, with no common wrapping directory. What does this indicate, and why is it dangerous?

3. When would you choose `xz` over `gzip` for a backup, and when would you choose the opposite?

<details>
<summary>Answers</summary>

1. `-f` always consumes the very next token as the archive filename — here it consumes `-C` literally, creating a real archive named `-C` and then failing when it tries to treat `app` as a file in the current directory, since `-C`'s directory-change behavior never took effect.

2. It's a tarbomb signature — the archive was built by archiving a directory's contents rather than the directory itself, so extraction scatters every file into whatever directory you're standing in rather than one clean subdirectory.

3. `xz` when compression ratio matters more than CPU time (a compliance archive written once, rarely read); `gzip` when the constraint is speed (a nightly backup with a tight completion window).

</details>

---

## Part 1 — Concepts

### 1.1 Delta-transfer — the difference from `tar`+`scp`

`tar`+`scp` (Demo 14a) always moves the whole archive, every time, regardless
of how much actually changed since the last transfer. `rsync` instead
compares the source and destination and transfers only the pieces of a
file that differ, using a checksum-based block comparison — its own
"remote-update protocol." For a directory that's mostly unchanged since
the last sync (a log directory with one new file, a codebase with a few
edited lines), this can mean transferring a small fraction of the total
data instead of the whole tree.

> **Source:** rsync(1) manual page — the tool's own description states it
> "uses the rsync remote-update protocol...to greatly speed up file
> transfers when the destination file already exists," transferring "just
> the differences between two sets of files" via a checksum-search
> algorithm. This is the mechanism, not a marketing claim — but the
> *actual* bytes-saved figure for any specific dataset is something you'd
> measure with `--stats` on a real re-sync, not something this demo can
> state as a number without running it. ⚠️ [VERIFY] — no live re-sync was
> performed in this session to produce a concrete before/after figure.

This is also why `rsync` needs itself installed on *both* ends of a
transfer when working over SSH (unlike `scp`, which only needs a plain SSH
server) — both sides run the comparison protocol.

### 1.2 Core flags

| Flag | Letter meaning | What it does |
|---|---|---|
| `-a` | archive | Shorthand for `-rlptgoD`: recursive, preserve symlinks, permissions, times, group, owner, device files |
| `-v` | verbose | Print each file as it's transferred |
| `-z` | compress | Compress file data during transfer (same method `gzip` uses) — useful over slow links |
| `-h` | human-readable | Sizes in K/M/G rather than raw bytes |
| `-n` | dry-run | Show what would happen, without doing it — see §1.3 |

**A precision worth having about `-a`:** per the manual page, `-a` does
**not** include `-H` (hard-link preservation) — hard-link detection is
expensive, so it's deliberately left out of the "preserve almost
everything" bundle and must be requested explicitly if you need it. This
is the same category of precision as Demo 14's note that `parted`'s
filesystem-type argument is a hint, not an action — knowing exactly what
a bundled flag does and doesn't cover prevents a false sense of "archive
mode preserves everything."

### 1.3 `--dry-run` — the one habit that prevents most real incidents

```bash
rsync -avn source/ destination/
```

`-n` (or `--dry-run`) makes `rsync` report exactly what it *would* transfer,
create, or delete, without touching the destination at all. This matters
most in combination with `--delete` (§1.7) — the manual page states this
plainly: `--delete` "can be dangerous if used incorrectly," and explicitly
recommends a dry run first to confirm the file list before it's too late
to undo. This isn't a hypothetical caution — real incidents from mistyped
source paths combined with `--delete` and no dry-run are a well-documented,
recurring category of data-loss report in `rsync`'s own community history.
Treat `--dry-run` as non-optional before any command that includes
`--delete`, not as an occasional extra check.

### 1.4 The trailing-slash rule — the classic gotcha

This is the single most common point of confusion for anyone new to
`rsync`, and it's worth internalizing precisely rather than by trial and
error:

> **A trailing slash on the *source* path changes the result. A trailing
> slash on the destination path never matters.**

```bash
rsync -a src dst      # copies src ITSELF into dst → dst/src/...
rsync -a src/ dst     # copies the CONTENTS of src into dst → dst/...
```

Think of it as: no trailing slash means "copy this directory" (by name);
a trailing slash means "copy the contents of this directory" (not the
directory itself). Get this backwards on a *second* run against the same
destination and you get a nested duplicate — `dst/src/src/...` — because
the first run already created `dst/src`, and a second slash-less run
copies `src` into it again, one level deeper. Part 3's Break-Fix scenario
walks through exactly this.

> **Source:** rsync(1) manual page, confirmed unchanged in current 3.x-era
> guides — this specific behavior has been stable across every rsync
> version referenced in the sources checked for this demo (spanning the
> 2.x manual through current 3.3.0-era documentation).

### 1.5 `--exclude` and `--exclude-from` — pattern matching during sync

```bash
rsync -av --exclude='*.tmp' --exclude='node_modules' src/ dst/
rsync -av --exclude-from='exclude-patterns.txt' src/ dst/
```

Per the manual page's pattern rules: a pattern with no `/` matches the
filename anywhere in the tree (`*.tmp` excludes any file ending `.tmp` at
any depth); a pattern starting with `/` anchors to the base of the
transfer; a pattern ending in `/` matches directories only. Multiple
`--exclude` flags accumulate — each adds to the exclude list rather than
replacing it — and `--exclude-from` reads one pattern per line from a
file, useful once the list gets long enough to be unwieldy on the command
line (the same idea as `tar`'s `--exclude` from Demo 14a, applied to
`rsync`'s own matching syntax rather than being identical syntax).

### 1.6 `--bwlimit` — throttling transfer speed

```bash
rsync -av --bwlimit=5000 src/ user@remote:dst/
# limits the transfer to roughly 5000 KB/s
```

Per the manual page, this option is most effective on larger files
(several megabytes and up) — `rsync` sends a block of data, then pauses
if the transfer is running faster than the specified limit, averaging out
to roughly the requested rate over time. ⚠️ [VERIFY — timing claim, docs
only]: the exact throttling behavior in practice (how tightly it holds to
the limit, how it behaves on many small files vs. few large ones) wasn't
independently measured in this session; the mechanism described here is
what the documentation states, not something observed running.

### 1.7 `rsync` over SSH vs. local sync, and vs. `tar`+`scp`

```bash
rsync -avz -e ssh src/ user@remote:/backup/dst/    # push to a remote host
rsync -avz -e ssh user@remote:/backup/src/ dst/    # pull from a remote host
rsync -av src/ dst/                                 # local sync, no network at all
```

`-e ssh` tells `rsync` to use SSH as its transport — the same command
works identically for local-to-local sync (no `:` in either path) and
remote sync (a `:` separator triggers the remote-shell transport). Both
ends need `rsync` installed for the remote case, since the comparison
protocol runs on both sides, not just one.

| | `rsync` | `tar` + `scp` (Demo 14a) |
|---|---|---|
| Repeated syncs to the same destination | Transfers only changed data — fast after the first run | Re-sends the whole archive every time |
| One-off transfer, destination doesn't exist yet | No meaningful advantage — first run transfers everything either way | Simple, no extra tool needed if `scp`/SSH access exists |
| Precise control over what's included/excluded per-run | `--exclude`/`--include` rules, evaluated per file at transfer time | Exclusions baked into the archive at creation time (Demo 14a §1.5) — can't be changed after the fact |
| Bandwidth-constrained links | `--bwlimit`, plus delta-transfer means less data moves in the first place | No native throttling; whole archive size is what's sent |
| Typical use in this series | Nightly backups, ongoing mirrors, anything synced more than once | One-off backups, log shipping to cold storage, vendor package transfer |

## Part 2 — Hands-On Lab

**Before starting:** confirm `rsync` is installed (`which rsync`); install
it if not (`sudo apt install rsync` / `sudo dnf install rsync`). This
lab was not run in the environment that authored it — run each task for
real and treat the `# Expected` comments as what the documentation
predicts, not as a transcript to compare against blindly.

### Commands used in this demo

| Command | Flag meaning | Purpose |
|---|---|---|
| `rsync -a` | archive | Recursive sync preserving permissions/times/owner/group/symlinks |
| `rsync -n` / `--dry-run` | dry-run | Report what would happen without doing it |
| `rsync -v` | verbose | List each file transferred |
| `rsync -z` | compress | Compress data in transit |
| `rsync --delete` | — | Remove destination files no longer present in source |
| `rsync --exclude` / `--exclude-from` | — | Skip matching files during sync |
| `rsync --bwlimit=N` | — | Throttle transfer to roughly N KB/s |
| `rsync -e ssh` | — | Use SSH as the transport for a remote sync |

### Task 1 — Local sync, trailing slash both ways

```bash
mkdir -p ~/14b-lab/src/subdir
echo "file one" > ~/14b-lab/src/one.txt
echo "file two" > ~/14b-lab/src/subdir/two.txt
mkdir -p ~/14b-lab/dst-noslash ~/14b-lab/dst-slash
cd ~/14b-lab

rsync -av src dst-noslash
find dst-noslash
# Expected (per §1.4): dst-noslash/src/one.txt, dst-noslash/src/subdir/two.txt
#   — the source directory ITSELF was copied in

rsync -av src/ dst-slash
find dst-slash
# Expected (per §1.4): dst-slash/one.txt, dst-slash/subdir/two.txt
#   — only the CONTENTS were copied in, no src/ level created
```

### Task 2 — Dry run before a real sync

```bash
echo "file three" > ~/14b-lab/src/three.txt
rsync -avn src/ dst-slash
# Expected (per §1.3): a listing of just "three.txt" as what WOULD be
# transferred, with no actual change made yet — confirm by re-listing
# dst-slash and seeing three.txt still absent

rsync -av src/ dst-slash
ls dst-slash
# Now three.txt should actually be present
```

### Task 3 — Exclude patterns

```bash
mkdir -p ~/14b-lab/src/node_modules
echo "should not sync" > ~/14b-lab/src/node_modules/pkg.js
echo "temp data" > ~/14b-lab/src/scratch.tmp

rsync -avn --exclude='*.tmp' --exclude='node_modules' src/ dst-slash
# Expected (per §1.5): the dry-run listing should show neither
# node_modules/ nor scratch.tmp as items to transfer

cat > ~/14b-lab/exclude-patterns.txt <<'EOF'
*.tmp
node_modules
EOF
rsync -avn --exclude-from=exclude-patterns.txt src/ dst-slash
# Expected: identical result to the inline --exclude flags above
```

### Task 4 — `--delete`, always preceded by `--dry-run`

```bash
rm ~/14b-lab/src/one.txt
rsync -avn --delete src/ dst-slash
# Expected (per §1.3/§1.7): dry-run output should list one.txt as a
# file that WOULD be deleted from dst-slash, since it's no longer in src

rsync -av --delete src/ dst-slash
ls dst-slash
# Confirm one.txt is now actually gone from dst-slash, matching src
```

### Task 5 — Bandwidth throttling

```bash
dd if=/dev/zero of=~/14b-lab/src/bigfile bs=1M count=20 2>/dev/null
time rsync -av --bwlimit=500 src/bigfile /tmp/throttled-copy
# Expected (per §1.6): the transfer takes measurably longer than an
# unthrottled copy of the same file — run without --bwlimit for
# comparison and note the difference yourself; the exact timing depends
# on your machine and isn't something this demo can predict precisely
```

### Task 6 — Remote sync over SSH (if you have SSH access to a second host or VM)

```bash
rsync -avz -e ssh src/ user@remote-host:/tmp/14b-remote-test/
# Expected (per §1.7): identical behavior to the local sync in Task 1,
# just transported over SSH; both ends need rsync installed

rsync -avzn -e ssh --delete src/ user@remote-host:/tmp/14b-remote-test/
# Dry-run first, exactly as in Task 4 — the habit doesn't change just
# because the destination is remote
```

## What You Learned

1. ✅ Explained what makes `rsync`'s delta-transfer approach different
   from `tar`+`scp`, and when that difference actually matters
2. ✅ Used `-a` for an archive-preserving sync and can state precisely
   what its flag bundle does and does not preserve (notably: not hard links)
3. ✅ Ran `--dry-run` before a real sync and read its output correctly
4. ✅ Stated the trailing-slash source-path rule precisely and predicted
   its effect on destination layout, both ways
5. ✅ Built exclude patterns with `--exclude` and `--exclude-from`
6. ✅ Throttled a transfer with `--bwlimit`, and can choose between
   `rsync` and `tar`+`scp` for a given backup scenario

## Part 3 — Break-Fix Scenario

### Scenario — NovaTech Ops: two rsync incidents

> **Incident report — NovaTech Ops, ticket #4718**
>
> Alert 1: "our nightly `rsync` backup has been running for weeks, but
> the backup directory just doubled in size overnight with no explanation
> — there's now a directory inside the backup with the same name as the
> backup itself, and it seems to contain a second copy of everything."
> Alert 2: "someone ran the backup script with a typo in the source path
> and `--delete` enabled — the destination now has files missing that
> definitely existed yesterday, and there was no `--dry-run` step in the
> script to catch it beforehand."

**Incident 1 — a nested duplicate directory appeared overnight**

```bash
# The nightly cron job, unchanged for weeks:
rsync -a /data/app-source /backup/nightly
```

Tasks:
1. Referencing §1.4, what's the most likely cause of a nested duplicate
   appearing, given this exact command was run repeatedly without
   modification?
2. Why would this problem *not* show up on the very first run, only on a
   later one?
3. What's the one-character fix, and why does it need to be applied to
   this specific command rather than to the destination path?

**Incident 2 — `--delete` ran without a preceding dry-run and removed real files**

```bash
# The script, as actually run (note the source path):
rsync -a --delete /data/ap-source/ /backup/nightly/
# (intended: /data/app-source/ — note the typo, "ap-source" not "app-source")
```

Tasks:
1. Given §1.3's guidance and the manual page's own explicit warning about
   `--delete`, what should have happened before this command ran for real?
2. Why does a typo'd, effectively-empty or wrong source path combine so
   badly with `--delete` specifically — what does `rsync` "think" it's
   supposed to do given that input?
3. What's the concrete fix for the script itself, not just this one
   incident?

<details>
<summary>Answers</summary>

**Incident 1:** This is the trailing-slash mistake from §1.4, and it's a
well-documented, recurring category of `rsync` surprise, not unique to
this scenario. `rsync -a /data/app-source /backup/nightly` (no trailing
slash on the source) copies the *directory itself* into the destination —
the first run creates `/backup/nightly/app-source/...`. It won't show up
as a problem on the first run, because the first run's result looks
correct: the backup directory now contains the expected data, just nested
one level under `app-source/`. The problem appears on the *second* run,
because the command is unchanged and still copies `app-source` (the
directory, by name) into `/backup/nightly` — but `/backup/nightly` already
has an `app-source/` inside it from the first run, so the second run's
`app-source` lands *inside* the first run's `app-source`, producing
`/backup/nightly/app-source/app-source/...` — a nested duplicate, exactly
matching the reported symptom. The fix is a trailing slash on the source:
`rsync -a /data/app-source/ /backup/nightly/app-source/` (copying
*contents into* a specifically named destination), or equivalently
`rsync -a /data/app-source/ /backup/nightly/` if `/backup/nightly` is
meant to directly mirror `app-source`'s contents rather than contain a
subdirectory named after it. The fix belongs on the source path
specifically because that's where the trailing-slash rule actually
applies — per §1.4, a trailing slash on the destination never changes
anything.

**Incident 2:** Per §1.3, any command combining `--delete` with a real
run should always be preceded by the exact same command with `--dry-run`
added, so the file list `rsync` intends to delete can be reviewed before
anything is actually removed — this is stated explicitly, not just as
general caution, in `rsync`'s own manual page for the `--delete` option.
The typo'd source path (`ap-source` instead of `app-source`) means
`rsync` is being told, truthfully as far as it can tell, that the
"correct" state of `/backup/nightly` should reflect whatever exists at
`/data/ap-source/` — which is either empty, nonexistent, or contains
unrelated/stale data. `rsync` has no way to know this path is a typo
rather than an intentional, if unusual, source; it just does what
`--delete` says: bring the destination in line with the source, deleting
anything in the destination that isn't in that (wrong) source. The fix
for the script itself is structural, not just correcting this one typo:
add a `--dry-run` invocation of the exact same command as a mandatory
preceding step (potentially gated on "did the file list look reasonable"
before the real run proceeds, not just run-and-discard), and consider
validating that the source path actually exists and is non-empty before
any `--delete` run executes at all — a script that silently proceeds
against a typo'd path is the actual root cause here, not the typo itself.

</details>

## Interview Prep

**Q1. A teammate asks why you'd reach for `rsync` instead of just `tar`-ing a directory and `scp`-ing it over, the way Demo 14a's pattern works. When do you actually recommend `rsync` instead?**

When the same destination gets synced more than once — a nightly backup,
an ongoing mirror, a deploy that updates the same remote directory
repeatedly. `rsync`'s delta-transfer means only the changed portions of
files move on each subsequent run, not the whole tree every time. For a
genuinely one-off transfer, where the destination doesn't exist yet and
never will be synced again, `tar`+`scp` is simpler and has no meaningful
disadvantage — the delta-transfer benefit only shows up on repeat runs.

**Q2. A script runs `rsync -a --delete` on a schedule with no dry-run step. What's your first concern reviewing this script, and what would you add?**

The unconditional combination of `--delete` with no preceding `--dry-run`
step is the concern — `rsync`'s own manual page explicitly warns that
`--delete` can be dangerous if used incorrectly, precisely because a
wrong or typo'd source path doesn't produce an error; it produces a
different, unintended deletion. I'd add a `--dry-run` invocation of the
same command as a required preceding step, and ideally a sanity check
that the source path exists and isn't unexpectedly empty before the real
`--delete` run is allowed to proceed at all.

**Q3. What does `-a` actually preserve, and what's the one thing people commonly assume it covers but doesn't?**

`-a` expands to `-rlptgoD` — recursive, symlinks, permissions, times,
group, owner, and device files. What it does *not* include is hard-link
preservation (`-H`) — the manual page is explicit that this is left out
deliberately because detecting hard links is comparatively expensive, not
an oversight. Anyone relying on `-a` alone to preserve a hard-linked
directory structure (a common pattern in some backup tools) needs to add
`-H` explicitly.

## Key Takeaways

---

1. **`rsync`'s core advantage over `tar`+`scp` (Demo 14a) is delta-transfer: only the changed portions of files move on a repeat sync, not the whole tree.** This makes it the right tool specifically for anything synced more than once — nightly backups, ongoing mirrors — and gives no meaningful advantage for a genuine one-off transfer.
2. **The trailing slash on the *source* path changes the result; the trailing slash on the destination never matters.** No slash copies the directory itself (by name) into the destination; a trailing slash copies only its contents. Getting this backwards on a repeated sync produces a nested duplicate directory on the second run — not the first, which is exactly why it's easy to miss until it's already happened more than once.
3. **`-a` (archive mode) is `-rlptgoD` — and deliberately does *not* include `-H` (hard links).** Knowing precisely what a bundled flag covers, rather than assuming "archive mode preserves everything," matters the same way it did for `tar -czf` in Demo 14a.
4. **Always run the exact same command with `--dry-run` before any real run that includes `--delete`.** This is stated as explicit guidance in `rsync`'s own manual page, not just general caution — a typo'd or wrong source path combined with `--delete` doesn't error out, it silently deletes real destination files that no longer appear to exist in the (wrong) source.
5. **`--exclude`/`--exclude-from` patterns are evaluated per-file at sync time, unlike `tar`'s `--exclude` (Demo 14a), which is baked in at archive creation.** This means exclusion rules can change between runs without rebuilding anything, which is part of what makes `rsync` better suited to ongoing, evolving sync jobs.
6. **`--bwlimit` throttles by pausing between data blocks to average out to roughly the requested rate — it's most effective on larger files, per the tool's own documentation.** The exact behavior on many small files versus few large ones wasn't independently measured for this demo and is worth confirming on your own workload before relying on it for a hard bandwidth ceiling.
7. **`rsync` needs itself installed on both ends of a remote sync, unlike `scp`, which only needs a plain SSH server.** This is a direct consequence of delta-transfer requiring a comparison protocol that runs on both sides — the one meaningful deployment prerequisite `tar`+`scp` doesn't share.

> **Demo scope:** Primary concept: rsync's delta-transfer sync model and
> its safe operation, in contrast to Demo 14a's archive-and-send pattern.
> Supporting concepts: -a flag bundle, --dry-run safety, trailing-slash
> semantics, --exclude patterns, --bwlimit, local vs. SSH transport.
> Estimated completion time: 35 minutes (reading + hands-on + verification),
> ASSUMING rsync is installed and working — add setup time otherwise.
> Checkpoints: 6 natural stopping points (after Task 1 through Task 6).
> **Verification status:** built from official documentation, not live
> execution — see the note at the top of this file.

## Quick Commands Reference

### Core sync

| Command | Purpose |
|---|---|
| `rsync -a src/ dst/` | Archive-mode sync, contents of src into dst |
| `rsync -a src dst/` | Archive-mode sync, src itself (as a subdirectory) into dst |
| `rsync -avn src/ dst/` | Dry run — always do this before a real `--delete` run |
| `rsync -avz src/ dst/` | Archive + verbose + compress in transit |

### Safety and filtering

| Command | Purpose |
|---|---|
| `rsync -av --delete src/ dst/` | Sync AND remove destination files no longer in source |
| `rsync --exclude='pattern'` | Skip matching files, evaluated per sync |
| `rsync --exclude-from=FILE` | Read exclude patterns from a file, one per line |

### Remote and throttled transfer

| Command | Purpose |
|---|---|
| `rsync -avz -e ssh src/ user@host:dst/` | Push to a remote host over SSH |
| `rsync -avz -e ssh user@host:src/ dst/` | Pull from a remote host over SSH |
| `rsync --bwlimit=N` | Throttle transfer to roughly N KB/s |

## Appendix A — Anki Cards

**`14b-rsync-backup-anki.csv`:**

```
#deck:Linux Mastery::Module 4 - Storage::Demo 14b - rsync for Sync & Backup
#separator:Comma
#columns:Front,Back,Tags
"What is rsync's core advantage over tar+scp for a directory that gets synced to the same destination repeatedly?","rsync uses delta-transfer — it compares source and destination and transfers only the changed portions of files, not the whole tree every time. tar+scp re-sends the entire archive on every run regardless of how much actually changed.","demo14b,rsync,fundamentals"
"rsync -a src dst and rsync -a src/ dst produce different results. What's the rule, and which one copies src's CONTENTS rather than src itself?","A trailing slash on the SOURCE path changes the result; a trailing slash on the destination never matters. rsync -a src/ dst copies the contents of src into dst. rsync -a src dst copies src itself (as a subdirectory) into dst.","demo14b,rsync,trailing-slash,fundamentals"
"The same rsync -a src dst command (no trailing slash) is run twice against the same destination. What happens on the second run that didn't happen on the first?","A nested duplicate directory appears. The first run creates dst/src/. The second run, unchanged, again copies the src directory itself into dst — but dst already contains an src/ from the first run, so the new copy lands inside it: dst/src/src/.","demo14b,rsync,trailing-slash,break-fix"
"Does rsync's -a (archive mode) preserve hard links?","No — -a expands to -rlptgoD, which deliberately does NOT include -H (hard-link preservation). Hard-link detection is comparatively expensive, so it's left out of the bundle on purpose and must be requested explicitly with -H if needed.","demo14b,rsync,archive-mode,common-mistake"
"What should always precede a real rsync run that includes --delete?","The exact same command with --dry-run (-n) added, so the file list rsync intends to delete can be reviewed first. rsync's own manual page explicitly recommends this — --delete is documented as dangerous if used incorrectly, since a wrong source path doesn't error, it silently deletes real destination files.","demo14b,rsync,delete,safety"
"A script runs rsync -a --delete against a typo'd source path (e.g. /data/ap-source instead of /data/app-source). What does rsync actually do, and why doesn't it fail with an error?","rsync has no way to know the path is a typo — it treats /data/ap-source as the intended source and brings the destination in line with it, deleting any destination files that don't exist in that (wrong, likely empty or unrelated) source. This is exactly why a dry-run step matters: it surfaces the unexpected file list before real deletion happens.","demo14b,rsync,delete,break-fix"
"How does rsync's --exclude differ from tar's --exclude (Demo 14a) in terms of WHEN it's applied?","rsync's --exclude is evaluated per-file at sync time, on every run — the rules can change between runs without rebuilding anything. tar's --exclude is baked into the archive permanently at creation time and can't be changed afterward for that archive.","demo14b,rsync,tar,exclude,comparison"
"Why does rsync need to be installed on BOTH ends of a remote (SSH) transfer, unlike scp which only needs a plain SSH server on the remote end?","rsync's delta-transfer relies on a comparison protocol that runs on both sides — both the sending and receiving rsync processes participate in figuring out which blocks actually changed. scp has no such comparison; it just copies bytes, so it only needs SSH access, not rsync itself, on the remote end.","demo14b,rsync,ssh,fundamentals"
"When would tar+scp (Demo 14a) still be the better choice over rsync?","For a genuine one-off transfer to a destination that doesn't exist yet and won't be synced again — delta-transfer's advantage only shows up on repeat syncs to the same destination. For a single transfer, tar+scp is simpler with no meaningful disadvantage.","demo14b,rsync,tar,comparison"
"What does --bwlimit=500 do, per rsync's own documentation, and what's its documented mechanism?","Throttles the transfer to roughly 500 KB/s. Per the manual page, rsync sends a block of data, then pauses if the transfer is running faster than the specified limit — the result averages out to approximately the requested rate over time, most effective on larger files.","demo14b,rsync,bwlimit,fundamentals"
```

## Appendix B — Quiz

**`14b-rsync-backup-quiz.md`:**

````markdown
# Quiz — Demo 14b: rsync for Sync & Backup

> One correct answer per question unless stated otherwise.
> Target: 80% or above before moving to the next demo.

---

**Q1.** What's the fundamental mechanism that makes `rsync` different from `tar`+`scp` for a repeatedly-synced destination?

- A) `rsync` compresses data more efficiently than `gzip`
- B) `rsync` transfers only the changed portions of files (delta-transfer) rather than re-sending the whole tree every time
- C) `rsync` is simply a faster network protocol than SCP with no other functional difference
- D) `rsync` automatically deletes old backups to save space

<details>
<summary>Answer</summary>

**B** — Delta-transfer, using a checksum-based comparison of source and destination, is the core mechanism. This is what makes `rsync` advantageous specifically for repeat syncs to the same destination, not a general speed or compression improvement.

Trap: A and C both invent generic performance claims not specific to the actual mechanism; D invents automatic behavior `rsync` doesn't perform on its own (deletion only happens with `--delete` explicitly specified).

</details>

---

**Q2.** `rsync -a src dst` and `rsync -a src/ dst` are run against a fresh, empty `dst`. Do they produce the same result?

- A) Yes, always — trailing slashes never affect `rsync`'s behavior
- B) No — `src dst` (no trailing slash) copies the `src` directory itself into `dst`, producing `dst/src/...`; `src/ dst` (trailing slash) copies only the contents, producing `dst/...` directly
- C) No — the trailing slash only matters when syncing over SSH, not for local syncs
- D) Yes, but only because `dst` is empty; they'd differ on a non-empty destination

<details>
<summary>Answer</summary>

**B** — This is the trailing-slash rule: the slash on the source path determines whether the directory itself or just its contents gets copied. This applies identically whether `dst` is empty or not, and identically for local or remote (SSH) transfers.

Trap: A wrongly claims no effect at all; C invents a remote-only restriction that doesn't exist; D correctly identifies a difference exists but for the wrong reason — the difference isn't contingent on `dst`'s emptiness.

</details>

---

**Q3.** The same `rsync -a app-source /backup/nightly` command (no trailing slash) runs nightly via cron, unmodified for weeks. What eventually happens, and why not on the first run?

- A) Nothing changes — this command is idempotent and safe to repeat indefinitely
- B) A nested duplicate directory appears starting on the SECOND run — the first run creates `/backup/nightly/app-source/`, and the second run copies `app-source` into that already-existing subdirectory again, producing a doubly-nested path
- C) `rsync` throws an error on the second run because the destination already contains a directory with that name
- D) The backup silently stops updating after the first run

<details>
<summary>Answer</summary>

**B** — This is exactly the trailing-slash mistake from §1.4. The first run's result looks correct (data present under `app-source/`), which is why the problem isn't caught immediately — it's only the second and subsequent runs that produce visible, growing nesting.

Trap: A wrongly assumes idempotency that this specific command doesn't have; C invents an error `rsync` doesn't throw — it happily nests directories rather than refusing; D invents a silent-failure mode that isn't what actually happens.

</details>

---

**Q4.** Does `rsync -a` preserve hard links between files in the source directory?

- A) Yes — `-a` preserves "almost everything," hard links included
- B) No — `-a` expands to `-rlptgoD`, which deliberately excludes `-H` (hard-link preservation); `-H` must be added explicitly
- C) Only when combined with `-z` (compression)
- D) Only on the destination side, never on the source side

<details>
<summary>Answer</summary>

**B** — This is a deliberate exclusion, not an oversight — hard-link detection is comparatively expensive, so it's left out of the archive-mode bundle by design, per the manual page.

Trap: A overstates what "archive mode" actually covers; C and D both invent unrelated conditions that have no bearing on hard-link handling.

</details>

---

**Q5.** What should always precede a real `rsync` run that includes `--delete`, per the tool's own documentation?

- A) A `--bwlimit` flag, to avoid overwhelming the network during deletion
- B) The exact same command with `--dry-run` added, to review the file list `rsync` intends to delete before it actually happens
- C) A `-z` flag, since deletions transfer more data than they appear to
- D) Nothing extra is needed if `-v` (verbose) is already included

<details>
<summary>Answer</summary>

**B** — `rsync`'s own manual page explicitly recommends a dry run before any `--delete` operation, precisely because a wrong or mistyped source path doesn't produce an error — it silently and validly instructs `rsync` to delete destination files that don't exist in that (wrong) source.

Trap: A and C both invent unrelated flag requirements that don't address the actual risk; D wrongly assumes verbose output substitutes for a dry run — `-v` shows what happened during a REAL run, which is already too late for anything it reports as deleted.

</details>

---

**Q6.** A script runs `rsync -a --delete` against a typo'd source path that happens to be empty or nonexistent. What does `rsync` do, and why?

- A) It refuses to run and exits with an error, since the source doesn't contain the expected files
- B) It proceeds normally, treating the (wrong) path as the intended source, and deletes any destination files that don't exist in that source — since `rsync` has no way to know the path is a mistake rather than an intentional, if empty, source
- C) It automatically falls back to the previously used source path
- D) It skips the `--delete` step specifically, but still performs the sync

<details>
<summary>Answer</summary>

**B** — `rsync` executes exactly what it's told; it can't distinguish a typo from a deliberate (if unusual) choice of source. This is precisely why the `--dry-run` habit from §1.3 matters — it's the only thing that surfaces this kind of mistake before real damage occurs.

Trap: A, C, and D all invent protective behaviors `rsync` doesn't actually have — it's a literal tool that does what its arguments say, not one that infers intent.

</details>

---

**Q7.** How does the timing of `rsync --exclude` pattern evaluation differ from `tar --exclude` (Demo 14a)?

- A) They're identical — both are evaluated once, at creation/first-sync time, and can't change afterward
- B) `rsync --exclude` is evaluated per-file on every sync run, meaning exclusion rules can change between runs without rebuilding anything; `tar --exclude` is baked into the archive permanently at creation
- C) `tar --exclude` is evaluated per-run like `rsync`'s; `rsync --exclude` is baked in permanently at the first sync
- D) Neither tool supports changing exclude patterns after the first use

<details>
<summary>Answer</summary>

**B** — This is a genuine, meaningful difference tied to what each tool actually is: `tar` creates a static archive once; `rsync` re-evaluates its file selection logic (including excludes) fresh on every invocation, which is part of what makes it well suited to an ongoing sync job whose exclusion needs might change over time.

Trap: A and D both wrongly claim neither tool's excludes are ever reconsidered; C reverses which tool does which.

</details>

---

**Q8.** Why does `rsync` need to be installed on both the sending and receiving machine for a remote (SSH) transfer, unlike `scp`?

- A) `rsync` requires a graphical interface on the remote machine
- B) `rsync`'s delta-transfer comparison protocol runs on both sides — both the sending and receiving processes participate in determining which blocks actually changed; `scp` just copies bytes with no such comparison, so it only needs SSH access remotely
- C) `rsync` and `scp` have identical installation requirements; this is a common misconception
- D) `rsync` only needs to be installed remotely, the same as `scp`

<details>
<summary>Answer</summary>

**B** — This follows directly from §1.1's delta-transfer mechanism: comparing source and destination to find the differing blocks requires both ends to participate in that comparison, which `scp`'s simple byte-copy approach never needs to do.

Trap: A invents an irrelevant GUI requirement; C and D both wrongly claim no difference exists, when the both-ends requirement is a direct, real consequence of how `rsync` actually works.

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