# Demo M03-07 — VPN Fundamentals

## Overview

Demo M03-04 covered how traffic is routed, bridged, bonded, and distributed
across networks you control. This demo covers what happens when the path
between two hosts is a network you *don't* control: a VPN builds a
private, encrypted layer-3 link across it. The focus is WireGuard, because
it is a small enough tool to understand end to end, with a conceptual
comparison to OpenVPN. The recurring production problem with VPNs is
rarely the encryption. It is routing, firewalls, and above all MTU: a
tunnel that is "up" but silently fails on anything bigger than a small
packet.

---

## Demo Objectives

By the end of this demo you will be able to:

1. Explain what a VPN tunnel is at the Linux level: a virtual interface, routes, and encapsulation
2. Configure a WireGuard point-to-point tunnel and verify it with `wg show`
3. Explain `AllowedIPs` and its two roles: outbound routing and inbound source filtering
4. Prove the traffic is actually encrypted by comparing a capture on the tunnel interface against one on the underlying link
5. Calculate tunnel MTU from encapsulation overhead and diagnose an MTU black hole
6. Compare WireGuard and OpenVPN conceptually and state when each fits
7. Triage a broken tunnel in order: handshake, routing and `AllowedIPs`, then MTU

---

## Directory Structure

```
07-vpn-fundamentals/
├── README.md                         # this file
├── m03-07-vpn-fundamentals-anki.csv  # Anki flashcard deck
└── m03-07-vpn-fundamentals-quiz.md   # standalone quiz
```

---

## Environments

| Task | Ubuntu 24.04 | Rocky Linux 9.8 | EC2 Amazon Linux |
|---|---|---|---|
| All lab tasks | ✅ | ✅ | Not required |
| Install | `sudo apt install wireguard-tools tcpdump nftables` | `sudo dnf install epel-release && sudo dnf install wireguard-tools tcpdump nftables` | Not required |

> ⚠️ [VERIFY — docs only, not live-tested]: Ubuntu 24.04's kernel ships
> the `wireguard` module. On Rocky 9 the module's availability depends on
> the kernel build; confirm with `sudo modprobe wireguard && lsmod | grep
> wireguard` before starting, and note the result here. This lab touches
> no real interface: everything runs inside two network namespaces joined
> by a virtual cable, so it cannot interrupt an SSH session.

---

## Recall Check

> Answer from memory before reading further. From Demo M03-04.

1. Two static routes overlap: a `/16` via one gateway and a `/24` inside
   it via another. Which is used for an address in the `/24`, and which
   command proves it?

2. A bond in `active-backup` mode was expected to double throughput. Why
   was that never realistic?

3. Why does a multi-homed server need more than one routing table?

<details>
<summary>Answers</summary>

1. The `/24`: longest-prefix match wins regardless of the order the routes
   were added. `ip route get ADDR` shows the route the kernel would
   actually choose.

2. `active-backup` provides redundancy only: one link carries traffic and
   the other waits as standby. Only LACP (`mode=4`), with matching switch
   configuration, can improve throughput.

3. Because routing sometimes has to depend on more than the destination,
   such as the source address, so that replies leave through the same
   interface the request arrived on. Policy routing (`ip rule`) selects
   among multiple tables for that. This matters again in this demo: a
   full-tunnel VPN relies on it.

</details>

---

## Part 1 — Concepts

### 1.1 What a VPN tunnel is, in Linux terms

A VPN is not a special kind of connection. On Linux it is three ordinary
things working together:

1. **A virtual network interface** (`wg0`, `tun0`). Packets sent into it
   are handed to the VPN software instead of a physical NIC.
2. **Routes** that decide which destinations go into that interface.
   Everything from Demo M03-01 and Demo M03-04 applies unchanged.
3. **Encapsulation and encryption.** The software wraps each inner packet
   inside an encrypted outer packet (for WireGuard, a UDP datagram) and
   sends it across the real network to the peer, which unwraps it.

Two distinctions come up constantly:

| Term | Meaning |
|---|---|
| **Site-to-site** | Two networks joined through a pair of gateways; hosts behind them need no VPN software. Gateways must forward packets (`net.ipv4.ip_forward=1`). |
| **Remote access** | A single client connects into a network. Often needs NAT or routing on the server so the client can reach internal hosts. |
| **Full tunnel** | All of the client's traffic goes through the VPN (default route via the tunnel). |
| **Split tunnel** | Only specific destination networks go through the VPN; everything else uses the normal gateway. |

Full versus split tunnel is purely a routing decision, which makes it the
same longest-prefix-match logic from Demo M03-04 §1.1.

### 1.2 The WireGuard model

WireGuard's design is deliberately small. A tunnel consists of:

- **An interface** with an IP address on a private tunnel subnet
- **A private key** for this host and **a public key** for each peer
- **A list of peers**, each with a public key, optional endpoint
  (`IP:port`), and `AllowedIPs`

```bash
wg genkey | tee private.key | wg pubkey > public.key
```

Public keys are exchanged out of band (you copy them yourself). There is
no certificate authority, no negotiation of cipher suites, and no
username/password layer. WireGuard uses a fixed, modern set of
primitives, so there is nothing to misconfigure at that level.

**`AllowedIPs` is the concept that matters most, because it does two jobs:**

```
Outbound  A packet leaving through wg0 is sent to the peer whose
          AllowedIPs contains the packet's destination. It acts as a
          routing table that selects the peer.

Inbound   A decrypted packet from a peer is accepted only if its source
          address is inside that peer's AllowedIPs. It acts as a source
          filter, dropping spoofed or misrouted inner packets.
```

This is called *cryptokey routing*. Two consequences follow. First, if no
peer's `AllowedIPs` covers a destination, the sender gets an error
(`Required key not available`) rather than a silent drop. Second,
`AllowedIPs = 0.0.0.0/0` means "this peer can carry any destination",
which is how a full tunnel is expressed.

**WireGuard is silent and stateless from the outside.** It runs over UDP
(default port 51820), does not answer packets from unknown keys, and only
performs a handshake when there is traffic to send. It re-keys roughly
every two minutes while traffic flows. Two practical results:

- A peer that has been idle shows an old (or no) "latest handshake". That
  is normal, not a fault.
- A host behind NAT must send something periodically to keep its NAT
  mapping open, or the other side can no longer reach it.
  `PersistentKeepalive = 25` on the NATed peer does this.

A persistent configuration uses `wg-quick`:

```ini
# /etc/wireguard/wg0.conf   (mode 600, owned by root)
[Interface]
Address = 10.200.0.1/24
ListenPort = 51820
PrivateKey = <this-host-private-key>

[Peer]
PublicKey = <peer-public-key>
AllowedIPs = 10.200.0.2/32
```

```bash
sudo wg-quick up wg0
sudo systemctl enable --now wg-quick@wg0
```

> **Private keys are credentials.** Keep `wg0.conf` at mode `600`, never
> commit it to version control (a theme from the series' Git-based
> firewall-rule practice), and rotate a key if it ever leaks.

> ⚠️ [VERIFY — docs only, not live-tested]: when a `wg-quick` peer uses
> `AllowedIPs = 0.0.0.0/0`, `wg-quick` installs policy-routing rules using
> a firewall mark, so that the tunnel's own UDP packets bypass the tunnel
> instead of looping. This is the practical reason Demo M03-04 §1.2 exists.
> Confirm with `ip rule show` after bringing up a full-tunnel config on
> your lab VM.

### 1.3 WireGuard vs. OpenVPN (conceptual)

| | WireGuard | OpenVPN |
|---|---|---|
| Where it runs | Kernel module (on Linux), small codebase | User-space daemon over a `tun`/`tap` device |
| Authentication | Static public-key pairs exchanged out of band | TLS with certificates (PKI) or static keys, plus optional username/password and plugins |
| Crypto | Fixed modern set, no negotiation | Configurable ciphers and TLS versions |
| Transport | UDP only | UDP or TCP (TCP can pass restrictive firewalls, at a performance cost) |
| Configuration | A few lines per peer | Larger config, certificate lifecycle to manage |
| Strengths | Simple, fast, easy to audit | Mature, wide client support, rich enterprise features (per-user auth, pushed routes, layer-2 `tap`) |

Neither is "better". WireGuard fits host-to-host and site-to-site links
where you control both ends and want a small, predictable config. OpenVPN
fits large fleets that need certificate-based identity, per-user
revocation, or TCP/443 traversal. IPsec is a third common option you will
meet on cloud VPN gateways; it is out of scope here. The troubleshooting
below (routes, firewall, MTU) applies to all of them.

### 1.4 MTU over a tunnel: the silent failure

Encapsulation adds bytes. If the outer packet no longer fits the physical
link's MTU, one of three things happens: it is fragmented, it is dropped
with an ICMP "fragmentation needed" message (path MTU discovery), or it is
dropped silently. The third is the dangerous one.

WireGuard's per-packet overhead:

```
Outer IPv4 header     20 bytes   (40 bytes if the underlay is IPv6)
UDP header             8 bytes
WireGuard header+tag  32 bytes
                      ------------------------------
IPv4 underlay         60 bytes
IPv6 underlay         80 bytes
```

WireGuard's default interface MTU is **1420**, which is `1500 - 80`: it
budgets for the worst case (IPv6 underlay) on a standard 1500-byte link.
If the real path MTU is lower (PPPoE, another tunnel underneath, a cloud
overlay, some mobile networks), the default is too large.

**The classic symptom pattern:**

- `ping` works (small packets)
- SSH login works, but the session freezes when a command prints a lot of
  output
- HTTPS connects to a site, then stalls loading a large page
- `scp`/`curl` of a big file hangs at a few KB

Small packets fit; anything near full size does not. Because the same
packets that fail are the large ones, the tunnel looks healthy by every
quick test, which is why this class of fault wastes hours.

**Diagnosis uses the "do not fragment" flag:**

```bash
ping -M do -s 1372 10.200.0.2     # -M do sets DF; -s is the ICMP payload size
```

Packet size on the wire is payload + 28 (20 IP + 8 ICMP). Find the largest
payload that still gets a reply, add 28, and that is the effective MTU of
the path through the tunnel. `tracepath` does a similar discovery
automatically.

Two different failures look different:

| Output | Meaning |
|---|---|
| `ping: local error: message too long, mtu=1320` | The *local* interface MTU is smaller than the packet. Fast, honest, and a hint at the tunnel's configured MTU. |
| Silence (100% loss) at sizes above a threshold | The packet left, but a hop on the path dropped it and nothing told you. This is the black hole. |

**Fixes, in order of preference:**

1. **Set the tunnel MTU correctly.** In `wg-quick`, add `MTU = 1380` (or
   whatever the path supports) to `[Interface]`, or `ip link set dev wg0
   mtu N`. TCP derives its segment size from the interface MTU, so TCP
   flows then self-correct.
2. **Clamp TCP MSS on a forwarding gateway** when you cannot control every
   client, so TCP handshakes advertise a segment size that fits:

   ```bash
   sudo nft add rule inet filter forward tcp flags syn tcp option maxseg size set rt mtu
   ```

3. Fix the broken hop (allow ICMP "fragmentation needed", stop dropping
   fragments). Often outside your control, which is why 1 and 2 exist.

### 1.5 Triage order for a broken tunnel

Work from the bottom of the stack upward, and let `wg show` tell you which
layer to look at:

| `wg show` evidence | Layer | Check next |
|---|---|---|
| Sent bytes grow, received is 0, no `latest handshake` | **Reachability / keys** | Is UDP 51820 allowed through every firewall (Demo M03-02/12a)? Is the endpoint right? Are the public keys the right way round? |
| Handshake is recent, but pings fail | **Routing / `AllowedIPs`** | `ip route get` for the destination, both sides' `AllowedIPs`, `ip_forward` on gateways, NAT if needed |
| Handshake recent, small packets work, large ones hang | **MTU** | `ping -M do -s N`, interface MTU, fragment/ICMP handling on the path |

Start with `wg show`, not with packet captures. Its counters usually
identify the layer in one command.

---

## Part 2 — Hands-On Lab

### Commands used in this demo

| Command | Flag meaning | Purpose |
|---|---|---|
| `ip netns add NAME` | — | Create an isolated network namespace (a private network stack for the lab) |
| `ip link add A type veth peer name B` | — | Create a virtual cable with two ends |
| `ip link add wg0 type wireguard` | — | Create a WireGuard interface |
| `wg genkey` / `wg pubkey` | — | Generate a private key / derive its public key |
| `wg set wg0 peer KEY ...` | `allowed-ips`, `endpoint` | Configure a peer on a running interface |
| `wg show` | — | Show handshake time, transfer counters, peers |
| `ip netns exec NS CMD` | — | Run a command inside a namespace |
| `ping -M do -s N` | `-M do`: set DF; `-s`: payload size | Probe path MTU |
| `tcpdump -ni IFACE -c N` | `-n`: no DNS; `-i`: interface; `-c`: stop after N packets | Capture packets on one interface |

### Task 1 — Build the underlay: two namespaces and a virtual cable

```bash
sudo ip netns add ns-a
sudo ip netns add ns-b

sudo ip link add veth-a type veth peer name veth-b
sudo ip link set veth-a netns ns-a
sudo ip link set veth-b netns ns-b

sudo ip -n ns-a addr add 10.99.0.1/24 dev veth-a
sudo ip -n ns-b addr add 10.99.0.2/24 dev veth-b
sudo ip -n ns-a link set lo up
sudo ip -n ns-b link set lo up
sudo ip -n ns-a link set veth-a up
sudo ip -n ns-b link set veth-b up

sudo ip netns exec ns-a ping -c 2 10.99.0.2
# Confirm: the underlay works before any VPN exists. If this fails, nothing
# later is a WireGuard problem.
```

`ns-a` and `ns-b` stand in for two separate hosts. `10.99.0.0/24` is the
"real network" between them; the tunnel will use `10.200.0.0/24`.

### Task 2 — Generate keys

```bash
mkdir -p /tmp/wg-lab && cd /tmp/wg-lab
umask 077
wg genkey | tee a.key | wg pubkey > a.pub
wg genkey | tee b.key | wg pubkey > b.pub
ls -l
# Confirm: key files are mode 600. Each side needs its own private key and
# the OTHER side's public key, never the other side's private key.
```

### Task 3 — Configure the tunnel and verify

```bash
cd /tmp/wg-lab

# Side A
sudo ip -n ns-a link add wg0 type wireguard
sudo ip netns exec ns-a wg set wg0 \
  private-key a.key listen-port 51820 \
  peer "$(cat b.pub)" allowed-ips 10.200.0.2/32 endpoint 10.99.0.2:51820
sudo ip -n ns-a addr add 10.200.0.1/24 dev wg0
sudo ip -n ns-a link set wg0 up

# Side B
sudo ip -n ns-b link add wg0 type wireguard
sudo ip netns exec ns-b wg set wg0 \
  private-key b.key listen-port 51820 \
  peer "$(cat a.pub)" allowed-ips 10.200.0.1/32 endpoint 10.99.0.1:51820
sudo ip -n ns-b addr add 10.200.0.2/24 dev wg0
sudo ip -n ns-b link set wg0 up

# Before any traffic: no handshake yet (WireGuard is on-demand)
sudo ip netns exec ns-a wg show

# Generate traffic through the tunnel
sudo ip netns exec ns-a ping -c 3 10.200.0.2

sudo ip netns exec ns-a wg show
# Confirm: "latest handshake" is now a few seconds ago, and "transfer"
# shows bytes both received and sent.
sudo ip netns exec ns-a ip route get 10.200.0.2
# Confirm: the route goes via dev wg0.
```

### Task 4 — Prove the traffic is encrypted

Capture on the tunnel interface and on the underlying link at the same
time, while pinging:

```bash
sudo ip netns exec ns-b timeout 6 tcpdump -ni veth-b -c 4 udp port 51820 &
sudo ip netns exec ns-b timeout 6 tcpdump -ni wg0 -c 4 icmp &
sleep 1
sudo ip netns exec ns-a ping -c 3 10.200.0.2
wait
```

Confirm the difference between the two captures:

- On `wg0` you see readable `ICMP echo request/reply` between `10.200.0.1`
  and `10.200.0.2`. This is the decrypted inner traffic.
- On `veth-b` you see only `UDP` between `10.99.0.1` and `10.99.0.2` on
  port 51820, with opaque lengths and no ICMP visible. This is what any
  device in the middle of a real network would see.

### Task 5 — Reproduce and fix an MTU black hole

Note the starting state:

```bash
sudo ip -n ns-a link show wg0 | grep -o 'mtu [0-9]*'
# Expected: mtu 1420 (the WireGuard default)
```

**Step 5a — Simulate a path with a smaller MTU that also drops fragments.**
Real paths like this exist (a lower-MTU link plus a firewall that drops
fragments):

```bash
sudo ip -n ns-a link set veth-a mtu 1400
sudo ip -n ns-b link set veth-b mtu 1400

sudo ip netns exec ns-b nft add table ip fraglab
sudo ip netns exec ns-b nft add chain ip fraglab pre \
  '{ type filter hook prerouting priority -450; policy accept; }'
sudo ip netns exec ns-b nft add rule ip fraglab pre \
  'ip frag-off & 0x3fff != 0 counter drop'
```

> ⚠️ [VERIFY — docs only, not live-tested]: priority `-450` places the
> rule before IPv4 defragmentation (`-400`), so the rule sees fragments
> before they are reassembled. If the counter in step 5c stays at zero
> while pings still fail, lower the priority further and re-test.

**Step 5b — Observe the symptom.** Small works; large does not:

```bash
sudo ip netns exec ns-a ping -c 3 -s 56 10.200.0.2        # works
sudo ip netns exec ns-a ping -c 3 -W 2 -s 1372 10.200.0.2 # 100% loss
```

The tunnel is up, `wg show` shows a recent handshake, and pings work, yet
anything near full size vanishes. This is the page-the-network-team
moment.

**Step 5c — Diagnose.** Find the largest payload that gets through:

```bash
sudo ip netns exec ns-a ping -M do -c 2 -W 2 -s 1312 10.200.0.2   # works
sudo ip netns exec ns-a ping -M do -c 2 -W 2 -s 1313 10.200.0.2   # fails
sudo ip netns exec ns-b nft list ruleset
# Confirm: the counter on the drop rule has increased, so fragments are
# arriving and being discarded.
```

1312 + 28 = 1340 bytes is the largest inner packet the path can carry.
That equals `1400 (link MTU) - 60 (IPv4 + UDP + WireGuard)`, which is the
overhead arithmetic from §1.4 reproduced by measurement.

**Step 5d — Fix the tunnel MTU on both ends:**

```bash
sudo ip -n ns-a link set wg0 mtu 1320
sudo ip -n ns-b link set wg0 mtu 1320

sudo ip netns exec ns-a ping -M do -c 2 -s 1292 10.200.0.2   # works (1292+28=1320)
sudo ip netns exec ns-a ping -M do -c 2 -s 1372 10.200.0.2   # fails fast, locally
sudo ip netns exec ns-a ping -c 3 -s 1372 10.200.0.2         # works (no -M do)
```

Interpret the last three results:

- `-s 1292 -M do` works: the packet now fits the tunnel and the path.
- `-s 1372 -M do` now fails immediately with `local error: message too
  long, mtu=1320`. Failing loudly and locally is the improvement over
  silent loss.
- `-s 1372` without `-M do` works because the sender fragments the *inner*
  packet to fit `wg0`'s MTU before encryption, so every outer packet fits
  the link.

### Task 6 — Break `AllowedIPs` and read the error

```bash
cd /tmp/wg-lab
sudo ip netns exec ns-a wg set wg0 peer "$(cat b.pub)" allowed-ips 10.200.0.99/32

sudo ip netns exec ns-a ping -c 2 10.200.0.2
# Expected: "sendmsg: Required key not available". The route still points
# at wg0, but no peer's AllowedIPs covers 10.200.0.2, so WireGuard has
# nowhere to send the packet.

# Restore
sudo ip netns exec ns-a wg set wg0 peer "$(cat b.pub)" allowed-ips 10.200.0.2/32
sudo ip netns exec ns-a ping -c 2 10.200.0.2
```

This error is distinct from a timeout. It tells you the problem is local
configuration (`AllowedIPs`), not the network.

### Task 7 — Clean up

```bash
sudo ip netns del ns-a
sudo ip netns del ns-b
rm -rf /tmp/wg-lab
ip netns list
# Deleting a namespace removes the interfaces inside it, including wg0, the
# veth pair, and the nftables rules.
```

---

## What You Learned

- ✅ Explained a VPN tunnel as an interface plus routes plus encapsulation
- ✅ Configured a WireGuard tunnel and verified it with `wg show`
- ✅ Explained `AllowedIPs` as both outbound routing and inbound source filtering
- ✅ Proved encryption by comparing captures on `wg0` and the underlying link
- ✅ Calculated tunnel MTU from overhead and diagnosed an MTU black hole with `ping -M do`
- ✅ Compared WireGuard and OpenVPN and stated when each fits
- ✅ Triaged a broken tunnel in order: handshake, routing, MTU

**Key Takeaway:** A VPN failure almost never means "the encryption is
broken". It means one specific layer is wrong, and `wg show` tells you
which: no handshake points to reachability, a handshake with failed pings
points to routing or `AllowedIPs`, and a healthy handshake where only
large packets fail points to MTU. A tunnel that passes `ping` has proven
only that small packets work.

---

## Part 3 — Break-Fix Scenario

### Scenario — NovaTech Ops: two tunnel incidents

> **Incident report — NovaTech Ops, ticket #4561**
>
> Alert 1: "Site-to-site tunnel to the Ottawa office is up and pings work,
> but SSH sessions freeze whenever a command prints a long output, and
> `scp` stalls at a few KB." Alert 2: "A new WireGuard client cannot reach
> anything. It was configured an hour ago and has never connected."

**Incident 1 — tunnel up, but large transfers hang**

```bash
$ sudo wg show wg0
interface: wg0
  public key: Xk3...=
  listening port: 51820

peer: Qm9...=
  endpoint: 203.0.113.20:51820
  allowed ips: 10.200.0.0/24
  latest handshake: 21 seconds ago
  transfer: 48.20 KiB received, 61.77 KiB sent

$ ip link show wg0 | grep -o 'mtu [0-9]*'
mtu 1500
$ ping -c 2 10.200.0.2
2 packets transmitted, 2 received, 0% packet loss
$ ping -M do -c 2 -W 2 -s 1400 10.200.0.2
2 packets transmitted, 0 received, 100% packet loss
$ ping -M do -c 2 -W 2 -s 1272 10.200.0.2
2 packets transmitted, 2 received, 0% packet loss
```

Tasks:
1. Which layer from §1.5 does the evidence point to, and which single line
   of output rules out the other two layers?
2. Why does `ping` succeed while SSH output freezes?
3. What is the largest inner packet this path carries, and what should
   `wg0`'s MTU be set to?

**Incident 2 — new client never connects**

```bash
$ sudo wg show wg0
interface: wg0
  public key: Hp2...=
  listening port: 51820

peer: Lr7...=
  endpoint: 198.51.100.40:51820
  allowed ips: 10.200.0.0/24

$ # no "latest handshake" line, and nothing received
$ sudo nft list ruleset | grep -E 'policy|dport'
policy drop
tcp dport 22 accept
```

(`wg show` on the client side reads `transfer: 0 B received, 592 B sent`.)

Tasks:
1. Which layer is failing, and what in the client-side output confirms
   handshakes are being attempted but not answered?
2. Is `AllowedIPs` the problem? Justify your answer.
3. What is the fix, and why is the rule's absence easy to miss given that
   `ss` shows the interface is listening?

<details>
<summary>Answers</summary>

**Incident 1:** This is the MTU layer. The recent handshake (21 seconds ago)
with traffic in both directions rules out reachability and keys, and the
fact that `ping` succeeds rules out routing and `AllowedIPs`. The decisive
evidence is the pair of `-M do` results: a payload of 1400 fails while 1272
succeeds. `ping` works and SSH freezes because small packets fit under the
real path MTU, while an SSH session printing a long output sends
full-size packets that exceed it and are silently dropped. `wg0` is at the
default-for-a-1500-link value of 1500, far larger than the path supports.
The inner limit lies between 1272+28 = 1300 and 1400+28 = 1428; bisect
with further `-M do` probes to find the exact ceiling, then set `wg0`'s
MTU to that value (or a conservative amount below it) on both ends, for
example `ip link set dev wg0 mtu 1380` once confirmed. Making the setting
persistent means adding `MTU = ...` to the `[Interface]` section of the
`wg-quick` config.

**Incident 2:** This is the reachability layer. The client-side
`transfer: 0 B received, 592 B sent` shows it is sending handshake
initiations (a few hundred bytes) and getting nothing back, and the server
shows no handshake at all. `AllowedIPs` is not the problem: it only
affects traffic after a handshake succeeds, and no handshake has
completed. The server's default-drop firewall (`policy drop`) only allows
`tcp dport 22`, so the client's UDP packets to port 51820 are discarded
before WireGuard ever sees them. The fix is `nft add rule inet filter
input udp dport 51820 accept` (and the same in the persistent ruleset).
It is easy to miss because WireGuard never answers unknown or unreachable
peers, so there is no "connection refused" message, and `ss -ulpn` shows
the port listening, which only proves the socket exists, not that packets
can reach it. This is the same lesson as Demo M03-02: "listening" and
"reachable" are separate facts.

</details>

---

## Interview Prep

**Q1. A WireGuard tunnel is up and `ping` across it works, but users report that large downloads hang. What do you check and why?**

This is the signature of an MTU black hole. I would confirm the handshake
is recent (so the keys and reachability are fine), then probe with `ping
-M do -s N` to find the largest payload that gets a reply. If small
packets work and larger ones vanish, the tunnel MTU is bigger than what the
path can carry once encapsulation overhead is added. I would lower the
`wg0` MTU on both ends to match the measured limit, and on a forwarding
gateway I would also consider clamping TCP MSS so clients I don't control
negotiate a size that fits.

---

**Q2. What does `AllowedIPs` do in WireGuard, and why is it different from a normal routing table entry?**

It does two jobs. Outbound, it selects which peer receives a packet based
on the destination address, which is why it behaves like a routing table.
Inbound, it filters: a decrypted packet is accepted only if its source
address is within that peer's `AllowedIPs`. It is separate from the kernel
routing table, which still decides that traffic should enter `wg0`
in the first place. If the routes send traffic into `wg0` but no peer's
`AllowedIPs` covers the destination, the sender gets `Required key not
available`.

---

**Q3. When would you choose OpenVPN over WireGuard, and when the reverse?**

WireGuard when I control both ends and want a small, fast, simple
configuration, such as site-to-site links or server-to-server access.
OpenVPN when the requirement is something WireGuard's static-key model
doesn't offer: certificate-based per-user identity and revocation across a
large fleet, username/password or plugin-based authentication, pushed
per-client routes, or TCP on port 443 to traverse restrictive networks.
The operational basics (routing, firewall, MTU) are the same either way.

---

## Key Takeaways

1. **A VPN tunnel is an interface, routes, and encapsulation.** Nothing
   about it is exempt from the routing and firewall rules from earlier
   demos; full versus split tunnel is just which destinations the routes
   send into the tunnel interface.
2. **`AllowedIPs` is routing and a source filter in one.** Getting it
   wrong produces a local `Required key not available` error or silently
   dropped inbound packets, not a network timeout.
3. **`wg show` identifies the failing layer.** No handshake means
   reachability or keys; a handshake but failed pings means routing or
   `AllowedIPs`; a handshake with only large packets failing means MTU.
   An idle peer with an old handshake is normal, not a fault.
4. **WireGuard adds 60 bytes (IPv4 underlay) or 80 (IPv6) per packet.**
   The default MTU of 1420 assumes the worst case on a 1500-byte link; any
   path with a smaller MTU needs a smaller tunnel MTU.
5. **An MTU black hole passes every small test.** `ping` works, handshakes
   succeed, and only full-size packets die. `ping -M do -s N` finds the
   real limit, and a local `message too long` error is the healthy,
   fixed outcome.
6. **Fix MTU at the tunnel first, clamp MSS at gateways second.** Setting
   the interface MTU lets TCP self-correct; MSS clamping covers clients
   you can't touch.
7. **WireGuard doesn't answer what it can't authenticate.** A firewall
   that blocks UDP 51820 produces no error at all, only a client that
   sends handshakes forever. Check the firewall before suspecting keys.

---

## Quick Commands Reference

### WireGuard

| Command | Purpose |
|---|---|
| `wg genkey \| tee priv.key \| wg pubkey > pub.key` | Generate a keypair |
| `wg set wg0 peer KEY allowed-ips CIDR endpoint IP:PORT` | Add or change a peer on a running interface |
| `wg show` | Show handshake time, transfer counters, peers |
| `wg-quick up wg0` / `down wg0` | Bring a config-file tunnel up or down |
| `systemctl enable --now wg-quick@wg0` | Start a tunnel at boot |

### Diagnosis

| Command | Purpose |
|---|---|
| `ping -M do -s N DEST` | Probe path MTU (packet size = N + 28) |
| `tracepath DEST` | Discover path MTU hop by hop |
| `ip link set dev wg0 mtu N` | Set tunnel MTU |
| `ip route get ADDR` | Confirm whether traffic enters the tunnel |
| `tcpdump -ni IFACE udp port 51820` | Capture encrypted traffic on the underlay |

### Namespaces (lab)

| Command | Purpose |
|---|---|
| `ip netns add NAME` / `del NAME` | Create or remove an isolated network stack |
| `ip netns exec NS CMD` | Run a command inside a namespace |
| `ip -n NS CMD` | Run an `ip` command inside a namespace |

---

## Appendix A — Anki Cards

**`m03-07-vpn-fundamentals-anki.csv`:**

```
#deck:Linux Mastery::Module 3 - Networking::Demo M03-07 - VPN Fundamentals
#separator:Comma
#columns:Front,Back,Tags
"Name the three ordinary Linux components that together make up a VPN tunnel.","A virtual network interface (e.g. wg0), routes that send chosen destinations into it, and encapsulation/encryption of packets sent to the peer.","demo13c,vpn,fundamentals"
"What is the difference between a full tunnel and a split tunnel?","Full tunnel sends all client traffic through the VPN (default route via the tunnel); split tunnel sends only specific destination networks through it. Both are expressed purely through routing.","demo13c,vpn,routing"
"Why must the gateways in a site-to-site tunnel have net.ipv4.ip_forward=1?","Hosts behind each gateway send their traffic to the gateway, which must forward packets between its LAN and the tunnel interface rather than only processing packets addressed to itself.","demo13c,vpn,forwarding"
"In WireGuard, who needs whose key, and how are public keys exchanged?","Each side needs its own private key and the other side's public key. There is no CA; public keys are copied between hosts out of band.","demo13c,wireguard,keys"
"What file mode should /etc/wireguard/wg0.conf have, and why?","600, owned by root, because it contains the interface's private key, which is a credential and must not be readable or committed to version control.","demo13c,wireguard,security"
"A WireGuard peer shows an old latest handshake but has had no traffic for an hour. Is that a fault?","No. WireGuard handshakes on demand and re-keys about every two minutes only while traffic flows, so an idle peer's stale handshake is normal.","demo13c,wireguard,troubleshooting"
"Why does WireGuard connectivity fail silently when a firewall blocks UDP 51820?","WireGuard never answers unauthenticated or unreachable packets, so there is no refusal message; the client just sends handshake attempts forever. ss showing the port listening only proves the socket exists.","demo13c,wireguard,firewall"
"What is the per-packet overhead of WireGuard over an IPv4 underlay, and how is it made up?","60 bytes: 20 (outer IPv4) + 8 (UDP) + 32 (WireGuard header and authentication tag). It is 80 bytes over IPv6.","demo13c,mtu,overhead"
"What does ping -M do -s 1372 test, and what size is the packet on the wire?","It sends an ICMP echo with the Don't Fragment flag set; the on-wire packet is payload plus 28 bytes (20 IP + 8 ICMP), so 1400 bytes here.","demo13c,mtu,ping"
"A gateway forwards traffic for clients you cannot reconfigure and the tunnel has a lower-than-default path MTU. What can you do at the gateway?","Clamp TCP MSS so handshakes advertise a segment size that fits, e.g. an nftables rule using tcp option maxseg size set rt mtu on SYN packets in the forward chain.","demo13c,mtu,mss"
"Which of these protocols can OpenVPN use as transport that WireGuard cannot?","TCP (including TCP/443). WireGuard is UDP only.","demo13c,openvpn,comparison"
"On an AllowedIPs = 0.0.0.0/0 config, what does wg-quick add so the tunnel's own UDP packets do not loop back into the tunnel?","Policy-routing rules using a firewall mark, so packets carrying the mark use the normal routing table. This is why multiple routing tables matter (Demo M03-04 section 1.2).","demo13c,wireguard,policy-routing"
"How do you make a wg-quick tunnel start at boot?","sudo systemctl enable --now wg-quick@wg0, with the configuration in /etc/wireguard/wg0.conf.","demo13c,wireguard,systemd"
```

---

## Appendix B — Quiz

**`m03-07-vpn-fundamentals-quiz.md`:**

````markdown
# Quiz — Demo M03-07: VPN Fundamentals

> One correct answer per question unless stated otherwise.
> Target: 80% or above before moving to Demo M03-08.

---

**Q1.** `ping 10.200.0.2` from a WireGuard host returns `sendmsg: Required key not available`. `ip route get 10.200.0.2` shows the route goes via `wg0`. What is the most likely cause?

- A) The remote host is offline
- B) A firewall is dropping UDP 51820
- C) No configured peer has an `AllowedIPs` entry covering 10.200.0.2
- D) The tunnel MTU is too large

<details>
<summary>Answer</summary>

**C** — The kernel route correctly sends traffic into `wg0`, but WireGuard
selects a peer by matching the destination against each peer's `AllowedIPs`.
With no match it has no key to encrypt with and fails locally with that
error.

Trap: A and B would produce a timeout or silent loss, not an immediate
local error. D produces silent loss of large packets, not this message.

</details>

---

**Q2.** Why is WireGuard's default interface MTU 1420 rather than 1500?

- A) Encapsulation adds overhead, and 1420 leaves room for the worst case (IPv6 underlay, 80 bytes) on a 1500-byte link
- B) 1420 is the maximum size the WireGuard protocol supports
- C) Linux caps virtual interfaces at 1420
- D) 1420 is chosen to match typical TCP window sizes

<details>
<summary>Answer</summary>

**A** — 1500 minus 80 bytes of overhead (40 outer IPv6 + 8 UDP + 32
WireGuard) is 1420. It is a safe default that assumes the worst-case
underlay.

Trap: B, C, and D invent limits that do not exist.

</details>

---

**Q3.** A tunnel passes `ping`, and SSH logs in normally, but the session freezes when a command prints a screenful of output. The handshake is recent. What is the most likely problem?

- A) Wrong public key
- B) `AllowedIPs` too narrow
- C) The tunnel MTU is larger than the path supports, so full-size packets are silently dropped
- D) UDP 51820 is blocked

<details>
<summary>Answer</summary>

**C** — Small packets (login, keystrokes) fit; large output produces
full-size packets that exceed the real path MTU and are dropped without
notice. A recent handshake rules out keys and firewalling (A, D), and
working pings rule out `AllowedIPs` (B).

</details>

---

**Q4.** `wg show` lists a peer with `transfer: 0 B received, 1.2 KiB sent` and no `latest handshake`. What should you check first?

- A) Whether UDP to the peer's endpoint is permitted by every firewall on the path, and that the endpoint and keys are correct
- B) The tunnel MTU
- C) The peer's `AllowedIPs`
- D) `net.ipv4.ip_forward`

<details>
<summary>Answer</summary>

**A** — Handshake initiations are being sent but nothing comes back, so
the failure is below routing: reachability, endpoint, firewall, or keys.
`AllowedIPs` (C), MTU (B), and forwarding (D) only matter after a
handshake succeeds.

</details>

---

**Q5.** Which statement about WireGuard versus OpenVPN is accurate?

- A) Both negotiate cipher suites at connection time
- B) WireGuard uses a fixed set of cryptographic primitives with static public keys; OpenVPN typically uses TLS with certificates and has configurable ciphers
- C) WireGuard can run over TCP/443 to bypass restrictive firewalls
- D) OpenVPN runs in the kernel and WireGuard in user space

<details>
<summary>Answer</summary>

**B** — That is a core design difference. A is wrong because WireGuard
does not negotiate ciphers. C is wrong because WireGuard is UDP only. D
reverses the actual architecture.

</details>

---

**Q6.** After setting `wg0` MTU to 1320, `ping -M do -s 1372 10.200.0.2` prints `local error: message too long, mtu=1320`. What does that tell you?

- A) The path is dropping packets silently
- B) The packet exceeds `wg0`'s own MTU and is rejected locally, which is the correct fast failure
- C) The peer is unreachable
- D) The tunnel is misconfigured and must be rebuilt

<details>
<summary>Answer</summary>

**B** — Rejecting an oversized DF packet at the local interface is the
healthy behavior; it replaces silent loss with an explicit message and
reveals the configured MTU. The sender should use a smaller packet or
allow fragmentation.

</details>

---

**Q7.** What does `AllowedIPs = 0.0.0.0/0` on a client's peer entry accomplish?

- A) It makes that peer able to carry traffic for any destination, which is how a full tunnel is expressed
- B) It disables encryption for that peer
- C) It allows any host to connect to the client
- D) It limits the tunnel to the local subnet

<details>
<summary>Answer</summary>

**A** — `0.0.0.0/0` means "every IPv4 destination goes to this peer", and
tools like `wg-quick` then route the default route through the tunnel.

Trap: B, C, and D invent behaviors; `AllowedIPs` does not change
encryption or who may initiate connections.

</details>

---

**Q8.** A WireGuard client behind a NAT router is reachable from the server right after connecting, but after several idle minutes the server can no longer send to it. What is the usual fix?

- A) Increase the MTU
- B) Set `PersistentKeepalive = 25` on the NATed client so it sends periodic packets that keep the NAT mapping open
- C) Switch the tunnel to TCP
- D) Add the server's public key twice

<details>
<summary>Answer</summary>

**B** — NAT devices expire idle UDP mappings, and WireGuard sends nothing
while idle. A periodic keepalive from the NATed side keeps the mapping
alive. WireGuard has no TCP mode (C), and A and D are unrelated.

</details>

---

| Score | Action |
|---|---|
| 7/8 or 8/8 | Import Anki cards, move to Demo M03-08 |
| 6/8 | Review the wrong answers, then proceed |
| 5/8 | Re-read the relevant sections, retry those questions |
| Below 5/8 | Re-read the full demo and redo the walkthrough before proceeding |
````