# Demo M05-10 — diff Comparison

## Overview

Every config-drift investigation, every "what changed between these two
versions" question, and every patch you've ever applied ultimately relies
on `diff`. This demo covers reading `diff` output correctly (unified vs.
context format), comparing entire directory trees at once, resolving a
three-way conflict with `diff3`, and applying a diff as a patch — the full
loop from "what's different" to "make it match."

## Demo Objectives

1. Read unified diff format (`+`/`-` lines, `@@` hunk headers) fluently,
   and use `diff`'s exit status (`0`/`1`/`2`) and `-q` for scripting
2. Read context diff format, explain when it's still used, and use `-y`
   (side-by-side) and `-i`/`-w` to reduce noise from case or whitespace
3. Compare entire directory trees with `diff -r`, including a fast
   files-only pass with `-q`
4. Resolve a three-way merge conflict with `diff3`, including reading its
   full inline conflict-marker set
5. Apply a diff as a patch with `patch`, including reversing one,
   stripping leading path components with `-p`, and explaining how
   `patch` locates where a hunk applies
6. Explain how `git diff` relates to standard `diff` output, including why
   its `a/`/`b/`-prefixed paths matter for applying it with plain `patch`

## Directory Structure

```
10-diff-comparison/
├── README.md                        # this file
├── m05-10-diff-comparison-anki.csv  # Anki flashcard deck
└── m05-10-diff-comparison-quiz.md   # standalone quiz
```

## Environments

| Task | Ubuntu 24.04 | Rocky Linux 9.8 | EC2 Amazon Linux |
|---|---|---|---|
| All lab tasks | ✅ | ✅ | Not required |

## Recall Check — 08-awk-processing

Answer these from memory before continuing — no looking back at Demo M05-08.

1. Why does `awk '{print $2}'` handle `ps aux` output correctly while
   `cut -d' ' -f2` often doesn't?

2. An `awk` summation (`sum += $2`) on malformed numeric input reports
   `Total: 0` instead of an error. Why?

3. What can an `awk` associative array do that `sort | uniq -c`
   cannot?

<details>
<summary>Answers</summary>

1. `awk` collapses repeated whitespace into a single field separator
   automatically; `cut` treats every space as a separate boundary, producing
   spurious empty fields on irregularly-spaced output.

2. `awk` performs implicit numeric coercion — an unparseable field is
   silently treated as `0` in arithmetic context rather than raising an
   error.

3. Group by one field while counting or summing a different field —
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

**`diff`'s exit status is meaningful and worth building scripts around,
the same way `grep`'s and `xargs`'s are (Demos M05-05/19):**

```bash
diff /tmp/file-a.txt /tmp/file-a.txt; echo "identical: $?"
diff /tmp/file-a.txt /tmp/file-b.txt > /dev/null; echo "different: $?"
diff /tmp/file-a.txt /tmp/does-not-exist.txt 2>/dev/null; echo "error: $?"
```
```
identical: 0
different: 1
error: 2
```
> Live-verified. `0` means the files are identical, `1` means they
> differ, `2` means `diff` itself couldn't complete the comparison (a
> missing file, a permissions problem) — three distinct outcomes, not
> just "worked" vs. "didn't."

**`-q` isn't only for directory trees (§1.3) — it works on two plain
files too, and pairs directly with the exit status above for a clean
scripting idiom:**

```bash
if diff -q /tmp/file-a.txt /tmp/file-a.txt > /dev/null; then
    echo "identical"
else
    echo "differ"
fi
```
> Live-verified. `diff -q` on two identical files prints nothing at all
> (exit `0`); on two different files it prints one line, `Files X and Y
> differ` (exit `1`) — either way, `diff -q ... > /dev/null` followed by
> checking `$?` is the standard "are these two files the same?" test,
> the same shape as `grep -q` from Demo M05-05.

### 1.2 Context diff, side-by-side, and reducing noise

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

**A third format, `-y`, lays both files out as two literal side-by-side
columns instead of either interleaving or block-separating them:**

```bash
diff -y --width=50 /tmp/file-a.txt /tmp/file-b.txt
```
```
line1                                            line1
line2                                        |   CHANGED
line3                                            line3
                                              >   line4
```
> Live-verified. A `|` between the columns marks a changed line; `<`/`>`
> mark a line present only on the left/right side, respectively. This is
> genuinely a different reading experience from unified or context diff
> — useful specifically when eyeballing two versions of a short file
> where seeing both complete versions side by side, without interleaving
> at all, is the clearest presentation. `--width` controls the total
> column width; without it, `-y` defaults to your terminal's width, which
> can wrap awkwardly on a narrow terminal.

**`-i` and `-w` filter out specific kinds of "noise" differences before
comparison even happens:**

```bash
printf 'Hello World\n' > /tmp/case1.txt
printf 'hello world\n' > /tmp/case2.txt
diff /tmp/case1.txt /tmp/case2.txt        # reports a difference
diff -i /tmp/case1.txt /tmp/case2.txt     # -i: ignore case — no output, identical
```
```bash
printf 'hello   world\n' > /tmp/ws1.txt
printf 'hello world\n' > /tmp/ws2.txt
diff /tmp/ws1.txt /tmp/ws2.txt            # reports a difference
diff -w /tmp/ws1.txt /tmp/ws2.txt         # -w: ignore whitespace — no output, identical
```
> Live-verified both. `-i` and `-w` are the right tools when a "real"
> difference is being drowned out by a formatting inconsistency you
> don't actually care about — comparing a config file that's been
> re-indented but not otherwise changed, for instance. Using them
> habitually on every comparison isn't the goal — they can hide genuine
> whitespace-sensitive bugs — but reaching for them deliberately when
> case or whitespace really is irrelevant to the comparison at hand
> avoids a lot of noisy, unhelpful diff output.

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
```

```
Files /tmp/dir-a/file2.txt and /tmp/dir-b/file2.txt differ
Only in /tmp/dir-b: file3.txt
```

`-q` (quiet) reports only *which* files differ or exist on only one side —
`Files X and Y differ`, one line per file — never the actual line-by-line
content diff. This is a faster first pass across a large tree before
drilling into specific files with a full `diff -r` or plain `diff` on just
the ones that turned out to matter.

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
```

```
<<<<<<< /tmp/mine.txt
my-version line
||||||| /tmp/ancestor.txt
original line
=======
their-version line
>>>>>>> /tmp/theirs.txt
```

`-m` (merge) produces a merged file with conflict markers inline at points
of actual conflict, and automatically-resolved content everywhere else.
Note the **four** markers, not three: `<<<<<<<` opens "mine"'s version,
`|||||||` shows the common ancestor's original content (easy to overlook,
but genuinely useful — it tells you exactly what both sides started from,
not just what they each changed it to), `=======` separates "mine" from
"theirs," and `>>>>>>>` closes "theirs"'s version. This is the mechanism
underneath most version control merge conflicts you've likely already
seen in git — though git's *default* conflict display often omits the
`|||||||` ancestor section unless `merge.conflictstyle` is set to
`diff3`, even though raw `diff3 -m` always includes it.

### 1.5 patch — applying a diff, and stripping path prefixes

```bash
diff -u /tmp/file-a.txt /tmp/file-b.txt > /tmp/changes.patch
cp /tmp/file-a.txt /tmp/file-a-copy.txt
patch /tmp/file-a-copy.txt < /tmp/changes.patch
diff /tmp/file-a-copy.txt /tmp/file-b.txt
# No output = identical — the patch was applied correctly, file-a-copy.txt
# now matches file-b.txt exactly.
```

**`patch` locates *where* to apply a hunk using its context lines, not just
line numbers.** Each hunk carries a few unchanged lines immediately
surrounding the actual change (the leading-space lines in unified format,
§1.1); `patch` searches the target file for text matching that context and
applies the change there. This is exactly why a patch that applied cleanly
once can fail later on a file that's since drifted — if an unrelated edit
altered those same surrounding lines, `patch` can no longer find a
confident match for where the hunk belongs, even though the target file's
*content* it actually wants to change is untouched.

**`patch` isn't limited to unified-format input** — it also correctly
applies context-format (`-c`) diffs, auto-detecting the format from the
patch file itself. You don't need to convert a context diff to unified
before applying it.

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

**`-p` (`--strip`) controls how many leading path components `patch`
strips from the paths recorded inside the diff before looking for the
target file — this is the specific, commonly-confusing gap between a
patch's internal paths and where you're actually applying it, and it's
directly relevant to §1.6's `git diff` output, whose paths are always
prefixed `a/`/`b/`:**

```bash
patch -p0 < git-generated.patch
```
```
can't find file to patch at input line 5
Perhaps you used the wrong -p or --strip option?
The text leading up to this was:
--------------------------
|diff --git a/tracked.txt b/tracked.txt
|index a29bdeb..181a4bd 100644
|--- a/tracked.txt
|+++ b/tracked.txt
--------------------------
```
> Live-verified — this exact failure, against a real `git diff`-generated
> patch applied in a directory containing only `tracked.txt` (no literal
> `a/` subdirectory). `-p0` (the explicit "don't strip anything" setting)
> looks for the path exactly as written in the patch — `a/tracked.txt` —
> which doesn't exist relative to where the patch is being applied.

```bash
patch -p1 < git-generated.patch
```
```
patching file tracked.txt
```
> Live-verified — `-p1` strips exactly one leading path component (the
> `a/` or `b/` git prefix), leaving `tracked.txt`, which does exist.
> `-p1` is the standard, almost-always-correct choice for a patch
> generated by `git diff`, specifically because of that `a/`/`b/`
> prefixing convention. Many `patch` builds also have some automatic
> guessing behavior that succeeds without an explicit `-p` in simple
> cases — but relying on that guess rather than specifying `-p1`
> explicitly for a known git-style patch is exactly the kind of "worked
> by luck" habit worth avoiding, the same reasoning as always testing a
> `sed -i` without `-i` first (Demo M05-07) rather than trusting a command to
> behave as hoped.

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
because it's the same format. The `index abc1234..def5678 100644` line
specifically shows the blob hash of the file's content before and after
the change (`abc1234..def5678`) and its file mode (`100644` — a regular,
non-executable file) — metadata `diff -u` alone has no equivalent for,
since a plain filesystem diff has no concept of git's internal object
hashes. This is why a `.patch` file exported from `git diff` can generally
be applied with plain `patch`, and why understanding standard `diff -u`
output transfers directly to reading any git diff, PR, or code review
without needing git-specific interpretation rules for the hunk content
itself.

**The `a/` and `b/` prefixes on every path in this output are exactly
what §1.5's `-p1` addresses.** Git always writes paths this way
regardless of the actual repository structure — `a/` for the "before"
side, `b/` for the "after" side — so applying a `git diff`-exported patch
with plain `patch` (outside of `git apply`, which handles this
automatically) means remembering `-p1` to strip that prefix and land on
the real path.

## Part 2 — Hands-On Lab

### Commands used in this demo

| Command | Flag meaning | Purpose |
|---|---|---|
| `diff -u file1 file2` | `-u`: unified | Standard modern diff format |
| `diff -c file1 file2` | `-c`: context | Older two-block format |
| `diff -y file1 file2` | `-y`: side-by-side | Two literal columns, `\|`/`<`/`>` markers |
| `diff -i` / `-w` | ignore case / whitespace | Filter out formatting-only differences |
| `diff` exit status | `0`/`1`/`2` | Identical / differ / error — scriptable |
| `diff -q file1 file2` | `-q`: quiet | One-line differ/no-output identical result |
| `diff -r dir1 dir2` | `-r`: recursive | Compare entire directory trees |
| `diff -rq dir1 dir2` | `-r`+`-q` | List only which files differ |
| `diff3 mine ancestor theirs` | — | Three-way comparison |
| `diff3 -m mine ancestor theirs` | `-m`: merge | Produce merged output with all four conflict markers |
| `patch file < diff.patch` | — | Apply a diff (unified or context format) |
| `patch -pN file < diff.patch` | `-p`/`--strip` | Strip N leading path components before locating the target |
| `patch -R file < diff.patch` | `-R`: reverse | Undo an applied patch |
| `patch --dry-run` | — | Check if a patch would apply cleanly, without applying |

### Task 1 — Unified diff, exit codes, and -q on plain files

```bash
printf 'alpha\nbeta\ngamma\n' > /tmp/t1a.txt
printf 'alpha\nBETA-CHANGED\ngamma\ndelta\n' > /tmp/t1b.txt
diff -u /tmp/t1a.txt /tmp/t1b.txt
# Identify: which line changed, which line was added, and confirm the
# @@ hunk header's numbers match what you'd expect from the file content.

diff /tmp/t1a.txt /tmp/t1a.txt; echo "identical exit: $?"
diff /tmp/t1a.txt /tmp/t1b.txt > /dev/null; echo "different exit: $?"
diff -q /tmp/t1a.txt /tmp/t1b.txt
```

### Task 2 — Context diff, side-by-side, and noise-reduction flags

```bash
diff -c /tmp/t1a.txt /tmp/t1b.txt
diff -y --width=60 /tmp/t1a.txt /tmp/t1b.txt

printf 'Hello World\n' > /tmp/t2-case1.txt
printf 'hello world\n' > /tmp/t2-case2.txt
diff /tmp/t2-case1.txt /tmp/t2-case2.txt
diff -i /tmp/t2-case1.txt /tmp/t2-case2.txt
# Confirm -i finds them identical despite the case difference
```

### Task 3 — Directory comparison: full diff vs. files-only

```bash
mkdir -p /tmp/t3a /tmp/t3b
echo "same" > /tmp/t3a/unchanged.txt
echo "same" > /tmp/t3b/unchanged.txt
echo "old" > /tmp/t3a/modified.txt
echo "new" > /tmp/t3b/modified.txt
echo "extra" > /tmp/t3b/onlyinb.txt

diff -r /tmp/t3a /tmp/t3b
diff -rq /tmp/t3a /tmp/t3b
# Compare the two: -rq shows only WHICH files differ or are one-sided,
# never the actual content diff — confirm both onlyinb.txt and
# modified.txt are correctly flagged by each command, just with
# different levels of detail.
```

### Task 4 — diff3 conflict resolution, reading all four markers

```bash
echo "shared original" > /tmp/t4-ancestor.txt
cp /tmp/t4-ancestor.txt /tmp/t4-mine.txt
cp /tmp/t4-ancestor.txt /tmp/t4-theirs.txt
sed -i 's/original/mine-edit/' /tmp/t4-mine.txt
sed -i 's/original/theirs-edit/' /tmp/t4-theirs.txt

diff3 /tmp/t4-mine.txt /tmp/t4-ancestor.txt /tmp/t4-theirs.txt
diff3 -m /tmp/t4-mine.txt /tmp/t4-ancestor.txt /tmp/t4-theirs.txt
# Identify all four markers in the -m output: <<<<<<<, |||||||, =======,
# >>>>>>> — confirm the ||||||| section shows the ancestor's original
# content, distinct from both "mine" and "theirs".
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

# Confirm patch also accepts context-format diffs directly, no conversion:
diff -c /tmp/t1a.txt /tmp/t1b.txt > /tmp/t5-context.patch
cp /tmp/t1a.txt /tmp/t5-target2.txt
patch /tmp/t5-target2.txt < /tmp/t5-context.patch
diff /tmp/t5-target2.txt /tmp/t1b.txt   # confirm no output = match, same as the unified version
```

### Task 6 — git diff, path-strip with -p, same format underneath

```bash
mkdir -p /tmp/t6-repo && cd /tmp/t6-repo
git init -q
echo "version 1" > file.txt
git add file.txt && git commit -q -m "initial"
echo "version 2" > file.txt
git diff > /tmp/t6.patch
git checkout -- file.txt
cat /tmp/t6.patch
# Identify the same ---/+++/@@ structure from Task 1, plus the git-specific
# header lines above it — including the index line's blob-hash-before..
# blob-hash-after and file-mode metadata, and the a/ b/ path prefixes.

mkdir -p /tmp/t6-target
cp file.txt /tmp/t6-target/
cd /tmp/t6-target
patch -p0 < /tmp/t6.patch
# Confirm this FAILS — "can't find file to patch" — since -p0 looks for
# the literal path a/file.txt, which doesn't exist here.

cp /tmp/t6-repo/file.txt .
patch -p1 < /tmp/t6.patch
diff file.txt /tmp/t6-repo/file.txt
# (t6-repo/file.txt is still "version 1" since we checked it out above —
# rerun the git diff steps if you want a live comparison; the key
# confirmation here is that -p1 patched successfully where -p0 failed)
```

## What You Learned

1. ✅ Read unified diff format (`+`/`-` lines, `@@` hunk headers) fluently,
   and used `diff`'s exit status and `-q` for scripting
2. ✅ Read context diff format, used `-y` for side-by-side comparison, and
   used `-i`/`-w` to filter out case/whitespace noise
3. ✅ Compared entire directory trees with `diff -r`, and used `diff -rq`
   for a faster files-only pass
4. ✅ Resolved a three-way merge conflict with `diff3`, reading all four
   inline conflict markers (`<<<<<<<`, `|||||||`, `=======`, `>>>>>>>`)
5. ✅ Applied a diff as a patch (unified or context format), including
   reversing one, dry-running first, stripping path prefixes with `-p`,
   and explaining how `patch` locates a hunk's application point via
   context lines
6. ✅ Explained how `git diff` relates to standard `diff` output, and why
   its `a/`/`b/` path prefixes require `-p1` when applying with plain `patch`

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
change, which `patch` uses per §1.5 to locate where to apply the change)
no longer match the file's current content at the expected location,
causing the hunk to fail to apply. This isn't a corrupted patch file; it's
a stale patch being applied against a file that's no longer in the state
the patch assumes. `patch --dry-run` is exactly the right command — it
reports success/failure without touching the file, letting you investigate
context mismatches before deciding whether to force, regenerate, or
manually reconcile the patch.

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

---

**Q2. A patch that applied cleanly last week fails to apply today on the same target file, with no changes to the patch file itself. What's the most likely cause?**

The target file has drifted — someone else made an unrelated change to the
same file in the interim, and the patch's context lines (used by `patch`
to locate the correct application point, not line numbers alone) no
longer match the file's current content there. This isn't necessarily a
corrupted patch; `patch --dry-run` against the current file state is the
correct first diagnostic step, since it reveals exactly this kind of
mismatch without risking a partial application.

---

**Q3. A patch exported from `git diff` fails with "can't find file to patch" when applied with plain `patch` in the target directory. What's the likely cause, and the fix?**

`git diff` always prefixes paths with `a/` (before) and `b/` (after),
regardless of the actual repository layout. Applying with `patch -p0`
(the default, no stripping) makes `patch` look for the literal path
`a/filename`, which doesn't exist outside a git checkout structured that
way. The fix is `patch -p1`, which strips exactly one leading path
component — the `a/`/`b/` prefix — leaving the real filename to search
for. This is a distinct problem from a stale/drifted target file (a
context-mismatch failure); a `-p`-level mismatch fails immediately and
explicitly, before `patch` even attempts to locate the hunk's context.

---

**Q4. What does `diff3`'s conflict marker set actually show, and when does `diff3` resolve a difference without flagging a conflict?**

`diff3 -m`'s full marker set (`<<<<<<<`, `|||||||`, `=======`, `>>>>>>>`)
shows "mine," the common ancestor's original content, and "theirs" as
three distinct sections at a genuine conflict point — both compared
versions changed the same content differently from the ancestor, which
`diff3` cannot resolve automatically. If only one side changed a given
piece of content (with the other matching the ancestor unchanged), `diff3`
resolves it silently without any conflict marker at all, since there's no
ambiguity about which version's change should win.

---

## Key Takeaways

---

1. **Unified diff (`-u`) is the standard format everywhere that matters
   today** — `git diff`, code review tools, and most generated `.patch`
   files all use it, interleaving old/new content with `+`/`-` markers
   rather than context diff's separate before/after blocks. `-y`
   (side-by-side) is a third, distinct presentation, useful for eyeballing
   two complete short-file versions at once.
2. **`diff`'s exit status is meaningful and scriptable: `0` identical, `1`
   differ, `2` error.** `diff -q` (works on plain files, not just
   directories) pairs with it for the standard "are these two files the
   same?" idiom — `diff -q a b > /dev/null; echo $?` — the same shape as
   `grep -q`.
3. **`-i` (ignore case) and `-w` (ignore whitespace) filter out
   formatting-only noise before comparison** — the right tool when a real
   difference is being drowned out by an irrelevant formatting change,
   used deliberately rather than habitually.
4. **`diff -r` flags files present in only one tree with `Only in DIR:
   file` — a distinct signal from a content change (`<`/`>`), easy to skim
   past if only scanning for change markers.** `diff -rq` gives the same
   which-files-differ signal faster, without any content detail, as a
   first pass across a large tree.
5. **`diff3`'s full marker set is four parts, not three** — `<<<<<<<`
   (mine), `|||||||` (the common ancestor's original content, easy to
   overlook), `=======` (separator), `>>>>>>>` (theirs). A change on only
   one side resolves automatically, without any conflict flag.
6. **A patch failing to apply after previously working cleanly usually
   means the target file has drifted, not that the patch is corrupted.**
   `patch` locates a hunk's application point using its surrounding
   context lines, not line numbers alone — an unrelated edit to those same
   lines breaks that match. `patch --dry-run` diagnoses this without
   risking a partial application.
7. **`-p`/`--strip` controls how many leading path components `patch`
   removes before searching for the target file — `-p1` is the standard
   fix for a `git diff`-exported patch, whose paths are always prefixed
   `a/`/`b/` regardless of actual repo structure.** This fails immediately
   and explicitly ("can't find file to patch"), a distinct failure mode
   from context-line drift, which fails per-hunk instead.
8. **`patch -R` reverses a patch using the exact same diff file** — no
   need to generate a separate reverse patch. `patch` also accepts
   context-format diffs directly, auto-detecting the format, with no
   manual conversion to unified required.
9. **`git diff`'s output is standard unified diff with git-specific header
   metadata layered on top** — including an `index` line carrying
   before/after blob hashes and file mode, information plain `diff -u` has
   no equivalent for. Understanding plain `diff -u` transfers directly to
   reading any git diff or PR without needing separate git-specific
   interpretation rules for the actual change content.

> **Demo scope:** Primary concept: comparing files and directories, and
> applying the differences. Supporting concepts: unified/context/side-by-side
> diff formats, exit-status scripting, noise-reduction flags, diff3
> three-way comparison and its full marker set, patch application/reversal,
> path-component stripping, context-based hunk location, directory diffs,
> git diff's relationship to standard diff.
> Estimated completion time: 40 minutes (reading + hands-on + verification).
> Checkpoints: 6 natural stopping points (after Task 1 through Task 6).

## Quick Commands Reference

### Comparing files

| Command | Purpose |
|---|---|
| `diff -u file1 file2` | Unified diff — the modern standard |
| `diff -c file1 file2` | Context diff — older, two-block format |
| `diff -y file1 file2` | Side-by-side columns |
| `diff -i` / `-w` | Ignore case / ignore whitespace |
| `diff -q file1 file2` | One-line differ/no-output identical result |
| `diff a b; echo $?` | `0`=identical, `1`=differ, `2`=error |

### Comparing directories

| Command | Purpose |
|---|---|
| `diff -r dir1 dir2` | Recursive tree comparison, full diff content |
| `diff -rq dir1 dir2` | List only which files differ (fast first pass) |

### Three-way comparison

| Command | Purpose |
|---|---|
| `diff3 mine ancestor theirs` | Show conflicts (`====`) between two divergent versions |
| `diff3 -m mine ancestor theirs` | Produce merged output with all four inline conflict markers |

### Applying patches

| Command | Purpose |
|---|---|
| `diff -u a b > x.patch` | Generate a patch file |
| `patch --dry-run file < x.patch` | Check if it would apply cleanly, without applying |
| `patch file < x.patch` | Apply (unified or context format, auto-detected) |
| `patch -p1 < x.patch` | Strip one leading path component (the `a/`/`b/` git prefix) |
| `patch -R file < x.patch` | Reverse an applied patch |

## Appendix A — Anki Cards

**`m05-10-diff-comparison-anki.csv`:**

```
#deck:Linux Mastery::Module 5 - Text Processing & Search Tools::Demo M05-10 - diff Comparison
#separator:Comma
#columns:Front,Back,Tags
"In a unified diff hunk header @@ -1,3 +1,4 @@, what do the two number pairs mean?","-1,3 means the first file's shown section starts at line 1 and spans 3 lines; +1,4 means the second file's shown section starts at line 1 and spans 4 lines.","demo22,diff,unified,hunk-header"
"What are diff's three possible exit codes, and what does each mean?","0 = files identical, 1 = files differ, 2 = an error occurred (e.g. a missing file) and diff couldn't complete the comparison — three distinct outcomes, not just success/failure.","demo22,diff,exit-codes,fundamentals"
"What's the standard scripting idiom for 'are these two files identical?' using diff?","diff -q file1 file2 > /dev/null; then check $? — -q prints nothing for identical files (exit 0) or one 'Files X and Y differ' line for different ones (exit 1). Same shape as grep -q from Demo M05-05.","demo22,diff,scripting,cross-demo"
"What do -i and -w do when comparing two files with diff?","-i ignores case differences; -w ignores whitespace differences. Both filter out formatting-only noise before the comparison happens, useful when a real difference is being drowned out by an irrelevant formatting change.","demo22,diff,noise-reduction"
"What does diff -y produce that's different from both -u (unified) and -c (context)?","Two literal side-by-side columns — the two files' complete content shown next to each other, with | marking a changed line and </> marking a line present only on one side. A third, distinct presentation from interleaved (unified) or block-separated (context).","demo22,diff,side-by-side"
"diff -r shows a content change for file2.txt as < old-line / > new-line, but the actual difference is that file3.txt exists in only one of the two directories. What output signals that?","'Only in DIR: file3.txt' — a distinct signal from the </> content-change markers, easy to miss if only scanning for content diffs.","demo22,diff,directory,fundamentals"
"What does diff3's ==== conflict marker actually indicate?","A genuine three-way conflict — both compared versions changed the same content differently from the common ancestor, which diff3 cannot resolve automatically and requires a human decision.","demo22,diff3,conflict"
"A patch applied cleanly last week but fails to apply today on the same target file, with the patch file itself unchanged. What's the most likely cause?","The target file has drifted — someone made an unrelated change to the same file in the interim, and the patch's context lines no longer match the file's current content at the expected location.","demo22,patch,troubleshooting"
"A patch generated by git diff fails with 'can't find file to patch' when applied with plain patch. What's the cause, and the fix?","git diff always prefixes paths with a/ and b/, regardless of actual repo structure. patch -p0 (default) looks for the literal a/filename, which doesn't exist outside a git checkout. Fix: patch -p1, which strips exactly one leading path component.","demo22,patch,git-diff,strip"
"What does patch -p (--strip) control, and why is -p1 the standard choice for a git diff-exported patch?","How many leading path components patch removes from the diff's internal paths before searching for the target file. -p1 strips exactly the a/ or b/ prefix git always adds, landing on the real filename.","demo22,patch,strip,fundamentals"
"How do you reverse an already-applied patch, and does this require a separately-generated reverse patch file?","patch -R file < original.patch — the -R flag reverses the SAME diff file used to apply it originally; no separate reverse patch needs to be generated.","demo22,patch,reverse"
"Why does understanding plain diff -u output transfer directly to reading git diff or a GitHub pull request?","git diff's output is unified diff format with additional git-specific header metadata (diff --git, index/blob hash lines) layered on top — everything below the ---/+++/@@ lines follows the exact same rules as standard diff -u.","demo22,git-diff,cross-tool"
```

## Appendix B — Quiz

**`m05-10-diff-comparison-quiz.md`:**

````markdown
# Quiz — Demo M05-10: diff Comparison

> One correct answer per question unless stated otherwise.
> Target: 80% or above before moving to Demo M06-01.

---

**Q1.** `diff file1.txt file2.txt` is run twice — once on two identical files, once on two files that differ. What are the respective exit codes?

- A) `0` and `0` — `diff` only signals errors via exit code, not content differences
- B) `0` for identical, `1` for different — a third code, `2`, is reserved for when `diff` itself can't complete the comparison (e.g. a missing file)
- C) `1` for identical, `0` for different — the codes are inverted from typical Unix convention
- D) `diff` never sets a meaningful exit code; only its printed output matters

<details>
<summary>Answer</summary>

**B** — `diff`'s exit status directly encodes the comparison result: `0` (identical), `1` (differ), `2` (error), making it directly usable in shell conditionals without parsing output text.

Trap: A understates what the exit code communicates; C reverses the actual convention; D wrongly dismisses a genuinely meaningful, scriptable exit status.

</details>

---

**Q2.** `diff -q file1.txt file2.txt` is run on two identical files. What's printed, and what's the exit code?

- A) `Files file1.txt and file2.txt are identical`, exit `0`
- B) Nothing at all is printed, exit `0`
- C) Nothing at all is printed, exit `1`
- D) `Files file1.txt and file2.txt differ`, regardless of whether they're actually identical

<details>
<summary>Answer</summary>

**B** — `-q` only prints something when files actually differ (`Files X and Y differ`); for identical files, it's silent, with exit code `0` confirming the match.

Trap: A invents an explicit "identical" message `-q` doesn't print; C pairs the correct silent output with the wrong exit code; D wrongly claims the differ-message always prints regardless of actual content.

</details>

---

**Q3.** A patch generated by `git diff` is applied with `patch -p0 < change.patch` in a directory containing only the target file (no `a/` subdirectory), and it fails with "can't find file to patch." What's the correct fix?

- A) Regenerate the patch with a different `diff` flag
- B) Use `patch -p1` instead — `git diff` always prefixes paths with `a/`/`b/`, and `-p1` strips exactly that one leading path component, leaving the real filename `patch` can actually find
- C) The patch file is corrupted and must be regenerated from a clean git state
- D) Rename the target file to include an `a/` prefix in its path

<details>
<summary>Answer</summary>

**B** — This is a path-stripping mismatch, not a corruption or generation issue. `-p0` (the default, no stripping) searches for the literal `a/filename` path git always writes; `-p1` removes that one leading component.

Trap: A and C both misdiagnose a `-p`-level issue as a generation/corruption problem; D proposes restructuring the target environment instead of using the flag `patch` provides for exactly this situation.

</details>

---

**Q4.** How is a `-p`-level mismatch failure (Q3) different from a context-line-drift failure (a patch failing because the target file has since changed)?

- A) They are the same failure, just with different error text
- B) A `-p`-level mismatch fails immediately with "can't find file to patch," before `patch` even attempts to locate a hunk's context; a context-drift failure occurs per-hunk, after `patch` has found the file but can't match the surrounding unchanged lines to the file's current content
- C) A `-p`-level mismatch can only happen with context-format diffs, never unified
- D) Context-line drift can be fixed with `-p`; a `-p`-level mismatch cannot

<details>
<summary>Answer</summary>

**B** — These are genuinely distinct failure modes at different stages of `patch`'s process: file-location failure (wrong `-p` level) happens before any hunk is even examined; context-mismatch failure happens once the file is found but its content near the hunk doesn't match what the patch expects.

Trap: A wrongly conflates two different failure mechanisms; C invents an unrelated format restriction; D confuses which flag fixes which problem — `-p` fixes location, not content-context mismatches.

</details>

---

**Q5.** `diff3 -m mine.txt ancestor.txt theirs.txt` produces output containing `<<<<<<<`, `|||||||`, `=======`, and `>>>>>>>` at a conflict point. What does the `|||||||` section specifically show?

- A) It's a formatting separator with no actual content, purely visual
- B) The common ancestor's original content — what both "mine" and "theirs" started from, before either side's changes
- C) A duplicate of the `=======` line, included for readability
- D) The line number where the conflict occurs

<details>
<summary>Answer</summary>

**B** — `diff3 -m`'s full four-marker output includes the ancestor's original content between `<<<<<<<` and `=======`, distinct from both "mine" and "theirs" — genuinely useful context that's easy to overlook if you only expect the three markers git's default conflict display shows.

Trap: A wrongly dismisses it as empty formatting; C wrongly claims duplication; D invents an unrelated line-number-display purpose.

</details>

---

**Q6.** `diff -i file1.txt file2.txt` reports no differences, but a plain `diff file1.txt file2.txt` (no flags) does. What's the most likely explanation?

- A) `-i` is broken and should be reported as a bug
- B) The files differ only in letter case (e.g. "Hello" vs. "hello"); `-i` ignores case when comparing, so a purely case-based difference disappears under `-i` but shows up without it
- C) `-i` compares only the first line of each file
- D) `-i` requires both files to be identical in length to work at all

<details>
<summary>Answer</summary>

**B** — This is exactly `-i`'s intended effect: filtering out case-only differences so they don't register as a "real" difference, useful when case genuinely doesn't matter for the comparison at hand.

Trap: A misdiagnoses correct, documented behavior as a malfunction; C and D both invent restrictions `-i` doesn't actually have.

</details>

---

**Q7.** In context diff (`-c`) output, a line is prefixed with `!`. What does this specifically mean?

- A) The line was added, present only in the second file
- B) The line changed between the two files being compared — distinct from unified diff's separate `+`/`-` markers for the same kind of edit
- C) The line contains a syntax error
- D) The line is a comment and should be ignored

<details>
<summary>Answer</summary>

**B** — Context diff uses `!` to mark a changed line within its two-block before/after structure — a different notation from unified diff's `+`/`-` pair for the same underlying edit, which is exactly why misreading `!` as if it meant something else would misinterpret the change.

Trap: A misapplies unified diff's `+` meaning to context diff's different marker; C and D both invent meanings `!` doesn't have in this context.

</details>

---

**Q8.** A `git diff` shows `index abc1234..def5678 100644` above the `---`/`+++` lines. What does this line represent?

- A) The line numbers where the change begins and ends
- B) The blob hash of the file's content before (`abc1234`) and after (`def5678`) the change, plus the file's mode (`100644`, a regular non-executable file) — metadata plain `diff -u` has no equivalent for
- C) The commit hashes of the two most recent commits touching this file
- D) A checksum used only to detect if the patch file itself was corrupted in transit

<details>
<summary>Answer</summary>

**B** — This is git-specific object metadata layered on top of the standard unified diff format — the blob hashes identify the exact file content git is tracking before and after, and the mode reflects file permissions/type. It has no counterpart in plain filesystem `diff -u`, which has no concept of git's internal object model.

Trap: A confuses this with hunk header line numbers (a separate, `@@`-prefixed line); C misidentifies blob hashes as commit hashes; D invents an unrelated corruption-detection purpose.

</details>

---

Score guide:

| Score | Action |
|---|---|
| 7/8 or 8/8 | Import Anki cards, move to Demo M06-01 |
| 6/8 | Review the wrong answers, then proceed |
| 5/8 | Re-read the relevant sections, retry those questions |
| Below 5/8 | Re-read the full demo and redo the walkthrough before proceeding |
````