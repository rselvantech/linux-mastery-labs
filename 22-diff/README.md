# Demo 22 — diff in Detail

## Overview

Every config-drift investigation, every "what changed between these two
versions" question, and every patch you've ever applied ultimately relies
on `diff`. This demo covers reading `diff` output correctly (unified vs.
context format), comparing entire directory trees at once, resolving a
three-way conflict with `diff3`, and applying a diff as a patch — the full
loop from "what's different" to "make it match."

## Demo Objectives

1. Read unified diff format (`+`/`-` lines, `@@` hunk headers) fluently
2. Read context diff format and explain when it's still used
3. Compare entire directory trees with `diff -r`
4. Resolve a three-way merge conflict with `diff3`
5. Apply a diff as a patch with `patch`, including reversing one
6. Explain how `git diff` relates to standard `diff` output

## Directory Structure

```
22-diff/
└── README.md   # This file — full demo content, including Anki (Appendix A)
                 # and Quiz (Appendix B), extracted later via script.
```

## Environments

| Task | Ubuntu 24.04 | Rocky Linux 9.8 | EC2 Amazon Linux |
|---|---|---|---|
| All lab tasks | ✅ | ✅ | Not required |

## Recall Check — 21-awk

Answer these from memory before continuing — no looking back at Demo 21.

**Q1.** Why does `awk '{print $2}'` handle `ps aux` output correctly while
`cut -d' ' -f2` often doesn't?

**Q2.** An `awk` summation (`sum += $2`) on malformed numeric input reports
`Total: 0` instead of an error. Why?

**Q3.** What can an `awk` associative array do that `sort | uniq -c`
cannot?

<details>
<summary>Answers</summary>

**A1.** `awk` collapses repeated whitespace into a single field separator
automatically; `cut` treats every space as a separate boundary, producing
spurious empty fields on irregularly-spaced output.

**A2.** `awk` performs implicit numeric coercion — an unparseable field is
silently treated as `0` in arithmetic context rather than raising an
error.

**A3.** Group by one field while counting or summing a different field —
`uniq` has no concept of fields at all, only whole-line identity.

</details>

## Part 1 — Concepts

### 1.1 Unified diff format — the modern default

```bash
printf 'line1\nline2\nline3\n' > /tmp/file-a.txt
printf 'line1\nCHANGED\nline3\nline4\n' > /tmp/file-b.txt
diff -u /tmp/file-a.txt /tmp/file-b.txt
```

```
--- /tmp/file-a.txt    2026-08-01 10:00:00
+++ /tmp/file-b.txt    2026-08-01 10:00:05
@@ -1,3 +1,4 @@
 line1
-line2
+CHANGED
 line3
+line4
```

Reading this line by line: `---`/`+++` identify the two files being
compared (`-` side is the first file, `+` side is the second). `@@ -1,3
+1,4 @@` is the **hunk header** — `-1,3` means "starting at line 1 of the
first file, 3 lines shown"; `+1,4` means "starting at line 1 of the second
file, 4 lines shown." Within the hunk: a leading space means unchanged
(context), `-` means removed (only in the first file), `+` means added
(only in the second file).

**`-u` (unified) is the standard choice for nearly everything today** —
it's what `git diff` produces by default, what code review tools expect,
and what `patch` most commonly consumes. It's called "unified" because it
merges the old-content and new-content views into one interleaved block,
rather than showing them as two separate, harder-to-correlate sections.

### 1.2 Context diff — the older format, and why it still exists

```bash
diff -c /tmp/file-a.txt /tmp/file-b.txt
```

```
*** /tmp/file-a.txt    2026-08-01 10:00:00
--- /tmp/file-b.txt    2026-08-01 10:00:05
***************
*** 1,3 ****
  line1
! line2
  line3
--- 1,4 ----
  line1
! CHANGED
  line3
+ line4
```

Context diff (`-c`) shows the old and new content as **two separate
blocks** rather than interleaved — more verbose, but for some reviewers
genuinely easier to read when changes are dense, since the "before" and
"after" states are each fully readable as standalone blocks rather than
requiring you to mentally reconstruct one from `+`/`-` markers interleaved
with the other. It predates unified diff and is much less common in modern
tooling, but recognizing it matters — some older systems, legacy patch
files, and certain `diff` output defaults on less common platforms still
use it, and misreading `!` (changed) as if it were `+`/`-` would
misinterpret the change entirely.

### 1.3 Comparing entire directory trees

```bash
mkdir -p /tmp/dir-a /tmp/dir-b
echo "content1" > /tmp/dir-a/file1.txt
echo "content2" > /tmp/dir-a/file2.txt
echo "content1" > /tmp/dir-b/file1.txt
echo "content2-modified" > /tmp/dir-b/file2.txt
echo "content3" > /tmp/dir-b/file3.txt

diff -r /tmp/dir-a /tmp/dir-b
```

```
diff -r /tmp/dir-a/file2.txt /tmp/dir-b/file2.txt
1c1
< content2
---
> content2-modified
Only in /tmp/dir-b: file3.txt
```

`-r` recurses into subdirectories, comparing every file present in either
tree. `Only in DIR: file` flags files that exist in one tree but not the
other — a genuinely different signal from a content change, and one that's
easy to skim past if you're only looking for `<`/`>` change markers.

```bash
diff -rq /tmp/dir-a /tmp/dir-b
# -q: report only WHICH files differ, not the actual diff content — a
# faster first pass across a large tree before drilling into specific
# files with a full diff.
```

### 1.4 diff3 — three-way comparison and conflict resolution

`diff3` compares three versions of a file at once — the classic use case is
a merge: two people's independent changes to a common ancestor:

```bash
echo "original line" > /tmp/ancestor.txt
cp /tmp/ancestor.txt /tmp/mine.txt
cp /tmp/ancestor.txt /tmp/theirs.txt
sed -i 's/original/my-version/' /tmp/mine.txt
sed -i 's/original/their-version/' /tmp/theirs.txt

diff3 /tmp/mine.txt /tmp/ancestor.txt /tmp/theirs.txt
```

```
====
1:1c
my-version line
2:1c
original line
3:1c
their-version line
```

`====` marks a genuine conflict — both "mine" and "theirs" changed the same
line differently from the common ancestor, and `diff3` can't resolve that
automatically; a human has to choose. If only one side had changed the
line (the other matching the ancestor unchanged), `diff3` would resolve it
without flagging a conflict at all — conflicts are specifically for
divergent, incompatible changes to the same content.

```bash
diff3 -m /tmp/mine.txt /tmp/ancestor.txt /tmp/theirs.txt
# -m: merge — produce a merged file with conflict markers inline
# (<<<<<<<, =======, >>>>>>>) at points of actual conflict, and
# automatically-resolved content everywhere else — this is the mechanism
# underneath most version control merge conflict markers you've likely
# already seen in git.
```

### 1.5 patch — applying a diff

```bash
diff -u /tmp/file-a.txt /tmp/file-b.txt > /tmp/changes.patch
cp /tmp/file-a.txt /tmp/file-a-copy.txt
patch /tmp/file-a-copy.txt < /tmp/changes.patch
diff /tmp/file-a-copy.txt /tmp/file-b.txt
# No output = identical — the patch was applied correctly, file-a-copy.txt
# now matches file-b.txt exactly.
```

**Reversing a patch uses the same diff, with `-R`** — no need to generate a
separate reverse patch:

```bash
patch -R /tmp/file-a-copy.txt < /tmp/changes.patch
diff /tmp/file-a-copy.txt /tmp/file-a.txt
# No output = back to the original.
```

```bash
patch --dry-run /tmp/file-a-copy.txt < /tmp/changes.patch
# --dry-run: show whether the patch WOULD apply cleanly, without actually
# modifying anything — the correct way to check for conflicts before
# committing to an apply, especially on a file that may have drifted
# since the patch was generated.
```

### 1.6 git diff — the same underlying format, different context

```bash
cd /tmp && git init -q gitlab-demo && cd gitlab-demo
echo "line1" > tracked.txt
git add tracked.txt && git commit -q -m "initial"
echo "line1 modified" > tracked.txt
git diff
```

```
diff --git a/tracked.txt b/tracked.txt
index abc1234..def5678 100644
--- a/tracked.txt
+++ b/tracked.txt
@@ -1 +1 @@
-line1
+line1 modified
```

**`git diff`'s output is unified diff format with extra git-specific header
metadata** (the `diff --git`/`index` lines, blob hashes) — everything below
the `---`/`+++`/`@@` lines reads exactly the way §1.1 already covered,
because it's the same format. This is why a `.patch` file exported from
`git diff` can generally be applied with plain `patch`, and why
understanding standard `diff -u` output transfers directly to reading any
git diff, PR, or code review without needing git-specific interpretation
rules for the hunk content itself.

## Part 2 — Hands-On Lab

### Commands used in this demo

| Command | Flag meaning | Purpose |
|---|---|---|
| `diff -u file1 file2` | `-u`: unified | Standard modern diff format |
| `diff -c file1 file2` | `-c`: context | Older two-block format |
| `diff -r dir1 dir2` | `-r`: recursive | Compare entire directory trees |
| `diff -rq dir1 dir2` | `-q`: quiet | List only which files differ |
| `diff3 mine ancestor theirs` | — | Three-way comparison |
| `diff3 -m mine ancestor theirs` | `-m`: merge | Produce merged output with conflict markers |
| `patch file < diff.patch` | — | Apply a diff |
| `patch -R file < diff.patch` | `-R`: reverse | Undo an applied patch |
| `patch --dry-run` | — | Check if a patch would apply cleanly, without applying |

### Task 1 — Unified diff, read the hunk header

```bash
printf 'alpha\nbeta\ngamma\n' > /tmp/t1a.txt
printf 'alpha\nBETA-CHANGED\ngamma\ndelta\n' > /tmp/t1b.txt
diff -u /tmp/t1a.txt /tmp/t1b.txt
# Identify: which line changed, which line was added, and confirm the
# @@ hunk header's numbers match what you'd expect from the file content.
```

### Task 2 — Context diff, same change

```bash
diff -c /tmp/t1a.txt /tmp/t1b.txt
# Compare against Task 1's unified output — same underlying change,
# different presentation. Confirm you can identify the same edit in
# context format's two-block structure.
```

### Task 3 — Directory comparison

```bash
mkdir -p /tmp/t3a /tmp/t3b
echo "same" > /tmp/t3a/unchanged.txt
echo "same" > /tmp/t3b/unchanged.txt
echo "old" > /tmp/t3a/modified.txt
echo "new" > /tmp/t3b/modified.txt
echo "extra" > /tmp/t3b/onlyinb.txt

diff -r /tmp/t3a /tmp/t3b
diff -rq /tmp/t3a /tmp/t3b
```

### Task 4 — diff3 conflict resolution

```bash
echo "shared original" > /tmp/t4-ancestor.txt
cp /tmp/t4-ancestor.txt /tmp/t4-mine.txt
cp /tmp/t4-ancestor.txt /tmp/t4-theirs.txt
sed -i 's/original/mine-edit/' /tmp/t4-mine.txt
sed -i 's/original/theirs-edit/' /tmp/t4-theirs.txt

diff3 /tmp/t4-mine.txt /tmp/t4-ancestor.txt /tmp/t4-theirs.txt
diff3 -m /tmp/t4-mine.txt /tmp/t4-ancestor.txt /tmp/t4-theirs.txt
```

### Task 5 — Generate, apply, and reverse a patch

```bash
diff -u /tmp/t1a.txt /tmp/t1b.txt > /tmp/t5.patch
cp /tmp/t1a.txt /tmp/t5-target.txt

patch --dry-run /tmp/t5-target.txt < /tmp/t5.patch
patch /tmp/t5-target.txt < /tmp/t5.patch
diff /tmp/t5-target.txt /tmp/t1b.txt   # confirm no output = match

patch -R /tmp/t5-target.txt < /tmp/t5.patch
diff /tmp/t5-target.txt /tmp/t1a.txt   # confirm no output = back to original
```

### Task 6 — git diff, same format underneath

```bash
mkdir -p /tmp/t6-repo && cd /tmp/t6-repo
git init -q
echo "version 1" > file.txt
git add file.txt && git commit -q -m "initial"
echo "version 2" > file.txt
git diff > /tmp/t6.patch
cat /tmp/t6.patch
# Identify the same ---/+++/@@ structure from Task 1, plus the git-specific
# header lines above it.
```

## What You Learned

1. ✅ Read unified diff format (`+`/`-` lines, `@@` hunk headers) fluently
2. ✅ Read context diff format and explained when it's still used
3. ✅ Compared entire directory trees with `diff -r`
4. ✅ Resolved a three-way merge conflict with `diff3`
5. ✅ Applied a diff as a patch, including reversing one and dry-running first
6. ✅ Explained how `git diff` relates to standard `diff` output

## Part 3 — Break-Fix Scenario

### Scenario — NovaTech Ops: two diff/patch incidents

> **Incident report — NovaTech Ops, ticket #4699**
>
> Alert 1: "a config patch that applied cleanly last week now fails to
> apply on the same target file." Alert 2: "a `diff -r` between two config
> directories shows nothing, but a teammate insists the directories aren't
> identical."

**Incident 1 — patch that worked last week now fails**

```bash
$ patch /etc/novatech/app.conf < enable-feature.patch
patch: **** malformed patch at line 8: ...
$ # (or, alternatively:)
$ patch: Hunk #1 FAILED at 12.
```

Tasks:
1. Reference §1.5 — what's the correct first diagnostic step before
   assuming the patch file itself is corrupted?
2. What's a plausible reason the exact same patch file would apply cleanly
   one week and fail the next, given nothing about the patch file itself
   changed?
3. What command shows you the situation without risking a partially-applied
   file?

**Incident 2 — diff -r shows nothing, teammate insists directories differ**

```bash
$ diff -r /etc/nginx/sites-available /backup/nginx/sites-available
$ # no output at all
$ ls /etc/nginx/sites-available
default  novatech-app
$ ls /backup/nginx/sites-available
default  novatech-app  novatech-app.old
```

Tasks:
1. `diff -r` produced no output, but `ls` shows the backup directory has an
   extra file. Is `diff -r` malfunctioning?
2. Reference §1.3 — what output would you actually expect to see for a
   file present in one directory but not the other, and did it appear here?
3. Re-examine the command — is there a reason `diff -r` might not have
   reported the extra file even though it should?

<details>
<summary>Answers</summary>

**Incident 1:** The correct first step is `patch --dry-run` against the
*current* state of the target file, not assuming the patch itself is
corrupted — a dry run shows immediately whether it would apply cleanly
without risking a partial, broken application. The most plausible
explanation for the same patch failing a week later: the target file
`/etc/novatech/app.conf` has drifted since the patch was generated —
someone else made an unrelated change to the same file in the interim, and
the patch's context lines (the unchanged lines surrounding the actual
change, which `patch` uses to locate where to apply the change) no longer
match the file's current content at the expected location, causing the
hunk to fail to apply. This isn't a corrupted patch file; it's a stale
patch being applied against a file that's no longer in the state the patch
assumes. `patch --dry-run` is exactly the right command — it reports
success/failure without touching the file, letting you investigate context
mismatches before deciding whether to force, regenerate, or manually
reconcile the patch.

**Incident 2:** `diff -r` is not malfunctioning, but the specific command
shown genuinely should have reported `Only in /backup/nginx/sites-available:
novatech-app.old` given §1.3's documented behavior — the fact that it
didn't suggests something about the actual command run differs from what's
shown, or the file's presence changed between the `ls` and the `diff -r`
run (a real timing/ordering issue worth checking first), or, most likely
given how this class of incident actually happens, an unstated `--exclude`
pattern or symlink situation caused `diff -r` to skip that specific file
without it being obvious from the command as transcribed. The practical
lesson from working through this rather than the specific resolved cause:
`Only in DIR: file` is the expected signal for a file existing in one tree
but not the other, and when directory-tree diff output doesn't match what
`ls` independently shows, checking the *exact* command actually run
(flags, trailing slashes, exclusions) is the right next diagnostic step —
`diff -r`'s core behavior for this case is well-defined and reliable, so an
apparent contradiction is more likely a command-detail mismatch than
`diff -r` genuinely failing to detect an extra file.

</details>

## Interview Prep

**Q1. Why is unified diff (`-u`) the standard format used by `git diff` and most modern tooling, over context diff?**
Unified diff interleaves the old and new content into a single block with
`+`/`-` markers, which is more compact and easier to correlate at a glance
than context diff's two separate before/after blocks — for the kind of
line-by-line, often-small changes typical in code review and version
control, the interleaved format is generally faster to read. Context diff
still exists and is occasionally encountered with legacy tooling or dense
changes where some reviewers prefer fully-readable before/after blocks,
but it's not the default anywhere significant today.

**Q2. A patch that applied cleanly last week fails to apply today on the same target file, with no changes to the patch file itself. What's the most likely cause?**
The target file has drifted — someone else made an unrelated change to the
same file in the interim, and the patch's context lines (used by `patch`
to locate the correct application point) no longer match the file's
current content there. This isn't necessarily a corrupted patch; `patch
--dry-run` against the current file state is the correct first diagnostic
step, since it reveals exactly this kind of mismatch without risking a
partial application.

**Q3. What does `diff3`'s conflict marker (`====`) actually indicate, and when does `diff3` resolve a difference without flagging a conflict?**
`====` indicates a genuine three-way conflict — both compared versions
changed the same content differently from the common ancestor, which
`diff3` cannot resolve automatically. If only one side changed a given
piece of content (with the other matching the ancestor unchanged), `diff3`
resolves it silently without a conflict marker, since there's no
ambiguity about which version's change should win.

**Q4. Why does understanding plain `diff -u` output transfer directly to reading `git diff` or a GitHub/GitLab pull request diff?**
Because `git diff`'s output is unified diff format with additional
git-specific header metadata (`diff --git`, blob hash `index` lines) layered
on top — everything below the `---`/`+++`/`@@` lines follows the exact same
rules as standard `diff -u` output. There's no separate "git diff syntax"
to learn for the actual change content; it's the same format `patch`
already understands.

## Key Takeaways

1. **Unified diff (`-u`) is the standard format everywhere that matters
   today** — `git diff`, code review tools, and most generated `.patch`
   files all use it, interleaving old/new content with `+`/`-` markers
   rather than context diff's separate before/after blocks.
2. **`diff -r` flags files present in only one tree with `Only in DIR:
   file` — a distinct signal from a content change (`<`/`>`), easy to skim
   past if only scanning for change markers.**
3. **`diff3`'s `====` conflict marker means both compared versions changed
   the same content differently from a common ancestor.** A change on only
   one side resolves automatically, without a conflict flag.
4. **A patch failing to apply after previously working cleanly usually
   means the target file has drifted, not that the patch is corrupted.**
   `patch --dry-run` diagnoses this without risking a partial application.
5. **`patch -R` reverses a patch using the exact same diff file** — no
   need to generate a separate reverse patch.
6. **`git diff`'s output is standard unified diff with git-specific header
   metadata layered on top.** Understanding plain `diff -u` transfers
   directly to reading any git diff or PR without needing separate
   git-specific interpretation rules for the actual change content.

> **Demo scope:** Primary concept: comparing files and directories, and
> applying the differences. Supporting concepts: unified/context diff
> formats, diff3 three-way comparison, patch application/reversal,
> directory diffs, git diff's relationship to standard diff.
> Estimated completion time: 27 minutes (reading + hands-on + verification).
> Checkpoints: 6 natural stopping points (after Task 1 through Task 6).

## Quick Commands Reference

### Comparing files

| Command | Purpose |
|---|---|
| `diff -u file1 file2` | Unified diff — the modern standard |
| `diff -c file1 file2` | Context diff — older, two-block format |

### Comparing directories

| Command | Purpose |
|---|---|
| `diff -r dir1 dir2` | Recursive tree comparison, full diff content |
| `diff -rq dir1 dir2` | List only which files differ (fast first pass) |

### Three-way comparison

| Command | Purpose |
|---|---|
| `diff3 mine ancestor theirs` | Show conflicts (`====`) between two divergent versions |
| `diff3 -m mine ancestor theirs` | Produce merged output with inline conflict markers |

### Applying patches

| Command | Purpose |
|---|---|
| `diff -u a b > x.patch` | Generate a patch file |
| `patch --dry-run file < x.patch` | Check if it would apply cleanly, without applying |
| `patch file < x.patch` | Apply |
| `patch -R file < x.patch` | Reverse an applied patch |

## Appendix A — Anki Cards

**Note:** extracted into a standalone `.csv` by the series extraction script.

```
#deck:Linux Mastery::Module 4 - Text Processing & Scripting for DevOps::22-diff
#separator:Comma
#columns:Front,Back,Tags
"In a unified diff hunk header @@ -1,3 +1,4 @@, what do the two number pairs mean?","-1,3 means the first file's shown section starts at line 1 and spans 3 lines; +1,4 means the second file's shown section starts at line 1 and spans 4 lines.","demo22,diff,unified,hunk-header"
"In unified diff output, what do a leading space, a leading -, and a leading + on a line each mean?","Leading space: unchanged context line, present in both files. Leading -: line removed, only in the first file. Leading +: line added, only in the second file.","demo22,diff,unified,fundamentals"
"diff -r shows a content change for file2.txt as < old-line / > new-line, but the actual difference is that file3.txt exists in only one of the two directories. What output signals that?","'Only in DIR: file3.txt' — a distinct signal from the </> content-change markers, easy to miss if only scanning for content diffs.","demo22,diff,directory,fundamentals"
"What does diff3's ==== conflict marker actually indicate?","A genuine three-way conflict — both compared versions changed the same content differently from the common ancestor, which diff3 cannot resolve automatically and requires a human decision.","demo22,diff3,conflict"
"If only one of two versions changes a piece of content in diff3 (the other matching the ancestor unchanged), does diff3 flag a conflict?","No — diff3 resolves it silently without a conflict marker, since there's no ambiguity about which version's change should win when only one side actually changed anything.","demo22,diff3,conflict"
"A patch applied cleanly last week but fails to apply today on the same target file, with the patch file itself unchanged. What's the most likely cause?","The target file has drifted — someone made an unrelated change to the same file in the interim, and the patch's context lines no longer match the file's current content at the expected location.","demo22,patch,troubleshooting"
"What does patch --dry-run do, and why run it before applying a patch that might have gone stale?","It reports whether the patch would apply cleanly WITHOUT actually modifying the file — the correct way to check for context mismatches or conflicts before committing to an apply, avoiding a partially-applied broken file.","demo22,patch,dry-run"
"How do you reverse an already-applied patch, and does this require a separately-generated reverse patch file?","patch -R file < original.patch — the -R flag reverses the SAME diff file used to apply it originally; no separate reverse patch needs to be generated.","demo22,patch,reverse"
"Why does understanding plain diff -u output transfer directly to reading git diff or a GitHub pull request?","git diff's output is unified diff format with additional git-specific header metadata (diff --git, index/blob hash lines) layered on top — everything below the ---/+++/@@ lines follows the exact same rules as standard diff -u.","demo22,git-diff,cross-tool"
"When would context diff (-c) still realistically be encountered today, given unified diff (-u) is the modern standard?","Legacy tooling, older patch files, or certain platforms/reviewers that default to or prefer context diff's separate before/after block presentation over unified diff's interleaved format — it's much less common but not extinct.","demo22,diff,context-format"
```

## Appendix B — Quiz

**Note:** extracted into a standalone `.md` by the series extraction script.

> One correct answer per question unless stated otherwise.
> Target: 80% or above before moving to Demo 23.

**Q1. In unified diff, a line prefixed with `-` means:**

- A) The line was added, present only in the second file
- B) The line was removed, present only in the first file
- C) The line is unchanged context
- D) The line has a syntax error

<details>
<summary>Answer</summary>

**B** — `-` marks a removed line (first file only); `+` marks an added line (second file only); a leading space marks unchanged context. Trap: A reverses the actual meaning; C and D both misassign the marker's purpose.

</details>

---

**Q2. `diff -r dir1 dir2` shows content changes for shared files but a file exists in `dir2` only. What output flags this?**

- A) Nothing — `diff -r` cannot detect files unique to one directory
- B) `Only in dir2: filename`
- C) The file is silently skipped with no output
- D) An error is raised, halting the comparison

<details>
<summary>Answer</summary>

**B** — This is `diff -r`'s documented signal for files present in only one tree, distinct from content-change markers. Trap: A and C both wrongly claim this case goes undetected; D overstates the consequence — comparison continues normally.

</details>

---

**Q3. `diff3`'s `====` marker appears in output. What does it indicate?**

- A) A syntax error in one of the input files
- B) A genuine conflict — both compared versions changed the same content differently from the common ancestor
- C) The files are completely identical
- D) One of the three files is missing

<details>
<summary>Answer</summary>

**B** — This is the specific, correct meaning of a `diff3` conflict marker. Trap: A, C, and D all invent unrelated conditions that `====` doesn't actually signal.

</details>

---

**Q4. A patch that applied cleanly last week fails today on the same target file. The patch file itself hasn't changed. What's the most likely explanation?**

- A) `patch` has a time-based expiration on all patch files
- B) The target file has drifted — an unrelated change since then means the patch's context lines no longer match
- C) The patch file has silently corrupted itself over time
- D) File permissions always expire after 7 days

<details>
<summary>Answer</summary>

**B** — This is the realistic, common cause: `patch` locates its application point using context lines, and target-file drift breaks that match. Trap: A, C, and D all invent time-based mechanisms that don't exist.

</details>

---

**Q5. What does `patch --dry-run` do?**

- A) Applies the patch, then immediately reverses it
- B) Reports whether the patch would apply cleanly, without modifying the target file at all
- C) Applies the patch faster than a normal run
- D) Requires root privileges, unlike a normal patch apply

<details>
<summary>Answer</summary>

**B** — This is exactly what `--dry-run` is for: risk-free verification before committing to an actual apply. Trap: A misdescribes it as apply-then-revert; C and D invent unrelated behaviors.

</details>

---

**Q6. How do you reverse an already-applied patch?**

- A) `patch file < original.patch` a second time
- B) `patch -R file < original.patch`
- C) Generate a brand new reverse-direction diff and apply that instead
- D) Patches cannot be reversed once applied

<details>
<summary>Answer</summary>

**B** — `-R` reverses using the same original diff file; no separate reverse patch needs generating. Trap: A would likely fail or double-apply; C describes unnecessary extra work; D is factually wrong.

</details>

---

**Q7. Why does understanding standard `diff -u` output transfer directly to reading `git diff` or a pull request diff?**

- A) `git diff` uses a completely different, git-specific format requiring separate study
- B) `git diff`'s output is unified diff format with additional git-specific header metadata layered on top; the actual change content follows the same rules
- C) `git diff` only works after converting to unified format manually
- D) Pull requests use a proprietary format unrelated to `diff`

<details>
<summary>Answer</summary>

**B** — The `---`/`+++`/`@@` hunk structure and `+`/`-`/space line markers are identical to standard `diff -u`; git just adds header metadata above it. Trap: A, C, and D all wrongly imply git diff requires separate interpretation rules for the actual content.

</details>

---

**Q8. When might context diff (`-c`) still be relevant today, given unified diff is the modern standard?**

- A) Never — context diff is fully obsolete and unsupported everywhere
- B) Legacy tooling, older patch files, or reviewers who prefer its separate before/after block presentation
- C) Context diff is required for any file larger than 1MB
- D) `git diff` defaults to context format on Rocky9 specifically

<details>
<summary>Answer</summary>

**B** — Context diff is uncommon but not extinct, and understanding it avoids misreading `!` (changed) as if it were a `+`/`-` marker. Trap: A overstates its obsolescence; C and D both invent false technical requirements.

</details>