# Demo M02-05 — systemd Timers

## Overview

systemd doesn't just start and stop services — it can also schedule them. `.timer`
units are systemd's answer to cron: a first-class unit type that triggers another
unit (almost always a matching `.service`) on a calendar schedule, a delay after
boot, or a delay after the last run. This demo covers `.timer` unit anatomy, the
two scheduling styles (calendar-based and monotonic), how to make a timer survive
a machine being powered off when it should have fired, and when to reach for a
timer instead of cron in a production environment.

---

## Demo Objectives

By the end of this demo you will be able to:

1. Explain why systemd timers exist alongside cron, and when to choose each
2. Write a `.service` + `.timer` unit pair from scratch and enable it correctly
3. Configure `OnCalendar=` expressions and validate them with `systemd-analyze calendar`
4. Configure `OnBootSec=` and `OnUnitActiveSec=` monotonic timers
5. Configure `Persistent=true` and explain exactly what it does and does not guarantee
6. Read `systemctl list-timers` output to determine what will run next and why
7. Diagnose a timer that silently never fires

---

## Directory Structure

```
05-systemd-timers/
├── README.md                       # this file
├── m02-05-systemd-timers-anki.csv  # Anki flashcard deck
└── m02-05-systemd-timers-quiz.md   # standalone quiz
```

---

## Environments

| Task | Ubuntu 24.04 | Rocky Linux 9.8 | EC2 Amazon Linux |
|---|---|---|---|
| All lab tasks | ✅ | ✅ | Not required |

---

## Recall Check

> Answer from memory before reading further. From Demo M02-04.

1. You run `systemctl enable myapp.service` without `--now`, then reboot the
   machine. Is `myapp` running after the reboot? What about if you had run
   `systemctl start myapp.service` instead, without `enable`, and then rebooted?

2. Your unit file has `Restart=on-failure` but the service keeps failing and
   systemd stops trying after a few attempts. What directive controls that
   limit, and where does it live in the unit file?

3. You edit `/etc/systemd/system/myapp.service` directly. The service is
   already running. What single command must you run before your changes take
   effect, and what does skipping it leave you with?

<details>
<summary>Answers</summary>

1. No — `enable` without `--now` only creates the symlink in the target's
   `.wants/` directory; it doesn't start anything now. After reboot, systemd
   will start it because the symlink exists. With `start` alone (no `enable`),
   it runs now but there's no symlink — after reboot it will *not* come back.

2. `StartLimitBurst=` (paired with `StartLimitIntervalSec=`), inside the
   `[Unit]` section — not `[Service]`.

3. `systemctl daemon-reload`. Skipping it leaves systemd running against its
   in-memory copy of the old unit file — your edits exist on disk but have no
   effect until the daemon re-reads unit files, which `daemon-reload` triggers.

</details>

---

## Part 1 — Concepts

### 1.1 What is a systemd timer, and why not just cron?

A `.timer` unit is a systemd unit type whose entire job is to activate another
unit — almost always a same-named `.service` — on a schedule. Functionally it
overlaps with cron, but the two aren't interchangeable in practice:

| | cron | systemd timer |
|---|---|---|
| Logging | Own log file or syslog, easy to miss | `journalctl -u name.service` — same place as every other service log |
| Missed-run handling | None built in (anacron partially addresses this) | `Persistent=true` — catch up automatically |
| Dependency awareness | None — runs on schedule regardless of system state | Can depend on other units (`After=`, `Requires=`) via the paired service |
| Resource control | None | Full systemd resource control (`MemoryMax=`, `CPUQuota=`, etc.) via the service unit |
| Failure visibility | Silent unless you check mail/logs yourself | `systemctl status`, `systemctl list-timers`, standard unit failure states |
| Precision | Minute-level | Sub-second, with configurable `AccuracySec=` |

Neither is "correct" universally. cron is simpler, ubiquitous, and fine for
non-critical periodic jobs on a box you're not otherwise systemd-managing
closely. Timers earn their complexity when the job's reliability matters enough
that you want it logged, resource-controlled, and recoverable the same way your
services are.

### 1.2 Timer unit anatomy — the .timer/.service pairing

A timer unit almost always has the same base name as the service it triggers:

```ini
# novatech-backup.timer
[Unit]
Description=Run NovaTech backup on schedule

[Timer]
OnCalendar=*-*-* 02:00:00
Persistent=true

[Install]
WantedBy=timers.target
```

```ini
# novatech-backup.service
[Unit]
Description=NovaTech nightly backup job

[Service]
Type=oneshot
ExecStart=/usr/local/bin/novatech-backup.sh
```

**The naming convention is load-bearing, not cosmetic.** If a `.timer` doesn't
specify `Unit=` under `[Timer]`, systemd assumes the target is the identically
named `.service` — `novatech-backup.timer` activates `novatech-backup.service`
by default. Name them differently and you must add `Unit=other-name.service`
explicitly, or the timer will silently look for a service that doesn't exist
under the default name.

`Type=oneshot` matters here too: timer-triggered services are almost always
`oneshot` — they run once, do their job, and exit. systemd considers the run
"successful" or "failed" based on the exit code, which is what shows up in
`systemctl list-timers` and `journalctl`.

**Enabling a timer** uses the same four-combination model from Demo M02-04, applied
to the `.timer` unit, not the `.service`:

```bash
sudo systemctl enable --now novatech-backup.timer
# --now: enables (survives reboot) AND starts the timer counting immediately.
# Note: you enable/start the .timer, never the .service directly — the timer
# is what gets scheduled; the service only runs when the timer fires it.
```

### 1.3 OnCalendar= — calendar-based scheduling syntax

`OnCalendar=` expressions describe *when*, in real wall-clock time, a timer
should fire. The general shape is `weekday YYYY-MM-DD HH:MM:SS`, with `*` as a
wildcard for any field and `/` for "every N":

| Expression | Meaning |
|---|---|
| `*-*-* 02:00:00` | Every day at 02:00:00 |
| `Mon *-*-* 03:00:00` | Every Monday at 03:00 |
| `*-*-01 00:00:00` | First of every month, midnight |
| `*-*-* *:00/15:00` | Every 15 minutes, on the hour boundary |
| `Mon..Fri *-*-* 09:00:00` | Weekdays only, 09:00 |
| `daily` | Shorthand for `*-*-* 00:00:00` |
| `weekly` | Shorthand for `Mon *-*-* 00:00:00` |

Never hand-verify a calendar expression by reasoning about it — always check it:

```bash
systemd-analyze calendar "Mon..Fri *-*-* 09:00:00"
# Prints: normalized form, next elapse time, and time until next elapse.
# This catches typos (e.g. "Mon..Fri" mistyped as "Mon-Fri", which parses as
# something entirely different) before you've deployed a timer that never fires.
```

### 1.4 OnBootSec= / OnUnitActiveSec= — monotonic scheduling

Monotonic timers don't care about wall-clock time — they care about elapsed time
relative to an event:

```ini
[Timer]
OnBootSec=15min
OnUnitActiveSec=1h
```

- `OnBootSec=15min` — fire 15 minutes after the system boots
- `OnUnitActiveSec=1h` — fire again 1 hour after the *last time this unit
  activated* (i.e., repeating interval, measured from the previous run, not
  from boot or from a fixed clock time)

This combination — an initial `OnBootSec=` plus a repeating `OnUnitActiveSec=`
— is systemd's equivalent of "run shortly after startup, then every N
thereafter," without needing a calendar expression at all. It's the right
choice when the job cares about *frequency*, not a specific *time of day*
(health checks, cache warmers, periodic cleanup tasks).

### 1.5 Persistent=true — catching up missed runs

`Persistent=true` tells systemd: if this timer *would have* fired while the
system was off (or the timer was inactive), fire it once, immediately, the next
time the timer becomes active — then resume the normal schedule.

**What it actually checks:** systemd stores the last trigger time for a
persistent timer in `/var/lib/systemd/timers/`. On timer activation, it compares
that stored time against what the schedule says should have happened since —
if a scheduled run was missed, it fires once to catch up.

**What it does NOT do:**
- It does not fire once *per* missed occurrence — a laptop off for a week with a
  daily backup timer gets one catch-up run, not seven.
- It has no effect on `OnBootSec=`/`OnUnitActiveSec=`-only timers in a
  meaningful way beyond the same catch-up-once semantics — it's most commonly
  discussed in the context of `OnCalendar=` timers because that's where "the
  machine was off when this should have run" is the obvious failure mode.
- It does not retroactively run anything that failed — only things that never
  ran because the timer wasn't active.

```ini
[Timer]
OnCalendar=daily
Persistent=true
```

This single line is the difference between a laptop backup timer that silently
never runs (because the laptop is rarely on at 2 AM) and one that reliably runs
once per day, at whatever time the laptop is next turned on after a miss.

### 1.6 systemctl list-timers — inspecting the schedule

```bash
systemctl list-timers --all
```

```
NEXT                        LEFT     LAST                         PASSED  UNIT                      ACTIVATES
Thu 2026-07-11 02:00:00 UTC 8h left  Wed 2026-07-10 02:00:00 UTC  16h ago novatech-backup.timer     novatech-backup.service
```

Read left to right: **NEXT** and **LEFT** tell you when it will next fire and
the countdown; **LAST** and **PASSED** tell you when it last fired and how long
ago; **UNIT** is the timer itself; **ACTIVATES** is the service it will trigger.
`--all` includes inactive timers — without it, a timer that's enabled but not
currently active (e.g. `Persistent=true` hasn't kicked in yet, or it's disabled)
won't show up, which is a common source of "wait, where did my timer go"
confusion.

---

## Part 2 — Hands-On Lab

### Commands used in this demo

| Command | Flag meaning | Purpose |
|---|---|---|
| `systemctl enable --now name.timer` | `--now`: also start immediately | Enable + activate a timer in one step |
| `systemd-analyze calendar "EXPR"` | — | Validate and preview an OnCalendar expression |
| `systemctl list-timers --all` | `--all`: include inactive timers | See every timer's next/last run |
| `systemctl status name.timer` | — | Check whether a timer itself is active |
| `journalctl -u name.service` | `-u`: filter by unit | See the output/history of a timer-triggered service |
| `systemctl start name.service` | — | Manually fire a timer-triggered service right now, for testing |

### Task 1 — Create a basic .service + .timer pair

```bash
sudo tee /etc/systemd/system/novatech-backup.service > /dev/null << 'EOF'
[Unit]
Description=NovaTech nightly backup job

[Service]
Type=oneshot
ExecStart=/bin/bash -c 'echo "Backup ran at $(date)" >> /var/log/novatech-backup.log'
EOF

sudo tee /etc/systemd/system/novatech-backup.timer > /dev/null << 'EOF'
[Unit]
Description=Run NovaTech backup on schedule

[Timer]
OnCalendar=*-*-* 02:00:00
Persistent=true

[Install]
WantedBy=timers.target
EOF

sudo systemctl daemon-reload
# Required after writing new unit files, same rule as Demo M02-04 — systemd hasn't
# read these files yet until you tell it to.

sudo systemctl enable --now novatech-backup.timer
systemctl list-timers --all | grep novatech-backup
```

Notice you enabled and started the **timer**, not the service — the service
stays `inactive (dead)` until the timer fires it or you trigger it manually.

### Task 2 — OnCalendar syntax and systemd-analyze calendar

```bash
systemd-analyze calendar "*-*-* 02:00:00"
# Confirms next elapse and interval — this is your syntax-checker, use it
# before deploying, not after debugging a timer that never fired.

systemd-analyze calendar "Mon..Fri *-*-* 09:00:00"
systemd-analyze calendar "*-*-01 00:00:00"

# Deliberately test a typo:
systemd-analyze calendar "Mon-Fri *-*-* 09:00:00"
# "Mon-Fri" (single dash) is NOT the same as "Mon..Fri" (double dot) — this
# either fails to parse or parses to something unintended, depending on
# version. Always validate; never assume the expression means what it looks
# like it means.
```

### Task 3 — Monotonic timers (OnBootSec/OnUnitActiveSec)

```bash
sudo tee /etc/systemd/system/novatech-cleanup.service > /dev/null << 'EOF'
[Unit]
Description=NovaTech temp file cleanup

[Service]
Type=oneshot
ExecStart=/bin/bash -c 'find /tmp/novatech -mtime +1 -delete; echo "Cleanup ran at $(date)" >> /var/log/novatech-cleanup.log'
EOF

sudo tee /etc/systemd/system/novatech-cleanup.timer > /dev/null << 'EOF'
[Unit]
Description=Periodic NovaTech temp cleanup

[Timer]
OnBootSec=2min
OnUnitActiveSec=15min

[Install]
WantedBy=timers.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now novatech-cleanup.timer
systemctl list-timers --all | grep novatech-cleanup
```

This timer fires 2 minutes after boot, then every 15 minutes after its own last
run — regardless of what time of day it is. Compare its `list-timers` output
against `novatech-backup.timer`'s: the backup timer's NEXT is always the same
clock time; the cleanup timer's NEXT shifts relative to when it last ran.

### Task 4 — Persistent timers — surviving downtime

```bash
sudo systemctl stop novatech-backup.timer
sudo systemctl disable novatech-backup.timer
sleep 5   # standing in for "the machine was off overnight"
sudo systemctl enable --now novatech-backup.timer

journalctl -u novatech-backup.service --since "5 minutes ago"
# With Persistent=true, expect to see a catch-up run logged shortly after
# re-enabling — systemd noticed the scheduled 02:00:00 run was missed while
# the timer was inactive and fired it once immediately.
```

```bash
# Now remove Persistent=true and repeat, to see the contrast:
sudo sed -i '/Persistent=true/d' /etc/systemd/system/novatech-backup.timer
sudo systemctl daemon-reload
sudo systemctl stop novatech-backup.timer
sudo systemctl disable novatech-backup.timer
sleep 5
sudo systemctl enable --now novatech-backup.timer
journalctl -u novatech-backup.service --since "5 minutes ago"
# No catch-up run this time — the timer just waits for the next scheduled
# 02:00:00, silently, with no indication anything was missed.
```

### Task 5 — cron vs systemd-timer side-by-side

```bash
# Equivalent cron entry for the same daily-2AM backup job:
# crontab -e, then add:
# 0 2 * * * /usr/local/bin/novatech-backup.sh

# Where did it run, and how do you know it worked?
crontab -l
grep CRON /var/log/syslog 2>/dev/null || grep CRON /var/log/cron 2>/dev/null
# Compare the discoverability of this against:
journalctl -u novatech-backup.service --since today
systemctl status novatech-backup.timer
```

The cron job's evidence trail depends on syslog/cron log conventions that vary
by distro and are easy to overlook. The timer's evidence trail is exactly where
every other systemd unit's evidence trail is — `journalctl -u`, `systemctl
status` — no special-casing required because you already know how to look.

---

## What You Learned

- ✅ Explained why systemd timers exist alongside cron, and when to choose each
- ✅ Wrote a `.service` + `.timer` unit pair from scratch and enabled it correctly
- ✅ Configured `OnCalendar=` expressions and validated them with `systemd-analyze calendar`
- ✅ Configured `OnBootSec=` and `OnUnitActiveSec=` monotonic timers
- ✅ Configured `Persistent=true` and demonstrated exactly what it catches and what it doesn't
- ✅ Read `systemctl list-timers` output to determine what will run next and why
- ✅ Diagnosed a timer that silently never fires, using `systemd-analyze calendar`

**Key Takeaway:** A `.timer` is only ever half the picture — it schedules, a paired `.service` does the work, and the naming convention connecting them is load-bearing, not decoration. When a timer "does nothing," the fault is almost always in one of two places: a calendar expression that loaded but never resolved to a next elapse time, or a unit that's enabled without being started (or vice versa) — the exact same enable/start independence from Demo M02-04, just applied to a `.timer` instead of a `.service`.

---

## Part 3 — Break-Fix Scenario

### Scenario — NovaTech Ops: two timer incidents

> **Incident report — NovaTech Ops, ticket #4471**
>
> Two separate teams have filed tickets this week. The reporting team says their
> nightly summary email never arrives. The platform team says a log-rotation
> timer "just stopped" after a maintenance window. You're on call. Investigate
> both.

**Incident 1 — the summary email that never arrives**

```bash
$ systemctl list-timers --all | grep novatech-report
                                                                     novatech-report.timer     novatech-report.service
$ systemctl status novatech-report.timer
● novatech-report.timer - NovaTech daily summary report
     Loaded: loaded (/etc/systemd/system/novatech-report.timer; enabled)
     Active: active (waiting)
    Trigger: n/a
$ cat /etc/systemd/system/novatech-report.timer
[Timer]
OnCalendar=*-*-* 08-00-00

[Install]
WantedBy=timers.target
```

Tasks:
1. Explain what `Trigger: n/a` combined with a loaded, enabled, "active (waiting)"
   timer is telling you — is the timer broken, or is something else wrong?
2. `systemd-analyze calendar "*-*-* 08-00-00"` and read the output carefully.
   What's actually wrong with this expression, and why did it not throw a loud
   error when the unit file was written?
3. Fix the unit file and confirm with `systemd-analyze calendar` and
   `systemctl list-timers` that NEXT now shows a real, near-future time.

**Incident 2 — the log-rotation timer that "just stopped"**

```bash
$ systemctl status novatech-logrotate.timer
● novatech-logrotate.timer - NovaTech log rotation
     Loaded: loaded (/etc/systemd/system/novatech-logrotate.timer; disabled)
     Active: inactive (dead)
$ systemctl list-timers --all | grep novatech-logrotate
                                                                     novatech-logrotate.timer  novatech-logrotate.service
$ journalctl -u novatech-logrotate.service --since "1 week ago" | tail -5
-- No entries --
```

Tasks:
1. `disabled` + `inactive` — what does that combination mean happened during
   the maintenance window, in terms of what a systemd unit file edit requires?
   (Hint: connect this back to Demo M02-04's four-combination `enable`/`start` model.)
2. The unit file still exists on disk and is syntactically fine. Re-enable it
   correctly — not just `systemctl start`, which would only fix it until the
   next reboot.
3. This timer has `Persistent=true`. After re-enabling it, is a catch-up run for
   the missed week of log rotations guaranteed? Check `journalctl` after
   re-enabling and explain what you actually observe, referencing §1.5.

<details>
<summary>Answers</summary>

**Incident 1:** The timer is loaded and enabled correctly — the problem is the
`OnCalendar=` expression itself. `08-00-00` uses hyphens where the syntax
requires colons (`08:00:00`) for a time-of-day. systemd's calendar parser is
lenient about malformed expressions in some versions — rather than failing unit
load outright, it can fail to compute a next elapse time, which is exactly what
`Trigger: n/a` means: the timer is structurally valid and enabled, but systemd
has nothing scheduled because it couldn't resolve when to fire. Fix:
`OnCalendar=*-*-* 08:00:00`.

**Incident 2:** `disabled` + `inactive` means the timer's symlink in
`timers.target.wants/` was removed (or never (re)created) and the timer isn't
currently running either — most likely a maintenance script or manual cleanup
ran `systemctl disable` (or deleted/recreated the unit file, which requires
`daemon-reload` + re-`enable`) without following up with `enable --now`. Fix:
`sudo systemctl daemon-reload && sudo systemctl enable --now
novatech-logrotate.timer`. On `Persistent=true`: it's not unconditionally
guaranteed — it catches up on *scheduled occurrences that were missed while
the timer was active but the system was off*, and also fires once if the timer
itself was inactive/disabled across a scheduled time (as covered in §1.5) — but
you should still verify, not assume: `journalctl -u novatech-logrotate.service
--since "5 minutes ago"` to confirm the catch-up run actually happened.

</details>

---

## Interview Prep

**Q1. Why would a production team choose a systemd timer over a cron job for a critical nightly job?**

The main reasons are observability and recoverability, not raw scheduling power
— cron can express the same schedules. A timer-triggered service logs to
`journalctl` in the same place as every other service, gets the same
`systemctl status`/failure-state visibility, can depend on other units via
`After=`/`Requires=`, and — critically — `Persistent=true` gives automatic
catch-up if the run was missed, which cron has no native equivalent for. For a
job where "did it actually run last night" needs a fast, reliable answer, a
timer is the safer default.

---

**Q2. You check `systemctl list-timers` and a timer you expect to see isn't listed. What are the first two things you check, and why in that order?**

First, re-run with `--all` — by default `list-timers` only shows active
timers, so a disabled or currently-inactive-but-otherwise-fine timer simply
won't appear without it; this is the most common false alarm. Second, if it
still doesn't show up, check `systemctl status name.timer` directly to see
whether the unit even loaded — a missing or malformed unit file (or one never
`daemon-reload`'d) won't show up in `list-timers` at all because systemd
doesn't know it exists yet.

---

**Q3. What exactly does `Persistent=true` guarantee, and what's a scenario where someone would wrongly assume it covers something it doesn't?**

It guarantees a single catch-up run for a scheduled occurrence that was missed
because the timer was inactive (system off, timer disabled, etc.) at the
scheduled time. A common wrong assumption: expecting one catch-up run *per*
missed occurrence — e.g. thinking a laptop off for 5 days with a daily timer
will fire 5 times when it comes back online. It fires once, then resumes the
normal schedule. Another wrong assumption: expecting it to retry a run that
executed but *failed* — Persistent only catches runs that never happened at
all, not failed runs, which need `Restart=`/`OnFailure=` handling instead.

---

**Q4. Given a timer with `OnCalendar=daily` and one with `OnBootSec=15min` `OnUnitActiveSec=1h`, which would you pick for a job that must run once specifically between 2–3 AM, and which for a health check that should run roughly hourly regardless of clock time?**

The `OnCalendar=` style (specifically something like `OnCalendar=*-*-*
02:30:00`) for the 2–3 AM job — only calendar-based scheduling ties execution
to wall-clock time. The monotonic `OnBootSec=`/`OnUnitActiveSec=` pair for the
hourly health check — it doesn't care what time it is, only that roughly an
hour has passed since the last run, which is both simpler to reason about and
self-correcting if the machine reboots.

---

## Key Takeaways

1. **A `.timer` unit schedules; it doesn't do the work.** The actual job lives in
   a same-named `.service` (almost always `Type=oneshot`) — you enable/start the
   timer, never the service directly, and the timer activates the service on
   schedule.
2. **`OnCalendar=` and `OnBootSec=`/`OnUnitActiveSec=` answer different
   questions.** Calendar syntax answers "at what time of day/week/month" —
   monotonic syntax answers "how long since boot or since the last run,"
   independent of clock time. Picking the wrong style for the job's actual
   requirement is the most common timer design mistake.
3. **Always validate calendar expressions with `systemd-analyze calendar`
   before deploying.** A malformed expression (like `08-00-00` instead of
   `08:00:00`) can load successfully and show `enabled`/`active (waiting)`
   while never actually computing a next run — `Trigger: n/a` is the tell, and
   it's easy to miss without checking.
4. **`Persistent=true` catches one missed occurrence, not all of them, and not
   failed runs.** It's a single catch-up trigger for scheduled time that
   passed while the timer was inactive — not a retry mechanism and not a
   multiplier for multiple missed windows.
5. **`systemctl list-timers` without `--all` only shows active timers.** A
   disabled or currently-inactive timer disappearing from the default output
   is the most common false alarm when a timer "seems to be gone."
6. **Timer-triggered services inherit full systemd observability.** Unlike
   cron, failures, logs, and status all live in the same `journalctl`/
   `systemctl status` places you already check for every other service — this
   is the main practical argument for timers over cron in production, not
   scheduling syntax power.
7. **Re-enabling a timer after a unit file edit follows the same rules as any
   other unit (Demo M02-04).** `daemon-reload` after any file change, and
   `enable --now` (not just `start`) if it needs to survive the next reboot —
   a timer left `disabled` will not come back after a reboot even if you
   manually `start` it now.

---

## Quick Commands Reference

### Timer unit management

| Command | Purpose |
|---|---|
| `systemctl enable --now name.timer` | Enable (survive reboot) and start a timer in one step |
| `systemctl disable --now name.timer` | Stop and remove reboot-persistence |
| `systemctl status name.timer` | Check whether the timer itself is loaded/active |
| `systemctl start name.service` | Manually fire the target service right now (bypasses schedule, useful for testing) |

### Schedule inspection

| Command | Purpose |
|---|---|
| `systemctl list-timers` | Active timers: next/last run times |
| `systemctl list-timers --all` | Include inactive/disabled timers |
| `systemd-analyze calendar "EXPR"` | Validate an OnCalendar expression and preview next elapse |

### OnCalendar syntax quick reference

| Expression | Meaning |
|---|---|
| `daily` | `*-*-* 00:00:00` |
| `weekly` | `Mon *-*-* 00:00:00` |
| `*-*-* HH:MM:SS` | Every day at that time (colons, not hyphens) |
| `Mon..Fri *-*-* HH:MM:SS` | Weekdays only |
| `*-*-01 00:00:00` | First of every month |

---

## Appendix A — Anki Cards

**`m02-05-systemd-timers-anki.csv`:**

```
#deck:Linux Mastery::Module 2 - System Administration::Demo M02-05 - systemd Timers
#separator:Comma
#columns:Front,Back,Tags

"You write a novatech-report.timer that shows enabled and active (waiting) in systemctl status, but systemctl list-timers shows Trigger: n/a for it. What's the most likely cause?","The OnCalendar= expression is malformed in a way systemd's parser doesn't reject outright — it loads fine but can't compute a next elapse time. Verify with systemd-analyze calendar against the exact expression string.","demo08,oncalendar,troubleshooting"

"You create novatech-backup.service and novatech-backup.timer. What single command do you run to activate the timer so it both starts now and survives reboot?","systemctl enable --now novatech-backup.timer — enable the .timer unit, never the .service directly; --now starts it immediately in addition to enabling persistence.","demo08,timer-basics,systemctl"

"A .timer file has no Unit= line under [Timer]. Which service does it activate?","The identically-named .service unit by default — novatech-backup.timer activates novatech-backup.service unless Unit= explicitly points elsewhere.","demo08,timer-basics,naming-convention"

"What's the difference between OnCalendar=*-*-* 02:00:00 and OnUnitActiveSec=1h in terms of what triggers the next run?","OnCalendar is wall-clock time — always 2:00 AM regardless of when the timer last ran. OnUnitActiveSec is relative to the timer's own last activation — always 1 hour after the previous run, whatever clock time that lands on.","demo08,oncalendar,monotonic"

"A laptop with a daily 2 AM backup timer (Persistent=true) is powered off from Monday night through Thursday morning. How many catch-up backup runs happen when it's turned back on Thursday?","One — Persistent=true fires a single catch-up run for the most recent missed occurrence, not one run per missed day.","demo08,persistent,common-mistake"

"Persistent=true is set on a timer. Last night's scheduled run executed but failed with a non-zero exit code. Does Persistent=true retry it?","No — Persistent only catches runs that never happened because the timer was inactive at the scheduled time. It does not retry runs that executed and failed; that's a job for Restart=/OnFailure= directives on the service unit instead.","demo08,persistent,common-mistake"

"You run systemctl list-timers and don't see a timer you know exists. What's the first thing to check?","Re-run with --all — by default list-timers only shows currently active timers; a disabled or inactive timer won't appear without the flag.","demo08,list-timers,troubleshooting"

"Why is journalctl -u name.service generally considered an advantage of systemd timers over cron for critical jobs?","Timer-triggered services log to the same journalctl location and expose the same systemctl status/failure-state visibility as every other systemd-managed service — no separate log-file or syslog convention to remember, unlike cron.","demo08,cron-comparison,observability"

"You edit an existing novatech-cleanup.timer file to change OnUnitActiveSec= from 15min to 30min. What must you run before the change takes effect, consistent with the Demo M02-04 unit-file-edit rule?","systemctl daemon-reload — systemd reads unit files into memory; edits on disk have no effect until daemon-reload tells it to re-read them, same rule as any other unit type.","demo08,daemon-reload,cross-demo"

"A timer shows disabled and inactive (dead) after a maintenance window. What does re-running only systemctl start name.timer fail to fix?","It starts the timer for the current boot, but disabled means no symlink exists in timers.target.wants/ — after the next reboot the timer will not come back. enable --now (not just start) is required to fix both the immediate and persistent state.","demo08,enable-vs-start,cross-demo"

"What is the correct Type= for a oneshot service triggered by a timer, and why does that choice matter for how systemd reports success/failure in list-timers output?","Type=oneshot — the service is expected to run once and exit; systemd uses its exit code to determine whether that run succeeded or failed, which is what LAST/PASSED reflects in list-timers and status output.","demo08,unit-anatomy,oneshot"
```

---

## Appendix B — Quiz

**`m02-05-systemd-timers-quiz.md`:**

````markdown
# Quiz — Demo M02-05: systemd Timers

> One correct answer per question unless stated otherwise.
> Target: 80% or above before moving to Demo M02-07.

---

**Q1.** `systemctl list-timers --all` shows columns `NEXT`, `LEFT`, `LAST`,
`PASSED`, `UNIT`, `ACTIVATES`. Which pair tells you when a timer will next
fire and how long until then, and which pair tells you about its most recent
run?

- A) `NEXT`/`LEFT` = next fire time and countdown; `LAST`/`PASSED` = most
  recent run time and elapsed time since
- B) `LAST`/`PASSED` = next fire time and countdown; `NEXT`/`LEFT` = most
  recent run and elapsed time — the columns are reversed from what the names suggest
- C) All four columns describe the next run only, from different angles
- D) `UNIT`/`ACTIVATES` describe timing; `NEXT`/`LAST` describe which units are involved

<details>
<summary>Answer</summary>

**A** — Read left to right exactly as named: `NEXT`/`LEFT` for the upcoming
fire, `LAST`/`PASSED` for the most recent one. `UNIT` is the timer itself,
`ACTIVATES` is the service it triggers.

</details>

---

**Q2.** Beyond logging to `journalctl`, what's another concrete capability a
timer-triggered service gets that a plain cron job does not, per the
cron-vs-timer comparison?

- A) Timer-triggered services can use systemd's resource controls
  (`MemoryMax=`, `CPUQuota=`, etc.) and declare dependencies on other units
  via `After=`/`Requires=` — cron has neither
- B) Timer-triggered services run faster than cron jobs at the kernel level
- C) cron jobs cannot be scheduled more often than once per minute, but this
  is the only difference
- D) Timer-triggered services automatically retry on any failure; cron jobs cannot

<details>
<summary>Answer</summary>

**A** — Resource control and dependency awareness are real, distinct
advantages beyond just observability — a timer-triggered service can be
capped on memory/CPU and ordered relative to other units the way any other
systemd unit can; cron has no equivalent mechanism for either.

Trap: D overstates automatic behavior — retry-on-failure still requires an
explicit `Restart=`/`OnFailure=` directive, it isn't automatic just because
it's a timer.

</details>

---

**Q3.** You create `cleanup-job.timer` intending it to activate
`maintenance.service` — a different base name. What must you add to the
timer file, and what happens if you forget?

- A) Nothing needed — systemd will ask interactively which service to bind to
- B) Add `Unit=maintenance.service` under `[Timer]`; without it, systemd
  looks for `cleanup-job.service` by default (the identically-named
  convention) and silently won't find or trigger `maintenance.service` at all
- C) Add `Requires=maintenance.service` under `[Unit]` instead — `Unit=` is
  only for `.socket` activation
- D) Rename `maintenance.service` to `cleanup-job.service` — differing names
  between a timer and its target are never supported

<details>
<summary>Answer</summary>

**B** — The naming convention is a default, not a requirement, but skipping
the explicit `Unit=` when names differ means the timer silently targets
whatever `cleanup-job.service` would be (likely nonexistent), never
`maintenance.service`.

</details>

---

**Q4.** Task 2 in this demo has you run `systemd-analyze calendar` on
several expressions *before* writing them into a real timer unit. Why is
this the recommended habit, rather than deploying first and checking
`list-timers` afterward if something seems off?

- A) `systemd-analyze calendar` is required by systemd — a timer literally
  cannot be created without running it first
- B) A malformed expression can load and enable successfully while never
  computing a next elapse time (`Trigger: n/a`) — nothing about the loading
  process itself flags the problem, so checking after deployment means
  discovering the failure only when the job silently never ran
- C) It's purely a style preference with no functional difference either way
- D) `systemd-analyze calendar` fixes typos automatically once run

<details>
<summary>Answer</summary>

**B** — This is exactly what happens in this demo's own Break-Fix Incident 1:
`08-00-00` loads fine, shows `enabled`/`active (waiting)`, and gives no error
— the only way to catch it before it silently fails to run is validating the
expression up front.

</details>

---

**Q5.** In `OnCalendar=` syntax, what is the difference between
`Mon..Fri *-*-* 09:00:00` and `Mon-Fri *-*-* 09:00:00` (single hyphen)?

- A) They are equivalent — both mean "Monday through Friday"
- B) `Mon..Fri` (double dot) is the correct range syntax; `Mon-Fri` (single
  hyphen) is not valid range syntax and either fails to parse or resolves to
  something other than the intended weekday range, depending on version
- C) `Mon-Fri` is the modern replacement syntax for the older `Mon..Fri` form
- D) Both are invalid; only comma-separated `Mon,Tue,Wed,Thu,Fri` works

<details>
<summary>Answer</summary>

**B** — The double-dot is systemd's actual range operator. A single hyphen is
not the same syntax and does not reliably mean the same thing — exactly why
Task 2 has you deliberately test this typo with `systemd-analyze calendar`
rather than assume it "looks close enough."

</details>

---

**Q6.** You need a job to run roughly every hour, regardless of what time of
day it happens to land on, and to also run shortly after boot. Which timer
configuration fits best?

- A) `OnCalendar=hourly`
- B) `OnBootSec=5min` combined with `OnUnitActiveSec=1h`
- C) `OnCalendar=*-*-* *:00:00` only
- D) `Persistent=true` alone, with no other directive

<details>
<summary>Answer</summary>

**B** — Monotonic timers (`OnBootSec=` + `OnUnitActiveSec=`) are the right
tool when the requirement is "roughly every N, including shortly after boot,"
independent of clock time. A and C both tie execution to wall-clock hour
boundaries, which technically satisfies "roughly hourly" but not the "shortly
after boot" requirement on its own. D is not a scheduling directive at all —
it only affects catch-up behavior for an existing schedule.

</details>

---

**Q7.** `Persistent=true` is set on a daily timer. What does systemd actually
compare to decide whether a catch-up run is needed when the timer becomes
active again?

- A) It checks whether the system clock has changed since last boot
- B) It compares the last stored trigger time (kept in
  `/var/lib/systemd/timers/`) against what the schedule says should have
  happened since — if a scheduled run was missed, it fires once to catch up
- C) It checks the exit code of the last run and retries if non-zero
- D) It asks the administrator interactively whether to run a catch-up job

<details>
<summary>Answer</summary>

**B** — This is the actual mechanism, not just the observed behavior:
systemd persists the last trigger time to disk specifically so it can make
this comparison across reboots or periods where the timer was inactive.

Trap: C describes retry-on-failure, which `Persistent=true` explicitly does
NOT do — that's a separate concern for `Restart=`/`OnFailure=`.

</details>

---

**Q8.** The cron-vs-timer comparison table lists cron's scheduling precision
as minute-level, versus systemd timers' sub-second precision (configurable
via `AccuracySec=`). When does this precision difference actually matter in
practice?

- A) Never — it's a purely theoretical distinction with no real use case
- B) For typical daily/hourly maintenance jobs it rarely matters, but it
  becomes relevant when multiple services need tightly coordinated timing
  relative to each other, which minute-level cron scheduling can't express
- C) It matters for every single timer, since systemd always fires at the
  exact configured moment regardless of `AccuracySec=`
- D) Sub-second precision is only available on Rocky Linux, not Ubuntu

<details>
<summary>Answer</summary>

**B** — For most periodic jobs (backups, cleanup, reports), minute-level
precision is more than adequate — the practical value of sub-second precision
shows up in scenarios needing tight coordination between multiple scheduled
units, not routine daily jobs.

Trap: C ignores that `AccuracySec=` is a deliberate configurable window, not
a guarantee of exact-instant firing — systemd batches timer wakeups for
power efficiency by default.

</details>

---

| Score | Action |
|---|---|
| 8/8 | Import Anki cards, move to Demo M02-07 |
| 6-7/8 | Review the wrong answers, then proceed |
| 4-5/8 | Re-read the relevant sections, retry those questions |
| Below 4/8 | Re-read the full demo and redo the walkthrough before proceeding |
````