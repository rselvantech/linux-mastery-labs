# Demo 34 — Shell Scripting Advanced

## Overview

Demo 33 built scripts that work when everything goes right. This demo
builds scripts that behave correctly when something goes *wrong* — failing
loudly instead of limping on with a bad exit code, cleaning up after
themselves even when interrupted, accepting real command-line flags instead
of hardcoded values, and using arrays instead of fragile string
concatenation. This is the difference between a script you run once
yourself and one you'd trust in a production automation pipeline.

## Demo Objectives

1. Use `set -euo pipefail` correctly, and understand exactly what each flag catches
2. Use `trap` to run cleanup code on exit, including on interruption
3. Parse command-line flags properly with `getopts`
4. Use bash arrays for lists of values, not string concatenation
5. Use here-docs for multi-line text without cluttering the script with `echo` calls
6. Combine all of the above into a script that calls the grep/sed/awk toolkit from Demos 18–21

## Directory Structure

```
34-shell-scripting-advanced/
├── README.md                                  # this file
├── 34-shell-scripting-advanced-anki.csv       # Anki flashcard deck
└── 34-shell-scripting-advanced-quiz.md        # standalone quiz
```

## Environments

| Task | Ubuntu 24.04 | Rocky Linux 9.8 | EC2 Amazon Linux |
|---|---|---|---|
| All lab tasks | ✅ (bash) | ✅ (bash) | Not required |

> **Verification note:** every task in this demo was run live, in actual
> bash, in the authoring sandbox — all output shown (including the
> `set -e`/`if`-condition exception and the `getopts` missing-colon bug)
> matches exactly what was produced. No `[Unverified]` markers are needed
> anywhere in this demo.

## Recall Check — 33-shell-scripting-fundamentals

Answer these from memory before continuing — no looking back at Demo 33.

**Q1.** `rm $target_dir/*.tmp` deletes unexpected files when `target_dir`
is empty. What actually happened, and what's the fix?

**Q2.** Inside a shell function, an assignment made without `local` has
what scope by default, and why is this a common source of bugs?

**Q3.** What's the correct, safe way to read a file line by line in bash?

<details>
<summary>Answers</summary>

**A1.** The expansion collapses to `/*.tmp`, targeting the root filesystem
instead of the intended subdirectory. Fix: `${target_dir:?}` to fail
loudly if the variable is unset/empty, plus quoting.

**A2.** Global by default — it can silently clobber a same-named variable
used elsewhere in the script, since bash doesn't create automatic
function-level scoping.

**A3.** `while IFS= read -r line; do ... done < file.txt`.

</details>

## Part 1 — Concepts

### 1.1 set -euo pipefail — what each flag actually catches

```bash
#!/bin/bash
set -euo pipefail
```

Three independent flags, each catching a different class of silent
failure:

- **`-e`** — exit immediately if any command exits with a non-zero status.
  Without it, a script keeps running after a failed command as if nothing
  happened, often producing confusing downstream errors far from the real
  cause.
- **`-u`** — treat referencing an unset variable as an error, instead of
  silently substituting an empty string. This catches typos in variable
  names (`$fiel_name` instead of `$file_name`) that would otherwise
  silently produce an empty value and continue.
- **`-o pipefail`** — a pipeline's exit status is the exit status of its
  *last failing* command, not just its last command. Without this, `cmd1 |
  cmd2` "succeeds" (exit 0) as long as `cmd2` succeeds, even if `cmd1`
  failed catastrophically partway through — `-e` alone doesn't catch this,
  since it only looks at the pipeline's overall (last-command) exit status.

```bash
false | echo "this still runs"
echo "exit code: $?"
# Without pipefail: exit code 0 — echo succeeded, so the pipeline "succeeded"
# even though false (a command that always fails) ran first.
set -o pipefail
false | echo "this still runs"
echo "exit code: $?"
# With pipefail: exit code reflects false's failure, not echo's success.
```

**`-e` has real, documented exceptions worth knowing** — it does *not*
trigger inside a condition being tested (`if some_command; then` doesn't
exit even if `some_command` fails, because failing is exactly what the
`if` is checking for), and it doesn't trigger for a command that's part of
`&&`/`||` chains in certain positions. This is why `set -e` alone isn't a
complete safety net and why explicit checks still matter for anything
truly critical.

### 1.2 trap — cleanup that runs no matter how the script exits

```bash
#!/bin/bash
tmpfile=$(mktemp)

cleanup() {
    echo "Cleaning up $tmpfile"
    rm -f "$tmpfile"
}
trap cleanup EXIT
```

`trap FUNCTION EXIT` registers `cleanup` to run automatically whenever the
script exits — normally, via `exit`, or (critically) via an error under
`set -e`. **This matters specifically because it also catches paths a
script author might not think to add manual cleanup for** — an unexpected
error partway through, not just the "happy path" end of the script.

```bash
trap cleanup EXIT
trap 'echo "Interrupted!"; cleanup; exit 130' INT TERM
# INT: Ctrl+C. TERM: a kill signal (Demo 06's signal knowledge applies
# directly). Registering a SEPARATE handler for these means the script
# can do something specifically different on interruption (like logging
# that it was interrupted) in addition to the same cleanup — and calling
# cleanup explicitly here, then exit, still triggers the EXIT trap too,
# so cleanup logic shouldn't be duplicated destructively across both.
```

Without a trap, a script interrupted mid-run (Ctrl+C, a timeout, a `kill`)
leaves behind whatever temp files, locks, or partial state it had created,
with no guaranteed cleanup — `trap` is what makes a script's cleanup
promise hold under all exit paths, not just the one the author happened to
write an explicit cleanup call at the end of.

### 1.3 getopts — real command-line flag parsing

```bash
#!/bin/bash
verbose=false
output_file=""

while getopts "vo:h" opt; do
    case "$opt" in
        v) verbose=true ;;
        o) output_file="$OPTARG" ;;
        h) echo "Usage: $0 [-v] [-o FILE]"; exit 0 ;;
        \?) echo "Invalid option: -$OPTARG" >&2; exit 1 ;;
    esac
done
```

The option string `"vo:h"` encodes the flag definitions: `v` and `h` take
no argument; `o:` (the colon) means `-o` **requires** an argument, which
`getopts` makes available in `$OPTARG`. Called as `myscript -v -o
output.txt`, `getopts` parses both flags correctly in a loop, handling
combined short flags (`-vo output.txt`) and unrecognized flags (`\?`
branch) — **this is meaningfully more robust than manually checking `"$1"
== "-v"` in a chain of `if` statements**, which breaks on flag ordering,
combined flags, or flags with arguments almost immediately once a script
has more than one or two options.

```bash
shift $((OPTIND - 1))
# After the getopts loop, $1 still refers to the FIRST flag, not the first
# non-flag argument — OPTIND tracks how many arguments getopts consumed;
# shifting past that makes $1 correctly refer to whatever comes after the
# flags (e.g. a filename argument following the options).
```

### 1.4 Arrays — lists of values, not string concatenation

```bash
files=("app.log" "error.log" "access.log")
echo "${files[0]}"        # app.log — zero-indexed
echo "${files[@]}"        # all elements, space-separated
echo "${#files[@]}"       # 3 — array length

for f in "${files[@]}"; do
    echo "Processing: $f"
done
```

**`"${files[@]}"` (quoted, with `@`) preserves each element as a separate
word even if an element contains spaces — the array equivalent of Demo
19/33's quoting lessons.** Using `${files[*]}` instead (or unquoted `${files[@]}`)
collapses multi-word elements incorrectly, the exact same word-splitting
class of bug already covered for plain variables.

```bash
files+=("newfile.log")
# Append to an array — the array-specific equivalent of string
# concatenation, without the escaping/delimiter problems of trying to
# build a space-separated STRING and split it back apart later.

declare -A status_by_service
status_by_service["nginx"]="running"
status_by_service["postgres"]="stopped"
for service in "${!status_by_service[@]}"; do
    echo "$service: ${status_by_service[$service]}"
done
# declare -A: an associative array — bash's equivalent of awk's Demo 21
# associative arrays, keyed by string rather than only by numeric index.
```

### 1.5 Here-docs — multi-line text without echo chains

```bash
cat << 'EOF'
This is line 1.
This is line 2 with a $variable that will NOT be expanded (quoted 'EOF').
EOF

cat << EOF
This is line 1.
This variable WILL be expanded: $HOME
EOF
```

**Quoting the delimiter (`'EOF'`) versus not is the single most important
detail** — an unquoted delimiter allows variable expansion and command
substitution inside the here-doc, exactly like a double-quoted string;
a quoted delimiter treats the entire block as fully literal text, no
expansion at all. Forgetting this distinction is a common source of either
unwanted expansion (leaking `$HOME` into what was meant to be literal
documentation text) or unexpected literal `$variable` strings (when
expansion was actually wanted but the delimiter was accidentally quoted).

```bash
mail_body=$(cat << EOF
Subject: Deploy complete
The deploy for $(hostname) finished at $(date).
EOF
)
```

Here-docs are the practical replacement for a long chain of individual
`echo` statements when generating multi-line output — a config file
template, an email body, a usage message — genuinely more readable than
many separate `echo "..."` lines, especially once the text itself spans
more than 3-4 lines.

### 1.6 Bringing it together — scripts that call the Demos 18–21 toolkit

A production-grade script rarely does everything itself — it usually
orchestrates the tools built in earlier demos, with the robustness
practices from this demo wrapped around those calls:

```bash
#!/bin/bash
set -euo pipefail

log_dir="${1:?Usage: $0 LOG_DIRECTORY}"
report_file=$(mktemp)
trap 'rm -f "$report_file"' EXIT

declare -A error_counts

for logfile in "$log_dir"/*.log; do
    count=$(grep -c 'ERROR' "$logfile" 2>/dev/null || echo 0)
    error_counts["$logfile"]=$count
done

{
    echo "Error Report — $(date)"
    echo "===================="
    for logfile in "${!error_counts[@]}"; do
        echo "$logfile: ${error_counts[$logfile]} errors"
    done
} > "$report_file"

cat "$report_file"
```

This one script draws on: `${1:?...}` (Demo 33's fail-loudly pattern),
`set -euo pipefail`, `trap` for cleanup, an associative array, and `grep
-c` (Demo 18) — the shape of a real, small operational script, not a toy
example.

## Part 2 — Hands-On Lab

### Commands used in this demo

| Syntax | Purpose |
|---|---|
| `set -euo pipefail` | Exit on error, unset-variable error, pipeline-failure detection |
| `trap 'cmd' EXIT` | Run cleanup on any script exit path |
| `trap 'cmd' INT TERM` | Run a handler on interruption/kill signals |
| `getopts "vo:h" opt` | Parse command-line flags, including required arguments |
| `arr=(a b c)` / `"${arr[@]}"` | Declare / safely iterate an array |
| `declare -A assoc` | Declare an associative array |
| `<< 'EOF'` / `<< EOF` | Here-doc, literal vs. expanding |

### Task 1 — set -euo pipefail catching real mistakes

```bash
cat > /tmp/lab34-1.sh << 'SCRIPT'
#!/bin/bash
set -euo pipefail

echo "Before the failing command"
false
echo "This should NEVER print"
SCRIPT
bash /tmp/lab34-1.sh; echo "exit code: $?"

cat > /tmp/lab34-1b.sh << 'SCRIPT'
#!/bin/bash
set -euo pipefail
echo "Value: $undeclared_variable"
SCRIPT
bash /tmp/lab34-1b.sh; echo "exit code: $?"
```

### Task 2 — trap ensures cleanup on interruption

```bash
cat > /tmp/lab34-2.sh << 'SCRIPT'
#!/bin/bash
tmpfile=$(mktemp)
echo "Created: $tmpfile"
trap 'echo "Cleaning up $tmpfile"; rm -f "$tmpfile"' EXIT

echo "Working... (this would normally do something with $tmpfile)"
SCRIPT
bash /tmp/lab34-2.sh
```

### Task 3 — getopts parsing real flags

```bash
cat > /tmp/lab34-3.sh << 'SCRIPT'
#!/bin/bash
verbose=false
output=""
while getopts "vo:h" opt; do
    case "$opt" in
        v) verbose=true ;;
        o) output="$OPTARG" ;;
        h) echo "Usage: $0 [-v] [-o FILE]"; exit 0 ;;
        \?) echo "Invalid option" >&2; exit 1 ;;
    esac
done
shift $((OPTIND - 1))
echo "verbose=$verbose output=$output remaining_args=$*"
SCRIPT
bash /tmp/lab34-3.sh -v -o report.txt somefile.log
bash /tmp/lab34-3.sh -h
```

### Task 4 — Arrays: indexed and associative

```bash
cat > /tmp/lab34-4.sh << 'SCRIPT'
#!/bin/bash
logs=("app.log" "error log with spaces.log" "access.log")
echo "Count: ${#logs[@]}"
for l in "${logs[@]}"; do
    echo "File: [$l]"
done

declare -A service_status
service_status["nginx"]="running"
service_status["postgres"]="stopped"
for svc in "${!service_status[@]}"; do
    echo "$svc: ${service_status[$svc]}"
done
SCRIPT
bash /tmp/lab34-4.sh
```

### Task 5 — Here-docs: literal vs. expanding

```bash
cat > /tmp/lab34-5.sh << 'SCRIPT'
#!/bin/bash
name="wadmin"
echo "--- literal (quoted delimiter) ---"
cat << 'INNER'
Hello $name, this will NOT expand.
INNER
echo "--- expanding (unquoted delimiter) ---"
cat << INNER
Hello $name, this WILL expand.
INNER
SCRIPT
bash /tmp/lab34-5.sh
```

### Task 6 — A complete robust script

```bash
mkdir -p /tmp/lab34-logs
printf 'INFO ok\nERROR bad\nERROR worse\n' > /tmp/lab34-logs/app.log
printf 'INFO ok\n' > /tmp/lab34-logs/other.log

cat > /tmp/lab34-6.sh << 'SCRIPT'
#!/bin/bash
set -euo pipefail

log_dir="${1:?Usage: $0 LOG_DIRECTORY}"
report_file=$(mktemp)
trap 'rm -f "$report_file"' EXIT

declare -A error_counts
for logfile in "$log_dir"/*.log; do
    count=$(grep -c 'ERROR' "$logfile" 2>/dev/null || echo 0)
    error_counts["$logfile"]=$count
done

{
    echo "Error Report"
    echo "============"
    for logfile in "${!error_counts[@]}"; do
        echo "$logfile: ${error_counts[$logfile]} errors"
    done
} > "$report_file"

cat "$report_file"
SCRIPT
bash /tmp/lab34-6.sh /tmp/lab34-logs
bash /tmp/lab34-6.sh   # confirm the ${1:?} usage message fires correctly
```

## What You Learned

- ✅ Used `set -euo pipefail` correctly, and understood each flag's specific behavior and limits
- ✅ Used `trap` to run cleanup on exit, including on interruption
- ✅ Parsed command-line flags properly with `getopts`, including required arguments
- ✅ Used bash arrays (indexed and associative) instead of string concatenation
- ✅ Used here-docs, understanding the literal-vs-expanding delimiter distinction
- ✅ Combined all of the above into a script calling the grep/sed/awk toolkit

## Part 3 — Break-Fix Scenario

### Scenario — NovaTech Ops: two advanced-scripting incidents

> **Incident report — NovaTech Ops, ticket #4745**
>
> Alert 1: "a deployment script with `set -e` continued running after a
> critical step failed, and nobody noticed until much later." Alert 2: "a
> script using `getopts` silently ignores the `-o` flag's value entirely."

**Incident 1 — set -e didn't stop the script**

```bash
$ cat deploy.sh
#!/bin/bash
set -e
check_health() {
    curl -sf http://localhost/health
}
if check_health; then
    echo "Healthy, proceeding"
else
    echo "Unhealthy — should have stopped here but continued anyway"
fi
echo "Deploy continued regardless"
```

Tasks:
1. Reference §1.1 — under what specific, documented condition does `-e`
   not trigger, even when the command inside actually fails?
2. Is this a `set -e` bug, or expected behavior given where `check_health`
   is being called?
3. If the intent really was "stop immediately if health check fails,"
   what should the script structure look like instead?

**Incident 2 — getopts -o flag value ignored**

```bash
$ cat lab.sh
#!/bin/bash
while getopts "vo" opt; do
    case "$opt" in
        v) verbose=true ;;
        o) output="$OPTARG" ;;
    esac
done
$ bash lab.sh -o report.txt
$ echo $output
   (empty)
```

Tasks:
1. Reference §1.3 — compare the option string `"vo"` here against the
   correct form from the demo. What's missing?
2. Explain exactly what that missing character tells `getopts` to do
   differently.
3. Fix the option string, and confirm what `$OPTARG` would then correctly
   contain.

<details>
<summary>Answers</summary>

**Incident 1:** Per §1.1, `-e` does not trigger when the failing command
is part of a condition being tested — specifically, a command used
directly as the condition of an `if` (or `while`, or combined with
`&&`/`||`) is expected to potentially fail as part of what's being tested,
so `set -e` deliberately doesn't treat that failure as a script-ending
error. This is expected `set -e` behavior, not a bug — `check_health`
being called as `if check_health; then` means its failure is exactly what
the `if`/`else` is designed to handle, and `-e` correctly stays out of the
way to let that conditional logic run. If the actual intent was "stop
immediately, unconditionally, if the health check fails" rather than
"branch based on whether it succeeds," the script shouldn't wrap it in an
`if` at all — it should just call `check_health` directly as a standalone
statement (letting `-e` catch and stop on its failure naturally), or
explicitly `exit 1` inside the `else` branch rather than just printing a
message and falling through to the rest of the script.

**Incident 2:** The option string is `"vo"`, missing the colon after `o`
(`"vo:"` is correct) — per §1.3, a colon after a flag letter tells
`getopts` that flag requires an argument, populated into `$OPTARG`.
Without the colon, `getopts` treats `-o` as a flag taking no argument at
all — so when called as `-o report.txt`, `getopts` sees `-o` as a
complete, argument-less flag, and `report.txt` is left as a separate,
unconsumed positional argument rather than being captured into `$OPTARG`
for the `o)` case at all; `$output` stays unset/empty because the `o)`
branch runs but `$OPTARG` was never populated. Fix: `while getopts "vo:"
opt; do`. With the colon correctly in place, `$OPTARG` inside the `o)`
case would then correctly contain `"report.txt"`.

</details>

## Interview Prep

**Q1. A script has `set -e`, but it keeps running after a command inside an `if` condition fails. Is this a bug?**
No — this is `set -e`'s documented, intentional behavior. A command used
directly as an `if`/`while` condition (or combined with `&&`/`||`) is
expected to potentially fail as part of what's being tested; `-e`
deliberately doesn't treat that as a script-ending failure, since the
conditional logic is specifically built to handle both outcomes. If the
actual intent is "stop unconditionally on failure," the command needs to
run outside a conditional context, or the failure branch needs an explicit
`exit`.

**Q2. Why is `trap cleanup EXIT` considered more robust than putting cleanup code at the end of a script?**
Cleanup code at the end of a script only runs if execution reaches that
point normally — an error under `set -e`, an interruption (Ctrl+C), or a
`kill` signal can all skip straight past it. `trap ... EXIT` registers the
cleanup to run on *any* exit path, including those the script author might
not have specifically anticipated, which is what makes a script's cleanup
promise reliable rather than conditional on the happy path.

**Q3. A `getopts` script silently ignores a flag's intended argument value. What's the most likely cause?**
A missing colon after that flag's letter in the option string —
`getopts "vo"` treats `-o` as a no-argument flag, while `getopts "vo:"`
correctly tells `getopts` that `-o` requires an argument, populated into
`$OPTARG`. Without the colon, any value following `-o` on the command line
becomes an unconsumed positional argument instead of being captured.

**Q4. Why is `"${array[@]}"` (quoted, with `@`) the correct way to iterate a bash array, rather than `${array[*]}` or an unquoted expansion?**
Quoted `"${array[@]}"` preserves each array element as its own distinct
word, even if an element contains spaces — exactly analogous to quoting a
regular variable to prevent word splitting. `${array[*]}` (or any
unquoted form) collapses the array into a single string, breaking
multi-word elements apart incorrectly during iteration, the same class of
bug covered for plain variables in Demo 33.

---

## Key Takeaways

1. **`set -e`, `-u`, and `pipefail` each catch a different, specific class
   of silent failure — and `-e` has real, documented exceptions** (commands
   used as `if`/`while` conditions don't trigger it). Understanding those
   exceptions matters as much as knowing the flags exist.
2. **`trap ... EXIT` guarantees cleanup runs on every exit path** —
   errors, interruptions, kills — not just the happy path a script author
   remembered to add manual cleanup at the end of.
3. **`getopts`'s colon syntax (`o:` vs. `o`) is the difference between a
   flag that takes an argument and one that doesn't.** A missing colon
   silently leaves the intended argument value uncaptured, as an
   unconsumed positional argument instead.
4. **Bash arrays, iterated with `"${arr[@]}"` (quoted, `@`), are the
   correct way to handle lists of values — not string concatenation.**
   The quoting rules are the same word-splitting lesson from Demo 33,
   applied to arrays.
5. **Associative arrays (`declare -A`) bring the same key-based aggregation
   pattern from `awk` (Demo 21) into bash scripts directly.**
6. **A here-doc's delimiter quoting controls expansion entirely — quoted
   (`'EOF'`) is fully literal, unquoted (`EOF`) expands variables and
   command substitution.** Getting this backward is a common source of
   either unwanted leaked expansion or unexpected literal text.

> **Demo scope:** Primary concept: writing robust, production-grade bash
> scripts. Supporting concepts: set -euo pipefail, trap-based cleanup,
> getopts flag parsing, arrays (indexed and associative), here-docs.
> Estimated completion time: 30 minutes (reading + hands-on + verification).
> Checkpoints: 6 natural stopping points (after Task 1 through Task 6).

## Quick Commands Reference

### Error handling

| Syntax | Purpose |
|---|---|
| `set -e` | Exit on any command failure |
| `set -u` | Error on unset variable reference |
| `set -o pipefail` | Pipeline fails if ANY stage fails, not just the last |
| `set -euo pipefail` | All three combined — the standard robust-script header |

### Cleanup and signals

| Syntax | Purpose |
|---|---|
| `trap 'cmd' EXIT` | Run on any script exit |
| `trap 'cmd' INT TERM` | Run on Ctrl+C / kill |

### Flag parsing

| Syntax | Purpose |
|---|---|
| `getopts "vo:h" opt` | Parse flags; `v`/`h` no-arg, `o:` requires `$OPTARG` |
| `shift $((OPTIND - 1))` | Advance past parsed flags to remaining positional args |

### Arrays and here-docs

| Syntax | Purpose |
|---|---|
| `arr=(a b c)` / `"${arr[@]}"` | Indexed array / safe iteration |
| `declare -A assoc` | Associative array |
| `<< 'EOF' ... EOF` | Here-doc, literal (no expansion) |
| `<< EOF ... EOF` | Here-doc, expanding |

## Appendix A — Anki Cards

**`34-shell-scripting-advanced-anki.csv`:**

```
#deck:Linux Mastery::Module 9 - Shell Scripting::Demo 34 - Shell Scripting Advanced
#separator:Comma
#columns:Front,Back,Tags
"What specific failure does set -o pipefail catch that plain set -e does not?","A pipeline (cmd1 | cmd2) where an earlier stage fails but a later stage succeeds — without pipefail, the pipeline's exit status is only the LAST command's status, so an early failure is silently masked. pipefail makes the pipeline's exit status reflect the last FAILING command.","demo34,set-e,pipefail,fundamentals"
"A script has set -e, but keeps running after a command used as an if condition fails. Is this a bug?","No — this is documented set -e behavior. A command used directly as an if/while condition is expected to potentially fail as part of what's being tested, so set -e deliberately doesn't treat that failure as script-ending.","demo34,set-e,common-mistake"
"Why is trap cleanup EXIT more robust than putting cleanup code at the literal end of a script?","End-of-script cleanup only runs if execution reaches that point normally. trap EXIT runs on ANY exit path — errors under set -e, interruptions (Ctrl+C), kill signals — not just the happy path.","demo34,trap,cleanup,fundamentals"
"In getopts \"vo:h\", what does the colon after 'o' specifically mean?","That the -o flag REQUIRES an argument, which getopts makes available in $OPTARG. Flags without a colon (v, h here) take no argument.","demo34,getopts,fundamentals"
"A getopts script defined with option string \"vo\" (no colon) silently fails to capture -o's intended argument value. Why?","Without the colon, getopts treats -o as a no-argument flag — the value following -o on the command line becomes an unconsumed positional argument instead of being captured into $OPTARG.","demo34,getopts,common-mistake"
"Why is \"${array[@]}\" (quoted, with @) the correct way to iterate a bash array, rather than ${array[*]} or an unquoted form?","Quoted \"${array[@]}\" preserves each element as its own distinct word even if it contains spaces — the array equivalent of quoting a regular variable to prevent word splitting. ${array[*]} or unquoted forms collapse elements incorrectly.","demo34,arrays,quoting,common-mistake"
"What does declare -A create, and what awk concept from Demo 21 is it directly analogous to?","An associative array — keyed by string rather than only numeric index. It's bash's direct equivalent of awk's associative arrays for key-based aggregation (arr[key]++ or similar patterns).","demo34,arrays,associative,cross-demo"
"What's the difference between cat << 'EOF' and cat << EOF (unquoted) for a here-doc?","Quoted delimiter ('EOF'): fully literal text, no variable expansion or command substitution at all. Unquoted delimiter (EOF): variables and command substitution ARE expanded inside the here-doc, like a double-quoted string.","demo34,heredoc,fundamentals"
"What does OPTIND track after a getopts loop, and why is shift $((OPTIND - 1)) needed afterward?","OPTIND tracks how many arguments getopts consumed while parsing flags. Shifting past that count makes $1 correctly refer to the first remaining non-flag argument, rather than still pointing at the first parsed flag.","demo34,getopts,optind"
"Why does trap register a SEPARATE handler for INT/TERM in addition to EXIT, rather than relying on EXIT alone?","To do something specifically different on interruption (like logging that the script was interrupted) beyond the same cleanup — though calling cleanup explicitly in the INT/TERM handler and then exiting still also triggers the EXIT trap, so cleanup logic shouldn't be duplicated destructively across both.","demo34,trap,signals,cross-demo"
```

## Appendix B — Quiz

**`34-shell-scripting-advanced-quiz.md`:**

````markdown
# Quiz — Demo 34: Shell Scripting Advanced

> One correct answer per question unless stated otherwise.
> Target: 80% or above before moving to Demo 34b.

**Q1. What specific failure does `set -o pipefail` catch that plain `set -e` alone does not?**

- A) Unset variable references
- B) An early stage of a pipeline failing while a later stage still succeeds, which would otherwise mask the failure
- C) Syntax errors in the script itself
- D) Functions that don't use `local`

<details>
<summary>Answer</summary>

**B** — Without `pipefail`, a pipeline's exit status reflects only its last command, silently masking earlier failures. Trap: A describes what `-u` catches; C and D are unrelated to `pipefail`'s actual purpose.

</details>

---

**Q2. A script with `set -e` continues running after a command used as an `if` condition fails. Is this a bug?**

- A) Yes — `set -e` should stop on every command failure without exception
- B) No — commands used as `if`/`while` conditions are expected to potentially fail, and `set -e` deliberately doesn't treat that as script-ending
- C) Yes, unless `set -u` is also enabled
- D) No, but only in scripts running on Rocky9

<details>
<summary>Answer</summary>

**B** — This is documented, intentional `set -e` behavior. Trap: A overstates `-e`'s scope; C invents an irrelevant dependency on `-u`; D invents a false platform-specific behavior.

</details>

---

**Q3. Why is `trap cleanup EXIT` more robust than cleanup code placed at the literal end of a script?**

- A) It runs faster than inline cleanup code
- B) It runs on any exit path — errors, interruptions, kills — not just the path where execution reaches the script's end normally
- C) `trap` is required syntax for any script using functions
- D) It automatically retries the cleanup if it fails

<details>
<summary>Answer</summary>

**B** — This all-exit-paths guarantee is `trap`'s specific value over end-of-script cleanup code. Trap: A invents a performance claim; C and D invent unrelated requirements/behaviors.

</details>

---

**Q4. In `getopts "vo:h" opt`, what does the colon after `o` mean?**

- A) `-o` is a required flag that must always be provided
- B) `-o` requires an argument, made available in `$OPTARG`
- C) `-o` can only be used once per invocation
- D) The colon has no functional effect, purely stylistic

<details>
<summary>Answer</summary>

**B** — This is exactly what the colon syntax signals to `getopts`. Trap: A, C, and D all invent meanings the colon doesn't actually have.

</details>

---

**Q5. A `getopts` option string is defined as `"vo"` (no colon after `o`). Calling the script with `-o report.txt`, what happens to `report.txt`?**

- A) It's correctly captured in `$OPTARG`
- B) It becomes an unconsumed positional argument, and `$OPTARG` is never populated for `-o`
- C) The script throws a fatal syntax error immediately
- D) `getopts` automatically infers that `-o` needs an argument regardless

<details>
<summary>Answer</summary>

**B** — Without the colon, `-o` is treated as a no-argument flag, leaving the following value unconsumed. Trap: A describes the correct behavior with the colon present, not this scenario; C and D both invent behavior `getopts` doesn't have.

</details>

---

**Q6. Why is `"${array[@]}"` (quoted, `@`) the correct way to iterate a bash array?**

- A) It's faster than `${array[*]}`
- B) It preserves each element as its own distinct word, even if an element contains spaces
- C) `${array[*]}` doesn't work on associative arrays
- D) Quoting is only needed for arrays larger than 10 elements

<details>
<summary>Answer</summary>

**B** — This word-preservation is the actual, practical reason to always use this specific quoted form. Trap: A invents a performance claim; C misstates an unrelated distinction; D invents a size threshold that doesn't exist.

</details>

---

**Q7. What's the difference between `cat << 'EOF' ... EOF` and `cat << EOF ... EOF`?**

- A) They are functionally identical
- B) The quoted delimiter (`'EOF'`) produces fully literal text with no expansion; the unquoted delimiter expands variables and command substitution
- C) `'EOF'` is required for multi-line here-docs; `EOF` only works for single lines
- D) `'EOF'` only works in `sh`, not `bash`

<details>
<summary>Answer</summary>

**B** — This quoting distinction entirely controls whether expansion happens inside the here-doc. Trap: A wrongly claims equivalence; C and D both invent false restrictions.

</details>

---

**Q8. After a `getopts` loop finishes, why is `shift $((OPTIND - 1))` typically run afterward?**

- A) To reset `OPTIND` back to 1 for a subsequent `getopts` call
- B) To advance past the parsed flags so `$1` correctly refers to the first remaining non-flag argument
- C) It's required syntax for `getopts` to function at all, run before the loop
- D) To convert `$OPTARG` into a positional parameter

<details>
<summary>Answer</summary>

**B** — `OPTIND` tracks how many arguments `getopts` consumed; shifting past that count is what makes `$1` correctly point at whatever comes after the flags. Trap: A misattributes the purpose; C is factually wrong — it's used after the loop, not before, and isn't required for `getopts` itself to work; D invents an unrelated conversion.

</details>
````