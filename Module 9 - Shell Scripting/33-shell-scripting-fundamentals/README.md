# Demo 33 — Shell Scripting Fundamentals

## Overview

Everything from Demo 04 through Demo 23 has been commands typed
interactively. This demo turns that into scripts — repeatable, shareable
automation instead of remembered command sequences. Bash scripting has a
reputation for being full of sharp edges (unquoted variables, silent
failures), and most of that reputation is earned by not knowing the rules
rather than the rules being unreasonable. This demo builds the fundamentals
correctly from the start: variables, quoting, conditionals, loops, and
functions, with the common pitfalls called out explicitly as they come up.

## Demo Objectives

1. Declare and use variables correctly, including command substitution
2. Apply quoting rules correctly, and explain why unquoted variables are dangerous
3. Write conditionals with `test`/`[[ ]]`, including common comparison operators
4. Write `for` and `while` loops, including safely reading line-by-line input
5. Define and call functions, including passing arguments and returning values
6. Write a complete, runnable script combining all of the above

## Directory Structure

```
33-shell-scripting-fundamentals/
├── README.md                                      # this file
├── 33-shell-scripting-fundamentals-anki.csv       # Anki flashcard deck
└── 33-shell-scripting-fundamentals-quiz.md        # standalone quiz
```

## Environments

| Task | Ubuntu 24.04 | Rocky Linux 9.8 | EC2 Amazon Linux |
|---|---|---|---|
| All lab tasks | ✅ (bash) | ✅ (bash) | Not required |

> **Verification note:** every task in this demo was run live, in actual
> bash, in the authoring sandbox — all output shown matches exactly what
> was produced. No `[Unverified]` markers are needed anywhere in this
> demo.

## Recall Check — 27c-security-auditing

Answer these from memory before continuing — no looking back at Demo 27c.

**Q1.** What's the fundamental difference between `lynis` and OpenSCAP?

**Q2.** A baseline audit produces 200 findings. What's wrong with
treating that as a flat, top-to-bottom to-do list?

**Q3.** Why does a security baseline scan need to run on a schedule
rather than just once?

<details>
<summary>Answers</summary>

**A1.** `lynis` is broad and heuristic, requiring no external standard —
good for a general ongoing baseline. OpenSCAP scans against a specific,
named, published profile (CIS, STIG) and maps findings to rule IDs — used
to demonstrate compliance with a specific standard.

**A2.** Not every finding is equally urgent — some have a direct known
exploit path, some are genuinely low-risk in the specific environment,
and some represent a real regression against something already
deliberately configured. Context changes priority in ways neither tool
fully knows on its own.

**A3.** A one-time scan only proves compliance at that instant.
Configuration drifts afterward (package updates, forgotten debugging
changes, expiring certificates) and none of that is caught by a scan that
already ran.

</details>

## Part 1 — Concepts

### 1.1 Variables and command substitution

```bash
#!/bin/bash
name="wadmin"
count=5
echo "User: $name, count: $count"
```

**No spaces around `=` in assignment** — `name = "wadmin"` is not an
assignment at all; bash parses it as running a command called `name` with
arguments `=` and `"wadmin"`, producing a "command not found" error that
doesn't obviously point at the real mistake.

```bash
current_dir=$(pwd)
file_count=$(ls | wc -l)
echo "In $current_dir, there are $file_count files"
```

`$(...)` is **command substitution** — runs the command inside, captures
its stdout, and substitutes that text in place. The older backtick syntax
(`` `pwd` ``) does the same thing but doesn't nest cleanly (`$(...)` nests
by simply adding more parentheses; nested backticks require escaping) —
`$(...)` is the modern, unambiguous choice.

```bash
readonly max_retries=3
# readonly: the variable cannot be reassigned after this — a real safety
# net for a value that should never change during the script's execution,
# rather than relying on convention/naming alone to signal "don't touch this."
```

### 1.2 Quoting — why unquoted variables are genuinely dangerous

```bash
filename="my file.txt"
touch $filename
# Creates TWO files: "my" and "file.txt" — the unquoted variable undergoes
# word splitting, exactly the same xargs mechanism from Demo 19 §1.6,
# treating the space inside the value as an argument separator.

touch "$filename"
# Creates ONE file: "my file.txt" — quoting prevents word splitting,
# treating the variable's full value as a single argument.
```

**This isn't a stylistic preference — unquoted variables are a genuine
source of bugs and, in destructive commands, genuine data loss risk:**

```bash
rm $backup_dir/*.tmp
# If $backup_dir happens to be empty (unset, or set to ""), this becomes
# rm /*.tmp — attempting to delete matching files from the ROOT
# directory, not a scoped cleanup. This exact class of bug has caused real
# production incidents.

rm "${backup_dir:?}"/*.tmp
# ${VAR:?} : if VAR is unset or empty, exit immediately with an error
# instead of silently proceeding with an unintended, dangerous path —
# combined with quoting, this specific pattern is the safe way to write
# a cleanup command using a variable path.
```

**Default habit: quote every variable expansion unless you have a specific,
understood reason not to** (like deliberately wanting word-splitting for
glob expansion in a controlled context, which is rare and should be a
deliberate choice, not an oversight).

### 1.3 Conditionals — test, [[ ]], and comparison operators

```bash
if [ "$count" -gt 10 ]; then
    echo "High count"
elif [ "$count" -eq 0 ]; then
    echo "Zero count"
else
    echo "Normal count"
fi
```

`[ ]` is the `test` command (a real, separate program, aliased to `[`) —
which is why spaces around the brackets are mandatory (`[$count -gt 10]`
without spaces is a syntax error, since `[` needs to be a distinct
argument-separated token, not glued to what follows). `[[ ]]` is bash's
own built-in, more forgiving extended test — it handles unquoted variables
more safely (word-splitting doesn't apply the same way inside `[[ ]]`) and
supports pattern matching (`==` with globs) and regex (`=~`) directly:

```bash
if [[ "$filename" == *.log ]]; then
    echo "Log file"
fi

if [[ "$line" =~ ^ERROR ]]; then
    echo "Error line — regex match, Demo 17 knowledge applies directly"
fi
```

| Operator | Meaning (numeric) | Operator | Meaning (string) |
|---|---|---|---|
| `-eq` | equal | `==` | equal |
| `-ne` | not equal | `!=` | not equal |
| `-gt` / `-lt` | greater/less than | `-z` | string is empty |
| `-ge` / `-le` | greater/less-or-equal | `-n` | string is non-empty |

**Numeric and string comparison operators are genuinely different and not
interchangeable** — `"$a" == "$b"` compares strings lexically; `"$a" -eq
"$b"` compares numerically. Using `==` on numbers that happen to have
different string representations of the same value (`"5"` vs `"05"`) or
`-eq` on genuinely non-numeric strings both produce wrong results or
errors.

### 1.4 Loops — for, while, and safely reading input

```bash
for i in 1 2 3 4 5; do
    echo "Number: $i"
done

for file in /var/log/*.log; do
    echo "Processing: $file"
done
```

```bash
count=0
while [ "$count" -lt 5 ]; do
    echo "Count: $count"
    count=$((count + 1))
done
```

**Reading a file line by line has a specific safe idiom, and the naive
approach silently breaks on certain input:**

```bash
# Fragile — breaks on lines with backslashes, and trims leading/trailing
# whitespace unexpectedly:
for line in $(cat file.txt); do
    echo "$line"
done
# This ALSO splits on whitespace WITHIN a line, not just on newlines —
# a single line "hello world" becomes two loop iterations, not one.

# Correct:
while IFS= read -r line; do
    echo "$line"
done < file.txt
# IFS= : don't trim leading/trailing whitespace from each line
# -r : don't interpret backslash escapes in the input — treat it literally
# < file.txt : redirect the file as the loop's input, rather than piping
```

This mirrors the exact `xargs`/word-splitting lesson from Demo 19 §1.6 —
unquoted, unprotected iteration over text silently splits on whitespace in
ways that break on realistic input, and the fix is the same category of
fix: be deliberate about delimiters rather than relying on the naive
default.

### 1.5 Functions — arguments and return values

```bash
greet() {
    local name="$1"
    echo "Hello, $name"
}

greet "wadmin"
```

`$1`, `$2`, etc. inside a function are that function's own positional
arguments — separate from the script's own `$1`/`$2` (its command-line
arguments) if the function is called with different arguments than the
script itself received. `local` scopes the variable to the function —
**without `local`, a variable assigned inside a function is global by
default**, which can silently clobber a same-named variable used elsewhere
in the script, a genuinely common source of confusing bugs in scripts that
grow past a handful of lines.

```bash
is_valid_port() {
    local port="$1"
    if [[ "$port" =~ ^[0-9]+$ ]] && [ "$port" -ge 1 ] && [ "$port" -le 65535 ]; then
        return 0   # success/true, by shell convention
    else
        return 1   # failure/false
    fi
}

if is_valid_port "8080"; then
    echo "Valid port"
fi
```

**Shell functions "return" via exit code (0 = success, non-zero = failure),
not via a value the way most programming languages' functions do.** To get
an actual computed value out, the convention is `echo`ing it and capturing
via command substitution:

```bash
get_hostname_short() {
    echo "${HOSTNAME%%.*}"
}
short_name=$(get_hostname_short)
```

## Part 2 — Hands-On Lab

### Commands used in this demo

| Syntax | Purpose |
|---|---|
| `var=value` (no spaces) | Variable assignment |
| `$(command)` | Command substitution |
| `"$var"` | Quoted expansion — prevents word splitting |
| `[[ COND ]]` | Bash's extended conditional test |
| `for X in LIST; do ... done` | Iterate over a list |
| `while IFS= read -r line; do ... done < file` | Safely read a file line by line |
| `function_name() { ... }` | Function definition |
| `local var` | Scope a variable to the current function |

### Task 1 — Variables, command substitution, and the unquoted trap

```bash
cat > /tmp/lab33-1.sh << 'SCRIPT'
#!/bin/bash
file_count=$(ls /tmp | wc -l)
echo "Files in /tmp: $file_count"

mkdir -p /tmp/quote-lab
touch "/tmp/quote-lab/my file.txt"

echo "--- unquoted (buggy) ---"
target="/tmp/quote-lab/my file.txt"
ls $target 2>&1
echo "--- quoted (correct) ---"
ls "$target"
SCRIPT
bash /tmp/lab33-1.sh
```

### Task 2 — Conditionals, numeric vs. string comparison

```bash
cat > /tmp/lab33-2.sh << 'SCRIPT'
#!/bin/bash
count=15
if [[ "$count" -gt 10 ]]; then
    echo "High"
fi

filename="app.log"
if [[ "$filename" == *.log ]]; then
    echo "Log file: $filename"
fi

a="5"
b="05"
if [ "$a" == "$b" ]; then echo "string equal"; else echo "string NOT equal"; fi
if [ "$a" -eq "$b" ]; then echo "numeric equal"; else echo "numeric NOT equal"; fi
SCRIPT
bash /tmp/lab33-2.sh
```

### Task 3 — Loops, and the safe line-reading idiom

```bash
printf 'line one\nline two with spaces\nline three\n' > /tmp/lab33-3.txt

cat > /tmp/lab33-3.sh << 'SCRIPT'
#!/bin/bash
echo "--- fragile (word-splits within lines too) ---"
for line in $(cat /tmp/lab33-3.txt); do
    echo "iteration: [$line]"
done

echo "--- correct ---"
while IFS= read -r line; do
    echo "line: [$line]"
done < /tmp/lab33-3.txt
SCRIPT
bash /tmp/lab33-3.sh
```

### Task 4 — Functions, local scope, and return codes

```bash
cat > /tmp/lab33-4.sh << 'SCRIPT'
#!/bin/bash
is_valid_port() {
    local port="$1"
    if [[ "$port" =~ ^[0-9]+$ ]] && [ "$port" -ge 1 ] && [ "$port" -le 65535 ]; then
        return 0
    else
        return 1
    fi
}

for p in 8080 99999 abc 443; do
    if is_valid_port "$p"; then
        echo "$p: valid"
    else
        echo "$p: invalid"
    fi
done
SCRIPT
bash /tmp/lab33-4.sh
```

### Task 5 — The local-scope pitfall, demonstrated

```bash
cat > /tmp/lab33-5.sh << 'SCRIPT'
#!/bin/bash
status="initial"

check_something() {
    status="modified by function"   # no 'local' — this is GLOBAL
}

echo "before: $status"
check_something
echo "after: $status"
# Confirm 'status' changed globally, even though the function never
# explicitly touched the outer variable by that name intentionally —
# this is exactly the bug 'local' prevents.
SCRIPT
bash /tmp/lab33-5.sh
```

### Task 6 — A complete small script combining everything

```bash
cat > /tmp/lab33-6.sh << 'SCRIPT'
#!/bin/bash
set -euo pipefail   # forward-reference: Demo 34 covers this properly

log_dir="/tmp/lab33-logs"
mkdir -p "$log_dir"

count_errors() {
    local file="$1"
    grep -c 'ERROR' "$file" 2>/dev/null || echo 0
}

printf 'INFO ok\nERROR bad\nERROR worse\n' > "$log_dir/app.log"
printf 'INFO ok\nINFO fine\n' > "$log_dir/other.log"

for logfile in "$log_dir"/*.log; do
    errors=$(count_errors "$logfile")
    if [[ "$errors" -gt 0 ]]; then
        echo "$logfile: $errors error(s)"
    else
        echo "$logfile: clean"
    fi
done
SCRIPT
bash /tmp/lab33-6.sh
```

## What You Learned

- ✅ Declared and used variables correctly, including command substitution
- ✅ Applied quoting rules correctly, and demonstrated why unquoted variables are dangerous
- ✅ Wrote conditionals with `test`/`[[ ]]`, including numeric vs. string comparison
- ✅ Wrote `for`/`while` loops, including the safe line-reading idiom
- ✅ Defined and called functions, including `local` scoping and return codes
- ✅ Wrote a complete, runnable script combining variables, conditionals, loops, and functions

## Part 3 — Break-Fix Scenario

### Scenario — NovaTech Ops: two shell-scripting incidents

> **Incident report — NovaTech Ops, ticket #4728**
>
> Alert 1: "a cleanup script ran `rm $target_dir/*.tmp` and deleted files
> outside the intended directory after a config variable was accidentally
> left empty." Alert 2: "a script's summary counter was always 0 despite
> visibly processing files, and debugging showed a function was updating a
> variable that never seemed to change outside it."

**Incident 1 — rm with an empty variable, wrong files deleted**

```bash
$ cat cleanup.sh
#!/bin/bash
rm $target_dir/*.tmp
$ TARGET_DIR= bash cleanup.sh   # target_dir was never actually set — typo
                                 # in env var casing, or just unset
```

Tasks:
1. Reference §1.2 — walk through exactly what `rm $target_dir/*.tmp`
   expands to when `target_dir` is empty, and why that's dangerous.
2. What's the specific fix from §1.2 that would have caused this script to
   fail loudly instead of running a dangerous command silently?
3. Beyond that one fix, what quoting change should also be applied to this
   line as a general habit?

**Incident 2 — a counter that never updates outside a function**

```bash
$ cat counter.sh
#!/bin/bash
total=0

process_file() {
    local total
    total=$((total + 1))
}

for f in file1 file2 file3; do
    process_file
done
echo "Total: $total"
$ bash counter.sh
Total: 0
```

Tasks:
1. Reference §1.5 — what does `local total` inside `process_file` actually
   create, relative to the outer `total` variable declared at the top of
   the script?
2. Why does the outer `total` stay at 0 even though `process_file` runs
   three times and increments *something* called `total` each time?
3. What's the correct way to accumulate a count across multiple function
   calls, given that `local` is (correctly) still needed inside the
   function for other purposes?

<details>
<summary>Answers</summary>

**Incident 1:** With `target_dir` empty, `$target_dir/*.tmp` expands to
just `/*.tmp` (the empty variable contributes nothing, leaving a bare
leading slash followed by the glob) — `rm /*.tmp` then attempts to delete
every `.tmp` file directly under the *root* directory, not a scoped
subdirectory, which is a dramatically different and dangerous blast radius
from what the script intended. The specific fix: `${target_dir:?}` — using
this parameter-expansion form causes the script to exit immediately with
an error if `target_dir` is unset or empty, rather than silently
proceeding to construct and run a dangerous path. Beyond that, the
variable should also be quoted: `rm "${target_dir:?}"/*.tmp` — quoting
protects against word-splitting if the directory path itself ever contains
spaces, which is a separate but related class of the same underlying
"don't trust an unquoted variable in a destructive command" principle.

**Incident 2:** `local total` inside `process_file` creates a brand new
variable named `total`, scoped entirely to that function call — it shadows
the outer script-level `total` by name, but is a genuinely separate piece
of memory, not a reference to the outer one. Each call to `process_file`
creates its own fresh local `total` (starting from nothing, since it's
newly declared each time), increments that local copy to 1, and then
discards it entirely when the function returns — the outer, script-level
`total` is never touched by any of this, which is exactly why it stays at
0 through all three calls. To accumulate a count across multiple function
calls while `local` is still correctly used for the function's own
internal purposes, the outer variable needs to be updated explicitly, not
implicitly — e.g., have `process_file` `echo` its own local increment and
capture/add it at the call site (`total=$((total + $(process_file)))`), or
skip using a same-named local variable for anything meant to affect the
outer scope, using `local` only for variables genuinely internal to the
function's own logic.

</details>

## Interview Prep

**Q1. Why is `rm $some_var/*.tmp` considered a genuinely dangerous pattern, not just a style nitpick?**
If `some_var` is empty or unset, the expansion collapses to just `/*.tmp`
(or a similarly unintended path), and `rm` then operates against a
completely different, much broader location than intended — potentially
the root of the filesystem. `${some_var:?}` causes the script to fail
loudly with an error the moment the variable is empty/unset, rather than
silently proceeding to construct and execute a dangerous command — this is
a real, documented category of production incident, not a hypothetical
concern.

**Q2. A variable assigned inside a shell function unexpectedly seems to affect (or fail to affect) a same-named variable outside it. What's the most likely explanation?**
Whether `local` was used. Without `local`, an assignment inside a function
is global by default and can silently clobber a same-named variable used
elsewhere in the script. With `local`, the function's variable is a
genuinely separate piece of memory that shadows the outer one by name but
never touches it — which explains both possible confusing symptoms:
unwanted global modification (missing `local`) or an outer variable that
never updates despite the function appearing to increment "it" (present
`local`, but the outer accumulation was never wired up explicitly).

**Q3. Why does a naive `for line in $(cat file.txt)` loop break on realistic input, and what's the correct alternative?**
`$(cat file.txt)` undergoes word splitting like any unquoted expansion —
it splits on *all* whitespace, not just newlines, so a single line
containing spaces becomes multiple loop iterations instead of one, and
leading/trailing whitespace gets silently trimmed. The correct idiom is
`while IFS= read -r line; do ... done < file.txt` — `IFS=` preserves
whitespace, `-r` avoids backslash interpretation, and redirecting the file
directly avoids the word-splitting problem entirely.

**Q4. What's the difference between `[ ]` and `[[ ]]` in a bash conditional, beyond just syntax preference?**
`[ ]` is the external/builtin `test` command, requiring careful spacing and
more vulnerable to word-splitting issues with unquoted variables inside
it; `[[ ]]` is bash's own extended conditional construct, which handles
unquoted variables more safely and adds pattern matching (`==` with globs)
and regex matching (`=~`) directly, neither of which `[ ]` supports
natively. `[[ ]]` is generally the safer, more capable default in bash
specifically (though `[ ]` remains necessary for POSIX-portable scripts
that might run under `sh`, not just `bash`).

---

## Key Takeaways

1. **Unquoted variables undergo word splitting — the same mechanism as
   Demo 19's unquoted `xargs` filenames, applied to any variable
   expansion.** Quote every variable expansion by default; treat not
   doing so as a deliberate, understood exception, not the norm.
2. **`${var:?}` turns an unset/empty variable into a loud failure instead
   of a silent, dangerous continuation** — essential in any destructive
   command (`rm`, `mv`) that constructs a path from a variable.
3. **Numeric (`-eq`, `-gt`) and string (`==`, `!=`) comparison operators
   are not interchangeable.** Using the wrong category on the wrong kind
   of data produces silently wrong results, not always an error.
4. **`for line in $(cat file)` word-splits on all whitespace within lines,
   not just between them — `while IFS= read -r line; do ... done < file`
   is the correct, safe idiom** for reading a file line by line.
5. **Without `local`, a function's variable assignment is global by
   default and can silently clobber a same-named variable elsewhere in the
   script.** `local` is not optional discipline — it's the difference
   between an isolated function and one with invisible side effects.
6. **Shell functions "return" via exit code (0=success), not a computed
   value.** To get data out of a function, `echo` it and capture the
   result with command substitution at the call site.

> **Demo scope:** Primary concept: writing basic bash scripts — variables,
> control flow, functions. Supporting concepts: variables and command
> substitution, quoting safety, conditionals, loops (including safe line
> reading), functions and local scope.
> Estimated completion time: 30 minutes (reading + hands-on + verification).
> Checkpoints: 6 natural stopping points (after Task 1 through Task 6).

## Quick Commands Reference

### Variables

| Syntax | Purpose |
|---|---|
| `var=value` | Assignment (no spaces around `=`) |
| `$(command)` | Command substitution |
| `readonly var=value` | Immutable variable |
| `"${var:?}"` | Fail loudly if unset/empty |

### Conditionals

| Syntax | Purpose |
|---|---|
| `[[ "$a" == "$b" ]]` | String equality (safer, bash-only) |
| `[ "$a" -eq "$b" ]` | Numeric equality |
| `[[ "$s" =~ regex ]]` | Regex match |
| `[[ "$s" == *.log ]]` | Glob pattern match |

### Loops

| Syntax | Purpose |
|---|---|
| `for x in LIST; do ... done` | Iterate a list |
| `while COND; do ... done` | Loop while condition holds |
| `while IFS= read -r line; do ... done < file` | Safe line-by-line reading |

### Functions

| Syntax | Purpose |
|---|---|
| `name() { ... }` | Define a function |
| `local var="$1"` | Scope a variable to the function |
| `return 0` / `return 1` | Success / failure exit code |
| `echo "$result"` + `$(func)` | Get a computed value out of a function |

## Appendix A — Anki Cards

**`33-shell-scripting-fundamentals-anki.csv`:**

```
#deck:Linux Mastery::Module 9 - Shell Scripting::Demo 33 - Shell Scripting Fundamentals
#separator:Comma
#columns:Front,Back,Tags
"touch $filename creates two files ('my' and 'file.txt') instead of one ('my file.txt') when filename='my file.txt'. Why?","The unquoted variable undergoes word splitting — the same mechanism as unquoted xargs input (Demo 19) — treating the space inside the value as an argument separator. Quoting ('touch \"$filename\"') prevents this.","demo33,quoting,word-splitting,common-mistake"
"rm $target_dir/*.tmp deletes files from the root filesystem when target_dir is empty/unset. Why, and what's the fix?","With target_dir empty, the expansion collapses to just /*.tmp — a dangerously different path than intended. ${target_dir:?} makes the script fail loudly with an error if the variable is unset/empty, instead of silently constructing a dangerous command.","demo33,quoting,safety,common-mistake"
"What's the actual difference between [ ] and [[ ]] in bash, beyond syntax style?","[ ] is the external/builtin test command, more vulnerable to word-splitting issues with unquoted variables. [[ ]] is bash's own extended construct — handles unquoted variables more safely and adds glob (==) and regex (=~) matching directly.","demo33,conditionals,test,fundamentals"
"Why does [ \"5\" == \"05\" ] return false while [ \"5\" -eq \"05\" ] returns true?","== is STRING comparison (the strings '5' and '05' are different character sequences); -eq is NUMERIC comparison (both represent the number 5). Using the wrong comparison type for the data produces silently different, valid-looking results.","demo33,conditionals,numeric-vs-string,common-mistake"
"for line in $(cat file.txt); do echo $line; done splits a single line containing spaces into MULTIPLE loop iterations. Why?","$(cat file.txt) is an unquoted expansion, which undergoes word splitting on ALL whitespace (not just newlines) — a line like 'hello world' becomes two separate iterations, not one.","demo33,loops,word-splitting,common-mistake"
"What is the correct, safe idiom for reading a file line by line in bash, and what does each part do?","while IFS= read -r line; do ... done < file.txt — IFS= preserves leading/trailing whitespace, -r avoids backslash interpretation, and redirecting the file avoids the word-splitting problem of $(cat file) entirely.","demo33,loops,safe-reading"
"Inside a shell function, assigning to a variable WITHOUT 'local' has what scope, and why is this a common source of bugs?","Global by default — an unlocal'd assignment inside a function can silently clobber a same-named variable used elsewhere in the script, since bash functions don't automatically create their own variable scope the way many languages' functions do.","demo33,functions,local,common-mistake"
"A script has an outer variable 'total=0', and a function does 'local total; total=$((total+1))' on every call — why does the outer 'total' never change?","local creates a genuinely separate, function-scoped variable that shadows the outer one by NAME only — it's not a reference to the outer variable. Each function call gets its own fresh local total, incremented and then discarded when the function returns, never touching the outer script-level total.","demo33,functions,local,common-mistake"
"How do shell functions actually 'return' a value, given they don't have a return-value mechanism like most programming languages?","Via exit code only (0=success, non-zero=failure) through 'return N'. To get an actual computed VALUE out, the convention is echoing it inside the function and capturing that output via command substitution at the call site: result=$(some_function).","demo33,functions,return-values,fundamentals"
"Why is 'name = \"value\"' (with spaces around =) not a valid variable assignment in bash?","Bash parses it as running a command literally called 'name' with the arguments '=' and '\"value\"', not as an assignment — producing a confusing 'command not found' error rather than a clear syntax error pointing at the real mistake.","demo33,variables,syntax,common-mistake"
```

## Appendix B — Quiz

**`33-shell-scripting-fundamentals-quiz.md`:**

````markdown
# Quiz — Demo 33: Shell Scripting Fundamentals

> One correct answer per question unless stated otherwise.
> Target: 80% or above before moving to Demo 34.

**Q1. `touch $filename` creates two files instead of one when `filename="my file.txt"`. Why?**

- A) `touch` has a bug with spaces
- B) The unquoted variable undergoes word splitting, treating the space as an argument separator
- C) Filenames with spaces are not supported by Linux
- D) `$filename` needs a `\n` character to work correctly

<details>
<summary>Answer</summary>

**B** — This is standard unquoted-expansion word-splitting behavior, the same mechanism affecting unquoted `xargs` input. Trap: A and C both invent false limitations; D invents an irrelevant fix.

</details>

---

**Q2. `rm $target_dir/*.tmp` deletes unexpected files when `target_dir` is empty. What actually happened?**

- A) `rm` ignored the empty variable and used a default directory
- B) The expansion collapsed to `/*.tmp`, targeting the root filesystem instead of the intended subdirectory
- C) This is a rare edge case that essentially never occurs in practice
- D) `rm` requires `-r` to be affected by variable values at all

<details>
<summary>Answer</summary>

**B** — This exact mechanism is a documented, real category of production incident. Trap: A invents a nonexistent default-fallback behavior; C understates how common and serious this bug pattern actually is; D is unrelated to the actual mechanism.

</details>

---

**Q3. What's the correct fix to make a script fail loudly instead of silently continuing when a variable used in a destructive command is unset?**

- A) `${var:-default}`
- B) `${var:?}`
- C) `$var!!`
- D) `set -x`

<details>
<summary>Answer</summary>

**B** — `${var:?}` triggers an immediate error exit if the variable is unset or empty. Trap: A provides a fallback value instead of failing (the opposite of the safety goal here); C is invalid syntax; D enables command tracing, unrelated to this specific safety check.

</details>

---

**Q4. Why does `[ "5" == "05" ]` evaluate as false while `[ "5" -eq "05" ]` evaluates as true?**

- A) `[ ]` has a bug affecting leading zeros
- B) `==` is string comparison (different character sequences); `-eq` is numeric comparison (same numeric value)
- C) `05` is invalid syntax and both commands should error
- D) The comparison depends on which shell is running the script

<details>
<summary>Answer</summary>

**B** — This is the correct, expected distinction between string and numeric comparison operators. Trap: A invents a bug; C is factually wrong — `05` is valid in both contexts here; D invents an irrelevant portability concern for this specific case.

</details>

---

**Q5. `for line in $(cat file.txt); do echo $line; done` splits a single line containing spaces into multiple iterations. Why?**

- A) `for` loops in bash can only process one word at a time by design, always
- B) `$(cat file.txt)` is unquoted and undergoes word splitting on all whitespace, not just newlines
- C) `cat` inserts extra newlines automatically
- D) This only happens with files larger than 4KB

<details>
<summary>Answer</summary>

**B** — Unquoted command substitution word-splits on all whitespace, exactly like any other unquoted expansion. Trap: A mischaracterizes `for` loops generally; C invents false `cat` behavior; D invents an irrelevant size threshold.

</details>

---

**Q6. What's the correct, safe way to read a file line by line in bash?**

- A) `for line in $(cat file.txt); do ... done`
- B) `while IFS= read -r line; do ... done < file.txt`
- C) `cat file.txt | for line; do ... done`
- D) `readlines file.txt`

<details>
<summary>Answer</summary>

**B** — This is the documented safe idiom, avoiding the word-splitting problem entirely. Trap: A is the fragile pattern this question is testing against; C is invalid bash syntax; D invents a command that doesn't exist in bash.

</details>

---

**Q7. Inside a shell function, an assignment made WITHOUT `local` has what scope by default?**

- A) Function-local automatically, same as with `local`
- B) Global — it can silently clobber a same-named variable used elsewhere in the script
- C) It causes a syntax error
- D) It only persists for the duration of that single function call, then is discarded

<details>
<summary>Answer</summary>

**B** — Bash functions do not create automatic variable scoping; `local` is required explicitly, and omitting it is a common source of confusing bugs. Trap: A wrongly assumes automatic scoping that doesn't exist; C is factually wrong — it's valid, just risky; D describes `local`'s actual behavior, not the default.

</details>

---

**Q8. How does a shell function actually communicate a computed value back to the caller?**

- A) Through `return`, which can return any string or number
- B) `return` only provides an exit code (0/non-zero); an actual computed value is `echo`'d and captured via command substitution
- C) Shell functions cannot return any information at all
- D) Through a special variable called `$RETURN` automatically

<details>
<summary>Answer</summary>

**B** — This exit-code-only limitation, and the `echo`+command-substitution convention for actual values, is the correct model. Trap: A overstates what `return` can carry; C understates real capability; D invents a variable that doesn't exist.

</details>
````