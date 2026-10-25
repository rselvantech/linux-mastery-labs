# Demo M01-04 — Shell Startup Files

#### <ins>1.1.1 The terms</ins>

## Overview

You add a line to `.bashrc` to fix your `PATH` once and for all, and
a cron job that needs that `PATH` still fails, because cron never
reads `.bashrc`. You SSH in and run a single remote command, and your
aliases are missing, because that command runs in a shell that skips
most startup files.

None of this is random. Bash decides which startup files to load from
a fixed, checkable set of conditions. Once you know the rule, "why
didn't my config apply here" stops being a mystery.

| Concept | The question it answers |
|---|---|
| Shell types | Which kind of shell am I in, and how do I check? |
| Which files load | Which file does each kind of shell read, and in what order? |
| Special cases | What are `BASH_ENV`, the sshd rule, and Ubuntu's guard line? |
| Jobs and the standard pattern | Why do cron and systemd miss my settings, and how do I fix it? |

| Concepts | Lab |
|---|---|
| Part 1 — Shell Types | Part 1 — Shell Types |
| Part 2 — Which Files Load | Part 2 — Which Files Load |
| Part 3 — Special Cases | Part 3 — Special Cases |
| Part 4 — Scheduled Jobs, Services and the Standard Pattern | Part 4 — Scheduled Jobs, Services and the Standard Pattern |

---

## Demo Objectives

By the end of this demo you will be able to:

1. Distinguish a login shell from a non-login shell, and an interactive shell from a non-interactive one, as two independent questions, and check which one you are in
2. Explain the difference between sourcing a file and running it as a script
3. State which user file loads for each of the four combinations, and demonstrate it directly
4. Explain that system-wide files load first, list what a real shell reads, and start a shell with no startup files
5. Explain what `BASH_ENV` is, what the sshd special case is, and why the default Ubuntu `.bashrc` can still skip your settings
6. Explain why cron jobs and systemd services do not pick up `.bashrc`, reproduce their environment, and fix it where the job runs
7. Implement the standard pattern of sourcing `.bashrc` from the login file, and avoid the Ubuntu trap of a new `.bash_profile` hiding `.profile`

---

## Directory Structure

```
04-shell-startup-files/
├── README.md                            # this file
├── m01-04-shell-startup-files-anki.csv  # Anki flashcard deck (Appendix A)
├── m01-04-shell-startup-files-quiz.md   # standalone quiz (Appendix B)
└── src/                                 # lab working directories (git-ignored)
    └── startup-lab/                     # a fake home directory used by every task
        └── bin/                         # a fake tool used in Part 4
```

All lab work happens inside `src/`. The lab commands create the
folders (`mkdir -p`), so you only need to be in the demo directory
(`04-shell-startup-files/`) when you start.

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

Load order is bash behavior. The default files differ by distro:

| | Ubuntu | Rocky |
|---|---|---|
| Login file shipped in `~` | `~/.profile` (it sources `~/.bashrc`) | `~/.bash_profile` (it sources `~/.bashrc`) |
| `~/.bashrc` | Starts with a guard that returns early unless the shell is interactive | Sources `/etc/bashrc`; no such guard |
| System-wide interactive file | `/etc/bash.bashrc` | `/etc/bashrc` |

> **Distro:** check your own defaults with `cat ~/.bash_profile ~/.profile ~/.bashrc 2>/dev/null | head -40`.

The lab uses a **fake home directory** (`HOME=$PWD bash ...`), so your
real startup files are never touched. Every command starts its own
bash, so it does not matter which shell your terminal is running.

---

## Recall Check — Demo M01-03

Before starting, answer from memory:

1. An alias works when you type it in a terminal, but not inside `bash -c '...'`. Why?
2. What are the exit statuses for "command not found" and "found but not executable"?
3. A new copy of a tool is installed earlier in `PATH`, but an open shell still runs the old one. What fixes it?

<details>
<summary>Answers</summary>

1. Aliases are expanded only in interactive shells by default. A function works in both.
2. `127` and `126`.
3. `hash -r`, because bash cached the old copy's path.

</details>

---

## Concepts

### Part 1 — Shell Types

What login and interactive mean, how to tell which kind of shell you
are in, and the vocabulary used throughout this demo.

---

#### 1.1 Before you start: the vocabulary

##### 1.1.1 The terms

| Term | Meaning |
|---|---|
| **Session** | One period of use: from logging in (or connecting by SSH) until you log out |
| **Login shell** | The first shell of a session, started when you log in |
| **Non-login shell** | Any shell started later from inside a session, such as a new terminal tab or a script |
| **Interactive shell** | A shell reading commands you type at a prompt (Demo M01-03) |
| **Non-interactive shell** | A shell running a script or a `bash -c '...'` command, with no prompt |
| **Startup file** | A file bash reads automatically when it starts, to set up `PATH`, aliases and so on. Also called an init file or **rc file** ("run commands") |
| **Source** | To run a file's commands **inside the current shell** (§1.1.2) |
| **Home directory** | Your personal directory, stored in `$HOME` and written `~` |

```
   You log in or connect
         |
         v
   +---------------+
   |  login shell  |  ---- reads login startup files ---->  you get a prompt
   +-------+-------+
           |  you open a new tab, run a script, start cron job ...
           v
   +---------------+
   | non-login     |  ---- reads different files (or none) ---->  work happens
   |    shell      |
   +---------------+
```

##### 1.1.2 Sourcing vs. running a script

A startup file only helps if its settings land in **your** shell.
That is why startup files are **sourced** (with `.` or `source`), not
run like a script:

| How you run it | Where it runs | Do its variables survive? |
|---|---|---|
| `bash setvar.sh` | In a **child** process | No: they vanish when the child exits |
| `. setvar.sh` (or `source setvar.sh`) | In the **current** shell | Yes |

```bash
printf 'X=from-file\n' > setvar.sh
bash setvar.sh
echo "after running as a script: [$X]"
# after running as a script: []

. setvar.sh
echo "after sourcing: [$X]"
# after sourcing: [from-file]
```

##### 1.1.3 `HOME` and the fake-home trick

Bash looks for `~/.bashrc` and friends under the directory in
`$HOME`. In the lab, `HOME=$PWD bash ...` is a one-shot variable
(Demo M01-03): it gives that single child bash a different `HOME`, so it
reads the lab's files and never your real ones.

```bash
HOME=/some/dir bash -c 'echo ~'
# /some/dir               ← ~ follows HOME
```

---

#### 1.2 Two independent questions

##### 1.2.1 The grid

Every bash is classified by **two separate questions**:

- **Login or non-login?** Was this shell started as the first shell of a session?
- **Interactive or non-interactive?** Is it reading commands at a prompt, or running a script or a single command?

| | Login | Non-login |
|---|---|---|
| **Interactive** | `ssh host` (a prompt); `su -`; `sudo -i`; a console login | A new tab in most Linux GUI terminals; typing `bash` in a shell; `su user`; `sudo -s` |
| **Non-interactive** | `bash -l -c 'cmd'` (rare) | A script; a cron job; `bash -c 'cmd'` |

##### 1.2.2 Common mix-ups

People often assume login and interactive are the same thing. They are
not:

| Situation | Login? | Interactive? |
|---|---|---|
| Desktop terminal tab | No | Yes |
| SSH session with a prompt | Yes | Yes |
| Cron job | No | No |
| `ssh host 'cmd'` (a single remote command) | No | No, with a twist (§3.2) |

> **Note:** `tmux` starts **login** shells in new windows by default, so a tmux window is not the same as a desktop terminal tab.

##### 1.2.3 Checking which kind of shell you are in

Two commands answer both questions:

| Command | Prints | Meaning |
|---|---|---|
| `shopt login_shell` | `login_shell     on` or `off` | `on` = login shell |
| `echo $-` | A string of flag letters | An `i` in it = interactive |

```bash
bash -c 'shopt login_shell; echo "flags: $-"'
# login_shell     off
# flags: hBc                ← no i: non-interactive

bash -i -c 'shopt login_shell; echo "flags: $-"'
# login_shell     off
# flags: hiBHc              ← has i: interactive
```

---

### Part 2 — Which Files Load

Which startup file each kind of shell reads and in what order, and
how to see it for yourself on a real system.

---

#### 2.1 The startup files at a glance

##### 2.1.1 System-wide and user files

| File | Kind | Read by |
|---|---|---|
| `/etc/profile` | System-wide | Login shells (it also loads every `/etc/profile.d/*.sh`) |
| `~/.bash_profile`, `~/.bash_login`, `~/.profile` | User, login | Login shells (first found wins, §2.2.2) |
| `/etc/bash.bashrc` (Ubuntu), `/etc/bashrc` (Rocky) | System-wide, interactive | Non-login interactive shells |
| `~/.bashrc` | User, interactive | Non-login interactive shells |
| `~/.bash_logout` | User | An interactive login shell when it exits |

System-wide files load **before** your own, and are where system-wide
`PATH` additions and locale settings usually come from.

---

#### 2.2 Login shells

##### 2.2.1 Load order

```
   login shell starts
        |
        v
   /etc/profile  --->  /etc/profile.d/*.sh            (system-wide)
        |
        v
   first found of:   ~/.bash_profile    found? use it and STOP
                     ~/.bash_login      only if the one above is missing
                     ~/.profile         only if both above are missing
        |
        v
   your command, or your prompt
```

To see this directly, put a marker in each file of a fake home:

```bash
echo 'echo "LOADED: .bash_profile"' > .bash_profile
echo 'echo "LOADED: .profile"'      > .profile
echo 'echo "LOADED: .bashrc"'       > .bashrc

HOME=$PWD bash -l -c 'echo body ran'
# Login shell, in order:
# 1. /etc/profile (which loads /etc/profile.d/*.sh)
# 2. first found of ~/.bash_profile, ~/.bash_login, ~/.profile
#    → .bash_profile found: prints LOADED, STOP (the other two are ignored)
# 3. the command: echo body ran
# LOADED: .bash_profile
# body ran
```

##### 2.2.2 The first-found rule

Bash reads **only one** of the three user login files. With
`.bash_profile` removed, it falls through the chain:

```bash
rm .bash_profile
HOME=$PWD bash -l -c 'echo body ran'
# LOADED: .profile
# body ran
```

---

#### 2.3 Non-login interactive shells

##### 2.3.1 Load order

```
   Ubuntu:   /etc/bash.bashrc  --->  ~/.bashrc  --->  your prompt

   Rocky:    ~/.bashrc  (which itself sources /etc/bashrc)  --->  your prompt
```

```bash
HOME=$PWD bash -i -c 'echo body ran'
# Non-login interactive shell, in order:
# 1. Ubuntu: /etc/bash.bashrc, then ~/.bashrc
#    Rocky : ~/.bashrc, which itself sources /etc/bashrc
# 2. the command: echo body ran
# LOADED: .bashrc
# body ran
```

---

#### 2.4 Non-interactive shells

##### 2.4.1 Nothing loads by default

A plain script, `bash -c 'cmd'` or a cron job reads **no startup
file**:

```
   non-interactive, non-login:     (nothing)  --->  your command
                                   BASH_ENV set?  --->  that file first   (§3.1)
```

```bash
HOME=$PWD bash -c 'echo body ran'
# Non-login non-interactive shell:
# 1. nothing is read (no BASH_ENV, no sshd special case)
# 2. the command: echo body ran
# body ran
```

---

#### 2.5 Seeing what a real shell loads

##### 2.5.1 Tracing with `bash -x`

`-x` (e**x**ecution trace: print every command as it runs) shows what
a real login shell sources:

```bash
bash -l -x -c true 2>&1 | grep -E '^\++ (\.|source) '
# + . /etc/profile.d/01-locale-fix.sh
# + . /home/you/.bashrc
```

That is an example; your list depends on your distro and files. The
trace shows files sourced *by other files*. Bash reads `/etc/profile`
and your `~/.profile` itself, so they do not appear as lines.

##### 2.5.2 Starting a shell with no startup files

Three options control which files bash reads:

| Option | Meaning |
|---|---|
| `--noprofile` | Do not read the login files |
| `--norc` | Do not read `~/.bashrc` |
| `--rcfile FILE` | Read `FILE` instead of `~/.bashrc` |

> **Gotcha:** long options (`--norc`) must come **before** short ones (`-i`). `bash --norc -i` works; `bash -i --norc` is an error (`bash: --: invalid option`).

---

### Part 3 — Special Cases

The three exceptions to the load rules: `BASH_ENV`, the sshd special
case, and Ubuntu's guard line.

---

#### 3.1 `BASH_ENV`

##### 3.1.1 Loading a file in a plain script

If the variable `BASH_ENV` is set, even a plain non-interactive shell
sources that file first:

```bash
echo 'echo "LOADED via BASH_ENV"' > envfile.sh

BASH_ENV=$PWD/envfile.sh HOME=$PWD bash -c 'echo body ran'
# 1. non-interactive shell, BASH_ENV is set → bash sources envfile.sh first
#    (prints LOADED via BASH_ENV)
# 2. the command: echo body ran
# LOADED via BASH_ENV
# body ran
```

It is rarely set on purpose, but it explains a script that behaves
differently on two servers: one has `BASH_ENV` set in its cron or
systemd configuration and the other does not.

---

#### 3.2 The sshd special case

##### 3.2.1 What happens

When sshd runs a single remote command, it starts bash
non-interactively. Bash has a special case for exactly this
situation: if it is a **fresh top-level shell** started by sshd (it can
tell from the `SSH_CLIENT` variable, or because its input is a network
socket), it **reads `~/.bashrc` anyway**.

```
   ssh host 'cmd'
        |
        v
   sshd starts:  bash -c 'cmd'     non-login, non-interactive, fresh shell, SSH_CLIENT set
        |
        v
   special case: bash reads ~/.bashrc anyway
        |
        v
   .bashrc runs ...  (until its guard line, see §3.3)
        |
        v
   cmd runs
```

> **Distro:** Debian and Ubuntu build bash this way (verified on Ubuntu 24.04). Run Task 3.2 on your own system to see whether yours does.

##### 3.2.2 Why the lab uses `env -i`

The special case only triggers in a **fresh** shell. From inside an
existing terminal, nesting hides it. `env -i` (**i**gnore environment:
start the command with an empty environment) makes the lab shell fresh,
like the one sshd starts.

---

#### 3.3 The Ubuntu guard line

##### 3.3.1 Why reading `.bashrc` is not enough

Even when `~/.bashrc` is read, your settings may not run. Ubuntu's
default `~/.bashrc` begins with a guard that stops reading the file
unless the shell is interactive:

```bash
case $- in
    *i*) ;;          # interactive: carry on
    *) return;;      # anything else: stop reading this file now
esac
```

##### 3.3.2 Above and below the guard

Everything **above** the guard runs in the sshd case. Everything
**below** it (your aliases, your `PATH` changes) is skipped:

```bash
# .bashrc:  echo "above the guard"  →  guard  →  echo "below the guard"

HOME=$PWD bash -i -c 'echo body ran'
# above the guard
# below the guard
# body ran

env -i PATH=/usr/bin:/bin SSH_CLIENT="1.2.3.4 5 6" HOME=$PWD bash -c 'echo body ran'
# 1. fresh top-level shell + SSH_CLIENT set → the sshd special case: read ~/.bashrc anyway
# 2. .bashrc line 1: echo "above the guard"          → prints
# 3. .bashrc guard : $- has no i → `return`           → bash STOPS reading .bashrc
# 4. .bashrc lines below the guard                    → never read
# 5. the command: echo body ran
# above the guard
# body ran
```

> **Warning:** a non-interactive remote shell that **prints** anything while reading `~/.bashrc` can corrupt `scp` and `sftp` transfers, because the output mixes into the data stream. Never `echo` from `.bashrc` outside the interactive part.

---

### Part 4 — Scheduled Jobs, Services and the Standard Pattern

Why cron jobs and systemd services miss your settings, how to
reproduce and fix that, and the standard pattern that gives every
interactive shell the same configuration.

---

#### 4.1 cron

##### 4.1.1 What cron runs

Cron runs each job with `/bin/sh -c` (unless the crontab sets
`SHELL=`) and gives it its own small environment, typically
`PATH=/usr/bin:/bin`. **No startup file is read.**

```
   you SSH in   --->  interactive login shell  --->  reads startup files  --->  PATH has /opt/aws-cli/bin  --->  aws found

   cron fires   --->  /bin/sh -c 'job'         --->  reads NO startup file --->  PATH=/usr/bin:/bin          --->  aws: not found
```

A `PATH` addition that lives only in `.bashrc` is invisible to cron.
That is why "it works when I run it by hand, but not from cron" is so
common.

---

#### 4.2 systemd

##### 4.2.1 What systemd runs

systemd starts the `ExecStart` program **directly, with no shell at
all**, so there are no startup files to read. Environment must be
declared in the unit:

```ini
[Service]
Environment=PATH=/opt/aws-cli/bin:/usr/local/bin:/usr/bin:/bin
ExecStart=/opt/deploy.sh
```

---

#### 4.3 Reproducing a job's environment

##### 4.3.1 `env -i`

You can reproduce cron's environment on the command line with `env -i`:

```bash
env -i PATH=/usr/bin:/bin bash -c 'echo "PATH=$PATH"; command -v aws'
# PATH=/usr/bin:/bin
#                          ← aws not found
```

---

#### 4.4 The fix: set it where the job runs

##### 4.4.1 Three places to set it

| Where | How |
|---|---|
| The crontab | A `PATH=` line at the top (and `SHELL=/bin/bash` if the job needs bash) |
| The script | `export PATH=...` at the top of the script |
| The systemd unit | `Environment=PATH=...` |

> **Rule:** never fix "works by hand, fails from cron" by editing `.bashrc` again. Set the value where the job actually runs.

---

#### 4.5 The standard pattern: source `.bashrc` from the login file

##### 4.5.1 Why it is needed

Login shells do not read `.bashrc`, but most people want their aliases
and `PATH` in every interactive shell. The standard fix is a login file
that sources it:

```bash
# inside ~/.bash_profile
if [ -f "$HOME/.bashrc" ]; then
  . "$HOME/.bashrc"
fi
```

Distros already ship this: Rocky's `~/.bash_profile`, and Ubuntu's
`~/.profile`.

##### 4.5.2 Following the order

```bash
HOME=$PWD bash -l -c 'command -v aws-fake'
# 1. /etc/profile (and /etc/profile.d/*.sh)
# 2. ~/.bash_profile found → STOP (.profile is ignored). Its lines run in order:
#      a. echo "LOADED: .bash_profile"   → prints
#      b. . "$HOME/.bashrc"              → .bashrc runs: export PATH="…/bin:$PATH"
# 3. the command: command -v aws-fake → found, because PATH now includes bin
```

---

#### 4.6 The Ubuntu trap

##### 4.6.1 A new `.bash_profile` hides `.profile`

Ubuntu ships no `~/.bash_profile`. If you create one, login shells read
it **instead of** `~/.profile` (the first-found rule), so everything
`~/.profile` did, including the `~/.local/bin` `PATH` logic and loading
`.bashrc`, silently stops.

##### 4.6.2 The fix

If you must create a `~/.bash_profile` on Ubuntu, have it source
`~/.profile`:

```bash
# ~/.bash_profile on Ubuntu
if [ -f "$HOME/.profile" ]; then
  . "$HOME/.profile"
fi
```

---

## Hands-On Lab

### Setup

#### Step 0 — Make sure you are in bash

```bash
echo $0
```

If it prints `zsh`, switch to bash for this lab (every command below
also starts its own bash, so this is a safety check):

```bash
bash
```

Run `echo $0` again to confirm. Type `exit` at the end of the lab to
return to your normal shell.

---

### Part 1 — Shell Types

Check which kind of shell you are in, and see the difference between
sourcing and running a file.

| Command | Flag meaning | Purpose |
|---|---|---|
| `bash -l` | `l` = **l**ogin | Start bash as a login shell |
| `bash -i` | `i` = **i**nteractive | Start bash as an interactive shell |
| `bash -c 'cmd'` | `c` = **c**ommand | Run one command non-interactively, then exit |
| `shopt login_shell` | n/a | Show whether this shell is a login shell |
| `echo $-` | n/a | Show the shell's flags (an `i` means interactive) |
| `.` (dot) | n/a | Source a file into the current shell (same as `source`) |

---

#### Task 1.1 — Which shell am I in? (Concepts §1.2)

```bash
# from the demo directory: 04-shell-startup-files/
mkdir -p src/startup-lab && cd src/startup-lab

bash -c 'shopt login_shell; echo "flags: $-"'
# login_shell     off
# flags: hBc

bash -l -c 'shopt login_shell; echo "flags: $-"'
# login_shell     on
# flags: hBc

bash -i -c 'shopt login_shell; echo "flags: $-"'
# login_shell     off
# flags: hiBHc

bash -l -i -c 'shopt login_shell; echo "flags: $-"'
# login_shell     on
# flags: hiBHc
```

**Observation:** `-l` flips `login_shell`, and `-i` adds the `i` flag.
The two switches are independent, which is why there are four
combinations.

---

#### Task 1.2 — Sourcing vs. running (Concepts §1.1)

```bash
printf 'X=from-file\n' > setvar.sh
unset X

bash setvar.sh
echo "after running as a script: [$X]"
# after running as a script: []

. setvar.sh
echo "after sourcing: [$X]"
# after sourcing: [from-file]

unset X
```

**Observation:** the same file set `X` both times, but only sourcing
put it in your shell.

---

### Part 2 — Which Files Load

Reproduce each load order with marker files, then see what a real
shell loads and how to start one with none.

| Command | Flag meaning | Purpose |
|---|---|---|
| `bash -x` | `x` = e**x**ecution trace | Print every command as it runs |
| `bash --noprofile` | n/a | Do not read the login files |
| `bash --norc` | n/a | Do not read `~/.bashrc` |
| `bash --rcfile FILE` | n/a | Read `FILE` instead of `~/.bashrc` |
| `HOME=$PWD cmd` | n/a | One-shot `HOME` for a single command (Demo M01-03) |

---

#### Task 2.1 — The four combinations (Concepts §2.2 to §2.4)

```bash
echo 'echo "LOADED: .bash_profile"' > .bash_profile
echo 'echo "LOADED: .profile"'      > .profile
echo 'echo "LOADED: .bashrc"'       > .bashrc

echo "--- login, non-interactive"
HOME=$PWD bash -l -c 'echo body ran'
# LOADED: .bash_profile
# body ran

echo "--- login, interactive"
HOME=$PWD bash -l -i -c 'echo body ran'
# LOADED: .bash_profile
# body ran

echo "--- non-login, interactive"
HOME=$PWD bash -i -c 'echo body ran'
# LOADED: .bashrc
# body ran

echo "--- non-login, non-interactive"
HOME=$PWD bash -c 'echo body ran'
# body ran
```

**Observation:** both login shells loaded only `.bash_profile`, even
though `.profile` and `.bashrc` also exist.

---

#### Task 2.2 — The first-found chain (Concepts §2.2.2)

```bash
rm .bash_profile
HOME=$PWD bash -l -c 'echo body ran'
# LOADED: .profile
# body ran

echo 'echo "LOADED: .bash_login"' > .bash_login
HOME=$PWD bash -l -c 'echo body ran'
# LOADED: .bash_login          ← .bash_login now beats .profile
# body ran

rm .bash_login
```

---

#### Task 2.3 — What does a real shell load? (Concepts §2.5.1)

This one uses your real home directory and only reads, it does not
change anything:

```bash
bash -l -x -c true 2>&1 | grep -E '^\++ (\.|source) '
```

Expect lines for the files sourced by other files, such as
`/etc/profile.d/*.sh` and your own `~/.bashrc`. Which files appear
depends on your distro.

---

#### Task 2.4 — Starting a shell with no startup files (Concepts §2.5.2)

```bash
HOME=$PWD bash -i -c 'echo body ran'
# LOADED: .bashrc
# body ran

HOME=$PWD bash --norc -i -c 'echo body ran'
# body ran                         ← .bashrc skipped

HOME=$PWD bash --noprofile -l -c 'echo body ran'
# body ran                         ← login file skipped

echo 'echo "LOADED: custom"' > custom.rc
HOME=$PWD bash --rcfile custom.rc -i -c 'echo body ran'
# LOADED: custom
# body ran                         ← custom.rc read instead of .bashrc

bash -i --norc -c 'echo body ran'
# bash: --: invalid option         ← a long option after -i is an error
```

---

### Part 3 — Special Cases

Reproduce `BASH_ENV`, the sshd special case and the guard line.

| Command | Flag meaning | Purpose |
|---|---|---|
| `env -i` | `i` = **i**gnore environment | Run a command with an empty environment |
| `BASH_ENV=file` | n/a | Make a non-interactive bash source `file` first |
| `SSH_CLIENT=...` | n/a | The variable that tells bash it was started by sshd |

---

#### Task 3.1 — `BASH_ENV` (Concepts §3.1)

```bash
echo 'echo "LOADED via BASH_ENV"' > envfile.sh

HOME=$PWD bash -c 'echo body ran'
# body ran                          ← plain script: nothing loads

BASH_ENV=$PWD/envfile.sh HOME=$PWD bash -c 'echo body ran'
# LOADED via BASH_ENV
# body ran
```

---

#### Task 3.2 — The sshd special case and the guard line (Concepts §3.2 and §3.3)

Write a `.bashrc` with a guard in the middle:

```bash
printf 'echo "above the guard"\ncase $- in\n    *i*) ;;\n    *) return;;\nesac\necho "below the guard"\n' > .bashrc
```

```bash
echo "--- interactive"
HOME=$PWD bash -i -c 'echo body ran'
# above the guard
# below the guard
# body ran

echo "--- plain non-interactive"
HOME=$PWD bash -c 'echo body ran'
# body ran

echo "--- started the way sshd starts it"
env -i PATH=/usr/bin:/bin SSH_CLIENT="1.2.3.4 5 6" HOME=$PWD bash -c 'echo body ran'
# above the guard
# body ran                           ← .bashrc was read, but stopped at the guard
```

**Observation:** in the last case `.bashrc` was read (the special
case), but only the part above the guard ran. If your distro prints
nothing extra here, its bash is built without the special case, which
is also a valid result to note.

---

### Part 4 — Scheduled Jobs, Services and the Standard Pattern

Reproduce cron's environment gap, fix it where the job runs, then
implement the standard pattern and see the Ubuntu trap.

| Command | Flag meaning | Purpose |
|---|---|---|
| `env -i` | `i` = **i**gnore environment | Simulate cron's empty environment |
| `command -v NAME` | `v` = describe | Show what a name resolves to (Demo M01-03) |
| `.` (dot) | n/a | Source a file into the current shell |

---

#### Task 4.1 — Reproduce the cron gap (Concepts §4.1 to §4.4)

Make a fake tool, and a `.bashrc` that adds its directory to `PATH`:

```bash
mkdir -p bin
printf '#!/bin/bash\necho fake-aws\n' > bin/aws-fake
chmod +x bin/aws-fake
printf 'export PATH="%s/bin:$PATH"\n' "$PWD" > .bashrc
```

```bash
echo "--- interactive: .bashrc loads, the tool is found"
HOME=$PWD bash -i -c 'command -v aws-fake'
# …/src/startup-lab/bin/aws-fake

echo "--- cron-like: no startup file, minimal PATH"
env -i PATH=/usr/bin:/bin HOME=$PWD bash -c 'echo "PATH=$PATH"; command -v aws-fake; echo "exit status: $?"'
# PATH=/usr/bin:/bin
# exit status: 1                     ← not found

echo "--- the fix: set PATH where the job runs"
env -i PATH="$PWD/bin:/usr/bin:/bin" bash -c 'command -v aws-fake; aws-fake'
# …/src/startup-lab/bin/aws-fake
# fake-aws
```

---

#### Task 4.2 — The standard pattern and the Ubuntu trap (Concepts §4.5 and §4.6)

Right now `.profile` exists (from Task 2.1) but `.bash_profile` does not:

```bash
HOME=$PWD bash -l -c 'command -v aws-fake; echo "exit status: $?"'
# LOADED: .profile
# exit status: 1                     ← the login shell never read .bashrc
```

Create a `.bash_profile` that sources `.bashrc`:

```bash
printf 'echo "LOADED: .bash_profile"\nif [ -f "$HOME/.bashrc" ]; then\n  . "$HOME/.bashrc"\nfi\n' > .bash_profile

HOME=$PWD bash -l -c 'command -v aws-fake'
# LOADED: .bash_profile
# …/src/startup-lab/bin/aws-fake     ← found: the login shell now loads .bashrc too

HOME=$PWD bash -c 'command -v aws-fake; echo "exit status: $?"'
# exit status: 1                     ← a non-interactive shell still does not
```

**Observation:** creating `.bash_profile` hid `.profile`: the
`LOADED: .profile` line is gone. This is the Ubuntu trap from §4.6.

**Clean up (optional, since `src/` is git-ignored):**

```bash
cd ../..                  # back to 04-shell-startup-files/
rm -rf src/startup-lab
exit                      # only if you started bash from zsh in Step 0
```

---

## What You Learned

- ✅ Told login from non-login and interactive from non-interactive, and checked a live shell with `shopt login_shell` and `$-`
- ✅ Saw that sourcing keeps a file's variables in your shell, while running it as a script does not
- ✅ Reproduced all four combinations, and the `.bash_profile` → `.bash_login` → `.profile` fallback
- ✅ Listed what a real shell sources, and started shells with `--norc`, `--noprofile` and `--rcfile`
- ✅ Used `BASH_ENV`, and reproduced the sshd special case and the guard line that stops `.bashrc` early
- ✅ Reproduced cron's empty-environment gap with `env -i` and fixed it where the job runs
- ✅ Implemented the `.bash_profile` → `.bashrc` pattern, and saw a new `.bash_profile` hide `.profile`

**Key Takeaway:** "works by hand, fails from cron" is almost always a
mismatch between where a setting lives and where the job runs. Manual
runs go through an interactive shell that reads your files. Cron and
systemd do not. Set what a job needs in the job itself.

---

## Break-Fix Scenario

**Scenario — NovaTech: the deploy script works by hand, fails from cron**

You are handed this ticket:

> *"Our nightly deploy script fails every time cron runs it at 2am,
> with `aws: command not found`. If I SSH in and run the same script
> by hand, it works every time. Nobody changed the script. `aws` is
> installed: I can see it in `/opt/aws-cli/bin`, and that path is in
> my `.bashrc`."*

**Reproduce it** (from `src/startup-lab`, using Task 4.1's fake tool):

```bash
HOME=$PWD bash -i -c 'command -v aws-fake'                 # works: "by hand"
env -i PATH=/usr/bin:/bin HOME=$PWD bash -c 'command -v aws-fake; echo "exit status: $?"'   # fails: "cron"
```

**Your task (answer before opening the resolution):**

1. Why does the script work by hand but fail from cron, when nothing about the script changed?
2. Will editing `.bashrc` again, or double-checking the path in it, fix this?
3. What is the actual fix?

<details>
<summary>Resolution — open after attempting</summary>

**1. Why it only works by hand:**

By hand, you SSH in and get an interactive login shell. It reads your
startup files (which load `.bashrc`), so `PATH` gets
`/opt/aws-cli/bin` and `aws` resolves. Cron starts the job with
`/bin/sh -c` and its own small environment, typically
`PATH=/usr/bin:/bin`. No startup file is read, so `/opt/aws-cli/bin`
is never added.

**2. Editing `.bashrc` will not help:**

The path in `.bashrc` was never wrong. The file is simply never read
in cron's context. No amount of editing a file that is not loaded
changes anything.

**3. The fix: set `PATH` where the job runs:**

```bash
# at the top of the crontab (and SHELL=/bin/bash if the job needs bash):
SHELL=/bin/bash
PATH=/opt/aws-cli/bin:/usr/local/bin:/usr/bin:/bin

# or at the top of the script itself:
export PATH="/opt/aws-cli/bin:$PATH"
```

**The lesson:** "works when I run it manually" is a signal to check
*how* the job normally runs. Cron, systemd and non-interactive SSH
commands do not rely on your interactive startup files, and fixing a
file the job never reads cannot help.

</details>

---

## Interview Prep

**Q1. A teammate adds a `PATH` export to `.bashrc` to fix a missing tool for a systemd service. The service still cannot find the tool. Why?**

systemd starts the `ExecStart` program directly, with no shell, so no
startup file is ever read. (Even with `ExecStart=/bin/bash -c '...'`,
that shell is non-interactive and non-login.) Declare the environment
in the unit instead, for example `Environment=PATH=...`.

---

**Q2. What is the difference between `ssh host` and `ssh host 'some-command'` for what loads?**

`ssh host` starts an interactive login shell, so the login file loads
(and `.bashrc` too, if that file sources it). `ssh host 'cmd'` starts
a non-interactive shell. On Debian and Ubuntu, bash's sshd special
case still reads `~/.bashrc`, but Ubuntu's default `.bashrc` returns
at its guard line, so anything below the guard (aliases, `PATH`
changes) is missing for that command.

---

**Q3. Why do login shells not load `.bashrc` by default?**

By design: the login file is for one-time session setup, and
`.bashrc` is for per-shell interactive settings that every new shell
should get. To get both in a login shell, the login file sources
`.bashrc`, which is why distros ship that line already.

---

**Q4. You create `~/.bash_profile` on Ubuntu to set a variable, and suddenly `~/.local/bin` tools stop being found in SSH sessions. Why?**

Login shells read the first found of `~/.bash_profile`,
`~/.bash_login`, `~/.profile`. The new `.bash_profile` is found first,
so Ubuntu's `~/.profile` (which adds `~/.local/bin` and sources
`.bashrc`) is no longer read. Have the new file source `~/.profile`.

---

**Q5. How do you test a script the way cron will run it?**

Use `env -i PATH=/usr/bin:/bin bash -c './script.sh'`. `env -i` starts
the command with an empty environment, so only the `PATH` you pass is
present and no startup file is read. If it fails there, it will fail
in cron.

---

## Key Takeaways

### Part 1 — Shell Types

1. **Login vs. non-login and interactive vs. non-interactive are two independent questions.** `shopt login_shell` and the `i` in `$-` tell you the answers for the shell you are in.
2. **Sourcing (`. file`) runs a file in the current shell; running it (`bash file`) uses a child.** Startup files are sourced so their settings stay in your shell.

### Part 2 — Which Files Load

3. **Login shells read the first found of `~/.bash_profile`, `~/.bash_login`, `~/.profile`; non-login interactive shells read `~/.bashrc`; plain non-interactive shells read nothing.** System-wide files such as `/etc/profile` and `/etc/profile.d/*.sh` load before the user file.
4. **`bash --norc`, `--noprofile` and `--rcfile` control what loads, and long options must come first.**

### Part 3 — Special Cases

5. **`BASH_ENV` is sourced even by a plain non-interactive shell.** It is the explicit exception.
6. **A fresh bash started by sshd for a remote command reads `~/.bashrc` (on Debian and Ubuntu builds).** Ubuntu's default `.bashrc` then stops at a guard line, so settings below it are skipped.

### Part 4 — Scheduled Jobs, Services and the Standard Pattern

7. **Cron and systemd do not read your startup files.** Cron uses `/bin/sh` and a minimal `PATH`; systemd starts the program with no shell at all.
8. **The fix is never "edit `.bashrc` again".** Set the value where the job runs: the crontab, the unit file, or the top of the script. `env -i` reproduces such an environment.
9. **The standard pattern is a login file that sources `.bashrc`.** On Ubuntu, creating `~/.bash_profile` hides `~/.profile` unless it sources it.

---

## Quick Commands Reference

### Which user file loads

| Shell type | File loaded |
|---|---|
| Login | `~/.bash_profile` → `~/.bash_login` → `~/.profile` (first found, stops) |
| Non-login, interactive | `~/.bashrc` |
| Non-login, non-interactive | None, unless `BASH_ENV` is set (or the sshd special case applies) |

### Invocation flags

| Flag | Meaning |
|---|---|
| `bash -l` | **l**ogin shell |
| `bash -i` | **i**nteractive shell |
| `bash -c 'cmd'` | **c**ommand: non-interactive, run one command and exit |
| `bash -x` | e**x**ecution trace: print every command as it runs |
| `bash --noprofile` | Skip the login files |
| `bash --norc` | Skip `~/.bashrc` |
| `bash --rcfile FILE` | Read `FILE` instead of `~/.bashrc` |
| `env -i cmd` | **i**gnore environment: run `cmd` with an empty environment (simulates cron or sshd) |

### Check your shell

| Command | Shows |
|---|---|
| `shopt login_shell` | `on` for a login shell |
| `echo $-` | Flags; `i` means interactive |

### Standard pattern (in `~/.bash_profile`)

```bash
if [ -f "$HOME/.bashrc" ]; then
  . "$HOME/.bashrc"
fi
```

---

## Appendix A — Anki Cards

**`m01-04-shell-startup-files-anki.csv`:**

```
#deck:Linux Mastery::Module 1 - Filesystem & Navigation::Demo M01-04 - Shell Startup Files
#separator:Comma
#columns:Front,Back,Tags

"What are the two independent questions bash uses to classify a shell?","Login vs non-login, and interactive vs non-interactive. All four combinations exist: ssh host = login + interactive; a GUI terminal tab = non-login + interactive; a cron job or script = non-login + non-interactive; bash -l -c 'cmd' = login + non-interactive.","demo00d,startup,axes"

"How do you check whether your current shell is a login shell, and whether it is interactive?","shopt login_shell prints on (login) or off. echo $- prints the flags; an i in them means interactive.","demo00d,startup,check"

"What is a startup file, and what is another name for it?","A file bash reads automatically when it starts, to set up PATH, aliases and so on. Also called an init file or rc file (run commands).","demo00d,startup,vocabulary"

"What is the difference between sourcing a file and running it as a script?","Sourcing (. file or source file) runs the file's commands in the CURRENT shell, so its variables survive. Running it (bash file) uses a child process, so its variables vanish when it exits. Startup files are sourced.","demo00d,startup,source"

"A login shell has .bash_profile, .bash_login and .profile all present. Which loads?","Only .bash_profile. Bash uses the FIRST found of .bash_profile, .bash_login, .profile and stops; it never loads more than one of the three.","demo00d,startup,fallback"

"Which file does a non-login interactive shell read, and which does a plain non-interactive shell read?","Non-login interactive: ~/.bashrc (on Ubuntu /etc/bash.bashrc first; on Rocky ~/.bashrc sources /etc/bashrc). Plain non-interactive (script, cron): none, unless BASH_ENV is set.","demo00d,startup,bashrc"

"What loads BEFORE your own login file in a login shell?","The system-wide /etc/profile, which loads every /etc/profile.d/*.sh. That is where system-wide PATH and locale settings usually come from.","demo00d,startup,etc-profile"

"What is BASH_ENV?","An environment variable naming a file that bash sources even for a plain non-interactive, non-login shell. It is the explicit exception to 'scripts load no startup file'.","demo00d,startup,bash_env"

"What is the sshd special case?","A fresh top-level bash started by sshd for a remote command (it sees SSH_CLIENT set, or a network socket on stdin) reads ~/.bashrc even though it is non-interactive. Debian and Ubuntu builds do this.","demo00d,startup,ssh"

"What does the guard at the top of Ubuntu's default ~/.bashrc do?","case $- in *i*) ;; *) return;; esac makes bash stop reading .bashrc unless the shell is interactive. Lines above it run in the sshd case; lines below it (aliases, PATH changes) are skipped.","demo00d,startup,guard"

"What environment does cron give a job, and which startup files does it read?","Cron runs the job with /bin/sh -c (unless SHELL= is set) and a small environment, typically PATH=/usr/bin:/bin. It reads no startup file, so settings only in .bashrc are invisible.","demo00d,startup,cron"

"A systemd service cannot find a tool whose path is in .bashrc. Why, and what is the fix?","systemd starts ExecStart directly with no shell, so no startup file is ever read. Declare the environment in the unit, e.g. Environment=PATH=....","demo00d,startup,systemd"

"What does env -i do, and why use it in this demo?","env -i (ignore environment) runs a command with an empty environment. It reproduces cron's or sshd's fresh start: only the variables you pass are present and no inherited PATH or SHLVL.","demo00d,startup,env-i"

"What do bash --norc, --noprofile and --rcfile do, and what is the ordering gotcha?","--norc skips ~/.bashrc, --noprofile skips the login files, --rcfile FILE reads FILE instead of ~/.bashrc. Long options must come BEFORE short ones: bash --norc -i works, bash -i --norc is an error.","demo00d,startup,options"

"When does ~/.bash_logout run?","When an interactive login shell exits. It is not run for bash -l -c 'cmd'.","demo00d,startup,logout"

"Why is the standard fix for 'aliases missing after SSH login' to make the login file source .bashrc, and what is the Ubuntu trap?","Login shells read the login file, not .bashrc, so the login file sources .bashrc (if [ -f ~/.bashrc ]; then . ~/.bashrc; fi). Trap: on Ubuntu, creating ~/.bash_profile hides ~/.profile (first found wins), so it must source ~/.profile.","demo00d,startup,pattern,ubuntu"
```

---

## Appendix B — Quiz

**`m01-04-shell-startup-files-quiz.md`:**

```
# Quiz — Demo M01-04: Shell Startup Files

> One correct answer per question. Target: 80% (8/10) before Demo M01-05.
> Assume bash.

---

**Q1.** What does `bash -l -c 'shopt login_shell; echo "flags: $-"'` print?

- A) `login_shell     off` and `flags: hBc`
- B) `login_shell     on` and `flags: hBc`
- C) `login_shell     on` and `flags: hiBHc`
- D) `login_shell     off` and `flags: hiBHc`

<details>
<summary>Answer</summary>

**B**. `-l` makes it a login shell (`on`), and `-c` makes it
non-interactive, so there is no `i` in the flags.

</details>

---

**Q2.** A fake home holds `.bash_profile`, `.profile` and `.bashrc`, each
printing its own name. What does `HOME=$PWD bash -l -c 'echo body ran'` print?

- A) `LOADED: .bash_profile`, `LOADED: .profile`, `LOADED: .bashrc`, then `body ran`
- B) `LOADED: .bash_profile`, then `body ran`
- C) `LOADED: .bashrc`, then `body ran`
- D) Only `body ran`

<details>
<summary>Answer</summary>

**B**. A login shell reads the first found of the three login files and
stops, and it does not read `.bashrc`.

</details>

---

**Q3.** `BASH_ENV=$PWD/envfile.sh bash -c 'echo body ran'` is run, where
`envfile.sh` prints `LOADED via BASH_ENV`. What is the output?

- A) `body ran`
- B) `LOADED via BASH_ENV`, then `body ran`
- C) `body ran`, then `LOADED via BASH_ENV`
- D) An error

<details>
<summary>Answer</summary>

**B**. `BASH_ENV` is sourced first, even for a plain non-interactive shell.

</details>

---

**Q4.** A cron job fails with "command not found" for a tool whose path is
in `.bashrc`. The same script works by hand. What is the correct fix?

- A) Re-check the spelling of the path in `.bashrc`
- B) Move the `PATH` line to `.profile`
- C) Set `PATH` in the crontab or at the top of the script
- D) Restart cron so it re-reads `.bashrc`

<details>
<summary>Answer</summary>

**C**. Cron runs with `/bin/sh` and a minimal environment and reads no
startup file, so the fix must be where the job runs.

</details>

---

**Q5.** On Ubuntu, `ssh host 'echo $PATH'` runs with the default `~/.bashrc`.
A `PATH` line sits below the guard
(`case $- in *i*) ;; *) return;; esac`). Does it take effect?

- A) Yes: `.bashrc` always runs fully for SSH commands
- B) No: bash reads `.bashrc` (the sshd special case), but the guard stops it before that line
- C) No: bash never reads `.bashrc` for SSH commands
- D) Yes, but only the first time

<details>
<summary>Answer</summary>

**B**. Everything above the guard runs, and everything below it does
not, because the shell is non-interactive.

</details>

---

**Q6.** On Ubuntu you create `~/.bash_profile` containing one `export`
line. After the next SSH login, `~/.local/bin` tools are missing. Why?

- A) `export` lines break `PATH`
- B) Login shells now read `.bash_profile` instead of `.profile`, so Ubuntu's `.profile` no longer runs
- C) `.bash_profile` is only read for non-login shells
- D) `~/.local/bin` is removed on each login

<details>
<summary>Answer</summary>

**B**. First-found wins, so the new file hides `.profile`. Have it
source `~/.profile`.

</details>

---

**Q7.** Which command starts an interactive shell that does **not** read
`~/.bashrc`?

- A) `bash -i --norc`
- B) `bash --norc -i`
- C) `bash -i --noprofile`
- D) `bash --rcfile -i`

<details>
<summary>Answer</summary>

**B**. Long options must come before short ones. A is an error, and
`--noprofile` (C) only skips the login files.

</details>

---

**Q8.** A systemd service cannot find a tool whose path is only in
`.bashrc`. What is the fix?

- A) Add `Requires=bashrc.service`
- B) Declare the environment in the unit, for example `Environment=PATH=...`
- C) Reboot so the service re-reads shell config
- D) Move `.bashrc` into `/etc`

<details>
<summary>Answer</summary>

**B**. systemd starts the program directly with no shell, so no startup
file is read.

</details>

---

**Q9.** `printf 'X=1\n' > setvar.sh` is created. What do these print?
`bash setvar.sh; echo "[$X]"` and then `. setvar.sh; echo "[$X]"`
(with `X` unset at the start).

- A) `[1]` and `[1]`
- B) `[]` and `[1]`
- C) `[1]` and `[]`
- D) `[]` and `[]`

<details>
<summary>Answer</summary>

**B**. Running with `bash` uses a child, so `X` never reaches your
shell. Sourcing with `.` runs in the current shell.

</details>

---

**Q10.** `.bashrc` adds `~/bin` to `PATH` (below any guard). What does
`env -i PATH=/usr/bin:/bin bash -c 'command -v mytool'` print for a tool
that lives only in `~/bin`?

- A) The path to `mytool`
- B) Nothing, with exit status `1`
- C) `command not found`, status `127`
- D) An alias definition

<details>
<summary>Answer</summary>

**B**. No startup file is read, so `~/bin` is not in `PATH` and
`command -v` finds nothing (it exits `1`).

</details>

---

Score guide:

| Score | Action |
|---|---|
| 10/10 | Import Anki cards, move to Demo M01-05 |
| 9/10 | Review the wrong answer, then proceed |
| 7-8/10 | Re-read the relevant section, retry those questions |
| Below 7/10 | Re-read the full demo and redo the lab |
```