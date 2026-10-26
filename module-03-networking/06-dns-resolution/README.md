# Demo M03-06 — DNS Resolution Path Failure

## Overview

An application logging intermittent `ECONNREFUSED` or timeout errors
that only happen on the *first* connection to a given hostname is a
classic symptom with a non-obvious cause: DNS resolution latency or
misconfiguration, not an application bug. This demo covers diagnosing a
DNS resolution path end to end — confirming which resolver is actually
in use (not just what a config file says), measuring resolution latency
separately from application-level timeouts, and walking the full
resolution chain when a lookup is slow or failing.

## Demo Objectives

1. Measure DNS resolution latency directly, separate from
   application-level timeout symptoms
2. Confirm which resolver path is actually active, rather than trusting
   a config file's contents at face value
3. Walk a full resolution path from root to authoritative to isolate
   which hop is slow or failing
4. Read resolver-side query logs when you administer the resolver
   yourself
5. Distinguish a DNS-latency cause from a TCP/TLS-latency cause using
   timestamp correlation against application logs

## Directory Structure

```
06-dns-resolution/
├── README.md                       # this file
├── m03-06-dns-resolution-anki.csv  # Anki flashcard deck
└── m03-06-dns-resolution-quiz.md   # standalone quiz
```

## Environments

| Task | Ubuntu 24.04 | Rocky Linux 9.8 | EC2 Amazon Linux |
|---|---|---|---|
| `getent hosts` | preinstalled | preinstalled | Not required |
| `dig` | `sudo apt install dnsutils` | `sudo dnf install bind-utils` | Not required |
| `resolvectl` (systemd-resolved) | preinstalled on default Ubuntu | not applicable (Rocky doesn't default to systemd-resolved) | Not required |

> **Verification note:** `getent hosts`, reading `/etc/nsswitch.conf`/
> `/etc/resolv.conf`, and the "empty resolv.conf causes an immediate
> failure" case were all run live in this sandbox. `dig`, `resolvectl`,
> and resolver-side log commands (`journalctl -u named`/`dnsmasq`)
> couldn't be installed or run here (no network for package installs) —
> those sections are doc-sourced, marked `[Unverified — test before
> use]`. One specific thing is worth disclosing: **this sandbox's own
> network setup transparently intercepts DNS queries regardless of
> `/etc/resolv.conf`'s configured nameserver** — pointing it at an
> unreachable test address (`203.0.113.1`) still resolved successfully
> and quickly, rather than timing out as the scenario below expects on a
> normal host. This is itself a real example of §1.2's point: never trust
> a config file's contents over confirming what's actually happening.

## Recall Check — 04-security-auditing

Answer these from memory before continuing — no looking back at Demo
M08-04.

**Q1.** What's the fundamental difference between `lynis` and OpenSCAP?

**Q2.** Why does a security baseline scan need to run on a schedule
rather than just once?

**Q3.** A finding maps back to something an earlier demo already
configured and verified. How should it be prioritized relative to a
brand-new finding?

<details>
<summary>Answers</summary>

**A1.** `lynis` is broad and heuristic, requiring no external standard.
OpenSCAP scans against a specific, named, published profile (CIS, STIG)
and maps findings to rule IDs.

**A2.** A one-time scan only proves compliance at that instant;
configuration drifts afterward and a scan that already ran can't catch
that.

**A3.** With MORE urgency — it represents an actual regression against
something already deliberately configured, not just an unaddressed
baseline gap.

</details>

## Part 1 — Concepts

### 1.1 Measuring DNS latency separately from application latency

An application reporting "connection timeout" or "connection refused" on
the *first* request to a hostname, but working fine on subsequent
requests to the same hostname, is a specific, recognizable shape — it
points at something that only happens once per hostname and is then
cached, which is exactly how DNS resolution behaves.

```bash
getent hosts example.com
```

Real output:

```
2606:4700:10::6814:179a example.com
2606:4700:10::ac42:93f3 example.com
```

```bash
time dig example.com
```

`[Unverified — test before use]` `dig` reports its own query time
explicitly (`;; Query time: N msec`), which is the direct way to separate
"DNS itself was slow" from "the connection after resolution was slow" —
`getent` (used above) answers whether resolution *succeeds*, but doesn't
report timing the way `dig` does.

### 1.2 Confirming which resolver is actually active

```bash
cat /etc/resolv.conf
```

**A config file's content is not proof of what's actually being used.**
Real, directly-observed proof from this very sandbox: `/etc/resolv.conf`
was deliberately set to an unreachable address
(`nameserver 203.0.113.1`), and lookups still succeeded, quickly —
something in this network path was resolving queries regardless of what
the file said. On a normal host the far more common version of this same
lesson is `systemd-resolved` or NetworkManager silently managing
resolution through its own stub listener, with `/etc/resolv.conf` either
symlinked to a generated file or simply not reflecting what's actually
being queried.

```bash
resolvectl status
```

`[Unverified — test before use]` On a system using `systemd-resolved`,
this shows the resolver actually in effect per network interface — the
authoritative answer to "what's really being used," which a bare `cat
/etc/resolv.conf` cannot guarantee on such a system.

**The general principle, true regardless of which specific mechanism is
intercepting or overriding resolution**: when a DNS-related fix doesn't
behave as expected, confirm what's actually resolving the query before
assuming the config file you edited is the thing in effect.

### 1.3 Walking the full resolution path

```bash
dig +trace example.com
```

`[Unverified — test before use]` `+trace` walks the resolution
iteratively from the root servers down through each delegation
(root → TLD → authoritative), showing the response time and referral at
every hop — the direct way to isolate *which specific hop* is slow or
failing, rather than only knowing that resolution overall was slow.
Without `+trace`, a normal `dig` query only shows the final answer (or
failure), hiding which step in the chain actually caused it.

### 1.4 The server-administration half — reading resolver logs

If you administer the resolver itself (not just querying as a client),
the query logs show what the resolver saw and did:

```bash
journalctl -u named       # BIND
journalctl -u dnsmasq     # dnsmasq
```

`[Unverified — test before use]` This is the half of DNS troubleshooting
that's commonly skipped when only client-side tools (`dig`,
`resolv.conf`) are taught — a client-side trace shows symptoms, but the
resolver's own log shows whether it forwarded the query upstream and got
no response, hit its own cache, or refused the query outright, which
determines where the actual fix belongs.

### 1.5 Confirming DNS, not TCP/TLS, is the actual cause

Correlate the *timestamp* of an application's first-connection failure
against DNS query timing (§1.1) and the application's own connection
attempt timing. If the DNS lookup's own timing accounts for most or all
of the delay before the application even attempts a TCP connection, the
cause is DNS; if DNS resolves quickly and the delay happens afterward
(during the TCP handshake or TLS negotiation), the cause is elsewhere
entirely, and no amount of DNS tuning will fix it. This distinction
matters because the two have completely different fixes (§1.6 tunes
resolution; a TCP/TLS-side problem needs Demo M03-08/Demo M08-03's tools
instead).

### 1.6 Remediation

```
Local caching resolver (systemd-resolved/dnsmasq):
  repeat lookups hit the local cache instead of re-querying a slow
  upstream every time — the single highest-leverage fix for "the FIRST
  connection is slow, subsequent ones are fine."

Sane timeout/attempts in resolv.conf:
  options timeout:2 attempts:2
  rather than inheriting a long default retry window that makes a
  genuinely failing upstream take many seconds to finally give up.

Restore the known-good config after any deliberate experiment:
  sudo cp /etc/resolv.conf.bak /etc/resolv.conf

Monitor DNS resolution time as its OWN metric, separate from total
request latency — the fastest way to catch this class of incident
before it's customer-visible, rather than discovering it only after
users report slow first-connections.
```

## Part 2 — Hands-On Lab

### Commands used in this demo

| Command | Purpose |
|---|---|
| `getent hosts HOSTNAME` | Resolve a hostname via the full NSS path (files, then DNS) |
| `cat /etc/nsswitch.conf` | Confirm the configured hosts-lookup order |
| `dig HOSTNAME` | Query DNS directly, reporting its own query time |
| `dig +trace HOSTNAME` | Walk the full resolution path, hop by hop |
| `resolvectl status` | Confirm the resolver actually in effect (systemd-resolved systems) |
| `journalctl -u named` / `-u dnsmasq` | Resolver-side query logs, if you administer it |

### Task 1 — Baseline resolution and the NSS path

```bash
grep hosts /etc/nsswitch.conf
getent hosts example.com
```

Confirm the `hosts:` line's order (commonly `files dns`) and that
`getent` successfully resolves against whichever comes first.

### Task 2 — Confirm a config file's contents aren't proof of behavior

```bash
cp /etc/resolv.conf /tmp/resolv.conf.bak
cat /etc/resolv.conf
echo "nameserver 203.0.113.1" | sudo tee /etc/resolv.conf
echo "options timeout:1 attempts:1" | sudo tee -a /etc/resolv.conf
cat /etc/resolv.conf
getent hosts example.com; echo "exit: $?"
```

**On most normal hosts**, this should now fail or hang (per the original
scenario this demo is built from) — `203.0.113.1` is a non-routable
test-net address with nothing listening. If it instead succeeds quickly
despite the unreachable nameserver, something in your network path
(a local proxy, `systemd-resolved`'s stub listener, a container network's
DNS redirect) is intercepting or overriding resolution — confirm which,
using `resolvectl status` or your platform's equivalent, rather than
trusting `/etc/resolv.conf` at face value. Restore the original
afterward:

```bash
sudo cp /tmp/resolv.conf.bak /etc/resolv.conf
```

### Task 3 — Measure resolution latency directly

```bash
time dig example.com
```

Identify the `;; Query time:` line in `dig`'s own output and compare it
against how long the overall command took to return.

### Task 4 — Walk the resolution path

```bash
dig +trace example.com
```

Identify which hop (root, TLD, or authoritative) took the longest, or
where a referral failed, if any did.

### Task 5 — Correlate DNS timing against an application symptom

```
Given:
  Application log: "connect timeout to api.internal at 14:02:03.100,
                     succeeded at 14:02:03.850 (750ms delay)"
  dig query time for the same hostname, run immediately after: 12ms

Is DNS the cause of this specific 750ms delay? What else should be
checked instead?
```

<details>
<summary>Answer</summary>

No — a 12ms DNS query time cannot account for a 750ms delay. DNS is
very likely not the bottleneck here, even though the symptom (slow first
connection) superficially matches a DNS-latency pattern. The next place
to look is the TCP handshake and TLS negotiation itself (Demo M03-08's
tooling for network-level latency, or Demo M08-03's tooling if this is a
certificate-chain-related TLS delay) — the DNS-latency hypothesis should
be ruled out by measurement, not assumed just because the symptom
matches the general shape.

</details>

## What You Learned

- ✅ Measured DNS resolution latency directly, separate from
  application-level timeout symptoms
- ✅ Confirmed which resolver path is actually active, rather than
  trusting a config file's contents
- ✅ Walked a full resolution path to isolate which hop is slow or
  failing
- ✅ Read resolver-side query logs for server-administration-level
  diagnosis
- ✅ Distinguished a DNS-latency cause from a TCP/TLS-latency cause using
  timestamp correlation

## Part 3 — Break-Fix Scenario

### Scenario — NovaTech Ops: "I fixed resolv.conf, but nothing changed"

> **Incident report — NovaTech Ops, ticket #5410**
>
> "An engineer noticed `/etc/resolv.conf` pointed at a decommissioned
> internal DNS server and updated it to the correct one. The intermittent
> first-connection failures continued completely unchanged, as if the
> edit never happened."

Tasks:
1. Reference §1.2 — name two different, realistic mechanisms that could
   explain why editing `/etc/resolv.conf` directly had no observable
   effect.
2. What single command (per §1.2) would distinguish between these
   possibilities, rather than guessing?
3. Having confirmed which mechanism it is, where does the actual fix need
   to be applied — the same file, or somewhere else entirely?

<details>
<summary>Answer</summary>

Two realistic mechanisms: (1) `systemd-resolved` or NetworkManager is
managing resolution through its own stub resolver, with `/etc/resolv.conf`
either a symlink to a generated file that gets regenerated on the next
network event (silently reverting the manual edit) or simply not what's
actually being queried; (2) a network-level mechanism (a transparent
proxy, a container network's DNS redirect, a corporate network's DNS
interception policy) redirects outbound DNS queries regardless of the
configured nameserver — exactly what this demo's own verification note
disclosed happening in its authoring sandbox. `resolvectl status` (or
confirming whether `/etc/resolv.conf` is a symlink and to where, on
systems without `resolvectl`) distinguishes the first case; if that shows
nothing relevant, testing resolution against a deliberately unreachable
nameserver (Task 2's exact experiment) and observing whether it still
succeeds distinguishes the second. Depending on which is confirmed, the
actual fix belongs either in `systemd-resolved`'s own configuration (not
the symlinked/generated `resolv.conf`) or in whatever network-level
policy is performing the redirect — editing `/etc/resolv.conf` again, in
either case, accomplishes nothing, which is exactly the symptom reported.

</details>

## Interview Prep

**Q1. An application fails on the first connection to a hostname but works fine afterward. What specific pattern does this suggest, and why?**
DNS resolution — a cold lookup that then gets cached, so subsequent
connections to the same hostname skip the slow or failing step entirely.
This specific "first time only" shape is the recognizable signature;
measuring DNS query time directly (`dig`) confirms or rules it out rather
than assuming based on the pattern alone.

**Q2. Someone edits `/etc/resolv.conf` to fix a DNS issue, and nothing changes. What should you check before assuming the edit was wrong?**
Whether `/etc/resolv.conf` is actually what's being used at all —
`systemd-resolved`/NetworkManager commonly manage resolution through
their own mechanism, with the file itself sometimes a symlink to a
generated file that gets regenerated and silently overwrites a manual
edit. `resolvectl status` (or checking what the file symlinks to)
confirms what's actually active before assuming the edit's content was
the problem.

**Q3. Why does `dig +trace` provide information a plain `dig` query doesn't?**
A plain query only shows the final answer or failure. `+trace` walks the
resolution path hop by hop — root, TLD, authoritative — showing response
time and referral at each step, which is what actually isolates *where*
in the chain a slow or failing resolution is happening, rather than only
knowing that it is.

**Q4. Why is monitoring DNS resolution time as its own separate metric valuable, rather than only tracking total request latency?**
Total request latency conflates DNS time, connection time, and
application processing time into one number — a DNS-specific latency
spike can be completely hidden inside an otherwise-normal-looking total.
Tracking it separately is what catches this exact class of incident
(first-connection-only delays) before it becomes customer-visible,
rather than discovering it only after the fact from vague "sometimes
slow" complaints.

---

## Key Takeaways

1. **"Fails on first connection, succeeds after" is DNS's recognizable
   signature** — a cold lookup followed by caching. Confirm with direct
   DNS query timing rather than assuming from the pattern alone.
2. **A config file's contents are not proof of what's actually
   happening.** `systemd-resolved`, NetworkManager, or network-level
   interception can all cause `/etc/resolv.conf` to be irrelevant to
   actual resolution behavior — confirm the active resolver directly.
3. **`dig +trace` isolates which specific hop is slow or failing**,
   where a plain query only shows the overall result.
4. **Resolver-side logs (`journalctl -u named`/`dnsmasq`) are the
   server-administration half of DNS troubleshooting**, commonly skipped
   when only client-side tools are taught.
5. **Correlate DNS query timing against the actual application-level
   delay before concluding DNS is the cause** — a symptom that looks like
   a DNS problem can still turn out to be TCP/TLS-side instead.
6. **Monitor DNS resolution time as its own metric**, separate from total
   request latency, to catch this class of incident before it's
   customer-visible.

> **Demo scope:** Primary concept: diagnosing a DNS resolution path
> failure end to end. Supporting concepts: query-time measurement,
> confirming the active resolver, full-path tracing, resolver-side log
> reading, DNS-vs-TCP/TLS correlation.
> Estimated completion time: 35 minutes (reading + hands-on + verification).
> Checkpoints: 5 natural stopping points (after Task 1 through Task 5).

## Quick Commands Reference

| Command | Purpose |
|---|---|
| `getent hosts HOSTNAME` | Resolve via the full NSS path |
| `grep hosts /etc/nsswitch.conf` | Confirm lookup order (files/dns) |
| `dig HOSTNAME` | Query DNS directly, with its own reported query time |
| `dig +trace HOSTNAME` | Walk the full resolution path hop by hop |
| `resolvectl status` | Confirm the resolver actually in effect |
| `journalctl -u named` / `-u dnsmasq` | Resolver-side query logs |

## Appendix A — Anki Cards

**`m03-06-dns-resolution-anki.csv`:**

```
#deck:Linux Mastery::Module 3 - Networking::Demo M03-06 - DNS Resolution Path Failure
#separator:Comma
#columns:Front,Back,Tags
"An application fails only on the FIRST connection to a hostname, then works fine. What pattern does this suggest?","DNS resolution — a cold lookup gets cached, so subsequent connections skip the slow/failing step entirely. This specific 'first time only' shape is DNS's recognizable signature.","demo-m03-06,dns,fundamentals"
"Why isn't /etc/resolv.conf's content proof of what resolver is actually being used?","systemd-resolved, NetworkManager, or network-level DNS interception can all make the file's content irrelevant to actual resolution behavior — the file can be a symlink to a generated file that gets silently regenerated, or queries can be redirected regardless of the configured nameserver.","demo-m03-06,dns,resolv-conf,common-mistake"
"What does resolvectl status show that cat /etc/resolv.conf cannot guarantee?","The resolver actually in effect per network interface on a systemd-resolved system — the authoritative answer to 'what's really being queried,' which the file's contents alone don't guarantee on such a system.","demo-m03-06,dns,resolvectl"
"What does dig +trace show that a plain dig query doesn't?","It walks the resolution path hop by hop (root -> TLD -> authoritative), showing response time and referral at each step — isolating WHICH hop is slow or failing, not just that resolution overall was slow.","demo-m03-06,dns,dig-trace"
"Why are resolver-side logs (journalctl -u named/dnsmasq) considered the commonly-skipped half of DNS troubleshooting?","Client-side tools (dig, resolv.conf) only show symptoms from the querying side. The resolver's own log shows whether it forwarded upstream and got no response, hit its cache, or refused the query — determining where the actual fix belongs.","demo-m03-06,dns,server-side"
"An application's connection delay is 750ms, but dig reports a 12ms query time for the same hostname. Is DNS the cause?","No — a 12ms query time cannot account for a 750ms delay. The DNS-latency hypothesis should be ruled out by measurement, not assumed just because the symptom superficially matches a DNS pattern. Check TCP/TLS-side latency instead.","demo-m03-06,dns,troubleshooting,correlation"
"What's the single highest-leverage fix for 'first connection is slow, subsequent ones are fine'?","A local caching resolver (systemd-resolved or dnsmasq) — repeat lookups hit the local cache instead of re-querying a slow upstream every time.","demo-m03-06,dns,remediation"
"Why should DNS resolution time be monitored as its own separate metric, not folded into total request latency?","Total request latency conflates DNS time, connection time, and processing time into one number, hiding a DNS-specific latency spike. Tracking it separately catches this class of incident before it's customer-visible.","demo-m03-06,dns,monitoring"
```

## Appendix B — Quiz

**`m03-06-dns-resolution-quiz.md`:**

````markdown
# Quiz — Demo M03-06: DNS Resolution Path Failure

> One correct answer per question unless stated otherwise.
> Target: 80% or above before moving to Demo M03-07.

**Q1. An application fails on the first connection to a hostname, then succeeds on every subsequent connection. What does this pattern suggest?**

- A) A memory leak in the application
- B) DNS resolution — a cold lookup followed by caching, so later connections skip the slow/failing step
- C) A firewall rule that only blocks the first packet
- D) Disk I/O contention

<details>
<summary>Answer</summary>

**B** — This "first time only" shape is DNS's recognizable signature.

</details>

---

**Q2. An engineer edits `/etc/resolv.conf` to fix a DNS problem, and nothing changes. What should be checked first?**

- A) Whether the edit was saved correctly, and nothing else
- B) Whether `/etc/resolv.conf` is actually what's being used for resolution at all — `systemd-resolved`/NetworkManager or network-level interception can make its content irrelevant
- C) Whether the hostname itself has a typo
- D) Whether the application has been restarted

<details>
<summary>Answer</summary>

**B** — Confirming the actual active resolver (`resolvectl status` or
equivalent) comes before assuming the file's content was the problem.

</details>

---

**Q3. What does `dig +trace` provide that a plain `dig` query doesn't?**

- A) A faster overall query time
- B) A hop-by-hop walk of the resolution path (root, TLD, authoritative), isolating which specific hop is slow or failing
- C) Automatic caching of the result
- D) The ability to query multiple hostnames at once

<details>
<summary>Answer</summary>

**B** — A plain query only shows the final answer or failure; `+trace`
shows where in the chain the problem actually is.

</details>

---

**Q4. Why are resolver-side logs (`journalctl -u named`/`dnsmasq`) described as commonly skipped?**

- A) They don't actually contain useful information
- B) Only client-side tools (`dig`, `resolv.conf`) are typically taught, leaving the resolver's own perspective (forwarded upstream, cache hit, refused) unexamined
- C) They require a paid support contract to access
- D) They only exist on Windows DNS servers

<details>
<summary>Answer</summary>

**B** — The resolver's own log determines where the actual fix belongs,
which client-side symptoms alone can't show.

</details>

---

**Q5. `dig` reports a 12ms query time for a hostname, but the application experienced a 750ms delay connecting to it. Is DNS the cause?**

- A) Yes, 12ms is still too slow and should be treated as the cause
- B) No — 12ms cannot account for a 750ms delay; the cause is more likely in TCP/TLS, and should be checked with the appropriate tools instead
- C) Yes, because any non-zero DNS time contributes to total latency equally
- D) It's impossible to determine without re-running the test 100 times

<details>
<summary>Answer</summary>

**B** — The DNS-latency hypothesis should be ruled out by measurement,
not assumed from a superficially matching symptom.

</details>

---

**Q6. What's the single highest-leverage fix for "the first connection to a hostname is slow, subsequent ones are fine"?**

- A) Increasing the application's own connection timeout
- B) A local caching resolver (systemd-resolved/dnsmasq), so repeat lookups hit the local cache instead of re-querying a slow upstream
- C) Disabling DNS resolution entirely and using hardcoded IPs everywhere
- D) Restarting the application server more frequently

<details>
<summary>Answer</summary>

**B** — This directly targets the cold-lookup-then-cached pattern driving
the symptom.

</details>

---

**Q7. Why should DNS resolution time be monitored as its own metric, separate from total request latency?**

- A) DNS metrics are required by a specific compliance standard
- B) Total request latency conflates DNS, connection, and processing time into one number, hiding a DNS-specific spike — tracking it separately catches this class of incident before it's customer-visible
- C) DNS resolution time is always zero and not worth tracking
- D) Total request latency is more accurate and DNS-specific tracking is redundant

<details>
<summary>Answer</summary>

**B** — Separating the metric is what makes this specific failure mode
visible before customers notice it.

</details>

---

Score guide:

| Score | Action |
|---|---|
| 7/7 | Import Anki cards, move to Demo M03-07 |
| 6/7 | Review the wrong answer, then proceed |
| 5/7 | Re-read the relevant section, retry those questions |
| Below 5/7 | Re-read the full demo and redo the walkthrough before proceeding |
````