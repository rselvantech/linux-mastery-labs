# Demo M08-02 — SELinux/AppArmor Deep Dive

## Overview

Demo M08-01 taught you to recognize when MAC is the cause of a failure and to
check its mode. This demo covers the next step: actually fixing a
MAC-caused failure without just disabling enforcement — permanent file
contexts, scoped policy booleans, and reading (skeptically) what
`audit2allow`/`aa-genprof` propose. The throughline is a genuinely common,
genuinely confusing production pattern: **a command works when you run it
as root over SSH, and fails when the real service runs the exact same
operation** — not because of a typo in the command, but because of how
the process was actually launched.

## Demo Objectives

1. Explain why a command that succeeds interactively as root can fail
   when the same action runs inside a real systemd-launched service —
   even one running as the same Unix user
2. Fix a persistent SELinux context mismatch with `semanage fcontext` +
   `restorecon`, and explain why `chcon` alone doesn't survive a relabel
3. Toggle a scoped, pre-defined policy behavior with SELinux booleans,
   without writing custom policy
4. Triage a denial with `ausearch`/`audit2allow`, and explain why loading
   its generated module without reading it first is a real risk
5. Build and refine an AppArmor profile from observed behavior with
   `aa-genprof`/`aa-logprof`
6. Diagnose "works as root, fails as service user" using the reasoning
   chain from Objective 1, not guesswork

## Directory Structure

```
02-selinux-apparmor/
├── README.md                         # this file
├── m08-02-selinux-apparmor-anki.csv  # Anki flashcard deck
└── m08-02-selinux-apparmor-quiz.md   # standalone quiz
```

## Environments

| Task | Ubuntu 24.04 | Rocky Linux 9.8 | EC2 Amazon Linux |
|---|---|---|---|
| SELinux tasks (§1.2-1.4) | Not applicable — Ubuntu defaults to AppArmor | ✅ (`policycoreutils-python-utils` for `semanage`) | Not required |
| AppArmor tasks (§1.5) | ✅ (`apparmor-utils` for `aa-genprof`/`aa-logprof`) | Not applicable — Rocky defaults to SELinux | Not required |

> **Verification note:** every tool this demo depends on was checked for
> live availability in the authoring sandbox — `semanage`, `restorecon`,
> `getsebool`, `setsebool`, `audit2allow`, `aa-genprof`, and `aa-logprof`
> are all absent; only the base `chcon` binary exists, and it has nothing
> to act on without SELinux actually enabled on this filesystem. None of
> this demo's commands or outputs could be run live here. The concepts —
> the context/boolean/policy-module model, and the domain-transition
> reasoning behind "works as root, fails as service user" — are stable,
> documented SELinux/AppArmor behavior; recommend a real verification pass
> on an actual Rocky9 (SELinux) or Ubuntu (AppArmor) machine before
> treating exact command output as guaranteed.

## Recall Check — 01-security-hardening

Answer these from memory before continuing — no looking back at Demo M08-01.

1. `ls -la` on a file shows completely correct DAC ownership and mode.
   What category of cause has that evidence NOT ruled out?
2. What's the actual difference between SELinux Permissive mode and
   Enforcing mode?
3. Why might a file moved into `/var/www/html/` from elsewhere have the
   wrong SELinux context, even with completely correct DAC permissions?

<details>
<summary>Answers</summary>

1. A MAC (SELinux/AppArmor) denial — DAC and MAC are two independent
   layers, and a process can be fully permitted by DAC while still
   blocked by MAC policy.

2. Permissive mode still evaluates every policy decision and logs what
   would be denied, without actually blocking it — a diagnostic tool, not
   "SELinux off." Enforcing mode actually blocks denials.

3. SELinux contexts are typically inherited from the parent directory
   only when a file is created there directly. A file moved in from
   elsewhere often keeps its original context, which may not match what
   the serving policy expects.

</details>

## Part 1 — Concepts

### 1.1 Why "works as root" doesn't mean "works as the service"

This is the reasoning chain behind the single most common SELinux-related
confusion in production, and it has nothing to do with typos in the
command being tested:

```
You SSH in, run: sudo -u appuser /opt/app/run.sh
→ succeeds

The real service, started by systemd, runs the SAME script as the SAME
Unix user (appuser)
→ fails with a permission-denied-looking error
```

**The missing piece: SELinux context is not determined by Unix UID.** An
interactive root/sudo shell over SSH typically runs in an unconfined
domain (commonly `unconfined_t`) — SELinux is barely restricting it at
all in practice, regardless of which Unix user you `sudo -u` into from
there. A process started by `systemd` from a unit file, on the other hand,
is assigned a context based on **how systemd launches it** — often a
specific, policy-restricted domain tied to that service (`httpd_t`,
a custom `myapp_t`, etc.) — which is enforced fully regardless of which
Unix user the service runs as.

**This means testing "as the service user" via `sudo -u`/`su` does not
reproduce the actual failure**, because it never puts the process into the
real service's confined domain. To actually reproduce and diagnose it, you
need to trigger the real code path — `systemctl start`, not a manual `su`
— and then read what SELinux actually blocked for *that* domain.

### 1.2 Contexts — the permanent fix, not just chcon

```bash
ls -Z /opt/app/data/
# user:role:type:level — e.g. system_u:object_r:var_t:s0
```

```bash
sudo chcon -t httpd_sys_content_t /opt/app/data/uploads/
```

`chcon` changes a file's context **immediately, but only until the next
relabel** — a full filesystem relabel (`restorecon -R /`, common after a
policy update or a `touch /.autorelabel` + reboot) reverts it back to
whatever the policy database says it should be, silently undoing a `chcon`
fix with no warning. This is a common cause of "the fix worked, then
mysteriously broke again after a patch/reboot."

The permanent fix registers the rule in the policy database first, then
applies it:

```bash
sudo semanage fcontext -a -t httpd_sys_content_t "/opt/app/data(/.*)?"
# -a: add a new rule. The path is a REGEX (note the (/.*)? pattern to
# match the directory and everything under it), not a literal glob.
sudo restorecon -Rv /opt/app/data/
# -R: recursive, -v: verbose — actually APPLIES the newly-registered
# rule to the files that already exist; semanage fcontext alone only
# updates the policy database, it doesn't touch any files itself.
```

**Two separate steps, two separate purposes**: `semanage fcontext` records
what the context *should* be, permanently, surviving a relabel;
`restorecon` applies that recorded rule to files right now. Skipping
`restorecon` after `semanage fcontext` means the database is correct but
the actual files are still wrong until the next relabel happens on its
own.

### 1.3 Booleans — scoped toggles without writing policy

Some genuinely common, genuinely legitimate deviations from a strict
default policy are anticipated by the policy authors themselves and
exposed as a named on/off switch, rather than requiring a custom module:

```bash
getsebool -a | grep httpd
# httpd_can_network_connect --> off
# httpd_can_sendmail --> off
```

```bash
sudo setsebool httpd_can_network_connect on
# Temporary — reverts on reboot
sudo setsebool -P httpd_can_network_connect on
# -P: Persistent — survives a reboot, written to the policy store
```

**Booleans exist specifically so a known, common, legitimate need doesn't
require writing custom policy at all.** Before reaching for `audit2allow`
(§1.4) or a custom module, check whether a boolean already covers the
exact behavior you need — `getsebool -a` searched by keyword is often
faster and always safer than generating new policy.

### 1.4 audit2allow — read it before you trust it

```bash
sudo ausearch -m avc -ts recent | audit2allow -M mymodule
```

This reads recent SELinux denials and generates a loadable custom policy
module (`mymodule.pp`) that would allow exactly what was denied. **This is
also its exact danger**: `audit2allow` turns every denial it's given into
an "allow" rule, with zero judgment about whether that denial was actually
a bug in the policy or the policy correctly doing its job — for example,
blocking a process that's already been compromised from doing something it
should never do. Blindly running the suggested `semodule -i mymodule.pp`
defeats the entire purpose of MAC: you've just told the system to stop
blocking whatever it was blocking, without ever asking *why* it was
blocking it.

```bash
audit2allow -w -a
# -w: human-readable explanation of WHY each denial happened, not just
# the raw allow-rule it would generate — read this before generating or
# loading anything.
cat mymodule.te
# Always read the generated .te (type enforcement) SOURCE file before
# loading the compiled module — confirm each requested permission is
# something you actually intend to grant, not just accept the tool's
# output wholesale.
```

### 1.5 AppArmor — building a profile from real behavior

Where SELinux's workflow (§1.2-1.4) is "define policy, then find and fix
denials against it," AppArmor's tooling supports actively **building** a
profile by exercising the real application and reviewing each access it
makes:

```bash
sudo aa-genprof /opt/app/bin/myapp
# Starts myapp in a learning mode. While it's running, use the
# application normally (exercise every code path you care about) — then
# return to the terminal and aa-genprof interactively asks, for each
# distinct access it observed, whether to Allow, Deny, Glob (generalize
# the specific path into a pattern), or ignore it.
```

```bash
sudo aa-logprof
# For a profile ALREADY in complain mode (Demo M08-01 §1.3), reviews the
# kernel audit log for anything the profile doesn't yet cover and
# interactively offers to add each one — the ongoing-maintenance
# counterpart to aa-genprof's initial-build workflow.
```

**The architectural difference from `audit2allow` is real, not just a
different command name**: `audit2allow` summarizes a log *after the fact*
into a proposed policy chunk you review once; `aa-genprof`/`aa-logprof`
interactively walk you through each access *as* (or shortly after) it
happens, asking you to make a judgment call per access rather than
accepting or rejecting one generated block wholesale. Both still require
the same underlying discipline from §1.4: don't reflexively say "Allow" to
everything just to make the complaining stop.

### 1.6 Bringing it together — diagnosing the root/service-user pattern

```
1. Confirm MAC is actually involved: getenforce / aa-status (Demo M08-01)
2. Reproduce via the REAL launch path — systemctl start, not su/sudo —
   because §1.1 means a manual reproduction attempt can succeed while
   the real service fails
3. Read the actual denial: ausearch -m avc (SELinux) or the AppArmor
   audit log for the specific profile
4. Check for an existing boolean first (§1.3) — often the fix needs no
   custom policy at all
5. If a context is the issue, fix it permanently (§1.2): semanage
   fcontext, then restorecon
6. Only reach for audit2allow/aa-genprof (§1.4/§1.5) once the simpler
   fixes are ruled out, and read what they generate before loading it
```

## Part 2 — Hands-On Lab

### Commands used in this demo

| Command | Purpose |
|---|---|
| `ls -Z` | Show a file's current SELinux context |
| `semanage fcontext -a -t TYPE 'path_regex'` | Permanently register the intended context for a path |
| `restorecon -Rv PATH` | Apply registered contexts to existing files |
| `getsebool -a` / `setsebool [-P] NAME on\|off` | List / toggle a policy boolean (temporary or persistent) |
| `ausearch -m avc -ts recent \| audit2allow -M NAME` | Generate a custom policy module from recent denials |
| `audit2allow -w -a` | Human-readable explanation of recent denials |
| `aa-genprof BINARY` | Interactively build an AppArmor profile from observed behavior |
| `aa-logprof` | Interactively extend an existing complain-mode profile |

### Task 1 — Reproduce the "works as root" trap

```bash
# As root/sudo, interactively:
sudo -u appuser /opt/app/bin/check-access.sh /opt/app/data/uploads
# succeeds

# The real way the service actually runs it:
sudo systemctl start appcheck.service
systemctl status appcheck.service
journalctl -u appcheck.service -n 20
```

Confirm (per §1.1) that the manual `sudo -u` attempt and the real
`systemctl start` can produce different outcomes for the identical
underlying command, and explain why before reading further.

### Task 2 — Fix a context permanently

```bash
ls -Z /opt/app/data/uploads/
sudo semanage fcontext -a -t httpd_sys_content_t "/opt/app/data/uploads(/.*)?"
sudo restorecon -Rv /opt/app/data/uploads/
ls -Z /opt/app/data/uploads/
```

Confirm the context actually changed, and that `semanage fcontext -l |
grep uploads` shows the rule persisted in the policy database (not just
applied once via `chcon`).

### Task 3 — Find and toggle a boolean before writing policy

```bash
getsebool -a | grep -i network
sudo setsebool httpd_can_network_connect on
getsebool httpd_can_network_connect
sudo setsebool -P httpd_can_network_connect on
```

### Task 4 — Triage a denial with audit2allow, and read before loading

```bash
sudo ausearch -m avc -ts recent
audit2allow -w -a
sudo ausearch -m avc -ts recent | audit2allow -M appcheck_fix
cat appcheck_fix.te
```

Before running `semodule -i appcheck_fix.pp`, read `appcheck_fix.te` line
by line and confirm every requested permission is something you actually
intend to grant — this is the step the danger in §1.4 is about, not a
formality.

### Task 5 — Build an AppArmor profile interactively (Ubuntu)

```bash
sudo aa-genprof /opt/app/bin/myapp
# In another terminal, exercise the application's real functionality
# Return to the aa-genprof terminal and work through each prompt
sudo aa-status | grep myapp
```

## What You Learned

- ✅ Explained why a command succeeding as root over SSH can still fail
  inside a real systemd-launched service running as the same Unix user
- ✅ Fixed a persistent SELinux context mismatch with `semanage fcontext`
  + `restorecon`, and why `chcon` alone doesn't survive a relabel
- ✅ Toggled a scoped policy boolean instead of writing custom policy
- ✅ Triaged a denial with `ausearch`/`audit2allow`, and read the
  generated module before considering loading it
- ✅ Built and refined an AppArmor profile from observed behavior with
  `aa-genprof`/`aa-logprof`
- ✅ Diagnosed "works as root, fails as service user" using the actual
  domain-transition reasoning, not guesswork

## Part 3 — Break-Fix Scenario

### Scenario — NovaTech Ops: "it works when I test it"

> **Incident report — NovaTech Ops, ticket #5140**
>
> "The on-call engineer tested `appcheck.sh` as the `appuser` account over
> SSH and it worked perfectly. The actual `appcheck.service` fails every
> time it runs on schedule. The engineer is now convinced the script has a
> race condition."

```bash
$ sudo -u appuser /opt/app/bin/appcheck.sh
OK
$ sudo systemctl start appcheck.service
$ systemctl status appcheck.service
● appcheck.service - App Check
     Active: failed (Result: exit-code)
$ sudo ausearch -m avc -ts recent | tail -5
type=AVC ... denied { read } for pid=... comm="appcheck.sh" ... scontext=system_u:system_r:appcheck_t:s0 tcontext=... tclass=file
```

Tasks:
1. Reference §1.1 — is "race condition" a reasonable next hypothesis
   given this evidence? What does the `ausearch` output actually show
   instead?
2. Why did the manual `sudo -u appuser` test succeed while the real
   service, running the identical script as the identical Unix user,
   failed?
3. What's the correct fix path from here, per §1.6's order of operations?

<details>
<summary>Answers</summary>

"Race condition" is not supported by this evidence and is the wrong next
hypothesis — the `ausearch` output already shows exactly what happened: an
SELinux denial, with `scontext=...appcheck_t:s0` naming the confined
domain the real service runs in. The manual test succeeded because
`sudo -u appuser` ran the script in an effectively unconfined interactive
shell context, never entering `appcheck_t` at all — the Unix user being
identical in both cases is irrelevant, because SELinux enforcement is tied
to the process's domain, not the Unix UID (§1.1). The correct fix path per
§1.6: first check whether an existing boolean already covers whatever
`appcheck_t` needs (`getsebool -a | grep` for a relevant keyword); if not,
determine whether the denial is a context problem on a specific file/path
(fix permanently with `semanage fcontext` + `restorecon`) or a genuinely
new permission the domain needs (generate and carefully read an
`audit2allow` module before loading it) — not a code-level race-condition
fix, which would do nothing to address an SELinux denial.

</details>

## Interview Prep

**Q1. A script works when you test it manually as a service account over SSH, but fails when the real systemd service runs it. What's the first thing to check, and why?**
Whether SELinux/AppArmor is involved, and specifically whether the manual
test actually reproduced the real service's confined domain. SELinux
context is assigned based on how a process is launched (a systemd unit's
label), not which Unix UID it runs as — a manual `sudo -u`/`su` test
typically runs in an unconfined interactive context and can succeed while
the real, policy-restricted service domain fails on the identical command.

**Q2. Why doesn't `chcon` permanently fix an SELinux context problem?**
`chcon` changes a file's context immediately, but the policy database's
recorded rule for that path is unchanged — a full relabel (`restorecon
-R`, or a policy update triggering one) reverts the file back to whatever
the database says, silently undoing the `chcon` fix. `semanage fcontext`
updates the database itself, so the correct context survives a relabel;
`restorecon` is then needed to apply that database change to existing
files right now.

**Q3. Why check for an existing SELinux boolean before generating a custom policy module with `audit2allow`?**
Booleans are pre-defined, reviewed, scoped toggles for known, legitimate
policy deviations — turning one on is safer and simpler than generating
and loading new custom policy for a need the policy authors already
anticipated. `audit2allow` should be reserved for denials no existing
boolean covers.

**Q4. What's the real risk in running `semodule -i` on whatever `audit2allow` generates, without reading it first?**
`audit2allow` has no judgment about whether a given denial was a policy
bug or the policy correctly doing its job — it turns every denial it's
shown into an allow rule. Loading a generated module unread risks granting
permissions that were never actually supposed to be allowed, including
ones that were blocking a genuinely malicious or compromised action —
defeating the purpose of having MAC in the first place.

## Key Takeaways

1. **SELinux context is tied to how a process was launched, not which
   Unix UID it runs as.** A manual `sudo -u`/`su` test can succeed in an
   unconfined shell while the real systemd-launched service, running the
   identical command as the identical user, fails in its own confined
   domain — reproduce via the real launch path, not a manual stand-in.
2. **`chcon` is temporary; `semanage fcontext` + `restorecon` is the
   permanent fix.** A relabel silently reverts a bare `chcon` change —
   register the rule in the policy database first, then apply it.
3. **Check for an existing boolean before writing custom policy.**
   `getsebool -a | grep` for a relevant keyword is faster and safer than
   generating a module for a need the policy authors already anticipated.
4. **`audit2allow` turns every denial into an allow rule with no judgment
   about whether the denial was correct.** Read the generated `.te` file
   and `audit2allow -w -a`'s explanation before ever loading a generated
   module — a denial might be MAC doing exactly its job.
5. **AppArmor's `aa-genprof`/`aa-logprof` build a profile interactively
   from observed behavior, architecturally different from `audit2allow`'s
   after-the-fact log summary** — but the same discipline applies: don't
   reflexively approve every prompted access just to make it stop
   complaining.
6. **Diagnose in order: confirm MAC involvement, reproduce via the real
   path, read the actual denial, check for a boolean, then fix context or
   generate policy as a last resort** — skipping straight to custom
   policy is how legitimate security blocks get silently disabled.

> **Demo scope:** Primary concept: diagnosing and permanently fixing
> MAC-caused failures beyond basic mode-switching, anchored on the
> "works as root, fails as service user" pattern. Supporting concepts:
> SELinux contexts (`semanage fcontext`/`restorecon`), booleans,
> `audit2allow` triage, AppArmor `aa-genprof`/`aa-logprof`.
> Estimated completion time: 40 minutes (reading + hands-on + verification).
> Checkpoints: 5 natural stopping points (after Task 1 through Task 5).

## Quick Commands Reference

### SELinux contexts and booleans

| Command | Purpose |
|---|---|
| `ls -Z PATH` | Show current context |
| `semanage fcontext -a -t TYPE 'regex'` | Permanently register intended context |
| `restorecon -Rv PATH` | Apply registered contexts to existing files |
| `getsebool -a` | List all booleans and their state |
| `setsebool [-P] NAME on\|off` | Toggle a boolean (temporary / persistent with `-P`) |

### Policy triage

| Command | Purpose |
|---|---|
| `ausearch -m avc -ts recent` | Recent SELinux denials |
| `audit2allow -w -a` | Human-readable explanation of denials |
| `ausearch ... \| audit2allow -M NAME` | Generate a loadable module — read before loading |
| `semodule -i NAME.pp` | Load a reviewed, approved policy module |

### AppArmor profile building

| Command | Purpose |
|---|---|
| `aa-genprof BINARY` | Interactively build a new profile from observed behavior |
| `aa-logprof` | Interactively extend an existing complain-mode profile |

## Appendix A — Anki Cards

**`m08-02-selinux-apparmor-anki.csv`:**

```
#deck:Linux Mastery::Module 8 - Security::Demo M08-02 - SELinux/AppArmor Deep Dive
#separator:Comma
#columns:Front,Back,Tags
"A script works when tested manually as a service account via sudo -u, but the real systemd service running the identical command as the identical user fails. Why?","SELinux context is tied to HOW a process is launched, not which Unix UID it runs as. A manual sudo -u test typically runs unconfined; the real systemd-launched service runs in its own confined domain, which is what actually gets enforced.","demo27a,selinux,context,common-mistake"
"Why doesn't chcon permanently fix an SELinux context mismatch?","chcon changes a file's context immediately, but the policy database's recorded rule for that path is unchanged. A relabel (restorecon -R, or a policy update) reverts the file back to whatever the database says, silently undoing the chcon fix.","demo27a,selinux,chcon,common-mistake"
"What are the two separate steps to permanently fix an SELinux context, and what does each one do?","semanage fcontext -a -t TYPE 'regex' registers the intended context in the policy database (persists across relabels). restorecon -Rv PATH applies that registered rule to files that already exist — semanage fcontext alone doesn't touch any files.","demo27a,selinux,semanage,restorecon"
"What are SELinux booleans, and why check for one before writing custom policy?","Pre-defined, scoped on/off toggles for known, legitimate policy deviations (e.g. httpd_can_network_connect). Checking getsebool -a first is safer and simpler than generating new policy for a need the policy authors already anticipated.","demo27a,selinux,booleans,best-practice"
"What's the real risk of running semodule -i on whatever audit2allow generates without reading it first?","audit2allow turns every denial it's shown into an allow rule with no judgment about whether the denial was actually a bug or the policy correctly doing its job. Loading it unread risks granting permissions that were meant to be blocked.","demo27a,audit2allow,risk"
"What does setsebool -P do differently from plain setsebool?","-P (persistent) writes the change to the policy store so it survives a reboot. Plain setsebool only changes the running state, reverting on reboot.","demo27a,selinux,setsebool"
"How is AppArmor's aa-genprof/aa-logprof workflow architecturally different from SELinux's audit2allow?","audit2allow summarizes a log AFTER THE FACT into one proposed policy chunk reviewed once. aa-genprof/aa-logprof interactively walk through each observed access as/near when it happens, asking for a judgment call per access rather than one bulk generated block.","demo27a,apparmor,selinux,comparison"
"What is aa-logprof used for, specifically, versus aa-genprof?","aa-logprof extends a profile ALREADY in complain mode by reviewing the kernel audit log for accesses it doesn't yet cover — the ongoing-maintenance counterpart to aa-genprof's initial profile-building workflow.","demo27a,apparmor,aa-logprof"
"What's the correct diagnostic order when MAC is suspected as the cause of a failure?","Confirm MAC is involved (getenforce/aa-status) -> reproduce via the REAL launch path, not a manual stand-in -> read the actual denial -> check for an existing boolean -> fix context permanently if that's the issue -> only then consider a custom policy module, read before loading.","demo27a,diagnosis,workflow"
```

## Appendix B — Quiz

**`m08-02-selinux-apparmor-quiz.md`:**

````markdown
# Quiz — Demo M08-02: SELinux/AppArmor Deep Dive

> One correct answer per question unless stated otherwise.
> Target: 80% or above before moving to Demo M08-03.

---

**Q1.** A script succeeds when run manually via `sudo -u appuser`, but
the real `systemd` service running the identical script as the identical
Unix user fails. What's the most likely explanation?

- A) A race condition specific to how systemd launches processes
- B) SELinux context is tied to how the process was launched, not its Unix UID — the manual test ran unconfined, while the real service runs in its own confined domain
- C) `systemd` services always run with fewer permissions than interactive shells, regardless of SELinux
- D) The script has a bug that only manifests when run non-interactively

<details>
<summary>Answer</summary>

**B** — This is the exact reasoning chain from §1.1, and the actual cause
in this demo's Break-Fix scenario, confirmed by the `ausearch` output
naming a specific confined domain.

</details>

---

**Q2.** After running `sudo chcon -t httpd_sys_content_t /srv/data/`, the
fix works — until a routine SELinux relabel reverts it. What should have
been done instead?

- A) Run `chcon` again after every relabel, indefinitely
- B) Use `semanage fcontext -a` to register the context permanently in the policy database, then `restorecon` to apply it
- C) Disable SELinux relabeling entirely
- D) `chcon` changes are always permanent; something else must have caused this

<details>
<summary>Answer</summary>

**B** — `semanage fcontext` is what survives a relabel; `chcon` alone
never does.

</details>

---

**Q3.** Before generating a custom policy module with `audit2allow` for a
denial, what should you check first?

- A) Whether the denial can be silenced by disabling SELinux entirely
- B) Whether an existing SELinux boolean (`getsebool -a`) already covers the exact behavior needed
- C) Whether the file's DAC permissions are correct
- D) Whether the process is running as root

<details>
<summary>Answer</summary>

**B** — Booleans are pre-vetted, scoped toggles for known needs — checking
first avoids generating and loading unnecessary custom policy.

</details>

---

**Q4.** What is the specific danger of running `semodule -i` on a module
`audit2allow` generated, without reading the `.te` file first?

- A) The module might fail to compile
- B) `audit2allow` grants an allow rule for every denial it's shown, with no judgment about whether the denial was a bug or a correct security block — loading it unread risks disabling a legitimate protection
- C) `semodule -i` requires a reboot to take effect
- D) Generated modules are always syntactically invalid

<details>
<summary>Answer</summary>

**B** — This is the actual risk §1.4 warns about — `audit2allow` has no
concept of "this denial was correct," only "this was denied."

</details>

---

**Q5.** How does `aa-genprof`'s workflow differ from `audit2allow`'s?

- A) They are functionally identical, just different command names for the same distro
- B) `audit2allow` summarizes a log after the fact into one block reviewed once; `aa-genprof` interactively asks for a judgment call on each observed access as the application is exercised
- C) `aa-genprof` only works on already-compiled policy modules
- D) `audit2allow` requires the application to be actively running; `aa-genprof` does not

<details>
<summary>Answer</summary>

**B** — This is a real architectural difference between the two tools'
workflows, not just a naming difference between distros.

</details>

---

**Q6.** What's the difference between `setsebool httpd_can_network_connect on` and `setsebool -P httpd_can_network_connect on`?

- A) They are identical
- B) Without `-P`, the change is temporary and reverts on reboot; `-P` makes it persistent, surviving a reboot
- C) `-P` only works for network-related booleans
- D) `-P` requires the boolean to already be off

<details>
<summary>Answer</summary>

**B** — `-P` writes the change to the persistent policy store; without it,
the change only affects the current running state.

</details>

---

**Q7.** Per §1.6's diagnostic order, what should happen immediately after
confirming MAC is involved in a failure?

- A) Immediately generate a custom policy module with `audit2allow`
- B) Reproduce the failure via the real launch path (e.g. `systemctl start`), not a manual `sudo -u`/`su` stand-in
- C) Disable SELinux/AppArmor to confirm the app works without it, then leave it disabled
- D) Check DAC permissions again, since MAC and DAC are the same layer

<details>
<summary>Answer</summary>

**B** — Confirming MAC involvement, then reproducing via the real path is
the order §1.6 lays out — a manual reproduction attempt can give a false
negative, as this demo's Break-Fix scenario shows directly.

</details>

---

Score guide:

| Score | Action |
|---|---|
| 7/7 | Import Anki cards, move to Demo M08-03 |
| 6/7 | Review the wrong answer, then proceed |
| 5/7 | Re-read the relevant section, retry those questions |
| Below 5/7 | Re-read the full demo and redo the walkthrough before proceeding |
````