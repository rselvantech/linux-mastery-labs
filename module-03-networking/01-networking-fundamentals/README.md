# Demo M03-01 — Networking Fundamentals

## Overview

Every server needs to be reachable, needs to reach other things, and needs to
agree with the rest of the world on what time it is and what names mean. This
demo covers the foundational layer: reading and configuring IPv4 addresses
and routes with the modern `ip` command, recognizing IPv6 address types,
inspecting the neighbor (ARP) table, how hostname resolution actually works
before DNS even gets involved, why clock sync matters more than it seems, and
setting up SSH key authentication correctly. Demo M03-02 builds on this with
monitoring, firewalls, and troubleshooting — this demo is the "get it
configured and reachable" half.

---

## Demo Objectives

By the end of this demo you will be able to:

1. Read and configure IPv4 addresses with `ip addr`; read and identify IPv6
   address types (link-local vs global)
2. Read routes with `ip route`, identify the default gateway, and add and
   remove a static route
3. Inspect the neighbor (ARP) table with `ip neigh`, completing the `ip`
   replacement of the older `ifconfig`/`route`/`arp` trio
4. Explain the actual hostname resolution order: `/etc/hosts` before DNS
5. Query DNS with `dig`, and explain what `/etc/resolv.conf` controls —
   including how its management differs between Ubuntu and Rocky9
6. Configure and verify time synchronization with chrony or
   systemd-timesyncd, including enabling and disabling NTP sync
7. Set up SSH key-based authentication correctly, including common
   permission pitfalls

---

## Directory Structure

```
01-networking-fundamentals/
├── README.md                                # this file
├── m03-01-networking-fundamentals-anki.csv  # Anki flashcard deck
└── m03-01-networking-fundamentals-quiz.md   # standalone quiz
```

---

## Environments

| Task | Ubuntu 24.04 | Rocky Linux 9.8 | EC2 Amazon Linux |
|---|---|---|---|
| All lab tasks | ✅ | ✅ | Not required |
| Time sync daemon | `systemd-timesyncd` (default) | `chrony` (default) | Not required |
| DNS resolver | `systemd-resolved` (default) | NetworkManager-managed `/etc/resolv.conf` (default) | Not required |

---

## Recall Check

> Answer from memory before reading further. From Demo M02-08.

1. `iostat` shows a device at high `%util`. What does `iostat` alone NOT
   tell you, and which tool fills that gap?

2. What do `sa1` and `sa2` actually do, and how are they scheduled?

3. `HISTORY=7` is set in `/etc/sysstat/sysstat`. An incident from 10 days
   ago needs investigating. What happens to that data, and why does this
   matter?

<details>
<summary>Answers</summary>

1. `iostat` reports per-device statistics only — it doesn't identify
   which process is responsible for the load. `iotop` fills that gap by
   attributing I/O to individual processes.

2. `sa1` collects a data sample frequently (traditionally every 10
   minutes); `sa2` produces a daily summary report. Both are scheduled via
   cron or a systemd timer depending on the distro — the same scheduling
   mechanisms covered in Demo M02-05.

3. It's already been deleted — 10 days is outside the 7-day retention
   window. This matters because postmortems often happen well after an
   incident, so retention set too short means the data won't exist by the
   time someone needs it.

</details>

---

## Part 1 — Concepts

### 1.1 ip — the modern replacement for ifconfig/route/arp

The `ip` command (from `iproute2`) is the current standard for everything
this demo covers — addresses, links, routes, and the neighbor (ARP) table —
replacing the older `ifconfig`/`route`/`arp` trio, which are deprecated on
modern distros and sometimes not even installed by default. Every subcommand
follows the same pattern: `ip OBJECT COMMAND`, e.g. `ip addr show`, `ip link
set`, `ip route add`, `ip neigh show`.

```bash
ip addr show
# Often abbreviated: ip a
ip link show
# Often abbreviated: ip l
ip neigh show
# Often abbreviated: ip n
```

### 1.2 IPv4 addressing — CIDR notation and ip addr

```bash
ip addr show eth0
```

```
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP
    link/ether 02:42:ac:11:00:02 brd ff:ff:ff:ff:ff:ff
    inet 172.17.0.2/16 brd 172.17.255.255 scope global eth0
    inet6 fe80::42:acff:fe11:2/64 scope link
```

`inet 172.17.0.2/16` is CIDR notation — the address plus a `/N` prefix length
denoting how many leading bits define the network portion. `/16` means the
first 16 bits (172.17) are the network, leaving 16 bits (65,536 addresses)
for hosts on that network. This single number replaces the older
address+netmask pair (`255.255.0.0`) — both express the same thing, but CIDR
is what every modern tool, including `ip`, expects and displays.

`scope global` means this is a routable, externally-reachable address;
`scope link` (seen on the `inet6` line below it) means the address only
works for on-link communication and is never routed — this distinction
matters again in §1.3.

```bash
sudo ip addr add 192.168.1.50/24 dev eth0
sudo ip addr del 192.168.1.50/24 dev eth0
```

**This is not persistent** — an address added with `ip addr add` disappears on
reboot or interface restart. Persistent configuration lives in
distro-specific files (Netplan on Ubuntu, NetworkManager keyfiles or
`nmcli`/`nmtui` on Rocky9) — `ip` is the live-inspection and
live-modification tool, not the persistence layer.

### 1.3 IPv6 basics — coexistence, not replacement

Every interface with IPv6 enabled gets a **link-local address** automatically
(`fe80::/10` range, seen in the `ip addr` output above) — this is used for
on-link communication (like ARP's role in IPv4) and is never routed off the
local network. Global IPv6 addresses (routable, internet-facing) are a
separate concept, typically assigned via SLAAC (stateless autoconfiguration)
or DHCPv6.

```bash
ip -6 addr show
ip -6 route show
```

For this demo, the practical takeaway is deliberately narrower than full
IPv6 administration: recognize link-local (`fe80::`) vs global addresses in
`ip addr` output, and know that `-6` flags the same `ip` subcommands to
operate on IPv6 instead of IPv4 — IPv4 and IPv6 run side by side on the same
interface, not as alternatives you choose between. Actually *configuring*
IPv6 addressing (SLAAC tuning, DHCPv6 client setup) is out of scope here —
that's environment- and use-case-specific enough to belong in a dedicated
networking demo rather than the fundamentals pass.

### 1.4 ip neigh — the ARP replacement

`ip neigh` (neighbor) shows the kernel's table of IP-to-MAC address mappings
for hosts on the local network segment — the direct modern replacement for
the old `arp -a`.

```bash
ip neigh show
```

```
192.168.1.1 dev eth0 lladdr 02:42:ac:11:00:01 REACHABLE
192.168.1.23 dev eth0 lladdr 02:42:ac:11:00:17 STALE
```

Each line maps an IP address to a MAC (`lladdr`) address the kernel has
learned on that interface, plus a state: `REACHABLE` (recently confirmed),
`STALE` (learned but not recently confirmed — still usable, will be
re-verified on next use), or `FAILED` (confirmation attempt failed). This
table is populated automatically as traffic flows — you don't typically add
entries by hand, but reading it is useful when diagnosing "I can ping the
gateway by IP but something upstream of ARP resolution looks wrong."

> ⚠️ [VERIFY] Confirmed directly: after a single real ping between two
> namespaced hosts with no prior ARP traffic, `ip neigh show` on the
> initiating side showed `10.55.0.2 dev va lladdr 8a:aa:fc:18:4b:4f
> REACHABLE` — a genuinely learned MAC from real ARP resolution, not a
> pre-seeded or simulated entry.

### 1.5 ip route — the default gateway and static routes

```bash
ip route show
```

```
default via 172.17.0.1 dev eth0
172.17.0.0/16 dev eth0 proto kernel scope link src 172.17.0.2
```

The `default via X dev Y` line is the default gateway — where traffic goes
when no more specific route matches. The second line is a kernel-derived
route for the local subnet, added automatically when the interface got its
address; you didn't configure it by hand.

> ⚠️ [VERIFY] Confirmed directly: adding an address to a real interface with
> no route commands run at all produced `192.168.199.0/24 dev vt0 proto
> kernel scope link src 192.168.199.5` in `ip route show` — the kernel adds
> this the instant the address is assigned. `proto kernel` is the tell that
> nobody configured this by hand.

```bash
sudo ip route add 10.0.5.0/24 via 172.17.0.1 dev eth0
sudo ip route del 10.0.5.0/24
```

Same persistence caveat as addresses: routes added this way are live-only and
vanish on reboot unless written into the distro's persistent network
configuration.

### 1.6 Hostname resolution — /etc/hosts comes before DNS

**The most commonly missed fact about hostname resolution: DNS is not the
first thing consulted.** The actual order is controlled by
`/etc/nsswitch.conf`:

```bash
grep ^hosts: /etc/nsswitch.conf
# hosts: files dns
```

`files` (meaning `/etc/hosts`) is checked before `dns` in the default order
on both distros. This means a stale or wrong entry in `/etc/hosts` silently
overrides DNS — a name resolving "wrong" is very often an `/etc/hosts` entry
someone forgot about, not a DNS problem at all.

> ⚠️ [VERIFY] Confirmed directly: with a made-up name added only to
> `/etc/hosts` (no real DNS record exists for it), `getent hosts` — which
> respects the full `nsswitch.conf` order — returned the `/etc/hosts`
> entry immediately; `dig`, which queries DNS directly and ignores
> `/etc/hosts` entirely, returned nothing for the same name. This is the
> concrete proof of the ordering, not just the documented default.

```bash
cat /etc/hosts
# 127.0.0.1   localhost
# 172.17.0.2  novatech-app-01
```

### 1.7 DNS basics — resolv.conf and dig

```bash
cat /etc/resolv.conf
# nameserver 8.8.8.8
# search novatech.internal
```

`nameserver` lines list DNS servers to query, in order; `search` appends a
domain suffix to unqualified names — querying `db01` actually queries
`db01.novatech.internal` first (the `search` value from above), and only
falls back to querying `db01` as-typed if that fails.

> ⚠️ [VERIFY] Confirmed directly against a real local DNS server holding
> only a record for `db01.novatech.internal`: querying the bare name
> `db01` with suffix-appending enabled resolved correctly (`10.0.9.9`);
> the identical bare-name query with suffix-appending disabled returned
> nothing. This confirms the suffix-then-fallback mechanism itself, tested
> via `dig`'s own `+search` behavior — the same suffix-appending logic
> `resolv.conf`'s `search` directive triggers for standard resolver-library
> lookups (`getent`, most applications).

**How `/etc/resolv.conf` is managed differs by distro — don't assume the
same mechanism applies to both:**

```
Ubuntu:   systemd-resolved is the default resolver. /etc/resolv.conf is
          typically a symlink to /run/systemd/resolve/stub-resolv.conf.
          Editing it directly can be silently overwritten on the next
          network change — `resolvectl status` shows the actual effective
          configuration in this case.

Rocky9:   NetworkManager writes /etc/resolv.conf directly as a real file
          by default — systemd-resolved is NOT the default resolver here.
          Editing it directly persists until NetworkManager next
          regenerates it (e.g. on a network restart or reconnect).
```

Either way, treat `/etc/resolv.conf` as generated/managed configuration
rather than something to hand-edit permanently — the same lesson as
addresses and routes (§1.2/§1.5): make the change through the distro's
actual configuration system, not by editing a file something else
regenerates.

> ⚠️ [VERIFY — docs only, not live-tested]: the specific
> `systemd-resolved`-symlink-vs-NetworkManager-real-file distinction above
> is standard, well-documented distro behavior, but this session's
> container has no `systemd`/`NetworkManager` to run and confirm it
> against directly. Confirm on your own Ubuntu/Rocky lab VM with `ls -la
> /etc/resolv.conf` before relying on it.

```bash
dig novatech.internal
dig +short novatech.internal        # just the answer, no verbose output
dig @8.8.8.8 novatech.internal      # query a specific server directly,
                                     # bypassing resolv.conf — useful for
                                     # isolating "is it my resolver config or
                                     # the DNS server itself"
```

### 1.8 Time synchronization — why it matters more than it seems

Clock drift breaks things that don't look like clock problems: TLS
certificate validation (certs have time-bounded validity), Kerberos
authentication, log correlation across servers, and cron/timer scheduling
accuracy (Demo M02-05). A machine whose clock has silently drifted can produce
confusing, seemingly-unrelated failures.

```bash
timedatectl status
```

```
               Local time: Fri 2026-07-18 14:22:01 UTC
           Universal time: Fri 2026-07-18 14:22:01 UTC
                 RTC time: Fri 2026-07-18 14:22:01
                Time zone: UTC (UTC, +0000)
System clock synchronized: yes
              NTP service: active
```

Ubuntu defaults to `systemd-timesyncd` (lightweight, client-only — syncs
*to* NTP servers but doesn't serve time to others); Rocky9 defaults to
`chrony` (fuller-featured — can also act as an NTP server, handles larger
clock jumps more gracefully). Both are configured similarly in spirit:

```bash
# systemd-timesyncd (Ubuntu):
cat /etc/systemd/timesyncd.conf | grep -v '^#'
systemctl status systemd-timesyncd

# chrony (Rocky9):
cat /etc/chrony.conf | grep ^server
chronyc tracking
```

`chronyc tracking`'s `System time` line shows the current offset from the
reference — a large or growing offset means sync isn't working correctly,
worth checking before trusting any timestamp-dependent output on that machine.

**Enabling and disabling NTP sync — an actual configuration action, not just
inspection:**

```bash
sudo timedatectl set-ntp false   # disable NTP-based sync
timedatectl status | grep "NTP service"
# NTP service: inactive

sudo timedatectl set-ntp true    # re-enable it
timedatectl status | grep "System clock synchronized"
# System clock synchronized: yes  (may take a few seconds to resync)
```

`timedatectl set-ntp` works the same way on both distros — it's a systemd
control surface that toggles whichever backend (`systemd-timesyncd` or
`chrony`) is active. This is genuinely changing configuration, not just
reading status — useful to know both because you may need to disable sync
temporarily (e.g. a VM whose hypervisor already handles time) and because
knowing it exists means "is NTP even turned on" is always a one-command check.

> ⚠️ [VERIFY — docs only, not live-tested]: this session's container has
> no `systemd` (`timedatectl`/`chronyc` need a running `systemd`/`chronyd`
> to demonstrate against), so the exact status output and toggle behavior
> above are documented rather than reproduced live here. Confirm on your
> own lab VM.

### 1.9 OpenSSH — client, server, and key-based authentication

```bash
sudo systemctl status sshd    # Rocky9
sudo systemctl status ssh     # Ubuntu (service name differs)
```

The server's behavior is controlled by `/etc/ssh/sshd_config`. Two settings
worth knowing by name even in a "fundamentals" pass: `PasswordAuthentication`
(whether password login is allowed at all) and `PermitRootLogin` (whether
`root` can SSH in directly) — both get hardened in Demo M07-01 (Security), but
recognizing them now matters for understanding what key-based auth is
actually replacing.

**Setting up key-based authentication:**

```bash
ssh-keygen -t ed25519 -C "novatech-ops-laptop"
# -t ed25519: modern, fast, secure key type — the current default
# recommendation for new keys (elliptic-curve, shorter keys than RSA at
# equivalent security, resistant to the timing-attack classes some older
# RSA implementations were vulnerable to). Prefer it over RSA/DSA for any
# newly generated key; only fall back to RSA for compatibility with a
# system too old to support ed25519.
# -C: a comment, purely descriptive, helps identify keys later.

ssh-copy-id wadmin@app-server-01
# Appends your public key to the remote user's ~/.ssh/authorized_keys —
# handles file/permission setup correctly, which matters because SSH is
# strict about permissions (see below).
```

**The single most common SSH key setup failure is permissions**, not the key
itself:

```bash
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
chmod 600 ~/.ssh/id_ed25519      # private key
chmod 644 ~/.ssh/id_ed25519.pub  # public key — fine to be world-readable
```

`sshd` refuses to use `authorized_keys` (or the `.ssh` directory itself) if
its permissions are too permissive — group- or world-writable — and fails
*silently back to password auth* rather than erroring loudly, which is
exactly what makes this failure mode confusing the first time you hit it.

> ⚠️ [VERIFY] Confirmed directly against a real running `sshd`: with
> `~/.ssh` at `777` and `authorized_keys` at `666`, key-based auth was
> refused (`Permission denied (publickey)`), and `sshd`'s own log stated
> the exact reason: `Authentication refused: bad ownership or modes for
> file /home/.../authorized_keys`. Restoring `700`/`600` with nothing
> else changed made the identical key succeed immediately. With password
> auth also enabled server-side, the failure message changed to
> `Permission denied (publickey,password)` — confirming password auth is
> genuinely offered as the next method, not just theoretically available.

---

## Part 2 — Hands-On Lab

### Commands used in this demo

| Command | Flag meaning | Purpose |
|---|---|---|
| `ip addr show` | — | List interfaces and their addresses |
| `ip addr add ADDR/PREFIX dev IFACE` | — | Add an address (live only, not persistent) |
| `ip neigh show` | — | Show the ARP/neighbor table (IP-to-MAC mappings) |
| `ip route show` | — | List routes, including default gateway |
| `ip route add NET via GW dev IFACE` | — | Add a static route (live only) |
| `ip route del NET` | — | Remove a static route (live only) |
| `dig +short NAME` | `+short`: answer only | Quick DNS lookup |
| `dig @SERVER NAME` | `@SERVER`: query directly | Bypass resolv.conf, isolate resolver vs. server issues |
| `timedatectl status` | — | Check sync status (both distros) |
| `timedatectl set-ntp true\|false` | — | Enable or disable NTP-based sync |
| `chronyc tracking` | — | Detailed sync offset (chrony/Rocky9) |
| `ssh-keygen -t ed25519` | `-t`: key type | Generate a modern SSH keypair |
| `ssh-copy-id user@host` | — | Install your public key on a remote host correctly |

### Task 1 — Inspect interfaces, addresses, and the neighbor table

```bash
ip addr show
ip link show
ip neigh show
```

Identify: which interface has a global (routable) address, which has only
link-local IPv6, what the interface's current state (`UP`/`DOWN`) is, and
which neighbor entries are `REACHABLE` vs `STALE`.

### Task 2 — Add and remove an address (live, non-persistent)

```bash
sudo ip addr add 192.168.100.50/24 dev eth0
ip addr show eth0 | grep 192.168.100.50
sudo ip addr del 192.168.100.50/24 dev eth0
ip addr show eth0 | grep 192.168.100.50   # confirm it's gone
```

### Task 3 — Read routes, then add and remove a static route

```bash
ip route show
# Identify the default gateway line and the automatically-added local
# subnet route — which one did you configure, and which did the kernel add?

# Add a static route to a documentation/test-only range (RFC 5737 —
# guaranteed non-routable on the real internet, safe to practice with)
GW=$(ip route show default | awk '{print $3}')
sudo ip route add 203.0.113.0/24 via $GW dev eth0
ip route show | grep 203.0.113.0

# Remove it
sudo ip route del 203.0.113.0/24
ip route show | grep 203.0.113.0   # confirm it's gone — no output
```

### Task 4 — Hostname resolution order, hands-on

```bash
grep ^hosts: /etc/nsswitch.conf

echo "203.0.113.99  novatech-decoy" | sudo tee -a /etc/hosts
dig +short novatech-decoy    # dig queries DNS directly — won't reflect /etc/hosts
getent hosts novatech-decoy  # getent respects nsswitch.conf's full resolution order
# Compare the two results — this demonstrates why dig alone can mislead you
# about "what will this machine actually resolve this name to."
sudo sed -i '/novatech-decoy/d' /etc/hosts
```

### Task 5 — DNS query, then verify and configure time sync

```bash
dig +short example.com
dig @8.8.8.8 +short example.com

# Verify current sync state
timedatectl status
# or on Rocky9:
chronyc tracking

# Now configure — toggle NTP sync off, confirm it stops, then re-enable
sudo timedatectl set-ntp false
timedatectl status | grep "NTP service"
# NTP service: inactive

sudo timedatectl set-ntp true
timedatectl status | grep "System clock synchronized"
# System clock synchronized: yes (may take a few seconds to resync)
```

### Task 6 — SSH key setup and the permissions trap

```bash
ssh-keygen -t ed25519 -C "demo11-lab-key" -f ~/.ssh/demo11_key -N ""
chmod 700 ~/.ssh
chmod 600 ~/.ssh/demo11_key
chmod 644 ~/.ssh/demo11_key.pub

# Deliberately break it, then observe:
chmod 644 ~/.ssh/demo11_key   # too permissive for a private key
ssh -i ~/.ssh/demo11_key -o PasswordAuthentication=no wadmin@localhost
# Expect a permissions-related refusal from ssh itself before it even
# attempts the connection — the client enforces private key permissions too.
chmod 600 ~/.ssh/demo11_key   # fix it
```

---

## What You Learned

- ✅ Read and configured IPv4 addresses with `ip addr`; read and identified IPv6 address types (link-local vs global)
- ✅ Read routes with `ip route`, identified the default gateway, and added and removed a static route
- ✅ Inspected the neighbor (ARP) table with `ip neigh`, completing the `ip` replacement of `ifconfig`/`route`/`arp`
- ✅ Explained the actual hostname resolution order: `/etc/hosts` before DNS
- ✅ Queried DNS with `dig`, explained what `/etc/resolv.conf` controls — including how its management differs between Ubuntu (systemd-resolved) and Rocky9 (NetworkManager)
- ✅ Configured and verified time synchronization with `timedatectl`/`chronyc`, including enabling and disabling NTP sync
- ✅ Set up SSH key-based authentication and diagnosed a permissions failure

**Key Takeaway:** Networking fundamentals on Linux come down to knowing which layer you're looking at and whether a change is live or persistent — `ip` commands are live-only, `/etc/hosts` silently outranks DNS, and both `/etc/resolv.conf` and SSH key trust are governed by rules (managed-file regeneration, strict permission checks) that fail silently rather than loudly when violated.

---

## Part 3 — Break-Fix Scenario

### Scenario — NovaTech Ops: two connectivity incidents

> **Incident report — NovaTech Ops, ticket #4531**
>
> Alert 1: a teammate says `internal-api.novatech.local` "resolves to the
> wrong server" on their laptop only — works fine for everyone else. Alert
> 2: a new hire can't SSH into `build-agent-02` with their key — "permission
> denied," even though they're sure the key is installed correctly.

**Incident 1 — resolves to the wrong server, one laptop only**

```bash
$ dig +short internal-api.novatech.local
10.0.4.12
$ cat /etc/hosts | grep novatech
10.0.4.99   internal-api.novatech.local   # added months ago, "temporarily"
$ getent hosts internal-api.novatech.local
10.0.4.99   internal-api.novatech.local
```

Tasks:
1. `dig` and `getent` gave different answers. Which one reflects what this
   machine will *actually* use when, say, a browser or `curl` resolves this
   name? Justify using §1.6.
2. Why does this only affect one laptop, given that DNS itself (per `dig`)
   returns the correct, current answer?
3. What's the fix, and why is checking `/etc/hosts` a mandatory first step
   any time DNS-based troubleshooting (`dig`) contradicts what a user
   actually experiences?

**Incident 2 — new hire's SSH key "permission denied"**

```bash
$ ssh -v build-agent-02
...
debug1: Authentications that can continue: publickey,password
debug1: Offering public key: /home/newhire/.ssh/id_ed25519
debug1: Authentications that can continue: publickey,password
...
Permission denied (publickey,password).
$ ls -la ~/.ssh
drwxrwxr-x 2 newhire newhire 4096 Jul 18 10:02 .
-rw-rw-r-- 1 newhire newhire  411 Jul 18 09:58 authorized_keys
-rw------- 1 newhire newhire  411 Jul 18 09:55 id_ed25519
```

Tasks:
1. The private key itself (`id_ed25519`) has correct permissions
   (`600`/`-rw-------`). What's actually wrong here — check the `.ssh`
   directory and `authorized_keys` permissions against §1.9's rules.
2. Why does `sshd` fail this *silently back to a password prompt* rather than
   printing an explicit "your authorized_keys permissions are wrong" error —
   what does that design choice mean for how you should approach any
   "publickey auth isn't working" report?
3. Fix the permissions and explain, in one sentence each, why `.ssh` needs
   `700` and `authorized_keys` needs `600` specifically (not more permissive).

<details>
<summary>Answers</summary>

**Incident 1:** `getent hosts` reflects reality — it respects the full
`nsswitch.conf` order (`files dns`), which means `/etc/hosts` is consulted
first and, since it has a matching entry, wins before DNS is ever queried.
`dig` bypasses that entirely and queries DNS directly, which is why it shows
the correct current answer while the machine itself still uses the stale one.
This only affects one laptop because `/etc/hosts` is a local, per-machine
file — nobody else's laptop has that leftover entry. Fix: remove the stale
line from `/etc/hosts`. Checking `/etc/hosts` first is mandatory precisely
because `dig` alone cannot detect this class of problem — it queries a
different, unaffected path than what the machine actually resolves through
day to day.

**Incident 2:** The `.ssh` directory is `775` (group and world writable) and
`authorized_keys` is `664` (group and world writable) — both too permissive.
Per §1.9, `sshd` refuses to trust `authorized_keys` if its permissions (or
its containing directory's) allow group/world write access, since that would
mean anyone in the group, or on some setups anyone at all, could add their
own key and gain access — and it fails back to password auth silently rather
than erroring, because from `sshd`'s perspective this isn't a broken
configuration, it's a security policy correctly declining to trust
untrustworthy file permissions. Practically, this means any "publickey auth
isn't working" report should have permissions on `.ssh` and
`authorized_keys` checked essentially first, before assuming the key itself
is the problem. Fix: `chmod 700 ~/.ssh && chmod 600
~/.ssh/authorized_keys`. `.ssh` needs `700` so only the owner can even list
or enter the directory; `authorized_keys` needs `600` so only the owner can
modify which keys are trusted for their own account.

</details>

---

## Interview Prep

**Q1. A name resolves correctly via `dig` but a user says the "wrong" server responds when they actually connect. What's your first check?**

`/etc/hosts` on that specific machine. `dig` queries DNS directly and
bypasses the full resolution order; the machine's actual resolution order
(`/etc/nsswitch.conf`, typically `files dns`) checks `/etc/hosts` first. A
stale local entry there silently overrides a correct DNS answer, and this is
one of the most common causes of "DNS looks right but the behavior is wrong"
reports.

---

**Q2. Why does `ip addr add`/`ip route add` disappear after a reboot, and what should you do instead for a permanent change?**

Because `ip` operates on the kernel's live network state directly — it's an
inspection-and-modification tool, not a configuration-persistence layer.
Permanent changes need to go into the distro's actual network configuration
(Netplan on Ubuntu, NetworkManager profiles/`nmcli` on Rocky9), which is what
gets re-applied on boot; `ip` commands are for live troubleshooting and
temporary changes, not the source of truth.

---

**Q3. An SSH key-based login fails with "permission denied," falling back to a password prompt. What do you check before assuming the key itself is wrong?**

Permissions on `~/.ssh` (must not be group/world-writable — `700`) and
`~/.ssh/authorized_keys` (`600`) on the target server. `sshd` silently
distrusts overly-permissive permissions on these and falls back to password
auth without an explicit error, which is exactly why this is worth checking
first rather than assuming the key itself is malformed or wasn't copied
correctly.

---

**Q4. Why does clock synchronization matter for a server that isn't obviously "time-sensitive" — say, a plain web application server?**

Clock drift causes failures that don't look like clock problems: TLS
certificate validation depends on the current time being within the
certificate's validity window, correlating logs across multiple servers
during an incident requires their clocks to actually agree, and any
timer-based scheduling (systemd timers, cron) depends on the system clock
being accurate. A server can run for a long time with a drifted clock before
anyone connects the resulting symptoms back to the actual cause.

---

## Key Takeaways

1. **`/etc/hosts` is checked before DNS, not after.** `/etc/nsswitch.conf`'s
   `hosts: files dns` order means a stale local `/etc/hosts` entry silently
   overrides a correct DNS answer — `dig` alone can't detect this because it
   queries DNS directly, bypassing that order entirely.
2. **`ip addr`/`ip route` changes are live-only, not persistent.** Permanent
   network configuration lives in the distro's own configuration system
   (Netplan, NetworkManager) — `ip` is for inspection, temporary changes, and
   practicing add/remove operations like the route added in Task 3.
3. **CIDR notation (`/N`) replaces netmask notation everywhere modern
   tooling is concerned.** `/24` and `255.255.255.0` mean the same thing, but
   `ip` and virtually every current tool expects and displays CIDR.
4. **SSH key auth failures are permissions problems more often than key
   problems.** `sshd` silently refuses group/world-writable `.ssh` or
   `authorized_keys` and falls back to password auth without an explicit
   error — check permissions before assuming the key itself is wrong.
5. **`dig @SERVER` bypassing `resolv.conf` is the fastest way to isolate
   "is it my resolver config or the actual DNS server."** Querying a known-
   good external server directly rules out local resolver misconfiguration
   as the cause.
6. **Clock drift causes failures that don't look like clock problems.** TLS
   validation, log correlation, and timer/cron scheduling accuracy all
   depend on synchronized time — check `timedatectl`/`chronyc` before
   chasing a mysteriously intermittent failure on unrelated grounds.
   `timedatectl set-ntp` is the one-command way to confirm sync is actually
   turned on, not just assumed to be.
7. **`/etc/resolv.conf` is managed differently per distro — don't hand-edit
   it expecting the change to stick.** Ubuntu's `systemd-resolved` treats it
   as a generated symlink; Rocky9's NetworkManager writes it as a real file
   but still regenerates it on network changes. Same underlying lesson as
   `ip addr`/`ip route`: change it through the distro's configuration
   system, not by hand.

---

## Quick Commands Reference

### Addressing, links, and neighbors

| Command | Purpose |
|---|---|
| `ip addr show` | List interfaces and addresses |
| `ip addr add ADDR/PREFIX dev IFACE` | Add address (live only) |
| `ip -6 addr show` | IPv6-specific address view |
| `ip link show` | Interface state (UP/DOWN) |
| `ip neigh show` | Show ARP/neighbor table (IP-to-MAC mappings) |

### Routing

| Command | Purpose |
|---|---|
| `ip route show` | List routes including default gateway |
| `ip route add NET via GW dev IFACE` | Add a static route (live only) |
| `ip route del NET` | Remove a static route (live only) |

### Hostname resolution and DNS

| Command | Purpose |
|---|---|
| `grep ^hosts: /etc/nsswitch.conf` | Confirm resolution order |
| `getent hosts NAME` | Resolve respecting the full nsswitch order |
| `dig +short NAME` | Quick DNS-only lookup (bypasses /etc/hosts) |
| `dig @SERVER NAME` | Query a specific DNS server directly |
| `resolvectl status` | Effective resolver config (Ubuntu/systemd-resolved) |

### Time sync

| Command | Purpose |
|---|---|
| `timedatectl status` | Sync status, both distros |
| `timedatectl set-ntp true\|false` | Enable/disable NTP-based sync |
| `chronyc tracking` | Detailed offset (chrony/Rocky9) |

### SSH key setup

| Command | Purpose |
|---|---|
| `ssh-keygen -t ed25519` | Generate a modern keypair |
| `ssh-copy-id user@host` | Install public key correctly on a remote host |
| `chmod 700 ~/.ssh && chmod 600 ~/.ssh/authorized_keys` | Required permissions for key auth to be trusted |

---

## Appendix A — Anki Cards

**`m03-01-networking-fundamentals-anki.csv`:**

```
#deck:Linux Mastery::Module 3 - Networking::Demo M03-01 - Networking Fundamentals
#separator:Comma
#columns:Front,Back,Tags
"A name resolves correctly via dig, but a user's machine connects to the wrong server. What's the first thing to check?","/etc/hosts on that specific machine — nsswitch.conf's default hosts: files dns order means /etc/hosts is checked before DNS, so a stale local entry silently overrides a correct DNS answer that dig alone (which queries DNS directly) won't reveal.","demo11,hostname-resolution,troubleshooting"
"Why does an address added with ip addr add disappear after a reboot?","ip operates on the kernel's live network state directly — it's an inspection/modification tool, not the persistence layer. Permanent changes must go into the distro's actual network configuration (Netplan on Ubuntu, NetworkManager on Rocky9).","demo11,ip-command,persistence"
"An SSH key login fails with permission denied and falls back to a password prompt. What should you check before assuming the key is wrong?","Permissions on ~/.ssh (must be 700) and ~/.ssh/authorized_keys (must be 600) on the target server. sshd silently distrusts group/world-writable permissions on these and falls back to password auth without an explicit error.","demo11,ssh,troubleshooting"
"What does dig @8.8.8.8 novatech.internal do differently from plain dig novatech.internal?","It queries the specified server (8.8.8.8) directly, bypassing /etc/resolv.conf entirely — useful for isolating whether a resolution problem is local resolver misconfiguration or an issue with the DNS server itself.","demo11,dns,dig"
"What does CIDR notation like /24 actually represent?","The number of leading bits that define the network portion of the address — /24 means the first 24 bits are network, leaving 8 bits (256 addresses) for hosts. It replaces the older netmask notation (255.255.255.0), which expresses the same thing differently.","demo11,ipv4,cidr"
"On an interface with IPv6 enabled, what is the fe80::/10 address used for, and is it routable off the local network?","It's the link-local address, used for on-link communication (similar in role to ARP for IPv4). It is never routed off the local network — global (routable, internet-facing) IPv6 addresses are a separate, additional assignment.","demo11,ipv6,link-local"
"Why does clock drift on a server cause failures that don't obviously look like clock problems?","TLS certificate validation depends on current time being within the cert's validity window, log correlation across servers requires agreeing clocks, and timer/cron scheduling accuracy depends on the system clock — none of these failures look like 'the clock is wrong' on the surface.","demo11,time-sync,troubleshooting"
"What's the default hostname resolution order set by /etc/nsswitch.conf's hosts: line on both Ubuntu and Rocky9?","files dns — /etc/hosts (files) is checked before DNS (dns).","demo11,hostname-resolution,basics"
"Ubuntu defaults to systemd-timesyncd and Rocky9 defaults to chrony for time sync. What's the key functional difference?","systemd-timesyncd is lightweight and client-only (syncs to NTP servers but can't serve time to others). chrony is fuller-featured — it can also act as an NTP server itself and handles larger clock jumps more gracefully.","demo11,time-sync,chrony,timesyncd"
"getent hosts NAME and dig NAME can return different answers for the same name. Why, and which one reflects what the machine will actually use?","getent respects the full nsswitch.conf resolution order (checking /etc/hosts before DNS); dig queries DNS directly, bypassing /etc/hosts. getent's answer is what the machine will actually resolve to in normal use.","demo11,hostname-resolution,dig,getent"
```

---

## Appendix B — Quiz

**`m03-01-networking-fundamentals-quiz.md`:**

````markdown
# Quiz — Demo M03-01: Networking Fundamentals

> One correct answer per question unless stated otherwise.
> Target: 80% or above before moving to Demo M03-02.

---

**Q1.** `ip route show` outputs these two lines:
```
default via 172.17.0.1 dev eth0
172.17.0.0/16 dev eth0 proto kernel scope link src 172.17.0.2
```
Which statement correctly distinguishes them?

- A) Both lines were added manually with `ip route add`
- B) The first line is the default gateway (where unmatched traffic goes); the second is a kernel-derived route for the local subnet, added automatically when the interface got its address
- C) The second line is the default gateway; the first is a manually configured static route
- D) These two lines conflict and only one can be active at a time

<details>
<summary>Answer</summary>

**B** — `default via X dev Y` is the default gateway route. The `172.17.0.0/16 ... proto kernel` line is added automatically by the kernel the moment the interface receives its address — nobody configured it by hand. `proto kernel` is the tell.

Trap: A assumes both are manual; the local-subnet route specifically is not. D invents a conflict that doesn't exist — both routes coexist and serve different purposes (specific subnet vs. everything else).

</details>

---

**Q2.** `ip addr show eth0` includes this line:
```
    inet6 fe80::42:acff:fe11:2/64 scope link
```
What does `scope link` tell you here, distinct from what `scope global` would mean?

- A) The address is currently down and unusable
- B) The address only works for on-link (local network segment) communication and is never routed off the local network — a global-scope address would be routable/internet-facing instead
- C) The address is reserved for future use and not yet active
- D) `scope link` means the interface is a virtual/loopback device

<details>
<summary>Answer</summary>

**B** — `scope link` marks an address (typically the automatic `fe80::` link-local IPv6 address) as valid only for communication on the immediate local network segment. `scope global` marks a routable, externally-reachable address. This is the same global-vs-link-local distinction covered for IPv6 in §1.3.

Trap: A and C invent states unrelated to what `scope` actually indicates; D is unrelated — loopback has its own distinct addressing, not marked this way.

</details>

---

**Q3.** In `/etc/ssh/sshd_config`, what do `PasswordAuthentication` and `PermitRootLogin` each control?

- A) `PasswordAuthentication` sets the SSH port; `PermitRootLogin` sets the idle timeout
- B) `PasswordAuthentication` controls whether password-based login is allowed at all; `PermitRootLogin` controls whether the `root` account can SSH in directly
- C) Both settings only take effect after a full server reboot
- D) `PermitRootLogin` controls password login; `PasswordAuthentication` controls root access — the names are reversed from what they do

<details>
<summary>Answer</summary>

**B** — `PasswordAuthentication` is the on/off switch for password-based login as a whole (independent of who's logging in); `PermitRootLogin` specifically governs whether `root` can authenticate over SSH at all, regardless of method. Both are commonly hardened together (disable both) but they control different things. Full hardening of these is covered in Demo M07-01 — this demo only requires recognizing what each setting is for.

Trap: D swaps the definitions; A invents unrelated meanings for both.

</details>

---

**Q4.** `/etc/resolv.conf` contains:
```
nameserver 8.8.8.8
search novatech.internal
```
You run `dig db01` (no domain suffix). What does the resolver actually query, and in what order?

- A) Only `db01` — `search` has no effect on `dig`
- B) Only `db01.novatech.internal` — the bare name is never tried
- C) `db01.novatech.internal` first (the `search` suffix appended), falling back to `db01` as-typed if that fails
- D) Both are queried simultaneously and the first response wins

<details>
<summary>Answer</summary>

**C** — The `search` directive appends its domain suffix to unqualified (no-dot) names before querying, trying the suffixed form first. Only if that fails does resolution fall back toward the name as typed. This is why an internal short name like `db01` "just works" inside the `novatech.internal` network without anyone typing the full FQDN.

Trap: A and B each describe only one half of the actual two-step behavior; D invents a race condition that isn't how resolution order works.

</details>

---

**Q5.** You need to route traffic for `203.0.113.0/24` through a specific gateway, then remove that route once you're done testing. Which command pair is correct?

- A) `ip addr add 203.0.113.0/24 via GW dev eth0` then `ip addr del 203.0.113.0/24`
- B) `ip route add 203.0.113.0/24 via GW dev eth0` then `ip route del 203.0.113.0/24`
- C) `ip route new 203.0.113.0/24 gateway GW` then `ip route remove 203.0.113.0/24`
- D) Routes cannot be removed once added without a reboot

<details>
<summary>Answer</summary>

**B** — `ip route add NET via GW dev IFACE` adds a static route; `ip route del NET` removes it. Like address changes, this is live-only and non-persistent — it vanishes on reboot regardless of whether you explicitly delete it first.

Trap: A confuses the `addr` object (for interface addresses) with the `route` object (for routing table entries) — they are not interchangeable. C invents subcommand names that don't exist in `ip`'s actual syntax.

</details>

---

**Q6.** Why is `ed25519` generally preferred over `RSA` when generating a new SSH keypair today?

- A) `ed25519` keys are always backward-compatible with every SSH server ever released, while RSA is not
- B) `ed25519` is an elliptic-curve key type that's faster and produces shorter keys than RSA at equivalent security, and avoids timing-attack classes some older RSA implementations were vulnerable to
- C) `ed25519` keys never need to be protected with a passphrase
- D) RSA keys are deprecated and rejected by all modern SSH servers

<details>
<summary>Answer</summary>

**B** — `ed25519` is the current default recommendation for new keys: shorter, faster, and avoids certain implementation-level vulnerability classes older RSA setups have had. RSA still works broadly and isn't rejected outright (D is false), so ed25519 is a preference for new keys, not a hard requirement — but the compatibility claim in A overstates it (very old systems may lack ed25519 support, which is the actual reason to fall back to RSA).

Trap: A overstates compatibility in the wrong direction; C and D are both factually wrong claims about key behavior.

</details>

---

**Q7.** `timedatectl status` shows `NTP service: active`. You want to confirm that time sync is actually configurable, not just a fixed default. Which command demonstrates this by actually changing state?

- A) `timedatectl status --verbose`
- B) `sudo timedatectl set-ntp false`, then `sudo timedatectl set-ntp true`
- C) `chronyc tracking --reset`
- D) `sudo systemctl restart NetworkManager`

<details>
<summary>Answer</summary>

**B** — `timedatectl set-ntp false`/`true` is the actual configuration action — it disables and re-enables NTP-based sync via whichever backend (`systemd-timesyncd` or `chrony`) is active, and works identically on both distros. `timedatectl status` alone (A) is read-only. C invents a flag that doesn't exist for `chronyc tracking`. D restarts an unrelated service and wouldn't reliably toggle NTP sync state.

</details>

---

**Q8.** Ubuntu and Rocky9 both use `/etc/resolv.conf`, but manage it differently by default. Which statement is accurate?

- A) Both distros manage it identically via `systemd-resolved`
- B) On Ubuntu, `/etc/resolv.conf` is typically a symlink managed by `systemd-resolved`; on Rocky9, NetworkManager writes it as a real file directly, and `systemd-resolved` is not the default resolver
- C) Rocky9 uses `systemd-resolved` by default while Ubuntu does not
- D) Neither distro manages `/etc/resolv.conf` automatically — it must always be hand-edited

<details>
<summary>Answer</summary>

**B** — Ubuntu's default resolver is `systemd-resolved`, with `/etc/resolv.conf` typically symlinked to `/run/systemd/resolve/stub-resolv.conf` — hand-edits can be silently overwritten. Rocky9's default is NetworkManager writing `/etc/resolv.conf` directly as a real file; `systemd-resolved` isn't the default there. On both, the safer path for a persistent change is through the distro's actual network configuration system rather than a direct hand-edit.

Trap: A and C both invent a symmetric or reversed default that doesn't match either distro's actual behavior; D ignores that both distros do manage this file automatically by default.

</details>

---

**Q9.** `ip neigh show` outputs `192.168.1.23 dev eth0 lladdr 02:42:ac:11:00:17 STALE`. What does `STALE` mean, and what classic command does `ip neigh` replace?

- A) The entry failed resolution and cannot be used until manually removed
- B) The mapping was learned but not recently reconfirmed — still usable, will be re-verified on next use. `ip neigh` replaces the old `arp -a`
- C) The IP address is no longer assigned to any device on the network
- D) `STALE` means the neighbor table itself needs to be flushed and rebuilt

<details>
<summary>Answer</summary>

**B** — `STALE` is a normal, usable state — the kernel simply hasn't reconfirmed that mapping recently; it will re-verify it automatically the next time it's used. It's distinct from `FAILED`, which does mean confirmation broke down. `ip neigh` is the direct modern replacement for the deprecated `arp -a`, completing the same `ip`-consolidates-`ifconfig`/`route`/`arp` pattern covered in §1.1/§1.4.

Trap: A confuses `STALE` with `FAILED`; C and D invent meanings unrelated to what the state field actually indicates.

</details>

---

| Score | Action |
|---|---|
| 8/9 or 9/9 | Import Anki cards, move to Demo M03-02 |
| 6-7/9 | Review the wrong answers, then proceed |
| 5/9 | Re-read the relevant sections, retry those questions |
| Below 5/9 | Re-read the full demo and redo the walkthrough before proceeding |
````