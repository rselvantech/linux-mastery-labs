# Demo 13a — Linux Bridging Fundamentals

## Overview

Every container network — Docker's default bridge, Kubernetes CNI
plugins, KVM/libvirt virtual machine networking — sits on top of the
same underlying kernel feature: a **software bridge**. A bridge acts
like a virtual switch, connecting multiple network interfaces
together at Layer 2 and forwarding traffic between them by learning
which MAC address lives behind which port — exactly like a physical
network switch, just implemented in the kernel instead of hardware.
Understanding this directly demystifies "how does Docker networking
actually work," rather than treating it as opaque magic.

---

## Demo Objectives

By the end of this demo you will be able to:

1. Create a Linux bridge and attach interfaces to it with `ip link`
2. Explain what a `veth` pair is and why it always comes in a
   connected two-ended pair
3. Use a network namespace to simulate a second "host" and prove
   real Layer 2 connectivity across a bridge
4. Read a bridge's forwarding database (`bridge fdb show`) and
   explain what MAC learning actually means, with real evidence
5. Explain why this is the same underlying mechanism Docker's
   default bridge network uses

---

## Directory Structure

```
13a-bridging-fundamentals/
├── README.md                              # this file
├── 13a-bridging-fundamentals-anki.csv        # Anki flashcard deck
└── 13a-bridging-fundamentals-quiz.md         # standalone quiz
```

---

## Environments

| Task | Ubuntu 24.04 | Rocky Linux 9.8 | EC2 Amazon Linux |
|---|---|---|---|
| All lab tasks | ✅ | ✅ | Not required |

Requires `iproute2` (provides `ip` and `bridge`) and `bridge-utils`
— install with `apt install iproute2 bridge-utils` /
`dnf install iproute bridge-utils` if missing. Every command needs
root (or `NET_ADMIN` capability) — bridges and network namespaces
are kernel-level network state, not user-space configuration.

---

## Recall Check

> Answer from memory before reading further. From Demo 13.

1. Two hosts on the same subnet can't reach each other, but both
   can reach the internet. What layer is most likely broken?
2. What's the practical difference between round-robin and
   least-connections load balancing?
3. Name one command that shows real-time bandwidth usage per
   interface.

<details>
<summary>Answers</summary>

1. Layer 2 (local link) — internet connectivity working means Layer
   3 routing and above is fine; a Layer 2 issue (ARP, switch/bridge
   misconfiguration, VLAN mismatch) would specifically block
   same-subnet traffic while leaving routed traffic unaffected.
2. Round-robin distributes requests evenly in sequence regardless of
   backend load; least-connections routes each new request to
   whichever backend currently has the fewest active connections —
   better under uneven request duration.
3. `iftop` (or `nload`/`vnstat` depending on what's installed) shows
   real-time per-interface or per-connection bandwidth.

</details>

---

## Part 1 — Concepts

### 13a.1 What a bridge actually is

A Linux bridge is a virtual Layer 2 switch implemented in the
kernel. Interfaces attached to it ("ports" on the bridge) can
exchange traffic directly, and the bridge learns which MAC address
is reachable through which port — the same behavior as a physical
network switch.

```bash
ip link add br0 type bridge
ip link set br0 up
```

A newly created bridge with nothing attached does nothing useful —
it's an empty virtual switch until interfaces are added to it.

### 13a.2 `veth` pairs — always two connected ends

A `veth` (virtual ethernet) interface is created as a **pair** —
anything sent into one end comes out the other, like a virtual
patch cable:

```bash
ip link add veth-host type veth peer name veth-ns
```

This creates two interfaces, `veth-host` and `veth-ns`, permanently
linked to each other. The typical pattern: one end stays on the
host (often attached to a bridge), the other end is moved into a
network namespace or a container — this is exactly how Docker
connects a container's network stack to the host's bridge.

### 13a.3 Attaching an interface to a bridge

```bash
ip link set veth-host master br0
```

`master br0` is what actually attaches the interface — the bridge
now forwards traffic to/from this port as part of its switching
behavior. Confirm with:

```bash
ip link show master br0
bridge link show
```

> ⚠️ [VERIFY] Confirmed directly: after `ip link set veth-host master
> br0`, `bridge link show` reported `veth-host` with `state
> forwarding` — the port is actively participating in the bridge's
> switching, not just administratively attached.

### 13a.4 Network namespaces — simulating a second host

A network namespace is an isolated network stack — its own
interfaces, routing table, and IP addresses, separate from the
host's. Moving one end of a `veth` pair into a namespace simulates
"a second machine" connected to the bridge:

```bash
ip netns add ns1
ip link set veth-ns netns ns1
ip netns exec ns1 ip addr add 192.168.50.2/24 dev veth-ns
ip netns exec ns1 ip link set veth-ns up
```

Commands run with `ip netns exec ns1 ...` execute inside that
isolated network namespace — `ip addr add` inside it only affects
that namespace's view of `veth-ns`, not the host's global network
state.

### 13a.5 Proving real connectivity, not just configuration

```bash
ip addr add 192.168.50.1/24 dev br0
ping -c 2 192.168.50.2          # from the host, through the bridge, to the namespace
ip netns exec ns1 ping -c 2 192.168.50.1   # the reverse direction
```

> ⚠️ [VERIFY] Confirmed directly: both directions succeeded with real
> ICMP replies (0% packet loss, sub-millisecond RTT) — actual Layer 2
> forwarding through the bridge, not just interfaces existing in an
> "up" state with no real traffic ever tested.

### 13a.6 MAC learning — the core bridging mechanism, proven

A bridge doesn't broadcast every packet to every port forever — it
**learns** which MAC address is reachable through which port by
observing traffic, then forwards subsequent traffic to that specific
port instead of flooding everything:

```bash
bridge fdb show br br0
```

> ⚠️ [VERIFY] Confirmed directly: after the ping test above, `bridge
> fdb show` listed `veth-ns`'s actual MAC address (verified by
> separately checking `ip link show veth-ns` inside the namespace)
> against `veth-host`, dynamically — not as a `permanent` static
> entry, but learned as a direct result of the real traffic that just
> occurred.

This is the mechanism that makes a bridge behave like a switch
rather than a hub: traffic to a known MAC goes only to the port
where the bridge last saw that MAC, not flooded to every attached
port.

### 13a.7 Why this matters beyond this demo

Docker's default bridge network, Kubernetes CNI plugins (many of
which create bridges or bridge-like constructs per node), and
KVM/libvirt's default networking all build on exactly this
mechanism — a kernel bridge, `veth` pairs connecting containers/VMs
to it, and MAC learning handling the actual forwarding. Nothing in
container networking is a separate, unrelated system from what
you've just built by hand.

> ⚠️ [VERIFY — docs only, not independently re-tested against Docker/K8s/
> libvirt themselves]: this demo verified plain Linux bridging, `veth`
> pairs, and namespaces directly and thoroughly. That Docker, CNI plugins,
> and libvirt specifically build on this same mechanism is well-documented
> and consistent with their published architecture, but this session did
> not have Docker available to inspect its actual bridge and confirm the
> claim against a real container runtime. If Docker is available in your
> environment, `bridge link show` and `bridge fdb show` against `docker0`
> are the direct way to confirm this yourself.

---

## Part 2 — Hands-On Lab

### Commands used in this demo

| Command | Flag meaning | Purpose |
|---|---|---|
| `ip link add ... type bridge` | — | Create a new bridge interface |
| `ip link add ... type veth peer name ...` | — | Create a connected veth pair |
| `ip link set IFACE master BRIDGE` | — | Attach an interface to a bridge |
| `bridge link show` | — | Show interfaces attached to bridges, with forwarding state |
| `bridge fdb show` | — | Show the bridge's learned MAC-address forwarding table |
| `ip netns add NAME` | — | Create a new network namespace |
| `ip netns exec NAME cmd` | — | Run a command inside a specific network namespace |

### Task 1 — Create a bridge and a veth pair

```bash
sudo ip link add br0 type bridge
sudo ip link add veth-host type veth peer name veth-ns
sudo ip link show br0
sudo ip link show veth-host
```

### Task 2 — Attach and bring everything up

```bash
sudo ip link set veth-host master br0
sudo ip link set br0 up
sudo ip link set veth-host up
sudo ip link set veth-ns up
sudo bridge link show
```

### Task 3 — Simulate a second host with a network namespace

```bash
sudo ip netns add ns1
sudo ip link set veth-ns netns ns1
sudo ip addr add 192.168.50.1/24 dev br0
sudo ip netns exec ns1 ip addr add 192.168.50.2/24 dev veth-ns
sudo ip netns exec ns1 ip link set veth-ns up
sudo ip netns exec ns1 ip link set lo up
```

### Task 4 — Prove connectivity both directions

```bash
ping -c 2 192.168.50.2
sudo ip netns exec ns1 ping -c 2 192.168.50.1
```

### Task 5 — Inspect the learned forwarding table

```bash
bridge fdb show br br0
sudo ip netns exec ns1 ip link show veth-ns
# Compare: veth-ns's real MAC address should appear in the bridge's
# fdb output, learned dynamically (no "permanent" flag) as a result
# of the ping traffic above
```

### Task 6 — Clean up

```bash
sudo ip netns delete ns1
sudo ip link delete br0
# Deleting the bridge also removes the still-attached veth-host end;
# the veth-ns end inside the deleted namespace goes with it
ip link show | grep -E "veth|br0" || echo "clean"
```

---

## What You Learned

- ✅ Created a bridge and a `veth` pair, and attached one end to the bridge
- ✅ Used a network namespace to simulate a second host
- ✅ Proved real bidirectional connectivity through the bridge with
  actual ICMP traffic, not just interface state
- ✅ Read the bridge's forwarding database and confirmed a real MAC
  address was learned dynamically from actual traffic
- ✅ Connected this mechanism to what Docker/Kubernetes/KVM networking
  actually builds on

**Key Takeaway:** A Linux bridge is a real, inspectable Layer 2
switch running in the kernel — `bridge fdb show` after real traffic
is the concrete proof that MAC learning isn't a black box, it's a
table you can read.

---

## Part 3 — Break-Fix Scenario

### Scenario — NovaTech: two containers on the same Docker bridge can't reach each other

You're handed this ticket:

> *"Two containers on the same custom Docker bridge network can ping
> the outside internet fine, but can't ping each other by IP. Docker
> says both are running and both have IPs on the same subnet."*

**Your task — answer from what this demo covered before revealing
the resolution:**

1. Given both containers can reach the internet (implying Layer 3
   routing works), what layer does this demo's content suggest
   checking first for same-subnet-but-can't-reach-each-other symptoms?
2. What's the equivalent of `bridge fdb show` you'd check on the
   actual Docker bridge to see whether both containers' MACs are
   even visible to it?
3. What would you check about each container's veth attachment,
   translated from this demo's `ip link show master BRIDGE` pattern?

<details>
<summary>Resolution — open after attempting</summary>

**1. Layer 2 first:**

Internet connectivity working confirms Layer 3 (routing) and above
are fine for each container individually. Two containers on the
identical subnet failing to reach each other directly is a classic
Layer 2 symptom — exactly the "same subnet, internet works, direct
reach doesn't" pattern from this demo's Recall Check. Check the
bridge itself, not routing.

**2. The Docker equivalent:**

```bash
bridge fdb show br docker0    # or whatever the custom bridge is named
```

If one or both containers' veth MAC addresses are missing from the
bridge's forwarding table, the bridge either never learned them (no
traffic has flowed yet) or something is preventing the learning/
forwarding path — worth checking `ip link show master docker0` next
to confirm both containers' veth ends are actually attached.

**3. Confirming actual attachment:**

```bash
ip link show master docker0
```

If a container's veth end isn't listed here, it was never actually
attached to the bridge, regardless of what Docker's own metadata
claims — exactly the same check this demo used (`ip link show
master br0`) to confirm `veth-host` was genuinely part of `br0`, not
just configured to be.

**The lesson:** container networking issues that look mysterious
through Docker's abstraction layer are often plain bridging issues
underneath — the same `bridge fdb show` / `ip link show master`
commands from this demo work identically on Docker's bridges,
because it's the same kernel mechanism.

</details>

---

## Interview Prep

**Q1. What's the practical difference between a `veth` pair and a
regular network interface?**

A `veth` interface is always created as a connected pair — anything
sent into one end comes out the other, like a virtual patch cable.
It has no independent existence; it's specifically designed to
connect two network namespaces (or a namespace to the host) to each
other, which is exactly how container runtimes attach a container's
isolated network stack to the host's bridge.

---

**Q2. How would you prove a Linux bridge is actually forwarding
traffic, not just that the interfaces are administratively "up"?**

Generate real traffic across it (a ping between two endpoints
attached to the bridge) and then check `bridge fdb show` — a
dynamically learned MAC entry (no `permanent` flag) for the
responding endpoint is direct evidence the bridge actually
processed and forwarded real packets, not just that link state
shows "up."

---

**Q3. Two VMs on the same libvirt bridge network can't reach each
other, though the bridge itself shows both interfaces attached. What
would you check next?**

Whether the bridge has actually learned each VM's MAC address —
`bridge fdb show` — since attachment alone doesn't guarantee
forwarding is working (a misconfigured interface could be attached
but not actually passing traffic). If MACs aren't appearing despite
real traffic attempts, the next check is whether traffic is even
reaching the bridge port at all (interface state, cabling/veth
integrity for virtual setups).

---

## Key Takeaways

1. **A Linux bridge is a real Layer 2 switch running in the kernel — `ip link set IFACE master BRIDGE` is what actually attaches a port, confirmed via `bridge link show`'s `state forwarding`.**
2. **A `veth` interface is always created as a connected pair — traffic into one end comes out the other, which is exactly the mechanism container runtimes use to connect an isolated network namespace to the host's bridge.**
3. **A network namespace is a fully isolated network stack, usable to simulate a second host for testing without needing real separate machines.**
4. **Real connectivity across a bridge should be proven with actual traffic (a ping), not assumed from interface state alone — confirmed directly with real bidirectional ICMP replies.**
5. **`bridge fdb show` after real traffic is concrete, inspectable proof of MAC learning — a dynamically learned (non-`permanent`) entry means the bridge genuinely processed that traffic, not just that a static rule exists.**
6. **Docker's default bridge, Kubernetes CNI, and KVM/libvirt networking all build on this exact mechanism — the same `bridge fdb show` / `ip link show master` commands apply directly when debugging container or VM networking issues.**

---

## Quick Commands Reference

### Bridge setup

| Command | Purpose |
|---|---|
| `ip link add BR type bridge` | Create a bridge |
| `ip link add A type veth peer name B` | Create a connected veth pair |
| `ip link set IFACE master BR` | Attach an interface to a bridge |
| `ip link set IFACE up` | Bring an interface up |

### Namespaces

| Command | Purpose |
|---|---|
| `ip netns add NAME` | Create a network namespace |
| `ip netns exec NAME cmd` | Run a command inside it |
| `ip netns delete NAME` | Remove it |

### Inspection

| Command | Purpose |
|---|---|
| `bridge link show` | Attached interfaces and forwarding state |
| `bridge fdb show br BRIDGE` | Learned MAC-address forwarding table |
| `ip link show master BRIDGE` | Confirm which interfaces are actually attached |

---

## Appendix A — Anki Cards

**`13a-bridging-fundamentals-anki.csv`:**

```
#deck:Linux Mastery::Module 3 - Networking::Demo 13a - Linux Bridging Fundamentals
#separator:Comma
#columns:Front,Back,Tags

"What is a Linux bridge, conceptually?","A virtual Layer 2 switch implemented in the kernel. Interfaces attached to it (bridge ports) can exchange traffic directly, and the bridge learns which MAC address is reachable through which port, exactly like a physical network switch.","demo13a,bridge,concept"

"What is a veth pair, and why does it always come as two ends?","A veth (virtual ethernet) interface is created as a connected PAIR - anything sent into one end comes out the other, like a virtual patch cable. This is exactly the mechanism container runtimes use to connect an isolated network namespace's interface to the host's bridge.","demo13a,veth,pair"

"ip link set veth-host master br0 is run. What does 'master' actually do, and how do you confirm it worked?","'master br0' attaches the interface to the bridge, making it an active port that participates in the bridge's switching. Confirm with 'bridge link show' - a correctly attached port shows 'state forwarding', not just administratively linked.","demo13a,bridge,attach,master"

"How do you PROVE a bridge is actually forwarding traffic, rather than just showing interfaces as administratively up?","Generate real traffic across it (e.g. a ping between two endpoints attached to the bridge), then check bridge fdb show - a dynamically learned MAC entry (no 'permanent' flag) is direct evidence the bridge processed and forwarded real packets, not just that link state shows up.","demo13a,bridge,verify,fdb"

"What does 'bridge fdb show' actually display, and what does a non-permanent entry mean?","The bridge's forwarding database - which MAC addresses the bridge has learned are reachable through which port. A non-permanent (dynamically learned) entry means the bridge observed real traffic from that MAC and recorded which port it arrived on - proof of actual learning, not static configuration.","demo13a,bridge,fdb,mac-learning"

"What is a network namespace, and what's it used for in this demo?","A fully isolated network stack - its own interfaces, routing table, and IP addresses, separate from the host's. Used here to simulate a second 'host' connected to the bridge via one end of a veth pair, without needing an actual separate machine.","demo13a,netns,namespace"

"Two containers on the same Docker bridge network can reach the internet but can't reach each other. What layer does this symptom point to?","Layer 2 (local link/switching), not Layer 3 routing - internet connectivity working confirms routing is fine for each container individually. Same-subnet-but-cant-reach-each-other is the classic Layer 2 symptom, pointing at the bridge itself (attachment, MAC learning) rather than routing configuration.","demo13a,bridge,layer2,troubleshooting"

"What's the Docker-networking equivalent of this demo's 'ip link show master br0' check?","ip link show master docker0 (or whatever the custom bridge is named) - confirms which veth interfaces are ACTUALLY attached to Docker's bridge, regardless of what Docker's own container metadata claims. The exact same kernel-level check applies since Docker bridges use this identical mechanism.","demo13a,bridge,docker,verify"

"Why does deleting a bridge with 'ip link delete br0' also remove the attached veth-host interface?","The veth-host end was attached (mastered) to br0; deleting the bridge tears down its attached ports as part of removing the bridge itself. The other end of the veth pair (inside a namespace, if not separately deleted) goes with whatever cleanup removes its associated namespace or the pair itself.","demo13a,bridge,cleanup,veth"

"What underlying kernel mechanism do Docker's default bridge network, Kubernetes CNI plugins, and KVM/libvirt networking all share?","A kernel bridge, veth pairs connecting containers/VMs to it, and MAC learning handling actual forwarding - the exact same mechanism built by hand in this demo. Container/VM networking isn't a separate system; it's this mechanism with automation layered on top.","demo13a,bridge,docker,kubernetes,libvirt"
```

---

## Appendix B — Quiz

**`13a-bridging-fundamentals-quiz.md`:**

````markdown
# Quiz — Demo 13a: Linux Bridging Fundamentals

> One correct answer per question. Target: 80% before Demo 13b.

---

**Q1.** What is a Linux bridge, conceptually?

- A) A firewall rule set that filters traffic between interfaces
- B) A virtual Layer 2 switch implemented in the kernel, forwarding
  traffic between attached interfaces by learning MAC addresses
- C) A routing table that directs traffic between subnets
- D) A DNS resolver for container hostnames

<details>
<summary>Answer</summary>

**B** — A bridge operates at Layer 2, learning which MAC address is
reachable through which attached port, exactly like a physical
network switch, just implemented in software in the kernel.

</details>

---

**Q2.** Why does a `veth` interface always get created as a pair,
never on its own?

- A) It's a historical accident with no current functional reason
- B) Anything sent into one end comes out the other, like a virtual
  patch cable — this is what makes it useful for connecting an
  isolated network namespace to the host's bridge
- C) Single veth interfaces are not supported by the Linux kernel at all
- D) A pair is required only for IPv6, not IPv4

<details>
<summary>Answer</summary>

**B** — The paired nature is the entire point — it's a virtual
cable connecting two network contexts (e.g. a container's namespace
and the host), which single-ended interfaces can't do.

</details>

---

**Q3.** After `ip link set veth-host master br0`, how do you confirm
the interface is actually forwarding, not just administratively
attached?

- A) `ip addr show veth-host` — any IP address confirms forwarding
- B) `bridge link show` — a correctly attached, forwarding port
  shows `state forwarding`
- C) There's no way to confirm this without generating traffic first
- D) `ping` the interface's own MAC address directly

<details>
<summary>Answer</summary>

**B** — `bridge link show`'s `state forwarding` output is the
direct confirmation that the port is actively part of the bridge's
switching behavior, not just linked.

</details>

---

**Q4.** `bridge fdb show` lists a MAC address against a port with no
`permanent` flag. What does this specifically indicate?

- A) The entry is invalid and should be ignored
- B) That MAC was dynamically learned from real observed traffic —
  concrete evidence the bridge actually forwarded packets from it
- C) The interface is about to be automatically removed
- D) `permanent` entries are the only ones that actually work

<details>
<summary>Answer</summary>

**B** — A non-permanent entry is proof of dynamic learning from
genuine traffic, which is stronger evidence of working connectivity
than interface state alone.

</details>

---

**Q5.** Two containers on the same Docker bridge network can reach
the internet but not each other. What does this symptom point to?

- A) A DNS resolution failure
- B) A Layer 2 issue with the bridge itself — internet connectivity
  working confirms Layer 3 routing is fine for each container
  individually; same-subnet-but-can't-reach-each-other is the
  classic Layer 2 symptom
- C) A firewall rule blocking all traffic, including internet access
- D) Both containers must be on different bridges by definition

<details>
<summary>Answer</summary>

**B** — This matches the demo's Recall Check pattern exactly:
internet access working rules out routing; direct same-subnet
failure points at the bridge/switching layer specifically.

</details>

---

**Q6.** What's the Docker-networking equivalent of this demo's `ip
link show master br0` check?

- A) `docker network inspect` alone, with no kernel-level check needed
- B) `ip link show master docker0` (or the custom bridge's name) —
  the identical kernel-level check, since Docker bridges use the
  same underlying mechanism
- C) `docker logs` on each container
- D) There is no equivalent; Docker networking is unrelated to Linux bridging

<details>
<summary>Answer</summary>

**B** — Docker's bridge networking is built on exactly this kernel
mechanism, so the same `ip link`/`bridge` commands apply directly,
not just conceptually.

</details>

---

**Q7.** A network namespace is created and a `veth` end is moved
into it. What is a network namespace, functionally?

- A) A firewall zone with predefined rules
- B) A fully isolated network stack — its own interfaces, routing
  table, and IP addresses, separate from the host's
- C) A DNS view specific to one application
- D) A read-only snapshot of the host's current network state

<details>
<summary>Answer</summary>

**B** — This is what makes it useful for simulating a second "host"
for testing bridging and routing without needing an actual separate
machine.

</details>

---

| Score | Action |
|---|---|
| 7/7 | Import Anki cards, move to Demo 13b |
| 6/7 | Review the wrong answer, then proceed |
| 5/7 | Re-read the relevant section, retry those questions |
| Below 5/7 | Re-read the full demo and redo the walkthrough before proceeding |
````