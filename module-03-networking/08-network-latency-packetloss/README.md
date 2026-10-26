# Demo M03-08 — Network Latency & Packet Loss Diagnosis

## Overview

A service behind a load balancer reported as "intermittently timing out"
could mean a dozen different things — this demo covers the specific tools
for the network-layer half of that investigation: injecting realistic
latency and packet loss on demand so you can test a service's actual
behavior under degraded conditions, then reading socket-level, packet-
level, and per-hop evidence to tell apart a local NIC problem, a flaky
upstream link, and a noisy-neighbor hypervisor stealing network capacity
the same way Module 7 covered CPU steal time.

## Demo Objectives

1. Simulate realistic latency, jitter, and packet loss with `tc netem`,
   to test a service's real behavior under degraded network conditions
2. Read `ss -ti` for retransmits and RTT on active sockets as a fast
   first signal of network trouble
3. Capture and read a `tcpdump` trace to distinguish delay from an
   actual dropped packet at the TCP handshake level
4. Use `mtr` for per-hop loss/latency to isolate whether a problem is
   local, at the load balancer, or further upstream
5. Rule out simulated loss versus a real NIC-level issue using
   interface error/drop counters
6. Break down exactly where request latency is being spent using
   `curl`'s built-in timing instrumentation

## Directory Structure

```
08-network-latency-packetloss/
├── README.md                                   # this file
├── m03-08-network-latency-packetloss-anki.csv  # Anki flashcard deck
└── m03-08-network-latency-packetloss-quiz.md   # standalone quiz
```

## Environments

| Task | Ubuntu 24.04 | Rocky Linux 9.8 | EC2 Amazon Linux |
|---|---|---|---|
| `tc`/`ip`/`ss` | `sudo apt install iproute2` (usually preinstalled) | `sudo dnf install iproute` (usually preinstalled) | Not required |
| `tcpdump` | `sudo apt install tcpdump` | `sudo dnf install tcpdump` | Not required |
| `mtr` | `sudo apt install mtr-tiny` | `sudo dnf install mtr` | Not required |

> **Verification note:** this sandbox has almost no network tooling
> installed at all — `tc`, `ip`, `ss`, `tcpdump`, `mtr`, and `ping` are
> all absent, and none could be installed (no network access for package
> downloads). Nearly this entire demo is therefore doc-sourced from
> stable, well-documented tool behavior and marked
> `[Unverified — test before use]` throughout — recommend a thorough
> verification pass on an actual Ubuntu/Rocky9 machine before relying on
> exact output formats. **One piece was genuinely live-tested**: `curl`
> itself is available here, and §1.6's timing-breakdown output
> (`time_namelookup`, `time_connect`, etc.) is real, captured output from
> an actual request in this sandbox — not a documentation paraphrase.

## Recall Check — 06-dns-resolution

Answer these from memory before continuing — no looking back at Demo
M03-06.

**Q1.** An application fails on the first connection to a hostname,
then succeeds on every later one. What does this pattern suggest?

**Q2.** Why isn't a config file's content proof of what's actually
happening with DNS resolution?

**Q3.** `dig` reports a 12ms query time, but the application experienced
a 750ms connection delay. Is DNS the cause?

<details>
<summary>Answers</summary>

**A1.** DNS resolution — a cold lookup that then gets cached, so later
connections skip the slow or failing step.

**A2.** `systemd-resolved`, NetworkManager, or network-level interception
can all make the file's content irrelevant to what's actually being
queried.

**A3.** No — 12ms cannot account for 750ms. The DNS-latency hypothesis
should be ruled out by measurement, and the delay is more likely
elsewhere (TCP/TLS).

</details>

## Part 1 — Concepts

### 1.1 tc netem — injecting realistic degradation on demand

```bash
sudo tc qdisc add dev eth0 root netem delay 150ms 50ms loss 5%
```

`[Unverified — test before use]` This adds a **queueing discipline**
(`qdisc`) to interface `eth0` that delays every packet by 150ms, jittered
±50ms, and randomly drops 5% of packets — simulating a flaky upstream
link or a hypervisor-level network problem **without needing an actual
flaky network to test against**. This is the practical value: you can
reproduce "the service times out intermittently" on a healthy network, on
demand, and confirm a fix (retry/backoff logic, a timeout setting)
actually helps before a real degraded network ever occurs.

```bash
ping -c 20 <target>
curl -w "@curl-format.txt" -o /dev/null -s https://<service>/health
```

`ping`'s own summary line reports loss percentage and RTT directly,
confirming the injected conditions are actually in effect before
drawing any conclusions from a service's behavior under them.

**Always remove the test qdisc afterward**:

```bash
sudo tc qdisc del dev eth0 root netem
```

Leaving injected latency/loss in place after the experiment is a
realistic, easy-to-make mistake — the interface keeps behaving
"degraded" for every subsequent use until explicitly removed.

### 1.2 ss -ti — the fast first signal

```bash
ss -ti
```

`[Unverified — test before use]` `-t` restricts to TCP sockets, `-i`
shows internal TCP information per socket — specifically the `retrans`
field (retransmission count and rate) and `rtt` (round-trip time,
mean/variance). **A rising retransmit count on an active connection is
one of the fastest available signals that something is wrong at the
network layer**, visible before request-level error rates necessarily
climb — a connection can retransmit several times and still eventually
succeed, meaning the application layer sees "slow" rather than "failed,"
while `ss -ti` already shows the underlying network-level struggle
directly.

### 1.3 tcpdump — delay versus drop at the handshake level

```bash
sudo tcpdump -i eth0 -w capture.pcap host <target>
```

`[Unverified — test before use]` Capture traffic to and from a specific
host during an actual failure window, writing to a file (`-w`) rather
than printing live, so the capture can be reviewed afterward without
needing to watch it happen in real time.

```bash
tcpdump -r capture.pcap -nn 'tcp[tcpflags] & (tcp-syn|tcp-ack) != 0'
```

Reading the capture back (`-r`), filtered to just SYN/SYN-ACK packets —
the TCP handshake specifically — isolates **delay from drop**: a SYN
followed by a SYN-ACK after an unusually long gap is delay; a SYN with no
SYN-ACK at all (followed by a retransmitted SYN) is an actual drop. These
produce similar-looking symptoms from the application's perspective
("the connection was slow to establish") but have different underlying
causes and different fixes.

### 1.4 mtr — isolating which hop

```bash
mtr -rwc 50 <target>
```

`[Unverified — test before use]` `-r` (report mode, non-interactive
output suitable for logging), `-w` (wide, showing full hostnames), `-c
50` (50 ping cycles per hop, for a more statistically meaningful loss
percentage than a single probe). `mtr` combines `traceroute`'s per-hop
path discovery with `ping`'s repeated-probe loss/latency measurement,
showing loss and latency **at every hop along the path**, not just the
final destination — which is the direct way to tell whether a problem is
local (the first hop or two), at the load balancer itself, or further
upstream, entirely outside your own infrastructure's control.

### 1.5 Ruling out simulated loss vs. a real NIC problem

```bash
ip -s link show eth0
```

`[Unverified — test before use]` Shows interface-level RX/TX statistics
including drop and error counters. **This is the cross-check specifically
against §1.1's deliberate `netem` injection**: if you're troubleshooting
a real incident and considering whether packet loss is a `netem`
leftover from a previous test (§1.1's "always remove the test qdisc"
point) versus an actual NIC/driver/cabling-level problem, the interface's
own error/drop counters distinguish them — `netem`-induced loss doesn't
show up as an interface-level error the way a genuine NIC problem does.

**If loss is confirmed at the network layer but the interface itself is
clean**, the next step is escalating to hypervisor or cloud-provider
network metrics — the same escalation pattern as Module 7's CPU steal
time: a VM's own interface can be healthy while the underlying physical
network path is actually degraded by something outside the guest's
visibility entirely.

### 1.6 curl's built-in timing breakdown

```bash
curl -w "@curl-format.txt" -o /dev/null -s https://example.com
```

With `curl-format.txt` containing:

```
    time_namelookup:  %{time_namelookup}s\n
       time_connect:  %{time_connect}s\n
    time_appconnect:  %{time_appconnect}s\n
   time_pretransfer:  %{time_pretransfer}s\n
      time_redirect:  %{time_redirect}s\n
 time_starttransfer:  %{time_starttransfer}s\n
                    ----------\n
         time_total:  %{time_total}s\n
```

Real output, from an actual request in this sandbox:

```
    time_namelookup:  0.017764s
       time_connect:  0.018137s
    time_appconnect:  0.061022s
   time_pretransfer:  0.061262s
      time_redirect:  0.000000s
 time_starttransfer:  0.062345s
                    ----------
         time_total:  0.062393s
```

**Each field is cumulative, so the gap *between* consecutive fields is
that specific phase's actual duration** — here, `time_namelookup`
(~18ms) is DNS; `time_connect - time_namelookup` (~0.4ms) is the TCP
handshake; `time_appconnect - time_connect` (~43ms) is TLS negotiation —
by far the largest single phase in this particular request. This is the
direct way to answer "where exactly is the time going" for one specific
request, cross-referencing cleanly against Demo M03-06's DNS-latency
measurement (`time_namelookup` specifically) and Demo M08-03's TLS-chain
material (`time_appconnect`) — the same total-latency-breakdown
discipline, scoped to whichever phase a specific incident actually
points at.

## Part 2 — Hands-On Lab

### Commands used in this demo

| Command | Purpose |
|---|---|
| `tc qdisc add dev IFACE root netem delay Xms loss Y%` | Inject latency/jitter/loss on an interface |
| `tc qdisc del dev IFACE root netem` | Remove the injected condition |
| `ss -ti` | Per-socket retransmit count and RTT |
| `tcpdump -i IFACE -w FILE host TARGET` | Capture traffic to/from a host |
| `tcpdump -r FILE -nn 'tcp[tcpflags] & (tcp-syn\|tcp-ack) != 0'` | Read back just the handshake packets |
| `mtr -rwc 50 TARGET` | Per-hop loss/latency report |
| `ip -s link show IFACE` | Interface-level error/drop counters |
| `curl -w "@FILE" -o /dev/null -s URL` | Per-phase request timing breakdown |

### Task 1 — Inject and confirm degraded conditions

```bash
sudo tc qdisc add dev eth0 root netem delay 150ms 50ms loss 5%
ping -c 20 8.8.8.8
```

Confirm `ping`'s summary line shows loss near 5% and RTT in the expected
150ms ± 50ms range.

### Task 2 — Check socket-level retransmits

```bash
ss -ti
```

While the `netem` condition from Task 1 is still active, identify the
`retrans` and `rtt` fields on any active connection.

### Task 3 — Capture and read the handshake

```bash
sudo tcpdump -i eth0 -w /tmp/capture.pcap host 8.8.8.8 &
curl -s -o /dev/null https://8.8.8.8 --max-time 5
sleep 1
sudo kill %1
tcpdump -r /tmp/capture.pcap -nn 'tcp[tcpflags] & (tcp-syn|tcp-ack) != 0'
```

Identify whether the SYN/SYN-ACK exchange shows delay, a retransmitted
SYN (drop), or both.

### Task 4 — Per-hop isolation

```bash
mtr -rwc 50 8.8.8.8
```

Identify which hop, if any, shows meaningfully higher loss or latency
than the hops before and after it.

### Task 5 — Rule out simulated vs. real loss

```bash
ip -s link show eth0
```

Confirm the interface's own error/drop counters do **not** reflect the
loss injected in Task 1 — `netem`-induced loss happens at the queueing
layer, not as an interface-level error.

```bash
sudo tc qdisc del dev eth0 root netem
```

Clean up before moving on.

### Task 6 — Request-level timing breakdown

```bash
cat > /tmp/curl-format.txt << 'EOF'
    time_namelookup:  %{time_namelookup}s\n
       time_connect:  %{time_connect}s\n
    time_appconnect:  %{time_appconnect}s\n
   time_pretransfer:  %{time_pretransfer}s\n
      time_redirect:  %{time_redirect}s\n
 time_starttransfer:  %{time_starttransfer}s\n
                    ----------\n
         time_total:  %{time_total}s\n
EOF
curl -w "@/tmp/curl-format.txt" -o /dev/null -s https://example.com
```

Identify which single phase accounts for the largest share of
`time_total`.

## What You Learned

- ✅ Simulated realistic latency, jitter, and packet loss with `tc
  netem` to test real service behavior under degraded conditions
- ✅ Read `ss -ti` for retransmits and RTT as a fast first network-layer
  signal
- ✅ Captured and read a `tcpdump` trace to distinguish delay from an
  actual dropped packet
- ✅ Used `mtr` to isolate whether a problem is local, at the load
  balancer, or further upstream
- ✅ Ruled out simulated loss versus a real NIC issue using interface
  error/drop counters
- ✅ Broke down exactly where request latency is spent with `curl`'s
  timing instrumentation

## Part 3 — Break-Fix Scenario

### Scenario — NovaTech Ops: the intermittent LB timeout

> **Incident report — NovaTech Ops, ticket #5440**
>
> "A service behind the load balancer times out roughly 1 in 20
> requests. `ping` to the service's host shows no loss at all. The
> on-call engineer is ready to close the ticket as 'network is fine.'"

```bash
$ ss -ti | grep retrans
    retrans:3/12 rtt:187.4/42.1
$ ip -s link show eth0
    RX: errors dropped  overrun mcast
         0       0        0      0
```

Tasks:
1. Reference §1.2/§1.5 — does a clean `ping` test actually rule out a
   network-layer problem here? What does the `ss -ti` output above
   suggest instead?
2. The interface counters show zero drops. Does this mean the
   retransmits aren't real, or does it point somewhere else per §1.5?
3. What's the next diagnostic step, and what would it specifically tell
   you that hasn't been confirmed yet?

<details>
<summary>Answer</summary>

A clean `ping` test does not rule out a network-layer problem — `ping`
is a simple, lightweight ICMP probe that can succeed cleanly even while
the actual TCP connections the load-balanced service uses experience
real retransmits, since TCP traffic and ICMP traffic can be treated
differently by an intermediate device, or the loss can be
transient/load-dependent in a way an occasional `ping` simply doesn't
happen to hit. The `ss -ti` output (`retrans:3/12`, meaning 3
retransmissions out of 12 segments on that connection) directly
contradicts "network is fine" and should not be dismissed just because
`ping` looked clean. Zero interface-level errors/drops doesn't mean the
retransmits aren't real — per §1.5, it specifically rules out a local
NIC/driver-level problem on *this* host, pointing the investigation
instead toward something further along the path: the load balancer
itself, or upstream, which is exactly what `mtr -rwc 50` (§1.4) would
confirm next — showing per-hop loss that would reveal whether the
problem is at the LB specifically or somewhere beyond it, information
neither `ping` nor the local interface counters can provide.

</details>

## Interview Prep

**Q1. Why can `ping` show zero loss while a service experiences real, measurable network problems?**
`ping` is a simple ICMP probe, occasional and lightweight — it can miss
transient or load-dependent loss entirely, and some network paths treat
ICMP differently from the actual TCP traffic a service uses. `ss -ti`'s
retransmit count on the actual active TCP connections is a more direct
signal than a separate, simpler protocol's test.

**Q2. A capture shows a SYN packet with no SYN-ACK response, followed by a retransmitted SYN. Is this delay or drop?**
Drop — a SYN followed eventually by a SYN-ACK, just later than expected,
would be delay; a SYN with no response at all, followed by the client
retransmitting the SYN itself, means the original packet (or its
response) was actually lost, not just slow.

**Q3. Interface-level error/drop counters show zero, but `ss -ti` shows real retransmits. What does this combination tell you, and what's the next step?**
It rules out a local NIC/driver-level problem on this specific host as
the cause, while confirming the retransmits themselves are real — the
problem is further along the path. `mtr`'s per-hop report is the next
step, since it can show loss/latency at the load balancer or further
upstream, which neither a clean local interface nor a simple `ping` test
can reveal.

**Q4. Why remove a `tc netem` qdisc immediately after a deliberate test, rather than leaving it in place?**
The interface continues behaving as artificially degraded for every
subsequent use until the qdisc is explicitly removed — leaving it in
place turns a deliberate, temporary test condition into an ongoing,
easy-to-forget-about problem affecting real traffic afterward.

---

## Key Takeaways

1. **`tc netem` lets you reproduce realistic network degradation on
   demand**, testing a service's actual behavior and confirming a fix
   helps, without waiting for a real flaky network to occur — always
   remove the test qdisc afterward.
2. **`ss -ti`'s retransmit count is a faster, more direct signal than
   `ping`** — a connection can retransmit several times and still
   eventually succeed, meaning the application only sees "slow," while
   `ss -ti` shows the underlying struggle directly.
3. **A `tcpdump` handshake capture distinguishes delay from an actual
   drop** — a late SYN-ACK is delay; a retransmitted SYN with no original
   response is a genuine drop, and the two need different fixes despite
   looking similar from the application's side.
4. **`mtr` shows loss/latency at every hop**, isolating whether a
   problem is local, at the load balancer, or further upstream — entirely
   outside your own infrastructure's visibility.
5. **Interface-level error/drop counters distinguish simulated
   (`netem`) loss from a real NIC problem** — and when loss is confirmed
   at the network layer with a clean interface, escalate to
   hypervisor/cloud-provider metrics, the same pattern as Module 7's CPU
   steal time.
6. **`curl`'s timing breakdown shows exactly which phase of a request
   actually took the time** — DNS, TCP connect, TLS negotiation, or
   transfer — cross-referencing directly against Demo M03-06's DNS timing
   and Demo M08-03's TLS-chain material.

> **Demo scope:** Primary concept: diagnosing network-layer latency and
> packet loss end to end. Supporting concepts: `tc netem` injection,
> socket-level retransmit signals, handshake-level packet capture,
> per-hop path isolation, simulated-vs-real loss, request timing
> breakdown.
> Estimated completion time: 38 minutes (reading + hands-on + verification).
> Checkpoints: 6 natural stopping points (after Task 1 through Task 6).

## Quick Commands Reference

| Command | Purpose |
|---|---|
| `tc qdisc add dev IFACE root netem delay Xms loss Y%` | Inject degradation |
| `tc qdisc del dev IFACE root netem` | Remove it |
| `ss -ti` | Per-socket retransmits and RTT |
| `tcpdump -i IFACE -w FILE host TARGET` | Capture traffic |
| `tcpdump -r FILE -nn 'tcp[tcpflags] & (tcp-syn\|tcp-ack) != 0'` | Read back just the handshake |
| `mtr -rwc 50 TARGET` | Per-hop loss/latency |
| `ip -s link show IFACE` | Interface error/drop counters |
| `curl -w "@FILE" -o /dev/null -s URL` | Per-phase request timing |

## Appendix A — Anki Cards

**`m03-08-network-latency-packetloss-anki.csv`:**

```
#deck:Linux Mastery::Module 3 - Networking::Demo M03-08 - Network Latency & Packet Loss Diagnosis
#separator:Comma
#columns:Front,Back,Tags
"What does tc qdisc add dev eth0 root netem delay 150ms 50ms loss 5% actually do?","Injects 150ms +/-50ms jittered delay and 5% random packet loss on eth0 — simulating a flaky link or noisy-neighbor hypervisor on demand, without needing a real degraded network to test against.","demo-m03-08,netem,fundamentals"
"Why can ping show zero loss while a service still experiences real network problems?","ping is a simple, occasional ICMP probe that can miss transient/load-dependent loss, and some paths treat ICMP differently from actual TCP traffic. ss -ti's retransmit count on real active connections is a more direct signal.","demo-m03-08,ping,ss,common-mistake"
"What does ss -ti's retrans field show, and why is it a faster signal than request-level error rate?","Retransmission count/rate on active TCP sockets. A connection can retransmit several times and still eventually succeed, meaning the application only sees 'slow' — ss -ti shows the underlying network-level struggle directly, before error rates necessarily climb.","demo-m03-08,ss,retransmits,fundamentals"
"How does a tcpdump handshake capture distinguish delay from an actual packet drop?","A SYN followed by a SYN-ACK after an unusually long gap is delay. A SYN with no SYN-ACK at all, followed by a retransmitted SYN, is an actual drop — similar application-level symptoms, different underlying cause and fix.","demo-m03-08,tcpdump,delay-vs-drop"
"What does mtr show that a single ping or traceroute doesn't?","Loss and latency at EVERY hop along the path, combining traceroute's path discovery with ping's repeated-probe measurement — isolating whether a problem is local, at a load balancer, or further upstream.","demo-m03-08,mtr,fundamentals"
"Interface error/drop counters (ip -s link show) are zero despite confirmed network-layer retransmits. What does this tell you?","It rules out a local NIC/driver-level problem on this specific host — netem-induced loss doesn't show as an interface error either. The problem is further along the path; escalate with mtr or to hypervisor/cloud metrics.","demo-m03-08,interface-counters,diagnosis"
"Why is 'loss confirmed but the interface is clean' escalated the same way as Module 7's CPU steal time?","Both represent a resource/path problem outside what's visible from inside the guest/host itself — a VM's own interface or CPU can be healthy while the underlying physical/hypervisor layer is actually degraded, requiring hypervisor/provider-level visibility to confirm.","demo-m03-08,steal-time,cross-demo"
"In curl's timing breakdown, how do you find the duration of a SPECIFIC phase (e.g. TLS negotiation) rather than just the cumulative total?","Each %{time_X} field is cumulative from request start — subtract the PRECEDING field from it. TLS negotiation duration = time_appconnect minus time_connect.","demo-m03-08,curl,timing,fundamentals"
```

## Appendix B — Quiz

**`m03-08-network-latency-packetloss-quiz.md`:**

````markdown
# Quiz — Demo M03-08: Network Latency & Packet Loss Diagnosis

> One correct answer per question unless stated otherwise.
> Target: 80% or above before moving to Demo M05-03.

**Q1. What does `tc qdisc add dev eth0 root netem delay 150ms loss 5%` accomplish?**

- A) It permanently fixes network latency on the interface
- B) It injects artificial delay and packet loss on the interface, for testing how a service behaves under degraded conditions
- C) It blocks all traffic on the interface entirely
- D) It only affects traffic to a single specified IP address

<details>
<summary>Answer</summary>

**B** — This lets you reproduce degraded conditions on demand for
testing, without needing a real flaky network.

</details>

---

**Q2. `ping` to a service shows 0% loss, but the service itself is intermittently timing out. What does this suggest?**

- A) The service's problem has nothing to do with the network at all
- B) `ping`'s simple ICMP probe can miss transient or load-dependent loss that affects actual TCP connections — check `ss -ti` for real retransmit evidence instead
- C) `ping` is unreliable and should never be used for any network diagnosis
- D) This combination is impossible; one of the two measurements must be wrong

<details>
<summary>Answer</summary>

**B** — `ping` and the actual TCP traffic can experience genuinely
different conditions; `ss -ti` checks the real connections directly.

</details>

---

**Q3. A `tcpdump` capture shows a SYN packet with no SYN-ACK response, followed by the client retransmitting the SYN. Is this delay or an actual drop?**

- A) Delay — the SYN-ACK is just running late
- B) An actual drop — no response at all to the original SYN means it (or its response) was genuinely lost
- C) This pattern always indicates a firewall actively blocking the connection
- D) This is normal TCP behavior and not indicative of any problem

<details>
<summary>Answer</summary>

**B** — A completely absent response, followed by retransmission, is the
signature of a genuine drop rather than mere delay.

</details>

---

**Q4. What does `mtr` provide that a single `ping` to the final destination does not?**

- A) Faster overall results
- B) Loss and latency measured at every hop along the path, isolating whether a problem is local, at an intermediate point, or far upstream
- C) The ability to bypass firewalls that block `ping`
- D) Automatic repair of any detected network issues

<details>
<summary>Answer</summary>

**B** — Per-hop visibility is `mtr`'s specific value over a single
end-to-end `ping`.

</details>

---

**Q5. Network-layer retransmits are confirmed via `ss -ti`, but `ip -s link show` reports zero interface errors/drops. What does this combination indicate?**

- A) The retransmits reported by `ss -ti` must not be real
- B) A local NIC/driver-level problem on this host is ruled out; the issue is likely further along the path and should be investigated with `mtr` or escalated to hypervisor/provider metrics
- C) The network interface needs to be physically replaced immediately
- D) This combination is a contradiction and indicates a monitoring bug

<details>
<summary>Answer</summary>

**B** — `netem`-induced or genuinely upstream loss doesn't register as a
local interface error — clean local counters point the investigation
elsewhere, not toward dismissing the retransmit evidence.

</details>

---

**Q6. In `curl`'s timing output, how do you determine how long TLS negotiation specifically took?**

- A) `time_appconnect` is already the TLS-only duration
- B) Subtract `time_connect` from `time_appconnect` — each field is cumulative from request start, so the gap between consecutive fields is that phase's actual duration
- C) TLS timing cannot be isolated from `curl`'s output at all
- D) Divide `time_total` by the number of redirects

<details>
<summary>Answer</summary>

**B** — The cumulative nature of these fields means subtracting
consecutive values is required to isolate one phase's actual duration.

</details>

---

**Q7. Why is "loss confirmed, but the interface is clean" escalated the same way as Module 7's CPU steal time?**

- A) They are literally the same metric measured differently
- B) Both indicate a problem outside what's visible from inside the guest/host itself — the underlying hypervisor or physical path can be degraded while local resource counters look healthy
- C) This is purely a coincidental naming similarity with no technical connection
- D) Neither issue can actually be diagnosed with any available tooling

<details>
<summary>Answer</summary>

**B** — Both point at the same category of blind spot: a resource or
path problem the guest/host itself cannot see or measure directly.

</details>

---

Score guide:

| Score | Action |
|---|---|
| 7/7 | Import Anki cards, move to Demo M05-03 |
| 6/7 | Review the wrong answer, then proceed |
| 5/7 | Re-read the relevant section, retry those questions |
| Below 5/7 | Re-read the full demo and redo the walkthrough before proceeding |
````