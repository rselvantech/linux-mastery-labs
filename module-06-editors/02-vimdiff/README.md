# Demo M06-02 — vimdiff: Visual Diff & Merge Workflows

## Overview

Demo M05-10 gave you `diff`/`patch` — text-based, read top-to-bottom, and
genuinely the right tool for scripted, reproducible transformations. But
when you're actually deciding *which* change to keep, side-by-side and
interactive beats reading a unified diff hunk by hunk. `vimdiff` is Vim's
built-in diff mode: the same file-comparison problem Demo M05-10 solved with
text output, now solved with synchronized, colored, interactive windows
you can act on directly — and it's the same tool `git mergetool` reaches
for by default. This demo covers reading a diff-mode display, navigating
between differences, transferring changes between windows, and setting up
a 3-way merge.

## Demo Objectives

1. Launch `vimdiff` on two files and read the diff-mode display — colored
   highlighting, filler lines, and folded unchanged blocks
2. Navigate between differences with `]c` and `[c`
3. Use `do` (diff obtain) and `dp` (diff put) to transfer a change between
   windows, and explain the direction each one moves in
4. Set up and resolve a simplified 3-way merge (base/local/remote)
5. Explain when `vimdiff` is the right tool versus `patch`/scripted
   resolution from Demo M05-10

## Directory Structure

```
02-vimdiff/
├── README.md                # this file
├── m06-02-vimdiff-anki.csv  # Anki flashcard deck
└── m06-02-vimdiff-quiz.md   # standalone quiz
```

## Environments

| Task | Ubuntu 24.04 | Rocky Linux 9.8 | EC2 Amazon Linux |
|---|---|---|---|
| All lab tasks | ✅ (`vim` preinstalled or `sudo apt install vim`) | ✅ (`vi`/`vim` preinstalled or `sudo dnf install vim`) | Not required |

> **Verification note:** As with Demo M06-01, this sandbox has no network
> access to install Vim, so nothing in this demo could be captured from a
> live session here. Most of what follows (diff-mode highlighting, `]c`/
> `[c` navigation, the general shape of a 3-way layout) is long-stable,
> widely-documented Vim behavior. Two specifics are marked
> `[Unverified — test before use]` below — the exact `do`/`dp` transfer
> direction, and precisely what a plain 3-window `vim -d` merge shows
> versus `git mergetool`'s actual 4-window layout — and should be
> confirmed on a real Ubuntu/Rocky9 machine before being treated as
> fully confirmed fact.

## Recall Check — 01-vim-neovim

Answer these from memory before continuing — no looking back at Demo M06-01.

1. What's the difference between the unnamed register and `"0`, and why
   does that matter after an unrelated `dd`?
2. A macro that worked on the first 12 lines of a 50-line replay starts
   producing wrong results afterward. Is this a Vim bug?
3. When would you choose `sed`/`awk` over recording a Vim macro for the
   same repetitive edit, and why?

<details>
<summary>Answers</summary>

1. The unnamed register is overwritten by *any* delete or yank,
   regardless of context. `"0`, the dedicated yank register, is updated
   only by yank commands — never by delete or change — so it still holds
   the most recent genuine yank even after an unrelated `dd` has
   overwritten the unnamed register.

2. No — the macro faithfully replays the exact recorded keystrokes every
   time. The problem is the macro's implicit assumption that every
   targeted line has the same structure, which breaks down once a line
   doesn't match that assumption.

3. When the change is fully mechanical, well-defined, and doesn't need
   human judgment per line — `sed`/`awk` express it reproducibly and
   rerunnably, while a macro lives only in that editing session's memory
   unless deliberately saved.

</details>

## Part 1 — Concepts

### 1.1 Why vimdiff, when Demo M05-10 already covered diff/patch

Unified diff output (Demo M05-10) is optimized for scripting and for a quick
read of *what* changed. It's not optimized for *deciding* what to do about
it when both sides have independently valid changes and a human needs to
pick, or combine, them. `vimdiff` puts the files side by side as real,
editable Vim buffers, synchronized so scrolling one scrolls the other, and
lets you act on a specific difference directly instead of manually
re-typing a resolution based on what you read in a hunk.

```bash
vimdiff /tmp/vdiff-a.txt /tmp/vdiff-b.txt
```

This is equivalent to opening the first file normally and manually
diffing in the second:

```bash
vim /tmp/vdiff-a.txt
:vertical diffsplit /tmp/vdiff-b.txt
```

**A note on saving:** this demo assumes you already know `:w` (write/save
the current buffer) and `:wq` (write and quit) from ordinary Vim use —
they're called out explicitly in the tasks only where confirming that a
resolved difference actually persisted to disk matters.

### 1.2 Reading the diff-mode display

Once in diff mode, Vim adds visual information beyond plain text:

```
Highlighted background   A line that differs between the two files
Filler lines (~)         Blank placeholder lines inserted so both windows
                          stay aligned line-for-line, even when one file
                          has extra lines the other doesn't
Foldcolumn + folded text  Blocks of IDENTICAL, unchanged lines are folded
                          away by default — you're shown differences,
                          not the entire file
```

The diff is computed once, when the windows are set up (or when you last
ran `:diffthis`) — it does **not** continuously re-diff as you type. Two
commands manage this directly:

```
:diffthis     # manually mark the current window as part of the diff
              # comparison (used when you open a second file into an
              # existing diff-mode window yourself, rather than starting
              # with vimdiff)
:diffoff      # remove the CURRENT window from diff mode
:diffoff!     # remove diff mode from ALL windows at once
:diffupdate   # force Vim to recompute the diff right now — needed if
              # something changed a file's on-disk content since the
              # comparison was last computed, and the highlighting looks
              # stale or wrong for what you're actually looking at
```

### 1.3 Navigating differences — ]c and [c

```
]c    # jump to the next difference (forward)
[c    # jump to the previous difference (backward)
```

This is the diff-mode equivalent of `n`/`N` after a search (Demo M06-01,
§1.2) — a dedicated motion for jumping directly between the things you
actually care about, instead of scrolling manually past folded, identical
blocks to find the next real difference.

### 1.4 do and dp — transferring a change between windows

```
do    # diff obtain — pull the difference FROM the OTHER window INTO
      # the current window                                  [Unverified — test before use]
dp    # diff put — push the difference FROM the current window
      # INTO the other window                                [Unverified — test before use]
```

Think of it from the current window's point of view: `do` brings a change
*in* (obtain), `dp` sends a change *out* (put). Both operate on whichever
diff hunk the cursor is currently positioned inside — same "act on what
the cursor is on" principle as Demo M06-01's operator+motion commands, just
scoped to a diff hunk instead of a word or line.

Because `do`/`dp` move content in opposite directions and the difference
between them matters a great deal (using the wrong one silently
overwrites a good change with a stale one — see this demo's Break-Fix),
always confirm which window is "current" (has the cursor) before using
either, rather than assuming based on left/right screen position alone.

### 1.5 A simplified 3-way merge

```bash
vim -d base.conf local.conf remote.conf
```

This opens three windows side by side, all in diff mode against each
other simultaneously: `base.conf` (the common ancestor before either side
changed anything), `local.conf` (your changes), and `remote.conf`
(someone else's changes). This is the same three-version shape a `git`
merge conflict represents — it's why `git mergetool` can drive Vim
directly for exactly this task.

**How this simplified version differs from `git mergetool`'s actual
setup:** `git mergetool` typically opens a *fourth* window — `MERGED` —
spanning the bottom of the screen, which is the actual file you edit and
save as the resolved result; `LOCAL`/`BASE`/`REMOTE` stay read-only
references above it.                                    `[Unverified — test before use]`
For this lab, we work with just the three windows directly and manually
choose one (`local.conf`) as the file we'll edit and save as the merged
result — the underlying `do`/`dp` mechanics are identical either way, just
without a dedicated fourth "output" window.

Unlike `diff3` (Demo M05-10), plain `vimdiff` does not automatically detect
or flag a genuine three-way conflict with a `====`-style marker — it
shows you all three versions' differences and leaves the judgment of
what's a real conflict entirely to you.

### 1.6 When vimdiff is the right tool vs. patch/scripted resolution

`vimdiff` earns its place when a human needs to look at competing changes
and decide — pick one side, combine parts of both, or rewrite something
neither side got quite right. `patch` (Demo M05-10) earns its place when the
change is already fully decided and just needs to be applied identically,
possibly unattended, possibly across many files. Reach for `vimdiff` when
resolving is a judgment call; reach for `patch`/a scripted diff-apply when
the resolution is already known and repeatable — the same underlying
distinction Demo M06-01 drew between Vim macros and `sed`/`awk`, applied here
to merging instead of editing.

## Part 2 — Hands-On Lab

### Commands used in this demo

| Command | Purpose |
|---|---|
| `vimdiff file1 file2` | Open two files in synchronized diff mode |
| `:vertical diffsplit file2` | Add a second file into diff mode against the currently open one |
| `:diffthis` / `:diffoff` / `:diffoff!` | Manually add/remove diff-mode status for a window |
| `:diffupdate` | Force Vim to recompute the diff right now |
| `]c` / `[c` | Jump to next / previous difference |
| `do` / `dp` | Obtain (pull in) / put (push out) a diff hunk |
| `vim -d file1 file2 file3` | Open a 3-way diff across three files |

### Task 1 — Two-file diff basics

```bash
printf 'line one\nline two\nline three\nline four\n' > /tmp/vdiff-a.txt
printf 'line one\nline TWO changed\nline three\nnew line five\n' > /tmp/vdiff-b.txt
vimdiff /tmp/vdiff-a.txt /tmp/vdiff-b.txt
```

Observe the highlighted lines and the filler line Vim inserts in
`vdiff-a.txt`'s window to keep it aligned with `vdiff-b.txt`'s extra
fifth line. Use `]c` to jump to the first difference, `]c` again for the
next, `[c` to jump back. Confirm you can reach every difference using
only `]c`/`[c` — no manual scrolling needed.

### Task 2 — do and dp: transferring a change

With the cursor inside the "line TWO changed" hunk and `vdiff-a.txt`'s
window active (current):

```
do
```

Confirm `vdiff-a.txt`'s window now shows "line TWO changed" too — the
difference was pulled in from the other window. Undo (`u`, Demo M06-01 §1.1),
then instead move the cursor to the "new line five" hunk while
`vdiff-b.txt`'s window is current, and run:

```
dp
```

Confirm `vdiff-a.txt`'s window now also shows "new line five" — this time
pushed out from the current window into the other one. Save both with
`:w` in each window, then confirm from the shell:

```bash
diff /tmp/vdiff-a.txt /tmp/vdiff-b.txt
# no output = files now identical, confirming both transfers actually
# persisted to disk, not just to the in-editor display
```

### Task 3 — A simplified 3-way merge

```bash
printf 'worker_processes 1;\nkeepalive_timeout 65;\n' > /tmp/base.conf
printf 'worker_processes 1;\nkeepalive_timeout 30;\n' > /tmp/local.conf
printf 'worker_processes 4;\nkeepalive_timeout 65;\n' > /tmp/remote.conf

vim -d /tmp/base.conf /tmp/local.conf /tmp/remote.conf
```

`local.conf` changed the timeout; `remote.conf` changed the worker count
— two independent, non-overlapping changes from the same base. With
`local.conf`'s window current, use `do`/`dp` against `remote.conf`'s
window to pull remote's `worker_processes 4;` change into `local.conf`,
so `local.conf` ends up with both changes: `worker_processes 4;` and
`keepalive_timeout 30;`. Save `local.conf` with `:w` — this is your
merged result.

```bash
cat /tmp/local.conf
# worker_processes 4;
# keepalive_timeout 30;
```

### Task 4 — Recognize the vimdiff-vs-patch boundary

You have a `.patch` file (Demo M05-10 format) representing a fully-decided,
already-reviewed configuration change that needs to go out identically to
30 servers overnight, unattended. Separately, you have two engineers'
independently-modified copies of the same config file, and someone needs
to look at both and decide what the final version should contain. Which
scenario calls for `patch`, and which calls for `vimdiff`, and why —
per §1.6?

## What You Learned

- ✅ Launched `vimdiff` on two files and read the diff-mode display —
  highlighted differences, filler lines, and folded unchanged blocks
- ✅ Navigated between differences with `]c` and `[c`
- ✅ Used `do` and `dp` to transfer a change between windows, in the
  correct direction for what was needed
- ✅ Set up and resolved a simplified 3-way merge across base/local/remote
  versions of a file
- ✅ Explained when `vimdiff` is the right tool versus `patch`/scripted
  resolution from Demo M05-10

## Part 3 — Break-Fix Scenario

### Scenario — NovaTech Ops: two vimdiff incidents

> **Incident report — NovaTech Ops, ticket #4803**
>
> Alert 1: "An engineer resolving a config merge says their carefully-typed
> local changes just vanished and got replaced with the old remote
> version." Alert 2: "A file was edited by a deploy script while someone
> had it open in vimdiff — the diff highlighting still shows the old
> differences and doesn't seem to notice the file changed."

**Incident 1 — the wrong-direction transfer**

```
(local.conf window is current, cursor inside a hunk where local's
 change is the one that should be kept)
dp
(local's good change was just pushed OUT to remote.conf's window —
 and now the engineer's next do, from the wrong window, pulled the
 stale remote content back into local.conf, overwriting their own edit)
```

Tasks:
1. Reference §1.4 — what does `dp` actually do, and why did running it
   from the local window (instead of `do`) push the good change away
   instead of keeping it?
2. What should the engineer have confirmed before running either command,
   given that `do` and `dp` move content in opposite directions?
3. Is the original local content still recoverable using anything covered
   in Demo M06-01 or this demo?

**Incident 2 — stale diff highlighting after an external edit**

```
$ vimdiff config-a.conf config-b.conf
(a deploy script overwrites config-b.conf on disk while this session is
 still open)
(the vimdiff window for config-b.conf still shows the OLD differences —
 not what's now actually on disk)
```

Tasks:
1. Reference §1.2 — when is the diff actually computed, and why doesn't
   Vim automatically notice an external change to a file it's displaying?
2. What single command forces Vim to recompute the diff against the
   file's current on-disk content?
3. What's the general lesson here about diff-mode display versus what's
   actually on disk at any given moment?

<details>
<summary>Answers</summary>

**Incident 1:** `dp` (diff put) pushes the difference *from the current
window out to the other window* — the exact opposite direction from `do`
(diff obtain), which pulls a difference *in*. Because `local.conf`'s
window was current and held the change worth keeping, running `dp` sent
that good change out to `remote.conf`'s window instead of protecting it —
`local.conf` itself was left unchanged at that point, but the engineer
then ran `do` from a window expecting to "get" something, which pulled
the still-stale remote content into `local.conf`, overwriting the good
change with old content. The engineer should have confirmed, before
running either command, which window was actually current and which
direction the specific command they were about to run would move content
— not assumed based on left/right screen position. If the original edit
was made recently in the same session and not yet saved over multiple
times, `u` (undo, Demo M06-01 §1.1) is the direct recovery path; if it was
already saved and overwritten, and was yanked into `"0` or a named
register at some point before the mistake, those registers (Demo M06-01 §1.5)
could still hold it — but neither is guaranteed once a save has already
happened and the buffer's undo history has moved past the point of loss.

**Incident 2:** Per §1.2, the diff is computed once, when the windows are
set up (or last explicitly refreshed) — Vim does not continuously watch
the file on disk for changes and re-diff automatically. An external
process writing to one of the files doesn't trigger a re-diff on its own;
the highlighting keeps showing whatever was true at the last computation,
which is now stale. `:diffupdate` forces Vim to recompute the diff against
each window's actual current content. The general lesson: diff-mode
display is a snapshot taken at setup/last-refresh time, not a live view —
any time a file might have changed outside the current Vim session (a
script, another editor, a git operation), refresh explicitly with
`:diffupdate` before trusting what's highlighted, rather than assuming the
display is automatically current.

</details>

## Interview Prep

**Q1. Why would a team configure `git mergetool` to use `vimdiff` instead of just reading the conflict markers `git` inserts directly into the file?**

Conflict markers (`<<<<<<<`, `=======`, `>>>>>>>`) present both sides as
flat text you read and manually edit around — workable, but easy to leave
a stray marker behind or misjudge which lines belong to which side on a
complex conflict. `vimdiff` presents the same information as synchronized,
navigable windows with `]c`/`[c` to jump straight to each actual
difference and `do`/`dp` to transfer content directly, rather than
retyping a resolution by hand — the same efficiency argument as Vim's
operator+motion grammar (Demo M06-01), applied to merging instead of editing.

---

**Q2. An engineer says "I ran `do` and my changes disappeared." What's the most likely explanation, given how `do` and `dp` work?**

`do` (diff obtain) pulls content *in* from the other window — it doesn't
push anything out. If the engineer's own good change disappeared right
after running `do`, the most likely explanation is that `do` correctly
did what it does (pull in from the other window), but the engineer was
in the window holding their good change when they ran it, so it was
overwritten by whatever the other window held — they needed `dp` (push
this out, protecting it) or to run `do` from the *other* window instead,
not `do` from the window they wanted to protect.

---

**Q3. Why doesn't `vimdiff` automatically flag a "conflict" the way `diff3` does with its `====` marker, even in a 3-way base/local/remote setup?**

`vimdiff` shows differences between windows — it's a display and
navigation tool, not a conflict-resolution algorithm. `diff3` (Demo M05-10)
specifically computes whether both non-base versions changed the *same*
content differently from the ancestor, and flags that case explicitly.
Plain `vim -d` across three files gives you the same three versions
side by side, but leaves the judgment of what constitutes a real,
unresolvable conflict entirely to the person looking at it.

---

**Q4. A file being viewed in `vimdiff` was just changed on disk by an external process. What does the vimdiff window show, and what fixes it?**

It keeps showing whatever the diff looked like the last time it was
computed — Vim doesn't watch the filesystem and re-diff automatically the
instant something else changes a file. The display is stale until
`:diffupdate` is run, which forces Vim to recompute the diff against each
window's actual current on-disk content.

---

## Key Takeaways

1. **`vimdiff` solves the same comparison problem as Demo M05-10's `diff`, but
   for judgment-based decisions instead of scripted, reproducible ones.**
   Reach for `vimdiff` when a human needs to look at competing changes and
   decide; reach for `patch` when the resolution is already fully decided
   and just needs applying, possibly unattended.
2. **The diff display is folded, filler-aligned, and computed once — not
   a continuous live view.** Unchanged blocks stay folded away by default;
   filler lines keep both windows aligned line-for-line; `:diffupdate`
   forces a recompute when a file changes outside the current session.
3. **`]c`/`[c` jump directly between actual differences**, the same way
   `n`/`N` jump between search matches (Demo M06-01) — no manual scrolling
   through folded, identical content required.
4. **`do` obtains (pulls in from the other window); `dp` puts (pushes out
   to the other window) — opposite directions, and mixing them up
   silently overwrites the wrong side.** Always confirm which window is
   current and which direction the command you're about to run actually
   moves content, rather than assuming from screen position.
5. **A 3-way `vim -d` layout mirrors the base/local/remote shape of a real
   merge conflict, which is why `git mergetool` can drive Vim directly for
   it** — but plain `vimdiff` doesn't auto-detect conflicts the way
   `diff3` does; it only presents the differences and leaves judgment to
   the person resolving them.
6. **Diff-mode display reflects a snapshot, not a live view of disk
   state.** Any time a file might have changed outside the current Vim
   session, refresh explicitly with `:diffupdate` before trusting the
   highlighting.

> **Demo scope:** Primary concept: visual, interactive diffing and merging
> of files with vimdiff. Supporting concepts: diff-mode display and
> navigation, `do`/`dp` obtain/put, simplified 3-way merge.
> Estimated completion time: 30 minutes (reading + hands-on + verification).
> Checkpoints: 4 natural stopping points (after Task 1 through Task 4).

## Quick Commands Reference

### Setting up diff mode

| Command | Purpose |
|---|---|
| `vimdiff file1 file2` | Open two files in synchronized diff mode |
| `vim -d file1 file2 file3` | Open a 3-way diff across three files |
| `:vertical diffsplit file2` | Add a second file into diff mode against the current one |
| `:diffthis` / `:diffoff` / `:diffoff!` | Manually add/remove diff-mode status (current / current / all windows) |
| `:diffupdate` | Force Vim to recompute the diff right now |

### Navigating and transferring

| Command | Purpose |
|---|---|
| `]c` / `[c` | Jump to next / previous difference |
| `do` | Obtain — pull a difference in from the other window |
| `dp` | Put — push a difference out to the other window |
| `:w` / `:wq` | Save the current buffer / save and quit |

## Appendix A — Anki Cards

**`m06-02-vimdiff-anki.csv`:**

```
#deck:Linux Mastery::Module 6 - Editors::Demo M06-02 - vimdiff
#separator:Comma
#columns:Front,Back,Tags
"What problem does vimdiff solve that Demo M05-10's unified diff output doesn't?","Unified diff is optimized for scripting and quickly reading what changed. vimdiff puts both files side by side as real, editable, synchronized windows, letting you act on a specific difference directly instead of manually re-typing a resolution based on reading a hunk.","demo-m06-02,vimdiff,fundamentals"
"When is the diff shown by vimdiff actually computed? What does NOT trigger a recompute?","Once, when the windows are set up (or last explicitly refreshed) — not continuously. An external process changing one of the files on disk does NOT trigger an automatic recompute; the highlighting stays stale until :diffupdate is run.","demo-m06-02,vimdiff,diffupdate"
"What do ]c and [c do, and what's the closest equivalent from Demo M06-01?","]c jumps to the next difference, [c to the previous one — the diff-mode equivalent of n/N repeating a search (Demo M06-01, §1.2), letting you jump directly between things you care about instead of scrolling manually.","demo-m06-02,vimdiff,navigation,cross-demo"
"What's the difference between do and dp, and why does mixing them up cause real damage?","do (obtain) pulls a difference IN from the other window; dp (put) pushes a difference OUT to the other window — opposite directions. Running the wrong one from the wrong window can silently overwrite a good change with stale content from the other side.","demo-m06-02,vimdiff,do-dp,common-mistake"
"Before running do or dp, what should you always confirm, given that they move content in opposite directions?","Which window is currently active (holds the cursor), and which direction the specific command about to be run actually moves content — not assumed from left/right screen position.","demo-m06-02,vimdiff,do-dp,best-practice"
"Does plain vim -d across three files (base/local/remote) automatically flag a genuine three-way conflict the way diff3 does?","No — unlike diff3 (Demo M05-10), which explicitly detects when both non-base versions changed the same content differently and flags it with a ==== marker, plain vimdiff only shows the differences between windows and leaves judging what's a real conflict entirely to the person looking at it.","demo-m06-02,vimdiff,3-way-merge,cross-demo"
"How does git mergetool's actual vimdiff layout typically differ from a plain three-window vim -d base local remote session?","git mergetool typically adds a fourth window, MERGED, spanning the bottom of the screen — the actual file you edit and save as the resolved result — while LOCAL/BASE/REMOTE stay as read-only references above it.","demo-m06-02,vimdiff,git-mergetool,3-way-merge"
"When should you reach for vimdiff versus patch/a scripted diff-apply for resolving a difference between two files?","vimdiff when a human needs to look at competing changes and make a judgment call — pick, combine, or rewrite. patch/scripted resolution when the change is already fully decided and just needs applying, possibly unattended, possibly across many files — the same underlying distinction as Vim macros vs sed/awk (Demo M06-01).","demo-m06-02,vimdiff,tool-boundary,cross-demo"
```

## Appendix B — Quiz

**`m06-02-vimdiff-quiz.md`:**

````markdown
# Quiz — Demo M06-02: vimdiff — Visual Diff & Merge Workflows

> One correct answer per question unless stated otherwise.
> Target: 80% or above before moving to Demo 24.

---

**Q1.** What does the foldcolumn/folded-text behavior in a `vimdiff`
session actually show you by default?

- A) Every line of both files, with no folding at all
- B) Blocks of identical, unchanged lines folded away, so you're shown differences rather than the entire file
- C) Only the first difference found, with the rest of the file hidden
- D) A summary count of total differences, with no actual content shown

<details>
<summary>Answer</summary>

**B** — Folding unchanged blocks is exactly what keeps a `vimdiff` session
usable on a large file — you see what's different, not every identical
line surrounding it.

</details>

---

**Q2.** You're in `vdiff-a.txt`'s window and want to pull in a change that
exists in `vdiff-b.txt`'s window but not yours. Which command, and why?

- A) `dp` — because "put" sounds like bringing something into place
- B) `do` — obtain pulls a difference IN from the other window into the current one
- C) `]c` — this both navigates and transfers in one step
- D) `:diffupdate` — this recomputes and automatically merges differences

<details>
<summary>Answer</summary>

**B** — `do` (obtain) is specifically the pull-in direction. `dp` (put)
moves the opposite way — out to the other window.

Trap: A relies on an intuitive-sounding guess about "put" rather than the
actual documented direction; C and D both invent capabilities these
commands don't have.

</details>

---

**Q3.** A deploy script overwrites one of the two files currently open in
a `vimdiff` session. What does the session show immediately afterward?

- A) The updated content, instantly — Vim watches every open file for external changes
- B) The old, stale diff — Vim computed the diff once at setup and doesn't automatically recompute on an external file change
- C) An error message stating the file changed outside Vim
- D) Both windows close automatically to prevent showing wrong data

<details>
<summary>Answer</summary>

**B** — This is exactly the scenario in this demo's Break-Fix Incident 2.
The diff is a snapshot computed at setup/last-refresh time; `:diffupdate`
is what forces a recompute against current on-disk content.

</details>

---

**Q4.** Why doesn't a plain 3-window `vim -d base.conf local.conf
remote.conf` session automatically flag a genuine conflict, the way
`diff3` does?

- A) `vimdiff` only supports two files at a time — three-file sessions silently ignore the third
- B) `vimdiff` shows differences between windows but has no built-in conflict-detection logic — that's specifically what `diff3`'s `====` marker adds, which `vimdiff` doesn't replicate
- C) Conflicts are always auto-resolved in favor of the `local` file
- D) `vimdiff` requires a plugin before it can open more than two files

<details>
<summary>Answer</summary>

**B** — `vimdiff` is a display/navigation tool for differences, not a
conflict-resolution algorithm. `diff3` specifically computes whether both
non-base versions changed the same content differently from the ancestor
and flags that case; plain `vimdiff` leaves that judgment to the person
looking at the three windows.

</details>

---

**Q5.** How does `git mergetool`'s typical `vimdiff`-based layout differ
from the simplified 3-window setup used in this demo's Task 3?

- A) They are identical — `git mergetool` just opens the same three windows
- B) `git mergetool` typically adds a fourth window, `MERGED`, which is the actual file you edit and save as the result, while `LOCAL`/`BASE`/`REMOTE` stay as read-only references
- C) `git mergetool` never uses `vimdiff` — it only supports GUI diff tools
- D) `git mergetool` opens five windows: `LOCAL`, `BASE`, `REMOTE`, `MERGED`, and a log window

<details>
<summary>Answer</summary>

**B** — The underlying `do`/`dp` mechanics are the same either way; `git
mergetool`'s real setup just adds a dedicated output window (`MERGED`)
rather than repurposing one of the three comparison windows directly, the
way this demo's simplified version does.

</details>

---

**Q6.** You have a fully-decided, already-reviewed configuration change
that needs to be applied identically to 30 servers overnight, unattended.
Per §1.6, is `vimdiff` or `patch` the right tool, and why?

- A) `vimdiff` — it's more thorough and catches more edge cases
- B) `patch` — the resolution is already fully decided and just needs to be applied consistently, without a human present to make judgment calls
- C) Either works equally well; the choice is purely stylistic
- D) `vimdiff`, because it can be scripted to run unattended across 30 servers

<details>
<summary>Answer</summary>

**B** — `vimdiff` earns its place specifically when a human needs to look
at competing changes and decide. Here nothing needs deciding — the change
is already fully known — which is exactly `patch`'s use case, especially
unattended and at scale.

Trap: D wrongly assumes `vimdiff`, an interactive tool, can run
unattended the way `patch` can.

</details>

---

**Q7.** An engineer is in `local.conf`'s window, holding a good change
they want to protect, and mistakenly runs `dp` instead of doing nothing.
What actually happens to `local.conf`'s own content as a direct result of
that single `dp`?

- A) `local.conf`'s content is immediately deleted
- B) `local.conf`'s content is unchanged by `dp` itself — `dp` pushes the current window's difference OUT to the other window; the danger comes from what's run next (e.g. a subsequent `do` pulling stale content back in), not from `dp` alone
- C) `dp` immediately overwrites `local.conf` with `remote.conf`'s content
- D) `dp` only works if `remote.conf`'s window is currently active, so nothing happens

<details>
<summary>Answer</summary>

**B** — This is the nuance in Break-Fix Incident 1: `dp` alone only
pushes the current window's content out to the other window — it doesn't
change the current window's own content. The actual loss in that incident
came from a subsequent `do`, run from the wrong window afterward, which
pulled stale content back in and overwrote the good change.

</details>

---

Score guide:

| Score | Action |
|---|---|
| 7/7 | Import Anki cards, move to Demo 24 |
| 6/7 | Review the wrong answer, then proceed |
| 5/7 | Re-read the relevant section, retry those questions |
| Below 5/7 | Re-read the full demo and redo the walkthrough before proceeding |
````