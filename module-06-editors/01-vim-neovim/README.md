# Demo M06-01 — Vim/Neovim — the DevOps/Cloud Editor

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
4. Select text in Visual mode and apply an operator to the selection
5. Use registers to cut, copy, and paste with precision, including the
   dedicated yank register that survives an unrelated delete
6. Record and replay a macro to automate a repetitive edit
7. Explain when Vim/Neovim is the right tool versus reaching for `sed`/`awk` instead

## Directory Structure

```
01-vim-neovim/
├── README.md                   # this file
├── m06-01-vim-neovim-anki.csv  # Anki flashcard deck
└── m06-01-vim-neovim-quiz.md   # standalone quiz
```

## Environments

| Task | Ubuntu 24.04 | Rocky Linux 9.8 | EC2 Amazon Linux |
|---|---|---|---|
| All lab tasks | ✅ (`vim` preinstalled or `sudo apt install vim`) | ✅ (`vi`/`vim` preinstalled or `sudo dnf install vim`) | Not required |

> **Verification note:** This demo's Vim/Neovim behavior claims are built
> from established, stable tool knowledge — this sandbox has no network
> access to install Vim and capture a live session, the same constraint
> already flagged in `06-cron-log-rotation`. Recommend a real
> verification pass on an actual Ubuntu/Rocky9 machine before treating
> register and macro edge-case behavior (§1.5, §1.6) as fully confirmed.

## Recall Check — 10-diff-comparison

Answer these from memory before continuing — no looking back at Demo M05-10.

1. In a unified diff hunk header `@@ -1,3 +1,4 @@`, what do the two
   number pairs mean?

2. What does `diff3`'s `====` conflict marker actually indicate?

3. A patch that applied cleanly last week fails today on the same
   target file, with the patch file itself unchanged. What's the most likely
   cause?

<details>
<summary>Answers</summary>

1. `-1,3`: the first file's shown section starts at line 1, spans 3
   lines. `+1,4`: the second file's shown section starts at line 1, spans 4
   lines.

2. A genuine three-way conflict — both compared versions changed the
   same content differently from the common ancestor, which `diff3` can't
   resolve automatically.

3. The target file has drifted — an unrelated change since then means
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
  with `i` (insert before the cursor), `a` (append after the cursor), `o`
  (open a new line below), or their capitalized counterparts `I` (insert
  at the first non-blank character of the line, regardless of where the
  cursor currently sits) and `A` (append at the end of the line); exited
  back to Normal with `Esc`.
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

**Undoing a change:** Normal mode also has `u` (undo the last change) and
`Ctrl-r` (redo — reverse an undo). These aren't part of the modal-editing
model itself, but you'll use `u` constantly while practicing throughout
this demo — it's how Task 3 lets you try two different edits against the
same starting text without retyping it each time.

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
/pattern   # search forward for a pattern (uses regex — Demo M05-01's
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
y$    # yank from cursor to end of line (not the whole line — if the
      # cursor isn't at column 0, y$ only captures what's after it)
cw    # change (delete + enter Insert mode) a word — a single command
      # that replaces "select word, delete it, start typing" entirely
```

**This composability — `{operator}{count}{motion}` — is the actual payoff
of the modal model.** `d3w` (delete 3 words) and `3dw` (repeat "delete a
word" 3 times) both work and produce the same result, because Vim's grammar
treats operator+count+motion as a flexible, composable unit rather than a
fixed set of memorized shortcuts — once the grammar clicks, you can express
edits you've never specifically practiced, just by combining known pieces.

**The doubled-operator shorthand (`dd`, `yy`) isn't unique to `d` and
`y` — it generalizes across most operators**: `cc` changes the whole
current line, `>>` indents the whole current line, and so on. Learning the
pattern once ("operator doubled = apply to the current line as a whole")
effectively teaches you the shorthand for every operator that supports it,
not just the two shown here.

### 1.4 Visual mode — select, then operate

Where Normal-mode operators pair with a *motion* to define what to act on,
Visual mode lets you *see* the selection directly before choosing an
operator — useful when the extent of what you want to change is easier to
eyeball than to express as a precise motion:

```
v      # enter character-wise Visual mode — extend the selection with any
       # motion (w, $, j, etc.), then apply an operator (d, y, c, ...) to
       # exactly what's highlighted
V      # enter Visual LINE mode — selects whole lines as you move up/down,
       # regardless of where the cursor sits within each line
<C-v>  # Visual BLOCK mode — select a rectangular block of text spanning
       # multiple lines, useful for editing a column of text (e.g. adding
       # the same prefix to several lines at fixed positions) rather than
       # whole lines or a linear character span
```

```
V3jd   # V: start Visual Line mode; 3j: extend the selection down 3 more
       # lines (4 total); d: delete the entire selected block — the same
       # operator-after-selection composability as Normal mode, just with
       # the extent chosen visually instead of via a single motion
```

Visual mode and Normal-mode operator+motion aren't competing techniques —
they're two ways of specifying the same underlying thing ("apply this
operator to this range of text"), and which one is faster depends on
whether the range is easier to express as a motion (`d$`, `d3w`) or as
something you'd rather see highlighted before committing to the edit.

### 1.5 Registers — precise cut, copy, paste

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

**A specific register worth knowing about even without any extra effort at
yank time: `"0`, the dedicated yank register.** Unlike the unnamed
register, `"0` is updated *only* by yank commands — never by delete or
change. This means `"0p` always pastes the most recent genuine yank, even
if a `dd` happened afterward and overwrote the unnamed register in the
meantime. It's a simpler safety net than remembering to use an explicit
named register every time: you don't have to decide in advance that a
yank is worth protecting, because `"0` protects every yank automatically.

`p` pastes after the cursor (or below the current line, for line-wise
content); `P` (capital) pastes before the cursor (or above the current
line) — the same lowercase/uppercase position distinction that shows up
elsewhere in Vim (like `i`/`I` for where Insert mode begins).

```
:reg       # show the current contents of every register — useful for
           # confirming what's actually stored before pasting, rather
           # than trusting memory
```

### 1.6 Macros — recording and replaying a repetitive edit

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

### 1.7 When Vim is the right tool vs. sed/awk

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
| `i` / `a` / `I` / `A` / `Esc` | Enter Insert mode / append / insert at line start / append at line end / return to Normal |
| `u` / `Ctrl-r` | Undo the last change / redo |
| `w` / `b` / `e` / `0` / `$` | Word-forward, word-back, word-end, line-start, line-end motions |
| `gg` / `G` / `NG` | File-start, file-end, go to line N |
| `dw` / `dd` / `yy` / `cw` | Operator+motion composition |
| `v` / `V` / `<C-v>` | Character-wise / line-wise / block-wise Visual mode |
| `"aX` | Target register `a` explicitly |
| `"0p` / `P` | Paste the dedicated yank register / paste before cursor |
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
Try `cc` on a line to confirm the doubled-operator shorthand generalizes
beyond `dd`/`yy`.

### Task 3 — Visual mode: select, then operate

```bash
printf 'keep this\nDELETE this line\nkeep this too\nand this one\n' > /tmp/vim-lab5.txt
vim /tmp/vim-lab5.txt
```

Navigate to the `DELETE` line: `V` (Visual Line mode) to select the whole
line, then `d` to delete the selection. Undo (`u`), then try `v` on the
same line (character-wise Visual), extend the selection over just the word
`DELETE` with `w` or `e`, and `d` again — confirm this time only the word
is removed, not the whole line, contrasting Visual Line's whole-line
granularity against Visual character-wise's precise span.

### Task 4 — Registers: the default trap and the "0 safety net

```bash
printf 'config line A\nconfig line B\nunrelated line C\n' > /tmp/vim-lab6.txt
vim /tmp/vim-lab6.txt
```

`"ayy` on line 1 to yank into register `a`, move down, `"byy` on line 2
into register `b`, then plain `yy` (no register) on line 1 again followed
by `dd` on line 3 (overwriting the *unnamed* register with the delete).
Confirm `"ap` and `"bp` still correctly paste their respective, untouched
content. Then confirm `"0p` pastes the plain `yy` from line 1 — even
though the `dd` that came after it overwrote the unnamed register, `"0`
was never touched by that delete.

### Task 5 — Record and replay a macro

```bash
printf 'line one\nline two\nline three\nline four\n' > /tmp/vim-lab2.txt
vim /tmp/vim-lab2.txt
```

Record: `qa` to start, `I# ` `<Esc>` (insert "# " at the start of the
line), `j` to move down, `q` to stop. Then `3@a` to replay it on the
remaining three lines — confirm all four lines now start with `# `.

### Task 6 — Search-based navigation with regex

```bash
printf 'apple\nbanana\ncherry\napricot\n' > /tmp/vim-lab3.txt
vim /tmp/vim-lab3.txt
```

Use `/^a` to search for lines starting with "a" (Demo M05-01's anchor
knowledge applies directly), then `n` to jump to the next match.

### Task 7 — Recognize the Vim-vs-sed boundary

```bash
printf 'DEBUG: msg1\nDEBUG: msg2\nDEBUG: msg3\n' > /tmp/vim-lab4.txt
```

Mentally compare two approaches to replacing every `DEBUG:` with `INFO:`
in this file: recording a Vim macro and replaying it 3 times, versus `sed
-i 's/DEBUG:/INFO:/g' /tmp/vim-lab4.txt`. For this specific, fully
mechanical, well-defined change, which approach is actually more
appropriate per §1.7, and why?

## What You Learned

- ✅ Explained the modal editing model — Normal, Insert, Visual — and why it exists
- ✅ Navigated efficiently with motions, no arrow keys or mouse needed
- ✅ Composed operators with motions to edit in a single command,
  including the doubled-operator shorthand generalizing beyond `dd`/`yy`
- ✅ Selected text in Visual mode (character-, line-, and block-wise) and
  applied an operator to the selection
- ✅ Used registers to cut, copy, and paste with precision, including
  `"0`, the dedicated yank register that survives an unrelated delete
- ✅ Recorded and replayed a macro to automate a repetitive edit
- ✅ Recognized when Vim is the right tool versus `sed`/`awk`

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
1. Reference §1.5 — why did `dd` overwrite what `yy` had stored, when they
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
completely untouched and available via `"ap` at any point afterward.
However, per §1.5, the content actually *is* recoverable here even without
that foresight: `"0`, the dedicated yank register, is updated only by
yank commands, never by delete or change — so even though `dd` on line 40
overwrote the unnamed register, it never touched `"0`, which still holds
line 5's original content from the `yy`. `"0p` recovers it. This is
exactly why `"0` is worth knowing even if you never proactively reach for
named registers: it's an automatic safety net for every yank, requiring no
extra keystrokes at the moment you yank something.

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

---

**Q2. An engineer yanks text with `yy`, then does an unrelated `dd` elsewhere, and loses the originally yanked content. What went wrong, and how is it avoided?**

Plain `yy`/`dd` without an explicit register both target the same default
unnamed register — the most recent operation always overwrites it,
regardless of whether the operations are related. Using an explicit named
register (`"ayy`, then `"ap` later) avoids this collision entirely, since
a plain `dd` elsewhere only affects the unnamed register, leaving the
named one untouched. There's also a simpler, automatic safety net for
exactly this situation: `"0`, the dedicated yank register, is updated only
by yanks, never by deletes — so `"0p` would recover the lost content here
even without having deliberately used a named register in advance.

---

**Q3. A macro that worked on the first several lines of a file starts producing wrong results partway through a large replay. What's the most likely cause?**

The macro faithfully replays the exact recorded keystrokes every time —
if the target lines aren't actually uniform in structure (some lack
content the macro assumes is there), the same keystrokes produce a
different, often wrong result on lines that don't match the assumption the
macro was recorded against. This isn't a Vim malfunction; it's a mismatch
between what the macro assumes about every line and what's actually true
across the full range — testing on a representative sample before a large
blind replay is the way to catch this early.

---

**Q4. When would recording a Vim macro be a worse choice than a `sed`/`awk` one-liner for the same repetitive edit?**

When the change is fully mechanical, well-defined, and doesn't need human
judgment per line — a `sed`/`awk` command expresses that transformation
reproducibly, can be rerun identically later or across many files, and
exists as a reviewable, scriptable artifact. A macro lives only in that
one editing session's memory unless deliberately saved, and is the better
choice specifically when editing is interactive/exploratory or needs
line-by-line human judgment that a fixed pattern-based tool can't express.

---

## Key Takeaways

1. **Modal editing exists to let Normal-mode keys be short commands without
   colliding with typing text.** `d`, `y`, `w` mean something specific in
   Normal mode precisely because Normal mode never needs them to insert
   characters — this is the foundation the rest of Vim's efficiency builds on.
2. **Operators compose with motions — `{operator}{count}{motion}` is a
   grammar, not a fixed list of shortcuts.** `d3w` and `3dw` both work and
   mean the same thing; the doubled-operator shorthand (`dd`, `yy`, and
   also `cc`, `>>`, ...) generalizes across operators once you know the
   pattern once.
3. **Visual mode and Normal-mode operator+motion express the same idea two
   ways: "apply this operator to this range."** Use whichever makes the
   range easier to specify — a precise motion, or a selection you'd rather
   see highlighted first. `v`/`V`/`<C-v>` cover character-, line-, and
   block-wise selection respectively.
4. **Plain `dd`/`yy`/`p` all share one default unnamed register — the most
   recent operation always overwrites it, regardless of context.** Use
   explicit named registers (`"ayy`, `"ap`) whenever you need to hold onto
   more than one piece of text at a time — and remember `"0`, the
   dedicated yank register, protects every yank automatically, surviving
   any delete that comes after it with no extra effort required.
5. **A macro faithfully replays exact keystrokes — it has no awareness of
   whether the target line actually matches what the macro assumes.**
   Testing on a representative sample before a large blind replay catches
   non-uniform target lines before they produce silently wrong results.
6. **Vim suits interactive, exploratory editing needing human judgment;
   `sed`/`awk` suit fully mechanical, reproducible, scriptable
   transformation.** A macro can technically express what `sed` would, but
   isn't the more maintainable choice once the edit is genuinely
   well-defined and repeatable without a human present.
7. **`vi`/`vim` is close to universally available on Linux systems by
   default**, which is the practical reason this fluency matters even in a
   world with more full-featured editors — it's frequently the only editor
   guaranteed present on a minimal server you're SSH'd into.

> **Demo scope:** Primary concept: modal editing fluency for fast,
> keyboard-only text editing. Supporting concepts: Normal/Insert/Visual
> modes, motions, operator+motion composition, registers (including the
> dedicated yank register), macros.
> Estimated completion time: 35 minutes (reading + hands-on + verification).
> Checkpoints: 7 natural stopping points (after Task 1 through Task 7).

## Quick Commands Reference

### Modes

| Key | Effect |
|---|---|
| `i` / `a` | Enter Insert mode (before/after cursor) |
| `I` / `A` | Enter Insert mode (line start / line end, regardless of cursor position) |
| `Esc` | Return to Normal mode |
| `u` / `Ctrl-r` | Undo the last change / redo |
| `v` / `V` / `<C-v>` | Visual (character) / Visual Line / Visual Block mode |

### Motions

| Key | Effect |
|---|---|
| `h j k l` | Left, down, up, right |
| `w` / `b` / `e` | Next word start / previous word start / word end |
| `0` / `$` | Line start / line end |
| `gg` / `G` / `NG` | File start / file end / line N |
| `/pattern` / `n` | Search forward / repeat search |

### Operators (combine with a motion or a Visual selection)

| Key | Effect |
|---|---|
| `d` | Delete |
| `y` | Yank (copy) |
| `c` | Change (delete + enter Insert) |
| `dd` / `yy` / `cc` / `>>` | Operate on the whole current line (generalizes across operators) |

### Registers and macros

| Key | Effect |
|---|---|
| `"aX` | Target register `a` for operation `X` |
| `"0p` | Paste the dedicated yank register (survives an unrelated delete) |
| `p` / `P` | Paste after / before the cursor |
| `:reg` | Show contents of all registers |
| `qa ... q` | Record a macro into register `a` |
| `N@a` / `@@` | Replay macro `a` N times / replay last macro used |

## Appendix A — Anki Cards

**`m06-01-vim-neovim-anki.csv`:**

```
#deck:Linux Mastery::Module 6 - Editors::Demo M06-01 - Vim/Neovim
#separator:Comma
#columns:Front,Back,Tags
"Why does Vim separate Normal mode and Insert mode instead of one unified typing mode?","So Normal-mode keys can be short, single-character commands (d, y, w) without ever colliding with typing text — Normal mode never needs those keys to insert characters, enabling a dense composable command vocabulary.","demo-m06-01,vim,modal-editing,fundamentals"
"What does d3w do, and how does it compare to 3dw?","Both delete 3 words forward and produce the same result — Vim's grammar treats {operator}{count}{motion} as flexible and composable regardless of where the count is placed relative to the operator.","demo-m06-01,vim,operators,motions"
"An engineer uses yy on one line, then later dd on a completely unrelated line, and loses the yanked content. Why?","Plain yy/dd without an explicit register both target the same default unnamed register — the most recent operation always overwrites it, regardless of whether the two operations are related or on unrelated lines.","demo-m06-01,vim,registers,common-mistake"
"How do you avoid the default-register-overwrite problem when you need to hold onto more than one piece of yanked/deleted text?","Use explicit named registers — e.g. \"ayy to yank into register a, \"ap to paste from it — a later plain dd/yy elsewhere only affects the unnamed register, leaving named registers untouched.","demo-m06-01,vim,registers,fix"
"What is a Vim macro, mechanically?","A recorded sequence of Normal-mode keystrokes, stored in a register the same way yanked text is (qa...q records into register a) — replayed with N@a.","demo-m06-01,vim,macros,fundamentals"
"A macro that worked on the first 12 lines of a 50-line replay starts producing wrong results afterward. Is this a Vim bug?","No — the macro faithfully replays the exact recorded keystrokes every time. The problem is the macro's implicit assumption that every targeted line has the same structure, which breaks down once a line doesn't match that assumption.","demo-m06-01,vim,macros,troubleshooting"
"What should you do before trusting a macro across a large number of lines (e.g. N@a with a high N)?","Test it on a representative sample of the target lines first, including any that look structurally different, rather than assuming uniformity across all of them just because the first few worked.","demo-m06-01,vim,macros,best-practice"
"When is a Vim macro a worse choice than a sed/awk one-liner for the same repetitive edit?","When the change is fully mechanical, well-defined, and doesn't need human judgment per line — sed/awk expresses it reproducibly and rerunnably, while a macro lives only in that editing session's memory unless deliberately saved.","demo-m06-01,vim,sed,awk,tool-boundary"
"What does the search command /^a followed by n actually do in Vim?","/^a searches forward for lines starting with 'a' (using the same regex anchor knowledge from Demo M05-01), landing the cursor on the first match; n repeats that search, jumping to the next match.","demo-m06-01,vim,search,cross-demo"
"Why is Vim/vi fluency practically valuable even given more full-featured editors exist?","vi/vim is close to universally available on Linux systems by default, and is frequently the only editor guaranteed present on a minimal server accessed over SSH.","demo-m06-01,vim,practical-value"
```

## Appendix B — Quiz

**`m06-01-vim-neovim-quiz.md`:**

````markdown
# Quiz — Demo M06-01: Vim/Neovim — the DevOps/Cloud Editor

> One correct answer per question unless stated otherwise.
> Target: 80% or above before moving to Demo 24.

---

**Q1.** What's the difference between entering Visual mode with `v` versus `V`?

- A) They are identical; `V` is just an alias for `v`
- B) `v` starts character-wise Visual mode (precise span, any length); `V` starts Visual Line mode (whole lines, regardless of where the cursor sits within each one)
- C) `v` only works on the current line; `V` works across the whole file
- D) `V` requires a count prefix; `v` does not

<details>
<summary>Answer</summary>

**B** — Lowercase `v` selects a precise character-by-character span as you move; uppercase `V` selects whole lines as you move up/down, ignoring cursor column position within each line. Both then take an operator applied to whatever's highlighted.

Trap: A wrongly claims equivalence; C and D both invent scope/syntax restrictions that don't exist.

</details>

---

**Q2.** After yanking a line with plain `yy`, you later run `dd` on a completely different line elsewhere in the file. Does `"0p` still paste the original `yy` content?

- A) No — `"0` is also overwritten by `dd`, exactly like the unnamed register
- B) Yes — `"0` (the dedicated yank register) is updated only by yank commands, never by delete or change, so it still holds the original yanked content regardless of the intervening `dd`
- C) Only if the `dd` and `yy` were on adjacent lines
- D) `"0` doesn't exist; only named registers `a`–`z` can be targeted explicitly

<details>
<summary>Answer</summary>

**B** — `"0` is specifically insulated from delete/change operations, making it a safety net that works automatically for every yank, without needing to have deliberately chosen a named register in advance.

Trap: A wrongly conflates `"0` with the unnamed register's overwrite behavior; C invents an adjacency requirement that doesn't matter; D is factually wrong — `"0` is a real, documented register.

</details>

---

**Q3.** What does `:reg` show, and why would you run it before pasting something important?

- A) It shows the file's line count and word count
- B) It shows the current contents of every register, letting you confirm what's actually stored before trusting memory about which register holds what
- C) It permanently deletes all register contents, resetting them
- D) It only works while recording a macro

<details>
<summary>Answer</summary>

**B** — `:reg` is a read-only inspection command, useful exactly when you're not fully certain what a given register currently holds — confirming before pasting avoids pasting the wrong thing based on a mistaken assumption.

Trap: A invents unrelated file statistics; C wrongly claims a destructive effect; D invents a scope restriction that doesn't exist.

</details>

---

**Q4.** Why is `cw` (change word) considered a single composed command rather than just a convenient abbreviation for "delete a word, then enter Insert mode"?

- A) It isn't actually different from `dw` followed by `i` — the distinction is purely cosmetic
- B) `cw` is the operator+motion grammar (`{operator}{motion}`) applied directly — it expresses "change to the end of this word" as one command, the same composability principle behind every other operator+motion pairing, not a special case
- C) `cw` only works on the last word of a line
- D) `cw` requires Visual mode to function

<details>
<summary>Answer</summary>

**B** — `cw` is simply the `c` (change) operator combined with the `w` (word) motion — exactly the same grammar as `dw`, `d$`, `y$`, and every other operator+motion pair, not a special-cased shortcut outside that system.

Trap: A understates it as purely cosmetic when it's actually the same underlying composable grammar; C and D both invent restrictions that don't exist.

</details>

---

**Q5.** What's the difference between `i` and `I` (capital) for entering Insert mode?

- A) They are identical
- B) `i` enters Insert mode exactly at the cursor's current position; `I` enters Insert mode at the first non-blank character of the line, regardless of where the cursor currently sits
- C) `i` only works in Visual mode; `I` only works in Normal mode
- D) `I` inserts a literal capital "I" character before switching modes

<details>
<summary>Answer</summary>

**B** — This is the same lowercase/uppercase "position" distinction that shows up elsewhere in Vim (like `p`/`P` for paste position) — `I` jumps to the start of the actual content on the line before entering Insert, useful for prefixing a line regardless of cursor position.

Trap: A wrongly claims equivalence; C invents a mode restriction that doesn't exist; D misreads `I` as inserting a literal character rather than being a mode-entry command.

</details>

---

**Q6.** `y$` is run with the cursor positioned partway through a line (not at column 0). What does it yank?

- A) The entire line, from column 0 to the end, regardless of cursor position
- B) Only the text from the cursor's current position to the end of the line — not the whole line
- C) Nothing — `y$` requires the cursor to be at the start of the line
- D) The entire file, since `$` can also mean end-of-file in some contexts

<details>
<summary>Answer</summary>

**B** — `y$` combines the yank operator with the end-of-line motion starting from wherever the cursor currently is — it's explicitly not the same as `yy` (whole line), and yanks less than the full line whenever the cursor isn't already at column 0.

Trap: A confuses `y$` with `yy`'s whole-line behavior; C invents a positional requirement that doesn't exist; D misapplies `$`'s meaning — in this context it always means end-of-line, never end-of-file (that's `G`).

</details>

---

**Q7.** Does the doubled-operator shorthand pattern (`dd`, `yy`) apply only to `d` and `y`, or does it generalize?

- A) It applies only to `d` and `y` — no other operators support this shorthand
- B) It generalizes across most operators — `cc` changes the whole current line, `>>` indents the whole current line, following the same "operator doubled = apply to the current line" pattern
- C) It generalizes, but only for operators that don't require Insert mode afterward
- D) The shorthand is deprecated in modern Vim/Neovim in favor of explicit motions

<details>
<summary>Answer</summary>

**B** — Once you recognize "operator doubled = whole current line" as a general Vim grammar rule rather than two memorized special cases, it applies to `cc`, `>>`, and other operators the same way it applies to `dd`/`yy`.

Trap: A wrongly narrows the pattern to just the two examples most commonly shown; C invents an arbitrary Insert-mode exception (`cc` itself, which does enter Insert mode afterward, disproves this); D invents a deprecation that isn't happening.

</details>

---

**Q8.** `V3jd` is run in Visual mode. What does this sequence accomplish?

- A) It deletes exactly 3 characters
- B) `V` starts Visual Line mode; `3j` extends the selection down 3 more lines (4 lines total); `d` deletes the entire selected block of lines
- C) It saves the file 3 times
- D) `3j` is invalid syntax inside Visual mode and would be ignored

<details>
<summary>Answer</summary>

**B** — This is Visual mode's operator-after-selection composability in action: `V` establishes line-wise selection, a motion (`3j`, move down 3 more lines) extends how much is selected, and the operator (`d`) then acts on the full selected range — 4 lines total (the starting line plus 3 more).

Trap: A misreads `3j` as a character count rather than a line-extension motion; C invents an unrelated save action; D wrongly claims motions don't work inside Visual mode, when extending a selection with motions is exactly how Visual mode is normally used.

</details>

---

Score guide:

| Score | Action |
|---|---|
| 8/8 | Import Anki cards, move to Demo 24 |
| 6-7/8 | Review the wrong answers, then proceed |
| 4-5/8 | Re-read the relevant sections, retry those questions |
| Below 4/8 | Re-read the full demo and redo the walkthrough before proceeding |
````