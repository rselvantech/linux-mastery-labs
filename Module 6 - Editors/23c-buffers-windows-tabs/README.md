# Demo 23c — Buffers, Windows, Tabs & Multi-File Editing

## Overview

Demo 23a asked you to work with multiple synchronized windows without
this series ever teaching what a "window" actually is in Vim, or how it
differs from a buffer or a tab — that gap gets closed here. Demo 23b's
`:g` command edits patterns *within* one file; this demo extends the same
idea *across* many files at once with `:argdo`/`:bufdo` — the natural
multi-file follow-through 23b left open. Along the way this demo also
covers one of the most common pieces of practical DevOps Vim knowledge
this series hasn't mentioned yet: saving a file you opened without
write permission, without closing and reopening it as root.

## Demo Objectives

1. Split the screen into multiple windows (`:split`/`:vsplit`) and move
   between them
2. Distinguish buffers (files loaded in memory), windows (viewports onto
   a buffer), and tabs (saved window layouts) from each other
3. List and navigate buffers directly, independent of window layout
4. Use tabs to hold separate window layouts side by side
5. Apply `:argdo` to run one Ex command across a whole set of files in a
   single pass
6. Apply `:bufdo` across all currently loaded buffers, and explain how
   its scope differs from `:argdo`'s
7. Save a file opened without write permission using the sudo-tee trick

## Directory Structure

```
23c-buffers-windows-tabs/
├── README.md                              # this file
├── 23c-buffers-windows-tabs-anki.csv      # Anki flashcard deck
└── 23c-buffers-windows-tabs-quiz.md       # standalone quiz
```

## Environments

| Task | Ubuntu 24.04 | Rocky Linux 9.8 | EC2 Amazon Linux |
|---|---|---|---|
| All lab tasks | ✅ (`vim` preinstalled or `sudo apt install vim`) | ✅ (`vi`/`vim` preinstalled or `sudo dnf install vim`) | Not required |

> **Verification note:** As with Demos 23/23a/23b, this sandbox has no
> network access to install Vim, so nothing here could be captured from a
> live session. Window/buffer/tab mechanics and the sudo-tee write trick
> are long-stable, widely-documented behavior. Two specifics are marked
> `[Unverified — test before use]` below — whether `:argdo` skips or
> aborts on a file that fails to load, and whether `:bufdo` includes
> unlisted/hidden buffers by default — and should be confirmed on a real
> machine before being treated as fully settled.

## Recall Check — 23b-vim-ex-mode

Answer these from memory before continuing — no looking back at Demo 23b.

1. What's the difference between `:s/old/new/` and `:%s/old/new/`, and
   why is mixing them up dangerous?
2. What does `:g/pattern/command` actually do, mechanically, and why
   doesn't it suffer from a macro's structural-assumption failure mode?
3. What is `:v/pattern/command` equivalent to?

<details>
<summary>Answers</summary>

1. `:s` affects only the current line; `:%s` (`%` = shorthand for `:1,$`,
   the whole file) affects every line. A single extra character turns a
   single-line fix into a whole-file change.

2. `:g` first identifies every line matching the pattern across the whole
   file, then runs the command against each one — matching happens before
   any editing starts, so it never assumes every targeted line shares the
   same structure the way a macro's fixed keystroke replay does.

3. `:g!` — both mean "run this command against every line that does NOT
   match the pattern."

</details>

## Part 1 — Concepts

### 1.1 Buffers, windows, and tabs — three different things, commonly confused

This is the single most commonly confused trio in Vim, and getting it
straight now makes everything else in this demo click:

```
Buffer   A file loaded into Vim's memory. Exists whether or not it's
         currently visible on screen. Opening a file loads it as a
         buffer; you can have many buffers loaded at once.

Window   A viewport ONTO a buffer — the rectangle of screen space
         showing you a buffer's content. Multiple windows can show
         the SAME buffer simultaneously, or different buffers.

Tab      A saved WINDOW LAYOUT — not "another file," and not a
         separate copy of anything. Each tab can hold its own
         arrangement of windows (one window, or several split ones),
         and different tabs can display the same or different buffers.
```

The relationship: buffers are the actual data (files in memory); windows
are how you view a buffer; tabs are how you group different window
arrangements. Demo 23a's two- and three-window `vimdiff` sessions were
always working with multiple buffers shown in multiple windows — this
section is the general-purpose version of that same mechanism.

### 1.2 Splitting the screen — :split / :vsplit and moving between windows

```bash
vim /tmp/file1.txt
```

```
:split /tmp/file2.txt     # or :sp — horizontal split, new window above,
                          # showing file2.txt
:vsplit /tmp/file2.txt    # or :vsp — vertical split, side by side
                          # (this is exactly what Demo 23a's
                          # :vertical diffsplit did)
```

**Moving between windows:**

```
Ctrl-w h/j/k/l   # move focus left/down/up/right, same directional
                 # letters as Demo 23's h/j/k/l motions
Ctrl-w w         # cycle to the next window
Ctrl-w =         # equalize all window sizes
:q               # close the CURRENT window (not the whole session,
                 # unless it's the only window left)
```

Splitting with no filename argument (`:split` alone) opens a second
window onto the **same buffer** you're already viewing — useful for
looking at two different parts of one large file simultaneously, which
is a genuinely different use case from splitting to show a different file.

### 1.3 The buffer list — independent of what's currently visible

```bash
vim /tmp/file1.txt /tmp/file2.txt /tmp/file3.txt
```

Opening three files this way loads all three as buffers, but only the
first is actually displayed — the other two exist in memory, unopened in
any window, until you switch to them:

```
:ls          # or :buffers — list every loaded buffer with its number
:bn          # next buffer, in the SAME window (buffer switches,
             # window doesn't change)
:bp          # previous buffer, same window
:b2          # jump directly to buffer number 2
:b filename  # jump directly by (partial) filename match
:bd          # delete (unload) the current buffer entirely
```

**The key distinction this section exists to make concrete:** switching
buffers in one window (`:bn`/`:bp`/`:b2`) changes *what you're looking
at*, without touching your window layout at all. Opening a new window
(§1.2) changes your *layout*, independent of which buffers end up shown
in it. You can have many buffers loaded and only one or two windows open
at any given moment — the counts aren't linked.

### 1.4 Tabs — separate window layouts, not separate copies of anything

```
:tabnew /tmp/file4.txt   # open file4.txt in a brand-new tab
gt                       # go to next tab
gT                       # go to previous tab
:tabs                    # list every open tab and the windows/buffers
                         # inside each one
```

Think of each tab as its own separate "desk" — its own arrangement of
windows — while the buffers themselves (the actual files in memory) are
shared across the whole session, not duplicated per tab. You could split
one tab into two windows while a second tab stays unsplit, showing a
single buffer full-screen; switching tabs with `gt`/`gT` switches which
desk's layout you're looking at, not which files exist.

### 1.5 :argdo — running one command across a set of files, in one pass

```bash
vim -es -c "args /tmp/exmulti/*.conf" -c "argdo %s/ERROR/WARN/g | update" -c "quit"
```

Broken down:

```
:args /tmp/exmulti/*.conf     # set the ARGUMENT LIST — the specific
                              # set of files this command targets —
                              # to every .conf file matching the glob
:argdo %s/ERROR/WARN/g | update
                              # for EACH file in the argument list:
                              # run Demo 23b's whole-file substitution
                              # against it, then | update saves it —
                              # but ONLY if that specific file was
                              # actually modified by the substitution
```

**Why `| update` instead of a blanket `:w`:** `update` writes the buffer
only if Vim's internal "modified" flag is set — a file in the argument
list that happened to contain no `ERROR` occurrences is left completely
untouched, including its modification timestamp. A blanket `:w` after
every file would rewrite every file's mtime regardless of whether
anything actually changed, which is a real, avoidable side effect in a
production config directory.

**What happens if one file in the argument list fails to load** (missing,
unreadable) — whether `:argdo` skips it and continues with the rest, or
aborts the whole operation — is worth confirming directly rather than
assumed. `[Unverified — test before use]`

### 1.6 :bufdo — running one command across every loaded buffer

```
:bufdo %s/ERROR/WARN/g | update
```

Same substitution-and-conditional-save pattern as `:argdo`, but scoped
differently: **`:argdo` operates on the argument list** (whatever was set
via `:args`, typically an explicit glob or file list); **`:bufdo`
operates on every currently loaded buffer**, which can include files
opened ad hoc during the session — via `:e`, `:split`, or anything else —
that were never part of the original argument list at all.

This distinction matters in practice: if you set `:args *.conf` and then,
mid-session, `:e`'d a completely unrelated file to check something, that
unrelated file becomes a loaded buffer — invisible to `:argdo` (it's not
in the argument list), but very much visible to `:bufdo` (it's a loaded
buffer). Running `:bufdo` when you meant `:argdo` can silently touch
files you never intended to include.

**Whether `:bufdo` includes unlisted/hidden buffers by default, or only
buffers Vim considers "listed"**, is worth confirming directly rather
than assumed. `[Unverified — test before use]`

### 1.7 The sudo-write trick — saving a file you can't write directly

You open a config file for a quick edit, forget it requires root to
write, and `:w` fails:

```
:w
E212: Can't open file for writing
```

Closing without saving and reopening as `sudo vim file` works, but throws
away your edit and forces you to redo it. Instead:

```
:w !sudo tee % > /dev/null
```

Broken down:

```
:w !cmd    # a completely different form of :w — instead of writing to
           # a filename, this PIPES the buffer's content to the shell
           # command cmd via stdin
%          # Vim's own filename expansion for "the current file's name"
           # (unrelated to bash's % — this is Vim's syntax, evaluated
           # before the shell ever sees the command)
sudo tee % # tee writes its stdin to the given filename AND echoes it
           # back to stdout — here, writing your buffer's content to
           # the actual file path, with root privileges
> /dev/null # discards tee's stdout echo, so it doesn't get dumped
           # into Vim's display
```

After this, Vim's own internal state doesn't know the file was written
through this side-channel — you may see a "file changed on disk, reload?"
prompt (`:e!` to reload cleanly) the next time something checks, since the
write happened via an external command, not Vim's normal write path.

## Part 2 — Hands-On Lab

### Commands used in this demo

| Command | Purpose |
|---|---|
| `:split` / `:vsplit` | Horizontal / vertical window split |
| `Ctrl-w` + `h/j/k/l` / `w` / `=` | Move between windows / cycle / equalize sizes |
| `:ls` / `:bn` / `:bp` / `:b N` / `:bd` | List / next / previous / jump to / delete a buffer |
| `:tabnew` / `gt` / `gT` / `:tabs` | New tab / next tab / previous tab / list tabs |
| `:args` / `:argdo ... \| update` | Set the argument list / run a command across it, conditional save |
| `:bufdo ... \| update` | Run a command across every loaded buffer, conditional save |
| `:w !sudo tee % > /dev/null` | Save via a piped shell command, for files needing elevated write access |

### Task 1 — Splitting and moving between windows

```bash
printf 'file one content\n' > /tmp/win-a.txt
printf 'file two content\n' > /tmp/win-b.txt
vim /tmp/win-a.txt
```

`:vsplit /tmp/win-b.txt` for a side-by-side layout. Move focus with
`Ctrl-w l` (right) and `Ctrl-w h` (left). Try `Ctrl-w w` to cycle instead.
Close the right window with `:q` and confirm the left one remains,
un-split.

### Task 2 — Buffers vs. windows

```bash
printf 'buffer one\n' > /tmp/buf1.txt
printf 'buffer two\n' > /tmp/buf2.txt
printf 'buffer three\n' > /tmp/buf3.txt
vim /tmp/buf1.txt /tmp/buf2.txt /tmp/buf3.txt
```

Run `:ls` — confirm all three are listed as loaded buffers even though
only `buf1.txt` is currently displayed. Use `:bn` twice to cycle through
all three in the same single window. Then `:split` to open a second
window, and in it run `:b2` to show `buf2.txt` — confirm you now have two
windows simultaneously showing two different buffers. `:bd 3` to unload
`buf3.txt`, then `:ls` again to confirm it's no longer listed.

### Task 3 — Tabs as separate layouts

```bash
printf 'file four\n' > /tmp/tab4.txt
```

From inside the Task 2 session: `:tabnew /tmp/tab4.txt` — confirm this
opens as a new, separate tab (`:tabs` to see the list), and that your
Task 2 window split is still intact when you switch back with `gT`. Split
the new tab too (`:vsplit`) and confirm switching tabs with `gt`/`gT`
preserves each tab's own layout independently.

### Task 4 — :argdo across many files

```bash
mkdir -p /tmp/exmulti
printf 'status: ERROR\nother line\n' > /tmp/exmulti/a.conf
printf 'status: ERROR\n' > /tmp/exmulti/b.conf
printf 'status: OK\n' > /tmp/exmulti/c.conf
```

Inside Vim: `:args /tmp/exmulti/*.conf`, then
`:argdo %s/ERROR/WARN/g | update`. Confirm from the shell afterward:

```bash
grep -r "WARN\|ERROR" /tmp/exmulti/
# a.conf and b.conf now show WARN; c.conf is untouched (no ERROR to
# begin with) — and per §1.5, c.conf's mtime should be unchanged too,
# since `update` never wrote it
ls -l /tmp/exmulti/
```

### Task 5 — :bufdo vs. :argdo: the scope difference, directly observed

Continuing the same session (or a fresh one with the same `:args` set as
Task 4): mid-session, open a completely unrelated file that was never
part of the argument list:

```
:e /tmp/exmulti/unrelated.txt
```

```bash
# (in a separate terminal, or before starting vim)
printf 'status: ERROR\n' > /tmp/exmulti/unrelated.txt
```

Run `:argdo %s/ERROR/WARN/g | update` again — confirm `unrelated.txt` is
**not** touched (it was never in the argument list). Now run
`:bufdo %s/ERROR/WARN/g | update` instead — confirm `unrelated.txt` **is**
touched this time, because `:bufdo` scopes to every loaded buffer, not
just the argument list.

### Task 6 — The sudo-write trick

```bash
sudo touch /tmp/root-owned.conf
sudo chown root:root /tmp/root-owned.conf
sudo chmod 600 /tmp/root-owned.conf
vim /tmp/root-owned.conf
```

Make an edit, then try `:w` — confirm it fails with a permission error.
Instead of quitting and reopening as `sudo vim`, run:

```
:w !sudo tee % > /dev/null
```

Confirm the save succeeds without losing the in-progress edit, then
verify from the shell:

```bash
sudo cat /tmp/root-owned.conf
```

## What You Learned

- ✅ Split the screen into multiple windows and moved between them
- ✅ Distinguished buffers, windows, and tabs from each other
- ✅ Listed and navigated buffers directly, independent of window layout
- ✅ Used tabs to hold separate window layouts side by side
- ✅ Applied `:argdo` to run one command across a whole set of files in a
  single pass
- ✅ Applied `:bufdo` across all loaded buffers and observed how its scope
  differs from `:argdo`'s
- ✅ Saved a file opened without write permission using the sudo-tee trick

## Part 3 — Break-Fix Scenario

### Scenario — NovaTech Ops: two workspace-scope incidents

> **Incident report — NovaTech Ops, ticket #4890**
>
> Alert 1: "A `:bufdo` cleanup command modified a file the engineer had
> completely forgotten was still open from earlier in the day." Alert 2:
> "An engineer tried to save a permission-denied file, panicked, and
> manually copy-pasted the content into a fresh `sudo` session instead of
> just saving it — and the copy-paste dropped a line."

**Incident 1 — the forgotten buffer**

```
(earlier that day: :e /tmp/legacy-notes.txt, to check something, then
 moved on without closing it)
(hours later, on a different task entirely)
:args /tmp/configs/*.conf
:bufdo %s/old-hostname/new-hostname/g | update
(legacy-notes.txt — never part of :args at all — was also modified)
```

Tasks:
1. Reference §1.6 — why did `:bufdo` touch a file that was never part of
   the `:args` list the engineer had just set?
2. What command should have been used instead, given the actual intent
   was clearly "only the files I just set as my argument list"?
3. What single command, run before the destructive one, would have
   revealed the forgotten buffer in time?

**Incident 2 — manual copy-paste instead of the tee trick**

```
(editing /etc/app/config.yaml, opened as a regular user)
:w
E212: Can't open file for writing
(panicked: manually selected and copied the buffer's visible text,
 opened a new sudo vim session, pasted it in — one line scrolled off
 screen during the copy and was never included)
```

Tasks:
1. Reference §1.7 — what command would have saved the exact in-progress
   edit directly, with no copy-paste step at all?
2. Why is a manual copy-paste across a permission boundary inherently
   riskier than the tee-based approach, independent of this specific
   dropped-line incident?
3. What should the engineer check after either approach, to confirm the
   final on-disk content actually matches what was intended?

<details>
<summary>Answers</summary>

**Incident 1:** `:bufdo` scopes to every currently *loaded buffer*, not
to the `:args` list — `legacy-notes.txt` was opened earlier via `:e` and
never closed, so it remained a loaded buffer for the rest of the session,
completely independent of the `:args /tmp/configs/*.conf` set later. The
engineer's actual intent — "only the files I just specified" — is exactly
what `:argdo` is for; `:argdo %s/old-hostname/new-hostname/g | update`
would have touched only the argument-list files and left
`legacy-notes.txt` alone entirely, since it was never added to `:args`.
`:ls` (§1.3), run before the destructive command, would have surfaced
every currently loaded buffer — including the forgotten one — in time to
notice something unexpected was still open.

**Incident 2:** `:w !sudo tee % > /dev/null` (§1.7) saves the buffer's
exact current content directly, through a piped shell command, with no
manual copying, selecting, or re-typing involved at all — the file ends
up with precisely what was in the buffer, byte for byte. A manual
copy-paste is inherently riskier independent of this specific incident
because it depends entirely on a human correctly selecting, copying, and
pasting potentially many lines without a visible boundary error — content
scrolled off-screen, a partial selection, or a paste that silently
mangles whitespace are all real, easy-to-miss failure modes that a direct
buffer-to-file write never has to risk. After either approach, diffing
the final on-disk file against what the editor's buffer actually held
(or, more practically, simply re-reading the saved file back and
reviewing it line by line) is the way to catch a silent content mismatch
before trusting the write blindly.

</details>

## Interview Prep

**Q1. What's the practical difference between a Vim buffer and a Vim window, and why does the distinction matter operationally?**

A buffer is a file loaded into memory — it exists whether or not it's
currently displayed. A window is a viewport showing you some buffer's
content. The distinction matters because you can have many files loaded
as buffers simultaneously while only one or two windows are actually open
— closing a window does not close the buffer it was showing, and running
a buffer-wide command like `:bufdo` can affect files that aren't even
currently visible on screen, which is exactly the kind of surprise this
demo's Break-Fix Incident 1 demonstrates.

---

**Q2. When would you reach for `:argdo` instead of `:bufdo` for a batch edit across multiple files?**

Whenever you want the edit scoped precisely to a known, explicit set of
files — `:args` sets that set deliberately, and `:argdo` only touches
what's in it. `:bufdo` instead touches every currently loaded buffer,
which can silently include files opened ad hoc earlier in the session for
unrelated reasons. If the intent is "exactly these files, nothing else,"
`:argdo` is the safer, more precise tool.

---

**Q3. Why is `| update` used instead of a blanket `:w` when running `:argdo`/`:bufdo` across many files?**

`update` only writes a buffer if Vim's internal "modified" flag is set —
a file that had no actual matches for the substitution is left completely
untouched, including its modification timestamp. A blanket `:w` would
rewrite every file's mtime regardless of whether its content actually
changed, which is an avoidable, unnecessary side effect, particularly in
a directory where mtimes matter (deployment tooling, config-drift
detection, etc.).

---

**Q4. An engineer needs to save an edit to a file they opened without realizing it required root to write. What's the fastest correct fix, and why is it better than quitting and reopening as root?**

`:w !sudo tee % > /dev/null` — it pipes the buffer's current content
directly to a `sudo`-elevated `tee`, writing the file with the exact
in-progress edit intact. Quitting and reopening as `sudo vim` would throw
away the unsaved edit entirely, forcing the engineer to redo it from
scratch — the tee approach preserves the work already done and requires
no re-typing or copy-pasting.

---

## Key Takeaways

1. **Buffers, windows, and tabs are three different things: a buffer is a
   file in memory, a window is a viewport onto a buffer, a tab is a saved
   window layout.** Getting this straight explains why closing a window
   doesn't close its buffer, and why multiple windows can show the same
   or different buffers simultaneously.
2. **The buffer list (`:ls`) is independent of your current window
   layout — many buffers can be loaded while only a few are actually
   visible.** `:bn`/`:bp`/`:b N` switch what a window shows without
   changing your layout at all.
3. **`:argdo` scopes to the explicit `:args` list; `:bufdo` scopes to
   every currently loaded buffer — including files opened ad hoc mid-
   session that were never part of the original argument list.** Using
   the wrong one can silently touch files you never intended to include.
4. **`| update` after `:argdo`/`:bufdo` writes a file only if it was
   actually modified**, avoiding unnecessary mtime changes on files that
   had nothing to change in the first place.
5. **`:w !sudo tee % > /dev/null` saves an edit through a piped shell
   command instead of Vim's normal write path** — the direct fix for a
   permission-denied save that preserves the in-progress edit exactly,
   with no copy-paste or re-typing needed.
6. **Tabs hold independent window layouts, not independent copies of
   files** — switching tabs changes which layout ("desk") you're looking
   at; the underlying buffers are shared across the whole session.
7. **A forgotten buffer, still loaded from earlier in the session, is
   invisible to `:argdo` but fully exposed to `:bufdo`.** `:ls` before any
   batch command is the direct way to check what's actually loaded before
   trusting a buffer-wide operation.

> **Demo scope:** Primary concept: multi-file/multi-view workspace
> navigation and batch editing in Vim. Supporting concepts: window
> splits, the buffer list, tabs, `:argdo`/`:bufdo`, the sudo-write trick.
> Estimated completion time: 38 minutes (reading + hands-on + verification).
> Checkpoints: 6 natural stopping points (after Task 1 through Task 6).

## Quick Commands Reference

### Windows

| Command | Purpose |
|---|---|
| `:split` / `:sp` | Horizontal split (new window above) |
| `:vsplit` / `:vsp` | Vertical split (side by side) |
| `Ctrl-w` + `h/j/k/l` | Move focus between windows |
| `Ctrl-w w` / `Ctrl-w =` | Cycle windows / equalize sizes |
| `:q` | Close the current window |

### Buffers

| Command | Purpose |
|---|---|
| `:ls` / `:buffers` | List all loaded buffers |
| `:bn` / `:bp` | Next / previous buffer, same window |
| `:b N` / `:b name` | Jump to a buffer by number / partial name |
| `:bd` | Delete (unload) the current buffer |

### Tabs

| Command | Purpose |
|---|---|
| `:tabnew file` | Open a new tab |
| `gt` / `gT` | Next / previous tab |
| `:tabs` | List all open tabs and their contents |

### Multi-file batch editing and permission workaround

| Command | Purpose |
|---|---|
| `:args glob` | Set the argument list |
| `:argdo cmd \| update` | Run `cmd` across the argument list, save only if modified |
| `:bufdo cmd \| update` | Run `cmd` across every loaded buffer, save only if modified |
| `:w !sudo tee % > /dev/null` | Save via a piped shell command (permission workaround) |

## Appendix A — Anki Cards

**`23c-buffers-windows-tabs-anki.csv`:**

```
#deck:Linux Mastery::Module 6 - Editors::Demo 23c - Buffers, Windows & Tabs
#separator:Comma
#columns:Front,Back,Tags
"What's the difference between a Vim buffer, a window, and a tab?","A buffer is a file loaded into memory, whether or not it's currently visible. A window is a viewport showing some buffer's content. A tab is a saved window layout, not a separate copy of any file — buffers are shared across the whole session.","demo23c,buffers,windows,tabs,fundamentals"
"You have three files loaded via vim file1 file2 file3, but only one window open. How many buffers are loaded, and how do you see them all?","All three are loaded as buffers regardless of what's currently displayed. :ls (or :buffers) lists every loaded buffer, whether or not it's shown in any window right now.","demo23c,buffers,ls"
"What's the difference between :bn/:bp and opening a new window?",":bn/:bp switch which buffer the CURRENT window displays, without changing your window layout at all. Opening a new window (:split/:vsplit) changes your layout, independent of which buffer ends up shown in it.","demo23c,buffers,windows,navigation"
"What's the actual difference in scope between :argdo and :bufdo?",":argdo operates only on the explicit :args list (e.g. a glob you set deliberately). :bufdo operates on every currently loaded buffer, which can include files opened ad hoc mid-session via :e or :split that were never part of the argument list.","demo23c,argdo,bufdo,scope,common-mistake"
"Why use :argdo %s/old/new/g | update instead of following the substitution with a blanket :w on every file?","update only writes a buffer if it was actually modified — a file with no matches for the substitution is left completely untouched, including its mtime. A blanket :w would rewrite every file's mtime regardless of whether its content changed.","demo23c,argdo,update,best-practice"
"An engineer opened a file earlier in the session via :e just to check something, then forgot about it. Later they run :bufdo across an unrelated batch edit. What happens to that forgotten file?","It gets modified too — :bufdo scopes to every loaded buffer, and that forgotten file is still a loaded buffer even though it was never part of any :args list. :ls before a destructive :bufdo command would have surfaced it in time.","demo23c,bufdo,common-mistake,break-fix"
"How do you save a file you opened in Vim without realizing it requires root permission to write, without losing your in-progress edit?",":w !sudo tee % > /dev/null — this pipes the buffer's current content to a sudo-elevated tee, writing the file directly with the exact edit intact, rather than quitting and reopening as root (which would lose the unsaved edit).","demo23c,sudo-write,permissions,fundamentals"
"In :w !sudo tee % > /dev/null, what does % expand to, and whose syntax is it — Vim's or the shell's?","% is Vim's own filename expansion for 'the current file's name,' evaluated by Vim before the resulting command is ever handed to the shell — unrelated to any % syntax the shell itself might have.","demo23c,sudo-write,vim-expansion"
"What does the tabs feature actually let you do that windows/splits alone don't?","Tabs hold entirely separate window layouts side by side — one tab could be split into several windows while another tab stays a single full-screen window — letting you switch between different 'desks' of layout without losing either arrangement.","demo23c,tabs,windows"
```

## Appendix B — Quiz

**`23c-buffers-windows-tabs-quiz.md`:**

````markdown
# Quiz — Demo 23c: Buffers, Windows, Tabs & Multi-File Editing

> One correct answer per question unless stated otherwise.
> Target: 80% or above before moving to Demo 24.

---

**Q1.** You run `vim file1.txt file2.txt file3.txt`. How many buffers
are loaded, and how many windows are open?

- A) 3 buffers, 3 windows — one per file, all visible at once
- B) 3 buffers loaded, only 1 window open, showing the first file
- C) 1 buffer, 1 window — the other two files are ignored
- D) 3 windows, 0 buffers — buffers only exist after you explicitly save

<details>
<summary>Answer</summary>

**B** — All three files load as buffers immediately, but only the first
is actually displayed in the single window Vim opens by default. The
other two exist in memory, reachable via `:ls`/`:bn`/`:b2`, etc.

</details>

---

**Q2.** What's the difference between `:bn` and opening a new window with `:split`?

- A) They're identical — both open a second window
- B) `:bn` switches which buffer the current window displays, without changing your window layout; `:split` changes your layout, independent of which buffer ends up shown
- C) `:bn` only works after `:split` has been run first
- D) `:split` deletes the previously active buffer

<details>
<summary>Answer</summary>

**B** — This is the core distinction §1.2/§1.3 build toward: buffer
switching and window layout changes are two independent operations.

</details>

---

**Q3.** You set `:args /tmp/configs/*.conf`, then mid-session open an
unrelated file with `:e /tmp/notes.txt` to check something. You then run
`:argdo %s/old/new/g | update`. Does `notes.txt` get modified?

- A) Yes — `:argdo` always affects every loaded buffer
- B) No — `:argdo` only affects files in the `:args` list, and `notes.txt` was never added to it
- C) Yes, but only if `notes.txt` happens to be alphabetically near the `.conf` files
- D) It depends on which window is currently active

<details>
<summary>Answer</summary>

**B** — `:argdo`'s scope is the explicit argument list, not every loaded
buffer. `:bufdo` would have affected `notes.txt`; `:argdo` does not.

</details>

---

**Q4.** Why does `:argdo %s/old/new/g | update` use `update` instead of
plain `:w` after the substitution?

- A) `update` is simply a faster alias for `:w` with no behavioral difference
- B) `update` only writes a file if it was actually modified, leaving files with no matches — and their mtimes — completely untouched; a blanket `:w` would rewrite every file's mtime regardless of whether anything changed
- C) `:w` cannot be used inside `:argdo` at all — it's a syntax error
- D) `update` skips writing entirely and only reports what would change

<details>
<summary>Answer</summary>

**B** — This is a real, meaningful difference in a production config
directory, not just a style preference — unnecessary mtime churn on
untouched files can itself cause problems for tooling that watches
modification times.

</details>

---

**Q5.** An engineer opened a file earlier in the session via `:e` to
check something, forgot to close it, then later ran a `:bufdo` command
intended for an unrelated batch of files. What happened, and what command
would have caught it beforehand?

- A) Nothing happened — `:bufdo` respects the original `:args` list automatically
- B) The forgotten file was also modified, since `:bufdo` scopes to every loaded buffer regardless of `:args`; `:ls`, run before the destructive command, would have surfaced it as a still-loaded buffer
- C) Vim would have thrown an error refusing to run `:bufdo` with an unrelated buffer present
- D) The forgotten file would only be affected if it was the most recently opened buffer

<details>
<summary>Answer</summary>

**B** — This is exactly this demo's Break-Fix Incident 1. `:ls` is the
direct, simple check that would have revealed the forgotten buffer before
a buffer-wide command touched it unexpectedly.

</details>

---

**Q6.** What does `:w !sudo tee % > /dev/null` actually do, mechanically?

- A) It saves the file normally, then separately asks for a sudo password as a formality
- B) It pipes the buffer's current content to a shell command (`sudo tee %`), which writes it to the current filename with elevated privileges; `> /dev/null` discards `tee`'s normal stdout echo so it isn't dumped into Vim's display
- C) It permanently changes the file's ownership to root before writing
- D) It only works if Vim itself was launched with `sudo` from the start

<details>
<summary>Answer</summary>

**B** — `:w !cmd` is a distinct form of `:w` that pipes to a shell
command instead of writing to a filename directly; `%` is Vim's own
filename expansion, evaluated before the shell ever sees the command.

Trap: D describes the workaround this trick specifically exists to avoid
— reopening with `sudo` from the start, which would lose any unsaved edit.

</details>

---

**Q7.** What does a tab actually contain in Vim — a separate copy of the
open files, or something else?

- A) A separate, independent copy of every currently open file
- B) A saved window layout — its own arrangement of windows — while the underlying buffers themselves are shared across the whole session, not duplicated per tab
- C) Nothing — tabs are purely cosmetic and have no functional effect
- D) A tab is identical to a buffer, just under a different name

<details>
<summary>Answer</summary>

**B** — Buffers are the actual files in memory, shared session-wide.
Tabs hold independent window *layouts* — you can split one tab into
multiple windows while another stays single-window, and switching tabs
switches which layout you're viewing, not which files exist.

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