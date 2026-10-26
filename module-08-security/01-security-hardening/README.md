# Demo M08-01 — Security and Hardening

## Overview

Every hardening task in this demo — auditing SSH config, checking for
repeated failed logins, tracking sensitive changes — sits on top of tools
this series has already built. This demo covers SELinux/AppArmor, SSH
hardening, `fail2ban`, and `auditd` basics — the core hardening toolkit
for a production Linux host. Firewall mechanics themselves
(`firewalld`/`ufw`) already live in Demo M03-02 — this demo references that
rather than re-teaching it.

## Demo Objectives

1. Explain Mandatory Access Control (SELinux/AppArmor) and how it differs from the permission model in Demo M01-10
2. Check and change SELinux/AppArmor mode, and read a basic denial
3. Harden SSH via `sshd_config`, building on Demo M03-01's key-auth setup
4. Configure `fail2ban` to block repeated failed login attempts
5. Use `auditd`/`ausearch` for basic sensitive-action tracking

## Directory Structure

```
01-security-hardening/
├── README.md                           # this file
├── m08-01-security-hardening-anki.csv  # Anki flashcard deck
└── m08-01-security-hardening-quiz.md   # standalone quiz
```

## Environments

| Task | Ubuntu 24.04 | Rocky Linux 9.8 | EC2 Amazon Linux |
|---|---|---|---|
| MAC system | AppArmor (default) | SELinux (default) | Not required |
| `fail2ban` | `sudo apt install fail2ban` | `sudo dnf install fail2ban` | Not required |
| `auditd` | `sudo apt install auditd` | `sudo dnf install audit` | Not required |

> **Verification note:** every tool this demo depends on was checked for
> live availability in the authoring sandbox — `getenforce`,
> `setenforce`, `aa-status`, `sshd`, `auditctl`, `ausearch`,
> `fail2ban-client`, and `semanage` are all absent, and there is no LSM
> interface mounted at all (`/sys/kernel/security/` does not exist), so
> none of this demo's commands or outputs could be run live here. The
> concepts and directive names (MAC vs. DAC, SELinux modes,
> `sshd_config` directives, `fail2ban`'s jail model, `auditd`'s watch/key
> model) are stable, well-documented behavior — recommend a real
> verification pass on an actual Ubuntu/Rocky9 machine before treating
> exact command output shapes as guaranteed.

## Recall Check — 04-sysctl-tuning

Answer these from memory before continuing — no looking back at Demo M07-04.

1. Does a value set with `sysctl -w` survive a reboot? What actually
   makes a setting persistent?
2. Two files in `/etc/sysctl.d/` set the same key to different values.
   Which one wins, and why?
3. A file in `/etc/sysctl.d/` has no `.conf` suffix. What happens when
   `sysctl --system` runs?

<details>
<summary>Answers</summary>

1. No — `sysctl -w` changes only the running kernel; nothing on disk
   records it. Persisting a setting means putting it in a `.conf` file
   under `/etc/sysctl.d/`.

2. The file that sorts later by file name — `sysctl --system` applies
   files in lexical order across the standard directories, and the last
   setting for a key wins, with no warning about the conflict.

3. It's ignored entirely — not applied, not even listed in the
   `--system` output, with no error raised.

</details>

## Part 1 — Concepts

### 1.1 Mandatory Access Control — a layer beyond Demo M01-10's permissions

Demo M01-10's `rwx` permissions and ACLs are **Discretionary Access Control
(DAC)** — the file's *owner* decides who can access it, and any process
running as that owner can access anything the owner is allowed to.
**Mandatory Access Control (MAC)** — SELinux and AppArmor — adds a
*second, independent* layer of restriction that the file owner cannot
override: even a process running as `root`, with full DAC permission to
touch a file, can be blocked by MAC policy from actually doing so.

This matters specifically for **defense in depth**: if an application gets
compromised and an attacker gains the privileges of the process it's
running as, DAC permissions alone only protect what that specific user
account can't already access — which, for a compromised web server running
as `root` or a broadly-privileged service account, might be very little.
MAC policy can constrain that same compromised process to only the
specific files, ports, and capabilities its *profile* explicitly allows,
regardless of what its Unix user ID would otherwise permit.

### 1.2 SELinux — modes, contexts, and denials

```bash
getenforce
# Enforcing, Permissive, or Disabled

sudo setenforce 0   # switch to Permissive (log denials, don't block them)
sudo setenforce 1   # switch back to Enforcing
```

**Permissive mode is a genuinely useful diagnostic tool, not just "SELinux
off."** In Permissive mode, SELinux still evaluates every policy decision
and logs what *would* have been denied, without actually blocking
anything — the correct way to test whether SELinux is the cause of an
unexplained failure, or to validate a new policy change safely before
enforcing it for real.

```bash
ls -Z /var/www/html/
# -Z: show SELinux context (user:role:type:level) alongside normal ls
# output — every file has a security context, not just standard rwx
# permissions.

sudo semanage port -a -t http_port_t -p tcp 8090
# Registers port 8090 as one SELinux considers valid for HTTP-type
# services — a common real-world need when running a service on a
# non-standard port, since SELinux's default policy only permits web
# servers to bind the ports it's been told are web-server ports.
```

```bash
sudo ausearch -m avc -ts recent
# -m avc: SELinux denial (Access Vector Cache) events specifically
# -ts recent: recent timestamp only — the direct way to see what SELinux
# has actually blocked, which is the first thing to check when a service
# fails mysteriously right after SELinux is confirmed Enforcing.
```

### 1.3 AppArmor — profiles and modes

```bash
sudo aa-status
# Lists every AppArmor profile currently loaded, and whether each is in
# "enforce" or "complain" mode.

sudo aa-complain /etc/apparmor.d/usr.sbin.nginx
# complain mode: same diagnostic-without-blocking idea as SELinux's
# Permissive — log what the profile would deny, without enforcing it.

sudo aa-enforce /etc/apparmor.d/usr.sbin.nginx
# Switch back to actually enforcing that specific profile.
```

**AppArmor's model is path-based (profiles reference specific file paths),
while SELinux's is label-based (contexts attached to files/processes,
independent of path)** — a genuinely different underlying approach, not
just different tooling for the same idea. This is part of why the two
aren't interchangeable and why each distro defaults to the one that fits
its broader configuration philosophy (Ubuntu: AppArmor, simpler and more
directly tied to specific applications; Rocky9/RHEL-family: SELinux, more
granular and centrally policy-driven).

### 1.4 SSH hardening — building on Demo M03-01

Demo M03-01 set up SSH key authentication. Hardening `/etc/ssh/sshd_config`
closes off what that key-only auth was implicitly working around:

```bash
sudo grep -E '^(PasswordAuthentication|PermitRootLogin|PubkeyAuthentication)' /etc/ssh/sshd_config
```

```
PasswordAuthentication no
PermitRootLogin no
PubkeyAuthentication yes
```

`PasswordAuthentication no` disables password login entirely — now that
Demo M03-01 set up key-based auth, there's no longer a reason to leave password
auth (a much weaker, brute-forceable auth method) available at all.
`PermitRootLogin no` blocks direct root SSH login — anyone needing root
access should authenticate as their own user first, then `sudo`, which
preserves an audit trail of *who* escalated, not just that "root did
something."

```bash
sudo sshd -t
# -t: TEST the config for syntax errors WITHOUT restarting sshd — the
# same "validate before applying" discipline this series has used
# elsewhere for other config-driven services. Never skip this step:
# a syntax error in sshd_config combined with an immediate restart can
# lock you out of a remote server entirely if the service fails to
# restart cleanly.
sudo systemctl restart sshd
# Same restart pattern as any systemd service (Demo M02-04) — restarting
# doesn't disconnect currently-active sessions, only affects new
# connection attempts.
```

### 1.5 fail2ban — automated response to repeated failures

```bash
sudo dnf install fail2ban   # or apt, per your distro
sudo systemctl enable --now fail2ban
```

```ini
# /etc/fail2ban/jail.local
[sshd]
enabled = true
maxretry = 5
findtime = 600
bantime = 3600
```

`maxretry`/`findtime`: 5 failed attempts within 600 seconds (10 minutes)
triggers a ban. `bantime`: the offending IP is banned for 3600 seconds (1
hour). **`fail2ban` doesn't invent its own blocking mechanism — it
actually manipulates the same firewall (`firewalld`/`ufw`/`iptables`) rules
Demo M03-02 already covered**, adding and removing ban rules dynamically as
attempts are detected and bans expire. This is a direct, concrete
application of Demo M03-02's firewall knowledge, not a separate blocking
system to learn independently.

```bash
sudo fail2ban-client status sshd
# Shows currently banned IPs and recent ban/attempt counts for the sshd
# jail specifically.
sudo fail2ban-client set sshd unbanip 203.0.113.50
# Manually unban a specific IP — necessary when a legitimate user
# (or an automated deployment system) triggers a ban by accident.
```

### 1.6 auditd — tracking sensitive actions

```bash
sudo systemctl enable --now auditd
sudo auditctl -w /etc/passwd -p wa -k passwd_changes
# -w: watch this specific file. -p wa: log Write and Attribute-change
# events (not reads — read-auditing a frequently-read file like
# /etc/passwd would generate enormous noise for little value).
# -k: a searchable KEY tag, making later ausearch queries much faster
# than grepping raw audit logs directly.

sudo ausearch -k passwd_changes
# Every logged event tagged with that key — who changed /etc/passwd,
# when, and via which process, directly answering "who did this" for a
# security-sensitive file.
```

`auditd` isn't a preventive control the way SELinux/AppArmor or firewall
rules are — it doesn't block anything. Its value is **accountability**:
providing a reliable, queryable record of who did what to sensitive
files/actions, which matters enormously during an actual incident
investigation even though it does nothing to stop the action from
happening in the first place.

## Part 2 — Hands-On Lab

### Commands used in this demo

| Command | Flag meaning | Purpose |
|---|---|---|
| `getenforce` / `setenforce 0\|1` | — | Check / change SELinux mode |
| `ausearch -m avc -ts recent` | `-m avc`: denial events | Find recent SELinux denials |
| `aa-status` / `aa-complain` / `aa-enforce` | — | AppArmor profile status and mode control |
| `sshd -t` | `-t`: test | Validate sshd_config before restarting |
| `fail2ban-client status sshd` | — | Ban status for the sshd jail |
| `auditctl -w PATH -p wa -k KEY` | `-w`: watch, `-p`: permissions, `-k`: key | Watch a sensitive file |
| `ausearch -k KEY` | — | Query audit events by key |

### Task 1 — SELinux mode and a denial (Rocky9) or AppArmor status (Ubuntu)

```bash
# Rocky9:
getenforce
sudo ausearch -m avc -ts recent | tail -10

# Ubuntu:
sudo aa-status
```

### Task 2 — SELinux context inspection or AppArmor mode switch

```bash
# Rocky9:
ls -Z /var/www/html/ 2>/dev/null || ls -Z /etc/ 2>/dev/null | head -5

# Ubuntu:
sudo aa-complain /etc/apparmor.d/usr.sbin.nginx 2>/dev/null || echo "profile not present in this lab environment"
```

### Task 3 — SSH hardening, test-before-restart

```bash
sudo grep -E '^(PasswordAuthentication|PermitRootLogin)' /etc/ssh/sshd_config
sudo sshd -t && echo "Config syntax OK"
```

### Task 4 — fail2ban jail status

```bash
sudo systemctl status fail2ban 2>/dev/null || echo "fail2ban not installed in this lab environment"
sudo fail2ban-client status sshd 2>/dev/null || echo "sshd jail not active"
```

### Task 5 — auditd: watch a file and search for events

```bash
sudo auditctl -w /etc/hosts -p wa -k hosts_file_changes
echo "# test change" | sudo tee -a /etc/hosts > /dev/null
sudo ausearch -k hosts_file_changes -ts recent | tail -10
sudo sed -i '/# test change/d' /etc/hosts
```

## What You Learned

- ✅ Explained MAC (SELinux/AppArmor) and how it differs from Demo M01-10's permission model
- ✅ Checked and changed SELinux/AppArmor mode, and read a basic denial
- ✅ Hardened SSH via `sshd_config`, testing before restarting
- ✅ Configured `fail2ban`, understanding it manipulates Demo M03-02's firewall rules directly
- ✅ Used `auditd`/`ausearch` for sensitive-action tracking

## Part 3 — Break-Fix Scenario

### Scenario — NovaTech Ops: two hardening incidents

> **Incident report — NovaTech Ops, ticket #4778**
>
> Alert 1: "a newly deployed web server returns 403 errors for all static
> files, even though file permissions and the web server config both look
> completely correct." Alert 2: "a legitimate deployment automation
> account got banned by fail2ban and can no longer deploy."

**Incident 1 — 403 errors despite correct permissions and config**

```bash
$ ls -la /var/www/html/index.html
-rw-r--r-- 1 nginx nginx 512 Aug 15 10:00 /var/www/html/index.html
$ cat /etc/nginx/nginx.conf | grep root
    root /var/www/html;
$ getenforce
Enforcing
```

Tasks:
1. Standard DAC permissions (`ls -la`) look correct, and the nginx config
   looks correct. Reference §1.1/§1.2 — what's a category of cause this
   evidence hasn't ruled out at all?
2. What single command would confirm or rule out that category?
3. Given the file was likely moved into `/var/www/html/` from somewhere
   else (a common deploy pattern), why might it have the wrong SELinux
   context even with completely correct DAC permissions?

**Incident 2 — legitimate deployment account banned by fail2ban**

```bash
$ sudo fail2ban-client status sshd
Status for the jail: sshd
|- Currently failed: 2
|- Total failed:     47
`- Currently banned: 1
   `- IP list:   10.0.9.50
$ # 10.0.9.50 is the CI/CD deployment server's IP
```

Tasks:
1. What most likely caused a *legitimate* automation account to trigger
   `fail2ban`'s ban threshold, given §1.5's `maxretry`/`findtime`
   configuration?
2. What's the immediate fix to restore deployment access, referencing the
   specific `fail2ban-client` command from §1.5?
3. What's the actual underlying problem to investigate and fix, beyond
   just unbanning the IP once?

<details>
<summary>Answers</summary>

**Incident 1:** Standard DAC permissions being correct doesn't rule out a
completely separate, independent restriction layer — SELinux (confirmed
`Enforcing` here) can block access even when DAC permissions would
otherwise allow it, exactly per §1.1's "second, independent layer" point.
`sudo ausearch -m avc -ts recent` would directly confirm or rule this out,
showing any SELinux denials logged around the time of the 403 errors. The
file was likely moved (`mv`/`cp` from a different location, or extracted
from a deploy archive) rather than created fresh in place — SELinux
contexts are typically inherited from the *parent directory* only when a
file is created there directly; a file moved in from elsewhere often
retains its *original* context, which may not match what `httpd`/`nginx`'s
policy expects to serve from `/var/www/html/`, producing exactly this
"permissions look right, SELinux blocks it anyway" symptom.

**Incident 2:** A CI/CD deployment account triggering a failed-login ban
threshold most commonly indicates the automation's SSH key or credentials
recently changed, rotated, or expired without the deployment pipeline
being updated to match — each deployment run then fails authentication
repeatedly (once per deploy attempt, potentially multiple times per run if
retried), accumulating toward `maxretry` within the `findtime` window
faster than a human user typically would through occasional typos.
Immediate fix: `sudo fail2ban-client set sshd unbanip 10.0.9.50` to
restore access right away. The actual underlying problem — beyond just
unbanning once — is verifying the deployment account's actual current SSH
key/credentials are correctly configured and match what's authorized on
the target server, since simply unbanning without fixing the credential
mismatch means the exact same ban will likely recur on the very next
deployment run.

</details>

## Interview Prep

**Q1. A service fails with what looks like a permissions error, but `ls -la` shows completely correct ownership and mode bits. What should you check next, and why?**
Whether SELinux or AppArmor (MAC) is active and blocking the action — DAC
permissions (what `ls -la` shows) and MAC policy are two independent
layers, and a process can be fully permitted by DAC while still being
blocked by MAC. `getenforce`/`aa-status` confirms whether MAC is active at
all, and `ausearch -m avc` (SELinux) or the relevant AppArmor logs reveal
the specific denial, which `ls -la` alone can never show since it only
describes the DAC layer.

**Q2. Why is testing `sshd_config` with `sshd -t` before restarting genuinely important, more than a general best practice?**
A syntax error in `sshd_config` combined with restarting the service can
mean `sshd` fails to come back up cleanly — for a remote server, that can
mean losing SSH access entirely with no way back in except out-of-band
console access, if that's even available. `sshd -t` validates the config
without touching the running service at all, catching exactly this class
of mistake before it becomes a lockout.

**Q3. `fail2ban` bans an IP by manipulating firewall rules. Does this mean it's a separate blocking mechanism from what Demo M03-02 covered?**
No — `fail2ban` directly manipulates the same `firewalld`/`ufw`/`iptables`
rules from Demo M03-02, adding and removing ban rules dynamically as failed
attempts are detected and bans expire. It's an automation layer on top of
the existing firewall mechanism, not an independent blocking system with
its own separate enforcement.

**Q4. What's the practical value of `auditd` if it doesn't actually prevent any action from happening?**
Accountability during incident investigation. `auditd` provides a
reliable, queryable record of who did what to specifically watched files
or actions — when something sensitive changes and you need to know *who*
did it, *when*, and *how*, `auditd`'s log is the authoritative source,
even though it did nothing to stop the change from happening in real time.
Prevention and accountability are different, complementary security
goals, and `auditd` specifically serves the latter.

---

## Key Takeaways

1. **MAC (SELinux/AppArmor) is a second, independent access-control layer
   beyond Demo M01-10's DAC permissions — not a replacement for it.** A
   process can be fully permitted by `rwx`/ACLs and still blocked by MAC
   policy; `ls -la` alone can never reveal a MAC denial.
2. **SELinux Permissive mode and AppArmor complain mode are diagnostic
   tools, not "MAC off."** Both log what would be denied without actually
   blocking, the correct way to test or validate policy safely.
3. **Always `sshd -t` before restarting `sshd` after an `sshd_config`
   edit.** A syntax error combined with a restart can lock you out of a
   remote server entirely.
4. **`fail2ban` isn't a separate blocking system — it directly manipulates
   the same firewall rules Demo M03-02 already covered**, automating what
   would otherwise be manual ban/unban firewall commands.
5. **`auditd` provides accountability, not prevention.** It doesn't block
   anything; its value is a reliable, queryable record of who did what to
   sensitive, watched files or actions, essential during incident
   investigation.

> **Demo scope:** Primary concept: auditing and hardening a Linux server's
> access-control and authentication layers. Supporting concepts: MAC vs.
> DAC, SELinux, AppArmor, SSH hardening, fail2ban, auditd.
> Estimated completion time: 32 minutes (reading + hands-on + verification).
> Checkpoints: 5 natural stopping points (after Task 1 through Task 5).

---

## Quick Commands Reference

### SELinux

| Command | Purpose |
|---|---|
| `getenforce` / `setenforce 0\|1` | Check / change mode |
| `ls -Z` | Show SELinux context |
| `ausearch -m avc -ts recent` | Recent denials |
| `semanage port -a -t TYPE -p tcp PORT` | Register a non-standard port |

### AppArmor

| Command | Purpose |
|---|---|
| `aa-status` | Loaded profiles and their modes |
| `aa-complain PROFILE` | Log-only, don't enforce |
| `aa-enforce PROFILE` | Actually enforce |

### SSH hardening

| Command | Purpose |
|---|---|
| `sshd -t` | Validate config before restart — never skip |
| `systemctl restart sshd` | Apply after validation |

### fail2ban and auditd

| Command | Purpose |
|---|---|
| `fail2ban-client status JAIL` | Ban status for a jail |
| `fail2ban-client set JAIL unbanip IP` | Manually unban |
| `auditctl -w PATH -p wa -k KEY` | Watch a sensitive file |
| `ausearch -k KEY` | Query events by key |

## Appendix A — Anki Cards

**`m08-01-security-hardening-anki.csv`:**

```
#deck:Linux Mastery::Module 8 - Security::Demo M08-01 - Security Hardening
#separator:Comma
#columns:Front,Back,Tags
"A service fails what looks like a permissions error, but ls -la shows completely correct DAC ownership/mode bits. What's a category of cause this hasn't ruled out?","MAC (SELinux/AppArmor) denial — DAC permissions (what ls -la shows) and MAC policy are two independent layers. A process can be fully permitted by DAC and still blocked by MAC.","demo-m08-01,mac,dac,troubleshooting"
"What's the difference between what SELinux Permissive mode and Enforcing mode actually do?","Permissive: SELinux still evaluates every policy decision and LOGS what would be denied, without actually blocking it. Enforcing: denials are actually blocked. Permissive is a diagnostic tool, not 'SELinux off'.","demo-m08-01,selinux,modes"
"Why must sshd -t be run before restarting sshd after editing sshd_config?","A syntax error combined with a restart can mean sshd fails to come back up cleanly, potentially locking you out of a remote server entirely. sshd -t validates config without touching the running service.","demo-m08-01,ssh,sshd-config,safety"
"Does fail2ban use its own independent blocking mechanism separate from firewalld/ufw?","No — fail2ban directly manipulates the SAME firewall rules covered in Demo M03-02, adding and removing ban rules dynamically as failed attempts are detected and bans expire. It's automation on top of the existing firewall, not a separate system.","demo-m08-01,fail2ban,firewall,cross-demo"
"A legitimate automation/CI account gets banned by fail2ban. What's the most likely underlying cause, beyond just unbanning it once?","The automation's SSH key/credentials likely changed or expired without the pipeline being updated — repeated failed auth attempts per deploy run accumulate toward maxretry faster than typical human error would. Unbanning alone doesn't fix the recurring cause.","demo-m08-01,fail2ban,troubleshooting"
"What does auditd actually provide, given it doesn't prevent any action from happening?","Accountability — a reliable, queryable record of who did what to specifically watched files/actions, essential during incident investigation, even though it does nothing to stop the action from happening in real time.","demo-m08-01,auditd,fundamentals"
"Why might a file moved into /var/www/html/ from elsewhere have the wrong SELinux context even with completely correct DAC permissions?","SELinux contexts are typically inherited from the parent directory only when a file is CREATED there directly. A file moved in (mv/cp from elsewhere, or extracted from an archive) often retains its ORIGINAL context, which may not match what the serving policy expects.","demo-m08-01,selinux,context,troubleshooting"
"What's the fundamental model difference between AppArmor and SELinux?","AppArmor is path-based (profiles reference specific file paths); SELinux is label-based (contexts attached to files/processes, independent of path) — genuinely different underlying approaches, not just different tooling for the same idea.","demo-m08-01,selinux,apparmor,comparison"
"Why does PermitRootLogin no matter for audit trail purposes, beyond just being 'more secure'?","It forces anyone needing root access to authenticate as their own user first, then sudo — preserving a record of WHO escalated to root, rather than just that 'root did something' with no individual accountability.","demo-m08-01,ssh,hardening,accountability"
```

## Appendix B — Quiz

**`m08-01-security-hardening-quiz.md`:**

````markdown
# Quiz — Demo M08-01: Security and Hardening

> One correct answer per question unless stated otherwise.
> Target: 80% or above before moving to Demo M08-02.

---

**Q1. A service fails what looks like a permissions error, but `ls -la` shows completely correct DAC ownership/mode bits. What should you check?**

- A) Re-run `chmod` with more permissive bits until it works
- B) Whether MAC (SELinux/AppArmor) is active and blocking the action — an independent layer `ls -la` can't reveal
- C) The file's creation timestamp
- D) Whether the disk has enough free space

<details>
<summary>Answer</summary>

**B** — DAC and MAC are independent layers; correct DAC permissions say nothing about MAC policy. Trap: A treats a MAC issue as a DAC problem; C and D are unrelated diagnostic dead ends.

</details>

---

**Q2. What does SELinux Permissive mode actually do?**

- A) Disables SELinux entirely, identical to `setenforce` being unavailable
- B) Still evaluates every policy decision and logs what would be denied, without actually blocking anything
- C) Only applies to network-related denials
- D) Permanently disables SELinux until a reboot

<details>
<summary>Answer</summary>

**B** — This is Permissive mode's actual, documented behavior — a diagnostic tool, not "off." Trap: A and D both overstate it as equivalent to disabling SELinux; C invents a scope restriction that doesn't exist.

</details>

---

**Q3. Why is `sshd -t` critical to run before restarting `sshd` after an `sshd_config` edit?**

- A) It's purely cosmetic and has no real consequence if skipped
- B) A syntax error combined with a restart can leave `sshd` failing to come back up, potentially locking you out of a remote server entirely
- C) It automatically fixes any syntax errors found
- D) It's only relevant for local, not remote, SSH servers

<details>
<summary>Answer</summary>

**B** — This is a real, serious risk `sshd -t` specifically guards against by validating without touching the running service. Trap: A dismisses a genuine safety step; C invents auto-fix behavior; D misapplies the risk as local-only, when remote lockout is the actual concern.

</details>

---

**Q4. Does `fail2ban` use a blocking mechanism independent of `firewalld`/`ufw`?**

- A) Yes, it has its own proprietary blocking system
- B) No — it directly manipulates the same firewall rules from Demo M03-02, adding/removing ban rules dynamically
- C) Yes, but only on Rocky9
- D) No, it blocks at the DNS level instead

<details>
<summary>Answer</summary>

**B** — `fail2ban` is automation layered on top of the existing firewall mechanism, not a separate enforcement system. Trap: A and C both invent an independent mechanism that doesn't exist; D misidentifies the actual blocking layer.

</details>

---

**Q5. A legitimate CI/CD deployment account gets banned by `fail2ban`. What's the most likely underlying cause?**

- A) `fail2ban` randomly bans a percentage of legitimate traffic by design
- B) The automation's SSH credentials likely changed/expired without the pipeline being updated, causing repeated failed auth attempts
- C) The deployment server's clock is out of sync
- D) `fail2ban` cannot distinguish automation traffic from attacks under any circumstances

<details>
<summary>Answer</summary>

**B** — This is the realistic, common cause — repeated failed auth from a stale credential accumulates toward the ban threshold quickly. Trap: A invents random banning that isn't how `fail2ban` works; C is an unrelated (though real) potential issue not indicated here; D overstates a limitation — proper credential management avoids this entirely.

</details>

---

**Q6. What does `auditd` actually provide, given it doesn't prevent any action from happening?**

- A) Real-time blocking of unauthorized file access
- B) Accountability — a reliable, queryable record of who did what to watched files/actions, valuable during incident investigation
- C) Automatic rollback of unauthorized changes
- D) Firewall rule generation based on observed traffic

<details>
<summary>Answer</summary>

**B** — This is `auditd`'s actual, specific value: a record for after-the-fact investigation, not prevention. Trap: A, C, and D all invent preventive or automated-response capabilities `auditd` doesn't have.

</details>

---

**Q7. A file moved into `/var/www/html/` from elsewhere has the wrong SELinux context despite correct DAC permissions. Why?**

- A) SELinux contexts are randomly assigned on every boot
- B) Contexts are typically inherited from the parent directory only when a file is created there directly; a moved file often retains its original context
- C) This only happens if the file was moved using `cp`, never `mv`
- D) SELinux contexts don't apply to files under `/var/www/`

<details>
<summary>Answer</summary>

**B** — This is the actual, common mechanism behind exactly this class of SELinux-related deployment surprise. Trap: A invents random behavior; C draws a false distinction between `cp`/`mv`; D is factually wrong — SELinux contexts apply broadly across the filesystem.

</details>

---

Score guide:

| Score | Action |
|---|---|
| 7/7 | Import Anki cards, move to Demo M08-02 |
| 6/7 | Review the wrong answer, then proceed |
| 5/7 | Re-read the relevant section, retry those questions |
| Below 5/7 | Re-read the full demo and redo the walkthrough before proceeding |
````