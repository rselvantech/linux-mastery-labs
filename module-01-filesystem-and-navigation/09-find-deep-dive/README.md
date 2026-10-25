# Demo 02a — `find` in Depth

## Overview

`ls` shows you what's in front of you. `find` answers a completely
different kind of question: "somewhere under this entire tree, show
me every file matching these criteria" — by name, age, size, type,
or any combination — and then, in the same command, optionally *act*
on every match. That combination — query plus action, in one
expression — is what makes `find` the tool you reach for cleaning up
old logs, locating a config buried three directories deep, or
auditing every file that changed in the last hour.

---

## Demo Objectives

By the end of this demo you will be able to:

1. Match files by name, type, and modification time with `-name`,
   `-type`, and `-mtime`/`-newer`
2. Exclude entire directories from a search with `-prune`, and
   explain why it must come before the filter it's protecting
3. Explain the difference between `-exec cmd {} \;` and
   `-exec cmd {} +`, and when the difference actually matters
4. Use `-delete` safely, understanding why it must always be paired
   with a specific filter
5. Explain when `find | xargs` and `find -exec +` solve the same
   problem, and why both exist

---

## Directory Structure

```
02a-find-deep-dive/
├── README.md                              # this file
├── 02a-find-deep-dive-anki.csv        # Anki flashcard deck
└── 02a-find-deep-dive-quiz.md         # standalone quiz
```

---

## Environments

| Task | Ubuntu 24.04 | Rocky Linux 9.8 | EC2 Amazon Linux |
|---|---|---|---|
| All lab tasks | ✅ | ✅ | Not required |

`find` is GNU findutils on both distros — identical behavior. No
install needed.

---

## Recall Check — Demo 02

Before starting, answer from memory:

1. `ls -la /` shows `bin -> usr/bin`. What is this called, and does
   it break scripts using `#!/bin/bash`?
2. What's the difference between a hard link and a symlink, in terms
   of what happens if the original file is deleted?
3. What does the sticky bit on `/tmp` actually restrict?

<details>
<summary>Answers</summary>

1. UsrMerge — modern distros symlink `/bin`, `/sbin`, `/lib` to their
   `/usr/` equivalents. Scripts using `/bin/bash` still work, the
   symlink resolves correctly.
2. A hard link keeps working if the original is deleted (both point
   to the same inode, either name keeps the data alive). A symlink
   breaks — it points to a path, and deleting the target leaves a
   dangling link.
3. It restricts deletion, not writing — anyone can create files in
   `/tmp`, but the sticky bit means you can only delete your own
   files, not anyone else's.

</details>

---

## Part 1 — Concepts

### 2a.1 The core model: an expression, not a flag list

`find` evaluates a chain of tests against every file under a
starting path, left to right, and by default prints anything that
passes all of them:

```bash
find /var/log -name "*.log" -mtime +30
# under /var/log, files named *.log AND older than 30 days
```

Every test is combined with implicit AND. This isn't a list of
independent options — it's closer to a small query language.

### 2a.2 `-name` and `-type`

```bash
find . -name "*.log"        # by name (glob pattern, quoted to stop
                              # the SHELL from expanding it first —
                              # find needs the literal pattern)
find . -type f               # regular files only
find . -type d               # directories only
```

### 2a.3 Time-based matching: `-mtime` and `-newer`

```bash
find . -name "*.log" -mtime +5    # modified MORE than 5 days ago
find . -name "*.log" -mtime -5    # modified LESS than 5 days ago
find . -name "*.log" -newer reference_file   # modified after reference_file was
```

> `-mtime` counts in whole days; `-newer` compares directly against
> another file's timestamp — more precise when "5 days" isn't the
> actual boundary you care about.

### 2a.4 `-prune` — excluding a whole subtree, verified

Without `-prune`, `find` descends into every directory, including
ones you don't want searched:

```bash
find . -name "*.txt"
# descends into excluded_dir too — every *.txt anywhere is found
```

`-prune` stops descent into a specific path, combined with `-o` (OR)
so the exclusion and the actual search don't fight each other:

```bash
find . -path ./excluded_dir -prune -o -name "*.txt" -print
# excluded_dir's contents never get visited at all
```

> ⚠️ [VERIFY] Confirmed directly: identical searches with and
> without `-prune` showed `excluded_dir/should_be_skipped.txt` in
> the unpruned results and completely absent — not filtered out
> after the fact, never visited — in the pruned results.

**Why `-path ... -prune -o ... -print` and not just `-prune`
alone:** `-prune` only stops descent for the matched path; without
the following `-o ... -print`, nothing would print at all for the
rest of the tree either, because `-prune` itself doesn't request
output.

### 2a.5 `-exec ... \;` vs. `-exec ... +` — not a stylistic choice

Both run a command against matches, but differently:

```bash
find . -name "*.txt" -exec echo "one:" {} \;
# one: ./normal_file.txt
# one: ./subdir/a.txt
# one: ./subdir/nested/b.txt
# — the command runs ONCE PER MATCH, {} replaced each time

find . -name "*.txt" -exec echo "batched:" {} +
# batched: ./normal_file.txt ./subdir/a.txt ./subdir/nested/b.txt
# — ALL matches passed to a SINGLE invocation (or as few as the
# command-line length limit allows), {} replaced once with the
# whole list
```

> ⚠️ [VERIFY] Confirmed directly: `\;` printed one line per file (N
> separate command invocations); `+` printed one line total, with
> every match as arguments to a single invocation.

**Why this matters beyond style:** `\;` spawns a new process per
match — correct but slow for thousands of files. `+` batches them,
the same performance idea as `xargs` batching from Demo 00b. For a
command that can accept multiple filenames at once (`rm`, `chmod`,
`grep -l`), `+` is almost always the better choice; `\;` is
necessary only when the command genuinely needs to run once per
file individually (e.g. it takes exactly one argument, or the exit
code of each individual run matters).

### 2a.6 `-delete` — always pair it with a specific filter

```bash
find . -name "*.tmp" -delete
```

> ⚠️ [VERIFY] Confirmed directly: this deleted only files matching
> `*.tmp`, leaving unrelated files untouched.

The danger isn't `-delete` itself — it's running it with a filter
broader than intended. `find . -delete` with no filter at all
deletes everything under `.`, recursively, no confirmation. Always
run the exact same expression with `-print` instead of `-delete`
first, read the output, and only then swap to `-delete` once you've
confirmed the match set is exactly what you expect.

### 2a.7 `find | xargs` vs. `find -exec +` — same problem, two tools

```bash
find . -name "*.txt" -print0 | xargs -0 -n1 echo "via xargs:"
find . -name "*.txt" -exec echo "via exec:" {} +
```

Both exist to solve "run a command against every match, safely and
efficiently" — `find -exec +` needs no pipe and no second tool;
`find | xargs` is more flexible when you need `xargs`-specific
features (`-P` for parallelism, `-I` for repositioning the
argument) that `-exec` doesn't offer. Neither is strictly better;
`-exec +` is simpler when a plain batched command is all you need.

---

## Part 2 — Hands-On Lab

### Commands used in this demo

| Command | Flag meaning | Purpose |
|---|---|---|
| `find -name` | — | Match by filename (glob pattern) |
| `find -type` | — | Match by type: `f` file, `d` directory |
| `find -mtime` | — | Match by modification age, in days (`+N` older, `-N` newer) |
| `find -newer` | — | Match files modified after a reference file's timestamp |
| `find -prune` | — | Stop descent into a matched path |
| `find -exec ... \;` | — | Run a command once per match |
| `find -exec ... +` | — | Run a command once, batched across all matches |
| `find -delete` | — | Delete every match — always pair with a specific filter |

### Task 1 — Name, type, and time matching

```bash
mkdir -p ~/find-lab && cd ~/find-lab
touch -d "10 days ago" old_file.log
touch -d "1 hour ago" recent_file.log
touch normal_file.txt
mkdir -p subdir/nested excluded_dir
touch subdir/a.txt subdir/nested/b.txt excluded_dir/should_be_skipped.txt

find . -name "*.log"
find . -type d
find . -name "*.log" -mtime +5
find . -name "*.log" -mtime -5
```

### Task 2 — `-prune` an excluded directory

```bash
echo "--- without -prune ---"
find . -name "*.txt"
echo "--- with -prune ---"
find . -path ./excluded_dir -prune -o -name "*.txt" -print
```

### Task 3 — `-newer` against a reference file

```bash
touch -d "5 hours ago" reference.marker
find . -name "*.log" -newer reference.marker
```

### Task 4 — `-exec \;` vs. `-exec +`, side by side

```bash
find . -name "*.txt" -exec echo "one:" {} \;
find . -name "*.txt" -exec echo "batched:" {} +
```

### Task 5 — Safe `-delete` workflow

```bash
mkdir -p delete-test && cd delete-test
touch keep.txt remove.tmp remove2.tmp

find . -name "*.tmp"           # ALWAYS preview with -print first
find . -name "*.tmp" -delete   # only then swap to -delete
ls                              # confirm only keep.txt remains
cd ..
```

---

## What You Learned

- ✅ Matched files by name, type, and modification time
- ✅ Reproduced `-prune` actually preventing descent into a
  directory, not just filtering its results afterward
- ✅ Reproduced the `\;` vs. `+` difference directly — one invocation
  per match vs. one batched invocation for all matches
- ✅ Ran a safe `-delete` workflow: preview with `-print`, confirm,
  then swap to `-delete`
- ✅ Connected `find -exec +` and `find | xargs` as two tools solving
  the same underlying batching problem

**Key Takeaway:** `find` is a query language, not a flag list — every
test combines with implicit AND, `-prune` stops descent (not just
filters output), and `-exec +`/`xargs` batching exists for the exact
same performance reason as any other tool that avoids spawning one
process per item.

---

## Part 3 — Break-Fix Scenario

### Scenario — NovaTech: the cleanup job deleted more than intended

You're handed this ticket:

> *"Our nightly cleanup job is supposed to delete `.tmp` files older
> than 7 days from `/var/tmp/app-cache`. Someone changed it recently
> and now it's deleting current, in-use cache files too — files that
> were created an hour ago are gone."*

You find the relevant line:

```bash
find /var/tmp/app-cache -name "*.tmp" -delete
```

**Your task — answer from what this demo covered before revealing
the resolution:**

1. What's missing from this command that would explain deleting
   files regardless of age?
2. What's the safe way to verify the fix before actually running it
   with `-delete`?
3. What's the corrected command?

<details>
<summary>Resolution — open after attempting</summary>

**1. What's missing:**

The `-mtime +7` age filter is gone. The command as written deletes
every `.tmp` file, regardless of how old it is — the "older than 7
days" requirement was silently dropped at some point, and `-delete`
happily deletes anything matching whatever filters remain, with no
awareness that an age check was ever intended.

**2. Safe verification:**

```bash
find /var/tmp/app-cache -name "*.tmp" -mtime +7 -print
```

Run the exact intended expression with `-print` instead of
`-delete` first, read the output line by line, confirm it's exactly
the set of files that should be removed, and only then change
`-print` to `-delete`.

**3. The corrected command:**

```bash
find /var/tmp/app-cache -name "*.tmp" -mtime +7 -delete
```

**The lesson:** `-delete` executes exactly what the rest of the
expression matched — it has no independent judgment about what
"should" be deleted. Every filter in the expression is load-bearing;
losing one silently (a bad edit, a copy-paste error) changes what
gets deleted with no warning, which is exactly why previewing with
`-print` before ever attaching `-delete` is non-negotiable for
anything destructive.

</details>

---

## Interview Prep

**Q1. Why does `-prune` need to be combined with `-o -print`, rather
than working on its own?**

`-prune` only stops `find` from descending into a matched path — it
doesn't request any output on its own. Without an explicit `-o
... -print` for the rest of the tree, nothing would print at all.
The `-o` (OR) combines "prune this path" with "otherwise, print
matches," so the two don't cancel each other out.

---

**Q2. When would you deliberately choose `-exec cmd {} \;` over the
batched `-exec cmd {} +`, given `+` is generally more efficient?**

When the command genuinely needs to run once per file individually
— it only accepts one argument at a time, or you need to check each
invocation's own exit code separately rather than one combined
result for a whole batch. For anything that can accept multiple
filenames at once and doesn't need per-file exit status, `+` is the
better default.

---

**Q3. A teammate runs `find . -delete` directly, with no other
filter, intending to clean up a specific set of temp files. What
happened, and what should they have done first?**

With no filter, `-delete` matches and removes every file under the
starting path, recursively, with no confirmation — not just the
intended temp files. They should have run the exact same expression
with `-print` instead of `-delete` first, verified the output was
exactly the intended file set, and only then switched to `-delete`.

---

## Key Takeaways

1. **`find` evaluates a chain of tests combined with implicit AND — it's a small query language, not an independent flag list.** Every test in the expression narrows the match set together.
2. **`-prune` stops descent into a directory entirely — confirmed directly that pruned content is never visited, not filtered out after the fact.** It must be combined with `-o ... -print` or nothing prints for the rest of the tree.
3. **`-exec cmd {} \;` runs once per match; `-exec cmd {} +` batches all matches into as few invocations as possible — confirmed directly, and this is the same performance principle behind `xargs` batching from Demo 00b.**
4. **`-delete` executes exactly what the rest of the expression matched, with no independent judgment — always preview with `-print` before ever swapping to `-delete`,** especially after any edit to the filters.
5. **`find -exec +` and `find | xargs` solve the same batching problem — reach for `-exec +` when a plain batched command is enough, and `xargs` when you need its extra features (`-P` parallelism, `-I` repositioning).**
6. **`-mtime` counts in whole days; `-newer` compares directly against another file's actual timestamp** — more precise when the boundary you care about isn't a clean day count.

---

## Quick Commands Reference

### `find` test flags

| Flag | Matches |
|---|---|
| `-name "pattern"` | Filename (glob pattern, quote it) |
| `-type f` / `-type d` | Regular files / directories |
| `-mtime +N` / `-mtime -N` | Modified more / less than N days ago |
| `-newer file` | Modified after `file`'s timestamp |

### `find` action flags

| Flag | Effect |
|---|---|
| `-print` | Print matches (default if no other action given) |
| `-delete` | Delete matches — always preview with `-print` first |
| `-exec cmd {} \;` | Run once per match |
| `-exec cmd {} +` | Run once, batched across all matches |
| `-prune` | Stop descent into a matched path (pair with `-o ... -print`) |

---

## Appendix A — Anki Cards

**`02a-find-deep-dive-anki.csv`:**

```
#deck:Linux Mastery::Module 1 - Filesystem & Navigation::Demo 02a - find in Depth
#separator:Comma
#columns:Front,Back,Tags

"find . -name '*.txt' -mtime +5 -type f combines three tests. How are they combined - AND, OR, or something else?","Implicit AND. find evaluates a chain of tests left to right, and by default a file must pass ALL of them to be printed - it's a small query language, not a list of independent options.","demo02a,find,expression,and"

"find . -name '*.txt' is run both with and without -prune excluding a subdirectory. What's the actual difference in behavior, not just output?","Without -prune, find DESCENDS INTO the excluded directory and then its contents show up as matches. With -prune, find never visits that directory's contents at all - it's excluded from traversal itself, not filtered out of results afterward. Confirmed directly: pruned content is completely absent, not just hidden.","demo02a,find,prune,traversal"

"Why must -prune be combined with -o -print rather than used alone?","-prune only stops descent into a matched path - it doesn't request any output by itself. Without an explicit -o (OR) combined with -print for the rest of the tree, nothing would print at all. The -o keeps 'prune this path' and 'print everything else' from canceling each other out.","demo02a,find,prune,print"

"find . -name '*.txt' -exec echo {} \\; and find . -name '*.txt' -exec echo {} + both run echo against matches. What's the actual difference?","\\; runs the command ONCE PER MATCH - a new process per file. + batches ALL matches into as few invocations as possible, passing them all as arguments to one call. Confirmed directly: \\; produced one output line per file; + produced one output line total with every match listed.","demo02a,find,exec,batching"

"When would you deliberately choose -exec cmd {} \\; over the more efficient -exec cmd {} +?","When the command genuinely needs to run once per file individually - it only accepts one argument at a time, or you need each invocation's own exit code checked separately rather than one combined result for a whole batch.","demo02a,find,exec,usecase"

"find . -delete is run with no other filter. What happens, and why is this dangerous?","It deletes EVERY file under the starting path, recursively, with no confirmation - -delete has no independent judgment, it executes exactly what the rest of the expression matched. With no filter, that's everything. This is why previewing with -print instead of -delete first is non-negotiable before anything destructive.","demo02a,find,delete,footgun"

"A cleanup script's find command lost its -mtime +7 filter somewhere along the way (a bad edit), leaving just find /path -name '*.tmp' -delete. What's the practical consequence?","It now deletes ALL .tmp files regardless of age, not just ones older than 7 days - including current, in-use files. -delete doesn't know an age filter was ever intended; it just executes whatever filters remain. Every filter in a destructive find expression is load-bearing.","demo02a,find,delete,break-fix"

"What's the safe workflow before running any find command with -delete?","Run the EXACT same expression with -print instead of -delete first. Read the output line by line and confirm it's precisely the intended set of files. Only then change -print to -delete. Never attach -delete to an expression you haven't previewed.","demo02a,find,delete,workflow"

"What's the difference between find -exec {} + and find -print0 | xargs -0, given both solve batching?","Both batch matches into as few invocations as possible for the same performance reason. find -exec + needs no pipe or second tool - simpler when a plain batched command is enough. find | xargs is more flexible when you need xargs-specific features like -P (parallelism) or -I (argument repositioning) that -exec doesn't offer.","demo02a,find,xargs,exec,comparison"

"What's the difference between -mtime +5 and -newer reference_file for time-based matching?","-mtile counts in whole days relative to now - a coarser boundary. -newer compares directly against another file's actual timestamp, which is more precise when the exact boundary you care about isn't a clean day count (e.g. 'modified since the last deploy', not 'modified in the last N days').","demo02a,find,mtime,newer"
```

---

## Appendix B — Quiz

**`02a-find-deep-dive-quiz.md`:**

```
# Quiz — Demo 02a: find in Depth

> One correct answer per question. Target: 80% before Demo 03.

---

**Q1.** `find . -name "*.log" -type f -mtime +30` combines three
tests. How are they combined?

- A) OR — a file matching any one test is printed
- B) Implicit AND — a file must pass all three tests to be printed
- C) The last test overrides the earlier ones
- D) They're evaluated in random order each run

<details>
<summary>Answer</summary>

**B** — `find` evaluates a chain of tests combined with implicit
AND by default; it's a small query language, not independent
options.

</details>

---

**Q2.** `find . -path ./excluded -prune -o -name "*.txt" -print` is
run. What actually happens to `excluded`'s contents?

- A) They're matched but filtered out of the final output
- B) `find` never descends into `excluded` at all — its contents are
  never visited during traversal
- C) `excluded` itself is deleted
- D) Only files directly inside `excluded` (not nested deeper) are skipped

<details>
<summary>Answer</summary>

**B** — `-prune` stops descent entirely, confirmed directly: pruned
content doesn't appear because it was never visited, not because it
was filtered afterward.

</details>

---

**Q3.** Why must `-prune` be combined with `-o ... -print` rather
than used by itself?

- A) `-prune` alone causes a syntax error
- B) `-prune` only stops descent — it doesn't request any output;
  without `-o -print`, nothing prints for the rest of the tree either
- C) `-o` is required syntax with no functional purpose
- D) `-prune` requires root privileges without `-o`

<details>
<summary>Answer</summary>

**B** — `-prune`'s only job is stopping traversal into a path; the
`-o -print` is what makes everything else actually print, combined
so the two don't cancel out.

</details>

---

**Q4.** `find . -name "*.txt" -exec echo {} \;` vs. `find . -name
"*.txt" -exec echo {} +` — what's the actual difference?

- A) `\;` is faster than `+`
- B) `\;` runs the command once per match (separate invocations); `+`
  batches all matches into as few invocations as possible
- C) `+` only works with `echo`, not other commands
- D) They produce identical behavior, just different syntax

<details>
<summary>Answer</summary>

**B** — Confirmed directly: `\;` produced one output line per file
(N separate invocations); `+` produced one output line total, all
matches passed to a single invocation.

</details>

---

**Q5.** `find /tmp/cache -delete` is run with no other filter. What
happens?

- A) Nothing — `-delete` requires at least one filter to run at all
- B) Every file under `/tmp/cache`, recursively, is deleted with no
  confirmation
- C) Only empty files are deleted
- D) `find` prompts for confirmation before deleting anything

<details>
<summary>Answer</summary>

**B** — `-delete` has no independent judgment; it deletes exactly
what the rest of the expression matched. With no filter, that's
everything under the starting path.

</details>

---

**Q6.** What's the safe workflow before attaching `-delete` to any
`find` expression?

- A) Run it with `-delete` once on a test system first
- B) Run the exact same expression with `-print` instead of
  `-delete`, confirm the output is exactly the intended file set,
  then switch to `-delete`
- C) Add `-i` for interactive confirmation — `find` supports this by default
- D) `-delete` is always safe if `-name` is specified

<details>
<summary>Answer</summary>

**B** — Previewing with `-print` first is the only way to confirm
the match set before anything destructive runs — `find` has no
built-in interactive confirmation flag like some other tools do.

</details>

---

**Q7.** `find -exec cmd {} +` and `find | xargs` both solve the same
underlying problem. What's the practical reason to choose `xargs`
over `-exec +`?

- A) `xargs` is always faster
- B) You need an `xargs`-specific feature `-exec` doesn't offer, like
  `-P` (parallelism) or `-I` (argument repositioning)
- C) `-exec +` doesn't actually work, it's an alias for `\;`
- D) `xargs` is required for any command with more than one argument

<details>
<summary>Answer</summary>

**B** — Both batch for the same performance reason; `-exec +` is
simpler when a plain batched command is all you need, `xargs` earns
its extra pipe when you need its specific additional features.

</details>

---

Score guide:

| Score | Action |
|---|---|
| 7/7 | Import Anki cards, move to Demo 03 |
| 6/7 | Review the wrong answer, then proceed |
| 5/7 | Re-read the relevant section, retry those questions |
| Below 5/7 | Re-read the full demo and redo the walkthrough before proceeding |
```