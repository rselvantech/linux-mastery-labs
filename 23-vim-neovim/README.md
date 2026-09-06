# Demo 23 — Vim/Neovim — the DevOps/Cloud Editor

## Overview

Every demo in this series has quietly assumed you already have a way to
edit files on a remote server — this demo makes that explicit and builds
real fluency. `vi`/`vim` is present on essentially every Linux system by
default (often the only editor guaranteed available on a minimal server),
and modal editing's real payoff is speed once it's internalized: composing
motions and operators lets you express an edit as a short command instead
of manual cursor navigation and selection. This demo covers the model,
motions, registers, and macros — enough to be genuinely fast, not just
able to save and quit.

## Demo Objectives

1. Explain the modal editing model — Normal, Insert, Visual — and why it exists
2. Navigate efficiently with motions, without arrow keys or a mouse
3. Compose operators with motions to edit in a single command
4. Use registers to cut, copy, and paste with precision
5. Record and replay a macro to automate a repetitive edit
6. Explain when Vim/Neovim is the right tool versus reaching for `sed`/`awk` instead

## Directory Structure

```
23-vim-neovim/
└── README.md   # This file — full demo content, including Anki (Appendix A)
                 # and Quiz (Appendix B), extracted later via script.
```

## Environments

| Task | Ubuntu 24.04 | Rocky Linux 9.8 | EC2 Amazon Linux |
|---|---|---|---|
| All lab tasks | ✅ (`vim` preinstalled or `sudo apt install vim`) | ✅ (`vi`/`vim` preinstalled or `sudo dnf install vim`) | Not required |

## Recall Check — 22-diff

Answer these from memory before continuing — no looking back at Demo 22.

**Q1.** In a unified diff hunk header `@@ -1,3 +1,4 @@`, what do the two
number pairs mean?

**Q2.** What does `diff3`'s `====` conflict marker actually indicate?

**Q3.** A patch that applied cleanly last week fails today on the same
target file, with the patch file itself unchanged. What's the most likely
cause?

<details>
<summary>Answers</summary>

**A1.** `-1,3`: the first file's shown section starts at line 1, spans 3
lines. `+1,4`: the second file's shown section starts at line 1, spans 4
lines.

**A2.** A genuine three-way conflict — both compared versions changed the
same content differently from the common ancestor, which `diff3` can't
resolve automatically.

**A3.** The target file has drifted — an unrelated change since then means
the patch's context lines no longer match the file's current content at
the expected location.

</details>

## Part 1 — Concepts

### 1.1 The modal model — why Vim has modes at all

Most editors have one mode: you type, and characters appear. Vim
deliberately separates **editing text** from **commanding the editor**
into different modes, so the same keys mean different things depending on
which mode you're in:

- **Normal mode** (the default, and where you spend most of your time) —
  keys are commands: navigation, deletion, operators. Nothing you type
  here inserts text.
- **Insert mode** — keys type text, exactly like a normal editor. Entered
  with `i`, `a`, `o`, etc.; exited back to Normal with `Esc`.
- **Visual mode** — select text (character-, line-, or block-wise), then
  apply an operator to the selection.

```bash
vim /tmp/vim-lab.txt
```

```
i          # enter Insert mode
Hello      # types "Hello" as text
<Esc>      # back to Normal mode — h/j/k/l now navigate, not type
```

**The entire point of this separation is that Normal-mode keys can be
short, single-character commands without colliding with typing text** —
`d` can mean "delete" specifically because Normal mode never needs `d` to
type the letter d. This is what makes Vim's command vocabulary so dense
and, once learned, so fast.

### 1.2 Motions — moving with intent, not just direction

```
h j k l    # left, down, up, right — the original arrow-key replacement,
           # kept on the home row so your hands never leave it
w          # forward to the start of the next word
b          # backward to the start of the previous word
e          # forward to the end of the current/next word
0          # start of the line
$          # end of the line
gg         # start of the file
G          # end of the file
5G         # go to line 5, specifically
/pattern   # search forward for a pattern (uses regex — Demo 17's
           # knowledge applies directly here)
n          # repeat the last search, forward
```

**Motions aren't just navigation — they're arguments to operators**, which
is where Vim's real efficiency comes from (§1.3). Knowing `w` moves to the
next word matters on its own, but its real power shows up combined with an
operator like `d` (delete).

### 1.3 Operators + motions — composing an edit in one command

```
dw    # delete (d) + word (w) = delete to the start of the next word
d$    # delete (d) + end-of-line ($) = delete from cursor to end of line
dd    # delete (d) + itself (d) = delete the whole current line, a common
      # special-case shorthand for "delete this motion's natural unit"
3dd   # delete 3 lines starting from the current one — a count prefix
      # multiplies the operation
yy    # yank (copy) the current line
y$    # yank from cursor to end of line
cw    # change (delete + enter Insert mode) a word — a single command
      # that replaces "select word, delete it, start typing" entirely
```

**This composability — `{operator}{count}{motion}` — is the actual payoff
of the modal model.** `d3w` (delete 3 words) and `3dw` (repeat "delete a
word" 3 times) both work and produce the same result, because Vim's grammar
treats operator+count+motion as a flexible, composable unit rather than a
fixed set of memorized shortcuts — once the grammar clicks, you can express
edits you've never specifically practiced, just by combining known pieces.

### 1.4 Registers — precise cut, copy, paste

```
"add       # delete into register a (explicit register instead of the
           # default unnamed one)
"ayy       # yank the current line into register a
"ap        # paste from register a
```

Vim has multiple registers simultaneously — the default unnamed register
(what plain `dd`/`yy`/`p` use), and 26 named registers (`a` through `z`)
you can target explicitly. **This matters because a plain `dd` overwrites
whatever was previously yanked/deleted** — if you copy something with `yy`,
then delete a different line with `dd` before pasting, the delete
overwrites the unnamed register and your original yank is gone from `p`'s
default target. Using explicit named registers avoids this collision when
you need to hold onto more than one piece of text at a time:

```
"ayy       # yank line 1 into register a
j"byy      # move down, yank line 2 into register b — a is untouched
"ap        # paste register a's content
"bp        # paste register b's content, independently
```

```
:reg       # show the current contents of every register — useful for
           # confirming what's actually stored before pasting, rather
           # than trusting memory
```

### 1.5 Macros — recording and replaying a repetitive edit

```
qa           # start recording into register a
0i# <Esc>    # (example edit: go to start of line, insert "# ", back to Normal)
j            # move to next line
q            # stop recording
```

```
5@a          # replay the macro stored in register a, 5 times
@@           # replay the LAST macro used, again — a shortcut once you're
             # already mid-repetition and want to keep going without
             # retyping the register name
```

**A macro is literally just a recorded sequence of Normal-mode
keystrokes**, stored in a register the same way yanked text is — this is
why macros and registers share the same naming (`qa`...`q` records into
register `a`, same letter used by `"ap` for pasted text). The practical use
case: any edit that needs to be repeated identically across many lines
(commenting out a block, reformatting a repeated structure) is often faster
recorded once and replayed N times than done manually N times, or than
writing a one-off `sed`/`awk` command for a genuinely one-time,
non-reusable edit.

### 1.6 When Vim is the right tool vs. sed/awk

Vim excels at **interactive, exploratory editing** — when you're not sure
exactly what needs to change until you're looking at the file, or the edit
needs human judgment line by line. `sed`/`awk` excel at **batch,
reproducible transformation** — the same well-defined change applied
consistently, especially across many files or as part of a script that
needs to run unattended, without a human present to make judgment calls.

A macro *can* express something `sed` would also express — but if the
transformation is genuinely mechanical and well-defined (not needing a
human to look at each line), a `sed`/`awk` one-liner is usually more
maintainable and rerunnable than a recorded macro, which lives only in that
one editing session's memory unless deliberately saved. Reach for Vim when
editing interactively at a terminal; reach for `sed`/`awk` when the edit
needs to be scripted, repeated across many files, or run without a human
present.

## Part 2 — Hands-On Lab

### Commands used in this demo

| Keys | Purpose |
|---|---|
| `i` / `a` / `Esc` | Enter Insert mode / append / return to Normal |
| `w` / `b` / `e` / `0` / `$` | Word-forward, word-back, word-end, line-start, line-end motions |
| `gg` / `G` / `NG` | File-start, file-end, go to line N |
| `dw` / `dd` / `yy` / `cw` | Operator+motion composition |
| `"aX` | Target register `a` explicitly |
| `qa ... q` | Record a macro into register `a` |
| `N@a` / `@@` | Replay a macro N times / replay the last macro used |

### Task 1 — Modes and basic motions

```bash
printf 'first line\nsecond line\nthird line\nfourth line\nfifth line\n' > /tmp/vim-lab.txt
vim /tmp/vim-lab.txt
```

In Vim: practice `w`/`b`/`e` across a line, `0`/`$` to jump to line
boundaries, `gg`/`G` to jump to file start/end, `5G` to jump to a specific
line number. Confirm you never left Normal mode except to briefly enter
Insert with `i` and return with `Esc`.

### Task 2 — Operators + motions

In the same file: `dw` on a word, `dd` on a whole line, `yy` then `p` to
duplicate a line, `cw` to replace a word in one motion. Try `3dd` to delete
three lines at once, and `d3w` vs `3dw` to confirm both delete 3 words.

### Task 3 — Registers, avoiding the default-overwrite trap

In the same file: `"ayy` on line 1 to yank into register `a`, move down,
`"byy` on a different line into register `b`, then `dd` on a third line
(overwriting the *unnamed* register only). Confirm `"ap` and `"bp` still
correctly paste their respective, untouched content — demonstrating that
explicit registers survived the `dd` that would have clobbered the default
unnamed register.

### Task 4 — Record and replay a macro

```bash
printf 'line one\nline two\nline three\nline four\n' > /tmp/vim-lab2.txt
vim /tmp/vim-lab2.txt
```

Record: `qa` to start, `I# ` `<Esc>` (insert "# " at the start of the
line), `j` to move down, `q` to stop. Then `3@a` to replay it on the
remaining three lines — confirm all four lines now start with `# `.

### Task 5 — Search-based navigation with regex

```bash
printf 'apple\nbanana\ncherry\napricot\n' > /tmp/vim-lab3.txt
vim /tmp/vim-lab3.txt
```

Use `/^a` to search for lines starting with "a" (Demo 17's anchor
knowledge applies directly), then `n` to jump to the next match.

### Task 6 — Recognize the Vim-vs-sed boundary

```bash
printf 'DEBUG: msg1\nDEBUG: msg2\nDEBUG: msg3\n' > /tmp/vim-lab4.txt
```

Mentally compare two approaches to replacing every `DEBUG:` with `INFO:`
in this file: recording a Vim macro and replaying it 3 times, versus `sed
-i 's/DEBUG:/INFO:/g' /tmp/vim-lab4.txt`. For this specific, fully
mechanical, well-defined change, which approach is actually more
appropriate per §1.6, and why?

## What You Learned

1. ✅ Explained the modal editing model — Normal, Insert, Visual — and why it exists
2. ✅ Navigated efficiently with motions, no arrow keys or mouse needed
3. ✅ Composed operators with motions to edit in a single command
4. ✅ Used registers to cut, copy, and paste with precision, avoiding the default-overwrite trap
5. ✅ Recorded and replayed a macro to automate a repetitive edit
6. ✅ Recognized when Vim is the right tool versus `sed`/`awk`

## Part 3 — Break-Fix Scenario

### Scenario — NovaTech Ops: two Vim incidents

> **Incident report — NovaTech Ops, ticket #4712**
>
> Alert 1: "an engineer yanked a config block with `yy`, then accidentally
> deleted an unrelated line with `dd` before pasting — the yanked content
> is gone." Alert 2: "a macro recorded to fix 50 lines only fixed the first
> 12 before silently stopping."

**Incident 1 — yanked content lost after an unrelated dd**

```
"yy on line 5 (config block start)
... navigates elsewhere ...
dd on line 40 (unrelated cleanup)
p — pastes line 40's content, NOT the originally yanked line 5
```

Tasks:
1. Reference §1.4 — why did `dd` overwrite what `yy` had stored, when they
   were used on completely different, unrelated lines?
2. What should the engineer have done differently from the start to avoid
   this exact loss?
3. Is the originally yanked content recoverable at this point, using
   anything covered in this demo?

**Incident 2 — macro stops working partway through**

```
qa                    (start recording)
0                     (go to start of line)
d$                    (delete to end of line — removing a trailing comment)
j                     (move down)
q                     (stop recording)
12@a                  (replay 12 more times — works for lines with a
                        trailing comment, then starts failing/doing
                        nothing useful on lines that DON'T have one)
```

Tasks:
1. Reference §1.3 — what does `d$` actually do on a line, and what happens
   when that same macro runs on a line where the intended trailing content
   doesn't exist?
2. Is this a Vim bug, or a mismatch between what the macro assumes about
   every line and what's actually true of all 50 lines?
3. What would need to be true about the target lines for a blind `N@a`
   replay to be safe, and what should you check before trusting one across
   many lines?

<details>
<summary>Answers</summary>

**Incident 1:** `dd` and `yy` (used without an explicit register) both
target the same default *unnamed* register — whichever operation happened
most recently overwrites whatever the unnamed register held before,
regardless of whether the two operations touched related or completely
unrelated lines; the register has no memory of "context," only the most
recent content. The engineer should have used an explicit named register
from the start — `"ayy` on line 5, then the later `dd` on line 40 would
only affect the unnamed register, leaving register `a`'s content
completely untouched and available via `"ap` at any point afterward. At
this point, with no explicit register used, the original content is not
recoverable through anything covered in this demo — Vim's undo history
(`u`) could potentially restore file state if the deletion is undone in
sequence, but the yanked-then-lost register content itself has no separate
recovery path once overwritten.

**Incident 2:** `d$` deletes from the cursor position to the end of the
current line — on a line that actually has trailing content (the intended
comment), this correctly removes it. On a line with no trailing content
after the cursor's starting position (e.g., a shorter line, or one already
lacking the comment), `d$` either deletes nothing meaningful or deletes
content that was never meant to be removed, depending on where the cursor
lands — either way, silently producing a different, likely wrong result
without any error, since `d$` is a perfectly valid operation regardless of
what's actually on the line. This is not a Vim bug — the macro is
replaying the exact recorded keystrokes faithfully every time; the actual
problem is the macro's implicit assumption that every one of the 50 lines
has the same shape (a trailing comment to remove), which doesn't hold for
lines 13 onward. For a blind `N@a` replay to be safe, every targeted line
needs to genuinely share the same structure the macro assumes — the
correct habit before trusting a large replay count is testing the macro on
a handful of representative lines first (including any that look
different), or reviewing the target range for uniformity, rather than
assuming 50 lines are all shaped identically just because the first few
were.

</details>

## Interview Prep

**Q1. Why does Vim separate Normal and Insert mode instead of just letting you type and use shortcut keys simultaneously, the way most editors do?**
Separating modes means Normal-mode keys can be short, single-character
commands (`d`, `y`, `w`) without ever colliding with typing actual text,
since Normal mode never needs those keys to insert characters. This is
what enables Vim's dense, composable command vocabulary
(`{operator}{count}{motion}`) — a design that wouldn't be possible if every
key had to simultaneously handle "type this character" and "this could
also be a command."

**Q2. An engineer yanks text with `yy`, then does an unrelated `dd` elsewhere, and loses the originally yanked content. What went wrong, and how is it avoided?**
Plain `yy`/`dd` without an explicit register both target the same default
unnamed register — the most recent operation always overwrites it,
regardless of whether the operations are related. Using an explicit named
register (`"ayy`, then `"ap` later) avoids this collision entirely, since
a plain `dd` elsewhere only affects the unnamed register, leaving the
named one untouched.

**Q3. A macro that worked on the first several lines of a file starts producing wrong results partway through a large replay. What's the most likely cause?**
The macro faithfully replays the exact recorded keystrokes every time —
if the target lines aren't actually uniform in structure (some lack
content the macro assumes is there), the same keystrokes produce a
different, often wrong result on lines that don't match the assumption the
macro was recorded against. This isn't a Vim malfunction; it's a mismatch
between what the macro assumes about every line and what's actually true
across the full range — testing on a representative sample before a large
blind replay is the way to catch this early.

**Q4. When would recording a Vim macro be a worse choice than a `sed`/`awk` one-liner for the same repetitive edit?**
When the change is fully mechanical, well-defined, and doesn't need human
judgment per line — a `sed`/`awk` command expresses that transformation
reproducibly, can be rerun identically later or across many files, and
exists as a reviewable, scriptable artifact. A macro lives only in that
one editing session's memory unless deliberately saved, and is the better
choice specifically when editing is interactive/exploratory or needs
line-by-line human judgment that a fixed pattern-based tool can't express.

## Key Takeaways

1. **Modal editing exists to let Normal-mode keys be short commands without
   colliding with typing text.** `d`, `y`, `w` mean something specific in
   Normal mode precisely because Normal mode never needs them to insert
   characters — this is the foundation the rest of Vim's efficiency builds on.
2. **Operators compose with motions — `{operator}{count}{motion}` is a
   grammar, not a fixed list of shortcuts.** `d3w` and `3dw` both work and
   mean the same thing; once the grammar clicks, you can express edits
   you've never specifically practiced.
3. **Plain `dd`/`yy`/`p` all share one default unnamed register — the most
   recent operation always overwrites it, regardless of context.** Use
   explicit named registers (`"ayy`, `"ap`) whenever you need to hold onto
   more than one piece of text at a time.
4. **A macro faithfully replays exact keystrokes — it has no awareness of
   whether the target line actually matches what the macro assumes.**
   Testing on a representative sample before a large blind replay catches
   non-uniform target lines before they produce silently wrong results.
5. **Vim suits interactive, exploratory editing needing human judgment;
   `sed`/`awk` suit fully mechanical, reproducible, scriptable
   transformation.** A macro can technically express what `sed` would, but
   isn't the more maintainable choice once the edit is genuinely
   well-defined and repeatable without a human present.
6. **`vi`/`vim` is close to universally available on Linux systems by
   default**, which is the practical reason this fluency matters even in a
   world with more full-featured editors — it's frequently the only editor
   guaranteed present on a minimal server you're SSH'd into.

> **Demo scope:** Primary concept: modal editing fluency for fast,
> keyboard-only text editing. Supporting concepts: Normal/Insert/Visual
> modes, motions, operator+motion composition, registers, macros.
> Estimated completion time: 30 minutes (reading + hands-on + verification).
> Checkpoints: 6 natural stopping points (after Task 1 through Task 6).

## Quick Commands Reference

### Modes

| Key | Effect |
|---|---|
| `i` / `a` | Enter Insert mode (before/after cursor) |
| `Esc` | Return to Normal mode |
| `v` / `V` | Enter Visual / Visual Line mode |

### Motions

| Key | Effect |
|---|---|
| `h j k l` | Left, down, up, right |
| `w` / `b` / `e` | Next word start / previous word start / word end |
| `0` / `$` | Line start / line end |
| `gg` / `G` / `NG` | File start / file end / line N |
| `/pattern` / `n` | Search forward / repeat search |

### Operators (combine with a motion)

| Key | Effect |
|---|---|
| `d` | Delete |
| `y` | Yank (copy) |
| `c` | Change (delete + enter Insert) |
| `dd` / `yy` | Operate on the whole current line |

### Registers and macros

| Key | Effect |
|---|---|
| `"aX` | Target register `a` for operation `X` |
| `:reg` | Show contents of all registers |
| `qa ... q` | Record a macro into register `a` |
| `N@a` / `@@` | Replay macro `a` N times / replay last macro used |

## Appendix A — Anki Cards

**Note:** extracted into a standalone `.csv` by the series extraction script.

```
#deck:Linux Mastery::Module 4 - Text Processing & Scripting for DevOps::23-vim-neovim
#separator:Comma
#columns:Front,Back,Tags
"Why does Vim separate Normal mode and Insert mode instead of one unified typing mode?","So Normal-mode keys can be short, single-character commands (d, y, w) without ever colliding with typing text — Normal mode never needs those keys to insert characters, enabling a dense composable command vocabulary.","demo23,vim,modal-editing,fundamentals"
"What does d3w do, and how does it compare to 3dw?","Both delete 3 words forward and produce the same result — Vim's grammar treats {operator}{count}{motion} as flexible and composable regardless of where the count is placed relative to the operator.","demo23,vim,operators,motions"
"An engineer uses yy on one line, then later dd on a completely unrelated line, and loses the yanked content. Why?","Plain yy/dd without an explicit register both target the same default unnamed register — the most recent operation always overwrites it, regardless of whether the two operations are related or on unrelated lines.","demo23,vim,registers,common-mistake"
"How do you avoid the default-register-overwrite problem when you need to hold onto more than one piece of yanked/deleted text?","Use explicit named registers — e.g. \"ayy to yank into register a, \"ap to paste from it — a later plain dd/yy elsewhere only affects the unnamed register, leaving named registers untouched.","demo23,vim,registers,fix"
"What is a Vim macro, mechanically?","A recorded sequence of Normal-mode keystrokes, stored in a register the same way yanked text is (qa...q records into register a) — replayed with N@a.","demo23,vim,macros,fundamentals"
"A macro that worked on the first 12 lines of a 50-line replay starts producing wrong results afterward. Is this a Vim bug?","No — the macro faithfully replays the exact recorded keystrokes every time. The problem is the macro's implicit assumption that every targeted line has the same structure, which breaks down once a line doesn't match that assumption.","demo23,vim,macros,troubleshooting"
"What should you do before trusting a macro across a large number of lines (e.g. N@a with a high N)?","Test it on a representative sample of the target lines first, including any that look structurally different, rather than assuming uniformity across all of them just because the first few worked.","demo23,vim,macros,best-practice"
"When is a Vim macro a worse choice than a sed/awk one-liner for the same repetitive edit?","When the change is fully mechanical, well-defined, and doesn't need human judgment per line — sed/awk expresses it reproducibly and rerunnably, while a macro lives only in that editing session's memory unless deliberately saved.","demo23,vim,sed,awk,tool-boundary"
"What does the search command /^a followed by n actually do in Vim?","/^a searches forward for lines starting with 'a' (using the same regex anchor knowledge from Demo 17), landing the cursor on the first match; n repeats that search, jumping to the next match.","demo23,vim,search,cross-demo"
"Why is Vim/vi fluency practically valuable even given more full-featured editors exist?","vi/vim is close to universally available on Linux systems by default, and is frequently the only editor guaranteed present on a minimal server accessed over SSH.","demo23,vim,practical-value"
```

## Appendix B — Quiz

**Note:** extracted into a standalone `.md` by the series extraction script.

> One correct answer per question unless stated otherwise.
> Target: 80% or above before moving to Demo 24.

**Q1. Why does Vim separate Normal mode from Insert mode?**

- A) It's a historical accident with no real design benefit today
- B) So Normal-mode keys can be short, single-character commands without ever colliding with typing text
- C) Insert mode is only needed for non-English keyboards
- D) Normal mode is purely for viewing files, not editing at all

<details>
<summary>Answer</summary>

**B** — This separation is the foundation of Vim's composable command vocabulary. Trap: A dismisses genuine design intent as accidental; C invents an unrelated keyboard-layout justification; D understates Normal mode's actual editing role (deletion, operators, etc. all happen there).

</details>

---

**Q2. `d3w` and `3dw` both delete 3 words forward. Why do both work identically?**

- A) Vim silently ignores the count in whichever position is "wrong"
- B) Vim's grammar treats `{operator}{count}{motion}` as a flexible, composable unit regardless of count placement
- C) This is a coincidence specific to the `w` motion only
- D) Only one of the two actually works; the other is a common misconception

<details>
<summary>Answer</summary>

**B** — This composability is the actual design principle, not a coincidence or a silent correction. Trap: A invents error-correcting behavior that isn't happening; C wrongly narrows this to one specific motion; D is factually wrong — both genuinely work.

</details>

---

**Q3. `yy` on one line, then later `dd` on a completely unrelated line, results in the originally yanked content being lost. Why?**

- A) `dd` always clears every register as a side effect
- B) Plain `yy`/`dd` without an explicit register both target the same default unnamed register, and the most recent operation overwrites it
- C) Vim only remembers the last 5 seconds of register history
- D) This only happens if the two lines are in the same file

<details>
<summary>Answer</summary>

**B** — This shared default-register behavior, regardless of context or relatedness between operations, is exactly the mechanism. Trap: A overstates the effect to "every register"; C invents a time-based limit; D invents an irrelevant same-file condition.

</details>

---

**Q4. How do you avoid the default-register-overwrite problem when holding onto more than one piece of text?**

- A) Always use `dd` instead of `yy` for everything
- B) Use explicit named registers, e.g. `"ayy` and `"ap`
- C) Save the file after every yank
- D) This cannot be avoided — Vim only supports one register total

<details>
<summary>Answer</summary>

**B** — Named registers (`a` through `z`) exist specifically to hold content independently of the default unnamed register. Trap: A doesn't address the actual problem; C is unrelated to register behavior; D is factually wrong — Vim has 26 named registers plus the unnamed one.

</details>

---

**Q5. Mechanically, what is a Vim macro?**

- A) A separate configuration file defining custom keybindings
- B) A recorded sequence of Normal-mode keystrokes, stored in a register
- C) A built-in Vim plugin for text formatting
- D) An external script Vim calls out to

<details>
<summary>Answer</summary>

**B** — `qa...q` records exactly this into register `a`, replayed with `N@a`. Trap: A, C, and D all invent unrelated mechanisms that aren't what a macro actually is.

</details>

---

**Q6. A macro works correctly on the first several lines of a large replay, then starts producing wrong results. Is this a Vim bug?**

- A) Yes — macros should detect when a line doesn't match and skip it
- B) No — the macro faithfully replays exact keystrokes; the problem is an assumption about line structure that stopped holding true
- C) Yes, but only for macros longer than 10 recorded keystrokes
- D) No, but only if the macro was recorded in Insert mode

<details>
<summary>Answer</summary>

**B** — This is a mismatch between the macro's implicit assumption and reality, not a malfunction — Vim has no way to know the macro's "intent" beyond the literal keystrokes. Trap: A assumes intelligent behavior Vim macros don't have; C and D both invent irrelevant conditions.

</details>

---

**Q7. What should you do before trusting a macro across a large replay count on many lines?**

- A) Nothing — macros are always safe once recorded correctly
- B) Test it on a representative sample of target lines first, including any that look structurally different
- C) Increase the replay count gradually by 1 each time
- D) Record the macro twice for redundancy

<details>
<summary>Answer</summary>

**B** — This is the practical habit that catches non-uniform target lines before a large blind replay produces silently wrong results. Trap: A ignores the real risk demonstrated in this demo's incident; C and D both invent ineffective workarounds.

</details>

---

**Q8. When is a `sed`/`awk` one-liner a better choice than a Vim macro for a repetitive edit?**

- A) Never — Vim macros are always faster
- B) When the change is fully mechanical, well-defined, and needs to be reproducible or rerun without human judgment
- C) Only when editing files larger than 1GB
- D) Only when working over SSH, never locally

<details>
<summary>Answer</summary>

**B** — This is the actual distinguishing factor: mechanical, reproducible, scriptable transformation favors `sed`/`awk`; interactive, judgment-requiring editing favors Vim. Trap: A overstates Vim's universal superiority; C and D both invent irrelevant constraints.

</details>