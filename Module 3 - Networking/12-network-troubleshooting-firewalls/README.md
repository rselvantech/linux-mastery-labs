# Demo 12 — Network Troubleshooting & Firewalls

## Overview

Demo 11 got a machine addressed, routed, resolvable, and reachable over SSH.
This demo covers the other half of networking: watching what's actually
happening on the wire, finding out which process owns which socket, and
controlling what traffic is allowed in and out in the first place. By the
end, you'll have a repeatable decision tree for "why can't this connection
get through" instead of guessing.

---

## Demo Objectives

By the end of this demo you will be able to:

1. Read `ss` output to identify listening sockets and connection states
2. Use `lsof` for full network-focused investigation: which process owns which port
3. Capture and read basic `tcpdump` output to confirm traffic is actually arriving
4. Configure `firewalld` (Rocky9) and `ufw` (Ubuntu) to allow and explicitly
   deny/reject traffic
5. Configure NAT/port redirection with each firewall tool
6. Apply a systematic decision tree to diagnose "connection refused/timeout" issues

---

## Directory Structure

```
12-network-troubleshooting-firewalls/
├── README.md                                          # this file
├── 12-network-troubleshooting-firewalls-anki.csv      # Anki flashcard deck
└── 12-network-troubleshooting-firewalls-quiz.md       # standalone quiz
```

---

## Environments

| Task | Ubuntu 24.04 | Rocky Linux 9.8 | EC2 Amazon Linux |
|---|---|---|---|
| Firewall tasks | `ufw` | `firewalld` | Not required (security groups differ) |
| `tcpdump` install | `sudo apt install tcpdump` | `sudo dnf install tcpdump` | Not required |

---

## Recall Check

> Answer from memory before reading further. From Demo 11.

1. A name resolves correctly via `dig` but a user's machine connects to
   a different, stale server. What's the first thing to check, and why does
   `dig` alone miss it?

2. An SSH key login fails and falls back to a password prompt. What two
   things should you check on the server before assuming the key itself is
   wrong?

3. Why does an address added with `ip addr add` disappear after a
   reboot?

<details>
<summary>Answers</summary>

1. `/etc/hosts` on that machine — `nsswitch.conf`'s default `hosts:
   files dns` order checks `/etc/hosts` before DNS, so a stale local entry
   silently overrides a correct DNS answer. `dig` queries DNS directly,
   bypassing that order, so it can't reveal this class of problem.

2. Permissions on `~/.ssh` (must be `700`) and `~/.ssh/authorized_keys`
   (must be `600`) — `sshd` silently distrusts group/world-writable permissions
   on these and falls back to password auth without an explicit error.

3. `ip` modifies the kernel's live network state only — it's not the
   persistence layer. Permanent changes require the distro's own network
   configuration system (Netplan on Ubuntu, NetworkManager on Rocky9).

</details>

---

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

> ⚠️ [VERIFY] Confirmed directly: two identical test servers, one bound to
> `0.0.0.0` and one to `127.0.0.1`, both showed correctly in `ss -tulpn`.
> Querying the `127.0.0.1`-bound one via `127.0.0.1` returned a real `200`;
> querying the identical service via the host's actual interface address
> returned connection refused (`curl` exit code `7`) — no firewall rule
> involved at all, purely the binding address.

```bash
ss -tan state established
# Show established TCP connections only — useful for "who is currently
# talking to this machine" without the noise of every listening socket.
# state accepts other values too: listening, syn-sent, time-wait, closed —
# established is the one you reach for most often when the question is
# "who's actively connected right now," not "what's configured to listen."
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

> ⚠️ [VERIFY] Confirmed directly: `lsof -i :8811 -P -n` against a real
> listening test server returned the exact owning process by name and PID
> (`python3 840 ... TCP *:8811 (LISTEN)`), matching the actual process that
> was started.

```bash
sudo lsof -i -P -n
# -P: show raw port numbers, don't resolve to service names (faster, avoids
# ambiguity — e.g. port 22 displaying as "ssh" is fine until you're
# scripting against the output and need a stable numeric field)
# -n: don't resolve hostnames either (same reasoning as ss -n)
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
# Combine filters with a boolean expression: only traffic to/from a
# specific host, AND on a specific port. `and`/`or`/`not` all work the same
# way here as in any boolean filter expression — e.g.
# `host 10.0.4.12 and (port 22 or port 443)` narrows to just those two
# ports from that one host, useful when a host talks on several ports and
# you only care about a subset.
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

> ⚠️ [VERIFY] Confirmed directly (in this same session's Demo 13b/13d
> work): a `tcpdump` capture to a file and filtered read-back showed a
> real TCP handshake byte-for-byte, confirming captured traffic reflects
> genuine on-wire packets, not an approximation.

### 1.4 firewalld (Rocky9) — zones, services, and explicit denies

```bash
sudo firewall-cmd --get-active-zones
sudo firewall-cmd --zone=public --list-all
```

firewalld organizes rules into **zones** — named rule sets you assign
interfaces to, representing different trust levels. `public` (the default
for most interfaces) assumes limited trust; `internal` and `trusted` assume
progressively more; a genuinely untrusted interface can even be assigned the
`drop` zone, which silently discards everything. Rather than writing raw
port rules, firewalld also supports named **services** (predefined
port/protocol bundles):

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

> ⚠️ [VERIFY] Confirmed directly against a real running `firewalld`: a
> port added without `--permanent` appeared in the runtime list, was
> absent from the permanent list, and — after `--reload` — **vanished from
> the runtime list entirely**, reproducing "the rule worked, then
> mysteriously disappeared" exactly. The identical sequence with
> `--permanent` survived `--reload` and appeared in both the runtime and
> permanent listings.

```bash
sudo firewall-cmd --zone=public --list-all
# Confirm both runtime and permanent state — a rule showing in --list-all
# after --reload confirms it actually persisted, not just applied once.
```

**Explicit denies — rich rules, and reject vs. drop:**

A zone is already deny-by-default for anything not explicitly allowed, so
most day-to-day work is adding allow rules. But sometimes you want a block
to be *visible and intentional* rather than merely "absent" — for example,
documenting that a legacy port is deliberately closed, not just forgotten.
firewalld's mechanism for this is a **rich rule**:

```bash
sudo firewall-cmd --zone=public --add-rich-rule='rule family="ipv4" port port="2323" protocol="tcp" reject' --permanent
sudo firewall-cmd --reload
sudo firewall-cmd --zone=public --list-rich-rules
```

`reject` sends the connecting client an immediate rejection response (a TCP
RST) — the client fails fast and knows the port is closed. `drop` instead
silently discards the packet with no response at all, leaving the client to
time out. Use `reject` for a port you want to fail fast and cleanly for
legitimate clients; use `drop` when you'd rather not confirm to a port
scanner that anything is even listening there — the port simply looks dead,
indistinguishable from "nothing at this address at all."

> ⚠️ [VERIFY — partially confirmed]: the rich-rule syntax itself was
> confirmed to apply and appear correctly in `firewall-cmd
> --list-rich-rules` during this session's testing. The client-facing
> RST-vs-silent-timeout distinction between `reject` and `drop` is
> standard, documented `nftables`/`netfilter` behavior (and the same
> mechanism Demo 12a confirmed directly for plain `nftables` rules) rather
> than something separately re-tested against `firewalld` specifically in
> this pass.

### 1.5 ufw (Ubuntu) — the simpler front end

```bash
sudo ufw status verbose
sudo ufw allow 443/tcp
sudo ufw allow from 10.0.4.0/24 to any port 5432
sudo ufw deny 23/tcp
```

`ufw` ("uncomplicated firewall") is a simpler front end over the same
underlying `netfilter` machinery firewalld also manages — no zones, rules
apply directly and immediately, no separate `--permanent` concept to forget.
This makes it faster for simple cases and less expressive for complex
multi-zone setups, which is exactly the tradeoff that explains why Ubuntu
defaults to it and Rocky9 defaults to the more elaborate firewalld. `ufw
deny` is ufw's equivalent of firewalld's rich-rule `reject`/`drop` — a
visible, explicit block rather than simply never allowing the port.

> ⚠️ [VERIFY] Confirmed directly: `ufw allow 8899/tcp` and `ufw deny
> 2323/tcp` both applied immediately with no reload step, and `nft list
> ruleset` showed the real underlying rules `ufw` created —
> `tcp dport 8899 ... accept` and `tcp dport 2323 ... drop` — confirming
> `ufw` is genuinely a front end over real `nftables` rules, not a
> separate bookkeeping layer.

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
# firewalld — one command:
sudo firewall-cmd --zone=public --add-forward-port=port=80:proto=tcp:toport=8080 --permanent
sudo firewall-cmd --reload
```

```bash
# ufw — requires editing /etc/ufw/before.rules directly; ufw's simplicity
# doesn't extend to NAT the way firewalld's does:
sudo cp /etc/ufw/before.rules /etc/ufw/before.rules.bak   # always back up first

# Add near the TOP of the file, before the *filter section:
#   *nat
#   :PREROUTING ACCEPT [0:0]
#   -A PREROUTING -p tcp --dport 80 -j REDIRECT --to-port 8080
#   COMMIT
#
# *nat / PREROUTING is the table+chain that intercepts incoming packets
# before routing decisions are made — the right place for "redirect this
# port to that port" rules. REDIRECT (rather than DNAT) is specifically for
# redirecting to a port on the SAME host; DNAT would be used to redirect to
# a different host entirely.
sudo ufw disable && sudo ufw enable   # reload rules including before.rules
sudo iptables -t nat -L PREROUTING -n # confirm the REDIRECT rule is active
```

> ⚠️ [VERIFY] Confirmed directly, and with a nuance worth flagging: the
> first attempt to verify this rule via `curl` from `127.0.0.1` on the
> same host returned nothing — locally-generated loopback traffic to the
> host's own address does **not** traverse `PREROUTING`, so a REDIRECT
> rule can look like it's doing nothing when tested this way. Testing
> again with a genuinely external client (a separate network namespace)
> confirmed it correctly: the client's request to port 80 was answered by
> the port-8080 server, and `ss -tlnp` confirmed nothing was actually
> listening on port 80 itself. If you ever test a `REDIRECT` rule and it
> "doesn't work" from `curl localhost`, that's expected — test from
> another machine (or namespace) instead.

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

---

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
| `firewall-cmd --zone=Z --add-rich-rule='... reject\|drop' --permanent` | — | Explicit deny/reject a port (Rocky9) |
| `ufw allow PORT/proto` | — | Allow a port (Ubuntu) |
| `ufw deny PORT/proto` | — | Explicitly deny a port (Ubuntu) |

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

### Task 4 — firewalld or ufw: allow, deny, and confirm persistence

```bash
# Rocky9:
sudo firewall-cmd --zone=public --add-port=8899/tcp --permanent
sudo firewall-cmd --reload
sudo firewall-cmd --zone=public --list-ports | grep 8899

# Now add an explicit, visible deny for a different port — a rich rule,
# not just the absence of an allow rule:
sudo firewall-cmd --zone=public --add-rich-rule='rule family="ipv4" port port="2323" protocol="tcp" reject' --permanent
sudo firewall-cmd --reload
sudo firewall-cmd --zone=public --list-rich-rules

# Ubuntu:
sudo ufw allow 8899/tcp
sudo ufw status verbose | grep 8899

# Explicit deny:
sudo ufw deny 2323/tcp
sudo ufw status verbose | grep 2323
```

### Task 5 — NAT/port redirection, both tools

```bash
# Rocky9:
sudo firewall-cmd --zone=public --add-forward-port=port=80:proto=tcp:toport=8899 --permanent
sudo firewall-cmd --reload
sudo firewall-cmd --zone=public --list-forward-ports

# Ubuntu — edit before.rules per §1.6, then reload:
sudo cp /etc/ufw/before.rules /etc/ufw/before.rules.bak
sudo sed -i '1i *nat\n:PREROUTING ACCEPT [0:0]\n-A PREROUTING -p tcp --dport 80 -j REDIRECT --to-port 8899\nCOMMIT\n' /etc/ufw/before.rules
sudo ufw disable && sudo ufw enable
sudo iptables -t nat -L PREROUTING -n
# Confirm the REDIRECT rule targeting port 8899 appears in the listing.
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

---

## What You Learned

- ✅ Read `ss` output to identify listening sockets and connection states
- ✅ Used `lsof` for full network-focused investigation
- ✅ Captured and read basic `tcpdump` output to confirm traffic arrival
- ✅ Configured `firewalld`/`ufw` to allow and explicitly deny/reject
  traffic, including the `--permanent` persistence trap and firewalld's
  `reject` vs. `drop` distinction
- ✅ Configured NAT/port redirection with each firewall tool
- ✅ Applied a systematic decision tree to diagnose connection failures

**Key Takeaway:** A firewall rule being correct doesn't mean the connection will work — the binding address (`0.0.0.0` vs `127.0.0.1`), whether traffic even arrives, and whether a rule actually persisted are three completely separate failure modes that all look like "it's not working" from the outside. The decision tree in §1.7 exists specifically to avoid jumping straight to the most commonly (and often wrongly) suspected culprit.

---

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

---

## Interview Prep

**Q1. A service's port is confirmed allowed in the firewall, but it's still unreachable from another machine. What's the first thing you'd check, and why not the firewall rules again?**

Whether the service is actually bound to `0.0.0.0` versus `127.0.0.1` with
`ss -tulpn`. A firewall rule allowing a port has no effect if the service
itself is only listening on localhost — that's a completely separate
failure mode from anything the firewall controls, and it's one of the most
common causes of "the firewall rule is right but it still doesn't work"
confusion, precisely because re-checking the firewall rules again won't
reveal it.

---

**Q2. Why is `tcpdump` diagnostically useful even when you already have `ss` and `lsof` telling you what's listening and what owns it?**

`ss` and `lsof` describe the local machine's socket and process state —
neither tells you whether packets are actually arriving on the wire. If a
connection attempt shows nothing in `tcpdump` on the destination server, the
problem is definitively upstream of this machine (routing, an intermediate
firewall, a cloud security group, or the client never sending) — no amount
of local firewall or application troubleshooting will fix an upstream
problem, and `tcpdump` is what proves that's where the issue actually is.

---

**Q3. A firewalld rule was added, confirmed working, and then vanished after a reboot. What's the most likely mistake, and how would you prevent it in the future?**

The rule was almost certainly added without `--permanent`, or `--permanent`
was used but `--reload` wasn't run to confirm the permanent configuration
actually matches. Runtime-only changes vanish on reload/reboot by design —
to prevent it, always pair `--permanent` additions with `--reload` and then
verify with `--permanent --list-ports` (not just the default runtime
`--list-ports`) before considering the change actually done.

---

**Q4. When would you use a firewalld rich rule with `reject` versus one with `drop`, given that a zone is already deny-by-default for anything not explicitly allowed?**

Both explicit denies are for making a block *visible and intentional* rather
than incidental — but they differ in what the connecting client experiences.
`reject` sends back an immediate rejection (a TCP RST), so a legitimate
client fails fast and clearly. `drop` silently discards the packet with no
response, leaving the client to time out — this is preferable when you'd
rather not confirm to a port scanner that anything is listening there at
all, since a dropped port is indistinguishable from nothing being at that
address.

---

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
7. **An explicit deny is different from an absent allow, and `reject` is
   different from `drop`.** A zone already blocks anything not explicitly
   allowed, so an explicit rich rule (firewalld) or `ufw deny` is for making
   a block visible/intentional rather than incidental. `reject` fails fast
   with a response for legitimate clients; `drop` silently discards,
   reducing what a port scanner can learn.

---

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
| `tcpdump -i IFACE host H and port N -n` | Combine host + port filters |

### firewalld (Rocky9)

| Command | Purpose |
|---|---|
| `firewall-cmd --zone=Z --list-all` | Show active rules for a zone |
| `firewall-cmd --zone=Z --add-service=S --permanent` | Allow a named service |
| `firewall-cmd --zone=Z --add-port=N/tcp --permanent` | Allow a raw port |
| `firewall-cmd --zone=Z --add-rich-rule='... reject' --permanent` | Explicit reject for a port |
| `firewall-cmd --zone=Z --add-rich-rule='... drop' --permanent` | Explicit silent drop for a port |
| `firewall-cmd --reload` | Apply permanent rules to runtime |
| `firewall-cmd --permanent --list-ports` | Confirm what will survive reboot |
| `firewall-cmd --add-forward-port=port=A:proto=tcp:toport=B --permanent` | NAT/port redirection |

### ufw (Ubuntu)

| Command | Purpose |
|---|---|
| `ufw status verbose` | Show active rules |
| `ufw allow PORT/proto` | Allow a port (applies immediately, no --permanent) |
| `ufw deny PORT/proto` | Explicitly deny a port |
| Edit `/etc/ufw/before.rules` (`*nat`/`PREROUTING`/`REDIRECT`) + `ufw disable && ufw enable` | NAT/port redirection |

---

## Appendix A — Anki Cards

**`12-network-troubleshooting-firewalls-anki.csv`:**

```
#deck:Linux Mastery::Module 3 - Networking::Demo 12 - Network Troubleshooting & Firewalls
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

---

## Appendix B — Quiz

**`12-network-troubleshooting-firewalls-quiz.md`:**

````markdown
# Quiz — Demo 12: Network Troubleshooting & Firewalls

> One correct answer per question unless stated otherwise.
> Target: 80% or above before moving to Demo 13.

---

**Q1.** You run `ss -tan state established` instead of plain `ss -tan`. What does adding `state established` change about the output, and when is this the right filter to reach for?

- A) It shows only UDP sockets, filtering out TCP entirely
- B) It shows only currently active, connected TCP sessions — useful for "who is talking to this machine right now," as opposed to what's merely configured to listen
- C) It shows only sockets that failed to establish a connection
- D) It has no effect; `state established` is ignored by `ss`

<details>
<summary>Answer</summary>

**B** — `state established` filters to TCP connections that are actively connected, not just listening or in other transitional states (`syn-sent`, `time-wait`, etc.). Reach for it when the question is "who is currently connected" rather than "what's configured to accept connections."

Trap: A misreads `-t` vs `-u` (protocol flags) as something `state` controls; C and D both invent behavior `state established` doesn't have.

</details>

---

**Q2.** You run `sudo tcpdump -i eth0 host 10.0.4.12 and port 22 -n`. What does this capture?

- A) All traffic on eth0, with a label added for host 10.0.4.12
- B) Only traffic on eth0 that is both to/from 10.0.4.12 AND on port 22 — both conditions must match
- C) Traffic on eth0 that is either from 10.0.4.12 OR on port 22 — either condition matches
- D) This is invalid syntax; `tcpdump` only accepts one filter at a time

<details>
<summary>Answer</summary>

**B** — `and` combines filter expressions so that both conditions must hold for a packet to be captured — here, the specific host AND that specific port. `or` would instead capture packets matching either condition. `tcpdump` fully supports combining filters this way; it isn't limited to one at a time.

Trap: C describes what `or` would do, not `and`; D wrongly claims a limitation `tcpdump` doesn't have.

</details>

---

**Q3.** What does a firewalld "zone" represent, conceptually?

- A) A physical network segment defined by cabling
- B) A named rule set representing a trust level, assigned to interfaces — e.g. `public` (limited trust), `internal`/`trusted` (progressively more trust), or `drop` (silently discard everything)
- C) A backup copy of the firewall configuration
- D) A scheduled time window during which rules are active

<details>
<summary>Answer</summary>

**B** — Zones let you apply different rule sets to different interfaces based on how much you trust the network they're connected to, without needing a separate firewall tool per interface. `public` is the typical default for most interfaces.

Trap: A, C, and D all invent meanings unrelated to what a zone actually represents.

</details>

---

**Q4.** You install `ufw` on a fresh Ubuntu server and immediately add several `allow` rules. Traffic is still not being filtered at all. What's the most likely explanation?

- A) `ufw` rules require a reboot before taking effect
- B) `ufw` is often installed but inactive by default — `ufw status` would show it as inactive, and `ufw enable` is required before any rules are enforced
- C) `ufw allow` rules only take effect for IPv6 traffic
- D) `ufw` cannot filter traffic until `firewalld` is also installed

<details>
<summary>Answer</summary>

**B** — Adding rules with `ufw allow`/`ufw deny` configures them, but `ufw` itself must be enabled (`sudo ufw enable`) before any enforcement happens at all. Checking `ufw status` is the first thing to confirm, not assuming rules are automatically active just because they were added.

Trap: A invents a reboot requirement that doesn't exist; C and D invent constraints and dependencies that aren't real.

</details>

---

**Q5.** In a firewalld rich rule, what is the functional difference between a `reject` target and a `drop` target?

- A) `reject` is for TCP only; `drop` is for UDP only
- B) `reject` sends the client an immediate rejection response (e.g. a TCP RST), so it fails fast; `drop` silently discards the packet with no response, leaving the client to time out
- C) They are functionally identical; `reject` is just the newer name for `drop`
- D) `drop` only works in the `public` zone; `reject` works in all zones

<details>
<summary>Answer</summary>

**B** — `reject` gives a legitimate client a fast, clear failure. `drop` gives no response at all, which is preferable when you don't want a port scanner to learn that anything is listening there — a dropped port looks indistinguishable from nothing being at that address.

Trap: A invents a protocol restriction that doesn't exist for either target; C wrongly claims they're the same; D invents a zone restriction that doesn't exist.

</details>

---

**Q6.** You need Ubuntu to redirect incoming traffic on port 80 to port 8080 on the same host. Which line, added to `/etc/ufw/before.rules` under a `*nat`/`PREROUTING` section, achieves this?

- A) `-A PREROUTING -p tcp --dport 80 -j DNAT --to-destination 8080`
- B) `-A PREROUTING -p tcp --dport 80 -j REDIRECT --to-port 8080`
- C) `-A POSTROUTING -p tcp --dport 80 -j MASQUERADE`
- D) `ufw allow 80/tcp redirect 8080`

<details>
<summary>Answer</summary>

**B** — `REDIRECT` is the correct target for redirecting to a different port on the *same* host, applied in the `PREROUTING` chain of the `nat` table (where incoming packets are intercepted before routing decisions). `DNAT` (A) is for redirecting to a different host, not just a different port on the same one. `MASQUERADE` (C) is an outbound source-NAT mechanism, unrelated to this scenario. `ufw` has no built-in `redirect` keyword (D) — this is exactly why the demo requires editing `before.rules` directly.

</details>

---

**Q7.** `sudo lsof -i -P -n` is run instead of plain `sudo lsof -i`. What do the `-P` and `-n` flags each specifically change?

- A) `-P` shows only TCP; `-n` shows only UDP
- B) `-P` shows raw port numbers instead of resolved service names (e.g. `22` instead of `ssh`); `-n` skips hostname resolution — both make output faster and avoid ambiguity
- C) `-P` requires root privileges; `-n` does not
- D) `-P` and `-n` are aliases for the same behavior; only one is needed

<details>
<summary>Answer</summary>

**B** — `-P` prevents `lsof` from translating port numbers into service names from `/etc/services` (avoiding ambiguity, especially useful when scripting against the output); `-n` prevents hostname resolution for addresses. Both exist for the same underlying reason `ss -n` does: speed and unambiguous, stable numeric output.

Trap: A invents a protocol-filtering role neither flag has; C and D both invent behavior that isn't accurate.

</details>

---

**Q8.** A zone in firewalld already blocks any port that hasn't been explicitly allowed. Given that, why would you ever add an explicit `reject` or `drop` rich rule for a port, instead of simply never allowing it?

- A) There's no reason — an explicitly denied port and a never-allowed port behave identically in every respect
- B) To make the block visible and intentional in the configuration (e.g. documenting that a legacy port is deliberately closed) and to control exactly how the client experiences the block — fast rejection vs. silent timeout — rather than relying on the default deny-by-omission behavior
- C) Explicit rich rules are required before firewalld will start at all
- D) Only rich rules are saved permanently; plain zone rules are always temporary

<details>
<summary>Answer</summary>

**B** — The functional traffic-blocking outcome of "never allowed" and "explicitly rejected/dropped" can be similar, but an explicit rich rule documents intent for anyone reading the configuration later, and it lets you choose the client-facing behavior (`reject`'s fast RST vs. `drop`'s silent timeout) rather than accepting whatever the zone's default happens to produce.

Trap: A dismisses a real, practical reason (visibility/intent and controlling client experience); C and D both invent requirements/behavior that aren't true of firewalld.

</details>

---

| Score | Action |
|---|---|
| 7/8 or 8/8 | Import Anki cards, move to Demo 13 |
| 6/8 | Review the wrong answers, then proceed |
| 5/8 | Re-read the relevant sections, retry those questions |
| Below 5/8 | Re-read the full demo and redo the walkthrough before proceeding |
````