# Demo 23b — Vim Ex-Mode: Line-Range Command Editing

## Overview

Everything in Demo 23 operated on wherever the cursor happened to be —
one word, one line, one visual selection at a time. Vim has a second,
completely different editing model living behind the `:` prompt: **Ex
mode**, where you address lines by *range* instead of by cursor position,
and issue a command that operates on that whole range at once. This is
the same idea Demo 20's `sed` addressing covered at the shell level —
`:%s/pattern/replacement/flags` is Vim's own version of exactly that
substitution syntax — plus a genuinely distinct capability `sed` doesn't
have built in the same way: the global command, `:g`, which runs an
arbitrary Ex command against every line matching a pattern, anywhere in
the file.

## Demo Objectives

1. Address a range of lines explicitly (`:1,5`, `:.,$`, `:'<,'>`) instead
   of relying on cursor position
2. Use `:%s/pattern/replacement/flags` and explain how it maps onto
   Demo 20's `sed` substitution syntax
3. Use the global command `:g/pattern/command` to run an Ex command
   against every matching line in the file
4. Use `:v` (or `:g!`) to invert a global command to non-matching lines
5. Use `:normal` from Ex mode to run a Normal-mode command against a range
6. Explain when Ex-mode range commands are a better fit than a Vim macro
   (Demo 23) for the same repetitive edit

## Directory Structure

```
23b-vim-ex-mode/
├── README.md                    # this file
├── 23b-vim-ex-mode-anki.csv      # Anki flashcard deck
└── 23b-vim-ex-mode-quiz.md       # standalone quiz
```

## Environments

| Task | Ubuntu 24.04 | Rocky Linux 9.8 | EC2 Amazon Linux |
|---|---|---|---|
| All lab tasks | ✅ (`vim` preinstalled or `sudo apt install vim`) | ✅ (`vi`/`vim` preinstalled or `sudo dnf install vim`) | Not required |

> **Verification note:** As with Demos 23 and 23a, this sandbox has no
> network access to install Vim, so nothing here could be captured from a
> live session. Range addressing, `:%s`, and `:g`/`:v` are long-stable,
> extensively-documented Vim behavior. One specific is marked
> `[Unverified — test before use]` below — the exact precedence/ordering
> when combining `:g` with `:normal` on a range that itself changes line
> count mid-execution — and should be confirmed on a real machine before
> being treated as fully settled.

## Recall Check — 23a-vimdiff

Answer these from memory before continuing — no looking back at Demo 23a.

1. What's the difference between `do` and `dp`, and which direction does
   each one move content?
2. Why doesn't a plain `vim -d` 3-way session automatically flag a
   conflict the way `diff3` does?
3. When is `vimdiff` the right tool versus `patch` for resolving a
   difference between two files?

<details>
<summary>Answers</summary>

1. `do` (obtain) pulls a difference IN from the other window into the
   current one; `dp` (put) pushes a difference OUT from the current
   window to the other one — opposite directions.

2. `vimdiff` is a display/navigation tool for differences, not a
   conflict-resolution algorithm — it has no built-in logic for detecting
   when both non-base versions changed the same content differently,
   which is specifically what `diff3`'s `====` marker adds.

3. `vimdiff` when a human needs to look at competing changes and decide;
   `patch` when the resolution is already fully decided and just needs
   applying, possibly unattended, possibly across many files.

</details>

## Part 1 — Concepts

### 1.1 Addressing lines by range, not by cursor position

Every Normal-mode command from Demo 23 acted on wherever the cursor
already was. Ex-mode commands work the opposite way: you specify *which
lines* first, as a range, then a command to run against all of them:

```
:5           # jump to line 5 (a range with no command is just navigation)
:1,5d        # delete lines 1 through 5
:.,$d        # delete from the CURRENT line (.) to the last line ($)
:.,+3d       # delete the current line plus the next 3
:'<,'>d      # delete whatever's currently marked as the last Visual
             # selection — Vim automatically sets '< and '> marks the
             # moment you make a Visual selection and press : afterward,
             # which is why typing : right after a Visual selection
             # pre-fills the command line with :'<,'> for you
```

**The general shape is `:{range}{command}`.** `.` means the current line,
`$` means the last line, `%` (covered next) means the whole file — these
are the range building blocks everything else in this demo composes.

### 1.2 `:%s` — Vim's substitution, and its relationship to sed

```
:%s/old/new/g
```

Break this down against Demo 20's `sed -i 's/old/new/g' file`:

```
%     range = the whole file (equivalent to :1,$)
s     the substitute command itself — same letter, same job as sed's s
/old/new/   pattern and replacement, identical syntax to sed
g     flag: replace ALL matches per line, not just the first — the exact
      same g flag, the exact same default-without-it behavior, as sed
```

**This isn't a coincidence — Vim's substitute command and `sed`'s `s///`
command share a common ancestor** (both ultimately derive from the `ed`
line editor's command language), which is why the syntax, the `g` flag,
and even backreferences in the replacement (`\1`, `\2` — Demo 20 §1.x)
transfer directly between the two without relearning anything.

**Other useful ranges with `:s`:**

```
:s/old/new/       # current line ONLY — no range prefix means "just
                   # the current line," not the whole file
:1,10s/old/new/g  # lines 1 through 10 only
:%s/old/new/gc     # c flag: confirm each replacement individually —
                   # y/n/a (yes/no/all) prompt per match, useful when
                   # you're not fully confident every match should change
```

### 1.3 The global command — `:g/pattern/command`

`:%s` operates on a fixed line range regardless of content. `:g` is
fundamentally different: it finds every line **matching a pattern**,
wherever those lines are in the file, and runs an arbitrary Ex command
against each one:

```
:g/pattern/d          # delete every line matching pattern, anywhere
:g/pattern/normal Ix   # for every matching line, run the Normal-mode
                       # command "insert x at the start of the line"
                       # (Demo 23 §1.1's I, not a range — the command
                       # being run per matching line is a Normal-mode
                       # one, invoked via :normal, covered in §1.5)
:g/pattern/s/x/y/      # for every matching line, ALSO run a substitution
                       # — this combines pattern-matching (:g) with
                       # line-editing (:s) in a single command
```

**Why this is a distinct capability, not just "sed with extra steps":**
`sed` operates on lines sequentially, in a single pass. `:g` first
identifies every matching line across the whole file, then runs the
command against each — meaning the command itself can safely change line
numbers or content elsewhere without the pattern-matching step getting
confused about which lines still match, because matching already happened
before any edits started.

### 1.4 `:v` — the inverse of `:g`

```
:v/pattern/d      # delete every line that does NOT match pattern —
                   # the complement of :g/pattern/d
:g!/pattern/d      # identical meaning — :v is shorthand for :g!
```

`:v` (or equivalently `:g!`) is useful for the opposite instinct: "keep
only lines matching this pattern, delete everything else" — rather than
"delete lines matching this pattern."

### 1.5 `:normal` — running a Normal-mode command from Ex mode

```
:normal Ix        # run "insert x at start of line" (Demo 23's I) as if
                   # typed in Normal mode, against the current line
:1,5normal A;      # run "append ; at end of line" (Demo 23's A) against
                   # lines 1 through 5
:g/pattern/normal A;   # combine with :g — append ; to the end of every
                       # line matching pattern, anywhere in the file
```

`:normal` is the bridge between Ex mode's range/pattern addressing and
Normal mode's operator+motion vocabulary from Demo 23 — anything you
already know how to do to one line with ordinary Normal-mode keystrokes,
`:normal` lets you apply to an entire range or every pattern-matched line
at once, without recording a macro.

**A genuine subtlety when combining `:g` with `:normal` on a range that
changes line count mid-execution** (e.g. a `:normal` command that itself
deletes or adds lines per match): the exact ordering/precedence in that
specific case is worth confirming directly rather than assumed.
`[Unverified — test before use]`
For straightforward per-line edits that don't change the total line count
(like the `Ix`/`A;` examples above), this isn't a concern.

### 1.6 Ex-mode ranges vs. a Vim macro — when to reach for which

Demo 23 covered macros (`qa`...`q`, `N@a`) as the tool for "repeat this
exact sequence of keystrokes N times." Ex-mode range commands are a better
fit whenever the edit can be expressed as "for lines matching X, do Y" —
which is most repetitive edits — because you state the *rule* once
(`:g/pattern/command`) rather than recording a specific sequence and
trusting it to replay correctly against lines that might not all share
the same structure (the exact failure mode in Demo 23's macro Break-Fix).
A macro remains the better fit only when the edit genuinely can't be
expressed as a pattern-plus-command rule — e.g., an edit that depends on
something a pattern can't capture, like "the third occurrence specifically,
regardless of what precedes it."

## Part 2 — Hands-On Lab

### Commands used in this demo

| Command | Purpose |
|---|---|
| `:1,5d` / `:.,$d` / `:'<,'>d` | Delete an explicit line range / current-to-end / last Visual selection |
| `:%s/old/new/g` | Substitute across the whole file, all matches per line |
| `:%s/old/new/gc` | Same, with per-match confirmation |
| `:g/pattern/command` | Run an Ex command against every line matching pattern |
| `:v/pattern/command` (or `:g!`) | Run an Ex command against every line NOT matching pattern |
| `:normal {keys}` | Run Normal-mode keystrokes against the current line or a range |

### Task 1 — Range basics

```bash
printf 'line one\nline two\nline three\nline four\nline five\n' > /tmp/exmode-a.txt
vim /tmp/exmode-a.txt
```

Try `:2,4d` (delete lines 2 through 4), undo (`u`), then `:.,$d` with the
cursor on line 3 (deletes line 3 through the end). Undo again. Select
lines 2-4 in Visual Line mode (`V`, then move down), press `:` and
confirm the command line pre-fills with `:'<,'>` — complete it with `d`
and confirm the same three lines are removed, this time via the
auto-populated Visual-range marks instead of typed line numbers.

### Task 2 — `:%s` and the sed comparison

```bash
printf 'ERROR: disk full\nERROR: disk full\nINFO: backup ok\n' > /tmp/exmode-b.txt
vim /tmp/exmode-b.txt
```

Run `:%s/ERROR/WARN/g` and confirm both matching lines change. Undo, then
compare directly against the shell equivalent from Demo 20:

```bash
sed -i 's/ERROR/WARN/g' /tmp/exmode-b.txt
```

Confirm both approaches produce an identical result on this file. Back in
Vim, try `:%s/ERROR/WARN/gc` and step through the `y`/`n`/`a` prompts on
a file with more matches than you want to change all at once.

### Task 3 — The global command

```bash
printf 'keep\nDEBUG line 1\nkeep\nDEBUG line 2\nkeep\n' > /tmp/exmode-c.txt
vim /tmp/exmode-c.txt
```

Run `:g/DEBUG/d` and confirm both `DEBUG` lines are removed, wherever they
were in the file, while the `keep` lines survive untouched. Undo, then try
`:v/DEBUG/d` (or `:g!/DEBUG/d`) and confirm the opposite: now only the
`DEBUG` lines survive.

### Task 4 — Combining `:g` with `:s` and `:normal`

```bash
printf 'TODO fix auth\nkeep this\nTODO fix logging\nkeep this too\n' > /tmp/exmode-d.txt
vim /tmp/exmode-d.txt
```

Run `:g/TODO/s/TODO/DONE/` and confirm only the `TODO` lines change, in
place, leaving the `keep this` lines untouched. Undo, then run
`:g/TODO/normal A !!!` (append `!!!`, via `:normal`'s use of Demo 23's `A`)
and confirm `!!!` is appended to the end of every `TODO` line only.

### Task 5 — Recognize the Ex-range vs. macro boundary

You need to prefix every line containing the word `WARNING` with `# ` —
in a file where `WARNING` lines are scattered unpredictably throughout,
mixed in with many other lines. Per §1.6, would you reach for a Vim macro
(Demo 23) or an Ex-mode `:g` command here, and what specifically makes
one a better fit than the other for this particular case?

## What You Learned

- ✅ Addressed a range of lines explicitly instead of relying on cursor
  position
- ✅ Used `:%s/pattern/replacement/flags` and connected it directly to
  Demo 20's `sed` substitution syntax
- ✅ Used `:g/pattern/command` to run an Ex command against every matching
  line, anywhere in the file
- ✅ Used `:v` to invert a global command to non-matching lines
- ✅ Used `:normal` from Ex mode to run a Normal-mode command against a
  range or pattern match
- ✅ Explained when Ex-mode range commands beat a Vim macro for the same
  repetitive edit

## Part 3 — Break-Fix Scenario

### Scenario — NovaTech Ops: two Ex-mode incidents

> **Incident report — NovaTech Ops, ticket #4855**
>
> Alert 1: "An engineer ran a substitution intending to fix one line and
> it changed the entire file instead." Alert 2: "A `:g` command meant to
> clean up a config file deleted lines that should have survived."

**Incident 1 — the missing range**

```
(intending to fix just the current line)
:s/prod-db-01/prod-db-02/
(works as expected — only the current line changes)
(five minutes later, same intent, different line)
:%s/prod-db-01/prod-db-02/
(every occurrence in the ENTIRE file just changed, not just this line)
```

Tasks:
1. Reference §1.1/§1.2 — what's the actual difference between `:s/.../.../ `
   and `:%s/.../.../ `, and which one did the engineer mean to run the
   second time?
2. Why does Vim not warn before a whole-file substitution the way some
   tools might prompt "are you sure"?
3. What flag, covered in §1.2, would have let the engineer catch this
   before it happened even while using `:%s`?

**Incident 2 — an over-broad global delete**

```
:g/tmp/d
(intended: delete lines referencing a specific temporary file path)
(actual: also deleted lines containing the word "attempt" — because
 the pattern "tmp" also matches inside unrelated words)
```

Tasks:
1. Reference §1.3 — what does `:g/pattern/command` actually match against
   by default, and why did `tmp` match more than the specific file path
   the engineer had in mind?
2. What change to the pattern itself (not the command) would have scoped
   this correctly to just the intended lines?
3. What's the general lesson about `:g`'s pattern matching versus what a
   human might have specifically pictured when writing it?

<details>
<summary>Answers</summary>

**Incident 1:** `:s/.../.../` with no range prefix operates on the
current line **only**. `:%s/.../.../` uses `%`, which is range shorthand
for the entire file (`:1,$`) — every matching occurrence, on every line,
changes. The second time, the engineer meant to repeat the same
single-line fix pattern from before but typed `%s` instead of plain `s` —
a small typo with a large blast radius, since both are entirely valid,
correctly-formed commands; Vim has no way to know the `%` wasn't
intentional. Vim doesn't prompt "are you sure" before a whole-file
substitution because `:%s` is an extremely common, entirely legitimate
operation — most substitutions genuinely are meant to apply file-wide, so
a confirmation prompt on every one would be more friction than protection
for the overwhelming majority of correct uses. The `c` flag from §1.2
(`:%s/old/new/gc`) is exactly the safety valve for this situation — it
forces a per-match `y`/`n`/`a` prompt, which would have caught the
unintended file-wide scope immediately, on the very first prompted match,
before any unwanted changes were made.

**Incident 2:** `:g/pattern/command` matches `pattern` as a substring
search against each line's full text by default, the same way a bare
regex without anchors matches anywhere it appears (Demo 17's unanchored
default matching) — it does not restrict matching to whole words or to
any particular position on the line. `"tmp"` matched inside `"attempt"`
because `attempt` literally contains the three characters `t`, `m`, `p`
in sequence — the pattern has no way to distinguish "the standalone word
tmp" from "any occurrence of these three characters anywhere." Scoping
this correctly requires making the pattern itself more specific — for
example, anchoring to the actual file path it was meant to match
(`:g/\/tmp\//d` to match only the literal path segment `/tmp/`), or using
a word-boundary-aware pattern if the intent really was the standalone word
`tmp` specifically (the same `\<tmp\>`-style boundary concept from Demo
17/18's whole-word matching). The general lesson: `:g`'s pattern matching
is exactly as literal and unanchored as any other regex-based tool
covered in this series — it matches what the pattern actually says, not
what the person writing it was specifically picturing, and vague patterns
reliably over-match in exactly this way.

</details>

## Interview Prep

**Q1. What's the practical difference between Vim's `:%s/old/new/g` and running `sed -i 's/old/new/g' file` from the shell, given the syntax is nearly identical?**

Functionally, on a straightforward substitution, the result is identical
— both descend from the same `ed` line-editor command language, which is
exactly why the syntax transfers directly. The practical difference is
context: `:%s` runs inside an already-open, already-being-edited Vim
buffer, so you can review the result immediately, undo it (`u`) if it's
wrong, or run further Ex/Normal-mode commands against the same buffer
right after — while `sed -i` edits the file directly from the shell,
outside any interactive editing session, which is the better fit when the
change needs to be scripted, unattended, or applied identically across
many files (Demo 23 §1.7's Vim-vs-sed boundary, applied here to Ex-mode
specifically).

---

**Q2. An engineer wants to delete every line containing the word `DEBUG`, wherever it appears in a 500-line file. Why is `:g/DEBUG/d` a better fit here than a Vim macro?**

A macro would need to be recorded against one specific line's structure
and then trusted to replay correctly across all 500 lines — exactly the
failure mode Demo 23's macro Break-Fix demonstrated, where lines that
don't match the recorded assumption produce silently wrong results.
`:g/DEBUG/d` instead states the actual rule directly — "for every line
matching this pattern, delete it" — and Vim finds and acts on every
matching line itself, with no assumption about surrounding lines' shape
required at all.

---

**Q3. What's the actual difference between `:v/pattern/d` and `:g!/pattern/d`?**

None — they're the same command. `:v` is documented shorthand for
`:g!`, both meaning "run this command against every line that does NOT
match the pattern," the inverse of plain `:g/pattern/command`.

---

**Q4. Why does `:normal` matter as a bridge between Ex mode and what Demo 23 already taught, rather than being a separate thing to memorize from scratch?**

`:normal` doesn't introduce new editing capability of its own — it lets
any Normal-mode keystroke sequence you already know from Demo 23 (motions,
operators, `I`/`A`, etc.) run against an Ex-mode range or every
pattern-matched line from `:g`, instead of being limited to wherever the
cursor currently sits. It's the connective piece that makes Ex mode's
range/pattern addressing and Normal mode's editing vocabulary work
together, rather than being two entirely separate skill sets.

---

## Key Takeaways

1. **Ex-mode commands address lines by explicit range or pattern match,
   not by cursor position — the opposite model from Demo 23's Normal-mode
   commands.** `:{range}{command}` is the general shape everything else in
   this demo builds on.
2. **`:%s/old/new/g` is Vim's version of `sed`'s `s///g`, sharing a common
   ancestor in the `ed` line editor** — syntax, the `g` flag, and even
   backreferences transfer directly between the two with nothing to
   relearn.
3. **A bare `:s` affects only the current line; `:%s` affects the whole
   file — the difference between the two is a single character with a
   large blast radius.** The `c` flag (confirm each match) is the safety
   valve when you're not fully certain of the scope before running it.
4. **`:g/pattern/command` finds every matching line across the whole file
   first, then runs the command against each one** — a fundamentally
   different mechanism from a macro's "replay these exact keystrokes N
   times," and immune to the structural-assumption failure mode that
   breaks macros on non-uniform lines.
5. **`:v` (or `:g!`) is exactly the inverse of `:g` — same pattern
   matching, opposite selection (non-matching lines instead of matching
   ones).**
6. **`:normal` bridges Ex-mode range/pattern addressing with Normal-mode's
   editing vocabulary from Demo 23** — anything you can already do to one
   line with ordinary keystrokes, `:normal` lets you apply across a whole
   range or every pattern match at once.
7. **`:g`'s pattern matching is exactly as literal and unanchored as any
   other regex tool in this series (Demo 17/18) — it matches substrings,
   not intentions.** A pattern like `tmp` matches inside `attempt` just as
   readily as it matches a standalone word; scope patterns as precisely
   as the actual intent requires.

> **Demo scope:** Primary concept: Ex-mode line-range and pattern-based
> command editing. Supporting concepts: range syntax, `:%s` substitution,
> the global command (`:g`/`:v`), `:normal`.
> Estimated completion time: 32 minutes (reading + hands-on + verification).
> Checkpoints: 5 natural stopping points (after Task 1 through Task 5).

## Quick Commands Reference

### Ranges

| Syntax | Meaning |
|---|---|
| `:N` | Line N |
| `:N,M` | Lines N through M |
| `:.,$` | Current line through end of file |
| `:'<,'>` | Last Visual selection (auto-filled after pressing `:` right after selecting) |
| `:%` | The whole file (shorthand for `:1,$`) |

### Substitution and global commands

| Command | Purpose |
|---|---|
| `:s/old/new/` | Substitute on the current line only, first match |
| `:%s/old/new/g` | Substitute across the whole file, all matches per line |
| `:%s/old/new/gc` | Same, with per-match confirmation |
| `:g/pattern/command` | Run an Ex command against every matching line |
| `:v/pattern/command` (= `:g!`) | Run an Ex command against every NON-matching line |
| `:normal {keys}` | Run Normal-mode keystrokes against the current line/range |

## Appendix A — Anki Cards

**`23b-vim-ex-mode-anki.csv`:**

```
#deck:Linux Mastery::Module 6 - Editors::Demo 23b - Vim Ex-Mode
#separator:Comma
#columns:Front,Back,Tags
"What's the general shape of an Ex-mode command, and how does it differ from how Demo 23's Normal-mode commands operate?","{range}{command} — Ex-mode commands address lines by explicit range or pattern match first, then act on all of them at once. Normal-mode commands (Demo 23) instead act on wherever the cursor already is.","demo23b,exmode,fundamentals,cross-demo"
"Why does :%s/old/new/g use almost identical syntax to sed's s/old/new/g?","Both descend from the ed line editor's original command language — the s command, the pattern/replacement syntax, the g flag, and even backreferences (\\1, \\2) transfer directly between the two.","demo23b,exmode,sed,cross-demo"
"What's the difference between :s/old/new/ and :%s/old/new/, and why is mixing them up dangerous?","A bare :s affects only the current line. :%s (% = the whole file, shorthand for :1,$) affects every line. The difference is a single character with a large blast radius — an engineer meaning to fix one line can accidentally change the entire file by typing % out of habit.","demo23b,exmode,substitution,common-mistake"
"What does the c flag do in :%s/old/new/gc, and when is it worth using?","Forces a per-match y/n/a confirmation prompt before each replacement. Worth using whenever you're not fully confident every match in the file should actually change — it catches an over-broad or accidental whole-file substitution before any unwanted changes are made.","demo23b,exmode,substitution,best-practice"
"What does :g/pattern/command actually do, mechanically, and how is this different from how a Vim macro works?","It first finds every line matching pattern across the whole file, then runs command against each match. A macro instead replays a fixed recorded keystroke sequence N times, assuming every targeted line shares the same structure — :g has no such assumption, since matching happens before any editing does.","demo23b,exmode,global-command,macros,cross-demo"
"What is :v, and what's its relationship to :g?",":v is documented shorthand for :g! — both mean 'run this command against every line that does NOT match the pattern,' the inverse of plain :g/pattern/command.","demo23b,exmode,global-command"
"What does :normal let you do that plain Ex-mode range/pattern commands alone cannot?","It runs an actual Normal-mode keystroke sequence (anything from Demo 23 — motions, operators, I/A, etc.) against an Ex-mode range or every :g pattern match, bridging the two editing models instead of keeping them separate.","demo23b,exmode,normal-command,cross-demo"
"A :g/tmp/d command deletes more lines than intended, including ones containing the word 'attempt.' Why?","Ex-mode pattern matching is unanchored substring matching by default (same as any regex tool in this series) — 'tmp' matches literally anywhere those three characters appear in sequence, including inside 'attempt.' The pattern needs to be made more specific (e.g. anchored to the actual intended text) to avoid this.","demo23b,exmode,pattern-matching,common-mistake,cross-demo"
```

## Appendix B — Quiz

**`23b-vim-ex-mode-quiz.md`:**

````markdown
# Quiz — Demo 23b: Vim Ex-Mode: Line-Range Command Editing

> One correct answer per question unless stated otherwise.
> Target: 80% or above before moving to Demo 24.

---

**Q1.** What's the difference between `:s/old/new/` and `:%s/old/new/`?

- A) They are identical — `%` is purely cosmetic
- B) `:s` affects only the current line; `:%s` affects the whole file (`%` is shorthand for `:1,$`)
- C) `:s` requires a Visual selection first; `:%s` does not
- D) `:%s` only works on the first line of the file

<details>
<summary>Answer</summary>

**B** — This is the exact mistake in this demo's Break-Fix Incident 1: a
single extra character (`%`) turns a single-line fix into a whole-file
change.

</details>

---

**Q2.** What does the `c` flag do in `:%s/old/new/gc`?

- A) It makes the substitution case-insensitive
- B) It forces a per-match y/n/a confirmation prompt before each replacement is applied
- C) It counts the number of matches without making any changes
- D) It copies the matched text to a register before replacing it

<details>
<summary>Answer</summary>

**B** — `c` (confirm) is the safety valve for exactly Incident 1's
scenario: it lets you catch an over-broad substitution on the first
prompted match, before any unwanted changes happen.

</details>

---

**Q3.** `:g/pattern/command` first identifies every matching line, then
runs the command against each one. Why does this matter compared to how
a Vim macro operates?

- A) It doesn't matter — both approaches work identically
- B) Matching happens before any editing starts, so the command can safely change line content or count elsewhere without the pattern-matching step getting confused about which lines still qualify — a macro instead assumes every targeted line shares the same structure as the one it was recorded against
- C) `:g` is simply a faster-executing version of a macro with no behavioral difference
- D) Macros can only operate on the current line, while `:g` can operate on the whole file — that's the only difference

<details>
<summary>Answer</summary>

**B** — This is the structural reason `:g` avoids the exact failure mode
Demo 23's macro Break-Fix demonstrated (a macro silently producing wrong
results on lines that don't match its recorded assumption).

</details>

---

**Q4.** What is `:v/pattern/d` equivalent to?

- A) `:g/pattern/d`
- B) `:g!/pattern/d`
- C) `:%s/pattern//`
- D) There is no equivalent — `:v` is a distinct, unrelated command

<details>
<summary>Answer</summary>

**B** — `:v` is documented shorthand for `:g!` — both mean "run this
command against every line that does NOT match the pattern."

</details>

---

**Q5.** `:g/tmp/d` unexpectedly deletes a line containing the word
"attempt." Why?

- A) `:g`'s pattern matching is unanchored substring matching by default — "tmp" matches anywhere those three characters appear in sequence, including inside "attempt"
- B) `:g` has a bug that causes it to match unrelated words at random
- C) "attempt" is a reserved Vim keyword that triggers false matches
- D) `:g` only matches whole words, so this shouldn't be possible — it must be a different pattern that actually matched

<details>
<summary>Answer</summary>

**A** — Ex-mode pattern matching follows the same unanchored,
substring-based default as every other regex tool in this series (Demo
17/18) — the pattern has no way to distinguish a standalone word from any
occurrence of the same characters inside a longer word, unless the
pattern itself is written to require that.

Trap: D wrongly assumes whole-word matching is the default behavior,
which it explicitly is not without additional pattern syntax.

</details>

---

**Q6.** What does `:normal Ix` do when run against a range, e.g.
`:1,5normal Ix`?

- A) It only works on line 1, ignoring the rest of the range
- B) It runs the Normal-mode command "insert x at the start of the line" (Demo 23's `I`) against each line from 1 to 5
- C) It inserts the literal text "Ix" once, at the top of the file
- D) `:normal` cannot be combined with a range — this syntax is invalid

<details>
<summary>Answer</summary>

**B** — `:normal` runs the given Normal-mode keystrokes against every
line in the specified range (or every `:g` pattern match), one line at a
time — exactly the bridge between Ex-mode addressing and Normal-mode
editing described in §1.5.

</details>

---

**Q7.** Per §1.6, when would recording a Vim macro still be the better
choice over an Ex-mode `:g` command for a repetitive edit?

- A) Never — `:g` is strictly superior in every case
- B) When the edit genuinely can't be expressed as a pattern-plus-command rule — e.g., "the third occurrence specifically, regardless of what precedes it," something a pattern match can't directly express
- C) Only when editing fewer than 10 lines
- D) Only when working with binary files

<details>
<summary>Answer</summary>

**B** — Most repetitive edits genuinely can be expressed as "for lines
matching X, do Y," which is exactly `:g`'s strength. A macro remains the
better fit specifically when the selection criterion isn't something a
pattern can capture at all.

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