# Demo 12 — Network Troubleshooting & Firewalls

## Overview

Demo 11 got a machine addressed, routed, resolvable, and reachable over SSH.
This demo covers the other half of networking: watching what's actually
happening on the wire, finding out which process owns which socket, and
controlling what traffic is allowed in and out in the first place. By the
end, you'll have a repeatable decision tree for "why can't this connection
get through" instead of guessing.

## Demo Objectives

1. Read `ss` output to identify listening sockets and connection states
2. Use `lsof` for full network-focused investigation: which process owns which port
3. Capture and read basic `tcpdump` output to confirm traffic is actually arriving
4. Configure `firewalld` (Rocky9) and `ufw` (Ubuntu) to allow/deny traffic
5. Configure NAT/port redirection with each firewall tool
6. Apply a systematic decision tree to diagnose "connection refused/timeout" issues

## Directory Structure

```
12-network-troubleshooting-firewalls/
└── README.md   # This file — full demo content, including Anki (Appendix A)
                 # and Quiz (Appendix B), extracted later via script.
```

## Environments

| Task | Ubuntu 24.04 | Rocky Linux 9.8 | EC2 Amazon Linux |
|---|---|---|---|
| Firewall tasks | `ufw` | `firewalld` | Not required (security groups differ) |
| `tcpdump` install | `sudo apt install tcpdump` | `sudo dnf install tcpdump` | Not required |

## Recall Check — 11-networking-fundamentals

Answer these from memory before continuing — no looking back at Demo 11.

**Q1.** A name resolves correctly via `dig` but a user's machine connects to
a different, stale server. What's the first thing to check, and why does
`dig` alone miss it?

**Q2.** An SSH key login fails and falls back to a password prompt. What two
things should you check on the server before assuming the key itself is
wrong?

**Q3.** Why does an address added with `ip addr add` disappear after a
reboot?

<details>
<summary>Answers</summary>

**A1.** `/etc/hosts` on that machine — `nsswitch.conf`'s default `hosts:
files dns` order checks `/etc/hosts` before DNS, so a stale local entry
silently overrides a correct DNS answer. `dig` queries DNS directly,
bypassing that order, so it can't reveal this class of problem.

**A2.** Permissions on `~/.ssh` (must be `700`) and `~/.ssh/authorized_keys`
(must be `600`) — `sshd` silently distrusts group/world-writable permissions
on these and falls back to password auth without an explicit error.

**A3.** `ip` modifies the kernel's live network state only — it's not the
persistence layer. Permanent changes require the distro's own network
configuration system (Netplan on Ubuntu, NetworkManager on Rocky9).

</details>

## Part 1 — Concepts

### 1.1 ss — the modern netstat replacement

```bash
ss -tulpn
```

```
Netid  State   Local Address:Port   Peer Address:Port  Process
tcp    LISTEN  0.0.0.0:22           0.0.0.0:*           users:(("sshd",pid=612,fd=3))
tcp    LISTEN  127.0.0.1:5432       0.0.0.0:*           users:(("postgres",pid=1204,fd=6))
```

Flag breakdown: `-t` (TCP), `-u` (UDP), `-l` (listening sockets only), `-p`
(show the owning process — requires root for other users' processes), `-n`
(numeric ports, skip `/etc/services` name lookup, faster and unambiguous).
`0.0.0.0:22` means sshd is listening on all interfaces; `127.0.0.1:5432`
means postgres is listening on localhost only — **this distinction matters
enormously for troubleshooting**: a service bound to `127.0.0.1` is
unreachable from any other machine no matter what the firewall allows, and
this is one of the most common "why can't I connect" root causes, easily
mistaken for a firewall problem.

```bash
ss -tan state established
# Show established TCP connections only — useful for "who is currently
# talking to this machine" without the noise of every listening socket.
```

`ss` replaces the older `netstat` (deprecated, sometimes not installed by
default) the same way `ip` replaced `ifconfig`/`route` in Demo 11 — same
underlying kernel data, faster and more capable tool.

### 1.2 lsof — full network investigation

Demo 02 and Demo 06 used `lsof | grep deleted` for a narrow purpose. Here's
the network side of `lsof`, in full:

```bash
sudo lsof -i                    # all network connections and listeners
sudo lsof -i :443                # what's using port 443 specifically
sudo lsof -i tcp -sTCP:LISTEN    # only TCP sockets in LISTEN state
sudo lsof -p 1204                # everything process 1204 has open, network or not
```

`lsof -i :PORT` answers the single most common port-conflict question
directly: "what's already using this port" — before you assume a service
failed to start for some other reason, when actually it's a `bind: address
already in use` failure from something else already holding that port.

```bash
sudo lsof -i -P -n
# -P: show raw port numbers, don't resolve to service names (faster, avoids
# ambiguity, same reasoning as ss -n)
# -n: don't resolve hostnames either
```

### 1.3 tcpdump — confirming traffic actually arrives

`ss` and `lsof` show you sockets and processes; neither tells you whether
packets are actually arriving on the wire. `tcpdump` does:

```bash
sudo tcpdump -i eth0 port 443 -n
# -i eth0: capture on this interface
# port 443: filter to this port only — capturing everything is noisy and
# usually unnecessary for a targeted question
# -n: don't resolve hostnames (same reasoning as ss -n / lsof -n — faster,
# and avoids DNS lookups interfering with the capture itself)
```

```bash
sudo tcpdump -i eth0 host 10.0.4.12 and port 22 -n
# Combine filters: only traffic to/from a specific host, on a specific port
```

**The diagnostic value of `tcpdump` is answering one specific question:** is
the traffic reaching this network interface at all? If a connection attempt
shows nothing in `tcpdump` on the destination server, the problem is
upstream of this machine entirely — routing, a firewall further out, a
security group, or the client never actually sending — and no amount of
`firewalld`/`ufw`/application-level troubleshooting on this machine will fix
it, because the traffic isn't arriving to be filtered or processed in the
first place. If `tcpdump` *does* show the traffic arriving but the
connection still fails, the problem is local — the firewall, or the
application itself.

### 1.4 firewalld (Rocky9) — zones and services

```bash
sudo firewall-cmd --get-active-zones
sudo firewall-cmd --zone=public --list-all
```

firewalld organizes rules into **zones** — named rule sets you assign
interfaces to, representing different trust levels (`public`, `internal`,
`trusted`, etc.). Rather than writing raw port rules, firewalld also
supports named **services** (predefined port/protocol bundles):

```bash
sudo firewall-cmd --zone=public --add-service=https --permanent
sudo firewall-cmd --zone=public --add-port=8080/tcp --permanent
sudo firewall-cmd --reload
```

**`--permanent` is not optional if you want the rule to survive a reload or
reboot** — without it, a rule applies only to the current runtime
configuration and vanishes on `--reload` or restart. This is the firewalld
equivalent of the `ip addr add` persistence trap from Demo 11: live changes
and permanent changes are separate, and forgetting `--permanent` is a
frequent source of "the rule worked, then mysteriously disappeared."

```bash
sudo firewall-cmd --zone=public --list-all
# Confirm both runtime and permanent state — a rule showing in --list-all
# after --reload confirms it actually persisted, not just applied once.
```

### 1.5 ufw (Ubuntu) — the simpler front end

```bash
sudo ufw status verbose
sudo ufw allow 443/tcp
sudo ufw allow from 10.0.4.0/24 to any port 5432
sudo ufw deny 23/tcp
```

`ufw` ("uncomplicated firewall") is a simpler front end over the same
underlying `netfilter`/`iptables` machinery firewalld also manages — no
zones, rules apply directly and immediately, no separate `--permanent`
concept to forget. This makes it faster for simple cases and less
expressive for complex multi-zone setups, which is exactly the tradeoff that
explains why Ubuntu defaults to it and Rocky9 defaults to the more elaborate
firewalld.

```bash
sudo ufw enable    # ufw is often installed but inactive by default — check
                    # status before assuming rules are being enforced at all
```

### 1.6 NAT and port redirection

Both tools can redirect incoming traffic on one port to a different internal
port or address — commonly used to expose a service running on a
non-privileged port (like `8080`) as though it were on the standard port
(`80`), without running the application itself as root.

```bash
# firewalld:
sudo firewall-cmd --zone=public --add-forward-port=port=80:proto=tcp:toport=8080 --permanent
sudo firewall-cmd --reload

# ufw (requires editing /etc/ufw/before.rules directly — ufw's simplicity
# doesn't extend to NAT the way firewalld's does):
# Add under *nat / :PREROUTING, then:
sudo ufw disable && sudo ufw enable   # reload rules including before.rules
```

The Ubuntu/ufw NAT workflow being noticeably clunkier than firewalld's is a
direct consequence of §1.5's tradeoff — ufw optimizes for simple allow/deny
rules, and NAT falls outside that simplicity, requiring dropping down to
edit the underlying rules file directly.

### 1.7 A firewall troubleshooting decision tree

When a connection can't reach a service, work through these in order rather
than guessing:

1. **Is the service actually listening, and on which address?** (`ss -tulpn`
   — is it `0.0.0.0` or `127.0.0.1`-only? §1.1's binding trap.)
2. **Does traffic actually arrive at the interface?** (`tcpdump` — if
   nothing shows up, the problem is upstream, not on this machine at all.)
3. **Is the local firewall blocking it?** (`firewall-cmd --list-all` /
   `ufw status verbose` — check the actual active rules, not what you
   believe you configured.)
4. **Is a rule applied but not persisted?** (§1.4's `--permanent` trap —
   confirm after a `--reload`, not just immediately after adding it.)

Following this order matters because each step rules out an entire category
of cause before moving to the next — checking the firewall first, before
confirming the service is even listening on the right address, wastes time
chasing rules that were never the problem.

## Part 2 — Hands-On Lab

### Commands used in this demo

| Command | Flag meaning | Purpose |
|---|---|---|
| `ss -tulpn` | `-t/-u`: TCP/UDP, `-l`: listening, `-p`: process, `-n`: numeric | Listening sockets and owning processes |
| `ss -tan state established` | `state established`: filter | Active connections only |
| `lsof -i :PORT` | — | What's using a specific port |
| `lsof -p PID` | — | Everything a specific process has open |
| `tcpdump -i IFACE port N -n` | `-i`: interface, `-n`: no DNS resolution | Confirm traffic arrives on the wire |
| `firewall-cmd --zone=Z --add-service=S --permanent` | `--permanent`: survive reload | Allow a named service (Rocky9) |
| `ufw allow PORT/proto` | — | Allow a port (Ubuntu) |

### Task 1 — ss: find listening sockets and the binding trap

```bash
sudo ss -tulpn
# Start a test listener bound to localhost only:
python3 -m http.server 8899 --bind 127.0.0.1 &
sudo ss -tulpn | grep 8899
# Note the 127.0.0.1 binding — this service is unreachable from any other
# host regardless of firewall rules. Confirm by trying to curl it from
# 127.0.0.1 (works) vs. the machine's actual IP (fails, connection refused,
# not a firewall block).
kill %1
```

### Task 2 — lsof: identify what's using a port

```bash
python3 -m http.server 8899 &
sudo lsof -i :8899
# Confirm it identifies the python process and its PID.
kill %1
```

### Task 3 — tcpdump: confirm arrival

```bash
sudo tcpdump -i lo port 8899 -n &
TCPDUMP_PID=$!
python3 -m http.server 8899 --bind 127.0.0.1 &
sleep 1
curl -s 127.0.0.1:8899 > /dev/null
sleep 1
kill $TCPDUMP_PID
kill %1
# Confirm tcpdump captured the SYN/ACK exchange — traffic did arrive at lo.
```

### Task 4 — firewalld or ufw: allow and confirm persistence

```bash
# Rocky9:
sudo firewall-cmd --zone=public --add-port=8899/tcp --permanent
sudo firewall-cmd --reload
sudo firewall-cmd --zone=public --list-ports | grep 8899

# Ubuntu:
sudo ufw allow 8899/tcp
sudo ufw status verbose | grep 8899
```

### Task 5 — NAT/port redirection

```bash
# Rocky9:
sudo firewall-cmd --zone=public --add-forward-port=port=80:proto=tcp:toport=8899 --permanent
sudo firewall-cmd --reload
sudo firewall-cmd --zone=public --list-forward-ports
```

### Task 6 — Work the decision tree on a deliberately broken setup

```bash
python3 -m http.server 8899 --bind 127.0.0.1 &
# A "client" on this same machine tries to reach it via the machine's real IP:
MY_IP=$(hostname -I | awk '{print $1}')
curl -m 3 http://$MY_IP:8899
# Work through §1.7's decision tree step by step to find the actual cause —
# resist jumping straight to checking the firewall.
kill %1
```

## What You Learned

1. ✅ Read `ss` output to identify listening sockets and connection states
2. ✅ Used `lsof` for full network-focused investigation
3. ✅ Captured and read basic `tcpdump` output to confirm traffic arrival
4. ✅ Configured `firewalld`/`ufw` to allow traffic, including the `--permanent` persistence trap
5. ✅ Configured NAT/port redirection with each firewall tool
6. ✅ Applied a systematic decision tree to diagnose connection failures

## Part 3 — Break-Fix Scenario

### Scenario — NovaTech Ops: two connectivity-and-firewall incidents

> **Incident report — NovaTech Ops, ticket #4547**
>
> Alert 1: "app-server-05's API is unreachable from the load balancer,
> worked yesterday." Alert 2: "firewalld rule I added for the new metrics
> port isn't there after last night's patching reboot."

**Incident 1 — API unreachable, worked yesterday**

```bash
$ ss -tulpn | grep 9200
tcp   LISTEN  127.0.0.1:9200   0.0.0.0:*   users:(("api-server",pid=2201,fd=8))
$ sudo firewall-cmd --zone=public --list-ports
9200/tcp
```

Tasks:
1. The firewall rule for port 9200 exists. Is the firewall the problem here?
   Work through §1.7's decision tree — which step actually identifies the
   real cause?
2. Given `ss`'s output, why would this have "worked yesterday" and stopped
   working now — what's a plausible explanation for a binding address
   changing between deploys?
3. What's the fix, and why won't any firewall change address it?

**Incident 2 — firewalld rule missing after reboot**

```bash
$ sudo firewall-cmd --zone=public --list-ports
8080/tcp 443/tcp
$ # (metrics port 9090 is missing — it was added yesterday)
```

Tasks:
1. Given only this output, what's the most likely reason the rule for 9090
   is gone after a reboot? Reference §1.4.
2. How would you confirm your hypothesis without needing to ask whoever
   added the rule how they did it?
3. Re-add the rule correctly this time, and explain what step confirms it
   will actually survive the *next* reboot too, not just this one.

<details>
<summary>Answers</summary>

**Incident 1:** No — the firewall is not the problem, and confirming that
first (it's allowing 9200/tcp) is exactly why step 3 of the decision tree
isn't where the answer is. The real cause is step 1: `ss` shows the service
bound to `127.0.0.1:9200`, not `0.0.0.0:9200` — it's only reachable from
`localhost` on this machine, which is unreachable from an external load
balancer no matter what the firewall allows. A plausible explanation: a
config change or redeploy altered the service's bind address (many
frameworks default to `127.0.0.1` for "local development safety" and require
explicit configuration to bind `0.0.0.0` in production) — something that
would silently break external reachability while leaving every firewall
rule completely unchanged and correct. The fix is changing the
application's bind configuration to `0.0.0.0` (or the specific external
interface), not touching the firewall at all — no firewall change can make
a `127.0.0.1`-bound service reachable from outside.

**Incident 2:** Most likely, the rule was added without `--permanent` (or
`--permanent` was used but `--reload` was never run afterward to confirm it
took effect in the persistent configuration) — a rule added only at runtime
vanishes exactly the way this one did, on the next reload/reboot. Confirm by
checking `sudo firewall-cmd --permanent --zone=public --list-ports` — if
9090 is absent from the *permanent* list even though it was presumably
present in the *runtime* list before the reboot wiped it, that confirms the
hypothesis without needing to ask anyone. Fix: `sudo firewall-cmd
--zone=public --add-port=9090/tcp --permanent && sudo firewall-cmd
--reload`. Confirming it will survive the *next* reboot specifically means
checking `--permanent --list-ports` (not just `--list-ports`, which shows
runtime state) after the `--reload` — the permanent list is what actually
gets reapplied on every future boot.

</details>

## Interview Prep

**Q1. A service's port is confirmed allowed in the firewall, but it's still unreachable from another machine. What's the first thing you'd check, and why not the firewall rules again?**
Whether the service is actually bound to `0.0.0.0` versus `127.0.0.1` with
`ss -tulpn`. A firewall rule allowing a port has no effect if the service
itself is only listening on localhost — that's a completely separate
failure mode from anything the firewall controls, and it's one of the most
common causes of "the firewall rule is right but it still doesn't work"
confusion, precisely because re-checking the firewall rules again won't
reveal it.

**Q2. Why is `tcpdump` diagnostically useful even when you already have `ss` and `lsof` telling you what's listening and what owns it?**
`ss` and `lsof` describe the local machine's socket and process state —
neither tells you whether packets are actually arriving on the wire. If a
connection attempt shows nothing in `tcpdump` on the destination server, the
problem is definitively upstream of this machine (routing, an intermediate
firewall, a cloud security group, or the client never sending) — no amount
of local firewall or application troubleshooting will fix an upstream
problem, and `tcpdump` is what proves that's where the issue actually is.

**Q3. A firewalld rule was added, confirmed working, and then vanished after a reboot. What's the most likely mistake, and how would you prevent it in the future?**
The rule was almost certainly added without `--permanent`, or `--permanent`
was used but `--reload` wasn't run to confirm the permanent configuration
actually matches. Runtime-only changes vanish on reload/reboot by design —
to prevent it, always pair `--permanent` additions with `--reload` and then
verify with `--permanent --list-ports` (not just the default runtime
`--list-ports`) before considering the change actually done.

## Key Takeaways

1. **A service bound to `127.0.0.1` is unreachable from any other host, no
   matter what the firewall allows.** `ss -tulpn`'s address column is the
   first thing to check in any "firewall looks right but it's still
   unreachable" investigation — this is a completely separate failure mode
   from firewall rules.
2. **`tcpdump` answers a question `ss`/`lsof` can't: did the traffic actually
   arrive?** If nothing shows up in a capture on the destination, the
   problem is upstream of this machine entirely — no local firewall or
   application fix will address it.
3. **firewalld's `--permanent` is not optional for a rule to survive a
   reload or reboot.** A rule that "worked, then disappeared" almost always
   means it was added at runtime only — confirm persistence with
   `--permanent --list-ports` after a `--reload`, not just immediately
   after adding the rule.
4. **`lsof -i :PORT` is the direct answer to "what's already using this
   port."** Reach for it before assuming a service failed to start for some
   unrelated reason when the real cause is a `bind: address already in use`.
5. **ufw trades expressiveness for simplicity; firewalld trades simplicity
   for expressiveness.** ufw's NAT workflow requiring direct file edits
   while firewalld's is a single command is a direct consequence of that
   tradeoff — matching the distro default philosophy each represents.
6. **A systematic decision tree beats guessing.** Service binding → traffic
   arrival → firewall rules → persistence, in that order, rules out entire
   categories of cause before moving to the next, rather than jumping
   straight to the firewall (the most commonly — and often wrongly —
   suspected culprit).

> **Demo scope:** Primary concept: diagnosing and filtering network traffic.
> Supporting concepts: ss, lsof network usage, tcpdump, firewalld, ufw, NAT/
> port redirection, a structured troubleshooting decision tree.
> Estimated completion time: 42 minutes (reading + hands-on + verification).
> Checkpoints: 6 natural stopping points (after Task 1 through Task 6).

## Quick Commands Reference

### Socket and process inspection

| Command | Purpose |
|---|---|
| `ss -tulpn` | Listening sockets + owning processes |
| `ss -tan state established` | Active connections only |
| `lsof -i :PORT` | What's using a specific port |
| `lsof -p PID` | Everything a process has open |

### Packet-level confirmation

| Command | Purpose |
|---|---|
| `tcpdump -i IFACE port N -n` | Confirm traffic arrives on the wire |

### firewalld (Rocky9)

| Command | Purpose |
|---|---|
| `firewall-cmd --zone=Z --list-all` | Show active rules for a zone |
| `firewall-cmd --zone=Z --add-service=S --permanent` | Allow a named service |
| `firewall-cmd --zone=Z --add-port=N/tcp --permanent` | Allow a raw port |
| `firewall-cmd --reload` | Apply permanent rules to runtime |
| `firewall-cmd --permanent --list-ports` | Confirm what will survive reboot |
| `firewall-cmd --add-forward-port=port=A:proto=tcp:toport=B --permanent` | NAT/port redirection |

### ufw (Ubuntu)

| Command | Purpose |
|---|---|
| `ufw status verbose` | Show active rules |
| `ufw allow PORT/proto` | Allow a port (applies immediately, no --permanent) |
| `ufw deny PORT/proto` | Deny a port |

## Appendix A — Anki Cards

**Note:** extracted into a standalone `.csv` by the series extraction script.

```
#deck:Linux Mastery::Module 3 - Networking & Storage::12-network-troubleshooting-firewalls
#separator:Comma
#columns:Front,Back,Tags
"A firewall rule allows port 9200, but the service on that port is still unreachable from another machine. What's the most likely cause?","The service is bound to 127.0.0.1 instead of 0.0.0.0 — check with ss -tulpn. A service bound to localhost is unreachable from any other host regardless of what the firewall allows.","demo12,ss,troubleshooting"
"Why is tcpdump diagnostically useful even after ss and lsof have already shown you the listening sockets and owning processes?","ss and lsof describe local socket/process state only — neither confirms whether packets actually arrive on the wire. If tcpdump shows nothing on the destination, the problem is upstream of this machine entirely, and no local firewall or app fix will address it.","demo12,tcpdump,troubleshooting"
"A firewalld rule was added and confirmed working, then vanished after a reboot. What's the most likely mistake?","The rule was added without --permanent (or --permanent was used but --reload was never run to confirm it). Runtime-only rules vanish on reload/reboot by design.","demo12,firewalld,common-mistake"
"How do you confirm a firewalld rule will actually survive the next reboot, not just that it's currently active?","Check firewall-cmd --permanent --list-ports (not the default runtime --list-ports) after running --reload — the permanent list is what gets reapplied on every future boot.","demo12,firewalld,persistence"
"What does lsof -i :PORT answer directly, and when is this the right first command to reach for?","What process (if any) is already using a specific port — reach for it before assuming a service failed to start for some unrelated reason, when the real cause might be a bind: address already in use conflict.","demo12,lsof,port-conflict"
"What's the key functional tradeoff between ufw and firewalld?","ufw trades expressiveness for simplicity (direct allow/deny rules, no zones, no --permanent concept); firewalld trades simplicity for expressiveness (zones, named services, more complex but more capable, e.g. easier NAT configuration).","demo12,ufw,firewalld,comparison"
"In ss -tulpn output, what's the difference between a socket showing 0.0.0.0:PORT versus 127.0.0.1:PORT?","0.0.0.0 means listening on all interfaces, reachable from other hosts (subject to firewall rules). 127.0.0.1 means listening on localhost only, unreachable from any other host no matter what the firewall allows.","demo12,ss,binding"
"What is the correct order of a network troubleshooting decision tree, and why does the order matter?","1) Is the service listening, and on which address? 2) Does traffic actually arrive (tcpdump)? 3) Is the local firewall blocking it? 4) Is a rule applied but not persisted? The order matters because each step rules out an entire category of cause before the next, avoiding wasted time chasing firewall rules that were never the problem.","demo12,troubleshooting,decision-tree"
"Why does Ubuntu's ufw require editing /etc/ufw/before.rules directly for NAT/port redirection, while firewalld handles it in one command?","This reflects ufw's core design tradeoff — it optimizes for simple allow/deny rules and doesn't extend that simplicity to NAT, requiring a drop down to the underlying rules file, while firewalld's zone/rule model was built to handle more complex scenarios like NAT natively.","demo12,ufw,nat,cross-tool"
"ss -n and lsof -n both use -n for the same underlying reason. What is it?","Skip DNS/service-name resolution and show raw numeric addresses/ports — faster, and avoids ambiguity or DNS lookups interfering with the diagnostic output itself.","demo12,ss,lsof,flags"
```

## Appendix B — Quiz

**Note:** extracted into a standalone `.md` by the series extraction script.

> One correct answer per question unless stated otherwise.
> Target: 80% or above before moving to Demo 13.

**Q1. A firewall rule allows port 9200. `curl` from another machine to that port still fails. `ss -tulpn` shows the service on `127.0.0.1:9200`. What's the diagnosis?**

- A) The firewall rule is misconfigured
- B) The service is bound to localhost only, unreachable from other hosts regardless of firewall rules
- C) DNS is misconfigured
- D) The port number is wrong

<details>
<summary>Answer</summary>

**B** — A `127.0.0.1`-bound service is unreachable externally no matter what the firewall allows; this is a completely separate failure mode from firewall configuration. Trap: A wrongly assumes the firewall must be the cause just because a service is unreachable; C and D are unrelated to what the evidence shows.

</details>

---

**Q2. Why would you run `tcpdump` on a destination server during a connectivity investigation, given that `ss` already shows you the listening sockets?**

- A) `tcpdump` is faster than `ss` for the same information
- B) To confirm whether packets are actually arriving at the interface — if not, the problem is upstream of this machine entirely
- C) `tcpdump` shows firewall rule configuration that `ss` doesn't
- D) `tcpdump` is required before `ss` can be used

<details>
<summary>Answer</summary>

**B** — `ss` shows local socket state only; `tcpdump` confirms whether traffic is arriving on the wire at all, which isolates whether the problem is on this machine or somewhere upstream. Trap: A misstates their purpose as competing tools; C and D invent capabilities/dependencies that don't exist.

</details>

---

**Q3. A firewalld rule for port 9090 was added, worked, then disappeared after a reboot. What's the most likely cause?**

- A) firewalld has a bug that periodically clears rules
- B) The rule was added without `--permanent`, or `--reload` was never run to confirm it
- C) Port 9090 is reserved and firewalld silently blocks it after reboot
- D) The reboot reset the network interface, which always clears all firewall rules

<details>
<summary>Answer</summary>

**B** — Runtime-only rules vanish on reload/reboot by design; this is the most common cause of a "working, then gone" firewalld rule. Trap: A, C, and D all invent explanations that misattribute intentional, documented behavior to a bug or unrelated cause.

</details>

---

**Q4. What does `lsof -i :8080` answer directly?**

- A) Whether port 8080 is open in the firewall
- B) What process, if any, currently has port 8080 open
- C) The DNS name associated with port 8080
- D) Whether port 8080 is a "well-known" port

<details>
<summary>Answer</summary>

**B** — `lsof -i :PORT` identifies the process using a specific port, directly answering "what's already using this port" — useful before assuming a service failed to start for an unrelated reason when a port conflict is the real cause. Trap: A, C, and D describe things other tools (firewalld/ufw, DNS tools, `/etc/services`) handle, not `lsof`.

</details>

---

**Q5. What's the core tradeoff between `ufw` and `firewalld`?**

- A) `ufw` is more secure by default
- B) `ufw` trades expressiveness for simplicity; `firewalld` trades simplicity for expressiveness
- C) `firewalld` only works with IPv6
- D) `ufw` requires a commercial license for production use

<details>
<summary>Answer</summary>

**B** — `ufw`'s simple allow/deny model is easier for common cases but less capable for complex scenarios (like NAT, which requires direct file edits); `firewalld`'s zone/service model is more capable but has more concepts to manage. Trap: A, C, and D all invent claims that aren't true of either tool.

</details>

---

**Q6. In `ss -tulpn` output, what does `0.0.0.0:PORT` indicate, as opposed to `127.0.0.1:PORT`?**

- A) `0.0.0.0` means the service is broken; `127.0.0.1` means it's healthy
- B) `0.0.0.0` means listening on all interfaces (reachable externally, subject to firewall); `127.0.0.1` means localhost-only (never reachable externally)
- C) They are functionally identical — just different notations
- D) `0.0.0.0` is IPv6 notation; `127.0.0.1` is IPv4

<details>
<summary>Answer</summary>

**B** — This is the exact distinction that explains many "firewall looks right but still unreachable" cases. Trap: A invents an unrelated health signal; C wrongly claims equivalence; D misidentifies both as the same IP version (both are IPv4; `0.0.0.0` is not an IPv6 notation).

</details>

---

**Q7. Why does `ufw`'s NAT/port-redirection setup require editing `/etc/ufw/before.rules` directly, unlike `firewalld`'s single-command approach?**

- A) `ufw` doesn't actually support NAT at all
- B) This reflects `ufw`'s design tradeoff — optimized for simple rules, not built to handle NAT with the same native simplicity firewalld's zone model provides
- C) `firewalld` doesn't really support NAT either; both require manual file edits
- D) `ufw` only works with IPv6, and NAT is an IPv4-only feature

<details>
<summary>Answer</summary>

**B** — This is the practical consequence of §1.5's simplicity-vs-expressiveness tradeoff between the two tools. Trap: A is factually wrong — `ufw` does support NAT, just less directly; C misstates firewalld's actual (simpler) NAT workflow; D invents an unrelated IP version constraint.

</details>

---

**Q8. What's the correct order for the network troubleshooting decision tree from this demo, and why does the order matter?**

- A) Firewall rules → service binding → traffic arrival → persistence; because firewalls are the most common cause
- B) Service binding → traffic arrival → firewall rules → persistence; because each step rules out a category of cause before the next
- C) Persistence → firewall rules → traffic arrival → service binding; because persistence issues are hardest to diagnose
- D) The order doesn't matter as long as all four are eventually checked

<details>
<summary>Answer</summary>

**B** — Checking binding first catches the "unreachable regardless of firewall" case immediately; traffic arrival next isolates local vs. upstream problems; only then do firewall rules and their persistence become relevant. Trap: A assumes firewalls are the default suspect, which wastes time when the actual cause is upstream in the list; C and D both understate why sequencing avoids wasted investigation time.

</details>