# Demo M01-03 — Environment & PATH

## Overview

Type a command name and the shell has to decide, in a fixed order,
what actually runs: an alias, a function, a builtin, or a file found
through `PATH`. Most of the time the answer is obvious. The times it
is not are exactly where this knowledge pays off: a teammate's alias
silently shadowing a real command, a cached lookup running an old
binary after an upgrade, or `which` pointing at a file that is not
what executes.

This demo covers the pieces in the order the shell uses them:

| Concept | The question it answers |
|---|---|
| Environment variables | What does a program I launch inherit from my shell? |
| `PATH` | Which directories are searched for a command, and in what order? |
| Resolution order | Alias, function, builtin or file: which one wins? |
| Alias vs. function | Why does my alias work in the terminal but not in a script? |
| `which` vs. `type` | How do I find out what will *actually* run? |
| Command hashing | Why does the old version still run after I installed a new one? |

---

## Demo Objectives

By the end of this demo you will be able to:

1. Explain the difference between a plain shell variable and an exported environment variable, and why child processes only receive exported ones
2. Read and modify `PATH`, and explain why prepending vs. appending changes priority
3. State the command-resolution order (alias → function → builtin → hashed location → `PATH` search) and demonstrate it with `type -a`
4. Explain why an alias works in a terminal but not in a script, while a function works in both, and bypass or remove an alias or function
5. Explain the difference between `which`, `type` and `command -v`, and why `which` can report a file that is not what runs
6. Explain what bash's command hash table is, and the two ways it goes stale (a new copy earlier in `PATH`, and a moved or removed file)
7. Explain why replacing a file *at the same path* does **not** leave a stale hash, and fix real stale cases with `hash -r`

---

## Directory Structure

```
03-environment-path/
├── README.md                         # this file
├── m01-03-environment-path-anki.csv  # Anki flashcard deck (Appendix A)
├── m01-03-environment-path-quiz.md   # standalone quiz (Appendix B)
└── src/                              # lab working directories (git-ignored)
    ├── pathlab/                      # Tasks 2 and 5: PATH priority, which vs. type
    │   ├── dir1/
    │   └── dir2/
    └── hashlab/                      # Tasks 6 and 7: command hashing
        ├── dir1/
        └── dir2/
```

All lab work happens inside `src/`. The lab commands create the
folders (`mkdir -p`), so you only need to be in the demo directory
(`03-environment-path/`) when you start.

> **Note:** `src/` must be in the repository's root `.gitignore`. If
> you did not add it in Demo M01-01, add this line once:
>
> ```
> src/
> ```

---

## Environments

| Task | Ubuntu 24.04 | Rocky Linux 9.8 | EC2 Amazon Linux |
|---|---|---|---|
| All lab tasks | ✅ | ✅ | Not required |

The resolution mechanics are bash behavior and identical on both
distros. Some practical points:

- **Run the lab in a child bash.** The lab defines aliases and functions and edits `PATH`. Step 0 starts a child shell so that `exit` at the end restores everything. This is also why the lab never shadows a real command such as `ls`.
- **Rocky's `which` is different in a terminal.** On Rocky 9, an interactive shell defines `which` as a *function* that passes your aliases and functions to `/usr/bin/which`, so plain `which` there *can* see aliases. Ubuntu's `which` cannot. The lab uses `command which`, which skips that function and runs the real program, giving the same result on both.
- **Rocky has a file called `/usr/bin/cd`.** `which cd` prints it there, while on Ubuntu it prints nothing. The lab uses `shopt` as its builtin example, which has no file on either distro. §1.5 uses the Rocky `cd` as an example.
- Default `PATH` contents differ slightly by distro.

---

## Recall Check — Demo M01-02

Before starting, answer from memory:

1. `cmd 2>&1 > file`: does stderr end up in `file`? Why or why not?
2. `grep -q pattern file` exits `1`. Is that an error?
3. What does `xargs -I{}` change about how input is split, and why should it not be combined with `-n`?

<details>
<summary>Answers</summary>

1. No. `2>&1` copies where stdout points *at that moment* (still the terminal), and then only stdout moves to the file. `> file 2>&1` captures both.
2. Not necessarily. For `grep`, `1` means no match; `2` means an error.
3. It reads one whole line per run instead of splitting on whitespace. Combined with `-n`, `xargs` warns that the options are mutually exclusive and ignores `-n`.

</details>

---

## Part 1 — Concepts

### 1.1 Environment variables: plain vs. exported

A plain shell variable exists only in the current shell. An
**exported** variable is also copied into the environment of every
child process the shell starts:

```bash
myvar="not exported"
bash -c 'echo child sees: [$myvar]'
# child sees: []            ← the child never saw it

export myvar2="exported"
bash -c 'echo child sees: [$myvar2]'
# child sees: [exported]    ← copied into the child's environment
```

`PATH`, `HOME`, `USER` and `EDITOR` are exported for exactly this
reason: every program you launch needs its own copy.

Three related facts:

- **It is a copy.** A child can change its copy, but the parent never sees the change.
- **One-shot variables:** `VAR=value command` gives `command` the variable for that single run only; your shell is unchanged.
- **Inspecting and removing:** `printenv NAME` (or `env`) shows exported variables, and `unset NAME` removes a variable.

```bash
FOO=bar bash -c 'echo [$FOO]'   # [bar]
echo "[$FOO]"                   # []   ← the shell never got it

export level=1
bash -c 'level=2; echo "child: $level"'   # child: 2
echo "parent: $level"                      # parent: 1
```

### 1.2 `PATH`: an ordered, colon-separated search list

```bash
echo "$PATH" | tr ':' '\n'
# /home/user/.local/bin
# /usr/local/sbin
# /usr/local/bin
# /usr/sbin
# /usr/bin
# /sbin
# /bin
```

`tr ':' '\n'` puts each directory on its own line, which is far easier
to read than the raw colon-separated string.

When you type a bare command name, bash searches these directories
**left to right** and uses the **first** match. Order is priority:

```bash
PATH="/new/priority/dir:$PATH"   # PREPEND: checked first, wins ties
PATH="$PATH:/fallback/dir"       # APPEND: checked last, only a fallback
```

A directory at the front can override a system-installed tool of the
same name; a directory at the back is used only when nothing earlier
has that name.

A `PATH` change like this lasts only for the current shell session.
Making it permanent is the job of the shell startup files (Demo M01-04).

### 1.3 The command-resolution order

When you type a name, bash checks these in order and stops at the
first match:

| Order | Type | Notes |
|---|---|---|
| 1 | Alias | Only expanded in interactive shells by default (§1.4) |
| 2 | Function | Works in any shell |
| 3 | Builtin | Part of bash itself, such as `cd`, `echo`, `shopt` |
| 4 | Hashed location | Bash's cached result of an earlier `PATH` search (§1.6) |
| 5 | `PATH` search | Left to right, first match wins |

Shell **keywords** such as `if`, `while` and `[[` are part of the
shell's grammar and are recognized by its parser, not looked up like
commands. `type -t if` reports `keyword`.

A name can exist at several levels at once. `echo` is both a bash
builtin and a file (`/usr/bin/echo`); the builtin wins:

```bash
type -a echo
# echo is a shell builtin
# echo is /usr/bin/echo
# echo is /bin/echo        ← listed only where /bin and /usr/bin are both in PATH
```

`type -a` lists every match in the order bash would consider them,
which makes it the quickest way to *see* the resolution order.

### 1.4 `alias` vs. function

#### Aliases work in a terminal but not in a script

In a terminal (an **interactive** shell) an alias works immediately:

```bash
alias greet="echo ALIAS-VERSION"
greet
# ALIAS-VERSION
```

The same alias inside a script or `bash -c` (a **non-interactive**
shell) is silently ignored, because aliases are only expanded in
interactive shells by default:

```bash
bash -c 'alias greet="echo ALIAS-VERSION"
greet'
# bash: line 2: greet: command not found
```

Turn alias expansion on with `shopt -s expand_aliases`:

```bash
bash -c 'shopt -s expand_aliases
alias greet="echo ALIAS-VERSION"
greet'
# ALIAS-VERSION
```

**Gotcha: an alias takes effect from the next line.** Bash reads a
whole line before running it, so defining and using an alias on the
same line (with `;`) fails even with `expand_aliases` on:

```bash
bash -c 'shopt -s expand_aliases; alias greet="echo ALIAS-VERSION"; greet'
# bash: line 1: greet: command not found
```

#### A function works everywhere

```bash
bash -c 'greet() { echo FUNCTION-VERSION; }
greet'
# FUNCTION-VERSION        ← no shopt needed
```

This is why scripts should use functions, not aliases, for
shortcuts.

#### When both exist, the alias wins

```bash
hello() { echo "FUNCTION-VERSION"; }
alias hello="echo ALIAS-VERSION"
hello
# ALIAS-VERSION

type -a hello
# hello is aliased to `echo ALIAS-VERSION'
# hello is a function
# hello ()
# {
#     echo "FUNCTION-VERSION"
# }
```

Define the function **before** the alias. If the alias already exists,
bash expands it inside the function definition and reports
`syntax error near unexpected token '('`.

#### Bypassing and removing an alias or function

| Form | Effect |
|---|---|
| `\hello` | Skips the **alias** (a leading backslash); the function still runs |
| `command hello` | Skips the alias **and** the function; runs a builtin or a file |
| `unalias hello` | Removes the alias |
| `unset -f hello` | Removes the function |

This is how you get the real command when a teammate's alias is
shadowing it, for example `\ls` or `command ls`.

### 1.5 `which` vs. `type` vs. `command -v`

| | `which` (the program) | `type` | `command -v` |
|---|---|---|---|
| Sees aliases | No | Yes | Yes |
| Sees functions | No | Yes | Yes |
| Sees builtins | No | Yes | Yes |
| Searches `PATH` | Yes | Yes | Yes |
| Reports what actually runs | **No** | Yes | Yes |
| Best for | A quick file lookup | Humans: `type -a`, or `type -t` for one word | **Scripts**: portable and short |

`which` only searches `PATH`. It has no idea an alias or function with
the same name exists, and will report a file that is **not** what runs.
Using the `mytool` file you build in the lab:

```bash
alias mytool="echo ALIAS-RUNS"
mytool                       # ALIAS-RUNS             ← what actually runs
command which mytool         # …/dir1/mytool          ← a file that does NOT run
type -t mytool               # alias                  ← the real answer
command -v mytool            # alias mytool='echo ALIAS-RUNS'
```

(`command which` runs the real `which` program. On Rocky, plain
`which` in a terminal is a function that *does* read aliases, so the
lab uses `command which` to get the same result on both distros.)

`which` also finds nothing for a pure builtin that has no file:

```bash
command which shopt; echo "exit code: $?"
# (nothing printed)
# exit code: 1

type shopt
# shopt is a shell builtin
```

A file on disk does not mean that file is what runs. On Rocky,
`which cd` prints `/usr/bin/cd`, yet typing `cd` runs the builtin
(it must, because only the builtin can change your shell's directory).
On Ubuntu `which cd` prints nothing. `type cd` gives the right answer
on both.

**Rule of thumb:** use `type` interactively and `command -v` in
scripts when you need to know what will run. Use `which` only to
confirm the on-disk location of a plain external command.

### 1.6 Command hashing: bash caches lookups, and the cache can go stale

Searching `PATH` on every invocation would be slow, so bash
remembers ("hashes") the full path the first time it finds a command.
`hash` shows the cache:

```bash
hash
# hits    command
#    1    /home/user/…/dir2/deploytool
```

**Reassigning `PATH` clears the whole cache automatically:**

```bash
PATH="$PATH"
hash
# hash: hash table empty
```

#### Stale case 1: a new copy appears earlier in `PATH`

If `PATH` itself is not reassigned, bash does not notice that a
better match has appeared. The cache still points at the old copy:

```bash
# PATH="…/dir1:…/dir2:…" throughout; only dir2 has the tool so far
deploytool                  # OLD-VERSION   (found in dir2, hashed)
# a new version is installed into dir1, which is earlier in PATH
command which deploytool    # …/dir1/deploytool   ← a fresh PATH search
deploytool                  # OLD-VERSION         ← still the cached dir2 copy

hash -r                     # clear the cache
deploytool                  # NEW-VERSION
```

This is the real "I installed the new version, why is the old one
still running?" scenario. `which` shows the right answer because it
does a fresh search; the shell's execution uses the cache.

#### Stale case 2: the hashed file was moved or removed

The cache points at a specific path. If that file disappears, bash
tries it anyway and fails, even though another copy exists later in
`PATH`:

```bash
deploytool; echo "exit code: $?"
# bash: …/dir1/deploytool: No such file or directory
# exit code: 127
```

The full path in the error message is the clue. `hash -r` fixes it.

#### What is *not* stale: replacing a file at the same path

The cache stores the **path**, not the file's contents. Overwriting
the file at the same path with a new version just works, with no
`hash -r` needed:

```bash
# dir1/deploytool is already hashed; now overwrite it with new contents
deploytool                  # NEWER-VERSION
```

---

## Part 2 — Hands-On Lab

### Commands used in this demo

| Command | Flag meaning | Purpose |
|---|---|---|
| `export` | n/a | Copy a variable into every child process's environment |
| `printenv` | n/a | Show an exported variable (or all of them) |
| `unset` | n/a | Remove a variable (`unset -f` removes a function) |
| `tr ':' '\n'` | n/a | Put each `PATH` directory on its own line |
| `alias` / `unalias` | n/a | Define / remove an alias |
| `type` | n/a | Report how a name would resolve |
| `type -a` | `a` = all | List every match, in the order bash considers them |
| `type -t` | `t` = type-only | Print one scriptable word: alias, function, builtin, file or keyword |
| `command -v` | `v` = describe | Portable, script-friendly "what is this name?" |
| `command which` | n/a | Run the real `which` program, skipping any `which` function |
| `hash` / `hash -r` | `r` = reset | Show / clear bash's cached command locations |
| `shopt -s expand_aliases` | `s` = set | Enable alias expansion in a non-interactive shell |

### Step 0 — Start a child bash

```bash
echo $0
```

If the output is `zsh`, or you want a disposable shell either way,
start a child bash:

```bash
bash
echo $0        # bash
```

Everything you define in this lab (variables, aliases, functions,
`PATH` changes) disappears when you type `exit` at the end.

### Task 1 — Exported vs. plain variables (Concepts §1.1)

```bash
myvar="not exported"
bash -c 'echo child sees: [$myvar]'
# child sees: []

export myvar2="exported"
bash -c 'echo child sees: [$myvar2]'
# child sees: [exported]

printenv myvar2                  # exported
printenv myvar; echo "exit: $?"  # exit: 1   ← not in the environment
```

**One-shot variable, and a child cannot change its parent:**

```bash
FOO=bar bash -c 'echo one-shot child sees: [$FOO]'
# one-shot child sees: [bar]
echo "parent: [$FOO]"
# parent: []

export level=1
bash -c 'level=2; echo "child: $level"'
# child: 2
echo "parent: $level"
# parent: 1

unset myvar2
bash -c 'echo after unset: [$myvar2]'
# after unset: []
```

### Task 2 — PATH priority: prepend vs. append (Concepts §1.2)

```bash
# from the demo directory: 03-environment-path/
mkdir -p src/pathlab/dir1 src/pathlab/dir2 && cd src/pathlab
ORIG_PATH="$PATH"

printf '#!/bin/bash\necho FROM-DIR1\n' > dir1/mytool
printf '#!/bin/bash\necho FROM-DIR2\n' > dir2/mytool
chmod +x dir1/mytool dir2/mytool
```

**Prepend: the first directory wins:**

```bash
PATH="$PWD/dir1:$PWD/dir2:$ORIG_PATH"
echo "$PATH" | tr ':' '\n' | head -3     # …/dir1, …/dir2, then your usual directories
mytool                                    # FROM-DIR1

PATH="$PWD/dir2:$PWD/dir1:$ORIG_PATH"
mytool                                    # FROM-DIR2: order flipped, priority flipped
```

**Append: only a fallback:**

```bash
PATH="$ORIG_PATH:$PWD/dir1"
mytool                                    # FROM-DIR1: nothing earlier has mytool

PATH="$PWD/dir2:$ORIG_PATH:$PWD/dir1"
mytool                                    # FROM-DIR2: the appended dir1 loses
```

### Task 3 — Alias in a terminal vs. a script (Concepts §1.4)

```bash
alias greet="echo ALIAS-VERSION"
greet
# ALIAS-VERSION                     ← works: this shell is interactive
```

**The same alias in a non-interactive shell:**

```bash
bash -c 'alias greet="echo ALIAS-VERSION"
greet'
# bash: line 2: greet: command not found
```

**Enable alias expansion:**

```bash
bash -c 'shopt -s expand_aliases
alias greet="echo ALIAS-VERSION"
greet'
# ALIAS-VERSION
```

**The one-line gotcha:**

```bash
bash -c 'shopt -s expand_aliases; alias greet="echo ALIAS-VERSION"; greet'
# bash: line 1: greet: command not found   ← alias is not active until the next line
```

**A function needs none of this:**

```bash
bash -c 'greet() { echo FUNCTION-VERSION; }
greet'
# FUNCTION-VERSION
```

```bash
unalias greet        # tidy up before the next task
```

### Task 4 — Precedence, `type -a` and bypassing (Concepts §1.3 and §1.4)

Define the function first, then the alias:

```bash
hello() { echo "FUNCTION-VERSION"; }
alias hello="echo ALIAS-VERSION"

hello
# ALIAS-VERSION                     ← alias wins

type -a hello
# hello is aliased to `echo ALIAS-VERSION'
# hello is a function
# hello ()
# {
#     echo "FUNCTION-VERSION"
# }

type -t hello
# alias
```

**Bypass and remove, one layer at a time:**

```bash
\hello                # FUNCTION-VERSION   ← backslash skipped the alias
command hello         # bash: hello: command not found   ← skipped alias AND function

unalias hello
hello                 # FUNCTION-VERSION   ← only the function is left

unset -f hello
hello                 # bash: hello: command not found
```

### Task 5 — `which`, `type` and `command -v` (Concepts §1.5)

Reuse the `mytool` files from Task 2 (run this from `src/pathlab`):

```bash
PATH="$PWD/dir1:$PWD/dir2:$ORIG_PATH"
alias mytool="echo ALIAS-RUNS"

mytool                    # ALIAS-RUNS          ← what actually runs
command which mytool      # …/dir1/mytool       ← a file that does NOT run
type -t mytool            # alias               ← the real answer
type -a mytool
# mytool is aliased to `echo ALIAS-RUNS'
# mytool is …/dir1/mytool
# mytool is …/dir2/mytool
command -v mytool         # alias mytool='echo ALIAS-RUNS'

unalias mytool
command -v mytool         # …/dir1/mytool       ← no alias: the file runs
```

**A builtin with no file:**

```bash
command which shopt; echo "exit: $?"   # (nothing printed)  exit: 1
type shopt                              # shopt is a shell builtin

type -a echo
# echo is a shell builtin
# echo is /usr/bin/echo

type -t if                              # keyword
```

On **Rocky only**, also try `which cd` (it prints `/usr/bin/cd`) and
`type -a cd` (the builtin is listed first). A file exists, but the
builtin is what runs.

### Task 6 — A stale hash: new copy earlier in `PATH` (Concepts §1.6)

```bash
mkdir -p ../hashlab/dir1 ../hashlab/dir2 && cd ../hashlab

printf '#!/bin/bash\necho OLD-VERSION\n' > dir2/deploytool
chmod +x dir2/deploytool
PATH="$PWD/dir1:$PWD/dir2:$PATH"

deploytool                # OLD-VERSION         ← only dir2 has it; now hashed
hash                      # hits  command / 1  …/dir2/deploytool
```

Now "install" a new version into `dir1`, which is earlier in `PATH`:

```bash
printf '#!/bin/bash\necho NEW-VERSION\n' > dir1/deploytool
chmod +x dir1/deploytool

command which deploytool  # …/dir1/deploytool   ← a fresh PATH search finds the new one
deploytool                # OLD-VERSION         ← stale: bash uses its cached dir2 path

hash -r
deploytool                # NEW-VERSION         ← fixed
```

### Task 7 — What is and is not stale (Concepts §1.6)

**Replacing a file at the same path is not stale.** `dir1/deploytool`
is hashed after the last command in Task 6, so overwrite it in place:

```bash
printf '#!/bin/bash\necho NEWER-VERSION\n' > dir1/deploytool
deploytool                # NEWER-VERSION   ← no hash -r needed: same path
```

**Removing the hashed file is stale:**

```bash
rm dir1/deploytool
deploytool; echo "exit: $?"
# bash: …/hashlab/dir1/deploytool: No such file or directory
# exit: 127          ← even though dir2/deploytool still exists

hash -r
deploytool                # OLD-VERSION   ← fresh search falls through to dir2
```

**Reassigning `PATH` clears the cache by itself:**

```bash
hash                      # shows the dir2 entry
PATH="$PATH"
hash                      # hash: hash table empty
```

**Clean up and leave the child shell (optional, since `src/` is git-ignored):**

```bash
cd ../..                  # from src/hashlab back to 03-environment-path/
rm -rf src/pathlab src/hashlab
exit                      # leaves the child bash; aliases, functions and PATH changes vanish
```

---

## What You Learned

- ✅ Showed that child processes only inherit exported variables, that a child cannot change its parent, and used `printenv`, `unset` and one-shot `VAR=value cmd`
- ✅ Flipped lookup priority by prepending and appending to `PATH`
- ✅ Saw an alias work in a terminal and fail in `bash -c`, fixed it with `shopt -s expand_aliases`, and hit the same-line gotcha
- ✅ Proved the alias-over-function precedence with `type -a`, and bypassed it with `\cmd`, `command cmd`, `unalias` and `unset -f`
- ✅ Showed `which` reporting a file that is not what runs, and used `type` and `command -v` instead
- ✅ Reproduced both stale-hash cases and fixed them with `hash -r`
- ✅ Confirmed that replacing a file at the same path is not stale, and that reassigning `PATH` clears the cache

**Key Takeaway:** `which` only answers "what is in `PATH`". `type`
and `command -v` answer the real question, "what will run when I type
this name", and `hash -r` fixes the one case where bash's own cache
disagrees with `PATH`.

---

## Part 3 — Break-Fix Scenario

### Scenario — NovaTech: "I installed the new tool, but the old one still runs"

A teammate reports:

> *"I installed the new `deploy-checker` into `/opt/novatech/bin`,
> which is first in everyone's `PATH`. The old copy is still in
> `/usr/local/bin`. `which deploy-checker` shows the new
> `/opt/novatech/bin` path. But in my open terminals, `deploy-checker`
> still runs the old logic. I did not touch `PATH`."*

**Your task (answer before opening the resolution):**

1. `which` shows the new location. Why might the command that actually runs still be the old one?
2. What single command fixes an already-open shell, without reassigning `PATH` or restarting?
3. Why does `which` show the right answer while execution is wrong? Is that a contradiction?
4. Suppose the installer had *overwritten* the old file in place at `/usr/local/bin/deploy-checker` instead. Would the stale-hash problem still occur?

<details>
<summary>Resolution — open after attempting</summary>

**1. Why the old version still runs:**

This is stale case 1 from §1.6. The teammate had already run
`deploy-checker` in that terminal, so bash cached the *old* copy's path
(`/usr/local/bin/deploy-checker`). `PATH` was never reassigned, so the
cache was not cleared, and bash never re-searched, even though a
better match now exists earlier in `PATH`.

**2. The fix:**

```bash
hash -r
```

This clears the cache, so the next `deploy-checker` triggers a fresh
`PATH` search and finds `/opt/novatech/bin`. A brand-new shell would
also work, since it starts with an empty cache.

**3. Not a contradiction:**

`which` does not use the hash table: it searches `PATH` fresh every
time, so it shows the new location. Actually running a command goes
through bash's resolution order (§1.3), which consults the hash table
before searching `PATH`. The two answer through different paths.

**4. No.**

The cache stores the *path*, not the contents. If the old file at
`/usr/local/bin/deploy-checker` were overwritten in place, the cached
path would simply run the new contents (Task 7). The stale problem
needs the new copy to appear at a *different* path that takes priority,
or the old file to be moved or removed.

**The lesson:** after installing a tool into a different `PATH`
directory than the old copy, run `hash -r` in open shells (or start
new ones). Do not trust `which` alone to confirm what an existing
session will run.

</details>

---

## Interview Prep

**Q1. A teammate's script starts with `alias grep="grep --color=auto"`, expecting every later `grep` to use color. Does it work?**

No. Aliases are expanded only in interactive shells unless
`shopt -s expand_aliases` is set, and even then the alias applies
from the *next line*. A plain script silently calls the real `grep`.
Use a function instead: `grep() { command grep --color=auto "$@"; }`.
`command grep` stops the function calling itself.

---

**Q2. `which python3` and `type python3` disagree on a machine with a `python3` alias. Which do you trust?**

`type`, because it reports what will execute: it checks aliases,
functions and builtins before `PATH`. The `which` program only
searches `PATH`, so it can name a file that is not what runs. (On
Rocky an interactive `which` is a function that does read aliases,
which is one more reason to use `type` or `command -v` consistently.)
In scripts, prefer `command -v`.

---

**Q3. You deployed a new CLI version into `/opt/tools/bin`, ahead of the old copy in `/usr/local/bin`. Some engineers still run the old one. What do you tell them?**

Run `hash -r` in the open shell, or open a new one. Their shell cached
the old copy's path from an earlier run, and reassigning `PATH` would
clear it but installing a better match earlier in `PATH` does not.
Note this applies when the new file is at a *different* path; if the
old file were overwritten in place, the cached path would already run
the new contents.

---

**Q4. A teammate's alias is shadowing a real command and you need the real one right now. What do you do?**

`\cmd` skips the alias, and `command cmd` skips both the alias and any
function with that name. `type -a cmd` first shows everything that
matches, and `unalias cmd` (or `unset -f cmd`) removes the shadow.

---

## Key Takeaways

1. **Only exported variables are copied into child processes, and the copy cannot change the parent.** `VAR=value cmd` gives one command a variable for a single run.
2. **`PATH` is searched left to right; the first match wins.** Prepending gives override priority; appending makes a directory a fallback. A change lasts only for the session (Demo M01-04 covers making it permanent).
3. **Resolution order is alias → function → builtin → hashed location → `PATH` search.** Verify it with `type -a`, and remember keywords like `if` are handled by the parser.
4. **An alias works in a terminal but not in a script, and a function works in both.** `shopt -s expand_aliases` enables aliases in a script, and an alias applies from the next line, not the same one. Define a function before an alias of the same name.
5. **`\cmd`, `command cmd`, `unalias` and `unset -f` are how you get past a shadowing alias or function.**
6. **`which` only searches `PATH`. `type` and `command -v` report what actually runs.** A file on disk (such as Rocky's `/usr/bin/cd`) does not mean that file is what runs.
7. **Bash caches command locations. Reassigning `PATH` clears the cache; `hash -r` clears it by hand.** It goes stale when a better copy appears earlier in `PATH`, or when the hashed file is moved or removed. Replacing a file at the same path is *not* stale, because the cache stores the path.

---

## Quick Commands Reference

### Command resolution order (highest to lowest priority)

| Order | Type | Notes |
|---|---|---|
| 1 | Alias | Interactive shells only, unless `expand_aliases` is on |
| 2 | Function | Any shell |
| 3 | Builtin | Part of bash (`cd`, `echo`, `shopt`) |
| 4 | Hashed location | Cached result of an earlier `PATH` search |
| 5 | `PATH` search | Left to right, first match |

### Finding out what runs

| Command | Answers |
|---|---|
| `type name` | How `name` resolves (human-readable) |
| `type -a name` | Every match, in precedence order |
| `type -t name` | One word: alias, function, builtin, file, keyword |
| `command -v name` | Portable, script-friendly answer |
| `command which name` | Only the `PATH` file lookup (blind to aliases and functions) |

### Bypassing and removing

| Form | Effect |
|---|---|
| `\name` | Skip the alias |
| `command name` | Skip alias and function |
| `unalias name` | Remove the alias |
| `unset -f name` | Remove the function |

### Hash table

| Command | Effect |
|---|---|
| `hash` | Show the cached command → path table |
| `hash -r` | Clear the cache |
| `PATH="$PATH"` | Reassigning `PATH` also clears the cache |

---

## Appendix A — Anki Cards

**`m01-03-environment-path-anki.csv`:**

```
#deck:Linux Mastery::Module 1 - Filesystem & Navigation::Demo M01-03 - Environment & PATH
#separator:Comma
#columns:Front,Back,Tags

"A plain variable myvar=x is set, then bash -c 'echo $myvar' runs. What does the child print, and why?","Nothing (empty). A plain variable exists only in the current shell. Child processes receive only variables that were exported with export.","demo00c,environment,export"

"What does 'FOO=bar command' do?","It gives command the variable FOO=bar for that single run only. The current shell does not get the variable.","demo00c,environment,one-shot"

"PATH=/new:$PATH vs PATH=$PATH:/new: what is the practical difference?","Prepending (/new first) gives it override priority over same-named tools later in PATH. Appending makes it only a fallback, used when nothing earlier matches.","demo00c,path,priority"

"alias greet='echo X' works in your terminal but not inside bash -c or a script. Why, and what is the fix?","Aliases are only expanded in interactive shells by default. In a script, use shopt -s expand_aliases (on its own line, before the alias is defined and used), or better, use a function.","demo00c,alias,scripts"

"shopt -s expand_aliases; alias g='echo X'; g all on ONE line fails with 'command not found'. Why?","Bash reads a whole line before running it, so an alias defined on the same line is not active yet. The alias takes effect from the next line.","demo00c,alias,gotcha"

"An alias and a function with the same name both exist. Which runs, and how do you verify it?","The alias wins (alias, then function, then builtin, then hash/PATH). Verify with type -a name, which lists every match in precedence order.","demo00c,alias,function,type-a"

"How do you run the real command when an alias or function is shadowing it?","\name skips the alias; command name skips the alias and the function. unalias name and unset -f name remove them.","demo00c,alias,bypass"

"Why define a function before an alias of the same name?","If the alias already exists, bash expands it inside the function definition and reports a syntax error near unexpected token '('.","demo00c,alias,function,syntax"

"which and type give different answers when an alias exists. Which tells you what actually runs?","type (and command -v). The which program only searches PATH and cannot see aliases, functions or builtins, so it can name a file that is not what runs. In scripts prefer command -v.","demo00c,which,type,command-v"

"command which shopt prints nothing and exits 1. Why?","shopt is a pure shell builtin with no file in PATH, so a PATH-only search finds nothing. type shopt reports 'shopt is a shell builtin'. (On Rocky, which cd does print /usr/bin/cd, but the builtin is still what runs.)","demo00c,which,builtin"

"What is bash's command hash table, and what clears it?","A cache of full paths found by earlier PATH searches. Reassigning PATH clears it automatically; hash -r clears it by hand; a new shell starts empty.","demo00c,hash,cache"

"A new copy of a tool is installed EARLIER in PATH, PATH is not reassigned, and the old copy still runs. Why and what fixes it?","Bash cached the old copy's path and does not re-search. hash -r (or a new shell) forces a fresh PATH search. which still shows the new copy because it searches fresh.","demo00c,hash,stale,hash-r"

"Does overwriting a hashed file at the SAME path leave a stale hash?","No. The cache stores the path, not the contents, so the new contents run. A hash goes stale only when a better copy appears at a different, earlier path, or the hashed file is moved or removed (then you see No such file or directory with the full path, exit 127).","demo00c,hash,same-path"
```

---

## Appendix B — Quiz

**`m01-03-environment-path-quiz.md`:**

```
# Quiz — Demo M01-03: Environment & PATH

> One correct answer per question. Target: 80% (8/10) before Demo M01-04.
> Assume bash.

---

**Q1.** `export DEPLOY_ENV=staging` is set in a parent shell, then a
child process starts and changes `DEPLOY_ENV` to `prod`. What does the
parent see afterwards?

- A) `prod`
- B) `staging`: the child had its own copy
- C) Empty
- D) An error

<details>
<summary>Answer</summary>

**B**. Exporting copies the variable into the child's environment.
Changes in the child never flow back to the parent.

</details>

---

**Q2.** A script defines `alias grep="grep -n"` at the top. What
happens to later `grep` calls in the script?

- A) They show line numbers
- B) The alias is silently ignored: scripts do not expand aliases unless `shopt -s expand_aliases` is set
- C) The script errors on the `alias` line
- D) Only the first call is affected

<details>
<summary>Answer</summary>

**B**. Non-interactive shells do not expand aliases by default. A
function works immediately.

</details>

---

**Q3.** Both an alias and a function named `hello` exist. Which runs,
and how do you verify it?

- A) The function; functions always win
- B) The alias; check with `type -a hello`
- C) Random
- D) Both, one after the other

<details>
<summary>Answer</summary>

**B**. Order is alias, function, builtin, hash/`PATH`. `type -a` lists
the matches in that order.

</details>

---

**Q4.** `bash -c 'shopt -s expand_aliases; alias hi="echo HI"; hi'`
prints `hi: command not found`. Why?

- A) `expand_aliases` does not exist
- B) An alias takes effect only from the next line, and everything here is on one line
- C) `echo` is not allowed in aliases
- D) `bash -c` disables all builtins

<details>
<summary>Answer</summary>

**B**. Bash reads a whole line before running it, so the alias is not
active yet. Putting the three commands on separate lines works.

</details>

---

**Q5.** A function `hello` and an alias `hello` both exist. What does
`\hello` run?

- A) The alias
- B) The function: the backslash skips the alias
- C) `command not found`
- D) Both

<details>
<summary>Answer</summary>

**B**. `\hello` skips the alias only. `command hello` would skip both
the alias and the function, and report `command not found` if no file
exists.

</details>

---

**Q6.** `which mytool` shows `/path/dir1/mytool`, but an alias named
`mytool` exists and is what runs. Why doesn't `which` show it?

- A) `which` is buggy
- B) The `which` program searches only `PATH` files and cannot see aliases, functions or builtins
- C) Aliases are stored in `PATH`
- D) Only on Rocky

<details>
<summary>Answer</summary>

**B**. Use `type` or `command -v` to see what really runs. (On Rocky,
the interactive `which` function does read aliases; `command which`
bypasses it. The principle holds everywhere: `which` is a file lookup.)

</details>

---

**Q7.** `command which shopt` prints nothing and exits `1`. What does
that indicate?

- A) `shopt` does not exist
- B) `shopt` is a pure builtin with no file in `PATH` for a `PATH`-only search to find
- C) A permissions problem
- D) `which` handles only 20 commands

<details>
<summary>Answer</summary>

**B**. `type shopt` reports "shopt is a shell builtin".

</details>

---

**Q8.** You reassign `PATH` with a new value. What happens to bash's
command hash table?

- A) Nothing: it persists until `hash -r`
- B) It is cleared automatically
- C) Only entries missing from the new `PATH` are removed
- D) It is rebuilt immediately

<details>
<summary>Answer</summary>

**B**. Immediately afterwards `hash` prints "hash table empty".

</details>

---

**Q9.** A new version of a tool is installed into a directory that is
*earlier* in `PATH` than the old copy. `PATH` is not touched, and
engineers in already-open shells still run the old version. What
fixes it?

- A) `hash -r`: those shells cached the old copy's path
- B) Nothing: bash re-checks `PATH` on every command
- C) `unalias -a`
- D) Logging out and back in is the only way

<details>
<summary>Answer</summary>

**A**. The cache points at the old path, and `hash -r` clears it. A
new shell would also work, but logging out (D) is not the *only* way.

</details>

---

**Q10.** The tool's old file is *overwritten in place* at the same path
with a new version. Do open shells that already ran it keep running
the old logic?

- A) Yes: the hash is stale, run `hash -r`
- B) No: the cache stores the path, not the contents, so the new contents run
- C) Only on Rocky
- D) Yes, until the shell restarts

<details>
<summary>Answer</summary>

**B**. A stale hash needs a better copy at a *different, earlier* path,
or the hashed file to be moved or removed (then you get
`No such file or directory` with the full path, and exit `127`).

</details>

---

Score guide:

| Score | Action |
|---|---|
| 10/10 | Import Anki cards, move to Demo M01-04 |
| 9/10 | Review the wrong answer, then proceed |
| 7-8/10 | Re-read the relevant section, retry those questions |
| Below 7/10 | Re-read the full demo and redo the lab |
```