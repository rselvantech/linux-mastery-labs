# Demo 05 — Users, Groups & Authentication

## Overview

Every action on a Linux system happens as some user. Every file has
an owner. Every process runs with an identity. Getting users and
groups right is the foundation that permissions (Demo 03) sit on
top of — and getting them wrong causes some of the most disruptive
incidents in production: a deploy account losing sudo access mid
pipeline, a service account that can log in interactively when it
shouldn't, or a single `usermod` flag silently wiping a user out of
every group they need.

This demo covers how Linux stores identity (`/etc/passwd`,
`/etc/shadow`, `/etc/group`), how to create and manage users and
groups safely, how `sudo` actually works, and the one Ubuntu vs
Rocky9 difference that catches almost every engineer moving between
distro families for the first time: the sudo group is not called
the same thing on both.

---

## Demo Objectives

By the end of this demo you will be able to:

1. Read and interpret every field in `/etc/passwd`, `/etc/shadow`,
   and `/etc/group`
2. Create, modify, and remove users with `useradd`, `usermod`,
   `userdel`
3. Explain the difference between a primary group and supplementary
   groups, and why the User Private Group scheme exists
4. Identify and avoid the `usermod -G` vs `-aG` danger — one of the
   most common group-management incidents
5. Grant sudo access correctly on both Ubuntu (`sudo` group) and
   Rocky9 (`wheel` group)
6. Edit sudo configuration safely with `visudo` and `/etc/sudoers.d/`
7. Set password aging policy with `chage`
8. Create a correct service account — no login shell, no
   interactive password, used only to run a service
9. Explain the `su` command and compare it with `sudo` — when
   each is appropriate and why `sudo` is the modern standard

---

## Directory Structure

```
05-users-groups/
├── README.md                          # this file
├── 05-users-groups-anki.csv           # Anki flashcard deck
└── 05-users-groups-quiz.md            # standalone quiz
```

---

## Environments

| Task | Ubuntu 24.04 | Rocky Linux 9.8 | EC2 Amazon Linux 2023 |
|---|---|---|---|
| All lab tasks | ✅ | ✅ | Not required |
| sudo group name | `sudo` | `wheel` | `wheel` (RHEL family) |

Run every task on both Ubuntu and Rocky9. The sudo group name
difference is the single most important distro-family fact in this
demo — confirmed in Task 4.

---

## Recall Check

> Answer from memory before reading further. From Demo 04.

1. `dnf install filebeat` fails with `GPG check FAILED` even though
   the repo file has `gpgcheck=1`. What is the actual cause?

2. `rpm -V nginx` returns `S.5....T.  c  /etc/nginx/nginx.conf`.
   Is this a problem? Why or why not?

3. You add the HashiCorp repository on Rocky9 and run
   `dnf install terraform`. The package downloads successfully
   but the install fails with `Error: GPG check FAILED`.
   `gpgcheck=1` is set in the repo file. What was skipped
   and what is the exact fix?

<details>
<summary>Answers</summary>

1. The GPG key was never imported into the RPM database. The
   repo file's `gpgkey=` line tells `dnf` where to find the key but
   does not import it automatically. Fix: `rpm --import URL`.

2. No — the `c` tag marks this as a config file. Config files are
   expected to be modified after install; `rpm -V` flags this as
   informational, not a failure. The concern would be the same
   flags on a binary with no `c` tag.

3. The GPG key was never imported into the RPM database.
   The repo file's `gpgkey=` line tells dnf where to find the
   key but does not import it automatically. Fix:
   `sudo rpm --import https://rpm.releases.hashicorp.com/gpg`
   then retry the install.

</details>

---

## Part 1 — Concepts

### 1.1 The three files that define every user

Linux stores user identity, password data, and group membership in
three separate plain-text files. Understanding their structure is
the foundation for everything else in this demo.

**`/etc/passwd`** — one line per user, despite the name, no
passwords are stored here (the password field was moved to
`/etc/shadow` decades ago for security — `/etc/passwd` must remain
world-readable for many tools to function, but password hashes
must not be):

```bash
cat /etc/passwd | grep wadmin
# wadmin:x:1000:1000:Wadmin User:/home/wadmin:/bin/bash
```

```
wadmin : x : 1000 : 1000 : Wadmin User : /home/wadmin : /bin/bash
  ↑      ↑    ↑      ↑         ↑              ↑             ↑
  │      │    │      │         │              │             └── login shell
  │      │    │      │         │              └── home directory
  │      │    │      │         └── GECOS field (comment — full name)
  │      │    │      └── primary GID
  │      │    └── UID
  │      └── password placeholder — 'x' means "see /etc/shadow"
  └── username
```

Fields, in order: `username:password:UID:GID:GECOS:home:shell`

```
UID ranges (Linux Standard Base convention):
0          root
1–999      system/service accounts (no interactive login expected)
1000+      regular human users (standard on most modern distros)
```

**What is the GECOS field?**

GECOS stands for **G**eneral **E**lectric **C**omprehensive
**O**perating **S**ystem — a mainframe operating system from the
1960s that early Unix systems used for print spooling and batch
job processing. The field was added to `/etc/passwd` to carry
the contact information needed for that integration, and the name
stuck long after the original purpose was gone.

On modern Linux systems, the GECOS field is commonly used
to store general information about the user, such as their full
name, office number, phone number, or other contact details.
For service accounts, it describes the account's purpose rather
than a person — exactly as used throughout this demo (`"NovaTech
App service account"`).

**`/etc/shadow`** — readable only by root, stores the actual
password hash and aging policy:

```
username:hash:lastchange:minage:maxage:warn:inactive:expire:reserved
```

```
username     Same as /etc/passwd — links the shadow entry to the
             passwd entry. MUST match exactly.

hash         The actual password hash in $id$salt$hash format.
             $id$ identifies the algorithm (see section 1.1a below).
             If the field is:
               * or !  → account cannot authenticate via password
                         (other methods like SSH keys may still work)
               !!      → password was never set (common on new
                          accounts before passwd is run)
               !hash   → password is LOCKED (! prepended to a
                          valid hash) — the hash is preserved but
                          login via password is blocked

lastchange   Days since Unix epoch (Jan 1, 1970) when the password
             was last changed. Used to calculate when it expires.
             0 means "password must be changed at next login"
             (what chage -d 0 sets — you saw this in Task 6)

minage       Minimum days that must pass before the user can change
             the password again. 0 = can change any time.
             Purpose: prevent users from immediately changing back
             to their old password, bypassing rotation policy.

maxage       Maximum days the password is valid. After lastchange +
             maxage days, the password is expired and the user is
             forced to change it at next login. 99999 ≈ "never"
             (273 years).

warn         Days before expiry that the user sees a warning
             ("Your password will expire in N days") at login.

inactive     Days AFTER password expiry during which the account
             is still allowed to log in (to change the password).
             After this window, the account is fully locked even
             though the password expiry check would normally just
             force a change. -1 = feature disabled.
             DIFFERENCE from expire: inactive is a grace window
             after the PASSWORD expires; expire (below) is a
             hard date the ACCOUNT itself becomes unusable.

expire       Absolute date (days since epoch) when the ACCOUNT
             expires and becomes unusable — regardless of password
             state. -1 = never expires. Set by chage -E.

reserved     Currently unused, always empty. Reserved for future
             use by the shadow-utils package.
```

This is the file `chage` reads and writes — covered in section 1.6.

**Hash algorithm prefixes in `/etc/shadow`:**

```
$1$    MD5 — obsolete, extremely weak, still present on very old systems
$5$    SHA-256 — uncommon
$6$    SHA-512 — used on Ubuntu 14.04–20.04 and Rocky/RHEL 8+
$y$    yescrypt — the DEFAULT on Ubuntu 22.04+ (including 24.04)
```

Your own output confirmed this:
```
wadmin:$y$j9T$HcMwF0...
        ↑
        yescrypt — because your system is Ubuntu 24.04,
        which switched from SHA-512 ($6$) to yescrypt ($y$)
        as the default in 22.04 LTS
```

Per Ubuntu's official security documentation: "Ubuntu 22.04 LTS
(Jammy Jellyfish) and later use yescrypt, which is based on
scrypt, a computationally expensive key derivation function."
Rocky Linux 9 still uses SHA-512 (`$6$`) — the distros differ
here. Check your actual system:

```bash
grep ENCRYPT_METHOD /etc/login.defs
# Ubuntu 24.04: ENCRYPT_METHOD YESCRYPT
# Rocky9:       ENCRYPT_METHOD SHA512
```

**Why yescrypt is stronger than SHA-512 for passwords:** SHA-512
is a general-purpose hash — extremely fast, which is what you
want for data integrity but bad for passwords, where fast means
billions of guesses per second on a GPU are feasible. Yescrypt is
deliberately slow and memory-intensive, making brute-force attacks
orders of magnitude more expensive.

**Password expiry vs account inactive vs account expiry — three different lockout mechanisms:**

```
Mechanism      Field       Trigger             What locks
──────────────────────────────────────────────────────────
Password       maxage      lastchange +        Forces password
expiry                     maxage days         CHANGE at next
                                               login — doesn't
                                               prevent login,
                                               just demands a
                                               new password

Inactive       inactive    password expired    LOCKS the account
grace period               + inactive days     entirely after the
                           without login        grace window —
                                               even the "change
                                               your password"
                                               prompt is gone

Account        expire      Absolute date       LOCKS the account
expiry                     (days since         entirely from that
                           epoch)              date regardless of
                                               password state
```

In your own Task 6 output:
```
Password expires      : Sep 28, 2026     ← password expiry (maxage)
Password inactive     : never            ← no grace-window lockout
Account expires       : Jul 30, 2026     ← hard account lockout date
```
After `chage -d 0`:
```
Password expires      : password must be changed   ← forces change NOW
Password inactive     : password must be changed   ← grace window also
Account expires       : Jul 30, 2026               ← unchanged
```
`chage -d 0` set lastchange to 0 (epoch), making maxage
immediately exceeded — password is treated as already expired,
forcing a change at next login.


**`/etc/group`** — one line per group:

```bash
cat /etc/group | grep -E "wadmin|sudo|wheel|docker"
# wadmin:x:1000:
# sudo:x:27:wadmin
# docker:x:998:wadmin
```

```
sudo : x : 27 : wadmin
 ↑     ↑   ↑      ↑
 │     │   │      └── supplementary members (comma-separated list)
 │     │   └── GID
 │     └── password placeholder (group passwords are obsolete, unused)
 └── group name
```

Fields, in order: `groupname:password:GID:members`

Note: `wadmin:x:1000:` has an **empty** members field. This is
expected — `wadmin` is the **primary** group for user `wadmin`.
Primary group membership is recorded in `/etc/passwd` (the GID
field), not listed in `/etc/group`'s members field. Only
**supplementary** membership appears in the members list.

**What are supplementary members, and why is the field sometimes empty?**

The members field in `/etc/group` lists only **supplementary**
(secondary) group memberships — users who belong to this group
*in addition to* their primary group. It does NOT list users
whose primary group is this group.

```
sudo:x:27:wadmin,deployer
↑                ↑
group name       supplementary members — users for whom sudo
                 is an ADDITIONAL group, not their primary group

wadmin:x:1000:
↑              ↑
group name     EMPTY — wadmin's primary group is "wadmin", but
               this fact is recorded in /etc/passwd (GID field),
               not here. The /etc/group members column only shows
               supplementary membership.
```
**`id`** = **i**dentity. Shows the UID, primary GID, and all
supplementary groups for a user. The single most useful command
for verifying what access a user actually has before and after
any group change.

```bash
id                  # your own current identity
id username         # identity of a specific user
```

**`getent`** = **get** **ent**ry. Queries system name service
databases — not just the local files, but whatever name service
is configured (LDAP, NIS, local files). On a plain server with
no LDAP, `getent passwd wadmin` reads `/etc/passwd` and produces
the same result as `grep "^wadmin:" /etc/passwd`. The advantage:
in a managed environment with centralized authentication, `getent`
still works correctly while `grep /etc/passwd` would return
nothing for LDAP users. Use `getent` rather than `grep` on
files — it's the right habit for environments that may ever add
centralized auth.

```bash
getent passwd username    # query user record from any source
getent group groupname    # query group record from any source
getent shadow username    # query shadow entry (requires root)
```


### 1.2 useradd, usermod, userdel

**`useradd`** = **user** **add**.

```bash
sudo useradd -m -s /bin/bash -c "Deploy Service Account" deployer
# -m  create home directory (/home/deployer) — NOT created by default
# -s  login shell
# -c  comment field (GECOS — typically the full name or description)

# Set a password
sudo passwd deployer
```

**Key flags:**

| Flag | Meaning | Effect |
|---|---|---|
| `-m` | create home (**m**ake) | Without it, no home directory is created |
| `-s` | shell | Sets login shell — use `/usr/sbin/nologin` for service accounts |
| `-c` | comment (GECOS) | Free-text description — full name typically |
| `-G` | supplementary groups | **Sets** the complete supplementary group list — see 1.4 danger |
| `-g` | primary group | Sets primary group (overrides UPG default) |
| `-r` | system account | UID from system range, no aging policy — see 1.7 |
| `-u` | specific UID | Force a specific UID number |

**What happens when `useradd` runs with no `-g`:**

On most modern distros, the **User Private Group (UPG)** scheme
creates a new group with the same name and GID as the username,
and sets that as the primary group:

```bash
sudo useradd -m deployer
id deployer
# uid=1002(deployer) gid=1002(deployer) groups=1002(deployer)
#                     ↑
#                     UPG — group "deployer" created automatically,
#                     set as deployer's primary group
```

This was covered in Demo 03 (§1.1, "Knowing your identity first")
— the same mechanism applies here. UPG means every regular user
effectively has a private group matching their username, and this
is the group new files get by default unless a setgid directory
overrides it.

**`usermod`** = **user** **mod**ify.

```bash
sudo usermod -s /usr/sbin/nologin deployer    # change shell
sudo usermod -c "Updated comment" deployer    # change GECOS field
sudo usermod -aG docker deployer              # ADD to docker group — see 1.4
sudo usermod -l newname oldname               # rename the login name
sudo usermod -d /new/home -m deployer         # move home directory
```

**`userdel`** = **user** **del**ete.

```bash
sudo userdel deployer          # removes user, leaves home directory
sudo userdel -r deployer       # -r = remove home directory and mail spool too
```

`userdel` without `-r` is the safer default for production —
preserves data in case removal was premature. Use `-r` only when
you are certain the home directory's contents are no longer needed.

**`passwd`** = **passw**or**d**. Sets or manages a user's
password — but also used for account locking and status checking,
as covered in section 1.8.

```bash
passwd              # change your own password (prompts interactively)
passwd username     # set another user's password (requires sudo)
passwd -S username  # Status — show password state without changing it
passwd -l username  # Lock — prepend ! to the hash, disabling login
passwd -u username  # Unlock — remove the ! prefix, re-enabling login
```

**Understanding `passwd -S` and `passwd -l`:**

`passwd -S username` — **S**tatus. Shows the password state
without changing anything:

```
novatech-svc L 2026-06-30 -1 -1 -1 -1
             ↑             ↑  ↑  ↑  ↑
             │             │  │  │  └─ inactive days (-1 = disabled)
             │             │  │  └─ max password age (-1 = no max)
             │             │  └─ min password age (-1 = no min)
             │             └─ last password change date
             └─ password status:
                L = Locked (! prepended to hash, or never set)
                P = Password set (usable password exists)
                NP = No Password (empty password field)
```

`passwd -l username` — **l**ock. Prepends `!` to the password
hash in `/etc/shadow`, making it impossible to authenticate with
any password. The original hash is preserved — `passwd -u`
(unlock) removes the `!` and restores the usable password. Lock
is NOT the same as "no password" — it means "there IS a hash
but login via password is blocked."

For service accounts: after `useradd -r`, the status is already
`L` (because `!!` — never set — also reads as locked). Running
`passwd -l` explicitly makes the intent visible and handles the
edge case where a password was set before you decided to make it
a service account.


**`sudo`** = **s**uper**u**ser **do**. Runs one command with
elevated privileges as root (or another specified user), governed
entirely by `/etc/sudoers` and files in `/etc/sudoers.d/`. Unlike
`su`, it does not open a persistent shell as another user — the
elevation lasts only for that one command.

```bash
sudo command              # run as root
sudo -u alice command     # run as alice specifically
sudo -i                   # open a login shell as root (modern
                          # replacement for su -)
sudo -l                   # list what commands you are allowed to run
sudo -l -U username       # list what another user is allowed to run
```

**`visudo`** = **vi** (editor) + **sudo**. The only correct way
to edit `/etc/sudoers` or any file in `/etc/sudoers.d/`. It edits
a temporary copy of the file, validates syntax before saving, and
refuses to write an invalid file. Named for `vi` because that was
the default editor when it was written — on modern systems it uses
whatever `$EDITOR` is set to (often `nano`). Always use `visudo
-f /etc/sudoers.d/filename` for drop-in files too, not a plain
text editor.

### 1.3 Primary vs supplementary groups — the concept

Every user has exactly **one** primary group and **zero or more**
supplementary groups.

```
Primary group:
  - Recorded in /etc/passwd (the GID field)
  - The group assigned to new files the user creates
    (unless a setgid directory overrides it — Demo 03, section 1.9)
  - A user always has exactly one

Supplementary groups:
  - Recorded in /etc/group (the members field of each group)
  - Grant additional access — e.g. sudo, docker, wheel
  - A user can have zero, one, or many
  - Checked by the kernel for every group-based permission decision
```

```bash
id deployer
# uid=1002(deployer) gid=1002(deployer) groups=1002(deployer),998(docker),27(sudo)
#                     ↑ primary           ↑ all groups (primary + supplementary)
```


**How the kernel actually uses groups when checking file permissions:**

When a process tries to access a file, the kernel checks the
file's permission bits and the process's identity — specifically:

```
Step 1: Is the process's UID the same as the file's owner UID?
        → Apply the OWNER (user) permission bits

Step 2: Is the process's GID or ANY of its supplementary GIDs
        the same as the file's group?
        → Apply the GROUP permission bits

Step 3: Neither matched?
        → Apply the OTHERS permission bits
```

The key is step 2: the kernel doesn't stop at the primary group —
it checks every group the process belongs to (primary +
supplementary) against the file's group. If any one of them
matches, the group permission bits apply.

**Concrete example:**

```
File:    -rw-r----- owned by root:docker
         (root can read/write, docker group can read, others: nothing)

User wadmin:
  primary group = wadmin (GID 1000)
  supplementary = docker (GID 998), sudo (GID 27)

Can wadmin read the file?
  Step 1: UID 1000 ≠ root (UID 0) — not the owner, skip owner bits
  Step 2: Check all groups:
            wadmin (1000) ≠ docker (998) — no match
            docker (998)  = docker (998) ← MATCH
            → Group permission bits apply: r-- → READ allowed
  Answer: YES, because docker is in wadmin's supplementary groups
```

This is exactly why `usermod -aG docker wadmin` (not bare `-G`)
is important — adding docker as a supplementary group is what
makes step 2 succeed for docker-owned files.

### 1.4 The `-aG` vs `-G` danger

This is one of the most common group-management incidents in
production — and the cause is a single missing character.

**`usermod -aG groupname username`** — ADD to existing groups.
`-a` = **append**. Keeps all current supplementary groups and adds
the new one.

**`usermod -G groupname username`** — REPLACE all supplementary
groups. No `-a` means: wipe every existing supplementary group
membership and set the list to exactly what you typed.

```bash
# Starting state
id deployer
# groups=1002(deployer),27(sudo),998(docker)

# WRONG — forgot -a
sudo usermod -G docker deployer
id deployer
# groups=1002(deployer),998(docker)
#                          ↑
#         sudo membership is GONE — deployer can no longer sudo
#         This was not the intent — the admin just wanted to ADD
#         deployer to docker, not remove them from sudo

# CORRECT — with -a
sudo usermod -aG sudo deployer    # restore sudo
id deployer
# groups=1002(deployer),998(docker),27(sudo)   ← both groups present
```

**Why this is dangerous in real incidents:** the command runs
silently — no warning, no confirmation, no error. The user simply
loses access the next time group membership is checked (often at
next login, since group membership is evaluated at login time, not
live). The failure mode — "sudo suddenly stopped working for this
one account" — can take time to connect back to a `usermod` command
run days earlier.

**The rule:** **always use `-aG`, never bare `-G`**, unless you
specifically intend to replace the entire supplementary group list.
If you do need to replace the full list deliberately, that is the
one legitimate use of bare `-G` — but it should be a deliberate,
documented action, not a habit.

```bash
# Verify current groups BEFORE any -G operation — make replacement explicit
id deployer
# Then if replacing deliberately, include ALL groups you want kept:
sudo usermod -G docker,sudo,deployer deployer
```

### 1.5a sudo, visudo, and sudoers.d

**`sudo`** = **s**uper **u**ser **do**. Runs a single command with
elevated privileges (root by default), governed by rules in
`/etc/sudoers` and `/etc/sudoers.d/`.

**Why you never edit `/etc/sudoers` directly with a text editor:**

A syntax error in `/etc/sudoers` can lock out **all** sudo access
on the system — including your own. If you cannot become root any
other way (no separate root login, no console access), a broken
sudoers file can mean total lockout.

**`visudo`** prevents this. It edits a temporary copy, validates
the syntax before saving, and refuses to save invalid syntax:

```bash
sudo visudo
# Opens /etc/sudoers in your default editor (often vi/nano)
# On save: validates syntax
# If invalid: shows the error, asks whether to re-edit, save anyway, or discard
# Never silently saves broken syntax
```

**What exactly happens when `/etc/sudoers` has a syntax error?**

Not "corruption" in the sense of a damaged file — the file itself
is still readable plain text. But `sudo` reads and parses the
entire file before every command. If it encounters ANY syntax
error anywhere in the file, it refuses to honour ANY rule in the
entire file — not just the broken line, the entire file fails.

```
Error on line 28 of /etc/sudoers
→ sudo refuses to run ANY command for ANY user
→ including the admin who made the edit
→ even if that admin's own valid rule is on line 5
```

This is the lockout: not corruption, but a parsing failure that
makes the whole file effectively empty, removing all sudo access
system-wide until the syntax is fixed. If your own sudo access is
gone and there's no alternative path to root (no root password
set, no console), you genuinely cannot fix it without out-of-band
access (cloud provider console, recovery mode, etc.).

`visudo` prevents this specifically: it validates the syntax of
the entire file before allowing a save, and refuses to write an
invalid file. The message you saw in Task 5:
```
>>> sudoers file: syntax error, line 28 <
What now?
```
is `visudo` catching the problem and giving you the option to
re-edit, save anyway (never do this), or discard — all before
writing the change.

**Sudoers syntax — granting full sudo to a group:**

```
%sudo      ALL         = ( ALL  :  ALL )       ALL
  ↑          ↑              ↑       ↑            ↑
  │          │              │       │            └─ WHAT: any command
  │          │              │       └── run as any GROUP
  │          │              └── run as any USER (including root)
  │          └── WHERE: on any host (useful for /etc/sudoers files
  │               copied across multiple servers)
  └── WHO: the "sudo" group (% prefix = group, not username)
```

Translating to plain English: "Any member of the `sudo` group,
from any host, may run any command as any user and any group."
This is full, unrestricted sudo access. The `(ALL:ALL)` part is
what allows sudo members to run commands as users *other* than
root too — useful when you need `sudo -u appuser command` to run
something as a specific service account rather than root.

On Rocky9/RHEL the identical line reads `%wheel ALL=(ALL) ALL` —
same meaning, different group name.

**Sudoers rule format — the four components, in order:**

```bash
# Format: who   where=(as_whom)   what
# ─────────────────────────────────────
who      = username or %groupname (% prefix = group)
where    = hostname (ALL = any host, typical for a single server)
as_whom  = (run_as_user:run_as_group) — who the command runs AS
what     = command path, or ALL for everything
```

Reading any sudoers line means identifying these four pieces in
order: **who** is granted access, **where** it applies (almost
always `ALL` for a single server — this field exists because one
`/etc/sudoers` file can be copied across many hosts with per-host
rules), **as whom** the command actually runs, and **what** it's
allowed to run. Misreading any one of the four is how a rule
intended to be narrow (one command) ends up broad (any command),
or vice versa.

```bash
# Grant a GROUP full sudo (the standard pattern for admin groups)
%sudo   ALL=(ALL:ALL) ALL    # Ubuntu
%wheel  ALL=(ALL) ALL        # Rocky9/RHEL

# Grant a SINGLE USER full sudo
alice   ALL=(ALL:ALL) ALL

# Grant a user one specific command, requires password
deployer ALL=(ALL) /usr/bin/systemctl restart myapp

# Grant a user one specific command, NO password required
deployer ALL=(ALL) NOPASSWD: /usr/bin/systemctl restart myapp

# Grant a user multiple specific commands, no password
deployer ALL=(ALL) NOPASSWD: \
    /usr/bin/systemctl restart myapp, \
    /usr/bin/systemctl status myapp

# Grant a user sudo as a SPECIFIC user only (not root)
ci-agent ALL=(appuser) NOPASSWD: /opt/app/deploy.sh
# ci-agent can run /opt/app/deploy.sh as appuser only —
# not as root, not as anyone else

# Grant a user everything EXCEPT a dangerous command
alice ALL=(ALL) ALL, !/usr/bin/su, !/bin/bash
# Alice can sudo everything EXCEPT opening a root shell
# Note: this is easy to bypass and not a security boundary

# Allow a group to read logs without password
%developers ALL=(ALL) NOPASSWD: /usr/bin/journalctl -u myapp*
```

**What is `NOPASSWD:`?**

By default, `sudo` prompts for **your own password** (not root's
password) before running the command — this confirms it is really
you, not someone who just left your terminal unlocked, making the
elevated call.

`NOPASSWD:` disables this confirmation for the rules that follow
it on the same line. The command still runs with elevated
privileges — the only thing removed is the password prompt. This
is common in CI/CD pipelines and automation where there is no
human present to type a password, but the rule must be carefully
scoped to specific commands — `NOPASSWD: ALL` effectively gives
passwordless root access to the whole system.

```
deployer ALL=(ALL) NOPASSWD: /usr/bin/systemctl restart myapp
                   ↑
                   No password required for THIS specific command.
                   deployer still needs a password for any other
                   sudo usage not explicitly NOPASSWD-listed.
```

**Why `NOPASSWD: ALL` specifically is dangerous — not just
"less secure":** scoping `NOPASSWD:` to one exact command
(`/usr/bin/systemctl restart myapp`) means the account can do
*only* that one thing without a password — a compromised CI token
can restart one service and nothing else. `NOPASSWD: ALL` removes
the password requirement for *every* command, including a shell
(`sudo bash`), which means a compromised account with this rule
has unrestricted, unaudited-by-password root access — indistinguishable
from having the root password itself. The difference between the
two isn't "more vs less convenient," it's "one narrow capability"
vs "full system compromise if this one account leaks."

**How does belonging to `sudo`/`wheel` actually confer elevated access**

It is because of the one line in added to `/etc/sudoers`. The
group name `sudo` or `wheel` has no special meaning to the kernel
itself — they are ordinary groups, no different from `docker` or
`developers`. The *only* thing that makes them special is the
presence of a matching rule in `/etc/sudoers` that grants ALL
commands to ALL members of that group.

If you deleted the `%wheel ALL=(ALL) ALL` line from
`/etc/sudoers`, members of the `wheel` group would immediately
lose all sudo access — the group itself would become a plain,
meaningless group. This is also why adding a user to the `sudo`
group on Rocky9 does nothing: the `sudo` group has no matching
rule in Rocky9's `/etc/sudoers`, so belonging to it grants
nothing.

```bash
# Confirm which group actually has the rule on your system
grep -E "^%sudo|^%wheel" /etc/sudoers
# Ubuntu output:  %sudo   ALL=(ALL:ALL) ALL
# Rocky9 output:  %wheel  ALL=(ALL) ALL
```


**`/etc/sudoers.d/`** — the preferred way to add custom sudo rules
without editing the main file:

```bash
# Create a dedicated file for one purpose — always use visudo for this too
sudo visudo -f /etc/sudoers.d/deployer-ci

# Contents:
deployer ALL=(ALL) NOPASSWD: /usr/bin/systemctl restart myapp, /usr/bin/systemctl status myapp
```

**Why `/etc/sudoers.d/` is preferred over editing the main file:**

```
Editing /etc/sudoers directly:
  - One large file — risk of one mistake affecting all rules
  - Harder to track which rule was added for which purpose
  - Package upgrades sometimes overwrite or merge sudoers — risky

/etc/sudoers.d/ drop-in files:
  - One file per purpose — deployer-ci, jenkins-deploy, etc.
  - Easy to add/remove a specific permission without touching others
  - Can be version-controlled and deployed via automation per-file
  - This is the standard production pattern
```

`visudo -f /path/to/file` validates syntax for any file, not just
the main `/etc/sudoers` — always use it for `/etc/sudoers.d/` files
too, never a plain text editor.

### 1.5b What are elevated privileges, and why are they needed?

A Linux system assigns every process a set of **privileges** —
the set of operations it is permitted to perform. Normal user
processes run with restricted privileges by default, preventing
them from harming other users' files, modifying system
configuration, or accessing protected hardware.

**Elevated privileges** means temporarily or permanently gaining
a higher level of access than your normal user account has. The
most common reason: certain operations (modifying `/etc/`,
installing packages, managing services, reading `/etc/shadow`)
are restricted to root or to users explicitly granted permission
— a normal user simply cannot perform them without elevation.

**The privilege types you'll encounter in daily Linux work:**

```
Privilege type   Mechanism              When used
─────────────────────────────────────────────────────────────
Root (UID 0)     Running as root        Maximum access to
                 directly               everything — avoided
                                        in modern practice
                                        because a mistake or
                                        compromise has no
                                        boundary

sudo             Elevate one command    Standard for
                 at a time, as your     interactive admin
                 own logged-in          work — logs who ran
                 identity, governed     what, limits scope
                 by /etc/sudoers        per rule

su               Switch to another      Older pattern —
                 user entirely          opens a full shell
                 (often root) for       as that user, not
                 the duration of        one command at a
                 the session            time (covered in
                                        detail in section
                                        1.5b below)

Capabilities     Fine-grained kernel-   Service processes
                 level privileges       that need ONE
                 (CAP_NET_BIND_SERVICE, specific elevated
                 CAP_SYS_TIME, etc.)    action without full
                 assigned per-binary    root — beyond this
                 or per-process         demo's scope
```

Modern best practice: use `sudo` for interactive admin tasks,
design services to run as dedicated accounts with minimal
permissions (Demo 05's service account pattern), and avoid
running anything as root unless there is no alternative.

### 1.5c su — switching user identity

**`su`** = **s**ubstitute **u**ser (commonly misread as "super
user" — the actual meaning is substitute/switch user). It opens
a new shell running entirely as a different user for the duration
of the session, rather than running one command at an elevated
level.

```bash
su -          # switch to root — opens a ROOT SHELL
              # requires root's actual password
              # the - flag loads root's full environment too
              # (without - you get root's UID but your PATH/env)

su - alice    # switch to alice — opens alice's shell
              # requires alice's password (not yours)

su -c "command" alice   # run one command as alice, then return
                        # closest to how sudo works, but still
                        # requires the TARGET user's password
```

**`su` vs `sudo` — the practical differences:**

```
                su                        sudo
────────────────────────────────────────────────────────
What it does   Opens a new SHELL as       Runs ONE COMMAND
               a different user           with elevated
                                          privileges, stays
                                          as you

Password       Requires the TARGET        Requires YOUR OWN
               user's password            password (the user
               (e.g. root's password)     running sudo)

Logging        No built-in audit log      Logs every command
               of what was run            to auth.log/journal
               during the session         with username +
                                          timestamp + command

Access control Anyone who knows the       Governed by
               target's password          /etc/sudoers — fine
               can su to them             grained, per-command
                                          rules possible

Modern usage   Less common — root         Standard for
               passwords often not        interactive admin
               set/shared at all          work on modern
               on modern systems          Linux systems
```

**Why `su -` to root is mostly replaced by `sudo -i` today:**
distributing root's password across a team, or even setting one
at all, is considered poor practice on modern systems —
compromising one person's knowledge of the root password
compromises the whole system. `sudo` lets each administrator use
their own credentials, with individual accounts audited and
revocable independently.

```bash
sudo -i     # open a root shell via sudo — equivalent to su -,
            # but uses YOUR password + sudoers rules, not root's
            # password. The modern replacement for "su -".
```

### 1.6 The Ubuntu vs Rocky9 sudo group difference — critical distinction

**This is the single most common mistake engineers make moving
between Debian-family and RHEL-family systems.**

```
Ubuntu (Debian family):   sudo group grants sudo access
Rocky9 (RHEL family):     wheel group grants sudo access
                          (RHEL, CentOS, Fedora, Amazon Linux — all wheel)
```

```bash
# Ubuntu
sudo usermod -aG sudo deployer
grep sudo /etc/sudoers
# %sudo   ALL=(ALL:ALL) ALL

# Rocky9
sudo usermod -aG wheel deployer
grep wheel /etc/sudoers
# %wheel  ALL=(ALL) ALL
```

Adding a user to the `sudo` group on Rocky9 does **nothing** — that
group either does not exist or has no sudoers rule. The same mistake
in reverse — adding to `wheel` on Ubuntu — also does nothing.

**Why this exists historically:** `wheel` is the traditional Unix
group name dating back to BSD systems, carried forward into
RHEL/Fedora. Debian/Ubuntu chose `sudo` as a clearer, self-describing
name when `sudo` became the standard privilege escalation tool
(rather than full `su` to root, which `wheel` traditionally gated).
Neither is "correct" — they are simply different distro-family
conventions you must know.

**Always verify the group exists and has a sudoers rule on the
specific system you are working on:**

```bash
getent group sudo wheel 2>/dev/null     # show whichever exists
grep -E "^%sudo|^%wheel" /etc/sudoers   # show which has a sudoers rule
```

### 1.7 chage — password aging policy

**`chage`** = **ch**ange **age**. Views or sets the password aging
fields stored in `/etc/shadow` (section 1.1).

**Key flags:**

| Flag | Meaning | Use |
|---|---|---|
| `-l` | list | View current aging settings |
| `-M` | maximum days | Force periodic password rotation |
| `-m` | minimum days | Prevent rapid re-changes to dodge rotation |
| `-W` | warning days | Days of warning shown before expiry |
| `-d` | last-change date | Set to `0` to force change at next login |
| `-E` | expire date | Set an account expiry date (YYYY-MM-DD) |
| `-I` | inactive days | Lock account N days after password expires (no login) |

```bash
# View current aging policy for a user
sudo chage -l deployer

# Last password change                                    : Jun 30, 2026
# Password expires                                        : Sep 28, 2026
# Password inactive                                       : never
# Account expires                                         : Jul 30, 2026
# Minimum number of days between password change          : 7
# Maximum number of days between password change          : 90
# Number of days of warning before password expires       : 14
```

**Reading the output, field by field:**

```
Last password change     : Jun 30, 2026
```
The date the password was last set/changed. Your Task 6 output
showed this as `Jun 30, 2026` — the day you ran `passwd deployer`.

```
Minimum number of days between password change : 7
```
After changing their password, the user must wait 7 days before
they can change it again. Purpose: without this, a user forced
to change their password could immediately change it back to the
old one, satisfying the rotation policy superficially while
actually keeping the same password.
Example: set min=7, max=90. User changes to `NewPass1` on day 1.
On day 2, they try to change back to their old password — denied.
They must keep `NewPass1` for at least 7 days.

```
Maximum number of days between password change : 90
```
The password expires 90 days after `Last password change`. At
that point, the user is forced to change their password at next
login. Setting 99999 (the default if you don't set it) means
effectively never — 99999 days ≈ 273 years.

```
Number of days of warning before password expires : 14
```
14 days before the password would expire, the user sees a warning
at login: "Your password will expire in N days." This gives them
time to change proactively rather than being surprised by a
forced change.

**What `Account expires` actually means:**

When the account expiry date is reached, the account becomes
**completely unusable** — not just password-locked. No password
will work, no SSH key will work, no `su` to that account will
succeed. The account ceases to exist as far as authentication is
concerned, even though its files and `/etc/passwd` entry remain
completely intact on disk.

This is different from a password expiry (which forces a password
change but still allows login) and from a locked password (which
blocks password authentication but may allow SSH key login).

```
Account expires:       hard cutoff — NO authentication of any kind
Password expires:      forced password change — login still works
Password locked (!):   blocks PASSWORD auth — SSH keys still work
```

Practical use: set account expiry (`chage -E YYYY-MM-DD`) for
contractor accounts with a known end date. When that date arrives,
the account automatically becomes unusable without anyone needing
to remember to disable it manually.


**Command Example:**

```bash
# Set maximum password age — force change every 90 days
sudo chage -M 90 deployer

# Set minimum days before a password can be changed again
# Prevents users from immediately changing back to avoid expiry rotation
sudo chage -m 7 deployer

# Set warning period
sudo chage -W 14 deployer

# Force password change on next login
sudo chage -d 0 deployer
# Sets last-change date to epoch (day 0) — system treats password
# as already expired, forces change at next login

# Set an account expiry date — useful for contractor/temp accounts
sudo chage -E 2026-12-31 deployer

# Interactive mode — prompts for each field
sudo chage deployer
```



**Practical use case:** contractor or temporary access — set `-E`
to the contract end date. The account becomes unusable automatically
without anyone needing to remember to disable it manually.

### 1.8 Service accounts — the correct pattern

A **service account** is a user created to run a service or process,
never for interactive human login.

**What makes a correct service account:**

```bash
sudo useradd -r -s /usr/sbin/nologin -d /opt/myapp -c "myapp service account" myapp-svc
# -r  system account — UID from the system range (below 1000 on
#     most distros), no password aging policy applied
# -s  /usr/sbin/nologin — this is the critical flag. If anyone
#     tries to log in as this user (ssh, su, login), they are
#     immediately rejected with a message and disconnected
# -d  home directory — often the application's install/data dir,
#     not /home/
# -c  comment describing the account's purpose
```

**Why `/usr/sbin/nologin` matters:**

```bash
su - myapp-svc
# This account is currently not available.
```

Without this, a service account with no shell restriction can still
be logged into directly — by anyone who obtains its password or an
SSH key, or by any process running with sufficient privilege to `su`
into it. Setting the shell to `nologin` closes this off entirely:
the account can still **run** processes (systemd starts the service
as this user via `User=myapp-svc` in the unit file — covered fully
in Demo 07) but cannot be used as an interactive login target.

**No password should be set either** — confirm with:

```bash
sudo passwd -S myapp-svc
# myapp-svc L 2026-01-15 0 99999 7 -1
#            ↑
#            L = locked (no usable password — correct for service accounts)
#            P = usable password set
#            NP = no password set at all (also acceptable, less explicit)
```

```bash
# Explicitly lock the account's password if not already locked
sudo passwd -l myapp-svc
```

**Why this pattern matters for security:** every process should run
as the least-privileged identity that can do its job. A web server
running as a dedicated `www-data`-style account, rather than root
or a human's account, limits the damage if that process is
compromised — the attacker inherits only that service account's
limited permissions, not a human's full access or root.

**Does a service account need a password or a group?**

**Password:** No. A service account should have no usable
password. On account creation with `useradd -r -s /usr/sbin/nologin`,
the password field in `/etc/shadow` is set to `!!` — meaning no
password has ever been set. This is correct and intentional.
`passwd -l` explicitly locks it (prepends `!` to any existing
hash), giving `L` in `passwd -S` output. Either `!!` (never set)
or `!hash` (locked) achieves the goal of no password
authentication — `passwd -l` is belt-and-suspenders for
accounts that might have had a password previously.

**Group:** A group IS automatically created — the UPG scheme
(section 1.3, first introduced for regular users) applies to
service accounts exactly the same way. When you run `useradd -r
novatech-svc` with no `-g` flag, a new group named `novatech-svc`
is created with a GID from the system range, set as the account's
primary group. `-r` changes *where the UID/GID numbers come from*
(the system range instead of 1000+) — it does not disable UPG or
create a fundamentally different kind of account. You saw this in
Task 7:
```
uid=999(novatech-svc) gid=988(novatech-svc) groups=988(novatech-svc)
```
The UPG group (988/novatech-svc) was created automatically. The
UID and GID are different here (999 vs 988) because the system
assigned the next available IDs from the system range separately
for users and groups — this is expected behavior.

---

## Part 2 — Hands-On Lab

### Commands used in this demo

| Command | Flag meaning | Purpose |
|---|---|---|
| `useradd` | — | Create a new user |
| `usermod` | — | Modify an existing user |
| `userdel` | — | Delete a user |
| `passwd` | — | Set or manage a user's password |
| `chage` | — | View/set password aging policy |
| `visudo` | — | Safely edit sudoers files with syntax validation |
| `id` | — | Show UID, GID, and all groups for a user |
| `getent` | — | Query system name service databases (passwd, group, shadow, etc.) |
| `su` | — | Switch to a different user identity |

**Key flags used in this demo:**

| Flag | Command | Flag meaning | Meaning |
|---|---|---|---|
| `-m` | `useradd` | make home | Create home directory |
| `-s` | `useradd`/`usermod` | shell | Set login shell |
| `-c` | `useradd`/`usermod` | comment (GECOS) | Set description field |
| `-r` | `useradd` | system (reserved range) | Create system account with UID < 1000 |
| `-G` | `useradd`/`usermod` | groups | Set supplementary groups (REPLACES existing) |
| `-a` | `usermod` | append | Append to groups (use with -G, never without) |
| `-aG` | `usermod` | append + groups | Safely add to a group without replacing others |
| `-r` | `userdel` | remove | Remove home directory and mail spool too |
| `-l` | `passwd` | lock | Lock password (prepend ! to hash) |
| `-S` | `passwd` | status | Show password status |
| `-M` | `chage` | maximum days | Set maximum password age |
| `-m` | `chage` | minimum days | Set minimum days before change allowed |
| `-W` | `chage` | warn days | Set warning period before expiry |
| `-d` | `chage` | date (last change) | Set last password change date |
| `-E` | `chage` | expire (account) | Set account expiry date |
| `-l` | `chage` | list | List current aging settings |
| `-f` | `visudo` | file | Edit a specific sudoers file (not just /etc/sudoers) |
| `-u` | `sudo` | user | Run command as a specific user |
| `-i` | `sudo` | interactive (login shell) | Open a login shell as the target user |
| `-l` | `sudo` | list | List allowed commands for current or specified user |

---

### Task 1 — Inspect the three identity files

**Run on Ubuntu, then Rocky9:**

```bash
# Your own entry in each file
grep "^$(whoami):" /etc/passwd
sudo grep "^$(whoami):" /etc/shadow
grep "^$(whoami):" /etc/group

# Full identity
id

# Use getent instead of grep — works even with non-file backends
# (LDAP, NIS) because it queries the configured name service, not
# just the local file
getent passwd $(whoami)
getent group sudo 2>/dev/null || getent group wheel 2>/dev/null
```

**Reading your own `/etc/shadow` line** — identify each field per
the breakdown in section 1.1. Note the hash algorithm prefix
(`$6$` = SHA-512, standard on modern distros).


**Reading actual `id` output:**

Ubuntu:
```
uid=1000(wadmin) gid=1000(wadmin) groups=1000(wadmin),4(adm),24(cdrom),27(sudo),30(dip),46(plugdev),100(users),989(docker)
```
- `uid=1000` — regular user (1000+ = human account, per UPG convention)
- `gid=1000(wadmin)` — primary group is your UPG group
- `groups=...` — the full list: your primary group first, then all
  supplementary groups the kernel uses when evaluating file permissions.
  `27(sudo)` confirms sudo group membership → sudo access on Ubuntu.
  `989(docker)` → can use Docker socket without sudo.

Rocky9:
```
uid=1000(wadmin) gid=1000(wadmin) groups=1000(wadmin),10(wheel)
```
Same user, different system — supplementary groups are minimal
(no docker, no cdrom etc.) because Rocky9 is a server install.
`10(wheel)` → wheel group membership → sudo access on Rocky9.

---

### Task 2 — Create a user and observe UPG

**Run on Ubuntu, then Rocky9:**

```bash
# Create the user — no -g flag, so UPG applies
sudo useradd -m -s /bin/bash -c "Deploy Service Account" deployer
sudo passwd deployer
# Set a password when prompted — used later in this demo

# Observe the UPG result
id deployer
grep deployer /etc/passwd
grep deployer /etc/group
# Note: primary GID matches a NEW group also named "deployer"

# Confirm home directory was created
ls -ld /home/deployer
```

**Paste the `id deployer` output — confirm the UPG group exists
and matches the username.**

**Now confirm `su` actually works for a normal account — the
positive case, not just the nologin failure you'll see in Task 7:**

```bash
su - deployer
# Prompts for the password you just set. On success, you get a
# genuinely new shell running as deployer:
whoami        # deployer
id            # matches deployer's own id, not yours
pwd           # /home/deployer — deployer's own home, not yours
exit          # back to your own shell
```

**Paste the `whoami`/`id` output from inside the `su - deployer`
session — compare it against your own `id` from Task 1.**

---

### Task 3 — The `-aG` vs `-G` danger, demonstrated live

**Run on Ubuntu, then Rocky9:**

```bash
# Set up a starting state — add deployer to two groups correctly first
# Ubuntu:
sudo usermod -aG sudo deployer
sudo usermod -aG docker deployer 2>/dev/null || true   # docker group may not exist on Rocky9

# Rocky9:
sudo usermod -aG wheel deployer

# Confirm starting state
id deployer
# Ubuntu:  groups=...(deployer),27(sudo),998(docker)
# Rocky9:  groups=...(deployer),10(wheel)

# --- Now demonstrate the danger ---
# WRONG — bare -G replaces the ENTIRE supplementary group list
sudo usermod -G docker deployer    # (Ubuntu) or pick any single other group on Rocky9

id deployer
# sudo/wheel membership is GONE — only docker (or whichever group) remains
# This is the incident: deployer silently lost sudo access

# --- Fix it ---
# Ubuntu:
sudo usermod -aG sudo deployer
# Rocky9:
sudo usermod -aG wheel deployer

id deployer
# Confirm both groups are present again
```

**Paste the `id deployer` output at three points: before, during
the incident (after the wrong command), and after the fix.**

---

### Task 4 — sudo group: Ubuntu vs Rocky9

**`groups`** — lists group memberships for a user in a
simple space-separated format, less detail than `id` but
easier to read at a glance:

```bash
groups                  # your own groups
groups username         # another user's groups
```

Compare: `id` shows UID, GID, and all groups with GID numbers.
`groups` shows only the group names, no numbers.

**Run on Ubuntu:**

```bash
# Confirm the sudo group has a sudoers rule
sudo grep -E "^%sudo" /etc/sudoers

# Confirm deployer is in the sudo group (from Task 3)
groups deployer

# Test as deployer
sudo -u deployer sudo whoami
# Should prompt for deployer's password, then print: root
```

**Reading `sudo -u deployer sudo whoami` — why two `sudo`s?**

```bash
sudo -u deployer sudo whoami
  ↑                ↑    ↑
  │                │    └─ print the effective username
  │                └─ the SECOND sudo: run with elevated
  │                   privileges, AS deployer — this tests
  │                   that deployer's own sudo access works
  └─ the FIRST sudo: run the following command AS deployer,
     using YOUR (wadmin's) sudo credentials to switch identity
```

In plain English: "Use MY sudo credentials to become deployer,
then use DEPLOYER's sudo credentials to become root, and print
who we are." If both succeed, you've confirmed: (1) your account
can impersonate deployer, and (2) deployer's sudo group
membership is working correctly.

In real production use you'd more simply use `su - deployer` and
then test, but this double-sudo approach lets you verify without
actually needing deployer's password.

**Run on Rocky9:**

```bash
# Confirm the wheel group has a sudoers rule
sudo grep -E "^%wheel" /etc/sudoers

# Confirm deployer is in the wheel group (from Task 3)
groups deployer

# Test as deployer
sudo -u deployer sudo whoami
# Should prompt for deployer's password, then print: root

# Now prove the mistake — try the WRONG group name for this distro
sudo usermod -aG sudo deployer 2>&1
# Ubuntu's group name does not exist on Rocky9 by default —
# observe the actual error or behaviour on your system
```

**Paste the `grep` output from both systems — note the different
group name in each sudoers file.**

---

### Task 5 — visudo and a sudoers.d drop-in

**Run on Ubuntu, then Rocky9:**

```bash

# Confirm deployer CANNOT run the command yet
sudo -u deployer sudo systemctl status cron
# Expected: "Sorry, user deployer is not allowed to execute..."
# If it succeeds here, deployer already has broad sudo access
# from an earlier step — reset by removing from sudo/wheel group
# and re-test after adding the specific sudoers.d rule


# Create a dedicated sudoers.d file for deployer — one specific
# command, no password required (typical CI/CD pattern)
sudo visudo -f /etc/sudoers.d/deployer-ci

# Add this line in the editor that opens:
# deployer ALL=(ALL) NOPASSWD: /usr/bin/systemctl restart cron, /usr/bin/systemctl status cron

# Save and exit — visudo validates syntax automatically

# Verify the file was created correctly
sudo cat /etc/sudoers.d/deployer-ci

# Test — should NOT prompt for a password
sudo -u deployer sudo systemctl status cron

# Test a command NOT in the allowed list — should fail
sudo -u deployer sudo systemctl restart sshd
# deployer is not allowed to run this command
```

**Now deliberately break syntax to see visudo's protection:**

```bash
sudo visudo -f /etc/sudoers.d/broken-test
# Type intentionally broken syntax, e.g.: deployer ALL=(ALL
# (missing closing paren and command)
# Try to save — visudo should refuse and show a parse error,
# offering to re-edit, save anyway (don't), or abandon changes

# Choose to abandon — confirm no broken file was created
ls /etc/sudoers.d/broken-test 2>&1
```

**Paste the validation error message visudo showed you.**

---

### Task 6 — chage password aging

**Run on Ubuntu, then Rocky9:**

```bash
# View deployer's current aging policy
sudo chage -l deployer

# Set a 90-day maximum password age and 7-day minimum
sudo chage -M 90 -m 7 -W 14 deployer

# Confirm the change
sudo chage -l deployer

# Set an account expiry date — simulating a contractor with a fixed end date
sudo chage -E $(date -d "+30 days" +%Y-%m-%d) deployer 2>/dev/null || \
  sudo chage -E $(date -v+30d +%Y-%m-%d) deployer    # macOS-style date fallback if needed

sudo chage -l deployer
# Confirm "Account expires" now shows a date 30 days out

# Force a password change at next login
sudo chage -d 0 deployer
sudo chage -l deployer
# "Last password change" should show the epoch date
```

**Reading your `chage -d 0` output:**

After `chage -d 0 deployer`:
```
Last password change    : password must be changed
Password expires        : password must be changed
Password inactive       : password must be changed
Account expires         : Jul 30, 2026
```

`chage -d 0` sets the `lastchange` field in `/etc/shadow` to 0
(Unix epoch = January 1, 1970). Since maxage is 90 days and the
"last change" is now 56 years ago, the password is immediately
and maximally expired — the system can't even calculate a
meaningful expiry date, so it shows `password must be changed`
for all password-related fields.

`Account expires` is unchanged (`Jul 30, 2026`) — that is an
independent field. `chage -d 0` only affects the password aging
fields, not the account expiry date.

The user won't be locked out — they can still log in, but will
be immediately forced to set a new password before getting a
shell prompt.

---

### Task 7 — Create a correct service account

**Run on Ubuntu, then Rocky9:**

```bash
# Create the service account
sudo useradd -r -s /usr/sbin/nologin -d /opt/novatech-app -c "NovaTech App service account" novatech-svc

# Confirm system-range UID (below 1000 on most distros)
id novatech-svc
grep novatech-svc /etc/passwd

# Confirm the account cannot be used for interactive login
su - novatech-svc
# Should show: This account is currently not available.

# Confirm password status — should be locked, not usable
sudo passwd -S novatech-svc

# Explicitly lock it if not already locked
sudo passwd -l novatech-svc
sudo passwd -S novatech-svc
```

**Reading your Task 7 output:**

```
uid=999(novatech-svc) gid=988(novatech-svc) groups=988(novatech-svc)
```
UID 999 and GID 988 are both from the system range (below 1000),
as expected for a `useradd -r` account. The UID and GID are
different (999 vs 988) because the system assigns the next
available number from the system range separately for users and
groups — this is normal, not an error.

```
su - novatech-svc
Password:
su: Authentication failure
```
Note that `su` prompted for a password even though `/usr/sbin/nologin`
is the shell. This is because `su` goes through PAM (Pluggable
Authentication Modules) and checks the password first, before
even attempting to start the shell. The `Authentication failure`
here is the password check failing (no usable password set — `!!`
in `/etc/shadow`), not the nologin shell blocking it.

If the account DID have a valid password, `su` would still reject
the login because `/usr/sbin/nologin` outputs "This account is
currently not available." and exits immediately — the shell that
`su` tries to start refuses to run.

```
sudo passwd -S novatech-svc
novatech-svc L 2026-06-30 -1 -1 -1 -1
```
Status `L` = Locked. The `-1` values in the aging fields mean
"feature disabled" — no minimum age, no maximum age, no warning
period, no inactive grace period. Correct for a service account
that should never be interactively managed through password aging
policies.

```
sudo passwd -l novatech-svc
passwd: password changed.
```
```
sudo passwd -S novatech-svc
novatech-svc L 2026-06-30 -1 -1 -1 -1
```
Status was already `L` before and remains `L` after — because
`useradd -r` created the account with `!!` (never set), which
already reads as locked. Running `passwd -l` on an already-locked
account is a no-op status-wise, but makes the intent explicit in
the hash field (changes `!!` to `!` + a lock marker) so future
admins can tell it was deliberately locked, not just never set up.

---

## What You Learned

- ✅ Read and interpreted every field in `/etc/passwd`, `/etc/shadow`, and `/etc/group`
- ✅ Created, modified, and removed users with `useradd`, `usermod`, `userdel`
- ✅ Explained the difference between a primary group and supplementary groups, and the User Private Group scheme
- ✅ Identified and avoided the `usermod -G` vs `-aG` danger
- ✅ Granted sudo access correctly on Ubuntu (`sudo` group) and Rocky9 (`wheel` group)
- ✅ Edited sudo configuration safely with `visudo` and `/etc/sudoers.d/`
- ✅ Set password aging policy with `chage`
- ✅ Created a correct service account with no login shell and a locked password
- ✅ Explained `su` vs `sudo` — the difference between switching user identity for a whole session and elevating one command at a time — and demonstrated both the successful case (Task 2) and the blocked case (Task 7)

**Key Takeaway:** Almost every user/group incident in production traces back to one of three things — a missing `-a` flag, the wrong sudo group name for the distro family, or a service account that was created without `nologin`. All three are one-line fixes once you know what to check.

---

## Part 3 — Break-Fix Scenario

### Scenario — NovaTech Ops: two access incidents

> *"Incident 1: Our CI/CD admin ran a usermod command to add the
> deployer account to a new monitoring group on Ubuntu. Now the
> deployment pipeline fails — deployer can no longer run sudo
> commands. Nobody removed deployer from sudo intentionally."*

> *"Incident 2: A new hire who came from an Ubuntu background set
> up a fresh Rocky9 server and tried to grant themselves sudo
> using the same command they always use. sudo still asks for a
> password they don't have and then denies them. They are stuck
> outside the system with no way to gain admin access."*

**Investigation — Incident 1:**

```bash
# Ubuntu — current state
id deployer
# uid=1002(deployer) gid=1002(deployer) groups=1002(deployer),998(monitoring)

# Check sudoers — sudo group rule still exists
grep "^%sudo" /etc/sudoers
# %sudo   ALL=(ALL:ALL) ALL

# Check command history (if available) or ask the admin what was run
# Admin confirms they ran:
# sudo usermod -G monitoring deployer
```

**Investigation — Incident 2:**

```bash
# Rocky9 — new hire ran:
sudo usermod -aG sudo newhire
id newhire
# uid=1003(newhire) gid=1003(newhire) groups=1003(newhire)
#                                              ↑
#                                              sudo group membership
#                                              did not take effect

getent group sudo
# (no output — sudo group does not exist on this Rocky9 system)

getent group wheel
# wheel:x:10:
```

**Your tasks:**

1. For Incident 1: what exactly went wrong, and what is the
   single command that restores deployer's access without
   removing the monitoring group membership they were supposed
   to gain?

2. For Incident 2: why did the `usermod -aG sudo` command not give
   the new hire sudo access on Rocky9, even though the syntax was
   correct and they used `-a` properly?

3. The new hire in Incident 2 is now locked out — no working sudo
   access on a fresh system. If they still have their own login
   (just no root access), what is the correct command to fix this?
   If they have no working login at all, what does this tell you
   about why root access recovery procedures matter operationally?

<details>
<summary>Resolution — open after attempting</summary>

**Incident 1 — root cause:**

The admin ran `usermod -G monitoring deployer` — bare `-G` without
`-a`. This replaced deployer's entire supplementary group list with
just `monitoring`, wiping out the existing `sudo` (and any other)
group memberships in the process. The admin's intent was correct —
add `monitoring` — but the missing `-a` flag caused a silent
replacement instead of an addition.

**Fix:**

```bash
sudo usermod -aG sudo,monitoring deployer
```

Using `-aG` with a comma-separated list adds both groups while
preserving every existing supplementary group deployer already had.
Always verify with `id deployer` before and after any group change
to confirm the result matches intent.

**Incident 2 — root cause:**

The command syntax was correct (`-aG` was used properly), but
`sudo` is the **Ubuntu/Debian** group name. On Rocky9 (RHEL family),
the group that grants sudo access is `wheel`. The `sudo` group
simply does not exist on Rocky9 by default, so `usermod -aG sudo
newhire` either fails outright or silently does nothing useful —
the new hire was added to a meaningless group reference, not the
group that actually has a sudoers rule (`%wheel`).

**Fix command, if they retain a working login:**
```bash
sudo usermod -aG wheel newhire
# Requires existing sudo/root access to run — this is the catch
```

**The deeper lesson (question 3):**

If the new hire's account has **no** existing sudo/root access and
no other path to root (no separate root password set, no console
access, no alternate admin account), this is a genuine lockout —
the fix command itself requires the very access that is missing.

This is exactly why production systems maintain at least one
alternate path to root: a documented root password kept in a secure
vault, cloud provider console/serial access (EC2 Instance Connect,
recovery mode), or a configuration management tool (Ansible,
Terraform user-data) that can re-provision access without needing
interactive sudo. Relying on a single human account's sudo group
membership as the *only* path to root is itself the underlying
risk this incident exposes — group naming mistakes are recoverable
only if a second path to privilege exists.

</details>

---

## Interview Prep

**Q1. What is the difference between `/etc/passwd` and
`/etc/shadow`? Why are they separate files?**

`/etc/passwd` stores user account information — UID, GID, home
directory, shell — and must remain world-readable because many
tools (like `ls -l` showing usernames) need to read it. `/etc/shadow`
stores the actual password hash and aging policy, and is readable
only by root. They were split decades ago specifically so that
password hashes are not exposed to every user on the system, which
was a real vulnerability when both lived in one world-readable file.

---

**Q2. A teammate runs `usermod -G docker alice` intending to add
Alice to the docker group. What is wrong with this command, and
what is the safe version?**

Bare `-G` replaces Alice's **entire** supplementary group list with
exactly what is specified — in this case, just `docker`. Any other
group Alice belonged to (sudo, wheel, other project groups) is
silently removed. The safe version is `usermod -aG docker alice`
— the `-a` flag means append, preserving existing group membership
while adding the new one. Always run `id alice` before and after to
confirm the result.

---

**Q3. You SSH into a fresh Rocky Linux 9 server and run
`usermod -aG sudo myuser`, expecting sudo access. It does not work.
Why?**

`sudo` is the Debian/Ubuntu convention for the group that grants
sudo privileges. RHEL-family distros (Rocky Linux, RHEL, CentOS,
Amazon Linux, Fedora) use `wheel` instead. The `sudo` group likely
does not exist on this system, or if it does, has no corresponding
rule in `/etc/sudoers`. The fix is `usermod -aG wheel myuser`. This
is one of the most common mistakes engineers make moving between
distro families, and worth verifying explicitly with
`grep -E "^%sudo|^%wheel" /etc/sudoers` on any unfamiliar system.

---

**Q4. Why should a service account use `/usr/sbin/nologin` as its
shell instead of `/bin/bash`?**

A service account exists only to run a process — it should never
be an interactive login target. Setting the shell to
`/usr/sbin/nologin` means any attempt to log in as that account
(via `ssh`, `su`, or direct console login) is immediately rejected.
The account can still run processes — `systemd` starts services as
this user via the unit file's `User=` directive, which does not go
through the login shell at all. This limits the blast radius if the
account's credentials are ever compromised: an attacker cannot get
an interactive shell as that user, only whatever the service process
itself is doing.

---

## Key Takeaways

1. **Know the three files that define every user's identity: `/etc/passwd` (account info, world-readable), `/etc/shadow` (password hash + aging, root-only), `/etc/group` (group membership).** `/etc/passwd` has no passwords despite its name — they moved to `/etc/shadow` specifically because `/etc/passwd` must stay world-readable for tools like `ls -l` to resolve usernames.

2. **`usermod -aG group username` adds to existing groups; bare `usermod -G group username` replaces the entire supplementary group list.** A missing `-a` silently wipes every other group membership — no warning, no error. Always verify with `id username` before and after any group change.

3. **The `sudo` group (Ubuntu/Debian) and `wheel` group (RHEL/Rocky/Amazon Linux) both grant sudo access only because of a matching rule in `/etc/sudoers` — the group names themselves have no special kernel meaning.** Adding a user to `sudo` on Rocky9 or `wheel` on Ubuntu does nothing useful; always verify with `grep -E "^%sudo|^%wheel" /etc/sudoers` on any unfamiliar system.

4. **Always use `visudo` (or `visudo -f` for drop-in files in `/etc/sudoers.d/`) to edit sudoers files — never a plain text editor.** A syntax error anywhere in `/etc/sudoers` causes the entire file to fail, removing all sudo access system-wide including your own. `visudo` validates before saving and refuses to write invalid syntax. `/etc/sudoers.d/` drop-in files are preferred over editing the main file for the same reason `/etc/cron.d/` and `/etc/systemd/system/` favor drop-ins: one file per purpose, easy to track and remove.

5. **`chage -d 0 username` forces a password change at next login; `chage -E YYYY-MM-DD` sets a hard account expiry date.** Password expiry forces a change but allows login; account expiry locks the account completely regardless of password state — three distinct lockout mechanisms: maxage, inactive grace period, and account expiry.

6. **A service account's two mandatory properties are `/usr/sbin/nologin` as its shell and no usable password (`passwd -S` showing `L` or `NP`).** `nologin` blocks interactive login but has zero effect on a process started by systemd — the running process is fully visible to `ps`, `kill`, and `renice` exactly like any other process.

7. **Ubuntu 22.04+ (including 24.04) uses yescrypt (`$y$`) for password hashing by default, not SHA-512 (`$6$`).** Rocky Linux 9 still uses SHA-512. Check your system's default with `grep ENCRYPT_METHOD /etc/login.defs`. Yescrypt is deliberately memory-intensive, making GPU brute-force attacks orders of magnitude more expensive than SHA-512.

8. **`su` switches your entire identity for a session and requires the target user's password; `sudo` elevates one command at a time using your own credentials, governed by `/etc/sudoers`.** Modern systems prefer `sudo` because it doesn't require distributing root's password, audits every command with username and timestamp, and access can be revoked per-user without affecting others.

---

## Quick Commands Reference

### User management

| Command | Purpose |
|---|---|
| `useradd -m -s SHELL -c "comment" name` | Create user with home dir |
| `useradd -r -s /usr/sbin/nologin name` | Create service account |
| `usermod -aG group name` | Add user to a group (safe) |
| `usermod -G group name` | Replace all supplementary groups (dangerous) |
| `usermod -s SHELL name` | Change login shell |
| `userdel name` | Delete user, keep home directory |
| `userdel -r name` | Delete user and home directory |

### Identity inspection

| Command | Purpose |
|---|---|
| `id` | Full identity: UID, GID, all groups |
| `id username` | Identity of a specific user |
| `whoami` | Current effective username only |
| `groups username` | List a user's groups |
| `getent passwd username` | Query passwd entry via name service |
| `getent group groupname` | Query group entry via name service |

### sudo and sudoers

| Command | Purpose |
|---|---|
| `visudo` | Safely edit `/etc/sudoers` with validation |
| `visudo -f /path/to/file` | Safely edit any sudoers-format file |
| `sudo -l` | List what the current user is allowed to run |
| `sudo -l -U username` | List what another user is allowed to run |

### Password and aging

| Command | Purpose |
|---|---|
| `passwd username` | Set/change a user's password |
| `passwd -l username` | Lock a password (disable login via password) |
| `passwd -S username` | Show password status (L/P/NP) |
| `chage -l username` | List current aging policy |
| `chage -M days username` | Set maximum password age |
| `chage -E YYYY-MM-DD username` | Set account expiry date |

### Distro-family sudo group reference

| Distro family | sudo-granting group |
|---|---|
| Debian / Ubuntu | `sudo` |
| RHEL / Rocky / CentOS / Fedora / Amazon Linux | `wheel` |

---

## Appendix A — Anki Cards

**`05-users-groups-anki.csv`:**

```
#deck:Linux Mastery::Module 2 - System Administration::Demo 05 - Users, Groups & Authentication
#separator:Comma
#columns:Front,Back,Tags

"A teammate runs usermod -G docker alice intending to add her to the docker group. What goes wrong and what is the safe command?","Bare -G REPLACES the entire supplementary group list with exactly what is specified — alice loses every other group she belonged to (sudo, wheel, etc.), keeping only docker. Safe version: usermod -aG docker alice — -a appends, preserving existing groups. Always verify with id alice before and after.","demo05,usermod,danger"

"You add a user to the sudo group on Rocky Linux 9 and sudo still does not work. Why?","sudo is the Debian/Ubuntu group name for sudo access. RHEL-family distros (Rocky, RHEL, CentOS, Amazon Linux, Fedora) use wheel instead. The sudo group likely does not exist or has no sudoers rule on this system. Fix: usermod -aG wheel username. Verify with: grep -E '^%sudo|^%wheel' /etc/sudoers","demo05,sudo,wheel,distro"

"Why do you always use visudo instead of editing /etc/sudoers directly with a text editor?","visudo validates syntax before saving and refuses to save invalid syntax. A direct text editor save with a syntax error can lock out ALL sudo access on the system, including your own, with no easy recovery if no other root path exists. visudo also works on /etc/sudoers.d/ files via visudo -f /path/to/file.","demo05,visudo,sudoers,safety"

"What is the User Private Group (UPG) scheme? What GID does a new user get by default?","When useradd creates a user with no -g flag, it also creates a new group with the same name and GID as the username, and sets that as the user's primary group. Example: useradd deployer creates both user deployer (UID 1002) and group deployer (GID 1002), with deployer's primary GID = 1002. This means a user's primary group typically matches their username on standard installs.","demo05,upg,groups"

"What makes a service account different from a regular user account? Name two required properties.","Two required properties: (1) Shell set to /usr/sbin/nologin — rejects any interactive login attempt (ssh, su, console) while still allowing systemd to start processes as this user via the unit file's User= directive. (2) No usable password — confirmed via passwd -S showing L (locked) or NP (no password). Typically also created with useradd -r for a system-range UID.","demo05,service-account,security"

"/etc/passwd shows wadmin:x:1000:1000:Wadmin User:/home/wadmin:/bin/bash. What does the x represent and where is the actual password data?","The x is a placeholder meaning 'see /etc/shadow for the password'. /etc/passwd must remain world-readable for many tools to function (e.g. displaying usernames), but storing password hashes in a world-readable file would be a serious vulnerability. /etc/shadow, readable only by root, stores the actual hash plus aging fields (last change, min/max age, warning days, expiry).","demo05,passwd,shadow,security"

"chage -l deployer shows 'Account expires: never'. How do you set this account to expire automatically in 30 days, useful for a contractor?","chage -E YYYY-MM-DD deployer — sets a hard account expiry date. After that date, the account becomes unusable automatically without anyone needing to remember to manually disable it. Useful for contractor or temporary access where the end date is known in advance.","demo05,chage,aging,contractor"

"What is the difference between a user's primary group and supplementary groups? Which file records each?","Primary group: exactly one per user, recorded in /etc/passwd (the GID field), assigned to new files the user creates by default. Supplementary groups: zero or more, recorded in /etc/group (each group's members field), grant additional access (sudo, wheel, docker, etc.). The kernel checks ALL of a user's groups — primary and supplementary — when evaluating group-based file permissions, not just the primary one.","demo05,groups,primary,supplementary"

"What is the GECOS field in /etc/passwd, and what does GECOS stand for?","GECOS stands for General Electric Comprehensive Operating System — a 1960s mainframe OS that early Unix systems interfaced with, requiring a contact/account field in /etc/passwd. On modern Linux it is simply a free-text comment field, typically used for a user's full name or a service account's purpose description. Format: comma-delimited (Full name, Room, Work phone, Home phone, Other) — most systems only fill the first subfield. Change with usermod -c or chfn.","demo05,gecos,passwd"

"What does $y$ mean at the start of a hash in /etc/shadow? What about $6$?","$y$ = yescrypt — the DEFAULT on Ubuntu 22.04 and later (including 24.04). $6$ = SHA-512 — the default on Rocky/RHEL 8+ and Ubuntu 14.04-20.04. Check your system with: grep ENCRYPT_METHOD /etc/login.defs. Yescrypt is deliberately slow and memory-intensive, making GPU brute-force attacks orders of magnitude more expensive than SHA-512.","demo05,shadow,hashing,yescrypt"

"What is the difference between su and sudo? When should each be used?","su (substitute user) opens a full shell as another user for the duration of the session and requires the TARGET user's password. sudo (superuser do) runs ONE command with elevated privileges using YOUR OWN password, governed by /etc/sudoers rules, and logs every command with username+timestamp. Modern practice: use sudo for interactive admin tasks (sudo -i replaces su -), never distribute root passwords, revoke access per-user by removing from sudoers/wheel group.","demo05,su,sudo,privileges"

"A syntax error on one line of /etc/sudoers — does it break just that rule or the entire file?","The entire file. sudo parses the whole file before every command. If it finds ANY syntax error anywhere, it refuses to honour ANY rule in the entire file — including rules for other users that are syntactically correct. Result: all sudo access is removed system-wide until the syntax is fixed. This is why visudo is mandatory — it validates syntax before saving and refuses to write an invalid file.","demo05,sudoers,visudo,syntax"

"What is the difference between password inactive, password expiry, and account expiry in /etc/shadow?","Password expiry (maxage): forces a password CHANGE at next login — doesn't lock you out, just demands a new password. Password inactive: grace period AFTER password expires — if the user doesn't log in during this window, the account locks entirely (no more 'change your password' prompt). Account expiry (chage -E): hard date when the ACCOUNT becomes completely unusable regardless of password state — no authentication method works. Three independent lockout mechanisms.","demo05,chage,shadow,password-aging"
```

---

## Appendix B — Quiz

**`05-users-groups-quiz.md`:**

````markdown
# Quiz — Demo 05: Users, Groups & Authentication

> One correct answer per question unless stated otherwise.
> Target: 80% or above before moving to Demo 06.

---

**Q1.** A sudoers rule reads `deployer ALL=(ALL) NOPASSWD:
/usr/bin/systemctl restart myapp`. A teammate wants to "make it
easier" and changes it to `deployer ALL=(ALL) NOPASSWD: ALL`. What's
the actual risk of this change, beyond "it's less secure"?

- A) No real difference — both let deployer restart myapp without a password
- B) The scoped rule lets a compromised `deployer` account do exactly
  one thing (restart myapp) with no password; `NOPASSWD: ALL` removes
  the password requirement for *every* command, including a full
  shell (`sudo bash`) — a compromised account becomes equivalent to
  having the root password itself
- C) `NOPASSWD: ALL` is invalid syntax and `visudo` would reject it
- D) The change only affects logging, not actual access

<details>
<summary>Answer</summary>

**B** — This is the real, concrete difference: one narrow capability
that's harmless if the account leaks, versus unrestricted root
access with no password barrier at all if it leaks. "Less secure"understates it —
it's the difference in the actual blast radius of a compromise.

</details>

---

**Q2.** In the sudoers line `%wheel ALL=(ALL) ALL`, identify what
each of the four components represents, in order.

- A) who=%wheel (the group), where=ALL (any host), as_whom=(ALL)
  (run as any user), what=ALL (any command)
- B) who=ALL, where=%wheel, as_whom=(ALL), what=ALL — the group name
  is actually the "where" field
- C) There are only three components in a sudoers line, not four
- D) who=%wheel, where=(ALL), as_whom=ALL, what=ALL — "where" and
  "as_whom" are reversed from their actual positions

<details>
<summary>Answer</summary>

**A** — Reading left to right: who is granted access, where it
applies (host), who the command runs as (in parentheses), and what
it's allowed to run. Misordering these when reading a rule is how a
narrow-looking rule turns out to be broad, or vice versa.

</details>

---

**Q3.** Why are `/etc/sudoers.d/` drop-in files preferred over
adding every custom rule directly into the main `/etc/sudoers` file?

- A) `/etc/sudoers.d/` files are processed faster by `sudo`
- B) One file per purpose makes it easy to track, add, and remove a
  specific permission without touching unrelated rules, and package
  upgrades are less likely to conflict with or overwrite a large,
  centrally-edited main file
- C) `visudo` cannot validate the main `/etc/sudoers` file, only files in `/etc/sudoers.d/`
- D) Rules in `/etc/sudoers.d/` do not require `visudo` at all

<details>
<summary>Answer</summary>

**B** — This is an organizational and risk-isolation benefit, not a
technical capability difference — `visudo` validates both locations
identically (trap in C and D). The real advantage is one file per
purpose, easy to version-control and remove independently.

</details>

---

**Q4.** Why does this demo recommend `getent passwd username`
over `grep "^username:" /etc/passwd`, even on a system with no LDAP
configured?

- A) `getent` is faster than `grep` on every system, LDAP or not
- B) `getent` queries whatever name service is actually configured
  (local files, LDAP, NIS) — on a plain local-files system it
  produces the same result as `grep`, but the habit means the same
  command keeps working correctly if centralized auth is ever added,
  where `grep /etc/passwd` would silently return nothing for those users
- C) `grep` cannot search `/etc/passwd` for security reasons
- D) `getent` is required by POSIX; `grep` on `/etc/passwd` is deprecated

<details>
<summary>Answer</summary>

**B** — On a plain local-files system, both commands give identical
results — the value of `getent` is entirely about not needing to
change your habits later if the environment adds centralized
authentication.

</details>

---

**Q5.** `chage -m 7` sets a *minimum* password age of 7 days. What
problem does this actually prevent, given that it seems to make
password changes more restrictive?

- A) It prevents brute-force password guessing attempts
- B) Without it, a user forced to change their password by rotation
  policy could immediately change it right back to their old
  password, satisfying the rotation requirement in name only while
  keeping the same password in practice — `minage` blocks that
  immediate revert
- C) It prevents the password from ever being changed at all
- D) It has no real security purpose — it exists only for legacy compatibility

<details>
<summary>Answer</summary>

**B** — This is the specific loophole `minage` closes: a mandatory
rotation policy is meaningless if the user can immediately switch
back to their previous password the moment the requirement is
satisfied.

</details>

---

**Q6.** A service account is created with `useradd -r -s
/usr/sbin/nologin novatech-svc` (no `-g` flag). Does the User
Private Group scheme still apply, given that `-r` was used?

- A) No — `-r` disables UPG entirely; service accounts get no
  automatically-created group
- B) Yes — `-r` only changes where the UID/GID numbers come from
  (the system range instead of 1000+); it doesn't disable UPG or
  create a fundamentally different kind of account, so a matching
  group is still created automatically
- C) Only on Ubuntu; Rocky9 requires an explicit `-g` flag even with `-r`
- D) UPG only applies to accounts created without `-s /usr/sbin/nologin`

<details>
<summary>Answer</summary>

**B** — `-r` is purely about the ID range service accounts draw
from — it doesn't change the UPG mechanism itself. This is exactly
why `id novatech-svc` shows a matching UPG-style group even for a
`-r` service account.

</details>

---

**Q7.** `passwd -S novatech-svc` shows `L` in the status field.
What does this mean, and how does it differ from `NP`?

- A) `L` means the account is Locked (a hash exists but is prefixed
  with `!`, blocking password login); `NP` means No Password was
  ever set at all — both block password authentication, but for
  different underlying reasons
- B) `L` and `NP` are interchangeable — both simply mean "no access
  at all, including SSH keys"
- C) `L` means the account is Limited to read-only actions; `NP`
  means Non-Privileged
- D) `L` means the password has expired; `NP` means it hasn't been
  set up yet but will accept any password

<details>
<summary>Answer</summary>

**A** — Both `L` and `NP` block password-based login, but they
record different histories: `L` means a password existed and was
deliberately locked; `NP` means one was simply never set. Neither
affects other authentication methods (like SSH keys) on its own —
`passwd`'s lock only governs password authentication specifically.

</details>

---

| Score | Action |
|---|---|
| 7/7 | Import Anki cards, move to Demo 06 |
| 6/7 | Review the wrong answer, then proceed |
| 5/7 | Re-read the relevant section, retry those questions |
| Below 5/7 | Re-read the full demo and redo the walkthrough before proceeding |
````