# Demo M01-01 — Wildcards, Globbing & Quoting

## Overview

Before a command runs, the shell rewrites what you typed. When you run
`rm session_*.tmp`, `rm` never sees the `*`. The shell replaces the
pattern with the list of matching filenames first, and `rm` only
receives that finished list.

Most of the time this is invisible and convenient. The rest of the
time it silently causes damage: a variable holds `*` instead of a
filename, or a filename contains a space, and the shell expands it
differently than you intended. No error, no warning, just the wrong
files deleted.

This demo covers the three rewriting mechanisms the shell applies to
every command, and the one control (quoting) that switches them off:

| Mechanism | What it does | Example |
|---|---|---|
| **Brace expansion** | Generates text from a template | `{a,b}` → `a b` |
| **Word splitting** | Cuts an unquoted variable's value into separate words at spaces | `$name` with `hello world` → `hello` `world` |
| **Globbing** | Replaces a wildcard pattern with matching file names | `*.txt` → `a.txt b.txt` |
| **Quoting** | Switches the mechanisms above off (fully or partly) | `"$name"` → `hello world` |

Every other demo in this series relies on this.

---

## Demo Objectives

By the end of this demo you will be able to:

1. Explain that the shell, not the command, performs expansion, and name the stages in the order it applies them
2. Use the glob wildcards `*`, `?`, `[...]` (including ranges such as `[1-3]`) and `[!...]`
3. Use brace expansion (`{a,b}`, `{1..5}`, and combined groups) and explain how it differs from globbing, including glob ranges vs. brace ranges
4. Explain word splitting and when it happens
5. Explain what unquoted, double-quoted and single-quoted text each allow and suppress
6. Escape a single special character with a backslash
7. Explain why a wildcard stored in a variable is expanded when the variable is *used*, not when it is *assigned*
8. Diagnose and fix the failure where an unquoted variable deletes the wrong files

---

## Directory Structure

```
01-globbing-quoting/
├── README.md                         # this file
├── m01-01-globbing-quoting-anki.csv  # Anki flashcard deck (Appendix A)
├── m01-01-globbing-quoting-quiz.md   # standalone quiz (Appendix B)
└── src/                              # lab working directories (git-ignored)
    ├── glob-lab/                     # Tasks 1-2: wildcards and brace expansion
    │   └── project/                  # filled in Task 2 with src, docs and tests
    ├── quoting/                      # Task 3: word splitting and quoting
    └── cleanup-demo/                 # Task 5: the rm footgun
```

All lab work happens inside `src/`. The folders are created by the
lab commands themselves (`mkdir -p`), so you only need to be in the
demo directory (`01-globbing-quoting/`) when you start.

> **Note: add `src/` to `.gitignore`.** The lab folders contain
> throwaway files (including files the lab deliberately deletes), so
> they should never be committed. Add this line to the repository's
> root `.gitignore` once:
>
> ```
> src/
> ```
>
> A bare `src/` pattern matches a directory named `src` at any depth,
> so it covers every demo that follows the same layout.

---

## Environments

| Task | Ubuntu 24.04 | Rocky Linux 9.8 | EC2 Amazon Linux |
|---|---|---|---|
| All lab tasks | ✅ | ✅ | Not required |

All behavior in this demo is **bash** behavior. Two traps catch students:

- **Your login shell may be zsh, not bash.** zsh handles no-match patterns, `!` characters, and unquoted variables differently, so several lab results will not match this demo. Step 0 of the lab switches you into bash.
- **`/bin/sh` is not bash.** On Ubuntu and Debian, `/bin/sh` is `dash`, which has no brace expansion. A script that uses `{a,b}` must start with `#!/bin/bash`, not `#!/bin/sh`.

---

## Part 1 — Concepts

### 1.1 The shell rewrites your command before running it

When you press Enter, the shell does not run your text as typed. It
first works through a series of rewriting stages (called *expansions*),
then runs the command with the rewritten result. You can see the
rewritten result by putting `echo` in front: `echo` prints exactly the
words the shell produced.

In bash, these `rewriting stages/expansions` run in this fixed order:

1. **Brace expansion**: `{a,b}` is turned into text (§1.3)
2. **Variable and other `$` expansions**: `$name` is replaced by its value (§1.4). This stage also handles `~`, `$(command)` and `$((math))`, shown below
3. **Word splitting**: unquoted variable values & expansion results are cut into words at spaces (§1.4)
4. **Globbing**: wildcard patterns are replaced by matching filenames (§1.2)
5. **Quote removal**: the quote characters and backslashes you typed are removed (§1.5 and §1.6)

That is the complete order for everything this course uses. Two
things happen even earlier and are covered elsewhere: **alias
expansion** (Demo M01-03) and, in interactive shells only, **history
expansion** (the `!` character). A rarely needed one, process
substitution `<(cmd)`, is left out.

Stage 2 is really a small group of expansions that run together, left
to right:

| Expansion | Example | Result |
|---|---|---|
| `~` (home directory) | `echo ~` | `/home/you` |
| `$name` (variable) | `x=hi; echo $x` | `hi` |
| `$(command)` (command output) | `echo $(echo hi)` | `hi` |
| `$((math))` (arithmetic) | `echo $((2+3))` | `5` |

These stages apply to the **words of a command** (its name and its
arguments). The right-hand side of an assignment such as
`tmpfile=session_*.tmp` is handled differently: §1.7 explains how, and
why it matters.

Quoting (§1.5) controls which of these stages are allowed to happen.

### 1.2 Globbing: wildcards that match file names

**Globbing** (also called **pathname expansion**) replaces a pattern with
the names of existing files that match it. It is the shell's job: for
`ls *.txt`, `ls` never receives `*.txt`. It receives `file1.txt file2.txt`.

There are four wildcard characters:

| Wildcard | Matches |
|---|---|
| `*` | Zero or more of any character |
| `?` | Exactly one of any character |
| `[...]` | Exactly one character from the set or range inside the brackets |
| `[!...]` | Exactly one character **not** in the set or range |

Given files `file.txt file1.txt file2.txt file3.log`:

```bash
echo file*.txt      # file.txt file1.txt file2.txt  (* can match zero characters)
echo file?.txt      # file1.txt file2.txt           (? needs exactly one: file.txt has none)
echo file[12].txt   # file1.txt file2.txt           (the character must be 1 or 2)
echo file[!1].txt   # file2.txt                     (any single character except 1)
```

#### Ranges inside brackets

Inside `[...]`, a hyphen between two characters means "every
character from the first to the last". So `[1-3]` is the same as
`[123]`, and `[a-c]` is the same as `[abc]`.

```bash
echo file[1-3].txt   # file1.txt file2.txt   (existing files whose character is 1, 2 or 3)
echo file[a-c].txt   # file[a-c].txt         (no such files, so passed through literally)
```

Three rules to remember:

- A bracket expression matches **exactly one character**, and a range runs between two **single characters**. `[1-10]` is *not* "1 to 10": it means the range `1-1` plus the character `0`, so it matches `1` or `0`.
- The hyphen is the only range syntax in globbing. **Two dots mean nothing special inside brackets**: `[1..5]` is a set of three characters, `1`, `.` and `5`.
- To *generate* a sequence such as 1 to 5, use brace expansion (`{1..5}`), not brackets.

| | Glob range `[1-3]` | Brace range `{1..3}` |
|---|---|---|
| Syntax | Hyphen inside square brackets | Two dots inside curly braces |
| What it does | Matches one character, **only on existing files** | Generates every value, **no filesystem check** |
| `file[1-3].txt` vs. `file{1..3}.txt` (only `file1.txt` and `file2.txt` exist) | `file1.txt file2.txt` | `file1.txt file2.txt file3.txt` |

Two behaviors that surprise people:

- **`*` does not match dotfiles.** `echo *` skips `.hidden`. The pattern itself must start with a dot (`echo .*`) to match it.
- **No match means no change.** If nothing matches, bash passes the pattern through **unchanged, as literal text**. It does not produce an empty list or an error. `echo *.xyz` prints `*.xyz`. A command that receives a literal wildcard as an argument almost always means nothing matched.

### 1.3 Brace expansion: generating text from a template

**Brace expansion** looks like globbing but is a different feature. It
generates text from a template and **never looks at the filesystem**.
The names it produces do not need to exist.

#### Form 1 — one group

Comma-separated items inside braces become separate words:

```bash
echo {a,b}        # a b
echo {a,b,c}      # a b c
```

#### Form 2 — text around the group

Anything before or after the braces is attached to every item:

```bash
echo x{a,b}y      # xay xby
echo file{1,2}.txt   # file1.txt file2.txt
```

#### Form 3 — ranges

`{start..end}` counts for you:

```bash
echo {1..5}       # 1 2 3 4 5
echo {a..e}       # a b c d e
```

#### Form 4 — two groups side by side

Bash takes each item from the first group and pairs it with every
item from the second group:

| First group `{a,b}` | Second group `{1,2}` | Result |
|---|---|---|
| a | 1 | `a1` |
| a | 2 | `a2` |
| b | 1 | `b1` |
| b | 2 | `b2` |

So `echo {a,b}{1,2}` prints `a1 a2 b1 b2`. The first group changes
slowest; swapping the groups, `{1,2}{a,b}`, gives `1a 1b 2a 2b`.

#### Practical uses

```bash
mkdir -p project/{src,docs,tests}   # three directories in one command
cp app.conf{,.bak}                  # expands to: cp app.conf app.conf.bak
```

In `app.conf{,.bak}` the first item is empty, so it produces
`app.conf` (nothing added) and `app.conf.bak`. This is the standard
quick backup idiom.

#### Brace expansion vs. globbing

| | Brace expansion | Globbing |
|---|---|---|
| Looks at the filesystem | No | Yes |
| Works if the files don't exist | Yes | No (nothing matches) |
| Example | `file{9,10}.txt` → always two names | `file*.txt` → only existing files |

#### Proof that brace expansion runs first

```bash
echo nomatch{a,b}*.zzz
# 1. brace expansion: nomatch{a,b}*.zzz → nomatcha*.zzz nomatchb*.zzz   (2 words)
# 2. variable       : none
# 3. word splitting : none (no variable)
# 4. globbing       : nomatcha*.zzz → no file matches → left as typed
#                     nomatchb*.zzz → no file matches → left as typed
# echo gets 2 arguments → nomatcha*.zzz nomatchb*.zzz
```

Bash first expanded the braces into two patterns, then tried to glob
each one. Neither matched anything, so each passed through literally.
If globbing had run first, you would get one pattern, not two.

### 1.4 Word splitting: how one value becomes several words

When a variable is **unquoted**, bash replaces it with its value and
then cuts that value into separate words at every space. This is
**word splitting**.

```bash
name="hello world"
printf '[%s]\n' $name
# 2. variable      : $name → hello world
# 3. word splitting: hello world → hello | world        (2 words)
# 4. globbing      : no wildcards → unchanged
# printf gets 2 arguments → prints:
# [hello]
# [world]
```

`printf '[%s]\n'` prints each argument it receives inside brackets on
its own line, so it shows exactly how many arguments the command got.
Here the variable held one value but the command received **two**
arguments.

After splitting, each resulting word is also eligible for globbing.
That is why an unquoted variable holding `*.txt` gets expanded into
filenames.

### 1.5 Quoting: switching expansions off

Quoting tells the shell how much rewriting to allow:

| Style | Brace expansion | `$variable` expands | `$(command)` runs | Word splitting | Globbing |
|---|---|---|---|---|---|
| Unquoted | Yes | Yes | Yes | **Yes** | **Yes** |
| `"double quotes"` | No | Yes | Yes | No | No |
| `'single quotes'` | No | **No** | **No** | No | No |

The shell works through the stages in a fixed order (§1.1), and
quoting decides which of them are skipped:

| Stage | What it does | Skipped by quotes? |
|---|---|---|
| 1. Brace expansion | `{a,b}` → `a b` | Yes (double and single) |
| 2. Variable / command expansion | `$name` → its value | Single quotes only |
| 3. Word splitting | Cuts the result at spaces | Yes (double and single) |
| 4. Globbing | Replaces wildcards with matching filenames | Yes (double and single) |
| 5. Quote removal | Removes the quote characters and backslashes you typed | No: this is the stage that removes them |

Using `name="hello world"` and `pat="*.txt"` with files `a.txt b.txt`.
None of the next five lines contains braces, so stage 1 has nothing
to do in them.

```bash
printf '[%s]\n' $name
# 2. variable      : $name → hello world
# 3. word splitting: hello world → hello | world        (2 words)
# 4. globbing      : neither word has a wildcard → unchanged
# printf gets 2 arguments → [hello] and [world], one per line

printf '[%s]\n' "$name"
# 2. variable      : $name → hello world
# 3. word splitting: SKIPPED (double quotes)
# 4. globbing      : SKIPPED (double quotes)
# printf gets 1 argument → [hello world]

printf '[%s]\n' $pat
# 2. variable      : $pat → *.txt
# 3. word splitting: one word, no spaces → nothing to split
# 4. globbing      : *.txt matches a.txt and b.txt → replaced by both
# printf gets 2 arguments → [a.txt] and [b.txt]

printf '[%s]\n' "$pat"
# 2. variable      : $pat → *.txt
# 3. word splitting: SKIPPED (double quotes)
# 4. globbing      : SKIPPED (double quotes), so * stays a literal *
# printf gets 1 argument → [*.txt]

echo '$name'
# 1-4. ALL SKIPPED (single quotes): nothing is expanded or split
# echo gets the 5 characters $name → $name
```

#### All four stages in one command

Brace expansion runs first, so you can watch each stage hand its
result to the next:

```bash
printf '[%s]\n' {a,b}$pat
# 1. brace expansion: {a,b}$pat → a$pat b$pat
# 2. variable       : a$pat b$pat → a*.txt b*.txt
# 3. word splitting : no spaces → nothing to split
# 4. globbing       : a*.txt → a.txt     b*.txt → b.txt
# printf gets 2 arguments → [a.txt] and [b.txt]
```

#### Brace expansion and quotes

Quotes protect the braces, so they stay as plain text:

```bash
echo {a,b}        # a b        braces typed outside quotes → expanded
echo "{a,b}"      # {a,b}      double quotes → braces stay literal
echo '{a,b}'      # {a,b}      single quotes → braces stay literal
```

Quotes protect only what is inside them:

```bash
echo {a,b}-file       # a-file b-file     the braces are outside any quotes → expanded
echo "{a,b}-file"     # {a,b}-file        the braces are inside the quotes → literal
```

#### Brace expansion and variables

Brace expansion only works on braces you **type**, not on braces that
arrive inside a variable's value. Stage 1 is already finished by the
time stage 2 puts the value in:

```bash
echo {1,2}        # 1 2        typed directly → expanded

x="{1,2}"
echo $x           # {1,2}      the braces came from a variable → NOT expanded
echo "$x"         # {1,2}      same result when quoted
```

The same is true when you store braces in an assignment. The
assignment keeps them as plain text, and they stay plain text later:

```bash
x={a,b}
echo $x           # {a,b}      stored as text, never expanded
```

#### Quote removal: the quotes themselves disappear

Quotes are instructions to the shell, not part of your text. After
the shell has used them to decide which stages to skip, the last
stage removes them, so the command never sees them:

```bash
printf '[%s]\n' "hello"
# 5. quote removal : the quote characters are removed → printf gets hello (5 letters)
# [hello]            ← not ["hello"]

echo "a   b"
# 3. word splitting: SKIPPED (double quotes), so the 3 spaces are kept
# 5. quote removal : the quotes are removed
# a   b

echo 'say "hi"'
# the double quotes are inside single quotes, so they are plain text
# 5. quote removal : only the outer single quotes are removed
# say "hi"
```

Like brace expansion, quote removal only works on what you **type**.
Quotes that arrive inside a variable's value stay:

```bash
x='"hi"'
echo $x
# 2. variable      : $x → "hi"   (the quotes are part of the value)
# 3. word splitting: nothing to split     4. globbing: nothing to match
# 5. quote removal : acts only on quotes you typed, not on the ones inside the value
# "hi"
```

#### Which stages act on what

| Stage | Acts on the text you typed | Acts on the results of `$` expansions |
|---|---|---|
| 1. Brace expansion | Yes | No |
| 2. `~`, `$name`, `$(cmd)`, `$((math))` | Yes (this stage produces the results) | n/a |
| 3. Word splitting | n/a | Yes (unquoted only) |
| 4. Globbing | Yes | Yes (unquoted only) |
| 5. Quote removal | Yes | No |

This table explains most of the surprises in this demo. A variable's
value **is** split and globbed (stages 3 and 4), which is why unquoted
variables are risky. But braces and quotes inside a value are left
alone (stages 1 and 5). `~` follows the same rule as braces: `echo ~`
gives your home directory, but `echo "~"` and `v="~"; echo $v` both
print a plain `~`.

**Rule of thumb:** put double quotes around every variable
(`"$name"`). Use single quotes when you want text passed exactly as
written, such as a regex pattern.

### 1.6 Escaping: protecting one character

A backslash `\` before a character removes that character's special
meaning, for that one character only:

```bash
echo \*.txt
# 1-3. brace / variable / splitting: nothing to do
# 4. globbing      : the backslash marks * as literal → not a wildcard, SKIPPED
# 5. quote removal : the backslash itself is removed
# echo gets 1 argument → *.txt
```

Escaping is useful **outside quotes**. Inside double quotes, wildcards
are already literal, and backslash only has an effect before `$`, `` ` ``,
`"`, `\`, or a newline:

```bash
echo "\*.txt"
# 4. globbing      : SKIPPED (double quotes)
# 5. quote removal : removes the quotes. The backslash STAYS, because * is not on the list of characters a backslash can escape inside "..."
# echo gets 1 argument → \*.txt

echo "a\$b"
# 2. variable      : \$ is on the list, so the backslash stops $b from being expanded
# 5. quote removal : removes the quotes and the backslash
# echo gets 1 argument → a$b
```

### 1.7 Why this is a production footgun

The dangerous pattern always has the same shape: **an unquoted
variable that should hold one filename ends up holding a wildcard
character or a space.** The shell cannot tell "a name that happens to
contain `*`" from "a pattern to expand", so it expands it, and the
command runs with full confidence on whatever that produced.

#### Why the wildcard is not expanded when it is stored

In the footgun, a variable ends up holding `session_*.tmp`, yet
nothing is expanded at that moment. Expansion applies to the words of
a command. The right-hand side of an assignment (`var=value`) gets
variable expansion, but **no word splitting, no globbing and no brace
expansion**. The value is stored as plain text:

```bash
tmpfile=session_*.tmp
# assignment: only variable expansion (and quote removal) apply
# 3. word splitting: NOT applied
# 4. globbing      : NOT applied
# stores the text session_*.tmp, even if matching files exist

x={a,b}
# 1. brace expansion: NOT applied in an assignment
# stores the text {a,b}
```

The glob runs later, at the moment the variable is **used unquoted as
a command argument**. The same stored text gives different results
depending on how it is used:

| Moment | What the shell does with the text `session_*.tmp` |
|---|---|
| `tmpfile=session_${session_id}.tmp` (assignment) | Stores it as plain text |
| `rm "$tmpfile"` | Passes the text unchanged, so `rm` looks for a file with that exact name |
| `rm $tmpfile` | Word-splits and globs it, so `rm` receives every matching filename |

Walking the two uses through the stages (files `session_42.tmp`,
`session_43.tmp` and `session_44.tmp` exist):

```bash
rm -v $tmpfile
# 2. variable      : $tmpfile → session_*.tmp
# 3. word splitting: one word, no spaces → nothing to split
# 4. globbing      : session_*.tmp matches session_42.tmp session_43.tmp session_44.tmp
# rm gets 3 arguments → deletes all three

rm -v "$tmpfile"
# 2. variable      : $tmpfile → session_*.tmp
# 3-4. SKIPPED (double quotes)
# rm gets 1 argument → the literal name session_*.tmp → "No such file or directory"
```

This is why quoting belongs at the point of **use**: the stored value
is harmless until an unquoted expansion turns it into a pattern.

---

## Part 2 — Hands-On Lab

### Commands used in this demo

| Command | Flag meaning | Purpose |
|---|---|---|
| `echo` | n/a | Prints the words the shell produced, the standard way to *see* an expansion before using it |
| `printf '[%s]\n'` | n/a | Prints each argument in brackets on its own line, so you can count arguments |
| `touch` | n/a | Creates empty files (or updates timestamps) |
| `mkdir -p` | `p` = parents | Creates a directory and any missing parents, no error if it exists |
| `rm -v` | `v` = verbose | Deletes files and prints each one removed |

### Step 0 — Make sure you are in bash

```bash
echo $0
```

If the output is `bash` (or `-bash`), continue. If it is `zsh`,
switch to bash for this lab:

```bash
bash
```

Run `echo $0` again to confirm. Type `exit` at the end of the lab to
return to your normal shell.

> **Why this matters:** zsh behaves differently from bash for this
> topic. It prints `no matches found` instead of passing a no-match
> pattern through, reads `!` as history expansion, and does not split
> or glob unquoted variables. The results below are bash results.

### Task 1 — Globbing (Concepts §1.2)

```bash
# from the demo directory: 01-globbing-quoting/
mkdir -p src/glob-lab && cd src/glob-lab
touch file.txt file1.txt file2.txt file3.log notes.md report_2024.csv report_2025.csv .hidden
ls -a
```

Run each line and compare with the expected output:

```bash
echo *.txt                # file.txt file1.txt file2.txt
echo file?.txt            # file1.txt file2.txt
echo file*.txt            # file.txt file1.txt file2.txt
echo file[12].txt         # file1.txt file2.txt
echo file[!1].txt         # file2.txt
echo file[1-3].*          # file1.txt file2.txt file3.log
echo file[a-c].txt        # file[a-c].txt  (a range with no matches: passed through literally)
echo report_202[45].csv   # report_2024.csv report_2025.csv
echo *                    # lists every name except .hidden (project is a directory, so it is listed too)
echo .*                   # .hidden
echo *.xyz                # *.xyz
```

**Observations:**

- `file?.txt` skips `file.txt` (`?` needs exactly one character) but `file*.txt` includes it (`*` allows zero).
- `*` skipped `.hidden`; only a pattern starting with `.` matched it. On bash older than 5.2 (for example Rocky 9), `echo .*` also lists `.` and `..`.
- `echo *.xyz` matched nothing, so the pattern came back as literal text. Remember this for Task 5.

### Task 2 — Brace expansion (Concepts §1.3)

Build it up one step at a time:

```bash
echo {a,b}                      # a b
echo x{a,b}y                    # xay xby
echo {1..5}                     # 1 2 3 4 5
echo {a..e}                     # a b c d e
echo {a,b}{1,2}                 # a1 a2 b1 b2
echo {1,2}{a,b}                 # 1a 1b 2a 2b
echo file{9,10,11}.nonexistent  # file9.nonexistent file10.nonexistent file11.nonexistent
```

The last command produced names even though none of those files
exist. That is brace expansion working without the filesystem.

Glob range vs. brace range (Concepts §1.2). Same files, different mechanism:

```bash
echo file[1-3].txt        # file1.txt file2.txt             (glob: only files that exist)
echo file{1..3}.txt       # file1.txt file2.txt file3.txt   (brace: generated, no check)
echo file[1..5].txt       # file1.txt                       (NOT a range: matches the characters 1 . or 5)
```

The last line is a common mistake. `[1..5]` is a bracket set, so it
matched `file1.txt` only because `file1.txt` exists, not because 1 to
5 was generated. Use `{1..5}` to generate a sequence.

Practical uses:

```bash
mkdir -p project/{src,docs,tests}
ls project                      # docs  src  tests

touch app.conf
cp app.conf{,.bak}
ls app.conf*                    # app.conf  app.conf.bak
```

Prove the order (brace first, then glob):

```bash
echo nomatch{a,b}*.zzz
# 1. brace expansion: → nomatcha*.zzz nomatchb*.zzz   (2 patterns)
# 4. globbing       : neither pattern matches a file → both left as typed
# nomatcha*.zzz nomatchb*.zzz
```

Two separate patterns means the braces expanded first.

### Task 3 — Word splitting and quoting (Concepts §1.4 and §1.5)

```bash
mkdir -p ../quoting && cd ../quoting
name="hello world"
```

**Count the arguments with printf:**

```bash
printf '[%s]\n' $name
# 2. variable      : $name → hello world
# 3. word splitting: → hello | world   (2 words)
# 4. globbing      : none
# [hello]
# [world]

printf '[%s]\n' "$name"
# 2. variable      : $name → hello world
# 3-4. SKIPPED (double quotes)
# [hello world]
```

**See the real consequence with `touch`:**

```bash
touch $name
# 2. variable      : $name → hello world
# 3. word splitting: → hello | world   (2 words)
# touch gets 2 arguments → creates 2 files
ls
# hello  world            ← two files, not the one you meant

rm hello world

touch "$name"
# 2. variable      : $name → hello world
# 3-4. SKIPPED (double quotes) → touch gets 1 argument → creates 1 file
ls
# 'hello world'           ← one file with a space in its name
```

In a terminal, `ls` wraps names that contain spaces in single quotes.
The quotes are `ls` formatting, not part of the file name.

**Wildcards stored in a variable:**

```bash
touch a.txt b.txt
pat="*.txt"

printf '[%s]\n' $pat
# 2. variable: $pat → *.txt   3. splitting: nothing to split
# 4. globbing: *.txt → a.txt b.txt    → [a.txt] [b.txt]

printf '[%s]\n' "$pat"
# 2. variable: $pat → *.txt   3-4. SKIPPED (double quotes)
# → [*.txt]
```

**All three quote styles side by side:**

```bash
echo $name '$name' "$name"
# hello world $name hello world

echo "today is $(echo Thursday)" 'today is $(echo Thursday)'
# today is Thursday today is $(echo Thursday)
```

Double quotes still ran `$(echo Thursday)`; single quotes did not.

### Task 4 — Escaping (Concepts §1.6)

```bash
echo \*.txt        # 4. globbing SKIPPED (escaped *), 5. backslash removed     → *.txt
echo "\*.txt"      # 4. SKIPPED (quotes); * is not escapable inside "...", so the
                   #    backslash STAYS                                           → \*.txt
echo '\*.txt'      # 1-4. ALL SKIPPED (single quotes): nothing is processed       → \*.txt
echo "a\$b"        # 2. \$ stops $b being expanded; 5. backslash removed          → a$b
```

### Task 5 — The `rm` footgun, reproduced and fixed (Concepts §1.7)

```bash
mkdir -p ../cleanup-demo && cd ../cleanup-demo
touch session_42.tmp session_43.tmp session_44.tmp important_data.tmp invoice_2024.tmp
ls
# important_data.tmp  invoice_2024.tmp  session_42.tmp  session_43.tmp  session_44.tmp
```

A script builds a filename from a session ID, but the ID was never
filled in properly and holds `*`:

```bash
session_id="*"
tmpfile=session_${session_id}.tmp
echo "tmpfile holds: [$tmpfile]"
# tmpfile holds: [session_*.tmp]
```

The wildcard is still plain text: assignment does not glob (Concepts
§1.7). Expansion happens when the variable is *used*. Compare the two
uses with `echo`, which is a safe way to look before you delete:

```bash
echo "$tmpfile"
# 2. variable: $tmpfile → session_*.tmp   3-4. SKIPPED (double quotes)
# session_*.tmp                                    ← quoted: unchanged

echo $tmpfile
# 2. variable: → session_*.tmp   3. splitting: nothing to split
# 4. globbing: session_*.tmp → 3 matching names
# session_42.tmp session_43.tmp session_44.tmp     ← unquoted: expanded now
```

**The safe version first (quoted):**

```bash
rm -v "$tmpfile"
# rm: cannot remove 'session_*.tmp': No such file or directory

ls
# all five files still present
```

Quoted, `rm` looked for a file literally named `session_*.tmp`,
found nothing, and failed loudly.

**Now the bug (unquoted):**

```bash
rm -v $tmpfile
# removed 'session_42.tmp'
# removed 'session_43.tmp'
# removed 'session_44.tmp'

ls
# important_data.tmp  invoice_2024.tmp
```

The shell expanded `session_*.tmp` against real files, so `rm`
received three valid filenames and deleted all three. Exit code `0`,
no error.

**Clean up and leave bash (optional, since `src/` is git-ignored):**

```bash
cd ../..                  # back to 01-globbing-quoting/
rm -rf src/glob-lab src/quoting src/cleanup-demo
exit                      # only if you started bash from zsh in Step 0
```

---

## What You Learned

- ✅ Confirmed which shell you were in, and why zsh gives different results
- ✅ Used `*`, `?`, `[...]` (including ranges like `[1-3]`) and `[!...]`, and saw that dotfiles are skipped and non-matching patterns pass through literally
- ✅ Built brace expansions step by step, including combined groups like `{a,b}{1,2}`, and proved it runs before globbing
- ✅ Told a glob range (`[1-3]`) from a brace range (`{1..3}`), and saw why `[1..5]` is not a range
- ✅ Saw word splitting turn one variable into several arguments, using `printf` and `touch`
- ✅ Compared unquoted, double-quoted and single-quoted behavior side by side, including that quotes switch brace expansion off
- ✅ Saw that braces typed outside quotes expand, but braces inside a variable's value never do
- ✅ Named all five stages in order (brace, `$`/`~`, word splitting, globbing, quote removal) and saw which act on typed text and which act on a variable's value
- ✅ Escaped single characters and saw where backslash does and does not work
- ✅ Saw that assignment stores a wildcard as plain text and that expansion happens at the point of use
- ✅ Reproduced the unquoted-variable `rm` footgun and fixed it with double quotes

**Key Takeaway:** quoting decides whether a variable is treated as
exactly the text it contains, or as something the shell may split and
expand. Put double quotes around every variable expansion unless you
specifically want splitting or globbing.

---

## Part 3 — Break-Fix Scenario

### Scenario — NovaTech nightly cleanup job deletes the wrong files

You're on call. The alert:

> *"The nightly cleanup job is supposed to delete ONE session's
> leftover temp file. Last night it deleted the temp files of all
> three sessions, including `session_44.tmp`, which a batch job was
> still writing to. Nobody changed the script. Only the session that
> ran last night changed."*

The relevant lines from the script:

```bash
tmpfile=session_${SESSION_SUFFIX}.tmp
rm -v $tmpfile
```

The logs show `SESSION_SUFFIX` was set to `*`, a placeholder left over
when an upstream step failed to fill in a real session ID.

**Your task (answer before opening the resolution):**

1. Why did `rm -v $tmpfile` delete three files instead of failing?
2. What single change to this line would have made last night's failure loud and harmless?
3. Is validating `SESSION_SUFFIX` upstream a replacement for quoting, or a separate fix?

<details>
<summary>Resolution — open after attempting</summary>

**1. Why three files were deleted:**

`tmpfile` held the text `session_*.tmp`. Unquoted, the shell
glob-expanded it against the directory before `rm` ran, so `rm`
received `session_42.tmp session_43.tmp session_44.tmp`. From `rm`'s
point of view those were three valid files it was asked to delete, so
it did, with no error.

**2. The single-line fix:**

```bash
rm -v "$tmpfile"
```

Quoted, the `*` stays literal. `rm` looks for a file named
`session_*.tmp`, does not find it, and fails with "No such file or
directory", exactly what Task 5 showed.

**3. Two separate fixes, ship both:**

Quoting limits the damage from *any* bad value, but it does not know
why the value was bad. Validating `SESSION_SUFFIX` upstream (reject
empty values and anything containing `*`, `?` or `[`) fixes the root
cause, a failed step silently producing a placeholder.

**The lesson:** an unquoted variable holding a wildcard can match
every file that fits the pattern, and the command runs as if that were
exactly what you asked for.

</details>

---

## Interview Prep

**Q1. What's the difference between `*` in a glob and `*` in a regular expression?**

They are unrelated despite the shared symbol. A glob `*` means "zero
or more of any character" and is expanded by the shell against
filenames before the command runs. A regex `*` means "zero or more of
the preceding element" and is evaluated by a regex engine such as
`grep`, `sed` or `awk`. For example, the glob `.*` matches names that
start with a dot, while the regex `.*` means "any character, repeated".
The regex demos cover this in depth.

---

**Q2. A teammate writes `rm $file` in a script. What's your concern, and what would you change?**

Unquoted, `$file` is subject to word splitting and globbing. A space
turns it into multiple arguments, and a wildcard makes `rm` delete
every match instead of one file. I'd change it to `rm "$file"`.

---

**Q3. Why does `echo *.nomatch` print `*.nomatch` instead of nothing or an error?**

When a glob matches nothing, bash passes the pattern through as
literal text. A script that receives a literal wildcard as an argument
is a strong sign an expected file is missing, not that zero files were
"correctly" selected. (`shopt -s failglob` makes bash error instead.)

---

**Q4. When would you deliberately use single quotes over double quotes?**

When the text must reach the command exactly as written, most
commonly a regex passed to `grep`, `sed` or `awk`. Double quotes would
let the shell expand `$1` or `` `cmd` `` inside the pattern before the
tool sees it.

---

## Key Takeaways

1. **The shell rewrites your command before running it, in a fixed order: brace expansion → variable expansion → word splitting → globbing (then quote removal).** Globbing replaces wildcards with matching filenames, and a command receiving a literal wildcard usually means nothing matched.
2. **Brace expansion generates text and never checks the filesystem, and it runs before globbing.** `{a,b}{1,2}` pairs every item of the first group with every item of the second (`a1 a2 b1 b2`). Glob ranges use a hyphen inside brackets and match one character (`[1-3]`); brace ranges use two dots and generate values (`{1..3}`). `[1..5]` is not a range. Quotes switch brace expansion off, and it only works on braces you type, never on braces inside a variable's value.
3. **Word splitting cuts an unquoted variable's value into separate words at spaces**, and each word can then be glob-expanded.
4. **Double quotes are the safe default.** They keep `$variable` and `$(command)` working but turn off word splitting and globbing. Single quotes turn off everything.
5. **A backslash affects only the one character after it**, and inside double quotes it only works before `$`, `` ` ``, `"`, `\` or a newline.
6. **Assignment stores text; expansion happens at use.** `var=session_*.tmp` does not glob. The wildcard is expanded when the variable is used unquoted as a command argument.
7. **The footgun is always the same: an unquoted variable meant to hold one filename holds a wildcard or a space.** Quote it as a blast-radius control, and validate the value to fix the root cause.

---

## Quick Commands Reference

### Glob wildcards

| Wildcard | Matches |
|---|---|
| `*` | Zero or more of any character |
| `?` | Exactly one of any character |
| `[abc]` / `[a-z]` | One character from the set or range (hyphen = range; `..` is not) |
| `[!abc]` | One character NOT in the set |

### Brace expansion forms

| Form | Expands to |
|---|---|
| `{a,b,c}` | `a b c` |
| `x{a,b}y` | `xay xby` |
| `{1..5}` | `1 2 3 4 5` (brace range: two dots) |
| `{a,b}{1,2}` | `a1 a2 b1 b2` |
| `file{,.bak}` | `file file.bak` |

### Quote types

| Style | Braces expand | Variable expands | Command substitution | Split / glob |
|---|---|---|---|---|
| Unquoted | Yes | Yes | Yes | Yes |
| `"double"` | No | Yes | Yes | No |
| `'single'` | No | No | No | No |

---

## Appendix A — Anki Cards

**`m01-01-globbing-quoting-anki.csv`:**

```
#deck:Linux Mastery::Module 1 - Filesystem & Navigation::Demo M01-01 - Globbing & Quoting
#separator:Comma
#columns:Front,Back,Tags

"In bash, echo *.xyz matches no files. What is printed, and why does it matter?","The literal text *.xyz. Bash passes a non-matching pattern through unchanged. A command that receives a literal wildcard as an argument usually means nothing matched.","demo-m01-01,globbing,nomatch"

"Why does file?.txt match file1.txt but not file.txt, while file*.txt matches both?","? needs exactly one character; file.txt has none between 'file' and '.txt'. * allows zero or more characters, so it matches both.","demo-m01-01,globbing,wildcards"

"What does echo {a,b}{1,2} print, and how is it built?","a1 a2 b1 b2. Each item of the first group is paired with every item of the second group.","demo-m01-01,brace,combinations"

"What does cp app.conf{,.bak} expand to?","cp app.conf app.conf.bak. The first brace item is empty, so it gives app.conf, and the second adds .bak. A quick backup idiom.","demo-m01-01,brace,backup"

"What is the key difference between brace expansion and globbing?","Brace expansion generates text and never checks the filesystem, so it works for files that don't exist. Globbing only expands to files that actually exist. Brace expansion runs first.","demo-m01-01,brace,glob,order"

"What is word splitting?","After an unquoted variable is replaced by its value, bash cuts that value into separate words at spaces (IFS). name=""hello world"" used as $name becomes two arguments.","demo-m01-01,wordsplitting"

"What does double-quoting a variable suppress, and what does it still allow?","It suppresses word splitting and globbing. $variable and $(command) still expand.","demo-m01-01,quoting,doublequote"

"What does single-quoting suppress that double-quoting does not?","Variable expansion and command substitution. Nothing inside single quotes is interpreted by the shell.","demo-m01-01,quoting,singlequote"

"echo ""\*.txt"" prints \*.txt with the backslash. Why?","Inside double quotes, backslash only works before $, `, "", \ or a newline. * is not one of them, and wildcards are already literal inside double quotes.","demo-m01-01,escaping,doublequote"

"A variable holds session_*.tmp and the script runs rm -v $tmpfile. What happens, and what is the fix?","The shell glob-expands the unquoted variable, so rm deletes every matching file with no error. Fix: rm -v ""$tmpfile"", which fails loudly instead. Also validate the value upstream.","demo-m01-01,quoting,footgun,rm"

"A variable is assigned tmpfile=session_*.tmp while matching files exist. Why isn't the wildcard expanded at assignment?","The right-hand side of an assignment gets variable expansion but no word splitting, globbing or brace expansion, so the text is stored as-is. Globbing happens later, when the variable is used unquoted as a command argument (rm $tmpfile).","demo-m01-01,assignment,globbing,footgun"

"What is the difference between file[1-3].txt and file{1..3}.txt, and what does file[1..5].txt match?","[1-3] is a glob range: one character from 1 to 3, matched only against existing files. {1..3} is a brace range: generates file1.txt file2.txt file3.txt with no filesystem check. [1..5] is not a range; it is a set of the characters 1, . and 5.","demo-m01-01,globbing,brace,range"

"echo {a,b} prints a b. What do echo ""{a,b}"" and echo '{a,b}' print, and why?","Both print {a,b}. Quotes (double or single) switch brace expansion off, so the braces stay literal. Quotes protect only what is inside them: echo {a,b}-file gives a-file b-file, but echo ""{a,b}-file"" gives {a,b}-file.","demo-m01-01,brace,quotes"

"What is quote removal, and when does it NOT apply?","The last expansion stage: bash removes the quote characters and backslashes you typed, so the command never sees them (printf '[%s]\n' ""hello"" prints [hello]). It only acts on text you typed: quotes that arrive inside a variable's value stay (x='""hi""'; echo $x prints ""hi"").","demo-m01-01,quoting,quote-removal"

"In what order does bash apply brace expansion, variable expansion, word splitting and globbing, and what is the proof that brace expansion runs before variable expansion?","1) brace expansion, 2) variable (and command) expansion, 3) word splitting, 4) globbing, then quote removal. Proof: x=""{1,2}""; echo $x prints {1,2}, because braces arriving inside a variable's value are never expanded (brace expansion already ran).","demo-m01-01,expansion,order"

"A script starts with #!/bin/sh and uses {a,b}. On Ubuntu it doesn't expand. Why?","On Ubuntu/Debian /bin/sh is dash, which has no brace expansion. Use #!/bin/bash.","demo-m01-01,shell,dash,portability"

"Why do several of this demo's commands give different results in zsh than in bash?","zsh errors on no-match patterns, treats ! as history expansion, and does not word-split or glob unquoted variables. Run the lab in bash.","demo-m01-01,shell,zsh,portability"
```

---

## Appendix B — Quiz

**`m01-01-globbing-quoting-quiz.md`:**

```
# Quiz — Demo M01-01: Wildcards, Globbing & Quoting

> One correct answer per question. Target: 80% (10/12) before Demo M01-02.
> Assume bash.

---

**Q1.** `echo report_*.csv` is run in a directory with no matching
files. What is printed?

- A) Nothing, an empty line
- B) An error: "no matches found"
- C) The literal text `report_*.csv`
- D) Every file in the directory

<details>
<summary>Answer</summary>

**C**. Bash passes a non-matching pattern through unchanged.

Trap: B is zsh's default behavior. In bash it only happens if
`failglob` is enabled.

</details>

---

**Q2.** What does `echo {a,b}{1,2}` print?

- A) `a b 1 2`
- B) `a1 b2`
- C) `a1 a2 b1 b2`
- D) `a1 b1 a2 b2`

<details>
<summary>Answer</summary>

**C**. Each item of the first group is paired with every item of the
second group, and the first group changes slowest. D has the right
combinations in the wrong order.

</details>

---

**Q3.** What is the key difference between brace expansion and globbing?

- A) They are the same mechanism with different syntax
- B) Brace expansion generates text without checking the filesystem; globbing only matches files that exist
- C) Globbing runs before brace expansion
- D) Brace expansion only works inside double quotes

<details>
<summary>Answer</summary>

**B**. `echo file{9,10}.nonexistent` expands both names even though
neither file exists. Brace expansion also runs before globbing, so C
is backwards.

</details>

---

**Q4.** `name="hello world"`. What does `printf '[%s]\n' $name` print?

- A) `[hello world]`
- B) `[hello]` and `[world]` on separate lines
- C) An error, since variables cannot contain spaces
- D) `[$name]`

<details>
<summary>Answer</summary>

**B**. Unquoted, the value is word-split at the space, so printf
receives two arguments. A is what `"$name"` would give.

</details>

---

**Q5.** A script runs `rm -v $tmpfile` and `$tmpfile` has become
`session_*.tmp`. Which single change makes this fail loudly instead
of deleting every matching file?

- A) `rm -vf $tmpfile`
- B) `rm -v "$tmpfile"`
- C) `rm -v ${tmpfile}`
- D) `rm -v $tmpfile*`

<details>
<summary>Answer</summary>

**B**. Quoting keeps the `*` literal, so `rm` looks for a file with
that exact name and reports "No such file or directory".

Trap: C only adds braces around the variable name; it does not stop
splitting or globbing.

</details>

---

**Q6.** What does single-quoting suppress that double-quoting does NOT?

- A) Globbing
- B) Word splitting
- C) Variable expansion and command substitution
- D) Nothing, they behave the same

<details>
<summary>Answer</summary>

**C**. Both suppress splitting and globbing. Only single quotes also
stop `$var` and `$(cmd)` from being evaluated.

</details>

---

**Q7.** `echo "\*.txt"` prints `\*.txt`, backslash included. Why?

- A) Backslash is not valid in bash
- B) Inside double quotes backslash only works before `$`, `` ` ``, `"`, `\` or a newline, and `*` is not one of them
- C) This is a bug in `echo`
- D) Double quotes always print backslashes

<details>
<summary>Answer</summary>

**B**. Wildcards are already literal inside double quotes, and `*` is
not on the short list of characters backslash can escape there. (`"\$"`
does work, since `$` is on the list.)

</details>

---

**Q8.** A script starts with `#!/bin/sh` and uses `{a,b}`. On Ubuntu
the braces are not expanded. Why?

- A) `/bin/sh` is `dash`, which does not support brace expansion
- B) Brace expansion requires root
- C) `/bin/sh` requires braces to be escaped
- D) The premise is wrong, this works fine

<details>
<summary>Answer</summary>

**A**. Brace expansion is a bash feature, not POSIX `sh`. Use
`#!/bin/bash`.

</details>

---

**Q9.** `tmpfile=session_*.tmp` is run in a directory containing
`session_42.tmp` and `session_43.tmp`. What does `echo "$tmpfile"` print?

- A) `session_42.tmp session_43.tmp`
- B) `session_*.tmp`
- C) An empty line
- D) An error

<details>
<summary>Answer</summary>

**B**. An assignment stores plain text: no globbing happens on the
right-hand side. The quoted `"$tmpfile"` keeps it that way.

Trap: A is what the *unquoted* `echo $tmpfile` prints, because
globbing happens when the variable is used, not when it is assigned.

</details>

---

**Q10.** Only `file1.txt` and `file2.txt` exist. What do
`echo file[1-3].txt` and `echo file{1..3}.txt` print?

- A) Both print `file1.txt file2.txt file3.txt`
- B) `file1.txt file2.txt` and `file1.txt file2.txt file3.txt`
- C) `file1.txt file2.txt file3.txt` and `file1.txt file2.txt`
- D) Both print `file1.txt file2.txt`

<details>
<summary>Answer</summary>

**B**. `[1-3]` is a glob range, matched only against existing files.
`{1..3}` is a brace range, which generates all three names without
checking the filesystem.

Note: `file[1..5].txt` is not a range at all. `[1..5]` is a set of
the characters `1`, `.` and `5`.

</details>

---

**Q11.** What do `echo {a,b}`, `echo "{a,b}"` and
`x="{1,2}"; echo $x` print, in that order?

- A) `a b`, `a b`, `1 2`
- B) `a b`, `{a,b}`, `{1,2}`
- C) `{a,b}`, `{a,b}`, `{1,2}`
- D) `a b`, `{a,b}`, `1 2`

<details>
<summary>Answer</summary>

**B**. Braces typed outside quotes expand. Double quotes switch brace
expansion off, so the braces stay literal. Braces that arrive inside a
variable's value are never expanded, because brace expansion (stage 1)
had already finished before `$x` was replaced (stage 2).

</details>

---

**Q12.** `x='"hi"'` stores the text `"hi"` with its double quotes. What
does `echo $x` print?

- A) `hi`
- B) `"hi"`
- C) `'hi'`
- D) An error

<details>
<summary>Answer</summary>

**B**. Quote removal (stage 5) only removes quotes you typed. The
quotes inside the variable's value are plain text, so they stay.

</details>

---

Score guide:

| Score | Action |
|---|---|
| 12/12 | Import Anki cards, move to Demo M01-02 |
| 11/12 | Review the wrong answer, then proceed |
| 9-10/12 | Re-read the relevant section, retry those questions |
| Below 9/12 | Re-read the full demo and redo the lab |
```