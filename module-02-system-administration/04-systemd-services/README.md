# Demo 07 — systemd and Service Management

## Overview

Every long-running service on a modern Linux server — nginx,
sshd, Docker, your own custom application — is managed by
**systemd**, the init system that starts as PID 1 and supervises
everything else for the life of the machine. Demo 06 showed that a
service account's process is just a normal process once it's
running; this demo covers the mechanism that actually starts it,
keeps it running, restarts it on failure, and controls whether it
survives a reboot.

This demo covers `systemctl`, writing your own unit file (using
the exact service account pattern from Demo 05), the enable-vs-start
distinction that catches almost everyone at some point, dependency
ordering, and reading service failures through `journalctl` —
including the Ubuntu-vs-Rocky9 logging difference first introduced
in Demo 01.

---

## Demo Objectives

By the end of this demo you will be able to:

1. Explain what systemd is, why it replaced SysVinit, and what a
   unit is
2. Use `systemctl` to start, stop, restart, reload, enable, and
   disable a service, and check its status
3. Explain the enable-vs-start distinction and all four
   combinations of enabled/started state
4. Write a correct `.service` unit file, including running it as a
   dedicated service account (Demo 05)
5. Configure automatic restart on failure with `Restart=`
6. Explain unit dependency ordering with `After=`, `Requires=`,
   and `Wants=`
7. Use `journalctl` to investigate a failed service, on both
   Ubuntu and Rocky9
8. Diagnose and fix a broken unit file from its failure state

---

## Directory Structure

```
07-systemd-services/
├── README.md                          # this file
├── 07-systemd-services-anki.csv       # Anki flashcard deck
└── 07-systemd-services-quiz.md        # standalone quiz
```

---

## Environments

| Task | Ubuntu 24.04 | Rocky Linux 9.8 | EC2 Amazon Linux 2023 |
|---|---|---|---|
| All lab tasks | ✅ | ✅ | Not required |
| `journalctl` log source | systemd-journald (same as Rocky9) | systemd-journald | Same as Rocky9 |
| Legacy syslog text file | `/var/log/syslog` also present | Not present by default | Not present by default |

Both distros use `journalctl`/journald identically for systemd
unit logs — the Demo 01 syslog-vs-journal difference is about
*other* system logs, not about how you read service logs in this
demo. Covered explicitly in section 1.7.

---

## Recall Check

> Answer from memory before reading further. From Demo 06.

1. `ps aux` shows a process with `STAT` = `Z`. Will `kill -9` fix
   it? What's the actual fix?

2. A background job started with `command &` dies the moment an
   SSH session disconnects. Why, and what's the fix?

3. A service account was created with `/usr/sbin/nologin` per
   Demo 05. Does this prevent its running process from appearing
   in `ps` or being managed normally?

<details>
<summary>Answers</summary>

1. No — a zombie has already finished executing; there's nothing
   left to signal. The fix targets the parent process: make it
   call `wait()`, or kill/restart it so the kernel re-parents the
   zombie to PID 1, which reaps it.

2. The background job is still a child of the shell. When the
   shell exits (including on SSH disconnect), the kernel sends
   `SIGHUP` to its children, killing the job unless protected.
   Fix: `nohup command &` from the start, or `disown -h %1` for
   an already-running job.

3. No — `nologin` only restricts interactive login. The process
   itself, once started, is a completely normal process: visible
   in `ps`/`top`, killable, reniceable, exactly like any other.

</details>

---

## Part 1 — Concepts

### 1.1 What systemd is, and why it replaced SysVinit

**systemd** is the init system on virtually every modern Linux
distribution, including both Ubuntu and Rocky Linux. It is the
very first process the kernel starts at boot — **PID 1**, the same
PID 1 introduced in Demo 06's process tree — and it supervises
every other process on the system for the machine's entire uptime.

**What came before — SysVinit, and why it was replaced:**

```
SysVinit (older)          systemd (current)
Sequential startup        Parallel startup — services that don't
— each service waits      depend on each other start simultaneously,
for the previous one      significantly faster boot times
to finish before the
next one starts

Shell scripts             Declarative unit files — a structured
(/etc/init.d/*) define    config format (.service, .socket, .timer)
how to start/stop a       describing WHAT to run and how, not a
service                   script of HOW to do it step by step

No built-in dependency    Explicit dependency declarations
tracking between          (After=, Requires=, Wants= — section 1.6)
services

No built-in automatic     Built-in Restart= directive — systemd
restart on crash          itself can restart a crashed service
                          (section 1.5), no external watchdog needed
```

The practical result: faster boots, services that declare what
they actually depend on instead of relying on script ordering by
convention, and automatic recovery from crashes without needing a
separate monitoring tool just to restart a dead process.

**Units — the basic concept:**

A **unit** is anything systemd manages, described by a single
configuration file. The type is determined by the file extension:

```
.service    A long-running process or one-off command — the
            primary focus of this demo (nginx.service, sshd.service)
.socket     A network or IPC socket that can start a service on
            first connection (socket activation) — not covered
            in this series per the series scope
.timer      A scheduled trigger, similar to cron — briefly
            introduced in section 1.8; full coverage in Demo 08
.target     A grouping of units, roughly equivalent to SysVinit's
            "runlevels" (e.g. multi-user.target ≈ normal boot state)
```

This demo focuses on `.service` units — everything else is
mentioned only where it directly informs how services behave.

### 1.2 systemctl — the core command

**`systemctl`** is how you interact with systemd — starting,
stopping, inspecting, and configuring units.

```bash
sudo systemctl start nginx       # start now (does NOT affect boot behavior)
sudo systemctl stop nginx        # stop now (does NOT affect boot behavior)
sudo systemctl restart nginx     # stop then start — full restart
sudo systemctl reload nginx      # ask the service to reload its config
                                 # WITHOUT a full restart — only works if
                                 # the service supports it (nginx does;
                                 # not every service implements reload)

sudo systemctl enable nginx      # configure to start on boot (does NOT
                                 # start it now)
sudo systemctl disable nginx     # remove from boot startup (does NOT
                                 # stop it now if it's currently running)

systemctl status nginx           # detailed current state — the command
                                 # you'll use more than any other in
                                 # this demo
systemctl is-active nginx        # prints "active" or "inactive" — useful
                                 # in scripts (exit code 0 = active)
systemctl is-enabled nginx       # prints "enabled" or "disabled"

systemctl list-units --type=service           # all loaded service units
systemctl list-units --type=service --state=running   # only running ones
systemctl --failed                            # units that failed to start
```

**Reading `systemctl status` output:**

```bash
systemctl status nginx
# ● nginx.service - A high performance web server
#      Loaded: loaded (/usr/lib/systemd/system/nginx.service; enabled)
#      Active: active (running) since Mon 2026-06-29 09:00:12 UTC; 2h ago
#    Main PID: 1823 (nginx)
#       Tasks: 3 (limit: 4683)
#      Memory: 4.2M
#         CPU: 142ms
#      CGroup: /system.slice/nginx.service
#              ├─1823 "nginx: master process /usr/sbin/nginx"
#              ├─1824 "nginx: worker process"
#              └─1825 "nginx: worker process"
```

```
●         A colored dot — green = active/running, red = failed,
          white = inactive/stopped — visible in a real terminal

Loaded    Confirms the unit file was found and parsed successfully,
          and shows whether it's enabled

Active    The current runtime state — this is what start/stop affects
          "active (running)" = a process is currently running
          "inactive (dead)" = not running
          "failed" = it tried to start/run and failed

Main PID  The primary process systemd is tracking for this unit —
          this connects directly to Demo 06's process concepts;
          everything under CGroup is the process tree systemd
          is supervising for this one unit
```

**Why check both `Loaded` and `Active` rather than assuming one
implies the other:** `Loaded` reflects the unit file's enabled/
disabled state — the boot-time setting from section 1.3. `Active`
reflects whether it's running right now. Reading only one line of
`systemctl status` and assuming it tells you both is exactly how
the "Disabled + Started" incident in this demo's Break-Fix scenario
goes unnoticed until the next reboot.

### 1.3 Enable vs start — the distinction that catches everyone

This is the single most common point of confusion when learning
systemd. **Enable** and **start** are two completely independent
settings:

```
enable / disable    Controls whether the unit starts AUTOMATICALLY
                    on the NEXT BOOT. Has zero effect on whether
                    it's running right now.

start / stop        Controls whether the unit is running RIGHT NOW.
                    Has zero effect on what happens at the next boot.
```

**All four real combinations, and what each one means in practice:**

```
Enabled + Started     Running now, AND will start automatically on
                      reboot. The normal state for a production
                      service you want always available.

Enabled + Stopped      Will start automatically on the NEXT reboot,
                      but is NOT running right now. You might see
                      this right after enabling a service you
                      haven't started yet, or after manually
                      stopping a service for maintenance — it will
                      come back at next boot regardless.

Disabled + Started     Running right now, but will NOT start again
                      after a reboot. This is exactly Incident 1
                      in this demo's break-fix — a service started
                      manually for testing, never enabled, silently
                      missing after the next reboot.

Disabled + Stopped     Not running, and will not start on reboot.
                      Fully off.
```

```bash
# Check both states independently — never assume one implies the other
systemctl is-active nginx
systemctl is-enabled nginx

# A combined view
systemctl status nginx | grep -E "Loaded|Active"
```

### 1.4 Writing a unit file

Unit files live in `/etc/systemd/system/` (for custom/locally-created
units — the correct location, analogous to `/etc/sudoers.d/` from
Demo 05: a dedicated, version-controllable location rather than
mixing with distro-provided units in `/usr/lib/systemd/system/`).

**A minimal, correct unit file:**

```ini
# /etc/systemd/system/novatech-app.service

[Unit]
Description=NovaTech sample application
After=network.target

[Service]
Type=simple
User=novatech-svc
Group=novatech-svc
ExecStart=/opt/novatech-app/bin/app
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

**Section by section:**

```
[Unit]
Description=    Human-readable name shown in systemctl status
After=          Ordering only — start AFTER this target/unit, but
                does not force it to be present (section 1.6)

[Service]
Type=simple     The most common type — ExecStart is the actual
                process; systemd considers it "started" the moment
                the process launches (no forking, no special
                readiness signal expected)
Type=forking    For older daemons that fork and exit the parent —
                less common with modern software
Type=oneshot    For a command that runs once and exits — combined
                with a .timer (section 1.8) for scheduled tasks

User=           Run as this user — THIS is the actual mechanism
Group=          that makes Demo 05's service account pattern work.
                Creating a /usr/sbin/nologin account accomplishes
                nothing on its own until a unit file's User=
                directive tells systemd to start the process as
                that account.

ExecStart=      The exact command to run — must be an absolute
                path, no shell expansion, no $PATH lookup

Restart=        on-failure (restart only on non-zero exit/crash),
                always (restart on any exit, even clean), no
                (default — never auto-restart)
RestartSec=     Delay before restarting — avoids a tight restart
                loop hammering a resource that's still recovering

[Install]
WantedBy=       Which target this unit becomes part of when enabled
                — multi-user.target is the standard target for a
                normal running system (roughly: full multi-user mode
                with networking, the typical state for a server).
                This directive is what `enable` actually acts on —
                enabling a unit creates a symlink into this target's
                dependency directory; without an [Install] section,
                a unit has nothing for `enable` to wire up at all.
```

**After creating or editing a unit file:**

```bash
sudo systemctl daemon-reload
# REQUIRED after any change to a unit file. systemd caches unit
# definitions in memory — daemon-reload tells it to re-read the
# files from disk. Forgetting this step is one of the most common
# "I edited the file but nothing changed" mistakes.

sudo systemctl enable --now novatech-app
# --now is a shortcut: enable AND start in one command
```

### 1.5 Restart= and automatic recovery

`Restart=` is what gives systemd its automatic-recovery
capability — something SysVinit had no equivalent for without a
separate, additional tool.

```ini
Restart=on-failure   # restart only if the process exits with a
                     # non-zero code, is killed by a signal, or times
                     # out — the most common production setting
Restart=always       # restart on ANY exit, even a clean one — useful
                     # for a process that should genuinely never stop
Restart=no           # default — never restart automatically
```

**`on-failure` vs `always` — choosing correctly matters:** a
service that calls `systemctl stop` internally as part of a
planned shutdown sequence should use `on-failure`, not `always` —
otherwise systemd fights the intentional stop and restarts it
anyway. `always` is correct only for a process that should never
exit under any circumstance, including ones it triggered itself.

```ini
RestartSec=5         # wait 5 seconds before restarting
StartLimitBurst=5    # if it crashes 5 times within...
StartLimitIntervalSec=60   # ...60 seconds, STOP trying — prevents
                     # an infinite crash-restart loop from consuming
                     # all CPU/log space on a genuinely broken service
```

**Why `RestartSec` and the start-limit settings both matter:**
without a delay, a crashing service can enter a tight restart loop
— crash, restart instantly, crash again — consuming CPU and
flooding logs. The start-limit settings give systemd a circuit
breaker: after too many restarts in too short a window, it gives
up and the unit shows `failed`, rather than retrying forever.

### 1.6 Dependency ordering — After, Requires, Wants

These three directives in `[Unit]` control *when* a unit starts
relative to others — but they answer different questions.

```
After=        ORDERING only. "Start after this unit/target, if it's
              present." Does NOT pull in the other unit as a
              dependency, and does NOT fail this unit if the other
              one is absent or fails. Most common — usually what
              you actually want.

Requires=     HARD dependency. If the required unit fails to start,
              THIS unit fails to start too. Strict and easy to use
              incorrectly — overusing Requires= can cause cascading
              failures across units that don't truly need each other.

Wants=        SOFT dependency. Tries to start the wanted unit, but
              if it fails, THIS unit starts anyway. The much safer
              default for "I'd like this to be running too" without
              creating a hard failure chain.
```

```ini
[Unit]
After=network.target
Wants=network-online.target
```

**Why ordering, not just dependency, matters in practice:** a
service that needs network access should declare `After=
network.target` so systemd doesn't start it before the network
stack is up — but `network.target` being "reached" doesn't
strictly guarantee an IP address is assigned yet (that's what
`network-online.target` more specifically waits for). This is a
real, common source of "my service failed on boot but works fine
if I restart it manually" — the service started before networking
was actually fully ready.

### 1.7 journalctl — reading systemd logs

**`journalctl`** reads the systemd journal — the binary log store
maintained by `systemd-journald` — and is the standard way to read
logs for any systemd-managed service, on **both** Ubuntu and
Rocky9 identically.

**Connecting back to Demo 01's syslog-vs-journal distinction:**
Demo 01 covered that Ubuntu writes general system logs to
`/var/log/syslog` (text, via rsyslog) while Rocky9 relies on the
binary journal. That distinction is about **general system
logging**. For **systemd unit logs specifically** — which is what
this demo cares about — both distros use `journalctl` identically;
Ubuntu's `rsyslog` typically just mirrors a copy of journal content
into `/var/log/syslog` for convenience, it doesn't replace
`journalctl` as the source of truth for a service's own logs.

```bash
journalctl -u nginx              # all logs for this specific unit
journalctl -u nginx -f           # follow live, like tail -f
journalctl -u nginx --since "10 minutes ago"
journalctl -u nginx --since today
journalctl -u nginx -p err       # only error-priority and above
journalctl -u nginx -n 50        # last 50 lines only
journalctl -u nginx --no-pager   # don't page output — useful in scripts

journalctl -xe                   # recent logs with extra explanatory
                                 # detail (-x), jumped to the end (-e)
                                 # — a good first command when something
                                 # just failed and you don't know what unit
```

**Priority levels for `-p`:** `emerg`, `alert`, `crit`, `err`,
`warning`, `notice`, `info`, `debug` — from most to least severe.
`-p err` shows `err` and everything more severe than it (`crit`,
`alert`, `emerg`), not just exact matches.

### 1.8 A brief note on timers

`.timer` units are systemd's equivalent of cron — they trigger
another unit (typically a `.service` with `Type=oneshot`) on a
schedule, with some advantages over cron (built-in logging via
`journalctl`, the ability to depend on other units, more flexible
calendar syntax). They're a distinct unit type from everything
else in this demo and deserve full treatment on their own —
**timers are covered in detail in Demo 08 — systemd Timers**, the
next demo in this module. For now, just recognize a `.timer` file
if you see one alongside a `.service` file of the same name —
that pairing is how scheduled systemd tasks work.

---

## Part 2 — Hands-On Lab

### Commands used in this demo

| Command | Flag meaning | Purpose |
|---|---|---|
| `systemctl start/stop/restart name` | — | Control a unit's running state right now |
| `systemctl reload name` | — | Reload config without a full restart (if supported) |
| `systemctl enable/disable name` | — | Control boot-time startup behavior |
| `systemctl enable --now name` | `--now` = also start immediately | Enable and start in one command |
| `systemctl status name` | — | Detailed current state of a unit |
| `systemctl is-active` / `is-enabled name` | — | Script-friendly state checks |
| `systemctl daemon-reload` | — | Reload unit files from disk after editing |
| `systemctl --failed` | — | List units that failed to start |
| `systemctl cat name` | — | Show the actual unit file contents |
| `systemctl show -p MainPID name` | `-p` = print this property only | Get just a unit's main PID |
| `journalctl -u name` | `-u` = unit | Read logs for a specific unit |
| `journalctl -u name -f` | `-f` = follow | Follow logs live, like `tail -f` |
| `journalctl -u name -p err` | `-p` = priority filter | Show only `err` and more severe |
| `journalctl -u name --since "..."` | `--since` = time filter | Logs from a given point onward |
| `journalctl -xe` | `-x` = extra detail, `-e` = jump to end | Recent logs with explanatory context |

---

### Task 1 — Inspect existing units

**Run on Ubuntu, then Rocky9:**

```bash
# List running services
systemctl list-units --type=service --state=running | head -15

# Pick a real, always-present service and inspect it in detail
systemctl status sshd 2>/dev/null || systemctl status ssh
# Note: the unit is named "ssh" on Ubuntu, "sshd" on Rocky9 —
# another small naming difference between distro families

systemctl is-active sshd 2>/dev/null || systemctl is-active ssh
systemctl is-enabled sshd 2>/dev/null || systemctl is-enabled ssh

# Find the actual unit file on disk
systemctl cat sshd 2>/dev/null || systemctl cat ssh
```

**Paste your `systemctl status` output for ssh/sshd — identify the
Main PID and confirm it matches an entry in `ps -ef` (tying back
to Demo 06).**

---

### Task 2 — Write a custom unit file using the Demo 05 service account

```bash
# Reuse the novatech-svc service account created in Demo 05
# (re-create it if needed, exactly as in Demo 05 Task 7):
sudo useradd -r -s /usr/sbin/nologin -d /opt/novatech-app -c "NovaTech App service account" novatech-svc 2>/dev/null

sudo mkdir -p /opt/novatech-app/bin
sudo tee /opt/novatech-app/bin/app > /dev/null << 'EOF'
#!/bin/bash
while true; do
  echo "$(date): novatech-app heartbeat"
  sleep 10
done
EOF
sudo chmod +x /opt/novatech-app/bin/app
sudo chown -R novatech-svc:novatech-svc /opt/novatech-app

# Create the unit file
sudo tee /etc/systemd/system/novatech-app.service > /dev/null << 'EOF'
[Unit]
Description=NovaTech sample application
After=network.target

[Service]
Type=simple
User=novatech-svc
Group=novatech-svc
ExecStart=/opt/novatech-app/bin/app
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF

# Required after creating/editing any unit file
sudo systemctl daemon-reload
```

**Paste the contents of your unit file exactly as written.**

---

### Task 3 — All four enable/start combinations

```bash
# Start with everything off
sudo systemctl stop novatech-app 2>/dev/null
sudo systemctl disable novatech-app 2>/dev/null

# Combination 1: Disabled + Stopped (fully off)
systemctl is-active novatech-app; systemctl is-enabled novatech-app

# Combination 2: Disabled + Started
sudo systemctl start novatech-app
systemctl is-active novatech-app; systemctl is-enabled novatech-app
# Running now, but will NOT survive a reboot

# Combination 3: Enabled + Stopped
sudo systemctl stop novatech-app
sudo systemctl enable novatech-app
systemctl is-active novatech-app; systemctl is-enabled novatech-app
# Will start on next boot, but is not running right now

# Combination 4: Enabled + Started (the normal production state)
sudo systemctl start novatech-app
systemctl is-active novatech-app; systemctl is-enabled novatech-app
```

**Paste the `is-active`/`is-enabled` output for all four
combinations, in order.**

---

### Task 4 — Restart=on-failure, proven by killing the process

```bash
# Confirm it's running and find its PID
systemctl status novatech-app | grep "Main PID"
MAIN_PID=$(systemctl show -p MainPID novatech-app | cut -d= -f2)
echo "Main PID: $MAIN_PID"

# Kill it directly with SIGKILL (Demo 06 concept) — simulating a crash
sudo kill -9 $MAIN_PID

# Wait past RestartSec, then check again
sleep 7
systemctl status novatech-app | grep -E "Active|Main PID"
# Should show a NEW PID — systemd restarted it automatically
```

**Paste the `Main PID` before and after the `kill -9` — confirm
the PID changed, proving `Restart=on-failure` actually worked.**

---

### Task 5 — Investigate logs with journalctl

```bash
# View this service's full log history
journalctl -u novatech-app --no-pager | tail -20

# Follow live for a few seconds
timeout 15 journalctl -u novatech-app -f

# Check around the time you killed it in Task 4
journalctl -u novatech-app --since "5 minutes ago" --no-pager
# Should show the heartbeat messages, a gap/crash indication, and
# the restart
```

**Paste the `journalctl` output spanning the kill/restart from
Task 4 — identify where the crash and restart appear in the log.**

---

### Task 6 — Fix a broken unit file

```bash
# Deliberately break the unit file with a wrong ExecStart path
sudo cp /etc/systemd/system/novatech-app.service /tmp/novatech-app.service.bak
sudo sed -i 's|/opt/novatech-app/bin/app|/opt/novatech-app/bin/app-WRONG|' /etc/systemd/system/novatech-app.service

sudo systemctl daemon-reload
sudo systemctl restart novatech-app

# Observe the failure
systemctl status novatech-app
# Active: failed (Result: exit-code)

journalctl -u novatech-app -n 20 --no-pager
# Should show something like "No such file or directory" referencing
# the wrong path

# Diagnose, then fix
sudo cp /tmp/novatech-app.service.bak /etc/systemd/system/novatech-app.service
sudo systemctl daemon-reload
sudo systemctl restart novatech-app
systemctl status novatech-app
# Active: active (running) — confirmed fixed
```

**Paste the `systemctl status` failure output and the
`journalctl` line that revealed the actual cause.**

---

## What You Learned

- ✅ Explained what systemd is, why it replaced SysVinit, and what a unit is
- ✅ Used `systemctl` to start, stop, restart, reload, enable, and disable a service, and checked its status
- ✅ Explained the enable-vs-start distinction and all four combinations of enabled/started state
- ✅ Wrote a correct `.service` unit file, running it as the dedicated service account created in Demo 05
- ✅ Configured automatic restart on failure with `Restart=` and `RestartSec=`, and proved it by killing the process directly
- ✅ Explained unit dependency ordering with `After=`, `Requires=`, and `Wants=`
- ✅ Used `journalctl` to investigate a running and a failed service, on both Ubuntu and Rocky9
- ✅ Diagnosed and fixed a broken unit file from its failure state, using `journalctl` to find the actual cause

**Key Takeaway:** systemd is the missing piece between "I wrote a script" and "this runs reliably in production." `User=` is what actually activates Demo 05's service account pattern, `Restart=` is what gives you Demo 06's crash recovery without a separate watchdog, and `journalctl -u` is always your first command when a service fails — not guessing, not grepping random log files.

---

## Part 3 — Break-Fix Scenario

### Scenario — NovaTech Ops: two service management incidents

> *"Incident 1: We started novatech-app manually last week to test
> it, confirmed it was working, and moved on. The server rebooted
> for a kernel update over the weekend. This morning the app is
> not running and nobody touched it."*

> *"Incident 2: A teammate wrote a new unit file for a different
> service. It fails immediately every time they try to start it.
> systemctl status just shows 'failed (Result: exit-code)' with no
> further detail in that view."*

**Investigation — Incident 1:**

```bash
systemctl is-active novatech-app
# inactive

systemctl is-enabled novatech-app
# disabled
```

**Investigation — Incident 2:**

```bash
systemctl status custom-app
# ● custom-app.service - Custom Application
#      Loaded: loaded (/etc/systemd/system/custom-app.service; enabled)
#      Active: failed (Result: exit-code)

journalctl -u custom-app -n 20 --no-pager
# custom-app[3401]: /opt/custom/bin/run.sh: Permission denied
```

**Your tasks:**

1. For Incident 1: what is the root cause, in terms of the four
   enable/start combinations, and what single command fixes it
   going forward without needing to manually start it after every
   future reboot?

2. For Incident 2: `journalctl` shows "Permission denied." Given
   what you know about service accounts (Demo 05) and unit files
   (`User=`, section 1.4), what are the two most likely causes?

3. After fixing Incident 2, what command sequence confirms the fix
   actually worked, not just that the unit file no longer has an
   obvious typo?

<details>
<summary>Resolution — open after attempting</summary>

**Incident 1 — root cause:**

The service was started manually (`systemctl start`) but never
enabled. This is exactly the "Disabled + Started" combination from
section 1.3 — running at the time, but with zero instruction to
systemd about what to do at the next boot. When the server
rebooted, there was nothing telling systemd to start it again.

**Fix:**
```bash
sudo systemctl enable --now novatech-app
```
`--now` both enables it (fixing future reboots) and starts it
immediately (fixing the current outage) in one command. Going
forward, always use `enable --now` together when bringing up a
new production service, rather than `start` alone followed by a
separate `enable` later (or never).

**Incident 2 — "Permission denied," two likely causes:**

1. **The `ExecStart` script itself lacks the execute bit** —
   `chmod +x` was never run on `/opt/custom/bin/run.sh`. This is a
   straightforward Demo 03 permissions issue, just surfacing here
   through systemd instead of a direct shell attempt.

2. **The service account specified in `User=` doesn't have
   permission to read/execute the file or its containing
   directory** — e.g., the unit file says `User=custom-svc` but
   the script's ownership or directory permissions only allow a
   different user. This connects Demo 03 (permissions), Demo 05
   (service accounts), and this demo's `User=` directive directly.

**Diagnosis:**
```bash
ls -la /opt/custom/bin/run.sh
# Check execute bit AND ownership/group against the unit's User=/Group=

sudo -u custom-svc /opt/custom/bin/run.sh
# Try running it manually AS the service account — reproduces the
# exact permission error outside of systemd, confirming which of
# the two causes it actually is
```

**Fix (whichever cause applies):**
```bash
sudo chmod +x /opt/custom/bin/run.sh        # if execute bit missing
sudo chown custom-svc:custom-svc /opt/custom/bin/run.sh   # if ownership wrong
```

**Question 3 — confirming the fix actually worked:**

```bash
sudo systemctl daemon-reload     # only needed if the unit FILE changed,
                                 # not needed for a permissions-only fix,
                                 # but harmless to run either way
sudo systemctl restart custom-app
systemctl status custom-app
# Active: active (running) — not just "no error shown," actually running

systemctl is-active custom-app
# active
# Checking is-active explicitly, rather than just eyeballing status,
# gives an unambiguous pass/fail you could also use in a script
```

</details>

---

## Interview Prep

**Q1. A teammate says "I ran `systemctl enable nginx`, so it's
running now." Is this correct?**

No. `enable` only configures whether a unit starts automatically
on the *next* boot — it has no effect on whether the unit is
running *right now*. To both enable and start immediately,
either run both commands (`systemctl enable nginx` then
`systemctl start nginx`) or use the combined shortcut
`systemctl enable --now nginx`. Checking `systemctl is-active`
explicitly is the only way to know the current running state.

---

**Q2. You write a `.service` unit file with `User=appuser` but the
service fails with a permission error trying to read its own
config file. What's the most likely cause and how do you verify it
quickly?**

Most likely, the files the service needs to read/write are owned
by a different user or lack the necessary permissions for
`appuser` specifically — the same kind of issue covered in Demo
03's permission model, just surfacing through systemd. The fastest
verification is to reproduce it directly: `sudo -u appuser
/path/to/the/command` run manually outside of systemd reproduces
the exact same permission error in a much faster feedback loop than
repeatedly restarting the unit and checking `journalctl` each time.

---

**Q3. Why does `Requires=` carry more risk than `Wants=` when
declaring a dependency between two units?**

`Requires=` is a hard dependency — if the required unit fails to
start, the dependent unit fails too, even if the dependent unit
could otherwise function. `Wants=` is a soft dependency — systemd
tries to start the wanted unit, but proceeds with starting the
dependent unit regardless of whether the wanted one succeeds.
Overusing `Requires=` across many units can create cascading
failures where one unrelated unit failing takes down several
others that didn't strictly need it to be running. `Wants=`
(alongside `After=` for ordering) is the safer default unless a
genuine hard dependency exists.

---

**Q4. A service keeps failing immediately every time it starts.
What is the single first command you should run, and why?**

`journalctl -u servicename -n 20` (or `journalctl -xe` if you
don't yet know which unit is involved). `systemctl status` shows
*that* it failed and the high-level result (`exit-code`,
`signal`, etc.), but `journalctl` shows the actual error output
the process produced — a wrong path, a permission error, a missing
dependency — which is almost always where the real diagnosis
happens. Guessing at the cause from `systemctl status` alone, or
searching unrelated log files, wastes time `journalctl -u` would
have answered immediately.

---

## Key Takeaways

1. **`enable`/`disable` controls boot-time startup; `start`/`stop` controls right-now running state — these are completely independent settings with four real combinations.** "Disabled + Started" — running now but gone after the next reboot — is one of the most common silent production incidents. Always use `enable --now` together when bringing up a new service.

2. **`User=`/`Group=` in a unit file is the actual mechanism that activates a Demo 05 service account.** Creating a `/usr/sbin/nologin` account by itself does nothing — it only becomes useful once a unit file tells systemd to run a process as that account.

3. **`Restart=on-failure` with `RestartSec=` gives automatic crash recovery with no separate watchdog tool needed — SysVinit had no built-in equivalent.** Always pair it with `StartLimitBurst=`/`StartLimitIntervalSec=` to avoid an infinite crash-restart loop on a genuinely broken service.

4. **`After=` is ordering only; `Requires=` is a hard dependency that can cascade failures; `Wants=` is the safer soft dependency for "I'd like this running too."** Default to `After=` + `Wants=` unless a unit genuinely cannot function at all without another one present.

5. **`journalctl -u servicename` is always the first command when a service fails — `systemctl status` tells you THAT it failed, `journalctl` tells you WHY.** Both Ubuntu and Rocky9 use `journalctl` identically for unit logs; Demo 01's syslog-vs-journal distinction is about general system logging, not service-specific logs.

6. **Always run `systemctl daemon-reload` after creating or editing any unit file** — systemd caches unit definitions in memory and won't see file changes until told to re-read them. Forgetting this is one of the most common "I fixed it but nothing changed" mistakes.

7. **`.timer` units are systemd's cron equivalent, paired with a `Type=oneshot` service — full coverage is in Demo 08, immediately following this one.** Recognize the pairing if you see it; that's enough for now.

---

## Quick Commands Reference

### Controlling units

| Command | Purpose |
|---|---|
| `systemctl start name` | Start now (no effect on boot behavior) |
| `systemctl stop name` | Stop now (no effect on boot behavior) |
| `systemctl restart name` | Stop then start |
| `systemctl reload name` | Reload config without full restart (if supported) |
| `systemctl enable name` | Start automatically on next boot (no effect now) |
| `systemctl disable name` | Don't start automatically on boot |
| `systemctl enable --now name` | Enable AND start in one command |

### Checking state

| Command | Purpose |
|---|---|
| `systemctl status name` | Full detailed current state |
| `systemctl is-active name` | "active"/"inactive", script-friendly |
| `systemctl is-enabled name` | "enabled"/"disabled", script-friendly |
| `systemctl --failed` | List all units that failed to start |
| `systemctl list-units --type=service` | List all loaded service units |

### Unit files

| Command | Purpose |
|---|---|
| `systemctl cat name` | Show the actual unit file contents |
| `systemctl daemon-reload` | Reload unit files from disk — required after any edit |
| `systemctl show -p MainPID name` | Get just the main PID of a running unit |

### Logs

| Command | Purpose |
|---|---|
| `journalctl -u name` | All logs for a specific unit |
| `journalctl -u name -f` | Follow live |
| `journalctl -u name -n 50` | Last 50 lines only |
| `journalctl -u name --since "10 minutes ago"` | Time-filtered |
| `journalctl -u name -p err` | Priority-filtered (err and more severe) |
| `journalctl -xe` | Recent logs with extra detail, jumped to the end |

---

## Appendix A — Anki Cards

**`07-systemd-services-anki.csv`:**

```
#deck:Linux Mastery::Module 2 - System Administration::Demo 07 - systemd and Service Management
#separator:Comma
#columns:Front,Back,Tags

"A service was started manually with systemctl start last week. After a server reboot, it's not running. What's the root cause?","The service was started but never enabled - the 'Disabled + Started' combination. enable/disable controls boot-time startup; start/stop controls right-now running state. These are completely independent. Fix: systemctl enable --now servicename - enables for future boots AND starts immediately in one command.","demo07,enable,start,systemctl"

"A unit file specifies User=appuser but the service fails with a permission error. What's the fastest way to verify the actual cause?","Reproduce it directly: sudo -u appuser /path/to/the/command run manually outside systemd. This reproduces the exact same permission error in a much faster feedback loop than repeatedly restarting the unit and checking journalctl each time. The actual fix is usually a Demo 03 permissions issue (missing execute bit, wrong ownership) surfacing through the User= directive.","demo07,permissions,user-directive,troubleshooting"

"What is the difference between Requires= and Wants= when declaring a unit dependency?","Requires= is a HARD dependency - if the required unit fails to start, this unit fails too, which can cascade failures across unrelated units. Wants= is a SOFT dependency - systemd tries to start the wanted unit but proceeds regardless of whether it succeeds. Wants= (paired with After= for ordering) is the safer default unless a unit genuinely cannot function at all without another one present.","demo07,requires,wants,dependencies"

"A service keeps failing immediately on every start attempt. What is the single first command to run, and why?","journalctl -u servicename -n 20 (or journalctl -xe if you don't know the unit yet). systemctl status shows THAT it failed and a high-level result code; journalctl shows the actual error output the process produced - wrong path, permission error, missing dependency - which is almost always where the real diagnosis happens.","demo07,journalctl,troubleshooting"

"You edit an existing unit file's ExecStart path, save it, and restart the service - it still uses the old path. What step was missed?","systemctl daemon-reload. systemd caches unit definitions in memory and does not automatically notice file changes on disk. daemon-reload tells it to re-read unit files. Required after ANY change to a unit file before the change takes effect, even before a restart.","demo07,daemon-reload,unit-files"

"What does Restart=on-failure with RestartSec=5 actually do, and why pair it with StartLimitBurst=?","Restart=on-failure tells systemd to automatically restart the service if it exits with a non-zero code, crashes, or is killed by a signal. RestartSec=5 waits 5 seconds before each restart attempt, avoiding a tight restart loop. StartLimitBurst= (with StartLimitIntervalSec=) acts as a circuit breaker - after too many restarts in too short a window, systemd stops trying and marks the unit failed, rather than retrying forever on a genuinely broken service.","demo07,restart,resilience"

"A unit file has User=novatech-svc and ExecStart=/opt/novatech-app/bin/app. What two things from Demo 05 does this directive actually connect to systemd?","User= is the mechanism that ACTIVATES a Demo 05 service account - creating a /usr/sbin/nologin account by itself accomplishes nothing until a unit file's User= directive tells systemd to actually run a process as that account. Without this connection, the carefully-configured service account would never be used to run anything.","demo07,user-directive,service-accounts,demo05"

"Why does journalctl work identically on both Ubuntu and Rocky9 for systemd unit logs, even though Demo 01 described different logging conventions for the two distros?","Demo 01's syslog-vs-journal distinction is about GENERAL system logging - Ubuntu mirrors logs into /var/log/syslog (text, via rsyslog) while Rocky9 relies solely on the binary journal. For SYSTEMD UNIT LOGS SPECIFICALLY, both distros use journalctl/systemd-journald identically - rsyslog on Ubuntu just mirrors a copy of journal content for convenience, it doesn't replace journalctl as the source of truth for a service's own logs.","demo07,journalctl,demo01,distro-differences"
```

---

## Appendix B — Quiz

**`07-systemd-services-quiz.md`:**

````markdown
# Quiz — Demo 07: systemd and Service Management

> One correct answer per question unless stated otherwise.
> Target: 80% or above before moving to Demo 08.

---

**Q1.** What is the actual practical advantage of systemd's
declarative unit files and built-in dependency tracking over
SysVinit's `/etc/init.d/` shell scripts?

- A) Unit files are shorter to type than shell scripts
- B) Services that don't depend on each other can start in
  parallel instead of sequentially, and dependencies are declared
  explicitly rather than relying on script ordering by convention
- C) Unit files can only be edited by root, improving security
- D) systemd doesn't actually offer any functional advantage —
  it's purely a stylistic change from scripts to config files

<details>
<summary>Answer</summary>

**B** — Parallel startup (faster boots) and explicit dependency
declarations are the real functional gains, along with systemd's
built-in `Restart=` — none of which SysVinit provided without
extra tooling.

</details>

---

**Q2.** A unit file's `[Service]` section has `Type=oneshot`
instead of the more common `Type=simple`. What kind of process is
`Type=oneshot` actually meant for?

- A) A long-running daemon that forks and detaches from its parent
- B) A command that runs once, completes, and exits — commonly
  paired with a `.timer` unit for scheduled execution
- C) A service that should never be allowed to restart
- D) A unit with no `ExecStart=` directive at all

<details>
<summary>Answer</summary>

**B** — `Type=simple` is for long-running processes where the
launched process itself is what systemd tracks as "the service."
`Type=oneshot` is for a command that does its work and exits —
exactly what a scheduled `.timer`-triggered task needs (full
timer coverage in Demo 08).

</details>

---

**Q3.** A service is configured with only `After=network.target`
and needs to reach an external API on startup. It sometimes fails
on boot but works fine when restarted manually a minute later.
What's the most likely explanation?

- A) `After=network.target` guarantees an IP address is assigned,
  so this must be an unrelated DNS issue
- B) `network.target` being "reached" doesn't strictly guarantee
  an IP address is actually assigned yet — `network-online.target`
  specifically waits for that; the service is starting before
  networking is truly ready
- C) `After=` directives are ignored on the first boot after
  installation
- D) The service needs `Type=oneshot` instead of `Type=simple` to
  wait for the network

<details>
<summary>Answer</summary>

**B** — This is a well-known real-world gotcha: `network.target`
only orders startup relative to the network stack existing, not
relative to it having a working connection. Add `Wants=
network-online.target` and `After=network-online.target` for
services that genuinely need connectivity at startup.

</details>

---

**Q4.** What is the difference between `Restart=on-failure` and
`Restart=always`, and which is correct for a process that
sometimes exits cleanly as part of its own planned shutdown logic?

- A) They behave identically in practice
- B) `on-failure` only restarts on a non-zero exit/crash/signal;
  `always` restarts on any exit including a clean one. A process
  with its own planned clean-exit logic should use `on-failure`,
  or `always` will fight the intentional stop and restart it anyway
- C) `always` is strictly safer and should be used everywhere
- D) `on-failure` requires `StartLimitBurst=` to function; `always`
  does not

<details>
<summary>Answer</summary>

**B** — Choosing `always` for a process that sometimes exits on
purpose creates a service that can never be cleanly stopped from
inside itself — systemd will just bring it back. `on-failure` is
the correct default for anything with its own legitimate exit paths.

</details>

---

**Q5.** A unit file's `[Install]` section contains
`WantedBy=multi-user.target`. What does this section and
directive actually control?

- A) Which user the service runs as
- B) Which target's dependency tree this unit joins when enabled —
  `systemctl enable` creates a symlink into that target based on
  this directive; without an `[Install]` section, there's nothing
  for `enable` to wire up at all
- C) The order in which this unit starts relative to other units
- D) Whether the unit restarts automatically on failure

<details>
<summary>Answer</summary>

**B** — `[Install]`/`WantedBy=` is specifically what `systemctl
enable` acts on. A unit with no `[Install]` section can still be
started manually, but `enable` has nothing to attach to — there's
no boot-time target membership to create.

Trap: A confuses this with `User=` in `[Service]`; C confuses it
with `After=` in `[Unit]`.

</details>

---

**Q6.** `journalctl -u myapp -p err` is run to check for problems.
Does this show only messages logged at exactly `err` priority, or
something broader?

- A) Only exact `err`-priority messages — `crit`/`alert`/`emerg`
  messages would be missed entirely
- B) `err` and everything more severe than it (`crit`, `alert`,
  `emerg`) — `-p` sets a floor, not an exact match
- C) All messages regardless of priority — `-p` only affects sort
  order, not filtering
- D) Only messages at `err` or less severe (`warning`, `notice`, etc.)

<details>
<summary>Answer</summary>

**B** — Priority filtering with `-p` is a threshold: everything at
or above the specified severity is shown. Missing this means a
search for `err` could silently overlook worse `crit`/`emerg`
entries if someone assumed it was an exact-match filter.

</details>

---

**Q7.** `systemctl status myapp` shows `Loaded: loaded (...;
enabled)` and separately `Active: active (running)`. Why check
both lines rather than assuming one implies the other?

- A) They always match, so checking both is redundant
- B) `Loaded`/its enabled state reflects boot-time configuration;
  `Active` reflects whether it's running right now — a service can
  be enabled but not currently running, or running but not
  enabled, independently of each other
- C) `Loaded` is only relevant immediately after a reboot
- D) `Active` supersedes `Loaded` and makes it irrelevant once a
  service has started at least once

<details>
<summary>Answer</summary>

**B** — This is the enable-vs-start independence from section 1.3,
visible directly in real `systemctl status` output. Reading only
one line and assuming it tells you both is exactly how a
"Disabled + Started" service goes unnoticed until the next reboot.

</details>

---

| Score | Action |
|---|---|
| 7/7 | Import Anki cards, move to Demo 08 |
| 6/7 | Review the wrong answer, then proceed |
| 5/7 | Re-read the relevant section, retry those questions |
| Below 5/7 | Re-read the full demo and redo the walkthrough before proceeding |
````