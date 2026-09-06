# Quiz — Demo 05: Users, Groups & Authentication

> One correct answer per question unless stated otherwise.
> Target: 80% or above before moving to Demo 06.

---

**Q1.** An admin runs `usermod -G backup deployer` intending to add
deployer to the `backup` group. `id deployer` afterward shows
`groups=1002(deployer),999(backup)` — the `sudo` group is gone.
What caused this?

- A) `backup` group has higher priority and overrides `sudo`
- B) Bare `-G` replaces the entire supplementary group list — every
  prior group except the primary group is removed
- C) `usermod` always removes `sudo` when adding a new group
- D) The admin needed to run `groupmod`, not `usermod`

<details>
<summary>Answer</summary>

**B** — `-G` (without `-a`) sets the supplementary group list to
exactly what is specified, discarding all previous supplementary
memberships. The fix is `usermod -aG backup deployer` (append),
or to restore both at once: `usermod -aG sudo,backup deployer`.

</details>

---

**Q2.** A user on Rocky Linux 9 is added to the `sudo` group with
`usermod -aG sudo username`, but sudo access does not work. Why?

- A) The command needs `-G` instead of `-aG`
- B) Rocky9 requires a reboot after group changes
- C) `sudo` is the Debian/Ubuntu group name — Rocky9/RHEL family
  uses `wheel` instead, which is the group with the sudoers rule
- D) Rocky9 does not support the `sudo` command at all

<details>
<summary>Answer</summary>

**C** — RHEL-family distros (Rocky, RHEL, CentOS, Amazon Linux,
Fedora) grant sudo access via the `wheel` group, not `sudo`.
Correct fix: `usermod -aG wheel username`. Verify which group has
the actual sudoers rule with `grep -E "^%sudo|^%wheel" /etc/sudoers`.

</details>

---

**Q3.** Why is `/etc/shadow` readable only by root, while
`/etc/passwd` remains world-readable?

- A) `/etc/passwd` contains no sensitive data at all; `/etc/shadow`
  stores password hashes which must not be exposed to every user
- B) `/etc/shadow` is larger and restricted for performance reasons
- C) `/etc/passwd` is deprecated and `/etc/shadow` replaces it
  entirely
- D) Both files have the same permissions on modern systems

<details>
<summary>Answer</summary>

**A** — `/etc/passwd` must stay world-readable because many tools
need to resolve UIDs to usernames (e.g. `ls -l` showing file
owners). `/etc/shadow` holds the actual password hash and aging
policy and is restricted to root to prevent offline password
cracking attempts against exposed hashes.

</details>

---

**Q4.** You need to create a Linux account that runs a background
service but must never be used for interactive login. Which two
settings are essential?

- A) `useradd -m` and a strong password
- B) `useradd -r -s /usr/sbin/nologin` plus a locked/no password
- C) `useradd -G wheel` and `/bin/bash` as shell
- D) `useradd -s /bin/false` only — no other settings needed

<details>
<summary>Answer</summary>

**B** — `-r` creates a system-range UID account, `-s /usr/sbin/nologin`
rejects any interactive login attempt while still allowing systemd
to run processes as the account. The password should also be
locked or unset (`passwd -S` showing `L` or `NP`) — a service
account with a usable password is still a login risk even with
`nologin` set, since some access paths (key-based SSH, for example)
do not require a shell to succeed in spawning a process.

Trap: D alone is incomplete — without addressing the password
state, the account is not fully secured.

</details>

---

**Q5.** What is the User Private Group (UPG) scheme?

- A) Every user shares one global group for simplicity
- B) `useradd` with no `-g` flag creates a new group matching the
  username and GID, set as that user's primary group
- C) Only root can have a private group
- D) UPG is a Rocky9-specific feature not present on Ubuntu

<details>
<summary>Answer</summary>

**B** — On standard installs, `useradd deployer` (no `-g`) also
creates group `deployer` with a matching GID and sets it as
deployer's primary group. This is why a user's primary group
typically matches their own username. UPG applies on both Ubuntu
and Rocky9 by default.

</details>

---

**Q6.** Why use `visudo` instead of opening `/etc/sudoers` in a
plain text editor?

- A) `visudo` is faster to type
- B) `visudo` validates syntax before saving and refuses to save
  broken syntax — a plain editor save with a typo can lock out all
  sudo access system-wide
- C) `visudo` is required by RHCSA exam rules only
- D) Plain text editors cannot open files in `/etc/`

<details>
<summary>Answer</summary>

**B** — A syntax error saved directly to `/etc/sudoers` can break
sudo entirely, including for the person who made the mistake. If
no other root access path exists, this is a serious lockout.
`visudo` (and `visudo -f` for `/etc/sudoers.d/` files) validates
before allowing a save.

</details>

---

**Q7.** A contractor's account needs to stop working automatically
on a known future date without manual intervention. Which command
sets this?

- A) `usermod -L contractor` (lock immediately)
- B) `chage -E 2026-12-31 contractor`
- C) `passwd -e contractor`
- D) `userdel --expire-date contractor`

<details>
<summary>Answer</summary>

**B** — `chage -E YYYY-MM-DD username` sets a hard account expiry
date in `/etc/shadow`. After that date, the account becomes
unusable automatically. `-L` (A) locks immediately, not on a
future date. `userdel` (D) has no such flag and would delete the
account now, not on a schedule.

</details>

---

| Score | Action |
|---|---|
| 7/7 | Import Anki cards, move to Demo 06 |
| 6/7 | Review the wrong answer, then proceed |
| 5/7 | Re-read the relevant section, retry those questions |
| Below 5/7 | Re-read the full demo and redo the walkthrough before proceeding |
