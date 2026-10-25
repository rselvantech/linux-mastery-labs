# Demo M07-04 — Basic sysctl Tuning

## Overview

Demo M07-02 limited what one *service* may consume. Some limits and
behaviours, though, belong to the whole *host*: how eagerly the kernel
swaps, how long a listening socket's queue may grow, how many file handles
the system may hold open. Linux exposes these as runtime parameters you
read and change with `sysctl`. They are also the easiest place to do
damage: a change takes effect instantly, produces no warning if it is
wrong, and — if you only change it at runtime — silently disappears at the
next reboot.

This demo teaches the safe mechanics: how to read a parameter, change it,
make the change persistent, predict which configuration file wins, and
notice the ways a change fails silently. It deliberately stays at the
administrative level — what a parameter controls and what can go wrong —
not the kernel internals behind it, which are outside this curriculum.

**Real-world scenario:** an engineer lowers `vm.swappiness` on a database
host, watches the effect, and goes home. After the next patch reboot the
setting is gone and the latency problem is back. A second engineer "fixes"
it by adding a file that never applies, because of one missing
three-letter suffix.

## Demo Objectives

By the end of this demo you will be able to:

1. Read a parameter with `sysctl` and map its dotted name to its
   `/proc/sys` path
2. Change a parameter at runtime with `sysctl -w`, and show that nothing
   on disk records the change
3. Persist a change in `/etc/sysctl.d/` and apply it with `sysctl --system`
4. Predict which file wins when two files set the same key, and recognize
   the silent failure modes: a file without `.conf`, and a mistyped key
5. Follow a safe-change workflow — record, change, measure, persist,
   verify, and write down a rollback
6. Explain what three common parameters control, and the risk of changing
   each
7. Choose between `sysctl`, a systemd directive (Demo M07-02), and a
   per-process limit for a given problem

## Directory Structure

```
04-sysctl-tuning/
├── README.md                      # this file
├── m07-04-sysctl-tuning-anki.csv  # Anki flashcard deck
└── m07-04-sysctl-tuning-quiz.md   # standalone quiz
```

## Environments

| Task | Ubuntu 24.04 | Rocky Linux 9.8 | EC2 Amazon Linux |
|---|---|---|---|
| All lab tasks | ✅ (`sysctl` is part of procps, preinstalled; writing needs root) | ✅ (same tool; paths below are the standard ones — confirm on your image) | Not required |

> **Verification note:** every `sysctl` behaviour in this demo — reads,
> `-w`, `-p`, `--system` ordering, the ignored file without `.conf`, the
> silent mistyped key, and `--system` not restoring values — was tested
> live in the authoring sandbox (Ubuntu 24.04.4, procps-ng 4.0.4, as
> root). Two things could not be tested there: that the settings are
> applied **at boot** (systemd is not PID 1 in the sandbox), and behaviour
> on Rocky Linux. Both are marked `[Unverified — test before use]` where
> they appear. Parameter values shown (for example `fs.file-max`) are from
> the authoring host and will differ on yours.

## Recall Check — 03-resource-saturation-triage

Answer these from memory before continuing — no looking back at Demo M07-03.

1. A guest VM looks saturated (`r` above the CPU count, no idle time) but
   `st` is high. Can you fix it from inside the guest?
2. Why do you discard the first row of `vmstat` output?
3. How does the run queue `r` tell you a CPU is saturated rather than just
   busy?

<details>
<summary>Answers</summary>

1. No — steal time is capacity the hypervisor never gave the guest; the
   fix is a different instance type or host, and adding threads makes the
   run queue longer without adding capacity.
2. It is an average since boot, not a live reading.
3. Compare it to the CPU count: `r` above the number of CPUs means work is
   waiting for a CPU (saturated); `r` at or below it with high `us` means
   busy but nothing queueing.

</details>

---

## Part 1 — Concepts

### 1.1 What sysctl reads and writes

The kernel exposes its tunable parameters as files under `/proc/sys`.
`sysctl` is the front end: it turns a dotted name into a path, so
`net.core.somaxconn` is the file `/proc/sys/net/core/somaxconn`.

```bash
sysctl vm.swappiness fs.file-max net.core.somaxconn
```

```
vm.swappiness = 60
fs.file-max = 407239
net.core.somaxconn = 4096
```

Both views show the same value:

```bash
sysctl -n net.core.somaxconn        # 4096
cat /proc/sys/net/core/somaxconn    # 4096
```

| Flag | Meaning | Use |
|---|---|---|
| `-a` | all | List every parameter (791 lines on the authoring host — the count varies by kernel) |
| `-n` | no name | Print the value only, useful in scripts |
| `-N` | names only | List names without values (49 `vm.*` names on the authoring host) |
| `-w` | write | Set a parameter at runtime |
| `-p [file]` | load | Load settings from a file; with no argument, from `/etc/sysctl.conf` |
| `--system` | load everything | Load every standard configuration location |

Change a value and the underlying file reflects it immediately:

```bash
sudo sysctl -w vm.swappiness=10
cat /proc/sys/vm/swappiness         # 10
```

### 1.2 Runtime versus persistent

`sysctl -w` changes the running kernel only. Nothing is written to disk,
which you can prove by searching for it:

```bash
grep -r vm.swappiness /etc/sysctl.d /etc/sysctl.conf /usr/lib/sysctl.d
# no output — the running value exists only in the kernel
```

To survive a reboot the setting must be in a configuration file. The
convention is one file per purpose in `/etc/sysctl.d/`. Ubuntu ships
several; one real example from the authoring host:

```
# /etc/sysctl.d/10-map-count.conf
# Increase the number of virtual memory areas that one process may request
# https://bugs.launchpad.net/ubuntu/+source/procps/+bug/2057792
vm.max_map_count=1048576
```

The format is `key = value` (spaces optional), with `#` starting a
comment. At boot the settings are applied by `systemd-sysctl` reading the
same locations `sysctl --system` reads `[Unverified — test before use:
systemd is not PID 1 in the authoring sandbox]`.

### 1.3 Which file wins

`sysctl --system` applies files in **lexical order of file name**, across
the standard directories, and the **last** setting for a key wins. The
standard locations are `/etc/sysctl.d`, `/run/sysctl.d`, and
`/usr/lib/sysctl.d`, plus `/etc/sysctl.conf` last (per `sysctl.d(5)`; only
`/etc/sysctl.d` and `/usr/lib/sysctl.d` were observed on the authoring
host). Files in `/etc` shadow same-named files in `/usr/lib`
`[Unverified — test before use]`.

Tested live: `50-demo.conf` set `vm.swappiness = 20`, `90-demo.conf` set
`vm.swappiness = 30`, and `60-noext` (no suffix) set `vm.swappiness = 40`.
Captured with output and errors piped together, so the two kinds of line
may group rather than interleave:

```
* Applying /etc/sysctl.d/50-demo.conf ...
* Applying /etc/sysctl.d/90-demo.conf ...
vm.swappiness = 20
vm.swappiness = 30
```

`sysctl -n vm.swappiness` afterwards printed `30`: the later file name won.
`60-noext` was never listed and never applied.

**Practical rule:** name your own overrides with a prefix that sorts
*after* the files you intend to override (for example `90-` or higher),
give them a descriptive name, and always end them in `.conf`.

### 1.4 Silent failures — a change can fail without telling you

Tested live on procps-ng 4.0.4:

| What you did | What happened |
|---|---|
| Mistyped a key with `sysctl -w vm.swapiness=10` | `sysctl: cannot stat /proc/sys/vm/swapiness: No such file or directory`, exit status 1 |
| Out-of-range value: `sysctl -w vm.swappiness=999` | `sysctl: setting key "vm.swappiness": Invalid argument`, exit status 1 |
| File in `/etc/sysctl.d` without a `.conf` suffix | Ignored; not listed in `--system` output |
| Mistyped key **inside a file**, with a valid line after it | `--system` printed **no error** for the typo; the valid line still applied |
| Two files set the same key | The later file name wins, with no warning about the conflict |

Two more observations to keep you honest about automation:

- `sysctl --system` returned exit status 1 on the authoring host even
  *without* any file of ours, because one shipped key
  (`net.core.default_qdisc`) does not exist in that environment. A
  non-zero exit therefore does not prove *your* file failed — and a zero
  exit does not prove it worked.
- `sysctl --system` only writes values found in files. Removing your file
  and re-running it does **not** restore the old value: on the authoring
  host, after setting `vm.swappiness=25` with no file mentioning that key,
  `sysctl --system` left it at 25.

The dependable check is always to **read the value back** after applying.

### 1.5 A safe-change workflow

1. **Record** the current value (`sysctl -n key`) and why you are
   changing it.
2. **Change** it at runtime with `sysctl -w`.
3. **Measure** against the symptom that motivated the change, using the
   tools from Demos M07-01/26c. If nothing improves, revert.
4. **Persist** it in a named `.conf` file, with a comment giving the
   reason, the date, and the previous value.
5. **Apply and verify**: `sudo sysctl --system`, then read the value back.
6. **Write the rollback**: remove the file *and* set the old value (the
   file removal alone does not revert the running kernel — see §1.4).

### 1.6 Three parameters, at the administrative level

**`vm.swappiness`** — how strongly the kernel prefers reclaiming memory by
swapping out process memory versus dropping cache. Lower values favour
keeping process memory in RAM. It is the host-wide counterpart to the
per-service controls from Demo M07-02. Risk: too low, and a machine under
memory pressure with little cache to drop can reach out-of-memory sooner.
The accepted range depends on the kernel version (the authoring host
rejected 999).

**`net.core.somaxconn`** — an upper bound on the accept-queue length a
listening socket may request. Per `listen(2)`, a larger `backlog` than this
value is silently capped to it, so raising `somaxconn` alone changes
nothing if the application itself asks for a small backlog — the
application's own setting has to be raised too.

**`fs.file-max`** — the system-wide ceiling on open file handles.
`/proc/sys/fs/file-nr` shows three numbers: allocated, unused, and the
maximum (`66  0  407239` on the authoring host). Most "too many open
files" errors are hit at the **per-process** limit long before this
system-wide one — see `/proc/PID/limits` (Demo M07-01) and `LimitNOFILE=` in a
unit file (Demo M07-02). A related parameter, `kernel.perf_event_paranoid`
(2 on the authoring host), controls how much access unprivileged users get
to the `perf` tools from Demo M07-01.

### 1.7 What sysctl is not for

| Problem | Right tool |
|---|---|
| Cap one service's memory or CPU | systemd directives — `MemoryMax=`, `CPUQuota=` (Demo M07-02) |
| Raise one process's open-file limit | `LimitNOFILE=` / `ulimit -n`; confirm in `/proc/PID/limits` |
| Change host-wide kernel behaviour | `sysctl` |
| Kernel modules and kernel internals | outside this curriculum |

Do not paste a list of "performance tunables" from a blog post into a
production host. Each parameter needs a measured symptom behind it.

---

## Part 2 — Hands-On Lab

### Commands used in this demo

| Command | Flag meaning | Purpose |
|---|---|---|
| `sysctl KEY [KEY...]` | — | Read parameters |
| `sysctl -n KEY` | `-n`: value only | Read for scripts |
| `sysctl -a` / `sysctl -N -a` | `-a`: all, `-N`: names only | List parameters |
| `sysctl -w KEY=VALUE` | `-w`: write | Set at runtime (needs root) |
| `sysctl -p [FILE]` | `-p`: load | Load one file, or `/etc/sysctl.conf` |
| `sysctl --system` | — | Load every standard location |
| `grep -r KEY DIRS` | `-r`: recursive | Find where a key is configured on disk |

### Task 1 — Read parameters

```bash
sysctl vm.swappiness fs.file-max net.core.somaxconn
sysctl -n net.core.somaxconn
cat /proc/sys/net/core/somaxconn
cat /proc/sys/fs/file-nr
sysctl -a 2>/dev/null | wc -l
sysctl -N -a 2>/dev/null | grep -c '^vm\.'
```

Confirm the two views of `somaxconn` agree, and that `file-nr` shows
three numbers.

### Task 2 — Change at runtime, and prove it is not persistent

```bash
sysctl -n vm.swappiness                     # record the current value
sudo sysctl -w vm.swappiness=10
sysctl vm.swappiness
cat /proc/sys/vm/swappiness
grep -r vm.swappiness /etc/sysctl.d /etc/sysctl.conf /usr/lib/sysctl.d
sudo sysctl -w vm.swapiness=10              # deliberate typo
sudo sysctl -w vm.swappiness=999            # deliberate out-of-range
```

Confirm the running value is 10, the `grep` finds nothing, and both errors
appear with exit status 1 (`echo $?`).

### Task 3 — Persist, apply, and verify

```bash
sudo sysctl -w vm.swappiness=60             # start from a known value
echo 'vm.swappiness = 10' | sudo tee /etc/sysctl.d/90-novatech-tuning.conf
sysctl -n vm.swappiness                     # still 60 — the file is not applied yet
sudo sysctl --system | grep novatech
sysctl -n vm.swappiness                     # now 10
```

### Task 4 — Precedence and silent failures

```bash
echo 'vm.swappiness = 20' | sudo tee /etc/sysctl.d/50-demo.conf
echo 'vm.swappiness = 40' | sudo tee /etc/sysctl.d/60-noext
sudo sysctl --system | grep -E 'demo|noext|novatech'
sysctl -n vm.swappiness
```

Predict the final value before you look. `90-novatech-tuning.conf` sorts
last, so it should win (10); `50-demo.conf` is applied earlier; `60-noext`
should not appear at all. Then test the silent typo:

```bash
printf 'vm.swapiness = 5\n' | sudo tee /etc/sysctl.d/95-typo.conf
sudo sysctl --system | grep -A1 95-typo
sysctl -n vm.swappiness                     # unchanged
```

Confirm no error names the typo, and that reading the value you *meant* to
change is what reveals nothing happened. Finally show that removing a file
does not undo its effect:

```bash
sudo rm /etc/sysctl.d/90-novatech-tuning.conf
sudo sysctl --system > /dev/null 2>&1
sysctl -n vm.swappiness                     # the value from the last file applied — not the old default
```

### Task 5 — Roll back properly

```bash
sudo rm -f /etc/sysctl.d/50-demo.conf /etc/sysctl.d/60-noext /etc/sysctl.d/95-typo.conf
sudo sysctl -w vm.swappiness=60             # the value you recorded in Task 2
sysctl -n vm.swappiness
grep -r vm.swappiness /etc/sysctl.d /etc/sysctl.conf /usr/lib/sysctl.d
```

Confirm the value is back and no file on disk still sets it.

### Task 6 — Pick the right tool

For each problem, name the mechanism (sysctl, a systemd directive, or a
per-process limit) and how you would confirm it:

1. One batch service keeps growing and must be capped at 512 MB without
   affecting anything else on the host.
2. A service logs "Too many open files" while `/proc/sys/fs/file-nr` shows
   the allocated count far below the maximum.
3. Under load a web server drops connections; its configuration asks for a
   listen backlog of 128 and `net.core.somaxconn` is 4096.

<details>
<summary>Answers</summary>

1. A systemd directive — `MemoryMax=512M` in the unit (Demo M07-02). A sysctl
   would change the whole host, not one service.
2. A per-process limit — the service has hit its own open-files limit, not
   the system-wide `fs.file-max`. Confirm with `/proc/PID/limits`; raise it
   with `LimitNOFILE=` in the unit.
3. The application's own backlog setting. `listen(2)` caps a request at
   `somaxconn`, but a request of 128 stays 128 — raising `somaxconn` alone
   changes nothing until the application asks for more.

</details>

### Cleanup

```bash
sudo rm -f /etc/sysctl.d/50-demo.conf /etc/sysctl.d/60-noext \
           /etc/sysctl.d/90-novatech-tuning.conf /etc/sysctl.d/95-typo.conf
sudo sysctl -w vm.swappiness=60
```

---

## What You Learned

- ✅ Read a parameter with `sysctl` and mapped its dotted name to its
  `/proc/sys` path
- ✅ Changed a parameter at runtime and proved nothing on disk recorded it
- ✅ Persisted a change in `/etc/sysctl.d/` and applied it with
  `sysctl --system`
- ✅ Predicted which file wins, and recognized the silent failures — a
  missing `.conf` suffix and a mistyped key
- ✅ Followed a record-change-measure-persist-verify-rollback workflow
- ✅ Explained what `vm.swappiness`, `net.core.somaxconn`, and
  `fs.file-max` control, and the risk of changing each
- ✅ Chose between sysctl, a systemd directive, and a per-process limit

---

## Part 3 — Break-Fix Scenario

### Scenario — NovaTech Ops: two "the tuning didn't stick" incidents

> **Incident report — NovaTech Ops, ticket #5080**
>
> Alert 1: "db-host-03 latency is back after last night's patch reboot.
> The swappiness fix from last week is gone." Alert 2: "We added the
> swappiness setting as a file in `/etc/sysctl.d`, ran `sysctl --system`,
> and the value still isn't what we set."

**Incident 1 — the fix that disappeared after a reboot**

```bash
$ sysctl -n vm.swappiness
60
$ grep -r vm.swappiness /etc/sysctl.d /etc/sysctl.conf /usr/lib/sysctl.d
$ history | grep swappiness
  412  sudo sysctl -w vm.swappiness=10
```

Tasks:
1. Reference §1.2 — why did the setting disappear, and what does the empty
   `grep` prove?
2. What is the fix, and what should the file contain besides the setting?
3. How can you confirm the file will apply, without rebooting the host?

**Incident 2 — a persisted setting that never takes effect**

`⚠️ Simulated scenario` — file names are illustrative; the behaviours are
the ones tested in Task 4.

```
/etc/sysctl.d/60-novatech-tuning       contains:  vm.swappiness = 10
/etc/sysctl.d/99-cloud-defaults.conf   contains:  vm.swappiness = 60
```

Tasks:
1. `sysctl --system` output never mentions `60-novatech-tuning`. Why?
2. After renaming it to `60-novatech-tuning.conf`, the value is still 60.
   Why?
3. Give two changes that would make the intended value apply, and the one
   command that proves it worked.

<details>
<summary>Answers</summary>

**Incident 1:** `sysctl -w` changes only the running kernel; nothing is
written to disk, so a reboot starts from the configured value (60 here).
The empty `grep` proves no file anywhere sets the key, so there was
nothing to re-apply. The fix is a named file such as
`/etc/sysctl.d/90-novatech-tuning.conf` containing the setting, plus a
comment with the reason, the date, and the previous value so the next
engineer can roll it back. Confirm without rebooting by running
`sudo sysctl --system` and reading the value back
(`sysctl -n vm.swappiness`); it should show the persisted value even if
you first set it to something else with `-w`. That validates the file, not
the boot-time load, which `systemd-sysctl` performs `[Unverified — test
before use]`.

**Incident 2:** The first file has no `.conf` suffix, and files without it
are ignored — they are never listed or applied. After renaming, both files
apply but in lexical order: `60-...` runs before `99-...`, and the later
file wins, so 60 overrides 10 with no warning. Either rename the tuning
file so it sorts after the one it must beat (for example
`995-...` or a name that sorts after `99-cloud-defaults`), or remove or edit
the `99-cloud-defaults.conf` setting if it is not needed. Prove it with
`sudo sysctl --system` followed by `sysctl -n vm.swappiness` — reading the
value back, because neither ignored files nor conflicts produce an error.

</details>

---

## Interview Prep

**Q1. What's the difference between `sysctl -w` and putting a setting in `/etc/sysctl.d/`?**

`sysctl -w` changes the running kernel only — nothing on disk records it,
so a reboot discards it. A file in `/etc/sysctl.d/` (ending in `.conf`) is
what gets re-applied at boot and by `sysctl --system`. A safe change uses
both: test at runtime, then persist once it is proven.

---

**Q2. Two files in `/etc/sysctl.d/` set the same key. Which one wins?**

The one that sorts later by file name — `sysctl --system` applies files in
lexical order and the last setting wins, with no warning about the
conflict. That is why local overrides are given a high-numbered prefix
such as `90-`, and why `grep -r KEY` across the sysctl directories is the
first check when a persisted value is not what you set.

---

**Q3. You persisted a tunable, `sysctl --system` printed no error, but the value did not change. What are the likely causes?**

The file lacks a `.conf` suffix and was ignored; a later-sorting file sets
the same key to something else; or the key is mistyped, which this
version of `sysctl` does not report when it appears inside a file. In every
case the reliable check is to read back the value you meant to change and
search the sysctl directories for the key.

---

**Q4. Why shouldn't you paste a list of recommended kernel tunables from an article onto a production host?**

Each parameter changes host-wide behaviour instantly, with no warning
when it is wrong, and the right value depends on the workload and
hardware. A change needs a measured symptom behind it, a recorded previous
value, verification afterwards, and a rollback — the record-change-
measure-persist-verify workflow — rather than a bulk copy.

---

## Key Takeaways

1. **`sysctl -w` changes only the running kernel.** Nothing on disk
   records it, so a reboot discards it; persist it in a `.conf` file under
   `/etc/sysctl.d/`.
2. **The dotted name is the path.** `net.core.somaxconn` is
   `/proc/sys/net/core/somaxconn`; both views show the same value.
3. **Later file names win.** `sysctl --system` applies files in lexical
   order and the last setting for a key wins, silently — name your own
   overrides so they sort after the files they must beat.
4. **Files without a `.conf` suffix are ignored, and a mistyped key inside
   a file produced no error** on procps-ng 4.0.4. Neither a quiet run nor
   the exit status proves a change applied; read the value back.
5. **Removing a file does not roll a change back.** `sysctl --system`
   writes only what files contain, so the old value must be set again
   explicitly.
6. **Know the scope of each tool.** `sysctl` changes host-wide behaviour;
   one service's limits belong in systemd directives (Demo M07-02); one
   process's open-file limit belongs in `LimitNOFILE=`/`ulimit`, confirmed
   in `/proc/PID/limits`.
7. **Raising `somaxconn` alone does nothing if the application asks for a
   small backlog** — `listen(2)` caps a request at the parameter, it does
   not raise a smaller one.

> **Demo scope:** Primary concept: reading, changing, and persisting
> kernel runtime parameters safely with sysctl. Supporting concepts:
> `/proc/sys` mapping, `/etc/sysctl.d` ordering and silent failures, the
> safe-change workflow, three common parameters.
> Estimated completion time: 30 minutes (reading + hands-on + verification).
> Checkpoints: 6 natural stopping points (after Task 1 through Task 6).

---

## Quick Commands Reference

### Reading

| Command | Purpose |
|---|---|
| `sysctl KEY` | Show a parameter and its value |
| `sysctl -n KEY` | Value only |
| `sysctl -a` / `sysctl -N -a` | List all parameters / names only |
| `cat /proc/sys/PATH` | The same value through its file |

### Changing and persisting

| Command | Purpose |
|---|---|
| `sudo sysctl -w KEY=VALUE` | Set at runtime only |
| `/etc/sysctl.d/NN-name.conf` | Persistent setting, applied in filename order |
| `sudo sysctl --system` | Apply every standard location |
| `sudo sysctl -p FILE` | Apply one file (no argument: `/etc/sysctl.conf`) |
| `grep -r KEY /etc/sysctl.d /etc/sysctl.conf /usr/lib/sysctl.d` | Find every place a key is configured |

---

## Appendix A — Anki Cards

**`m07-04-sysctl-tuning-anki.csv`:**

```
#deck:Linux Mastery::Module 7 - Performance::Demo M07-04 - sysctl Tuning
#separator:Comma
#columns:Front,Back,Tags
"Which sysctl flag sets a parameter at runtime?","-w (for example: sudo sysctl -w vm.swappiness=10). It changes the running kernel only.","demo26d,sysctl"
"What does sysctl -n do?","Prints the value only, without the name — useful in scripts.","demo26d,sysctl"
"What does sysctl -N do?","Lists parameter names only, without values.","demo26d,sysctl"
"Which /proc path corresponds to net.core.somaxconn?","/proc/sys/net/core/somaxconn — dots in the name become slashes under /proc/sys.","demo26d,sysctl,proc"
"Does a value set with sysctl -w survive a reboot?","No. Persist it in a .conf file under /etc/sysctl.d/.","demo26d,sysctl,persistence"
"Which command applies every standard sysctl configuration location?","sudo sysctl --system","demo26d,sysctl"
"In what order does sysctl --system apply files, and which setting wins?","Lexical order of file name across the directories; the last setting for a key wins.","demo26d,sysctl,precedence"
"What file suffix must a file in /etc/sysctl.d have to be applied?",".conf — files without it are ignored.","demo26d,sysctl,persistence"
"What does sysctl -p with no argument read?","/etc/sysctl.conf. Given a path, it loads only that file.","demo26d,sysctl"
"How does fs.file-max differ from LimitNOFILE= in a unit file?","fs.file-max is the system-wide ceiling on open file handles; LimitNOFILE= is the per-process/service limit, which is usually reached first.","demo26d,sysctl,limits,cross-demo"
```

---

## Appendix B — Quiz

**`m07-04-sysctl-tuning-quiz.md`:**

````markdown
# Quiz — Demo M07-04: Basic sysctl Tuning

> One correct answer per question unless stated otherwise.
> Target: 80% or above before moving to Demo M08-01.

---

**Q1.** An engineer runs `sudo sysctl -w vm.swappiness=10`, confirms it
with `sysctl vm.swappiness`, and the host is rebooted that night. The next
day the value is 60 again. What is the most likely explanation?

- A) The kernel ignores `sysctl -w` on production hosts
- B) `sysctl -w` changed only the running kernel; no file recorded it, so the reboot started from the configured value
- C) `vm.swappiness` cannot be persisted
- D) The reboot corrupted `/proc/sys`

<details>
<summary>Answer</summary>

**B** — nothing on disk recorded the change. A `.conf` file under
`/etc/sysctl.d/` is what survives a reboot.

</details>

---

**Q2.** Your file `/etc/sysctl.d/50-tuning.conf` sets `vm.swappiness = 10`.
The distribution's `/etc/sysctl.d/99-cloud.conf` sets it to 60. After
`sudo sysctl --system`, what is the value, and why?

- A) 10 — the lower number always wins
- B) 60 — files are applied in lexical order and the later file name wins, silently
- C) 10 — the file you wrote is always applied last
- D) The command fails with a conflict error

<details>
<summary>Answer</summary>

**B** — `99-cloud.conf` sorts after `50-tuning.conf`, so its value is
applied last. To win, your file must sort after it.

</details>

---

**Q3.** You created `/etc/sysctl.d/70-tuning` (no extension) with a valid
setting and ran `sudo sysctl --system`. Nothing changed and there was no
error. Why?

- A) Files must be executable to be applied
- B) Files without a `.conf` suffix are ignored — the file is never listed or applied
- C) `70-` is a reserved prefix
- D) `sysctl --system` only reads `/etc/sysctl.conf`

<details>
<summary>Answer</summary>

**B** — the ignored file does not even appear in the output, which is why
there is no error to notice.

</details>

---

**Q4.** A file contains `vm.swapiness = 5` (one `p`). `sudo sysctl --system`
prints no error. How do you find out the setting did not apply?

- A) Trust the exit status — zero means it worked
- B) Read back the value you meant to change (`sysctl -n vm.swappiness`) and see that it is unchanged
- C) Reboot and see whether the host fails to boot
- D) Nothing can be done; typos are undetectable

<details>
<summary>Answer</summary>

**B** — the mistyped key was silently ignored, and the exit status was
already non-zero on the authoring host for unrelated reasons. Reading back
the intended key is the reliable check.

</details>

---

**Q5.** Under load a web server drops connections. Its configuration asks
for a listen backlog of 128; `net.core.somaxconn` is 4096. An engineer
raises `somaxconn` to 65535. What changes?

- A) The backlog immediately becomes 65535
- B) Nothing — `listen(2)` caps a request at `somaxconn` but does not raise a smaller one; the application must ask for a larger backlog
- C) The server crashes
- D) The backlog becomes 4096

<details>
<summary>Answer</summary>

**B** — the parameter is a ceiling, not a default. The application's own
setting is the limiting value here.

</details>

---

**Q6.** One batch service is growing without bound and must be capped at
512 MB without affecting anything else on the host. Which mechanism fits?

- A) `sysctl -w vm.swappiness=0`
- B) A systemd directive on that service — `MemoryMax=512M` (Demo M07-02)
- C) `sysctl -w fs.file-max=512`
- D) Raising `net.core.somaxconn`

<details>
<summary>Answer</summary>

**B** — sysctl parameters are host-wide; a per-service cap belongs in the
unit file.

</details>

---

**Q7.** A service logs "Too many open files" while
`cat /proc/sys/fs/file-nr` shows the allocated count far below the maximum.
Where is the limit actually being hit?

- A) The system-wide `fs.file-max`
- B) The service's own per-process limit — confirm in `/proc/PID/limits` and raise it with `LimitNOFILE=`
- C) `vm.swappiness`
- D) The network backlog

<details>
<summary>Answer</summary>

**B** — the system-wide pool has plenty of room, so the process hit its
own open-files limit.

</details>

---

Score guide:

| Score | Action |
|---|---|
| 7/7 | Import Anki cards, move to Demo M08-01 |
| 6/7 | Review the wrong answer, then proceed |
| 5/7 | Re-read the relevant section, retry those questions |
| Below 5/7 | Re-read the full demo and redo the walkthrough before proceeding |
````