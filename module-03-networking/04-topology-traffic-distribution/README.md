# Demo 13 — Topology & Traffic Distribution

## Overview

Demo 12 covered diagnosing and filtering traffic on a single machine. This
demo zooms out: how traffic gets routed across multiple networks, how two
physical interfaces can act as one for redundancy or bandwidth, and how one
public-facing address can distribute load across many backend servers. These
are the building blocks of any topology beyond "one server, one network."

---

## Demo Objectives

By the end of this demo you will be able to:

1. Configure static routes for multi-network topologies, beyond the default gateway
2. Explain policy routing at a conceptual level — multiple routing tables, rule-based selection
3. Configure a bridge to connect network segments at layer 2
4. Configure bonding for redundancy or throughput, and distinguish bonding modes
5. Configure an nginx reverse proxy with an upstream block, including client-IP-forwarding headers
6. Explain the common load-balancing algorithms and directives, and when each fits

---

## Directory Structure

```
13-topology-traffic-distribution/
├── README.md                                  # this file
├── 13-topology-traffic-distribution-anki.csv  # Anki flashcard deck
└── 13-topology-traffic-distribution-quiz.md   # standalone quiz
```

---

## Environments

| Task | Ubuntu 24.04 | Rocky Linux 9.8 | EC2 Amazon Linux |
|---|---|---|---|
| All lab tasks | ✅ | ✅ | Not required |
| nginx install | `sudo apt install nginx` | `sudo dnf install nginx` | Not required |

---

## Recall Check

> Answer from memory before reading further. From Demo 12.

1. A firewall rule allows a port, but the service is still unreachable
   from another machine. `ss -tulpn` shows it on `127.0.0.1:PORT`. What's the
   diagnosis?

2. Why is `tcpdump` diagnostically useful even after `ss` and `lsof`
   already show listening sockets and owning processes?

3. A firewalld rule worked, then vanished after a reboot. What's the
   most likely mistake, and how would you confirm it?

<details>
<summary>Answers</summary>

1. The service is bound to localhost only — unreachable from any other
   host regardless of what the firewall allows. This is a completely separate
   failure mode from firewall configuration.

2. `ss`/`lsof` only describe local socket/process state; `tcpdump`
   confirms whether packets actually arrive on the wire. If nothing shows up,
   the problem is upstream of this machine entirely.

3. The rule was almost certainly added without `--permanent`, or
   `--reload` was never run to confirm it. Confirm with `firewall-cmd
   --permanent --list-ports` — the permanent list is what survives reboot.

</details>

---

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

> ⚠️ [VERIFY] Confirmed directly: with a `/16` route via one device and a
> more specific `/24` route (inside that same `/16`) via a *different*
> device, `ip route get` for an address inside the `/24` selected the
> `/24`'s device; an address inside the `/16` but outside the `/24`
> correctly fell back to the `/16`'s device — unambiguous proof of
> longest-prefix-match, not just trusting which line appears first in
> `ip route show`.

```bash
ip route show
# 10.20.0.0/16 via 192.168.1.254 dev eth0
# 10.20.5.0/24 via 192.168.1.253 dev eth0
# Traffic to 10.20.5.10 uses the /24 route (more specific), even though
# the /16 route also technically matches.
```

`ip route get ADDR` is the direct way to confirm which route the kernel
would actually pick for a given address, without having to work it out by
hand from the full routing table — useful for verifying longest-prefix
match instead of just trusting the theory.

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

> ⚠️ [VERIFY] Confirmed directly (this same session's Demo 13e build):
> these three default rules, plus the full custom-table/rule mechanism
> this section describes, were built and tested live in network
> namespaces — see Demo 13e for the complete verified walkthrough.

Those three default rules exist on every Linux system, policy routing or
not: `local` (priority 0, always checked first — handles addresses assigned
to the machine itself), `main` (priority 32766 — the single table you've
been using all along without realizing it was "a table" at all), and
`default` (priority 32767, rarely populated — a catch-all of last resort).
Adding real policy routing means adding *additional* tables and rules
alongside these three, not replacing them.

A common real-world use: a server with two network interfaces on two ISPs
needs traffic that *arrived* on ISP A's interface to have its *replies* go
back out through ISP A, not whatever the default table would otherwise
choose — asymmetric routing like that breaks stateful firewalls and
connection tracking on the way. Policy routing solves this with a rule like
"traffic sourced from ISP A's address uses ISP A's routing table."

This demo doesn't build a full policy-routing setup — Demo 13e does,
hands-on, for exactly the multi-homed case above — but recognizing `ip rule show` and
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

> ⚠️ [VERIFY] Confirmed directly (Demo 13a's full build): bridge creation,
> attachment, and forwarding-state confirmation, exactly as shown here,
> were built and verified live with a real bridged connection — see Demo
> 13a for the complete walkthrough, including proof of MAC learning via
> `bridge fdb show`.

### 1.4 Bonding — combining interfaces for redundancy or throughput

Bonding combines multiple physical interfaces into one logical interface —
but *which* problem it solves depends entirely on the bonding mode, and
conflating them is a common mistake:

| Mode | Name | What it actually provides |
|---|---|---|
| `mode=1` | `active-backup` | **Redundancy only.** One interface active, the other standby, failover on link loss. No throughput increase — only one link carries traffic at a time. |
| `mode=4` | `802.3ad` (LACP) | **Throughput *and* redundancy**, but requires the connected switch to support and be configured for LACP — this is not something you can configure unilaterally on the server side alone. |
| `mode=0` | `balance-rr` | Round-robin across links — throughput improvement for a single connection is possible but often not switch-compatible without matching configuration, and can cause out-of-order packet delivery. |

> ⚠️ [VERIFY — docs only, not live-tested]: this session's sandboxed
> container kernel has `CONFIG_BONDING` disabled entirely (confirmed by
> reading `/proc/config.gz` directly), so none of this section's bonding
> behavior could be exercised live. This is standard, well-documented
> kernel bonding-driver behavior on any normal Ubuntu/Rocky install, not
> something specific to this course, but it wasn't independently
> re-verified in this session. Confirm on your own lab VM before treating
> the mode-comparison table as gospel.

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

**A safety note before trying this on a real server:** bonding the physical
interface you're currently connected through (e.g. over SSH) can drop your
session entirely if the bond doesn't come up cleanly — always have
out-of-band console access before bonding a machine's only path to you, or
practice on non-critical interfaces first. Task 4 in this demo uses virtual
dummy interfaces for exactly this reason — the same `nmcli` workflow, with
zero risk to your actual connection.

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
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
```

These two headers matter beyond boilerplate, and solve related but distinct
problems:

```
X-Real-IP        Carries only the single most immediate client address.
                  Simple, but loses information if the request passed
                  through more than one proxy hop before reaching this one.

X-Forwarded-For   Carries a comma-separated chain of every hop the request
                  passed through. $proxy_add_x_forwarded_for appends this
                  hop's address to whatever chain already existed in the
                  incoming request — the right choice whenever a request
                  might traverse more than one proxy, since X-Real-IP alone
                  would silently drop everything before the last hop.
```

Without either header set, every backend server sees every request as
coming from the proxy's own IP, not the actual client — breaking IP-based
rate limiting, logging, and geolocation on the backend unless it's
explicitly told to trust and read one of these headers. Most production
setups set both: `X-Real-IP` for simple, quick reads, `X-Forwarded-For` for
anything that needs the full hop chain.

> ⚠️ [VERIFY] Confirmed directly against a real backend that echoes back
> exactly what it received: with `X-Real-IP` unset, the backend had no way
> to identify the real client at all — the header was simply absent
> (`X-Real-IP=None`). With the header explicitly set to `$remote_addr`, the
> backend received it correctly. In production, where the proxy and
> backend sit on different addresses, the difference is between the
> backend seeing the proxy's IP for every request versus the real client's.

### 1.6 Load-balancing algorithms and directives

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

> ⚠️ [VERIFY] Confirmed directly against a real running `nginx` with two
> live backends: the default (round-robin) configuration alternated
> `BACKEND-1, BACKEND-2, BACKEND-1, BACKEND-2, ...` across six consecutive
> requests. Adding `ip_hash;` and repeating the identical six requests
> from the same client address returned `BACKEND-1` for all six — genuine
> sticky-session behavior, not an assumption about what the directive
> should do.

Choosing `ip_hash` for session affinity is a real tradeoff, not a free
convenience: it means load distribution is only as even as client IP
diversity allows, and a backend going down disproportionately affects
whichever clients were hashed to it, unlike round-robin's even spread of
that same failure.

**Round-robin also accepts simple per-server modifiers, which are common
directives rather than separate algorithms:**

```nginx
upstream novatech_backend {
    server 10.0.4.10:8080 weight=3;   # receives 3x the requests of a
                                       # default-weight (1) server
    server 10.0.4.11:8080;
    server 10.0.4.12:8080 backup;     # only used if all non-backup
                                       # servers are unavailable
}
```

`weight=` shifts round-robin's otherwise-even distribution toward
higher-capacity backends proportionally — a bigger server gets more
requests rather than the same share as a smaller one. `backup` keeps a
server completely idle unless every primary server is down, useful for
disaster-recovery capacity you don't want absorbing normal traffic.

---

## Part 2 — Hands-On Lab

### Commands used in this demo

| Command | Flag meaning | Purpose |
|---|---|---|
| `ip route add NET via GW dev IFACE` | — | Add a static route to a specific network |
| `ip route get ADDR` | — | Show which route the kernel would actually use for an address |
| `ip rule show` | — | List policy routing rules |
| `ip link add name BR type bridge` | — | Create a bridge interface |
| `ip link set IFACE master BR` | — | Add an interface to a bridge |
| `ip link add NAME type dummy` | — | Create a virtual test interface (safe for lab bonding practice) |
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

### Task 4 — Create a real bond using safe, virtual interfaces

Bonding your actual network interface while connected over SSH risks losing
your session if anything goes wrong. Practice the full workflow safely
first using virtual dummy interfaces — same `nmcli` commands, zero risk to
your actual connectivity:

```bash
sudo modprobe dummy
sudo ip link add dummy0 type dummy
sudo ip link add dummy1 type dummy
sudo ip link set dummy0 up
sudo ip link set dummy1 up

sudo nmcli connection add type bond con-name bond-demo ifname bond-demo mode active-backup
sudo nmcli connection add type ethernet ifname dummy0 master bond-demo
sudo nmcli connection add type ethernet ifname dummy1 master bond-demo
sudo nmcli connection up bond-demo

cat /proc/net/bonding/bond-demo
# Confirm: "Bonding Mode: fault-tolerance (active-backup)" and both dummy0
# and dummy1 listed as slave interfaces.

# Clean up
sudo nmcli connection down bond-demo
sudo nmcli connection delete bond-demo
sudo ip link del dummy0
sudo ip link del dummy1
```

**If your environment already has a real bond configured** (uncommon in a
lab VM, more common on bare-metal or a pre-provisioned training host), you
can additionally inspect it read-only:

```bash
cat /sys/class/net/bonding_masters 2>/dev/null || echo "no real bonds present"
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
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
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

---

## What You Learned

- ✅ Configured static routes for multi-network topologies beyond the default gateway
- ✅ Explained policy routing conceptually — multiple tables, rule-based selection
- ✅ Configured a bridge to connect network segments at layer 2
- ✅ Configured bonding and distinguished bonding modes by what they actually provide
- ✅ Configured an nginx reverse proxy with an upstream block, including `X-Real-IP`/`X-Forwarded-For`
- ✅ Explained load-balancing algorithms and directives, and when each fits

**Key Takeaway:** Topology decisions are full of "correct but not what you expected" outcomes — a bond mode chosen for redundancy will never show a throughput gain, and `ip_hash` behind a shared NAT gateway will correctly send everyone to one backend. In both cases the mechanism isn't broken; the mismatch is between what a feature actually guarantees and what someone assumed it would do.

---

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

---

## Interview Prep

**Q1. A team assumes bonding two NICs automatically doubles their server's network throughput. When is this true, and when is it a costly misunderstanding?**

It's only true for `mode=4` (802.3ad/LACP), and even then only if the
connected switch also supports and is configured for LACP — it's not
something the server alone can guarantee. For `active-backup` (`mode=1`),
throughput never improves at all; it trades any throughput gain entirely for
redundancy and guaranteed compatibility with switches that don't support
LACP. Assuming any bonding configuration doubles throughput is a common and
costly misunderstanding when the actual mode chosen was `active-backup`.

---

**Q2. Why would you choose `ip_hash` over round-robin for a load-balanced application, and what does that choice cost you?**

`ip_hash` provides session affinity — the same client IP consistently
reaches the same backend, useful when the application keeps session state
in memory on a specific server rather than a shared store. The cost is that
load distribution becomes only as even as client IP diversity allows — a
large population of clients behind a shared NAT gateway can collapse
`ip_hash`'s effective input space dramatically, and a backend going down
disproportionately affects whichever clients were hashed to it, unlike
round-robin's even spread of that same failure.

---

**Q3. Why does route selection use longest-prefix match rather than the order routes were added in?**

Because the kernel needs a deterministic, unambiguous rule for overlapping
routes regardless of configuration order — "most specific match wins" gives
a predictable outcome that doesn't depend on remembering or controlling the
sequence routes were added in, which would be fragile and hard to reason
about at scale.

---

## Key Takeaways

1. **Route selection uses longest-prefix match, not add-order.** When
   overlapping routes exist, the more specific (longer prefix) route always
   wins, regardless of which was configured first. `ip route get ADDR`
   confirms which route the kernel would actually pick, rather than working
   it out by hand.
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
   read them — `X-Real-IP` for the single most recent hop, `X-Forwarded-For`
   for the full chain when more than one proxy is involved.
6. **Policy routing exists for exactly one class of problem: one routing
   table isn't enough.** Multi-homed servers, VPN split-tunneling, and
   asymmetric-routing avoidance are the practical reasons multiple routing
   tables and `ip rule` selection exist — recognizing when you need it
   matters more here than deep configuration, which is a large enough topic
   on its own.
7. **Round-robin's own modifiers — `weight=` and `backup` — are common
   directives, not separate algorithms.** `weight=` proportionally shifts
   share toward higher-capacity backends; `backup` keeps a server idle
   until every primary is down, useful for disaster-recovery capacity you
   don't want absorbing normal traffic.

---

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
| `ip link add NAME type dummy` | Create a virtual test interface (safe bonding practice) |
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
| `server ... weight=N;` | Proportionally shift share toward this backend |
| `server ... backup;` | Only used if all non-backup servers are down |

---

## Appendix A — Anki Cards

**`13-topology-traffic-distribution-anki.csv`:**

```
#deck:Linux Mastery::Module 3 - Networking::Demo 13 - Topology & Traffic Distribution
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

---

## Appendix B — Quiz

**`13-topology-traffic-distribution-quiz.md`:**

````markdown
# Quiz — Demo 13: Topology & Traffic Distribution

> One correct answer per question unless stated otherwise.
> Target: 80% or above before moving to Demo 14.

---

**Q1.** You suspect two overlapping routes might cause ambiguity for a specific destination address. Which command directly confirms which route the kernel would actually use, without you having to work it out by hand from the full table?

- A) `ip route show`
- B) `ip route get ADDR`
- C) `ip rule show`
- D) `ip addr show`

<details>
<summary>Answer</summary>

**B** — `ip route get ADDR` asks the kernel to resolve the actual route it would select for that specific address, directly confirming longest-prefix match (or any other selection logic) without manual inspection of the full table. `ip route show` (A) lists all routes but doesn't resolve which one wins for a specific address; `ip rule show` (C) lists policy routing rules, not routes themselves; `ip addr show` (D) is unrelated to routing.

</details>

---

**Q2.** `ip rule show` on a freshly installed system (no custom policy routing configured) shows three rules: `lookup local`, `lookup main`, `lookup default`. What does this tell you?

- A) Policy routing is already active and configured with custom rules
- B) These three rules exist by default on every Linux system regardless of whether policy routing is used — `main` is the single table used for ordinary routing all along
- C) The system has a networking misconfiguration that needs fixing
- D) This output only appears on servers with more than one network interface

<details>
<summary>Answer</summary>

**B** — These three default rules are present on every Linux system from the start; policy routing means adding *additional* tables and rules alongside them, not replacing this baseline. The `main` table is what you've been using this entire series without it being called out as "a table."

Trap: A, C, and D all misread a universal default as something exceptional or interface-count-dependent.

</details>

---

**Q3.** `ip link add name br0 type bridge` followed by `ip link set eth1 master br0` is run. What does the second command specifically do?

- A) Deletes `eth1` and replaces it with `br0`
- B) Adds `eth1` as a member interface of the bridge `br0`, so traffic on `eth1`'s segment is now switched through the bridge
- C) Creates a static route from `eth1` to `br0`
- D) Assigns `br0`'s IP address to `eth1`

<details>
<summary>Answer</summary>

**B** — `master br0` attaches `eth1` to the bridge as a member interface; from this point, `eth1` no longer has independent routing identity — traffic on its segment is forwarded (switched) through the bridge rather than routed.

Trap: A, C, and D all invent operations `ip link set ... master` doesn't perform.

</details>

---

**Q4.** `mode=0` (`balance-rr`) bonding is being considered for a pair of NICs. What's a specific risk this mode carries that `active-backup` (`mode=1`) does not?

- A) `balance-rr` cannot provide any redundancy at all if one link fails
- B) `balance-rr` can cause out-of-order packet delivery and is often not switch-compatible without matching configuration — risks `active-backup` doesn't carry, since only one link is ever active at a time in that mode
- C) `balance-rr` requires a firmware update on the NIC before use
- D) `balance-rr` only works with wireless interfaces

<details>
<summary>Answer</summary>

**B** — Round-robin distribution across links can deliver packets out of order at the receiving end, and many switches need explicit configuration to handle two links carrying traffic for what looks like one logical connection — neither issue applies to `active-backup`, where only one link is ever active. A is incorrect — round-robin bonding does still provide some redundancy if a link fails, just not `active-backup`'s specific failover model. C and D invent unrelated requirements.

</details>

---

**Q5.** To create a bond named `bond-demo` in `active-backup` mode using `nmcli`, attaching two interfaces as slaves, what is the correct minimum command sequence?

- A) A single command: `nmcli bond create bond-demo mode=1 eth1 eth2`
- B) `nmcli connection add type bond ... mode active-backup` to create the bond, then a separate `nmcli connection add type ethernet ifname IFACE master bond-demo` for each slave interface
- C) `ip link add bond-demo type bond` alone is sufficient; slaves attach automatically
- D) Bonds cannot be created with `nmcli`; only `ip link` supports bonding

<details>
<summary>Answer</summary>

**B** — Creating the bond device and attaching each slave interface are separate `nmcli connection add` invocations — one for the bond itself (specifying the mode), then one per slave interface naming the bond as `master`. A invents a single-command shortcut that doesn't exist in this form; C and D are both factually wrong about `nmcli`'s actual bonding workflow.

</details>

---

**Q6.** Why might a lab exercise deliberately use `ip link add dummy0 type dummy` virtual interfaces to practice bonding, rather than bonding the server's real network interface directly?

- A) Dummy interfaces are required by `nmcli` and real interfaces cannot be bonded at all
- B) Bonding the interface you're currently connected through (e.g. over SSH) risks dropping your session entirely if the bond doesn't come up cleanly — dummy interfaces let you practice the same workflow with zero risk to actual connectivity
- C) Dummy interfaces are faster than real interfaces for testing purposes
- D) Real interfaces cannot be used with `active-backup` mode

<details>
<summary>Answer</summary>

**B** — This is a genuine safety practice, not a tooling limitation: bonding your only path into a remote machine carries real risk of losing access if anything goes wrong mid-configuration. Dummy interfaces are virtual and carry no such risk, letting you rehearse the exact same `nmcli` commands safely first.

Trap: A, C, and D all invent technical restrictions that don't actually exist — real interfaces can absolutely be bonded, this is purely a safety-practice choice.

</details>

---

**Q7.** An nginx reverse proxy config sets `proxy_set_header X-Real-IP $remote_addr;` but not `X-Forwarded-For`. A request passes through two proxies before reaching this one. What's the practical consequence?

- A) No difference — both headers carry identical information in every scenario
- B) `X-Real-IP` will only show the immediate previous hop's address, losing the identity of the original client further back in the chain; `X-Forwarded-For` would have preserved the full chain
- C) The request will be rejected outright since `X-Forwarded-For` is mandatory
- D) `X-Real-IP` automatically falls back to behaving like `X-Forwarded-For` when multiple hops are detected

<details>
<summary>Answer</summary>

**B** — `X-Real-IP` carries only the single most immediate address, which in a multi-hop scenario is the previous proxy, not the original client. `X-Forwarded-For` (via `$proxy_add_x_forwarded_for`) preserves the entire chain by appending each hop's address to whatever chain arrived with the request — the right choice specifically when more than one proxy might be involved.

Trap: A wrongly claims equivalence; C and D both invent behavior neither header actually has.

</details>

---

**Q8.** An `upstream` block has `server 10.0.4.10:8080 weight=3;` alongside two other servers at default weight. What does this achieve?

- A) That server is used exclusively; the other two are ignored entirely
- B) That server receives proportionally more requests (roughly 3x a default-weight server) in the round-robin rotation, useful when it has more capacity than the others
- C) That server is treated as a backup, only used if the others fail
- D) `weight=3` sets a hard cap of 3 concurrent connections to that server

<details>
<summary>Answer</summary>

**B** — `weight=` shifts round-robin's normally-even distribution proportionally toward higher-weighted servers, without excluding the others — appropriate when backend capacity is unequal. A describes what `backup` alone (without other active servers) might approximate, not what `weight` does; C describes the `backup` directive, a different modifier entirely; D invents a connection-limiting behavior `weight` doesn't have.

</details>

---

| Score | Action |
|---|---|
| 7/8 or 8/8 | Import Anki cards, move to Demo 14 |
| 6/8 | Review the wrong answers, then proceed |
| 5/8 | Re-read the relevant sections, retry those questions |
| Below 5/8 | Re-read the full demo and redo the walkthrough before proceeding |
````