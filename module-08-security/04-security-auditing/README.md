# Demo M08-04 — Security Auditing Tools

## Overview

Demos M08-01/M08-02/M08-03 each hardened one specific thing — SSH, MAC policy, TLS.
This demo covers the tools that check **all of it at once, systematically,
and repeatedly**: `lynis` and OpenSCAP. Neither tool fixes anything by
itself — both produce a prioritized list of findings, and the actual
value is in reading that list correctly and acting on the parts that
matter, not treating every line as equally urgent.

## Demo Objectives

1. Explain what `lynis` and OpenSCAP are each for, and how they differ —
   heuristic baseline auditing versus standards-based compliance scanning
2. Run a `lynis` audit and read its hardening index and suggestions
3. Run an OpenSCAP scan against a specific SCAP profile and read a
   compliance report
4. Triage a long findings list — not every finding is equally urgent
5. Recognize which findings a baseline scan would surface for gaps this
   series already covered (SSH, MAC, TLS) if any of it were skipped or
   misconfigured
6. Explain why a baseline audit is run on a schedule, not once

## Directory Structure

```
04-security-auditing/
├── README.md                          # this file
├── m08-04-security-auditing-anki.csv  # Anki flashcard deck
└── m08-04-security-auditing-quiz.md   # standalone quiz
```

## Environments

| Task | Ubuntu 24.04 | Rocky Linux 9.8 | EC2 Amazon Linux |
|---|---|---|---|
| `lynis` | `sudo apt install lynis` | `sudo dnf install lynis` | Not required |
| OpenSCAP | `sudo apt install libopenscap8` (limited content) | `sudo dnf install openscap-scanner scap-security-guide` (full CIS/STIG content — the standard platform for this tool) | Not required |

> **Verification note:** neither `lynis` nor `oscap` could be installed
> in this sandbox (no network access — the same constraint already
> disclosed throughout this module). Everything below is built from
> stable, well-documented tool behavior — the general shape of `lynis`'s
> output (hardening index, `[SUGGESTION]`/`[WARNING]` lines) and
> `oscap`'s report structure are consistent across versions, but exact
> wording, scoring, and specific rule IDs are version- and
> profile-dependent. Recommend a real run on an actual Ubuntu/Rocky9
> machine before treating any specific number or rule ID here as
> guaranteed. `[Unverified — test before use]` applies to this entire
> demo's captured-looking output.

## Recall Check — 03-tls-certificates

Answer these from memory before continuing — no looking back at Demo M08-03.

1. What's the practical difference between `openssl verify`'s `error 20`
   and `error 10`?
2. Why must a server present its intermediate certificate, not just its
   own leaf certificate?
3. A certificate renewal reports success, but the service still serves
   the old, expired certificate. What's the likely cause?

<details>
<summary>Answers</summary>

1. `error 20` (unable to get local issuer certificate) means the chain is
   incomplete; `error 10` (certificate has expired) means the validity
   window has actually passed. Different causes, different fixes.
2. Clients only trust the root CA directly — the intermediate is what
   bridges a valid leaf certificate to a root the client already trusts.
3. Renewal replaced the file on disk, but the running service was never
   told to reload and re-read it.

</details>

## Part 1 — Concepts

### 1.1 lynis — broad, heuristic, no compliance framework required

`lynis` scans a huge range of system areas — boot process, authentication,
file permissions, kernel hardening, installed software, logging,
malware-scanner presence, and more — and reports each check as a
`[SUGGESTION]` or `[WARNING]`, plus an overall **hardening index** (0-100)
summarizing the pass rate. It requires no particular compliance framework
or pre-selected profile — it's opinionated on general good practice, not
tied to any specific published standard.

```bash
sudo lynis audit system
```

The general shape of the output:

```
[+] Boot and services
------------------------------------
  - Service Manager................: [ FOUND ]
  - Checking UEFI boot..............: [ DISABLED ]

[+] Authentication
------------------------------------
  - Checking password hashing method: [ SHA512 ]
  - Checking PAM password strength..: [ WEAK ]
    SUGGESTION: Configure minimum password length in PAM

...
Hardening index : 68 [############        ]
```

`[Unverified — test before use]`

**The hardening index alone is not the point** — a single number invites
treating "raise the number" as the goal, when a handful of specific,
high-severity `[WARNING]`s matter far more than a marginal index increase
from fixing several low-severity `[SUGGESTION]`s.

### 1.2 OpenSCAP — standards-based, tied to a specific profile

Where `lynis` is heuristic, OpenSCAP scans against a specific, named
**profile** — a published, versioned set of rules from a real compliance
standard (CIS Benchmarks, DISA STIG, PCI-DSS), delivered as SCAP content
(commonly the SCAP Security Guide, `scap-security-guide` package). A scan
result maps each finding directly to a specific rule ID in that published
standard — useful when the actual requirement is "prove compliance with
framework X," not just "improve security in general."

```bash
oscap xccdf eval --profile xccdf_org.ssgproject.content_profile_cis \
  --results results.xml --report report.html \
  /usr/share/xml/scap/ssg/content/ssg-rl9-ds.xml
```

```
Title   Result
--------------------------------------------------------------
Ensure /tmp Located On Separate Partition        fail
Ensure SSH Idle Timeout Interval is Configured    pass
Disable Kernel Support for USB via Bootloader     fail
...
```

`[Unverified — test before use]`

**The profile choice matters as much as running the scan at all** —
scanning against CIS when the actual requirement is DISA STIG produces a
report that answers the wrong question entirely, even though the scan
itself "succeeds."

### 1.3 The same underlying question, two different lenses

Both tools are ultimately answering "is this system configured
securely," but from different angles:

```
lynis:     broad, opinionated, no external standard required —
           good for a general baseline and ongoing self-assessment
OpenSCAP:  narrow to a specific published profile, produces evidence
           mapped to named rule IDs — good when you must demonstrate
           compliance with a specific external standard
```

Neither replaces the other, and neither replaces the targeted work from
Demos M08-01/M08-02/M08-03 — both tools would surface a *symptom* (weak SSH config,
a MAC policy left permissive, a certificate near expiry) without doing
the actual diagnosis and fix this series already covered in depth.

### 1.4 Triage — not every finding is equally urgent

A real `lynis` or OpenSCAP run against an unhardened host routinely
produces dozens to hundreds of findings. Treating the list as a flat,
top-to-bottom to-do list is a mistake. A reasonable triage order:

```
1. Findings with a direct, known exploit path (an outdated package with
   a published CVE; a service exposed that shouldn't be)
2. Findings that map to something this series already built and verified
   for a reason (§1.5) — these should already be passing; a failure here
   means a real regression, not just an unaddressed baseline gap
3. Findings that are genuinely low-risk in this specific environment
   (e.g. a physical-console-only hardening suggestion on a cloud VM with
   no physical console at all) — document why it's accepted, don't just
   ignore it silently
4. Everything else, roughly in order of how directly it reduces actual
   attack surface
```

**A finding being reported does not automatically mean it's the right
priority for this specific system** — context (cloud VM vs. physical
server, internal-only vs. internet-facing) changes what actually matters,
which neither tool can fully know on its own.

### 1.5 What a baseline scan would catch from this module

A scan run against a host that skipped or misconfigured any of Demos
M08-01/M08-02/M08-03's work would surface it as a finding, even without anyone
manually re-checking:

```
Demo M08-01  (SSH/fail2ban/auditd):  PermitRootLogin/PasswordAuthentication
                                  left at insecure defaults; fail2ban not
                                  installed/running; no audit rules
                                  watching sensitive files
Demo M08-02 (SELinux/AppArmor):     MAC left in Permissive/complain mode
                                  rather than Enforcing/enforce
Demo M08-03 (TLS/certificates):     a certificate approaching or past
                                  expiry; weak TLS protocol versions or
                                  cipher suites still enabled
```

This is the practical argument for running a baseline scan on a schedule
rather than once (§1.6): each of these is exactly the kind of thing that
can silently drift back out of compliance after this series' work is
done — a config gets reset by a package update, a MAC mode gets flipped
to Permissive for a debugging session and never flipped back, a
certificate renewal silently fails (Demo M08-03 §1.6).

### 1.6 Why this runs on a schedule, not once

A one-time scan proves the system was compliant *at that moment*. It says
nothing about a week later, after a package update reset a config file, a
temporary debugging change was never reverted, or a certificate crossed
its renewal window. The standard pattern is a scheduled run (a systemd
timer — Demo M02-05 — or cron) with results diffed against the previous run,
so a **new** finding that wasn't there last week gets attention
immediately, rather than being buried in a report alongside a hundred
long-accepted, already-triaged ones.

## Part 2 — Hands-On Lab

### Commands used in this demo

| Command | Purpose |
|---|---|
| `lynis audit system` | Run a full baseline audit |
| `lynis show profiles` | List available `lynis` scan profiles |
| `oscap xccdf eval --profile PROFILE --report FILE CONTENT.xml` | Run an OpenSCAP scan against a named profile |
| `oscap info CONTENT.xml` | List profiles available in a given SCAP content file |

### Task 1 — Run a lynis baseline audit

```bash
sudo lynis audit system
```

Locate the hardening index at the end of the run, and count how many
`[WARNING]` lines appear versus `[SUGGESTION]` lines — these are not
equally urgent (§1.4).

### Task 2 — Run an OpenSCAP scan against a specific profile

```bash
oscap info /usr/share/xml/scap/ssg/content/ssg-rl9-ds.xml
# lists every profile bundled in this content file — confirm you're
# choosing the profile that actually matches your compliance requirement
oscap xccdf eval --profile xccdf_org.ssgproject.content_profile_cis \
  --results results.xml --report report.html \
  /usr/share/xml/scap/ssg/content/ssg-rl9-ds.xml
```

Open `report.html` and find at least one `fail` result. Identify its rule
ID and what it's actually checking for.

### Task 3 — Triage a findings list

```bash
# From either tool's output, pick 5 findings and rank them using §1.4's
# order — direct exploit path, regression against this series' own work,
# environment-specific low-risk, everything else.
```

### Task 4 — Recognize which findings map back to Demos M08-01/M08-02/M08-03

For each of these hypothetical findings, name which earlier demo's work
it relates to, and what "fixed" actually looks like per that demo:

```
A: "SSH root login is permitted"
B: "SELinux is not enforcing"
C: "A certificate on this host expires in 3 days"
```

<details>
<summary>Answers</summary>

**A** — Demo M08-01 §1.4: `PermitRootLogin no` in `sshd_config`, validated
with `sshd -t` before restarting.

**B** — Demo M08-02: confirm with `getenforce`; if it's Permissive rather
than Enforcing, that's either a deliberate diagnostic state that should
have been reverted, or a regression — either way it needs addressing, not
silencing the finding.

**C** — Demo M08-03 §1.4/§1.6: this is the exact expiry scenario — reissue
before it lapses, and confirm the consuming service actually reloads the
new certificate once issued, not just that a new file exists on disk.

</details>

## What You Learned

- ✅ Explained what `lynis` and OpenSCAP are each for, and how they differ
- ✅ Ran a `lynis` audit and read its hardening index and suggestions
- ✅ Ran an OpenSCAP scan against a specific profile and read a
  compliance report
- ✅ Triaged a findings list instead of treating every line as equally
  urgent
- ✅ Recognized which findings map back to gaps in Demos M08-01/M08-02/M08-03's own
  work
- ✅ Explained why a baseline scan runs on a schedule, not once

## Part 3 — Break-Fix Scenario

### Scenario — NovaTech Ops: the audit that "passed" last quarter

> **Incident report — NovaTech Ops, ticket #5240**
>
> "We ran an OpenSCAP scan against the CIS profile last quarter and it
> passed. This week's incident review found SELinux had been in
> Permissive mode on that same host for six weeks, discovered only after
> an actual intrusion attempt succeeded further than it should have."

Tasks:
1. Does a compliance scan passing months ago tell you anything about the
   system's state today? Reference §1.6.
2. What kind of scanning cadence would have caught the Permissive-mode
   drift before it mattered, and how?
3. Was the choice of CIS as the profile itself relevant to this specific
   miss, or was the problem entirely about *when* the scan ran rather than
   *which* profile it used?

<details>
<summary>Answers</summary>

A scan passing months ago says nothing about today's state — it's a
snapshot, not a continuous guarantee, exactly per §1.6. A scheduled,
recurring scan (weekly or more often, via a systemd timer per Demo M02-05)
with each run's results diffed against the previous one would have
surfaced "SELinux enforcement status changed" as a new finding within days
of the mode actually changing, not left it undiscovered for six weeks.
The profile choice (CIS vs. another standard) isn't really the issue
here — most standard profiles, CIS included, do check SELinux/AppArmor
enforcement status; the actual gap was cadence, not coverage. A single
scan, however good the profile, only ever proves compliance at one
instant in time.

</details>

## Interview Prep

**Q1. What's the practical difference between `lynis` and OpenSCAP, and when would you reach for each?**

`lynis` is broad and heuristic — no external standard required, good for
an ongoing general baseline. OpenSCAP scans against a specific, named,
published profile (CIS, STIG, etc.) and produces findings mapped to rule
IDs — the right tool when the actual requirement is demonstrating
compliance with a specific external standard, not just general hardening.

---

**Q2. A `lynis`/OpenSCAP run produces 200 findings. What's wrong with treating this as a flat, top-to-bottom to-do list?**

Not every finding is equally urgent — some have a direct, known exploit
path; some are genuinely low-risk given the specific environment (a
physical-console hardening suggestion on a cloud VM with no console at
all); some represent an actual regression against something this series
already deliberately configured and verified. Triage by actual risk and
context, not by the order the tool happened to print them in.

---

**Q3. Why does a baseline security scan need to run on a schedule rather than once?**

A one-time scan only proves compliance at that instant. Configuration
drifts afterward — a package update resets a setting, a debugging change
never gets reverted, a certificate crosses its renewal window — and none
of that is caught by a scan that already ran. A scheduled, recurring scan
with results diffed against the previous run catches new findings close
to when they actually appear, rather than leaving them undiscovered for
months.

---

**Q4. A host passed an OpenSCAP CIS scan last quarter, but an internal MAC setting had silently drifted to a less secure state in the meantime. Was the tool wrong?**

No — the tool correctly reported the system's state at the time it ran.
The gap wasn't in the scan itself; it was in cadence — nothing re-checked
the system between that scan and the incident. This is exactly the
argument for scheduled scanning (§1.6), not a criticism of either tool's
accuracy.

---

## Key Takeaways

1. **`lynis` is broad and heuristic; OpenSCAP is narrow and
   standards-based.** Use `lynis` for an ongoing general baseline; use
   OpenSCAP when you must demonstrate compliance with a specific,
   published standard via named rule IDs.
2. **Neither tool fixes anything — both produce a prioritized-looking
   list that still needs actual triage.** A finding being reported
   doesn't automatically make it the right priority for this specific
   environment.
3. **A finding that maps back to something this series already
   configured (SSH hardening, MAC enforcement, valid certificates)
   represents a regression, not just an unaddressed gap** — treat it with
   more urgency than a fresh, never-addressed suggestion.
4. **A scan is a snapshot, not a continuous guarantee.** Passing today
   says nothing about next month; drift happens silently (a reset config,
   a forgotten debugging change, an expiring certificate) between scans.
5. **Run baseline scans on a schedule, with results diffed against the
   previous run**, so a new finding gets attention close to when it
   actually appeared, not buried among a hundred already-triaged ones
   months later.

> **Demo scope:** Primary concept: systematic, repeated security
> auditing with `lynis` and OpenSCAP. Supporting concepts: hardening
> index vs. compliance-profile scanning, findings triage, scan cadence.
> Estimated completion time: 30 minutes (reading + hands-on + verification).
> Checkpoints: 4 natural stopping points (after Task 1 through Task 4).

## Quick Commands Reference

| Command | Purpose |
|---|---|
| `lynis audit system` | Run a full baseline audit |
| `lynis show profiles` | List available `lynis` scan profiles |
| `oscap info CONTENT.xml` | List SCAP profiles available in a content file |
| `oscap xccdf eval --profile PROFILE --report FILE CONTENT.xml` | Run a scan against a named profile |

## Appendix A — Anki Cards

**`m08-04-security-auditing-anki.csv`:**

```
#deck:Linux Mastery::Module 8 - Security::Demo M08-04 - Security Auditing Tools
#separator:Comma
#columns:Front,Back,Tags
"What's the fundamental difference between lynis and OpenSCAP?","lynis is broad and heuristic, requiring no external standard — good for general ongoing baselines. OpenSCAP scans against a specific, named, published profile (CIS, STIG) and maps findings to rule IDs — used to demonstrate compliance with a specific standard.","demo-m08-04,lynis,openscap,comparison"
"What does lynis's 'hardening index' represent, and why shouldn't it be the primary goal?","An overall pass-rate summary (0-100). It shouldn't be the primary goal because a handful of specific high-severity WARNINGs matter far more than a marginal index increase from fixing several low-severity SUGGESTIONs.","demo-m08-04,lynis,hardening-index"
"Why does the choice of SCAP profile matter as much as running an OpenSCAP scan at all?","Scanning against the wrong profile (e.g. CIS when the actual requirement is DISA STIG) produces a report that answers the wrong compliance question entirely, even though the scan itself completes successfully.","demo-m08-04,openscap,profile"
"Why is treating a 200-finding audit report as a flat top-to-bottom to-do list a mistake?","Findings aren't equally urgent — some have a direct known exploit path, some are genuinely low-risk in the specific environment, and some represent a real regression against something already deliberately configured. Context changes priority in ways neither tool fully knows on its own.","demo-m08-04,triage,best-practice"
"Why does a security baseline scan need to run on a schedule rather than once?","A one-time scan only proves compliance at that instant. Configuration drifts afterward (package updates, forgotten debugging changes, expiring certificates) and none of that is caught by a scan that already ran.","demo-m08-04,scan-cadence,fundamentals"
"A host passed a compliance scan last quarter but had a MAC setting drift to Permissive mode in the meantime, undetected. Was the scan tool at fault?","No — the tool correctly reported the system's state at the time it ran. The gap was in cadence: nothing re-scanned the system between that pass and the drift. This argues for scheduled, diffed scanning, not a tool criticism.","demo-m08-04,scan-cadence,break-fix"
"If a baseline scan reports 'SSH root login is permitted,' which earlier demo's fix does that map to?","Demo M08-01 (SSH hardening): PermitRootLogin no in sshd_config, validated with sshd -t before restarting.","demo-m08-04,cross-demo,ssh"
```

## Appendix B — Quiz

**`m08-04-security-auditing-quiz.md`:**

````markdown
# Quiz — Demo M08-04: Security Auditing Tools

> One correct answer per question unless stated otherwise.
> Target: 80% or above — this is the final quiz in Module 8.

---

**Q1.** When would you reach for OpenSCAP over `lynis`?

- A) When you need a quick, general-purpose baseline with no external standard required
- B) When you must demonstrate compliance with a specific, published standard (CIS, STIG) via named rule IDs
- C) OpenSCAP and lynis are interchangeable in every situation
- D) Only when auditing a host with no network connectivity

<details>
<summary>Answer</summary>

**B** — This is OpenSCAP's specific value: standards-mapped findings, not
just general hardening suggestions.

</details>

---

**Q2.** Why shouldn't `lynis`'s hardening index alone be treated as the goal to optimize?

- A) The index is randomly generated and has no real meaning
- B) A handful of specific, high-severity findings matter far more than a marginal index increase from fixing several low-severity ones
- C) The index only ever measures disk usage, not security posture
- D) Raising the index requires disabling SELinux

<details>
<summary>Answer</summary>

**B** — Chasing the number can mean fixing easy, low-impact items while
higher-severity findings remain unaddressed.

</details>

---

**Q3.** An OpenSCAP scan is run against a CIS profile when the actual
organizational requirement is DISA STIG compliance. What's the problem?

- A) There is no problem — any profile satisfies any compliance requirement
- B) The report answers the wrong compliance question entirely, even though the scan itself completes successfully
- C) CIS and STIG profiles always produce identical results
- D) OpenSCAP cannot run against CIS profiles at all

<details>
<summary>Answer</summary>

**B** — A successfully-run scan against the wrong profile still fails to
satisfy the actual requirement.

</details>

---

**Q4.** A baseline audit produces 200 findings. What's the correct first
step?

- A) Fix every finding in the exact order the tool printed them
- B) Triage by actual risk and context — direct exploit paths and regressions against previously-configured hardening first, environment-specific low-risk findings last
- C) Ignore all findings below a `[WARNING]` severity permanently
- D) Re-run the scan until the number of findings decreases on its own

<details>
<summary>Answer</summary>

**B** — Not every finding carries equal urgency, and context the tool
doesn't fully know (cloud VM vs. physical server, internal vs.
internet-facing) changes real priority.

</details>

---

**Q5.** A finding reports that SELinux is in Permissive mode, on a host
where Demo M08-02's work had it Enforcing. How should this be treated
relative to a brand-new, never-addressed finding?

- A) Identically — all findings carry the same priority
- B) With MORE urgency — it represents an actual regression against something already deliberately configured and verified, not just an unaddressed baseline gap
- C) With LESS urgency, since it was presumably intentional
- D) It should be ignored, since Permissive mode is equivalent to Enforcing for practical purposes

<details>
<summary>Answer</summary>

**B** — Per §1.4/§1.5, a regression against prior deliberate work is a
stronger signal than a fresh, never-addressed suggestion.

</details>

---

**Q6.** Why should a security baseline scan run on a schedule rather than
just once, right after hardening a system?

- A) Scheduled scans are required by law in every jurisdiction
- B) Configuration drifts after the initial scan — package updates, forgotten debugging changes, expiring certificates — and a one-time scan can't catch any of that
- C) Running lynis or OpenSCAP more than once always produces different, contradictory results on an unchanged system
- D) There is no real reason; a single scan is sufficient if done thoroughly

<details>
<summary>Answer</summary>

**B** — A scan is a snapshot of that moment; drift after it happens
silently unless something re-checks the system.

</details>

---

**Q7.** A host passed a CIS-profile OpenSCAP scan last quarter. This
quarter, an incident revealed a MAC setting had drifted to a less secure
state weeks earlier, undetected. What does this indicate about the scan
tool itself?

- A) The tool was wrong or buggy — a passed scan should guarantee ongoing compliance
- B) Nothing wrong with the tool — it correctly reported the state at the time it ran; the actual gap was scanning cadence, not tool accuracy
- C) CIS profiles never check MAC/SELinux status, so this was expected
- D) OpenSCAP scans automatically re-run themselves; this shouldn't have been possible

<details>
<summary>Answer</summary>

**B** — This is the direct lesson from this demo's Break-Fix scenario —
a scan proves compliance at one instant, not continuously.

</details>

---

Score guide:

| Score | Action |
|---|---|
| 7/7 | Import Anki cards — Module 8 complete |
| 6/7 | Review the wrong answer, then proceed |
| 5/7 | Re-read the relevant section, retry those questions |
| Below 5/7 | Re-read the full demo and redo the walkthrough before proceeding |
````