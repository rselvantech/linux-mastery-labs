# Demo M02-06 — Cron & Log Rotation

## Overview

Demo M02-05 compared systemd timers against cron but never taught cron's own
syntax — this demo closes that gap. It also covers a topic mentioned in
passing since Demo M01-07 (`/var/log` grows forever unless something manages it)
but never explained: log rotation. The two belong together because
`logrotate`'s own trigger, on many systems, still runs *through* cron — the
same scheduling mechanism you're learning here, not a separate one.

---

## Demo Objectives

By the end of this demo you will be able to:

1. Read and write correct crontab syntax — all five schedule fields, step
   values, and the `@reboot`/`@daily`-style shorthands
2. Explain the difference between a user crontab, `/etc/crontab`, and
   `/etc/cron.d/` files — specifically the extra field system-wide cron
   files require that user crontabs do not
3. Explain `run-parts` and the `/etc/cron.{hourly,daily,weekly,monthly}/`
   directories, including the naming rule that silently skips scripts
4. Explain `cron.allow`/`cron.deny` and who can use cron on a given system
5. Write a correct `logrotate` configuration: `rotate`, `compress`,
   `missingok`, `notifempty`, `size`, and `postrotate`/`endscript`
6. Explain how `logrotate` actually gets triggered — via cron on some
   systems, via a `.timer`/`.service` pair on others — and connect this
   back to Demo M02-05
7. Diagnose a `logrotate` configuration that silently isn't rotating a log

---

## Directory Structure

```
06-cron-log-rotation/
├── README.md                          # this file
├── m02-06-cron-log-rotation-anki.csv  # Anki flashcard deck
└── m02-06-cron-log-rotation-quiz.md   # standalone quiz
```

---

## Environments

| Task | Ubuntu 24.04 | Rocky Linux 9.8 | EC2 Amazon Linux |
|---|---|---|---|
| All lab tasks | ✅ | ✅ | Not required |
| cron package name | `cron` | `cronie` (provides the same `crond` daemon) | Same as Rocky9 |
| `logrotate` install | Present by default in most installs; `sudo apt install logrotate` if missing | Present by default in most installs; `sudo dnf install logrotate` if missing | Present by default |
| `logrotate` trigger mechanism | `/etc/cron.daily/logrotate` (cron-based) on most current images | Often a `logrotate.timer`/`logrotate.service` pair (systemd-based) — verify with Task 6, don't assume | Verify with Task 6 |

The cron package name difference (`cron` vs `cronie`) is the same kind of
distro-family naming split you've already seen with `sudo`/`wheel` (Demo M02-02)
and `ssh`/`sshd` unit names (Demo M02-04) — different name, same job.

---

## Recall Check

> Answer from memory before reading further. From Demo M02-05.

1. A `.timer` unit has `OnCalendar=daily` and `Persistent=true`. The machine
   is off for exactly one missed run. How many catch-up runs happen when
   it's powered back on?

2. What's the practical advantage of a `.timer`-triggered service's logging
   over a cron job's, according to Demo M02-05's comparison?

3. You edit an existing `.timer` file's schedule. What must you run before
   the change takes effect?

<details>
<summary>Answers</summary>

1. One — `Persistent=true` fires a single catch-up run for the most recent
   missed occurrence, not one run per missed day.

2. Timer-triggered services log to `journalctl` in the same place as every
   other systemd-managed service, with the same `systemctl status`
   visibility — no separate log file or syslog convention to remember,
   unlike cron.

3. `systemctl daemon-reload` — systemd caches unit definitions in memory and
   won't see file changes until told to re-read them.

</details>

---

## Part 1 — Concepts

### 1.1 crontab syntax — the five fields

A crontab line has five time fields, in a fixed order, followed by the
command to run:

```
┌───────────── minute (0-59)
│ ┌───────────── hour (0-23)
│ │ ┌───────────── day of month (1-31)
│ │ │ ┌───────────── month (1-12)
│ │ │ │ ┌───────────── day of week (0-7, both 0 and 7 = Sunday)
│ │ │ │ │
* * * * *  command-to-run
```

```bash
# Every day at 2:30 AM
30 2 * * * /usr/local/bin/backup.sh

# Every 15 minutes
*/15 * * * * /usr/local/bin/healthcheck.sh

# Weekdays only, 9 AM
0 9 * * 1-5 /usr/local/bin/report.sh

# First of every month, midnight
0 0 1 * * /usr/local/bin/monthly-invoice.sh
```

**`*/N` is a step value** — "every N units, starting from the field's
minimum" — not "every Nth occurrence of something else." `*/15` in the
minute field means `:00, :15, :30, :45`, always aligned to the hour, not
"15 minutes after whenever this line was added."

**Special shorthand strings** — supported by both Ubuntu's `cron` and Rocky's
`cronie`, replacing all five fields at once:

```
@reboot    Run once, at startup
@daily     Same as: 0 0 * * *
@hourly    Same as: 0 * * * *
@weekly    Same as: 0 0 * * 0
@monthly   Same as: 0 0 1 * *
```

**Editing your own crontab:**

```bash
crontab -e      # edit your own crontab (opens $EDITOR)
crontab -l      # list your own crontab
crontab -r      # remove your entire crontab — no confirmation prompt, be careful
```

### 1.2 User crontabs vs system-wide cron — the field count that catches people

`crontab -e` edits a **user crontab**, stored per-user (commonly under
`/var/spool/cron/crontabs/username` on Debian-family, `/var/spool/cron/username`
on RHEL-family — a real path difference worth confirming on whichever system
you're actually on, since `crontab -l` is the reliable way to check contents
regardless of exact storage path). Every line in it runs **as that user** —
there's no user field, because the file itself already belongs to one user.

**`/etc/crontab` and files under `/etc/cron.d/` are different — they have
one extra field.** Because these files aren't owned by a single user the way
a personal crontab is, each line must say who to run the command as:

```
# /etc/crontab or /etc/cron.d/myjob — six fields, not five
┌───────────── minute
│ ┌───────────── hour
│ │ ┌───────────── day of month
│ │ │ ┌───────────── month
│ │ │ │ ┌───────────── day of week
│ │ │ │ │ ┌───────────── USER — the extra field
│ │ │ │ │ │
30 2 * * * root /usr/local/bin/system-backup.sh
```

**The common real mistake:** copying a working line from a personal
`crontab -e` entry into an `/etc/cron.d/` file without adding the user field.
The line silently fails to run — cron doesn't send an error to your terminal
for a malformed `/etc/cron.d/` entry, it just never executes it (or, on some
implementations, the second field is misread as part of the command, which
then fails for an unrelated-looking reason). Demo M02-05's Break-Fix Incident 1
in this demo is exactly this mistake.

### 1.3 run-parts and the cron.{hourly,daily,weekly,monthly} directories

Rather than editing crontab lines directly for routine maintenance jobs,
both distro families provide four directories that already have a schedule
wired up via `/etc/crontab` (or an equivalent mechanism):

```
/etc/cron.hourly/
/etc/cron.daily/
/etc/cron.weekly/
/etc/cron.monthly/
```

A script placed in one of these directories runs on that cadence — you
don't write a crontab line yourself; you just drop a script in the right
directory. This is invoked via **`run-parts`**, which executes every
runnable file in a directory in name order.

**The naming rule that silently skips scripts:** `run-parts` (as used by
Debian/Ubuntu's cron package) **skips any filename containing a period**
by default. A script named `backup.sh` placed in `/etc/cron.daily/` will
**not** run — the `.sh` extension is exactly what causes it to be skipped.
The fix is simple once you know it: name the script `backup` (no extension)
and ensure it's executable (`chmod +x`).

```bash
sudo tee /etc/cron.daily/cleanup-temp > /dev/null << 'EOF'
#!/bin/bash
find /tmp -type f -mtime +7 -delete
EOF
sudo chmod +x /etc/cron.daily/cleanup-temp
# Note: no .sh extension — this is deliberate, not an oversight
```

### 1.4 cron.allow / cron.deny — who can use cron at all

Two optional files control which users may create their own crontabs:

```
/etc/cron.allow    If this file exists, ONLY the users listed in it may
                   use crontab -e. Everyone else is denied, even root's
                   own convenience is not automatically exempted here.

/etc/cron.deny     If cron.allow does NOT exist but cron.deny does, every
                   user EXCEPT those listed may use crontab -e.

Neither file        On both Ubuntu's cron and Rocky's cronie, the default
exists              behavior is to allow all users — an empty cron.deny
                   ships by default on many installs, which has the same
                   practical effect as "allow everyone" once no one's name
                   is actually listed in it.
```

```bash
cat /etc/cron.allow 2>/dev/null || echo "no cron.allow"
cat /etc/cron.deny 2>/dev/null || echo "no cron.deny"
```

Restricting cron access this way matters on shared, multi-user servers where
not every account should be able to schedule arbitrary recurring commands —
a rarer concern on a single-purpose cloud instance, but a real one on shared
university, research, or internal dev servers.

### 1.5 logrotate — the problem it solves

Every service that writes logs (`nginx`, `sshd`, your own applications) will
fill a disk eventually if nothing manages log file size and age. `logrotate`
is the standard tool for this: it renames, compresses, and eventually deletes
old log files on a schedule, based on per-application configuration.

```
/etc/logrotate.conf     Global defaults (applies unless a per-app config
                        overrides them)
/etc/logrotate.d/       One file per application/service — the same
                        drop-in-directory pattern you've already seen with
                        /etc/sudoers.d/ (Demo M02-02), /etc/systemd/system/
                        (Demo M02-04), and /etc/cron.d/ (this demo, §1.2)
```

```bash
cat /etc/logrotate.d/nginx 2>/dev/null || echo "not installed on this system"
```

### 1.6 Writing a logrotate configuration

```
# /etc/logrotate.d/novatech-app
/var/log/novatech-app/*.log {
    daily
    rotate 14
    compress
    delaycompress
    missingok
    notifempty
    size 50M
    create 0640 novatech-svc novatech-svc
    postrotate
        systemctl reload novatech-app > /dev/null 2>&1 || true
    endscript
}
```

**Directive by directive:**

```
daily / weekly / monthly    How often to consider rotating this log
rotate 14                   Keep 14 old rotated copies before deleting
                            the oldest
compress                    gzip old rotated logs (all but the newest
                            one, if delaycompress is also set)
delaycompress               Delay compressing the MOST recently rotated
                            file by one more cycle — the app may still
                            have it open briefly; avoids compressing a
                            file mid-write
missingok                   Don't error if the log file doesn't exist
                            yet — common for a service that hasn't
                            logged anything since install
notifempty                  Don't rotate if the log file is empty —
                            avoids piling up empty rotated files for an
                            idle service
size 50M                    Rotate early if the log exceeds this size,
                            even before the daily/weekly/monthly
                            schedule would otherwise trigger it
create MODE USER GROUP      After rotating, create a new empty log file
                            with these permissions — critical for a
                            service account (Demo M02-02) that expects to
                            keep writing to the same filename
postrotate / endscript      A shell command block run once, after
                            rotation — almost always used to tell the
                            service to reopen its log file, since simply
                            renaming a file out from under a running
                            process does NOT make it start writing to
                            the new one automatically
```

**Why `postrotate` matters, concretely:** renaming a log file doesn't close
the file descriptor a running process already has open to it (Demo M01-08's
inode/directory-entry distinction applies directly here — the process still
holds the old inode via its open file descriptor, even after the filename
changes). Without a `postrotate` step telling the service to reopen its log
(`systemctl reload`, `kill -HUP`, or an app-specific reload command), the
service keeps writing to the renamed — soon to be compressed and eventually
deleted — old file, while the new file logrotate created sits empty forever.

**`create` vs `copytruncate`** — two different ways to solve the same
problem for apps that can't be told to reopen their log file:

```
create        Rename the old file, create a new empty one. Requires the
              app to reopen its log (postrotate) or it keeps writing to
              the renamed file.

copytruncate   COPY the current log's contents to the rotated filename,
              then truncate the ORIGINAL file to zero length in place —
              no rename happens at all, so the app's already-open file
              descriptor keeps working correctly with no reopen needed.
              Trade-off: a small window between the copy and the
              truncate where log lines could theoretically be lost.
```

Use `copytruncate` specifically when you cannot make the application reopen
its log file (no reload signal, no config-reload command) — otherwise
`create` + `postrotate` is the more common and slightly safer default.

### 1.7 How logrotate actually gets triggered

`logrotate` itself does not run continuously in the background waiting to
rotate things — something has to invoke it on a schedule, and which
mechanism does that varies by distro and install:

```
Cron-based (traditional):     /etc/cron.daily/logrotate — a script that
                              runs logrotate once a day, itself scheduled
                              via run-parts (§1.3) — the exact mechanism
                              this demo just covered.

systemd-timer-based (newer):  logrotate.timer + logrotate.service —
                              exactly the .timer/.service pairing from
                              Demo M02-05, just applied to logrotate instead
                              of a custom NovaTech job.
```

```bash
# Check which mechanism your system actually uses — don't assume:
ls /etc/cron.daily/logrotate 2>/dev/null && echo "cron-based"
systemctl list-timers | grep -i logrotate && echo "timer-based"
```

Either way, **the concepts you already know cover it completely** — this
demo didn't need to teach a third scheduling mechanism, because logrotate's
own trigger is always one of the two you already have.

### 1.8 Diagnosing a logrotate config that "does nothing"

```bash
sudo logrotate -d /etc/logrotate.d/novatech-app
# -d = debug/dry-run — shows exactly what logrotate WOULD do, without
# actually rotating anything. Always run this after writing or editing a
# config, before waiting for the real schedule to (maybe) trigger it.

sudo logrotate -f /etc/logrotate.d/novatech-app
# -f = force — actually rotates NOW regardless of schedule or size
# thresholds, for testing that postrotate/create actually work correctly
```

**Why a config can look correct but silently never rotate:** `logrotate`
tracks the last rotation time for every configured log in a status file —
commonly `/var/lib/logrotate/status` on Debian-family systems (the exact
path can vary by distro packaging, so confirm on your own system rather than
assuming). If the status file says a log was already rotated today, and the
schedule is `daily`, logrotate correctly does nothing when it runs again
later the same day — this can look identical to "logrotate isn't working"
if you're testing repeatedly in a short window. `logrotate -f` bypasses this
check entirely, which is exactly why it's the right tool for testing.

---

## Part 2 — Hands-On Lab

### Commands used in this demo

| Command | Flag meaning | Purpose |
|---|---|---|
| `crontab -e` | — | Edit your own user crontab |
| `crontab -l` | — | List your own user crontab |
| `crontab -r` | — | Remove your entire crontab |
| `run-parts /dir` | — | Run every executable, dot-free-named file in a directory |
| `logrotate -d FILE` | `-d` = debug/dry-run | Show what would happen without rotating |
| `logrotate -f FILE` | `-f` = force | Rotate immediately, ignoring schedule/size checks |

### Task 1 — crontab syntax, written and verified

```bash
crontab -e
# Add these lines, save and exit:
# */10 * * * * /usr/bin/true
# 0 3 * * 0 /usr/bin/true

crontab -l
# Confirm both lines are present exactly as written
```

### Task 2 — The user-field mistake, demonstrated

```bash
# WRONG — copied straight from a user crontab, missing the user field
sudo tee /etc/cron.d/broken-job > /dev/null << 'EOF'
* * * * * /bin/bash -c 'echo "ran at $(date)" >> /tmp/broken-job.log'
EOF

sleep 90
cat /tmp/broken-job.log 2>&1 || echo "never ran — confirms the missing-field problem"

# CORRECT — with the user field added
sudo tee /etc/cron.d/working-job > /dev/null << 'EOF'
* * * * * root /bin/bash -c 'echo "ran at $(date)" >> /tmp/working-job.log'
EOF

sleep 90
cat /tmp/working-job.log
# Should show at least one line — confirms the fix

sudo rm -f /etc/cron.d/broken-job /etc/cron.d/working-job
```

### Task 3 — run-parts and the dot-in-filename rule

```bash
# WRONG — .sh extension means run-parts skips this file
sudo tee /etc/cron.daily/skip-me.sh > /dev/null << 'EOF'
#!/bin/bash
echo "if you see this, run-parts did NOT skip it" >> /tmp/rp-test.log
EOF
sudo chmod +x /etc/cron.daily/skip-me.sh

sudo run-parts --test /etc/cron.daily
# --test: print what WOULD run, without running it — skip-me.sh should NOT appear

# CORRECT — no dot in the name
sudo cp /etc/cron.daily/skip-me.sh /etc/cron.daily/run-me
sudo run-parts --test /etc/cron.daily
# run-me SHOULD appear this time

sudo rm -f /etc/cron.daily/skip-me.sh /etc/cron.daily/run-me
```

### Task 4 — cron.allow / cron.deny

```bash
cat /etc/cron.allow 2>/dev/null || echo "no cron.allow present"
cat /etc/cron.deny 2>/dev/null || echo "no cron.deny present"
# Note which combination exists on your system, and per §1.4, what that
# combination means for who can currently use crontab -e
```

### Task 5 — Write and test a logrotate config

```bash
sudo mkdir -p /var/log/novatech-app
sudo bash -c 'echo "sample log line" > /var/log/novatech-app/app.log'

sudo tee /etc/logrotate.d/novatech-app > /dev/null << 'EOF'
/var/log/novatech-app/*.log {
    daily
    rotate 14
    compress
    delaycompress
    missingok
    notifempty
    size 50M
    create 0640 root root
    postrotate
        echo "postrotate ran at $(date)" >> /tmp/postrotate-test.log
    endscript
}
EOF

# Dry run first — always
sudo logrotate -d /etc/logrotate.d/novatech-app

# Now force it for real
sudo logrotate -f /etc/logrotate.d/novatech-app

ls -la /var/log/novatech-app/
# Expect: app.log (new, empty) and app.log.1 (the rotated original)
cat /tmp/postrotate-test.log
# Confirms the postrotate block actually executed
```

### Task 6 — Find logrotate's actual trigger mechanism on your system

```bash
ls /etc/cron.daily/logrotate 2>/dev/null && echo "This system uses cron to trigger logrotate"
systemctl list-timers --all | grep -i logrotate && echo "This system uses a systemd timer to trigger logrotate"
# Note which one your system actually uses — do not assume based on distro
# alone, confirm it directly, consistent with Demo M02-05's own list-timers habit
```

---

## What You Learned

- ✅ Read and wrote correct crontab syntax — all five fields, step values, and shorthand strings
- ✅ Explained the extra user field required in `/etc/crontab`/`/etc/cron.d/` files that user crontabs don't have
- ✅ Explained `run-parts` and demonstrated the dot-in-filename rule that silently skips scripts
- ✅ Explained `cron.allow`/`cron.deny` and checked which combination applies on a real system
- ✅ Wrote a correct `logrotate` configuration, including `postrotate`/`endscript`
- ✅ Explained how `logrotate` is actually triggered — cron or a systemd timer, depending on the system — and confirmed which one applies here
- ✅ Diagnosed a `logrotate` configuration using `-d` (dry-run) and `-f` (force), and explained why the status file can make a correct config look broken

**Key Takeaway:** Both halves of this demo share the same real failure mode — a config that looks syntactically correct but silently does nothing, because of one missing detail (the user field in system-wide cron, or a dot in a `run-parts`-scanned filename) or one piece of hidden state (logrotate's status file). `run-parts --test` and `logrotate -d` exist specifically so you can catch this before waiting for a schedule to (maybe) prove you wrong.

---

## Part 3 — Break-Fix Scenario

### Scenario — NovaTech Ops: two silent scheduling failures

> **Incident report — NovaTech Ops, ticket #4531**
>
> Alert 1: a nightly reconciliation job that was "definitely added to cron"
> never seems to run — no errors, no output, nothing. Alert 2: a service's
> log directory has ballooned to 40GB despite a `logrotate` config that
> "looks right" to everyone who's read it.

**Incident 1 — the cron job that never runs**

```bash
$ cat /etc/cron.d/reconcile-job
0 2 * * * /opt/novatech/bin/reconcile.sh
```

Tasks:
1. This file is under `/etc/cron.d/`, not a personal crontab. What's
   missing, and why does cron not simply error out loudly about it?
2. Write the corrected line.
3. What command would have caught this before waiting a full day to
   discover the job never ran?

**Incident 2 — the log directory that never shrinks**

```bash
$ cat /etc/logrotate.d/bloated-app
/var/log/bloated-app/*.log {
    daily
    rotate 7
    missingok
    notifempty
}
$ ls -la /var/log/bloated-app/
-rw-r--r-- 1 appuser appuser 40G Sep  8 03:00 app.log
```

Tasks:
1. `rotate 7` and `daily` both look correct. There's no `size` directive
   and no `compress` — could either of those, on their own, explain a
   single file reaching 40GB? What's actually missing from this config
   that would explain rotation never happening at all, versus rotation
   happening but not helping?
2. Run `logrotate -d` mentally against this config (or actually, if you
   have a test system) — what would you check first to determine whether
   rotation ran and failed, versus never ran at all?
3. Assuming rotation genuinely never triggered — what's the single most
   likely explanation given everything covered in §1.7 and §1.8?

<details>
<summary>Answers</summary>

**Incident 1:** The user field is missing. `/etc/cron.d/` files require six
fields (the extra one specifying which user to run as), unlike a personal
crontab where every line implicitly runs as the file's owner. Cron doesn't
loudly error on this because the malformed line doesn't necessarily fail
outright — depending on the cron implementation, it may simply never match a
valid schedule entry, or misinterpret the path as the user field and fail
silently in a way that produces no visible output anywhere a casual check
would find it. Corrected line:
```
0 2 * * * root /opt/novatech/bin/reconcile.sh
```
The command that would have caught this immediately: reviewing the file
itself against §1.2's six-field rule before deploying it — there's no
built-in `cron.d` linter, which is exactly why knowing this rule matters more
here than for most syntax details in this demo.

**Incident 2:** Neither `rotate 7` nor `daily` prevents a single log file
from growing huge *between* rotations — without a `size` directive, this
config only rotates on the daily schedule, regardless of how large the file
gets in the meantime; a sufficiently chatty app can still produce a 40GB
file in one day before the next scheduled rotation. That's a config
*design* gap (missing `size`), not necessarily a rotation *failure* — the
two questions are different: "is 40GB explained by no size cap" (yes,
plausible) versus "did rotation ever actually run" (needs checking
separately). To tell them apart: check for the existence of rotated files
(`app.log.1`, `app.log.2.gz`, etc.) — their presence means rotation has been
running, just not often enough or with generous enough limits; their total
absence means rotation never triggered at all, pointing back to §1.7 (check
whether logrotate's cron/timer trigger is actually enabled and firing) rather
than a config-tuning problem.

</details>

---

## Interview Prep

**Q1. A crontab line works fine when tested via `crontab -e`, but the identical line placed in `/etc/cron.d/` never runs. Why?**

`/etc/crontab` and files under `/etc/cron.d/` require an extra field — the
user to run the command as — that a personal `crontab -e` entry doesn't
need, since a personal crontab already belongs to one user implicitly.
Copying a five-field line from a personal crontab into a `cron.d` file
without adding that sixth field is one of the most common cron mistakes,
and it tends to fail silently rather than with an obvious error.

---

**Q2. A script placed in `/etc/cron.daily/` never runs, even though it's executable and the syntax is correct. What's the most likely cause?**

The filename likely contains a period — `run-parts` (as used by Debian/Ubuntu's
cron package) skips any file with a dot in its name by default, so
`backup.sh` is silently ignored while a script simply named `backup` would
run correctly. `run-parts --test` shows exactly what would execute without
actually running anything, which is the fastest way to confirm this.

---

**Q3. Why does a `postrotate` block matter for a service whose log `logrotate` just rotated?**

Renaming a log file doesn't close a process's already-open file descriptor
to it — the process keeps writing to the renamed (and soon compressed) old
file unless something tells it to reopen its log. `postrotate` runs a
command (commonly `systemctl reload` or a signal) specifically to make the
service open a fresh file handle to the newly-created log path. Without it,
the new file logrotate created sits empty while the service keeps writing to
what's effectively a vanishing file.

---

**Q4. Your team is debating whether `logrotate` is triggered by cron or by a systemd timer on a given server. How do you find out, rather than assume based on the distro?**

Check both directly: `ls /etc/cron.daily/logrotate` for the cron-based
mechanism, and `systemctl list-timers --all | grep -i logrotate` for the
timer-based one. Which mechanism a given install uses can vary even within
the same distro family depending on packaging choices, so confirming
directly — the same habit Demo M02-05 established for any timer — is more
reliable than assuming.

---

## Key Takeaways

1. **A personal crontab (`crontab -e`) has five fields; `/etc/crontab` and `/etc/cron.d/` files have six — the extra one is the user to run as.** Copying a line between the two without adding or removing that field is the single most common cron mistake, and it fails silently.
2. **`run-parts` skips any filename containing a dot by default.** A script in `/etc/cron.daily/` named with a `.sh` extension simply never runs — `run-parts --test` reveals this before you'd otherwise discover it a day later.
3. **`cron.allow`/`cron.deny` control who may schedule their own cron jobs at all** — most single-purpose systems have neither file meaningfully restricting anyone, but shared multi-user servers often do.
4. **A `logrotate` config needs `size` if a single day's growth alone could be a problem** — `rotate`/`daily` alone only bound how many old copies exist and how often rotation is considered, not how large a file can get between rotations.
5. **`postrotate`/`endscript` exists because renaming a file doesn't close a running process's file handle to it.** Without telling the service to reopen its log, it keeps writing to the renamed (soon-compressed) old file while the new one sits empty.
6. **`logrotate` itself is triggered by cron on some systems and a systemd `.timer` on others — verify directly, never assume.** Either way, it's one of the two scheduling mechanisms already covered in this series, not a third one.
7. **`logrotate -d` (dry-run) and `-f` (force) are the two commands for testing a config without waiting on its real schedule** — `-f` specifically bypasses the status-file check that can make a correct config look inert if you test it twice in the same day.

---

## Quick Commands Reference

### crontab

| Command | Purpose |
|---|---|
| `crontab -e` | Edit your own crontab |
| `crontab -l` | List your own crontab |
| `crontab -r` | Remove your entire crontab |
| `run-parts --test /dir` | Show what would run, without running it |

### logrotate

| Command | Purpose |
|---|---|
| `logrotate -d /etc/logrotate.d/name` | Dry-run — show what would happen |
| `logrotate -f /etc/logrotate.d/name` | Force rotation now, for testing |
| `cat /etc/logrotate.d/name` | View a specific application's config |

### Trigger-mechanism check

| Command | Purpose |
|---|---|
| `ls /etc/cron.daily/logrotate` | Confirm cron-based trigger |
| `systemctl list-timers \| grep logrotate` | Confirm timer-based trigger |

---

## Appendix A — Anki Cards

**`m02-06-cron-log-rotation-anki.csv`:**

```
#deck:Linux Mastery::Module 2 - System Administration::Demo M02-06 - Cron and Log Rotation
#separator:Comma
#columns:Front,Back,Tags

"A crontab line works via crontab -e but the identical line in /etc/cron.d/ never runs. Why?","/etc/crontab and /etc/cron.d/ files require an extra sixth field (the user to run as) that personal crontabs don't need, since a personal crontab already belongs to one user implicitly. Copying a five-field line into a cron.d file without adding the user field is a common mistake that fails silently.","demo-m02-06,cron,common-mistake"

"A script named backup.sh in /etc/cron.daily/ never runs despite being executable with correct syntax. Why?","run-parts (used by Debian/Ubuntu's cron package) skips any filename containing a dot by default. Renaming it to just backup (no extension) fixes it. run-parts --test shows exactly what would run without executing anything, catching this before waiting a day to discover it.","demo-m02-06,run-parts,common-mistake"

"What does the @daily shorthand expand to in crontab syntax?","0 0 * * * — midnight every day. Also: @reboot (once at startup), @hourly (0 * * * *), @weekly (0 0 * * 0), @monthly (0 0 1 * *).","demo-m02-06,crontab,shorthand"

"In crontab syntax, what does */15 in the minute field actually mean?","A step value: every 15 minutes starting from 0, always aligned to the hour — :00, :15, :30, :45. It does not mean '15 minutes after whenever this line was added.'","demo-m02-06,crontab,syntax"

"What does postrotate/endscript in a logrotate config actually solve?","Renaming a log file during rotation doesn't close a running process's already-open file descriptor to it. Without telling the service to reopen its log (postrotate running systemctl reload or a signal), the process keeps writing to the renamed, soon-to-be-compressed old file while the newly created file sits empty.","demo-m02-06,logrotate,postrotate"

"A logrotate config has rotate 7 and daily but no size directive. Could a single log file still grow to 40GB in one day?","Yes — rotate and daily only control how many old copies are kept and how often rotation is considered, not how large a file can grow between scheduled rotations. A size directive is needed to force early rotation if a file grows unusually large before the next scheduled check.","demo-m02-06,logrotate,size"

"How do you determine whether a given system triggers logrotate via cron or via a systemd timer, rather than assuming based on distro?","Check directly: ls /etc/cron.daily/logrotate for the cron-based mechanism, or systemctl list-timers --all | grep -i logrotate for the timer-based mechanism. Packaging choices can vary even within the same distro family.","demo-m02-06,logrotate,cross-demo"

"What is the difference between logrotate's create and copytruncate directives?","create renames the old file and creates a new empty one — requires the app to reopen its log (via postrotate) or it keeps writing to the renamed file. copytruncate copies the current log's contents to the rotated filename, then truncates the original in place — no rename happens, so no reopen is needed, at the cost of a small window where log lines could theoretically be lost between the copy and the truncate.","demo-m02-06,logrotate,copytruncate"

"A logrotate config looks correct but appears to do nothing when tested twice in the same day. What's the likely explanation, and what command bypasses it?","logrotate tracks the last rotation time per config in a status file. If it already rotated that log today and the schedule is daily, running logrotate again the same day correctly does nothing — this can look like a broken config. logrotate -f forces rotation immediately, bypassing the status-file check, which is why it's the right tool for testing.","demo-m02-06,logrotate,troubleshooting"

"If neither cron.allow nor cron.deny exists on a system, who can use crontab -e?","On both Ubuntu's cron and Rocky's cronie, the default behavior when neither file exists (or an empty cron.deny ships by default) is to allow all users to use crontab -e — cron.allow, if present, restricts to only the listed users; cron.deny, if present without cron.allow, blocks only the listed users.","demo-m02-06,cron-allow,cron-deny"
```

---

## Appendix B — Quiz

**`m02-06-cron-log-rotation-quiz.md`:**

````markdown
# Quiz — Demo M02-06: Cron & Log Rotation

> One correct answer per question unless stated otherwise.
> Target: 80% or above before moving to Demo M02-07.

---

**Q1.** A working line from a personal crontab (`crontab -e`) is copied
verbatim into `/etc/cron.d/nightly-job`. It never runs. What's the most
likely reason?

- A) `/etc/cron.d/` files require an absolute path for the command, which
  the personal crontab entry didn't have
- B) `/etc/cron.d/` files require a sixth field specifying which user to
  run the command as — a personal crontab entry has only five fields
  because it already belongs to one user implicitly
- C) `/etc/cron.d/` files are read-only after boot and cannot accept new entries
- D) System-wide cron files require root's crontab specifically, not any arbitrary user's

<details>
<summary>Answer</summary>

**B** — The extra user field is exactly what's missing. This is one of the
most common real cron mistakes, and it typically fails with no visible
error at all.

</details>

---

**Q2.** A script named `sync-report.sh` is placed in `/etc/cron.daily/`,
made executable, and has correct internal syntax. It never runs via the
daily cron cycle. Why?

- A) Scripts in `/etc/cron.daily/` must be renamed with a `.cron` extension
- B) `run-parts` skips any filename containing a dot by default — the
  `.sh` extension is exactly what causes it to be silently ignored
- C) `/etc/cron.daily/` only runs binaries, never shell scripts
- D) The script needs a shebang line pointing specifically to `/bin/dash`

<details>
<summary>Answer</summary>

**B** — Renaming it to `sync-report` (no extension) fixes this. `run-parts
--test` shows exactly what would run without executing anything, which
catches this before waiting a day to find out.

</details>

---

**Q3.** What does `*/10` in the minute field of a crontab line actually mean?

- A) Run 10 times total, then stop
- B) Every 10 minutes, aligned to the hour — `:00, :10, :20, :30, :40, :50`
- C) 10 minutes after the crontab was last edited
- D) Run once, 10 minutes after boot

<details>
<summary>Answer</summary>

**B** — Step values in crontab syntax are always aligned to the field's own
minimum, not relative to when the line was added or the system last booted.

</details>

---

**Q4.** Why does a `logrotate` configuration typically need a `postrotate`/
`endscript` block for an actively-running service?

- A) It's required by `logrotate`'s syntax even when nothing needs to happen after rotation
- B) Renaming a log file during rotation doesn't close the running
  process's already-open file descriptor to it — without telling the
  service to reopen its log, it keeps writing to the renamed (and soon
  compressed) old file while the new file sits empty
- C) `postrotate` is what actually performs the file rename; without it, no rotation happens at all
- D) It only matters for services that log to `journalctl`, not plain files

<details>
<summary>Answer</summary>

**B** — This connects directly to how inodes and open file descriptors
work (Demo M01-08): the rename doesn't affect what file a running process is
already writing to. `postrotate` (commonly `systemctl reload` or a signal)
is what makes the service pick up the new file.

</details>

---

**Q5.** A `logrotate` config has `rotate 7` and `daily`, with no `size`
directive. A single log file grows to 40GB in one day. Does this config
prevent that?

- A) Yes — `rotate 7` caps any single file at a proportional size automatically
- B) No — `rotate`/`daily` only control how many old copies are kept and
  how often rotation is considered, not how large a file can grow between
  scheduled rotations; a `size` directive is needed to force early
  rotation regardless of schedule
- C) Yes, but only if `compress` is also set
- D) No, and there is no `logrotate` directive that could address this at all

<details>
<summary>Answer</summary>

**B** — Size and schedule are two independent concerns in `logrotate`.
Without `size`, a chatty log can grow arbitrarily large between its
scheduled rotation checks.

</details>

---

**Q6.** You test a `logrotate` config twice within the same day and it
appears to do nothing the second time, even though the config looks
correct. What's the most likely explanation, and how do you test around it?

- A) `logrotate` is broken and needs to be reinstalled
- B) `logrotate` tracks the last rotation time per config in a status
  file — if it already rotated today, running it again the same day
  correctly does nothing under a `daily` schedule; `logrotate -f` forces
  rotation immediately, bypassing this check
- C) The config file must be re-created from scratch before every test
- D) `logrotate` only runs once per boot, regardless of testing

<details>
<summary>Answer</summary>

**B** — This is expected, correct behavior being mistaken for a bug.
`logrotate -f` is specifically the right tool when you need to force a test
regardless of the status file's record.

</details>

---

**Q7.** How do you determine whether `logrotate` is triggered by cron or by
a systemd timer on a specific server, rather than assuming based on the
distro?

- A) Check `/etc/os-release` — the distro name alone determines which
  mechanism is used, with no exceptions
- B) Check directly: `ls /etc/cron.daily/logrotate` for the cron-based
  path, or `systemctl list-timers --all | grep -i logrotate` for the
  timer-based one — packaging choices can vary even within a distro family
- C) `logrotate --show-trigger` reports this directly
- D) It cannot be determined without contacting the distro maintainers

<details>
<summary>Answer</summary>

**B** — This is the same "verify directly, don't assume" habit Demo M02-05
established for timers generally — packaging decisions aren't guaranteed
to be uniform across every install of a given distro.

</details>

---

**Q8.** If neither `cron.allow` nor `cron.deny` exists on a system, what's
the default behavior for `crontab -e`?

- A) No user can use `crontab -e` until one of the two files is created
- B) Only `root` can use `crontab -e`
- C) All users can use `crontab -e` — this is the default behavior on both
  Ubuntu's `cron` and Rocky's `cronie` when neither restriction file exists
- D) `crontab -e` prompts interactively to create `cron.allow` on first use

<details>
<summary>Answer</summary>

**C** — Absent any restriction file, cron implementations on both distro
families default to allowing all users, matching the common real-world
default for single-purpose or lightly-shared systems.

</details>

---

| Score | Action |
|---|---|
| 8/8 | Import Anki cards, move to Demo M02-07 |
| 6-7/8 | Review the wrong answers, then proceed |
| 4-5/8 | Re-read the relevant sections, retry those questions |
| Below 4/8 | Re-read the full demo and redo the walkthrough before proceeding |
````