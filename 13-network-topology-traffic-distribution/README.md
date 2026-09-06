# Demo 13 — Network Topology & Traffic Distribution

## Overview

Demo 12 covered diagnosing and filtering traffic on a single machine. This
demo zooms out: how traffic gets routed across multiple networks, how two
physical interfaces can act as one for redundancy or bandwidth, and how one
public-facing address can distribute load across many backend servers. These
are the building blocks of any topology beyond "one server, one network."

## Demo Objectives

1. Configure static routes for multi-network topologies, beyond the default gateway
2. Explain policy routing at a conceptual level — multiple routing tables, rule-based selection
3. Configure a bridge to connect network segments at layer 2
4. Configure bonding for redundancy or throughput, and distinguish bonding modes
5. Configure an nginx reverse proxy with an upstream block
6. Explain the common load-balancing algorithms and when each fits

## Directory Structure

```
13-network-topology-traffic-distribution/
└── README.md   # This file — full demo content, including Anki (Appendix A)
                 # and Quiz (Appendix B), extracted later via script.
```

## Environments

| Task | Ubuntu 24.04 | Rocky Linux 9.8 | EC2 Amazon Linux |
|---|---|---|---|
| All lab tasks | ✅ | ✅ | Not required |
| nginx install | `sudo apt install nginx` | `sudo dnf install nginx` | Not required |

## Recall Check — 12-network-troubleshooting-firewalls

Answer these from memory before continuing — no looking back at Demo 12.

**Q1.** A firewall rule allows a port, but the service is still unreachable
from another machine. `ss -tulpn` shows it on `127.0.0.1:PORT`. What's the
diagnosis?

**Q2.** Why is `tcpdump` diagnostically useful even after `ss` and `lsof`
already show listening sockets and owning processes?

**Q3.** A firewalld rule worked, then vanished after a reboot. What's the
most likely mistake, and how would you confirm it?

<details>
<summary>Answers</summary>

**A1.** The service is bound to localhost only — unreachable from any other
host regardless of what the firewall allows. This is a completely separate
failure mode from firewall configuration.

**A2.** `ss`/`lsof` only describe local socket/process state; `tcpdump`
confirms whether packets actually arrive on the wire. If nothing shows up,
the problem is upstream of this machine entirely.

**A3.** The rule was almost certainly added without `--permanent`, or
`--reload` was never run to confirm it. Confirm with `firewall-cmd
--permanent --list-ports` — the permanent list is what survives reboot.

</details>

## Part 1 — Concepts

### 1.1 Static routing beyond the default gateway

Demo 11 covered the default gateway — where traffic goes when nothing more
specific matches. Real topologies often need routes to *specific* networks
that shouldn't go through the default gateway at all:

```bash
sudo ip route add 10.20.0.0/16 via 192.168.1.254 dev eth0
```

This says: traffic to `10.20.0.0/16` goes via `192.168.1.254`, not the
default gateway — common when a second network (a separate VPC, a VPN-
connected office network, a management network) is reachable through a
different next-hop than general internet traffic.

**Route selection follows longest-prefix match, not first-match order.** If
both a `/16` route and a more specific `/24` route exist for overlapping
address ranges, the kernel picks whichever route has the longer (more
specific) prefix, regardless of which was added first:

```bash
ip route show
# 10.20.0.0/16 via 192.168.1.254 dev eth0
# 10.20.5.0/24 via 192.168.1.253 dev eth0
# Traffic to 10.20.5.10 uses the /24 route (more specific), even though
# the /16 route also technically matches.
```

### 1.2 Policy routing — when one routing table isn't enough

Everything so far has used the single default routing table. **Policy
routing** allows multiple routing tables, with rules selecting which table
applies based on criteria beyond just destination address — source address,
incoming interface, or firewall marks:

```bash
ip rule show
# 0: from all lookup local
# 32766: from all lookup main
# 32767: from all lookup default
```

A common real-world use: a server with two network interfaces on two ISPs
needs traffic that *arrived* on ISP A's interface to have its *replies* go
back out through ISP A, not whatever the default table would otherwise
choose — asymmetric routing like that breaks stateful firewalls and
connection tracking on the way. Policy routing solves this with a rule like
"traffic sourced from ISP A's address uses ISP A's routing table."

This demo doesn't build a full policy-routing setup — it's a large enough
topic to warrant its own treatment — but recognizing `ip rule show` and
knowing *why* multiple tables exist (multi-homed servers, VPN split-tunnel
setups, asymmetric routing avoidance) is the practical takeaway here.

### 1.3 Bridging — connecting segments at layer 2

A bridge joins multiple network interfaces into a single layer-2 broadcast
domain — traffic on one side is forwarded to the other as though they were
the same physical wire. This is the mechanism behind container networking
(Docker's default bridge network) and connecting a VM's virtual interface to
a physical network.

```bash
sudo ip link add name br0 type bridge
sudo ip link set eth1 master br0
sudo ip link set br0 up
sudo ip link set eth1 up
ip link show master br0
# Confirm eth1 is now a member of br0
```

Once bridged, `eth1` no longer has its own independent IP identity for
routing purposes — traffic addressed to devices on `eth1`'s segment is
switched, not routed, through the bridge.

### 1.4 Bonding — combining interfaces for redundancy or throughput

Bonding combines multiple physical interfaces into one logical interface —
but *which* problem it solves depends entirely on the bonding mode, and
conflating them is a common mistake:

| Mode | Name | What it actually provides |
|---|---|---|
| `mode=1` | `active-backup` | **Redundancy only.** One interface active, the other standby, failover on link loss. No throughput increase — only one link carries traffic at a time. |
| `mode=4` | `802.3ad` (LACP) | **Throughput *and* redundancy**, but requires the connected switch to support and be configured for LACP — this is not something you can configure unilaterally on the server side alone. |
| `mode=0` | `balance-rr` | Round-robin across links — throughput improvement for a single connection is possible but often not switch-compatible without matching configuration, and can cause out-of-order packet delivery. |

```bash
sudo nmcli connection add type bond con-name bond0 ifname bond0 mode active-backup
sudo nmcli connection add type ethernet ifname eth1 master bond0
sudo nmcli connection add type ethernet ifname eth2 master bond0
sudo nmcli connection up bond0
```

**The most common bonding mistake is assuming any bonding mode
automatically doubles throughput.** Only LACP (`mode=4`), correctly matched
with switch-side configuration, does that — `active-backup` trades any
throughput gain entirely for simplicity and guaranteed compatibility with
unmanaged or non-LACP switches.

### 1.5 Reverse proxies — nginx upstream configuration

A reverse proxy sits in front of one or more backend servers, forwarding
client requests to them and returning the response as though the proxy
itself served it. An `upstream` block defines the pool of backends:

```nginx
upstream novatech_backend {
    server 10.0.4.10:8080;
    server 10.0.4.11:8080;
    server 10.0.4.12:8080;
}

server {
    listen 80;
    server_name app.novatech.internal;

    location / {
        proxy_pass http://novatech_backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

`proxy_set_header X-Real-IP $remote_addr` matters beyond boilerplate: without
it, every backend server sees every request as coming from the proxy's own
IP, not the actual client — breaking IP-based rate limiting, logging, and
geolocation on the backend unless it's explicitly told to trust and read
this header.

### 1.6 Load-balancing algorithms

The `upstream` block above defaults to round-robin. Other algorithms change
behavior meaningfully:

```nginx
upstream novatech_backend {
    least_conn;                    # route to the backend with fewest active connections
    server 10.0.4.10:8080;
    server 10.0.4.11:8080;
}
```

```nginx
upstream novatech_backend {
    ip_hash;                       # same client IP always routes to the same backend
    server 10.0.4.10:8080;
    server 10.0.4.11:8080;
}
```

| Algorithm | Behavior | Fits when |
|---|---|---|
| Round-robin (default) | Requests distributed evenly in rotation | Backends are stateless and roughly equal capacity |
| `least_conn` | Routes to whichever backend has fewest active connections | Requests have significantly variable processing time |
| `ip_hash` | Same client always hits the same backend | The application needs session affinity ("sticky sessions") without a shared session store |

Choosing `ip_hash` for session affinity is a real tradeoff, not a free
convenience: it means load distribution is only as even as client IP
diversity allows, and a backend going down disproportionately affects
whichever clients were hashed to it, unlike round-robin's even spread of
that same failure.

## Part 2 — Hands-On Lab

### Commands used in this demo

| Command | Flag meaning | Purpose |
|---|---|---|
| `ip route add NET via GW dev IFACE` | — | Add a static route to a specific network |
| `ip rule show` | — | List policy routing rules |
| `ip link add name BR type bridge` | — | Create a bridge interface |
| `ip link set IFACE master BR` | — | Add an interface to a bridge |
| `nmcli connection add type bond ... mode MODE` | — | Create a bonded interface with a specific mode |
| `nginx -t` | `-t`: test config | Validate nginx configuration syntax before reloading |
| `systemctl reload nginx` | — | Apply nginx config changes without dropping connections |

### Task 1 — Static routing with longest-prefix match

```bash
sudo ip route add 10.20.0.0/16 via 192.168.1.254 dev eth0
sudo ip route add 10.20.5.0/24 via 192.168.1.253 dev eth0
ip route get 10.20.5.10
# Confirm the output shows the /24 route's gateway (192.168.1.253), not the
# /16 route's — demonstrating longest-prefix match in practice.
sudo ip route del 10.20.0.0/16
sudo ip route del 10.20.5.0/24
```

### Task 2 — Inspect policy routing tables (read-only)

```bash
ip rule show
ip route show table main
ip route show table local
# No changes here — just confirm you can identify which table a route
# belongs to and recognize this multi-table structure exists.
```

### Task 3 — Create a bridge

```bash
sudo ip link add name br-demo type bridge
sudo ip link set br-demo up
ip link show br-demo
sudo ip link del br-demo
```

### Task 4 — Bonding mode comparison (read-only inspection)

```bash
cat /sys/class/net/bonding_masters 2>/dev/null || echo "no bonds currently configured"
# If a bond exists in your environment:
cat /proc/net/bonding/bond0 2>/dev/null | head -20
# Identify: current bonding mode, active/backup interface state (if
# active-backup), and MII status of each slave interface.
```

### Task 5 — nginx reverse proxy with an upstream block

```bash
sudo tee /etc/nginx/conf.d/novatech-demo.conf > /dev/null << 'EOF'
upstream novatech_backend {
    server 127.0.0.1:8081;
    server 127.0.0.1:8082;
}

server {
    listen 8090;
    location / {
        proxy_pass http://novatech_backend;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
EOF

sudo nginx -t
# -t validates syntax without applying — catch config errors before reload,
# not after.
sudo systemctl reload nginx

python3 -m http.server 8081 &
python3 -m http.server 8082 &
for i in 1 2 3 4; do curl -s -o /dev/null -w "%{http_code}\n" http://localhost:8090; done
kill %1 %2
```

### Task 6 — Compare round-robin vs. ip_hash behavior

```bash
sudo sed -i '/upstream novatech_backend/a\    ip_hash;' /etc/nginx/conf.d/novatech-demo.conf
sudo nginx -t && sudo systemctl reload nginx
python3 -m http.server 8081 &
python3 -m http.server 8082 &
for i in 1 2 3 4; do curl -s http://localhost:8090 > /dev/null; done
# With ip_hash and requests all from the same client IP (localhost here),
# every request should land on the same backend — contrast this mentally
# with Task 5's round-robin default, which would have alternated.
kill %1 %2
sudo rm /etc/nginx/conf.d/novatech-demo.conf
sudo systemctl reload nginx
```

## What You Learned

1. ✅ Configured static routes for multi-network topologies beyond the default gateway
2. ✅ Explained policy routing conceptually — multiple tables, rule-based selection
3. ✅ Configured a bridge to connect network segments at layer 2
4. ✅ Configured bonding and distinguished bonding modes by what they actually provide
5. ✅ Configured an nginx reverse proxy with an upstream block
6. ✅ Explained load-balancing algorithms and when each fits

## Part 3 — Break-Fix Scenario

### Scenario — NovaTech Ops: two topology incidents

> **Incident report — NovaTech Ops, ticket #4560**
>
> Alert 1: "bonded interface bond0 shows no throughput improvement after
> switching from a single NIC — we expected double bandwidth." Alert 2: "load
> balancer sends all traffic to one backend server, the other two are idle."

**Incident 1 — bonding shows no throughput improvement**

```bash
$ cat /proc/net/bonding/bond0 | head -5
Ethernet Channel Bonding Driver: v6.5.0

Bonding Mode: fault-tolerance (active-backup)
Primary Slave: None
Currently Active Slave: eth1
```

Tasks:
1. Given the bonding mode shown, was "double bandwidth" ever a realistic
   expectation for this configuration? Reference §1.4's mode comparison
   table.
2. What would need to change — both on this server and elsewhere — to
   actually get a throughput improvement from bonding?
3. Is this bond misconfigured, or working exactly as designed for a
   different goal than the one expected? Justify your answer.

**Incident 2 — load balancer sends all traffic to one backend**

```bash
$ cat /etc/nginx/conf.d/lb.conf
upstream app_pool {
    ip_hash;
    server 10.0.4.10:8080;
    server 10.0.4.11:8080;
    server 10.0.4.12:8080;
}
$ # All traffic arrives via a single corporate NAT gateway, so every
$ # client request nginx sees originates from the same source IP: 10.0.9.1
```

Tasks:
1. Given `ip_hash` and the fact that every request appears to originate
   from `10.0.9.1`, explain exactly why all traffic lands on one backend —
   is nginx malfunctioning?
2. What's the actual tradeoff this incident reveals about `ip_hash`
   specifically, referenced in §1.6?
3. What load-balancing algorithm change would fix the *load distribution*
   problem, and what would it cost you in exchange (what was `ip_hash`
   providing that the fix gives up)?

<details>
<summary>Answers</summary>

**Incident 1:** No — `active-backup` mode was never going to provide
throughput improvement; per §1.4, it provides redundancy only, with a single
active link carrying traffic at any given time and the second link sitting
idle as standby. Getting an actual throughput improvement requires switching
to `mode=4` (802.3ad/LACP), and critically, that also requires the connected
switch to support and be explicitly configured for LACP — this can't be
achieved by server-side configuration alone. This bond is not misconfigured;
it's working exactly as `active-backup` is designed to work. The actual
problem is a mismatch between the chosen mode and the expected outcome, not
a fault in the bond itself.

**Incident 2:** nginx isn't malfunctioning — `ip_hash` is working exactly as
designed, routing based on client source IP. The problem is that every
request appears to come from the same source IP (`10.0.9.1`, the corporate
NAT gateway), so `ip_hash` has effectively only one input to hash against,
and consistently sends everything to the same backend as a direct
consequence. This is precisely the tradeoff §1.6 flags: `ip_hash`'s load
distribution is only as even as client IP diversity allows — behind NAT,
that diversity can collapse to nearly nothing. Switching to `least_conn` (or
plain round-robin) would fix the distribution problem, at the cost of losing
session affinity — if the application relies on a client consistently
hitting the same backend (e.g. for in-memory session state), that guarantee
goes away and would need to be replaced with a shared session store or
sticky-session mechanism that doesn't depend on source IP.

</details>

## Interview Prep

**Q1. A team assumes bonding two NICs automatically doubles their server's network throughput. When is this true, and when is it a costly misunderstanding?**
It's only true for `mode=4` (802.3ad/LACP), and even then only if the
connected switch also supports and is configured for LACP — it's not
something the server alone can guarantee. For `active-backup` (`mode=1`),
throughput never improves at all; it trades any throughput gain entirely for
redundancy and guaranteed compatibility with switches that don't support
LACP. Assuming any bonding configuration doubles throughput is a common and
costly misunderstanding when the actual mode chosen was `active-backup`.

**Q2. Why would you choose `ip_hash` over round-robin for a load-balanced application, and what does that choice cost you?**
`ip_hash` provides session affinity — the same client IP consistently
reaches the same backend, useful when the application keeps session state
in memory on a specific server rather than a shared store. The cost is that
load distribution becomes only as even as client IP diversity allows — a
large population of clients behind a shared NAT gateway can collapse
`ip_hash`'s effective input space dramatically, and a backend going down
disproportionately affects whichever clients were hashed to it, unlike
round-robin's even spread of that same failure.

**Q3. Why does route selection use longest-prefix match rather than the order routes were added in?**
Because the kernel needs a deterministic, unambiguous rule for overlapping
routes regardless of configuration order — "most specific match wins" gives
a predictable outcome that doesn't depend on remembering or controlling the
sequence routes were added in, which would be fragile and hard to reason
about at scale.

## Key Takeaways

1. **Route selection uses longest-prefix match, not add-order.** When
   overlapping routes exist, the more specific (longer prefix) route always
   wins, regardless of which was configured first.
2. **Only LACP bonding (`mode=4`) provides a real throughput improvement,
   and it requires switch-side cooperation.** `active-backup` (`mode=1`)
   provides redundancy only — assuming any bonding mode doubles throughput
   is a common, costly misunderstanding.
3. **A bridge operates at layer 2, connecting segments as if they were one
   wire — not routing between them.** This is the mechanism behind
   container and VM networking, distinct from anything `ip route` handles.
4. **`ip_hash` load balancing trades even distribution for session
   affinity.** Its effective fairness depends entirely on client IP
   diversity — behind a shared NAT gateway, that diversity can collapse and
   send all traffic to one backend, which is `ip_hash` working correctly,
   not malfunctioning.
5. **`X-Real-IP`/`X-Forwarded-For` headers matter beyond boilerplate.**
   Without them, every backend behind a reverse proxy sees the proxy's own
   IP as the client, breaking IP-based rate limiting, logging, and
   geolocation unless the backend is explicitly configured to trust and
   read them.
6. **Policy routing exists for exactly one class of problem: one routing
   table isn't enough.** Multi-homed servers, VPN split-tunneling, and
   asymmetric-routing avoidance are the practical reasons multiple routing
   tables and `ip rule` selection exist — recognizing when you need it
   matters more here than deep configuration, which is a large enough topic
   on its own.

> **Demo scope:** Primary concept: shaping and distributing network traffic
> across paths, links, and services. Supporting concepts: static and policy
> routing, bridging, bonding modes, nginx reverse proxy configuration,
> load-balancing algorithms.
> Estimated completion time: 28 minutes (reading + hands-on + verification).
> Checkpoints: 6 natural stopping points (after Task 1 through Task 6).

## Quick Commands Reference

### Routing

| Command | Purpose |
|---|---|
| `ip route add NET via GW dev IFACE` | Add a static route to a specific network |
| `ip route get ADDR` | Show which route the kernel would actually use for an address |
| `ip rule show` | List policy routing rules |
| `ip route show table TABLE` | Show routes in a specific table |

### Bridging and bonding

| Command | Purpose |
|---|---|
| `ip link add name BR type bridge` | Create a bridge |
| `ip link set IFACE master BR` | Add an interface to a bridge |
| `nmcli connection add type bond ... mode MODE` | Create a bond with a specific mode |
| `cat /proc/net/bonding/bondN` | Inspect a bond's current mode and slave status |

### nginx reverse proxy

| Command | Purpose |
|---|---|
| `nginx -t` | Validate config syntax before applying |
| `systemctl reload nginx` | Apply config changes without dropping connections |

### Load-balancing directives

| Directive | Behavior |
|---|---|
| *(default)* | Round-robin — even rotation across backends |
| `least_conn;` | Route to backend with fewest active connections |
| `ip_hash;` | Same client IP always routes to the same backend |

## Appendix A — Anki Cards

**Note:** extracted into a standalone `.csv` by the series extraction script.

```
#deck:Linux Mastery::Module 3 - Networking & Storage::13-network-topology-traffic-distribution
#separator:Comma
#columns:Front,Back,Tags
"Two overlapping static routes exist: a /16 and a more specific /24 for an address within that range. Which route does the kernel use?","The /24 route — the kernel uses longest-prefix match, always preferring the more specific route regardless of which was added first.","demo13,routing,longest-prefix-match"
"A bond is configured in active-backup mode (mode=1). A team expects doubled throughput after bonding two NICs. Is this expectation correct?","No — active-backup provides redundancy only, with one link active and the other on standby. Only LACP (mode=4), matched with switch-side configuration, provides a real throughput improvement.","demo13,bonding,common-mistake"
"Why does LACP bonding (mode=4) require more than just server-side configuration to actually improve throughput?","The connected switch must also support and be explicitly configured for LACP — this is not something achievable through server-side bonding configuration alone.","demo13,bonding,lacp"
"What's the fundamental difference between what a bridge does and what ip route does?","A bridge connects network segments at layer 2, forwarding traffic as though they were the same physical wire (switching); ip route operates at layer 3, directing traffic between distinct networks (routing).","demo13,bridging,fundamentals"
"An nginx upstream uses ip_hash, and all traffic lands on one backend even though there are three backends configured. Is nginx malfunctioning?","No — ip_hash is working correctly. If all client requests appear to originate from the same source IP (e.g. behind a shared NAT gateway), ip_hash has effectively one input to hash against and will consistently route to the same backend as a direct, correct consequence.","demo13,load-balancing,ip-hash,troubleshooting"
"What does an nginx reverse proxy backend see as the client's IP if proxy_set_header X-Real-IP $remote_addr is omitted?","The proxy's own IP address for every request — breaking IP-based rate limiting, logging, and geolocation on the backend unless it's explicitly configured to trust and read a forwarded-IP header.","demo13,nginx,reverse-proxy"
"What's the tradeoff of choosing ip_hash load balancing over round-robin?","ip_hash provides session affinity (same client always hits the same backend) but its load distribution is only as even as client IP diversity allows, and a backend outage disproportionately affects whichever clients were hashed to it, unlike round-robin's even spread of that failure.","demo13,load-balancing,tradeoffs"
"When does a server need policy routing (multiple routing tables) instead of a single default routing table?","When routing decisions need to depend on more than just destination address — e.g. a multi-homed server needing replies to go back out the same interface traffic arrived on (avoiding asymmetric routing), or VPN split-tunnel setups.","demo13,policy-routing,fundamentals"
"Which nginx load-balancing directive routes to whichever backend currently has the fewest active connections, and when does it fit better than round-robin?","least_conn — it fits better than plain round-robin when requests have significantly variable processing time, since round-robin doesn't account for how busy each backend currently is.","demo13,load-balancing,least-conn"
"nginx -t is run before systemctl reload nginx. What does this catch, and why does the order matter?","nginx -t validates configuration syntax without applying it, catching config errors before they'd be applied by a reload — running it first avoids reloading into a broken configuration.","demo13,nginx,workflow"
```

## Appendix B — Quiz

**Note:** extracted into a standalone `.md` by the series extraction script.

> One correct answer per question unless stated otherwise.
> Target: 80% or above before moving to Demo 14.

**Q1. A `/16` and a more specific `/24` route both match a destination address. Which route does the kernel use?**

- A) Whichever route was added first
- B) The `/24` route — longest-prefix match always wins regardless of add order
- C) Both routes are used simultaneously via load balancing
- D) The `/16` route, since broader routes take precedence

<details>
<summary>Answer</summary>

**B** — Longest-prefix match is deterministic and independent of configuration order. Trap: A and D both invent rules that aren't how Linux routing actually works; C describes ECMP-style behavior, not standard route selection.

</details>

---

**Q2. A bond is in `active-backup` mode. A team expects doubled network throughput. What's the correct assessment?**

- A) The expectation is correct — all bonding modes double throughput
- B) The expectation is incorrect — active-backup provides redundancy only, not throughput improvement
- C) The bond must be misconfigured if throughput didn't double
- D) Throughput will double automatically after the next reboot

<details>
<summary>Answer</summary>

**B** — `active-backup` trades throughput gain entirely for redundancy and switch compatibility; only LACP (`mode=4`), with switch-side support, provides a real throughput improvement. Trap: A, C, and D all assume a throughput benefit that this specific mode was never designed to provide.

</details>

---

**Q3. What does LACP bonding (`mode=4`) require beyond server-side configuration to actually work as intended?**

- A) Nothing — LACP is fully controlled by the server
- B) The connected switch must also support and be configured for LACP
- C) A minimum of four network interfaces
- D) A dedicated VLAN

<details>
<summary>Answer</summary>

**B** — LACP is a negotiated protocol between the server and switch; server-side configuration alone is insufficient. Trap: A wrongly assumes unilateral control; C and D invent requirements that don't exist.

</details>

---

**Q4. What's the fundamental operational difference between a bridge and `ip route`?**

- A) A bridge operates at layer 2 (switching between segments); `ip route` operates at layer 3 (routing between networks)
- B) They are functionally identical, just different command syntax
- C) A bridge only works with wireless interfaces
- D) `ip route` requires a bridge to function at all

<details>
<summary>Answer</summary>

**A** — This is the core distinction: bridging forwards traffic as though segments were one wire; routing directs traffic between distinct networks. Trap: B wrongly claims equivalence; C and D invent false dependencies.

</details>

---

**Q5. An `ip_hash`-configured load balancer sends all traffic to one backend. All client requests originate from the same NAT gateway IP. Is this a malfunction?**

- A) Yes — ip_hash should distribute evenly regardless of source IP
- B) No — ip_hash is working correctly; with only one effective source IP, it consistently hashes to the same backend
- C) Yes — this indicates the backend servers are down
- D) No, but only because nginx has a rare bug affecting NAT'd clients

<details>
<summary>Answer</summary>

**B** — `ip_hash` bases its routing entirely on client source IP; behind shared NAT, that diversity collapses, and consistent routing to one backend is the expected, correct behavior of the algorithm. Trap: A misunderstands what `ip_hash` optimizes for; C and D invent unrelated explanations.

</details>

---

**Q6. What does omitting `proxy_set_header X-Real-IP $remote_addr` in an nginx reverse proxy config cause?**

- A) The proxy stops forwarding requests entirely
- B) Backend servers see every request as originating from the proxy's own IP, breaking IP-based rate limiting/logging/geolocation
- C) SSL/TLS termination fails
- D) Load balancing switches to round-robin automatically

<details>
<summary>Answer</summary>

**B** — Without this header, the backend loses visibility into the actual client IP, since it only sees the proxy's connection. Trap: A, C, and D all invent unrelated failures this omission doesn't actually cause.

</details>

---

**Q7. What's the tradeoff of choosing `ip_hash` over round-robin for load balancing?**

- A) `ip_hash` is always slower to compute
- B) `ip_hash` provides session affinity but its fairness depends entirely on client IP diversity, and outages disproportionately affect hashed clients
- C) `ip_hash` requires a separate license
- D) `ip_hash` only works with UDP traffic

<details>
<summary>Answer</summary>

**B** — Session affinity is the benefit; uneven distribution when IP diversity is low (e.g. behind NAT) and disproportionate outage impact are the costs. Trap: A, C, and D all invent unrelated constraints.

</details>

---

**Q8. Why does a server need policy routing (multiple routing tables) rather than a single default table?**

- A) Policy routing is required for any server with more than one network interface
- B) When routing decisions need to depend on more than destination address alone — e.g. avoiding asymmetric routing on a multi-homed server
- C) Policy routing replaces the need for a default gateway entirely
- D) It's required whenever IPv6 is in use

<details>
<summary>Answer</summary>

**B** — Policy routing exists for scenarios like multi-homed asymmetric-routing avoidance or VPN split-tunneling, where source address or incoming interface — not just destination — needs to influence route selection. Trap: A overstates when it's needed (multiple interfaces alone don't require it); C and D invent false requirements.

</details>