# Demo M01-02 — I/O Redirection, File Descriptors & Exit Status

## Overview

Every command you run is a process, and every process is started with
three data streams already connected: one to read input from and two to
write output to. Results go to one output stream and error messages go
to the other, so the two never have to share a channel. **Redirection**
rewires those streams. The **exit status** is the other half of the
story: the single number every process leaves behind that says whether
it worked, regardless of what it printed.

Together they make scripts verifiable instead of guesswork. A script
that never checks an exit status cannot tell success from failure. A
script that redirects streams in the wrong order can silently lose the
very error message it needed.

This demo also covers pipes and `xargs`. A pipe connects one command's
output to the next command's input, and `xargs` turns that text into
*arguments* for another command. They live here because both depend on
streams, and `xargs` reports through an exit status.

| Concept | The question it answers |
|---|---|
| Vocabulary (§1.1) | What do process, stream, terminal and file descriptor mean? |
| Standard streams (§1.2) | Where does a process read from and write to? |
| Redirection (§1.3, §1.4) | How do I rewire those streams, and does the order matter? |
| Exit status (§1.5, §1.6) | Did the command work, and how do I act on that answer? |
| Pipes (§1.7) | How do I feed one command's output into another? |
| `xargs` (§1.8) | How do I turn output into *arguments* for another command? |

---

## Demo Objectives

By the end of this demo you will be able to:

1. Explain the terms process, argument, terminal, stream and file descriptor, and name the three standard streams (0, 1, 2)
2. Explain what redirection is, and use `>`, `>>`, `<`, `2>`, `&>`, `/dev/null` and `>&2`
3. Explain the two different meanings of `&` in redirection
4. Explain why the order of redirections matters (`> file 2>&1` vs. `2>&1 > file`)
5. Read the exit status with `$?`, and recognize what 0, 1, 2, 126, 127 and 128+N (130, 137) usually mean
6. Chain commands with `&&` and `||` based on exit status, and explain how a pipeline reports its status
7. Explain why `xargs` is needed (a pipe carries text on stdin, not arguments), and use `-n`, `-I`, `-t`, `-r` and `-P`
8. Explain how `-n` and `-I{}` differ, and avoid the whitespace footgun with `-print0` / `-0`
9. Explain why `xargs` exits `123` or `124` instead of the sub-command's own status

---

## Directory Structure

```
02-redirection-exit-codes/
├── README.md                               # this file
├── m01-02-redirection-exit-codes-anki.csv  # Anki flashcard deck (Appendix A)
├── m01-02-redirection-exit-codes-quiz.md   # standalone quiz (Appendix B)
└── src/                                    # lab working directories (git-ignored)
    ├── redir-lab/                          # Tasks 1-4 and the break-fix repro
    └── xargs-lab/                          # Tasks 5-9
```

All lab work happens inside `src/`. The lab commands create the
folders (`mkdir -p`), so you only need to be in the demo directory
(`02-redirection-exit-codes/`) when you start.

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

Redirection, exit status and `xargs` behave the same on both distros
(`xargs` is part of `findutils`, installed by default). Two
reminders:

- **Run the lab in bash.** Other shells such as zsh differ in details of redirection and variable handling. The results below are bash results. Step 0 of the lab switches you into bash.
- **`sh` is not bash.** A few commands use `sh -c`. On Ubuntu `sh` is `dash`; the commands used here work in both.

---

## Recall Check — Demo M01-01

Before starting, answer from memory:

1. In bash, `echo *.xyz` is run where nothing matches. What is printed?
2. Which stages does double-quoting a variable skip?
3. `tmpfile=session_*.tmp` is run while matching files exist. When is the wildcard expanded: at the assignment, or when the variable is used?

<details>
<summary>Answers</summary>

1. The literal text `*.xyz`, unchanged.
2. Word splitting and globbing (brace expansion also does not happen inside quotes). `$variable` expansion itself still happens.
3. When the variable is used unquoted as a command argument. Assignment stores plain text.

</details>

---

## Part 1 — Concepts

### 1.1 Before you start: the vocabulary

This demo uses a handful of terms. Here they are once, in plain words.

| Term | Meaning |
|---|---|
| **Program** | A file on disk that can be run, for example `/usr/bin/ls` |
| **Process** | A running copy of a program. Every command you run creates one |
| **Command** | What you type: a command name followed by arguments, for example `ls -l /etc` |
| **Argument** | A word after the command name that says what to work on or how, for example `-l` and `/etc`. Also called a command-line argument |
| **Shell** | The program (here, bash) that reads what you type, performs the expansions from Demo M01-01, and starts the process |
| **Terminal** | The window or console where you type commands and see their output. By default your keyboard supplies a process's input and your screen shows its output |
| **Stream** | A flow of bytes into or out of a process |
| **File descriptor (FD)** | The number the operating system uses to identify one of a process's open streams or files |
| **Pipe** | A connection that carries one command's output into the next command's input (`\|`) |
| **Exit status** | The number a process returns when it finishes. `0` means success (§1.5) |

How the pieces fit together when you run a command:

```
 You type:     ls -l /etc
                 |
                 v
          +-------------+   1. expands the line (Demo M01-01)
          |    shell    |   2. finds the program
          +------+------+   3. starts a process
                 |
                 v
          +-------------+
          |   process   |   running copy of /usr/bin/ls
          |    "ls"     |   arguments:  -l   /etc
          +-------------+
```

#### Two ways to give a command something to work on

A command can receive data as **arguments** or on its **standard
input** (stdin, §1.2). The same command behaves differently:

```bash
wc -l fruit.txt        # argument: you give a file NAME; wc opens the file itself
# 3 fruit.txt          ← wc knows the name, so it prints it

wc -l < fruit.txt      # stdin: the shell opens the file; wc just reads its stdin
# 3                    ← wc never saw a name

cat fruit.txt | wc -l  # stdin: the data arrives from another command through a pipe
# 3
```

Keep this distinction in mind: it is the whole reason `xargs` exists
(§1.8).

### 1.2 The three standard streams (file descriptors)

When the shell starts a process, the process begins with three streams
already open. Each has a number, its file descriptor:

| FD | Name | Default connection | Purpose |
|---|---|---|---|
| `0` | **stdin** (standard input) | Keyboard | Input: where the process reads data from |
| `1` | **stdout** (standard output) | Terminal screen | Normal output: where the process writes its results |
| `2` | **stderr** (standard error) | Terminal screen | Error output: where the process writes error and diagnostic messages |

```
                      +---------------------------+
   keyboard  ------>  | 0  stdin                  |
                      |                           |
                      |   process (the command)   |
                      |                           |
                      | 1  stdout  -------------->|------>  screen
                      | 2  stderr  -------------->|------>  screen
                      +---------------------------+
```

Both `1` and `2` go to the same screen by default, so they look
identical until you redirect one of them:

```bash
ls real.txt missing.txt
# ls: cannot access 'missing.txt': No such file or directory     ← stderr (FD 2)
# real.txt                                                        ← stdout (FD 1)
```

Two different streams, one screen: you cannot tell which line came
from which. Keeping them separate is what lets you capture a
command's real results in a file while still seeing the errors on
screen, or the reverse.

### 1.3 Redirection

**Redirection** means changing where a stream is connected. Instead of
the screen or the keyboard, you connect it to a file, or throw it away.
The shell sets this up **before** the command starts; the command does
not know it happened.

```
 Default:                              After   ls > out.txt:

   ls --1--> screen                      ls --1--> out.txt
      --2--> screen                         --2--> screen
```

| Operator | Effect |
|---|---|
| `> file` | Send stdout to `file`, **overwriting** it (short for `1> file`) |
| `>> file` | Send stdout to `file`, **appending** |
| `< file` | Read stdin from `file` (short for `0< file`) |
| `2> file` | Send stderr to `file` |
| `&> file` | Send stdout **and** stderr to `file` (bash shortcut) |
| `2>&1` | Send stderr to wherever stdout points right now (§1.4) |
| `>&2` | Send this command's stdout to stderr |
| `> /dev/null` | Discard the output: `/dev/null` accepts anything and throws it away |

#### Writing to a file: overwrite vs. append

```bash
echo "first" > notes.txt     # creates or overwrites notes.txt
echo "second" >> notes.txt   # adds a line to the end
```

#### Reading from a file

```bash
sort < fruit.txt             # sort reads the file as its stdin
```

#### Splitting the two output streams

```bash
ls real.txt missing.txt > out.txt 2> err.txt
```

```
   ls --1--> out.txt    (real.txt)
      --2--> err.txt    (No such file or directory)
```

#### Both streams into one file

Two ways to write it. They produce the same result:

```bash
ls real.txt missing.txt > all.txt 2>&1     # the portable form (works in any shell)
ls real.txt missing.txt &> all.txt         # bash shortcut for the line above
```

#### Discarding output

```bash
ls real.txt missing.txt 2> /dev/null       # hide errors, keep results
ls real.txt missing.txt > /dev/null 2>&1   # hide everything
```

#### What `&` means: two different uses

The `&` character shows up in redirection in two ways, which is why it
is confusing:

| Written as | What `&` means there | Example |
|---|---|---|
| `&>` (at the **start** of the operator) | "both": stdout **and** stderr together | `cmd &> all.log` |
| `>&` followed by a number (as in `2>&1`, `>&2`) | "**the stream with this number**", a file descriptor, not a file name | `cmd 2>&1` |

The second use is the reason `&` exists. Without it, the shell would
read the `1` as a **file name**:

```bash
ls missing.txt 2>1       # no &: sends stderr to a FILE named "1"
ls
# 1   real.txt           ← a new file called 1 appeared
cat 1
# ls: cannot access 'missing.txt': No such file or directory

ls missing.txt 2>&1      # with &: sends stderr to FD 1 (wherever stdout goes)
```

So read `2>&1` as "FD 2 goes where FD 1 goes", and `>&2` as "FD 1 goes
where FD 2 goes".

#### Writing your own error messages

`>&2` sends a command's normal output to stderr instead. It is how
scripts print their own error messages so they do not mix into the
results:

```bash
echo "error: backup failed" >&2
```

### 1.4 Order matters: `2>&1` is a snapshot, not a link

`2>&1` means "make stderr point at **whatever stdout points to right
now**". It does not link the two streams permanently. The shell reads
redirections **left to right**, so the order changes the result.

Take `ls real.txt missing.txt`, which produces one result (stdout) and
one error (stderr):

**`> file 2>&1`: both end up in the file**

| Redirection (left to right) | stdout (FD 1) points to | stderr (FD 2) points to |
|---|---|---|
| (start) | screen | screen |
| `> good.txt` | **good.txt** | screen |
| `2>&1` | good.txt | **good.txt** (copied from stdout) |

**`2>&1 > file`: only stdout ends up in the file**

| Redirection (left to right) | stdout (FD 1) points to | stderr (FD 2) points to |
|---|---|---|
| (start) | screen | screen |
| `2>&1` | screen | **screen** (copied from stdout) |
| `> bad.txt` | **bad.txt** | screen (does not follow) |

```
   > good.txt 2>&1                      2>&1 > bad.txt

   1 -----> good.txt                    1 -----> bad.txt
   2 -----> good.txt                    2 -----> screen
```

```bash
ls real.txt missing.txt > good.txt 2>&1
# Read left to right:
# 1. > good.txt : stdout (FD 1) → good.txt        stderr (FD 2) still → screen
# 2. 2>&1       : stderr (FD 2) → wherever FD 1 points NOW = good.txt
# 3. ls runs    : its listing AND its error both go to good.txt; the screen stays empty

ls real.txt missing.txt 2>&1 > bad.txt
# Read left to right:
# 1. 2>&1       : stderr (FD 2) → wherever FD 1 points NOW = the screen
# 2. > bad.txt  : stdout (FD 1) → bad.txt         stderr stays → screen
# 3. ls runs    : the listing goes to bad.txt; the error appears on screen
```

**The rule:** to capture both streams in one file, the file
redirection comes first: `> file 2>&1`. Writing `2>&1 > file` is a
common, silent mistake. It looks right and produces a file, just not
the content you expect.

### 1.5 Exit status: the number every command leaves behind

When a process finishes, it returns a number to the shell: its **exit
status** (also called the *exit code* or *return code*). The shell
stores the exit status of the most recent command in a special
variable named `?`, which you read as `$?`.

```
 you --> shell --> process  ("ls /nonexistent")
                      |
                      |  finishes and returns exit status 2
                      v
                the shell stores it in  $?     <---  echo $?   prints 2
```

`$?` is a **special parameter**: a variable the shell maintains for
you, named by a symbol, that you can only read. Two you will use in
this course:

| Special parameter | Holds |
|---|---|
| `$?` | The exit status of the most recent command |
| `$$` | The process ID (PID) of the current shell |

`$?` is overwritten by **every** command, including `echo`, so read it
immediately after the command you care about.

#### Where the number comes from

A program sets its exit status when it exits. In a script you choose it
yourself:

```bash
bash -c 'exit 5'
echo $?
# 5
```

The convention is simple: **`0` means success, and any non-zero value
(1 to 255) means failure**. The specific number tells you what kind.

| Status | Meaning |
|---|---|
| `0` | Success |
| `1` | General failure. Many tools also use it for a *normal* negative answer: `grep` returns `1` when it finds **no match** |
| `2` | Misuse or error. `grep` returns `2` when something went wrong, such as a missing file |
| `126` | The file was found but is not executable (permissions) |
| `127` | Command not found |
| `128+N` | Killed by signal number N. `130` = Ctrl-C (signal 2, SIGINT), `137` = SIGKILL (signal 9), `143` = SIGTERM (signal 15) |

The `grep` case is worth remembering: `1` means "searched fine, found
nothing", while `2` means "could not search". Scripts that treat both
the same lose that difference.

`137` is the status you see when the kernel's out-of-memory killer or
`kill -9` ends a process, which becomes important in the performance
module.

#### The exit status of a pipeline

A **pipeline** is commands joined with `|` (§1.7). `$?` holds the
status of the **last** command in the pipeline; earlier commands'
statuses are hidden. Bash keeps all of them in the array
`PIPESTATUS`:

```
   false | true            $? is 0   (the last command's status)
     |      |
     1      0              PIPESTATUS = 1 0
```

```bash
false | true
echo "last command: $?"            # last command: 0
false | true
echo "all: ${PIPESTATUS[@]}"       # all: 1 0
```

Remember this when you pipe into `xargs` (§1.8): `$?` reports
`xargs`'s status, not the status of the command before the pipe.

### 1.6 Acting on exit status: `&&`, `||` and `;`

| Form | Runs the second command... |
|---|---|
| `cmd1 && cmd2` | only if `cmd1` exited `0` |
| `cmd1 \|\| cmd2` | only if `cmd1` exited non-zero |
| `cmd1 ; cmd2` | always, whatever `cmd1` returned |

`&&` is the standard way to say "only continue if that worked", for
example `mkdir build && cd build`. Walking a chain through step by
step:

```bash
ls missing.txt 2> /dev/null && echo A || echo B
# 1. ls missing.txt  → fails, exit status 2
# 2. &&              : status is non-zero → skip `echo A`; status stays 2
# 3. ||              : status is non-zero → run `echo B`
# prints: B

ls real.txt > /dev/null && echo A || echo B
# 1. ls real.txt     → succeeds, exit status 0
# 2. &&              : status is 0 → run `echo A` (succeeds, status 0)
# 3. ||              : status is 0 → skip `echo B`
# prints: A

true && false || echo C
# 1. true → 0   2. && → run `false` → status 1   3. || → status non-zero → run `echo C`
# prints: C
# `a && b || c` is NOT if/else: c also runs when b fails.
```

### 1.7 Pipes: connecting stdout to stdin

A **pipe** (`|`) connects the **stdout** of the command on its left to
the **stdin** of the command on its right:

```
   ls *.tmp  --stdout-->  [ pipe ]  --stdin-->  wc -l
```

```bash
ls *.tmp | wc -l
# 1. shell globbing : *.tmp → a.tmp b.tmp c.tmp, so ls gets 3 arguments
# 2. ls             : prints 3 names, one per line, on its stdout
# 3. pipe           : carries that text into wc's stdin
# 4. wc -l          : counts the lines
# 3
```

Two facts matter for what follows:

- A pipe carries **stdout only**. Error messages (stderr) still go to the screen, unless you add `2>&1` before the pipe.
- A pipe delivers **text on stdin**, not arguments. A command that expects arguments gets nothing from it.

### 1.8 `xargs`: turning output into arguments

#### Why `xargs` exists

`find` locates files, `ls` lists them, `grep -l` names matching files,
but none of them *act* on the list. The obvious idea is to pipe the
list into the command that acts, for example `rm`. It does not work:

```bash
ls *.tmp | rm
# rm: missing operand
# Try 'rm --help' for more information.
```

Here is what happens:

```
  ls *.tmp  -->  a.tmp          +---------------------------+
                 b.tmp  --pipe->| rm                        |
                 c.tmp          |  stdin: "a.tmp b.tmp ..." |  <-- the text arrives here...
                                |  arguments: (none)        |  <-- ...but rm reads its ARGUMENTS
                                +---------------------------+
                                rm: missing operand
```

`rm` decides what to delete from its **arguments** and never reads its
stdin. The text sits unread, and `rm` complains that it got no
arguments. `xargs` is the bridge: it **reads stdin and turns each
piece of text into an argument** for a command you name.

```bash
ls *.tmp | xargs rm -v
# removed 'a.tmp'
# removed 'b.tmp'
# removed 'c.tmp'
```

```
  ls *.tmp  -->  a.tmp          +--------+      builds this command:
                 b.tmp  --pipe->| xargs  | ---> rm -v a.tmp b.tmp c.tmp ---> runs it
                 c.tmp          +--------+
                 (stdin text)   reads stdin,
                                splits it into items,
                                adds them as arguments
```

You can watch `xargs` build the command with `-t` (trace), which
prints it to stderr before running it:

```bash
ls *.tmp | xargs -t rm -v
# rm -v a.tmp b.tmp c.tmp        ← the command xargs built (stderr)
# removed 'a.tmp'                ← what rm printed (stdout)
# removed 'b.tmp'
# removed 'c.tmp'
```

#### What `xargs` does, in order

| Order | What `xargs` does |
|---|---|
| 1 | Reads all of stdin |
| 2 | Splits it into **items** (by default at spaces, tabs and newlines) |
| 3 | Builds a command: the command you named **plus the items as arguments** |
| 4 | Runs it, then exits with its own status (see "`xargs` exit status" below) |

If you name no command, `xargs` uses `echo`, which makes it easy to
see what it does:

```bash
printf 'a b c\nd e\n' | xargs
# a b c d e         ← one run: echo a b c d e
```

#### `-n`: how many items per run

By default, `xargs` puts **all** items on **one** command line. `-n N`
puts at most N items on each command line and runs the command as many
times as needed. The items are always added **at the end**.

```bash
printf 'a b c\nd e\n' | xargs -n2 echo "run:"
# run: a b
# run: c d
# run: e
```

```
  items:  a   b   c   d   e         (words; the line boundary is ignored)

  -n2:    [ a b ]  [ c d ]  [ e ]   →  echo run: a b
                                       echo run: c d
                                       echo run: e
```

`-n1` gives one item per run: `run: a`, `run: b`, `run: c`, `run: d`,
`run: e`.

#### `-I{}`: one line per run, at a position you choose

`-I{}` defines a **placeholder** (`{}`). `xargs` then:

- reads **whole lines**, not words (each line is one item, spaces kept), and
- runs the command **once per line**, putting the line **where `{}` appears**.

```bash
printf 'a b c\nd e\n' | xargs -I{} echo "run: [{}]"
# run: [a b c]
# run: [d e]
```

```
  lines:  "a b c"     "d e"

  -I{}:   [ a b c ]   [ d e ]      →  echo run: [a b c]
                                       echo run: [d e]
```

Because you control the position, `-I{}` can use the item more than
once, which `-n` cannot:

```bash
printf 'x.tmp\ny.tmp\n' | xargs -I{} mv {} {}.bak
# runs:  mv x.tmp x.tmp.bak
#        mv y.tmp y.tmp.bak
```

#### `-n` vs. `-I{}`: where they differ

Same input in all four rows: the two lines `a b c` and `d e`.

| Command | Splits the input into | Number of runs | What the command receives |
|---|---|---|---|
| `xargs echo "run:"` | words | 1 | `run: a b c d e` |
| `xargs -n2 echo "run:"` | words, grouped 2 at a time | 3 | `run: a b` / `run: c d` / `run: e` |
| `xargs -n1 echo "run:"` | words, 1 at a time | 5 | `run: a` / `run: b` / ... |
| `xargs -I{} echo "run: [{}]"` | **lines** | 2 | `run: [a b c]` / `run: [d e]` |

| | `-n N` | `-I{}` |
|---|---|---|
| Items are | words | whole lines |
| Items go | at the **end** of the command | wherever `{}` is, any number of times |
| Use it when | the command accepts many arguments and you only want to batch them | you need the item in a specific spot, or twice (`mv {} {}.bak`) |

Do not combine them. `xargs` warns
(`options --max-args and --replace/-I/-i are mutually exclusive`) and
ignores `-n`.

#### `-P`: run several at the same time

Normally `xargs` runs its commands **one after another**. `-P N` runs
up to N of them **at the same time** (in parallel), which is faster
when each one is slow and they do not depend on each other. Each job
below sleeps for 1 second:

```bash
time printf '1\n2\n3\n' | xargs -I{} sh -c 'sleep 1; echo done-{}'
# real  0m3.0s     ← one at a time: 1 + 1 + 1 seconds

time printf '1\n2\n3\n' | xargs -P3 -I{} sh -c 'sleep 1; echo done-{}'
# real  0m1.0s     ← all three at once
```

```
  one at a time (default):    |--job 1--|--job 2--|--job 3--|     3 s

  -P3:                        |--job 1--|
                              |--job 2--|
                              |--job 3--|                         1 s
```

With `-P`, jobs finish in any order, so the `done-N` lines may appear as
`done-2`, `done-1`, `done-3`.

#### Empty input and `-r`

If nothing arrives on stdin, GNU `xargs` still runs the command **once,
with no arguments**. For `echo` that is harmless; for `rm` it is an
error:

```bash
printf '' | xargs echo hi
# hi                         ← ran once anyway

printf '' | xargs rm
# rm: missing operand        ← and exit status 123

printf '' | xargs -r rm
#                            ← -r (no-run-if-empty): nothing to do, nothing runs
```

Use `-r` whenever the input might be empty.

#### The whitespace footgun and `-print0`

By default `xargs` splits on whitespace. This is the same mechanism
as an unquoted variable in Demo M01-01: a filename that contains a space
is torn into two items.

```bash
find spacedir -name "*.txt" | xargs -n1 echo "found:"
# 1. find   : prints 2 lines: "spacedir/has space.txt" and "spacedir/normal.txt"
# 2. xargs  : splits on whitespace → spacedir/has | space.txt | spacedir/normal.txt   (3 items!)
# 3. -n1    : echo runs 3 times
# found: spacedir/has
# found: space.txt
# found: spacedir/normal.txt
```

```
  find prints:    spacedir/has space.txt
                  spacedir/normal.txt
                         |
  xargs splits at every space, tab and newline:
                  spacedir/has | space.txt | spacedir/normal.txt     3 items, not 2
```

The fix is NUL-delimited input. `find -print0` separates names with a
NUL byte (which cannot appear in a filename), and `xargs -0` splits
only at NUL:

```bash
find spacedir -name "*.txt" -print0 | xargs -0 -n1 echo "found:"
# 1. find -print0 : separates the 2 names with NUL bytes instead of newlines
# 2. xargs -0     : splits only at NUL → 2 items, the space is kept
# 3. -n1          : echo runs 2 times
# found: spacedir/has space.txt
# found: spacedir/normal.txt
```

(The order of `find`'s results depends on the filesystem.)

#### `xargs` exit status

`xargs` does not pass the sub-command's exit status through. It
returns its own:

| `xargs` exits | Meaning |
|---|---|
| `0` | Every run succeeded |
| `123` | At least one run exited with a status from 1 to 125. Processing continues for the remaining input |
| `124` | A run exited with `255`. `xargs` **stops immediately** |

```bash
printf "ok\nbad\nok2\n" | xargs -I{} sh -c '[ "{}" = "bad" ] && exit 5 || echo "{}"'
# run 1 (ok)  : test is false → || runs `echo ok`        → status 0
# run 2 (bad) : test is true  → && runs `exit 5`         → status 5
# run 3 (ok2) : test is false → || runs `echo ok2`       → status 0
# xargs: at least one run exited 1-125 → xargs itself exits 123 (not 5)
```

Either way you lose which item failed and the real status. To find
out, the sub-command itself must report the failing item (for example
`echo "FAILED: {}" >&2`) before exiting.

---

## Part 2 — Hands-On Lab

### Commands used in this demo

| Command | Flag meaning | Purpose |
|---|---|---|
| `bash -c '...'` | `c` = command | Run the quoted text as a tiny script in a child shell |
| `sh -c '...'` | `c` = command | Same, using `sh` |
| `printf` | n/a | Print formatted text; used here to feed several lines into a pipe |
| `wc -l` | `l` = lines | Count lines |
| `grep -q` | `q` = quiet | Print nothing, just set the exit status |
| `time` | n/a | Report how long a command or pipeline took |
| `find -print0` | n/a | Print matches separated by NUL bytes instead of newlines |
| `xargs -n` | `n` = number | Maximum items per run |
| `xargs -I` | `I` = replace-string | Placeholder; one input line per run |
| `xargs -t` | `t` = trace | Print each command to stderr before running it |
| `xargs -r` | `r` = no-run-if-empty | Do not run the command if the input is empty |
| `xargs -P` | `P` = max-procs | Run up to N runs in parallel |
| `xargs -0` | n/a | Split input on NUL bytes instead of whitespace |

### Step 0 — Make sure you are in bash

```bash
echo $0
```

If the output is `bash` (or `-bash`), continue. If it is `zsh`, switch
to bash for this lab:

```bash
bash
```

Run `echo $0` again to confirm. Type `exit` at the end of the lab to
return to your normal shell.

### Task 1 — Streams and redirection (Concepts §1.2 and §1.3)

```bash
# from the demo directory: 02-redirection-exit-codes/
mkdir -p src/redir-lab && cd src/redir-lab
touch real.txt
printf 'banana\napple\ncherry\n' > fruit.txt
```

**Two streams, one screen:**

```bash
ls real.txt missing.txt
# ls: cannot access 'missing.txt': No such file or directory     ← stderr
# real.txt                                                        ← stdout

ls real.txt missing.txt > out.txt
# ls: cannot access 'missing.txt': No such file or directory     ← only stderr is left on screen
cat out.txt
# real.txt
```

**Overwrite vs. append:**

```bash
echo "first" > notes.txt
echo "second" >> notes.txt
cat notes.txt                # first / second

echo "third" > notes.txt     # > overwrites
cat notes.txt                # third
```

**Arguments vs. stdin (Concepts §1.1):**

```bash
wc -l fruit.txt              # 3 fruit.txt   ← argument: wc knows the name
wc -l < fruit.txt            # 3             ← stdin: no name
sort < fruit.txt             # apple / banana / cherry
```

**Splitting stdout and stderr:**

```bash
ls real.txt missing.txt > out.txt 2> err.txt
cat out.txt                  # real.txt
cat err.txt                  # ls: cannot access 'missing.txt': No such file or directory
```

**Both streams into one file, two ways:**

```bash
ls real.txt missing.txt > all1.txt 2>&1
ls real.txt missing.txt &> all2.txt
cmp all1.txt all2.txt && echo "identical"
# identical                   ← &> is just a shortcut
cat all1.txt                 # the error message and real.txt
```

**The `&` mistake: `2>1` vs. `2>&1`:**

```bash
ls missing.txt 2>1
ls
# 1   all1.txt  all2.txt  err.txt  fruit.txt  notes.txt  out.txt  real.txt
#     ↑ a file named "1" appeared
cat 1
# ls: cannot access 'missing.txt': No such file or directory
rm 1
```

**Discarding output:**

```bash
ls real.txt missing.txt 2> /dev/null        # real.txt
ls real.txt missing.txt > /dev/null 2>&1    # nothing at all
```

**Writing your own error with `>&2`:**

```bash
bash -c 'echo "to stdout"; echo "to stderr" >&2' 2> caught.txt
# to stdout                ← appears on screen
cat caught.txt             # to stderr
```

`bash -c '...'` runs the quoted text in a child shell. Only the
`echo ... >&2` line went to stderr, so `2> caught.txt` captured
exactly that line.

### Task 2 — Order matters (Concepts §1.4)

```bash
ls real.txt missing.txt > good.txt 2>&1
cat good.txt
# ls: cannot access 'missing.txt': No such file or directory
# real.txt

ls real.txt missing.txt 2>&1 > bad.txt
# ls: cannot access 'missing.txt': No such file or directory   ← still on screen
cat bad.txt
# real.txt
```

**Observation:** the same two redirections in a different order. In
the second command the error went to the screen because `2>&1` ran
while stdout still pointed at the screen.

### Task 3 — Exit status (Concepts §1.5)

```bash
grep root /etc/passwd > /dev/null
echo "match: $?"                      # 0

grep nosuchuser /etc/passwd
echo "no match: $?"                   # 1  (not an error: it searched fine)

grep root /etc/nosuchfile 2> /dev/null
echo "error: $?"                      # 2  (could not search)
```

**`$?` holds only the latest result:**

```bash
echo "a distraction"
echo "exit status now: $?"            # 0: this is echo's own status, not grep's
```

**Not found, not executable, and signals:**

```bash
nosuchcommand 2> /dev/null
echo "not found: $?"                  # 127

printf '#!/bin/bash\necho hi\n' > noexec.sh   # no execute permission yet
./noexec.sh
echo "not executable: $?"             # 126  (bash: ./noexec.sh: Permission denied)

bash -c 'exit 5'
echo "chosen status: $?"              # 5

bash -c 'kill -INT $$'
echo "SIGINT: $?"                     # 130  (128 + 2)

bash -c 'kill -KILL $$'
echo "SIGKILL: $?"                    # 137  (128 + 9; bash may also print "Killed")
```

`$$` is the process ID of that child shell, so the child sends the
signal to itself, and the parent sees the `128+N` status.

**The status of a pipeline:**

```bash
false | true
echo "last command: $?"               # 0
false | true
echo "all: ${PIPESTATUS[@]}"          # 1 0
```

### Task 4 — `&&`, `||` and `;` (Concepts §1.6)

```bash
ls real.txt > /dev/null && echo "real.txt exists"
# real.txt exists

ls missing.txt 2> /dev/null || echo "missing.txt does not exist"
# missing.txt does not exist

ls missing.txt 2> /dev/null && echo "this never prints"
echo "exit status: $?"                # 2  (the && skipped echo, so $? is still ls's)

false ; echo "runs anyway"            # ; always continues
grep -q root /etc/passwd && echo "root found"

ls missing.txt 2> /dev/null && echo A || echo B     # B
ls real.txt > /dev/null && echo A || echo B         # A
true && false || echo C                              # C  (a && b || c is not if/else)
```

`grep -q` prints nothing; the exit status alone drives the `&&`.

### Task 5 — Pipes, and why `xargs` exists (Concepts §1.7 and §1.8)

```bash
mkdir -p ../xargs-lab && cd ../xargs-lab
touch a.tmp b.tmp c.tmp
```

**A pipe carries stdout only:**

```bash
ls *.tmp | wc -l
# 3

ls a.tmp missing.tmp | wc -l
# ls: cannot access 'missing.tmp': No such file or directory   ← stderr skipped the pipe
# 1                                                             ← only stdout was counted
```

**Why a pipe into `rm` fails:**

```bash
ls *.tmp | rm
# rm: missing operand
# Try 'rm --help' for more information.

ls *.tmp
# a.tmp  b.tmp  c.tmp       ← nothing was deleted
```

**Bridging the gap with `xargs`:**

```bash
ls *.tmp | xargs -t rm -v
# rm -v a.tmp b.tmp c.tmp        ← the command xargs built (stderr, from -t)
# removed 'a.tmp'
# removed 'b.tmp'
# removed 'c.tmp'
```

**The pipeline's status belongs to `xargs`:**

```bash
touch a.tmp
ls a.tmp | xargs rm
echo "status: $?  all: ${PIPESTATUS[@]}"
# status: 0  all: 0 0
```

### Task 6 — `-n` vs. `-I{}` on the same input (Concepts §1.8)

```bash
printf 'a b c\nd e\n' | xargs
# a b c d e

printf 'a b c\nd e\n' | xargs echo "run:"
# run: a b c d e                  ← 1 run, all items

printf 'a b c\nd e\n' | xargs -n2 echo "run:"
# run: a b
# run: c d
# run: e                          ← 3 runs, 2 words each

printf 'a b c\nd e\n' | xargs -n1 echo "run:"
# run: a
# run: b
# run: c
# run: d
# run: e                          ← 5 runs, 1 word each

printf 'a b c\nd e\n' | xargs -I{} echo "run: [{}]"
# run: [a b c]
# run: [d e]                      ← 2 runs, 1 whole line each
```

**Where `-I{}` is the only choice (the item used twice):**

```bash
touch x.tmp y.tmp
printf 'x.tmp\ny.tmp\n' | xargs -I{} mv {} {}.bak
ls *.bak
# x.tmp.bak  y.tmp.bak
```

**Combining `-n` with `-I` triggers a warning:**

```bash
printf 'a b\n' | xargs -n2 -I{} echo "[{}]"
# xargs: warning: options --max-args and --replace/-I/-i are mutually exclusive, ignoring previous --max-args value
# [a b]
```

### Task 7 — `-r` and `-P` (Concepts §1.8)

**Empty input:**

```bash
printf '' | xargs echo hi
# hi                              ← ran once with no input

printf '' | xargs rm
# rm: missing operand
echo "status: $?"                 # 123

printf '' | xargs -r rm
echo "status: $?"                 # 0  (nothing ran)
```

**Parallel runs:**

```bash
time printf '1\n2\n3\n' | xargs -I{} sh -c 'sleep 1; echo done-{}'
# done-1 / done-2 / done-3          real about 0m3.0s

time printf '1\n2\n3\n' | xargs -P3 -I{} sh -c 'sleep 1; echo done-{}'
# done lines in any order           real about 0m1.0s
```

**Observation:** three 1-second jobs took about 3 seconds one at a
time, and about 1 second in parallel. The `done-N` order may change
from run to run.

### Task 8 — The whitespace footgun and the fix (Concepts §1.8)

```bash
mkdir -p spacedir
touch "spacedir/has space.txt" spacedir/normal.txt

echo "--- broken: default whitespace splitting ---"
find spacedir -name "*.txt" | xargs -n1 echo "found:"
# found: spacedir/has
# found: space.txt
# found: spacedir/normal.txt

echo "--- fixed: NUL-delimited ---"
find spacedir -name "*.txt" -print0 | xargs -0 -n1 echo "found:"
# found: spacedir/has space.txt
# found: spacedir/normal.txt
```

Now the destructive version, and what the split does to it:

```bash
find spacedir -name "*.txt" | xargs rm -v
# rm: cannot remove 'spacedir/has': No such file or directory
# rm: cannot remove 'space.txt': No such file or directory
# removed 'spacedir/normal.txt'
echo "status: $?"                   # 123

find spacedir -name "*.txt" -print0 | xargs -0 rm -v
# removed 'spacedir/has space.txt'
echo "status: $?"                   # 0
```

The first command silently skipped the file with the space. The
`-print0` / `-0` pair removed it correctly. The `123` is explained in
Task 9.

### Task 9 — `xargs` exit status (Concepts §1.8)

```bash
printf "ok\nbad\nok2\n" | xargs -I{} sh -c '[ "{}" = "bad" ] && exit 5 || echo "{}"'
echo "status: $?"
# ok
# ok2
# status: 123              ← not 5; xargs kept going after the failure
```

```bash
printf "1\n2\n3\n" | xargs -I{} sh -c 'echo run-{}; exit 255'
echo "status: $?"
# run-1
# xargs: sh: exited with status 255; aborting
# status: 124              ← stopped after the first run
```

Make the failing item visible, since `xargs` itself will not tell you:

```bash
printf "ok\nbad\nok2\n" | xargs -I{} sh -c 'if [ "{}" = "bad" ]; then echo "FAILED: {}" >&2; exit 5; fi; echo "{}"'
# ok
# FAILED: bad
# ok2
```

**Clean up and leave bash (optional, since `src/` is git-ignored):**

```bash
cd ../..                  # back to 02-redirection-exit-codes/
rm -rf src/redir-lab src/xargs-lab
exit                      # only if you started bash from zsh in Step 0
```

---

## What You Learned

- ✅ Defined process, argument, terminal, stream and file descriptor, and told arguments from stdin
- ✅ Split stdout from stderr with `>`, `2>`, `&>` and `/dev/null`, plus `>>`, `<` and `>&2`
- ✅ Told the two uses of `&` apart, and saw `2>1` create a file named `1`
- ✅ Reproduced the `2>&1` order problem and explained it as a left-to-right snapshot
- ✅ Read the exit status with `$?`, matched 0, 1, 2, 126, 127, 130 and 137 to their meanings, and saw that a pipeline reports its last command
- ✅ Chained commands with `&&`, `||` and `;`
- ✅ Saw why a pipe into `rm` fails and how `xargs` bridges stdin to arguments
- ✅ Compared `-n` and `-I{}` on the same input, and used `-t`, `-r` and `-P`
- ✅ Reproduced the whitespace footgun with `rm` and fixed it with `-print0` / `-0`
- ✅ Saw `xargs` exit `123` and `124`, and made the failing item visible

**Key Takeaway:** redirection and exit status make a script's behavior
verifiable instead of assumed. You should always be able to answer
precisely where the output went and whether each step worked.

---

## Part 3 — Break-Fix Scenario

### Scenario — NovaTech's log-capture script only captures half the failure

You are debugging a nightly script meant to capture a backup job's
complete output, errors included, in one log file:

```bash
run_backup.sh 2>&1 > /var/log/novatech/backup.log
```

The alert:

> *"The backup failed at 2am. `backup.log` shows the normal progress
> messages but nothing about the actual error. We need the error to
> know what broke, and it isn't in the file."*

**Reproduce it** (from `src/redir-lab`, with `ls` standing in for the
backup job: one result, one error):

```bash
ls real.txt missing.txt 2>&1 > backup.log
cat backup.log
```

**Your task (answer before opening the resolution):**

1. Why is the error missing from `backup.log`, even though the script uses `2>&1`?
2. Where did the error message actually go?
3. What is the one-line fix?

<details>
<summary>Resolution — open after attempting</summary>

**1. Why the error is missing:**

Redirections are read left to right (§1.4). `2>&1` ran first, so
stderr was pointed at wherever stdout pointed *at that moment*, which
was still its original destination. Only then did `> backup.log`
redirect stdout to the file. Stderr was already attached to the
original destination and did not follow.

**2. Where the error went:**

To whatever stdout was connected to before the redirection. In your
terminal that is the screen, as you saw in the repro. Under cron it is
a pipe that cron captures; cron mails that output to the crontab owner
if mail is configured, and otherwise it is lost. Either way, nobody
looking at `backup.log` ever sees it.

**3. The fix: reverse the order:**

```bash
run_backup.sh > /var/log/novatech/backup.log 2>&1
```

Redirect stdout to the file first, *then* point stderr at "wherever
stdout points now", which is already the file. In the repro this is
Task 2's `good.txt`.

**The lesson:** `2>&1` is a one-time snapshot, not a link between two
streams. When you want both captured together, write the file
redirection first.

</details>

---

## Interview Prep

**Q1. What is the difference between `2>&1 > file` and `> file 2>&1`? Which do you want?**

Redirections are applied left to right, and `2>&1` copies where
stdout points *at that moment*. In `> file 2>&1`, stdout goes to the
file first and stderr is then pointed at it, so both streams are in
the file. In `2>&1 > file`, stderr is copied to the screen first,
then only stdout moves to the file. To capture everything in one
place, write `> file 2>&1`.

---

**Q2. What does the `&` mean in `2>&1`, and how is it different from the `&` in `&>`?**

In `2>&1`, `&` means "the stream with this number": `&1` is file
descriptor 1, not a file named `1` (without `&`, `2>1` creates a file
called `1`). In `&>`, the `&` means "both": stdout and stderr together
go to the file. `&> file` is a bash shortcut for `> file 2>&1`.

---

**Q3. A script exits and `$?` is `127`. What does that tell you versus a plain `1`?**

`127` means "command not found": the shell could not locate an
executable by that name (typo, missing from `PATH`, or not installed).
`1` means the command ran and reported a failure itself. The number
tells you immediately whether to look for a missing tool or a failed
run.

---

**Q4. `grep -q pattern file` returns `1`. Is the script broken?**

Not necessarily. For `grep`, `1` means "searched successfully, found
no match", while `2` means a real error such as a missing file. A
script should treat them differently: "no match" may be a normal
outcome, while `2` needs attention.

---

**Q5. Why does `ls | rm` fail, and what does `xargs` change?**

A pipe sends text to the next command's stdin, but `rm` takes its
targets as arguments and never reads stdin, so it reports
`missing operand`. `xargs` reads stdin, splits it into items, and runs
`rm` with those items as arguments.

---

**Q6. When would you use `xargs -I{}` instead of `xargs -n`?**

When the item must go in a specific position or be used more than once,
such as `xargs -I{} mv {} {}.bak`. `-I{}` reads whole lines and puts
each one wherever `{}` appears. `-n` reads words and always appends
them at the end of the command. They cannot be combined.

---

**Q7. `xargs` exits `123` after a batch job where one item failed. How do you find out which, and why can't you tell from `$?`?**

`123` is a fixed "at least one run failed" signal, not the
sub-command's own status, so which item failed and why is lost at that
level. Make the sub-command report it before exiting, for example
`xargs -I{} sh -c 'cmd {} || echo "FAILED: {}" >&2'`.

---

## Key Takeaways

1. **Every process starts with three streams (0 stdin, 1 stdout, 2 stderr).** stdout and stderr are separate so results and errors never have to share a channel. Redirection changes where a stream is connected, before the command starts.
2. **`&` has two meanings in redirection.** `&>` means "both streams". In `2>&1` and `>&2` it means "the stream with this number", which is why `2>1` (no `&`) creates a file named `1`.
3. **`2>&1` is a one-time snapshot of where stdout points at that moment, read left to right.** `> file 2>&1` captures both streams; `2>&1 > file` silently leaves stderr on the screen.
4. **`$?` holds the exit status of the most recent command.** Check it immediately. Zero is success; 126 is not executable; 127 is not found; `128+N` is death by signal N (130 Ctrl-C, 137 SIGKILL).
5. **`grep` returns `1` for "no match" and `2` for an error.** A non-zero status is not always a failure.
6. **A pipeline's `$?` is the last command's status.** `PIPESTATUS` keeps them all.
7. **`&&` runs the next command only on success; `||` only on failure.** `a && b || c` is not if/else.
8. **`xargs` exists because a pipe carries text on stdin while most commands read arguments.** It reads stdin, splits it into items, and builds a command with the items as arguments.
9. **`-n` groups words and appends them at the end; `-I{}` takes whole lines and places them at `{}`.** Never combine them. Use `-t` to preview, `-r` for possibly empty input, `-P` to parallelize.
10. **`xargs`'s default whitespace splitting is the same footgun as unquoted variables.** `find -print0 | xargs -0` is the safe pairing for filenames that may contain spaces.
11. **`xargs` exits `123` (some run failed, keeps going) or `124` (a run exited 255, stops).** Neither is the sub-command's real status, so the sub-command itself must report the failing item.

---

## Quick Commands Reference

### Redirection operators

| Operator | Effect |
|---|---|
| `> file` | stdout → file, overwrite |
| `>> file` | stdout → file, append |
| `< file` | stdin ← file |
| `2> file` | stderr → file |
| `&> file` | stdout and stderr → file (bash) |
| `> file 2>&1` | both streams → file (portable, correct order) |
| `2>&1` | stderr → wherever stdout points now |
| `>&2` | this command's stdout → stderr |
| `> /dev/null` | discard output |

### Exit status

| Status | Meaning |
|---|---|
| `0` | Success |
| `1` | General failure; `grep`: no match |
| `2` | Misuse or error; `grep`: error |
| `126` | Found but not executable |
| `127` | Command not found |
| `130` | Ctrl-C (SIGINT) |
| `137` | SIGKILL (signal 9) |
| `143` | SIGTERM (signal 15) |

### Chaining

| Form | Meaning |
|---|---|
| `a && b` | run `b` only if `a` succeeded |
| `a \|\| b` | run `b` only if `a` failed |
| `a ; b` | run `b` always |

### `xargs`

| Flag | Meaning of the letter | Effect |
|---|---|---|
| `-n` | number | Max items per run, appended at the end |
| `-I` | replace-string | One whole line per run, placed at `{}` (do not combine with `-n`) |
| `-t` | trace | Print the command before running it |
| `-r` | no-run-if-empty | Do not run if the input is empty |
| `-P` | max-procs | Run in parallel (order not guaranteed) |
| `-0` | n/a | Split on NUL (pair with `find -print0`) |

---

## Appendix A — Anki Cards

**`m01-02-redirection-exit-codes-anki.csv`:**

```
#deck:Linux Mastery::Module 1 - Filesystem & Navigation::Demo M01-02 - Redirection & Exit Status
#separator:Comma
#columns:Front,Back,Tags

"What are the three standard streams, and why are stdout and stderr separate?","0 = stdin (input), 1 = stdout (normal output), 2 = stderr (error output). They are separate so results and errors never have to share one channel: you can capture output to a file while still seeing errors on screen, or the reverse.","demo-m01-02,redirection,filedescriptors"

"What is redirection?","Changing where one of a process's streams is connected: to a file, from a file, or to /dev/null, instead of the keyboard or screen. The shell sets it up before the command starts; the command does not know.","demo-m01-02,redirection,definition"

"What is the difference between > file and >> file?","> overwrites the file (or creates it); >> appends to the end of the file.","demo-m01-02,redirection,append"

"What are the two meanings of & in redirection?","In &> it means both streams: stdout and stderr together (bash shortcut for > file 2>&1). In 2>&1 and >&2 it means the stream with this number (a file descriptor), not a file name. Without it, 2>1 sends stderr to a file named 1.","demo-m01-02,redirection,ampersand"

"What does >&2 do, and when do you use it?","It sends a command's stdout to stderr. Scripts use it to print their own error messages (echo ""error"" >&2) so they do not mix into the normal results.","demo-m01-02,redirection,stderr"

"Why does 'cmd > file 2>&1' capture both streams, but 'cmd 2>&1 > file' only captures stdout?","Redirections are applied left to right and 2>&1 copies where stdout points AT THAT MOMENT. In '> file 2>&1' stdout is already the file when stderr is copied to it. In '2>&1 > file' stderr is copied to the screen first, then stdout moves to the file and stderr does not follow.","demo-m01-02,redirection,order,2>&1"

"What is an exit status, and what is $?","The exit status (also called exit code or return code) is the number a process returns when it finishes: 0 means success, 1-255 means failure. $? is a special parameter that holds the exit status of the most recent command.","demo-m01-02,exitstatus,dollar-question"

"$? shows 0 after grep matched, but 0 again after a plain echo run in between. Is grep's status still known?","No. $? holds only the MOST RECENT command's status. After echo ran, $? is echo's own status. Read $? immediately after the command you care about.","demo-m01-02,exitstatus,overwritten"

"grep exits 1 in one case and 2 in another. What does each mean?","1 = searched fine but found no match (often a normal outcome). 2 = a real error, such as a missing file. Scripts should not treat them the same.","demo-m01-02,exitstatus,grep"

"What do exit statuses 126, 127 and 130 mean?","126 = file found but not executable (permissions). 127 = command not found. 130 = terminated by Ctrl-C (SIGINT): 128 + signal number 2.","demo-m01-02,exitstatus,126,127,130"

"What does an exit status of 137 mean, and how is it derived?","The process was killed by signal 9 (SIGKILL), such as kill -9 or the kernel OOM killer. Statuses above 128 mean 128 + signal number: 128 + 9 = 137 (SIGTERM gives 143).","demo-m01-02,exitstatus,137,signals"

"What does $? hold after 'false | true', and where can you see every command's status?","$? holds the LAST command's status, so 0 here. Bash keeps all of them in the array PIPESTATUS (echo ${PIPESTATUS[@]} prints 1 0).","demo-m01-02,exitstatus,pipeline"

"What do 'a && b' and 'a || b' do, and is 'a && b || c' an if/else?","a && b runs b only if a exited 0. a || b runs b only if a exited non-zero. a ; b always runs b. a && b || c is NOT if/else: c also runs when b fails.","demo-m01-02,exitstatus,chaining"

"Why does 'ls *.tmp | rm' fail with 'missing operand', and what does xargs change?","A pipe carries text on the next command's stdin, but rm reads its targets from arguments and never reads stdin. xargs reads stdin, splits it into items and runs rm with those items as arguments: ls *.tmp | xargs rm.","demo-m01-02,xargs,why"

"For the input lines 'a b c' and 'd e', what do xargs echo, xargs -n2 echo and xargs -I{} echo [{}] produce?","xargs echo: one run, 'a b c d e'. xargs -n2 echo: three runs, 'a b' / 'c d' / 'e' (words, 2 at a time, appended at the end). xargs -I{} echo [{}]: two runs, '[a b c]' / '[d e]' (one whole line per run, placed at {}).","demo-m01-02,xargs,dash-n,dash-I"

"When must you use xargs -I{} instead of -n, and can you combine them?","When the item must be in a specific position or used more than once, e.g. xargs -I{} mv {} {}.bak. -n always appends words at the end. They cannot be combined: xargs warns the options are mutually exclusive and ignores -n.","demo-m01-02,xargs,dash-I"

"What does xargs -P3 do, and what is the catch?","It runs up to 3 runs at the same time instead of one after another, so three 1-second jobs take about 1 second instead of 3. The catch: jobs finish in any order, so output order is not guaranteed.","demo-m01-02,xargs,dash-P,parallel"

"What does xargs do when its input is empty, and how do you prevent it?","GNU xargs still runs the command once with no arguments (which makes 'xargs rm' fail with 'missing operand', status 123). Use -r (--no-run-if-empty) so nothing runs on empty input.","demo-m01-02,xargs,dash-r"

"find dir -name '*.txt' | xargs -n1 echo found: splits 'has space.txt' into two. Why, and what is the fix?","xargs splits on whitespace by default, the same footgun as an unquoted variable. Fix: find dir -name '*.txt' -print0 | xargs -0 -n1 echo found: (NUL bytes cannot appear in filenames).","demo-m01-02,xargs,footgun,print0"

"What do xargs exit statuses 123 and 124 mean?","123 = at least one run exited with a status from 1 to 125; xargs keeps processing the rest. 124 = a run exited with 255; xargs stops immediately. Neither is the sub-command's real status, so the sub-command must report the failing item itself.","demo-m01-02,xargs,exitstatus,123,124"
```

---

## Appendix B — Quiz

**`m01-02-redirection-exit-codes-quiz.md`:**

```
# Quiz — Demo M01-02: I/O Redirection, File Descriptors & Exit Status

> One correct answer per question. Target: 80% (10/12) before Demo M01-03.
> Assume bash.

---

**Q1.** You run `cmd 2>&1 > output.log`. Where does `cmd`'s stderr go?

- A) `output.log`, same as stdout
- B) The screen: `2>&1` copied stdout's location before stdout was redirected to the file
- C) `/dev/null`
- D) A separate `output.log.err` file

<details>
<summary>Answer</summary>

**B**. `2>&1` is a snapshot of where stdout points at that moment,
which is still the screen. Writing `> output.log 2>&1` captures both.

</details>

---

**Q2.** What does `ls missing.txt 2>1` do?

- A) Sends stderr to stdout
- B) Sends stderr to a file named `1`
- C) Discards stderr
- D) Sends stdout to a file named `2`

<details>
<summary>Answer</summary>

**B**. Without `&`, the `1` is read as a file name. `2>&1` (with `&`)
is the form that means "the stream numbered 1".

</details>

---

**Q3.** `grep "pattern" file.txt` runs, then `echo "checking..."`
runs, then you read `$?`. What does `$?` hold?

- A) `grep`'s exit status
- B) `echo`'s exit status: `$?` holds only the most recent command's status
- C) A combined status of both
- D) An error, since `$?` can only be read once

<details>
<summary>Answer</summary>

**B**. Every command overwrites `$?`, including `echo`. Read it
immediately after the command you care about.

</details>

---

**Q4.** Running `./deploy.sh` gives exit status `126`. What does that mean?

- A) The command was not found in `PATH`
- B) The file exists but does not have the execute permission
- C) The script succeeded with warnings
- D) The script was stopped by Ctrl-C

<details>
<summary>Answer</summary>

**B**. `126` is "found but not executable". `127` is "not found" and
`130` is Ctrl-C.

</details>

---

**Q5.** `grep -q nosuchuser /etc/passwd; echo $?` prints `1`. What does that mean?

- A) `grep` crashed
- B) The file `/etc/passwd` does not exist
- C) `grep` searched successfully and found no match
- D) `grep` was not found

<details>
<summary>Answer</summary>

**C**. For `grep`, `1` means no match; `2` would mean an error such as
a missing file; `127` would mean `grep` was not found.

</details>

---

**Q6.** `false | true; echo $?` prints what?

- A) `1`
- B) `0`
- C) `1 0`
- D) Nothing

<details>
<summary>Answer</summary>

**B**. `$?` holds the status of the **last** command in the pipeline.
`echo ${PIPESTATUS[@]}` would show `1 0`.

</details>

---

**Q7.** What is printed by `ls missing.txt 2> /dev/null && echo A || echo B`
when `missing.txt` does not exist?

- A) `A`
- B) `B`
- C) Both `A` and `B`
- D) Nothing

<details>
<summary>Answer</summary>

**B**. `ls` fails, so `&&` skips `echo A`, and `||` runs `echo B`
because the chain's last status was non-zero.

</details>

---

**Q8.** Why does `ls *.tmp | rm` print `rm: missing operand`?

- A) `rm` cannot delete `.tmp` files
- B) A pipe feeds the next command's stdin, but `rm` takes its targets as arguments
- C) `ls` printed nothing
- D) `rm` needs `-f`

<details>
<summary>Answer</summary>

**B**. `xargs` bridges the gap: `ls *.tmp | xargs rm`.

</details>

---

**Q9.** The input is the two lines `a b c` and `d e`. What does
`xargs -n2 echo "run:"` print?

- A) `run: a b c d e`
- B) `run: a b` / `run: c d` / `run: e`
- C) `run: [a b c]` / `run: [d e]`
- D) `run: a b c` / `run: d e`

<details>
<summary>Answer</summary>

**B**. `-n2` groups words two at a time and appends them to the
command, ignoring line boundaries. C is what `-I{}` gives (one whole
line per run).

</details>

---

**Q10.** `find dir -name "*.log" | xargs -n1 rm` runs where a file is
named `backup copy.log`. What happens?

- A) The file is deleted normally
- B) `xargs` splits the name into `backup` and `copy.log`, and `rm` fails on both because those files do not exist
- C) `xargs` refuses to run
- D) Only `backup` is deleted

<details>
<summary>Answer</summary>

**B**. This is the whitespace-splitting footgun. The fix is
`find dir -name "*.log" -print0 | xargs -0 -n1 rm`.

</details>

---

**Q11.** A batch `xargs` job has one item that exits `5`. What does
`echo $?` show after the pipeline?

- A) `5`
- B) `123`: a fixed "at least one run failed" status
- C) `1`
- D) `0`

<details>
<summary>Answer</summary>

**B**. `xargs` exits `123` when any run returns 1 to 125, and keeps
processing. (If a run exits `255`, `xargs` stops and exits `124`.) To
learn which item failed, the sub-command must report it.

</details>

---

**Q12.** Three independent jobs each sleep 1 second. About how long
does `xargs -P3` take to run them all?

- A) About 3 seconds
- B) About 1 second
- C) About 0 seconds
- D) About 6 seconds

<details>
<summary>Answer</summary>

**B**. `-P3` runs up to three at the same time, so the sleeps overlap.
Without `-P` they run one after another (about 3 seconds).

</details>

---

Score guide:

| Score | Action |
|---|---|
| 12/12 | Import Anki cards, move to Demo M01-03 |
| 11/12 | Review the wrong answer, then proceed |
| 9-10/12 | Re-read the relevant section, retry those questions |
| Below 9/12 | Re-read the full demo and redo the lab |
```