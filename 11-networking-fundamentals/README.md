# Demo 11 — Networking Fundamentals

## Overview

Every server needs to be reachable, needs to reach other things, and needs to
agree with the rest of the world on what time it is and what names mean. This
demo covers the foundational layer: reading and configuring IPv4/IPv6
addresses and routes with the modern `ip` command, how hostname resolution
actually works before DNS even gets involved, why clock sync matters more
than it seems, and setting up SSH key authentication correctly. Demo 12
builds on this with monitoring, firewalls, and troubleshooting — this demo is
the "get it configured and reachable" half.

## Demo Objectives

1. Read and configure IPv4/IPv6 addresses and interface state with `ip`
2. Read and add routes with `ip route`, and identify the default gateway
3. Explain the actual hostname resolution order: `/etc/hosts` before DNS
4. Query DNS with `dig`, and explain what `/etc/resolv.conf` controls
5. Configure and verify time synchronization with chrony or systemd-timesyncd
6. Set up SSH key-based authentication correctly, including common permission pitfalls

## Directory Structure

```
11-networking-fundamentals/
└── README.md   # This file — full demo content, including Anki (Appendix A)
                 # and Quiz (Appendix B), extracted later via script.
```

## Environments

| Task | Ubuntu 24.04 | Rocky Linux 9.8 | EC2 Amazon Linux |
|---|---|---|---|
| All lab tasks | ✅ | ✅ | Not required |
| Time sync daemon | `systemd-timesyncd` (default) | `chrony` (default) | Not required |

## Recall Check — 10-io-historical-monitoring

Answer these from memory before continuing — no looking back at Demo 10.

**Q1.** `iostat` shows a device at high `%util`. What does `iostat` alone NOT
tell you, and which tool fills that gap?

**Q2.** What do `sa1` and `sa2` actually do, and how are they scheduled?

**Q3.** `HISTORY=7` is set in `/etc/sysstat/sysstat`. An incident from 10 days
ago needs investigating. What happens to that data, and why does this matter?

<details>
<summary>Answers</summary>

**A1.** `iostat` reports per-device statistics only — it doesn't identify
which process is responsible for the load. `iotop` fills that gap by
attributing I/O to individual processes.

**A2.** `sa1` collects a data sample frequently (traditionally every 10
minutes); `sa2` produces a daily summary report. Both are scheduled via cron
or a systemd timer depending on the distro — the same scheduling mechanisms
covered in Demo 08.

**A3.** It's already been deleted — 10 days is outside the 7-day retention
window. This matters because postmortems often happen well after an incident,
so retention set too short means the data won't exist by the time someone
needs it.

</details>

## Part 1 — Concepts

### 1.1 ip — the modern replacement for ifconfig/route/arp

The `ip` command (from `iproute2`) is the current standard for everything
this demo covers — addresses, links, and routes — replacing the older
`ifconfig`/`route`/`arp` trio, which are deprecated on modern distros and
sometimes not even installed by default. Every subcommand follows the same
pattern: `ip OBJECT COMMAND`, e.g. `ip addr show`, `ip link set`, `ip route
add`.

```bash
ip addr show
# Often abbreviated: ip a
ip link show
# Often abbreviated: ip l
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

For this demo, the practical takeaway is narrower than full IPv6
administration: recognize link-local (`fe80::`) vs global addresses in `ip
addr` output, and know that `-6` flags the same `ip` subcommands to operate
on IPv6 instead of IPv4 — IPv4 and IPv6 run side by side on the same
interface, not as alternatives you choose between.

### 1.4 ip route — the default gateway and static routes

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

```bash
sudo ip route add 10.0.5.0/24 via 172.17.0.1 dev eth0
sudo ip route del 10.0.5.0/24
```

Same persistence caveat as addresses: routes added this way are live-only and
vanish on reboot unless written into the distro's persistent network
configuration.

### 1.5 Hostname resolution — /etc/hosts comes before DNS

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

```bash
cat /etc/hosts
# 127.0.0.1   localhost
# 172.17.0.2  novatech-app-01
```

### 1.6 DNS basics — resolv.conf and dig

```bash
cat /etc/resolv.conf
# nameserver 8.8.8.8
# search novatech.internal
```

`nameserver` lines list DNS servers to query, in order; `search` appends a
domain suffix to unqualified names (querying `db01` actually queries
`db01.novatech.internal` first). On modern systemd-based distros,
`/etc/resolv.conf` is frequently a symlink managed by `systemd-resolved`
rather than a static file — editing it directly can be silently overwritten;
`resolvectl status` shows the actual effective configuration in that case.

```bash
dig novatech.internal
dig +short novatech.internal        # just the answer, no verbose output
dig @8.8.8.8 novatech.internal      # query a specific server directly,
                                     # bypassing resolv.conf — useful for
                                     # isolating "is it my resolver config or
                                     # the DNS server itself"
```

### 1.7 Time synchronization — why it matters more than it seems

Clock drift breaks things that don't look like clock problems: TLS
certificate validation (certs have time-bounded validity), Kerberos
authentication, log correlation across servers, and cron/timer scheduling
accuracy (Demo 08). A machine whose clock has silently drifted can produce
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

### 1.8 OpenSSH — client, server, and key-based authentication

```bash
sudo systemctl status sshd    # Rocky9
sudo systemctl status ssh     # Ubuntu (service name differs)
```

The server's behavior is controlled by `/etc/ssh/sshd_config`. Two settings
worth knowing by name even in a "fundamentals" pass: `PasswordAuthentication`
(whether password login is allowed at all) and `PermitRootLogin` (whether
`root` can SSH in directly) — both get hardened in Demo 26 (Security), but
recognizing them now matters for understanding what key-based auth is
actually replacing.

**Setting up key-based authentication:**

```bash
ssh-keygen -t ed25519 -C "novatech-ops-laptop"
# -t ed25519: modern, fast, secure key type (preferred over older RSA for
# new keys). -C: a comment, purely descriptive, helps identify keys later.

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

## Part 2 — Hands-On Lab

### Commands used in this demo

| Command | Flag meaning | Purpose |
|---|---|---|
| `ip addr show` | — | List interfaces and their addresses |
| `ip addr add ADDR/PREFIX dev IFACE` | — | Add an address (live only, not persistent) |
| `ip route show` | — | List routes, including default gateway |
| `ip route add NET via GW dev IFACE` | — | Add a static route (live only) |
| `dig +short NAME` | `+short`: answer only | Quick DNS lookup |
| `dig @SERVER NAME` | `@SERVER`: query directly | Bypass resolv.conf, isolate resolver vs. server issues |
| `timedatectl status` | — | Check sync status (both distros) |
| `chronyc tracking` | — | Detailed sync offset (chrony/Rocky9) |
| `ssh-keygen -t ed25519` | `-t`: key type | Generate a modern SSH keypair |
| `ssh-copy-id user@host` | — | Install your public key on a remote host correctly |

### Task 1 — Inspect interfaces and addresses

```bash
ip addr show
ip link show
```

Identify: which interface has a global (routable) address, which has only
link-local IPv6, and what the interface's current state (`UP`/`DOWN`) is.

### Task 2 — Add and remove an address (live, non-persistent)

```bash
sudo ip addr add 192.168.100.50/24 dev eth0
ip addr show eth0 | grep 192.168.100.50
sudo ip addr del 192.168.100.50/24 dev eth0
ip addr show eth0 | grep 192.168.100.50   # confirm it's gone
```

### Task 3 — Read routes and identify the default gateway

```bash
ip route show
# Identify the default gateway line and the automatically-added local
# subnet route — which one did you configure, and which did the kernel add?
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

### Task 5 — DNS query and time sync check

```bash
dig +short example.com
dig @8.8.8.8 +short example.com

timedatectl status
# or on Rocky9:
chronyc tracking
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

## What You Learned

1. ✅ Read and configured IPv4/IPv6 addresses and interface state with `ip`
2. ✅ Read and added routes with `ip route`, identified the default gateway
3. ✅ Explained the actual hostname resolution order: `/etc/hosts` before DNS
4. ✅ Queried DNS with `dig`, explained what `/etc/resolv.conf` controls
5. ✅ Checked time synchronization status with `timedatectl`/`chronyc`
6. ✅ Set up SSH key-based authentication and diagnosed a permissions failure

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
   name? Justify using §1.5.
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
   directory and `authorized_keys` permissions against §1.8's rules.
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
Per §1.8, `sshd` refuses to trust `authorized_keys` if its permissions (or
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

## Interview Prep

**Q1. A name resolves correctly via `dig` but a user says the "wrong" server responds when they actually connect. What's your first check?**
`/etc/hosts` on that specific machine. `dig` queries DNS directly and
bypasses the full resolution order; the machine's actual resolution order
(`/etc/nsswitch.conf`, typically `files dns`) checks `/etc/hosts` first. A
stale local entry there silently overrides a correct DNS answer, and this is
one of the most common causes of "DNS looks right but the behavior is wrong"
reports.

**Q2. Why does `ip addr add`/`ip route add` disappear after a reboot, and what should you do instead for a permanent change?**
Because `ip` operates on the kernel's live network state directly — it's an
inspection-and-modification tool, not a configuration-persistence layer.
Permanent changes need to go into the distro's actual network configuration
(Netplan on Ubuntu, NetworkManager profiles/`nmcli` on Rocky9), which is what
gets re-applied on boot; `ip` commands are for live troubleshooting and
temporary changes, not the source of truth.

**Q3. An SSH key-based login fails with "permission denied," falling back to a password prompt. What do you check before assuming the key itself is wrong?**
Permissions on `~/.ssh` (must not be group/world-writable — `700`) and
`~/.ssh/authorized_keys` (`600`) on the target server. `sshd` silently
distrusts overly-permissive permissions on these and falls back to password
auth without an explicit error, which is exactly why this is worth checking
first rather than assuming the key itself is malformed or wasn't copied
correctly.

**Q4. Why does clock synchronization matter for a server that isn't obviously "time-sensitive" — say, a plain web application server?**
Clock drift causes failures that don't look like clock problems: TLS
certificate validation depends on the current time being within the
certificate's validity window, correlating logs across multiple servers
during an incident requires their clocks to actually agree, and any
timer-based scheduling (systemd timers, cron) depends on the system clock
being accurate. A server can run for a long time with a drifted clock before
anyone connects the resulting symptoms back to the actual cause.

## Key Takeaways

1. **`/etc/hosts` is checked before DNS, not after.** `/etc/nsswitch.conf`'s
   `hosts: files dns` order means a stale local `/etc/hosts` entry silently
   overrides a correct DNS answer — `dig` alone can't detect this because it
   queries DNS directly, bypassing that order entirely.
2. **`ip addr`/`ip route` changes are live-only, not persistent.** Permanent
   network configuration lives in the distro's own configuration system
   (Netplan, NetworkManager) — `ip` is for inspection and temporary changes.
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

> **Demo scope:** Primary concept: foundational network configuration and
> connectivity. Supporting concepts: IPv4/IPv6 addressing, ip route, hostname
> resolution order, DNS basics, time synchronization, SSH key authentication.
> Estimated completion time: 37 minutes (reading + hands-on + verification).
> Checkpoints: 6 natural stopping points (after Task 1 through Task 6).

## Quick Commands Reference

### Addressing and links

| Command | Purpose |
|---|---|
| `ip addr show` | List interfaces and addresses |
| `ip addr add ADDR/PREFIX dev IFACE` | Add address (live only) |
| `ip -6 addr show` | IPv6-specific address view |
| `ip link show` | Interface state (UP/DOWN) |

### Routing

| Command | Purpose |
|---|---|
| `ip route show` | List routes including default gateway |
| `ip route add NET via GW dev IFACE` | Add a static route (live only) |

### Hostname resolution and DNS

| Command | Purpose |
|---|---|
| `grep ^hosts: /etc/nsswitch.conf` | Confirm resolution order |
| `getent hosts NAME` | Resolve respecting the full nsswitch order |
| `dig +short NAME` | Quick DNS-only lookup (bypasses /etc/hosts) |
| `dig @SERVER NAME` | Query a specific DNS server directly |

### Time sync

| Command | Purpose |
|---|---|
| `timedatectl status` | Sync status, both distros |
| `chronyc tracking` | Detailed offset (chrony/Rocky9) |

### SSH key setup

| Command | Purpose |
|---|---|
| `ssh-keygen -t ed25519` | Generate a modern keypair |
| `ssh-copy-id user@host` | Install public key correctly on a remote host |
| `chmod 700 ~/.ssh && chmod 600 ~/.ssh/authorized_keys` | Required permissions for key auth to be trusted |

## Appendix A — Anki Cards

**Note:** extracted into a standalone `.csv` by the series extraction script.

```
#deck:Linux Mastery::Module 3 - Networking & Storage::11-networking-fundamentals
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

## Appendix B — Quiz

**Note:** extracted into a standalone `.md` by the series extraction script.

> One correct answer per question unless stated otherwise.
> Target: 80% or above before moving to Demo 12.

**Q1. `dig` shows the correct current answer for a hostname, but a user's browser connects to a different, stale server. What's the most likely explanation?**

- A) DNS caching on the DNS server itself
- B) A stale entry in that machine's `/etc/hosts`, which is checked before DNS
- C) `dig` is unreliable and should not be trusted
- D) The browser has its own separate DNS resolver that ignores the OS entirely

<details>
<summary>Answer</summary>

**B** — `/etc/nsswitch.conf`'s default `hosts: files dns` order means `/etc/hosts` is consulted first; a stale entry there silently overrides a correct DNS answer, which `dig` (querying DNS directly) can't reveal. Trap: A and D invent plausible-sounding but incorrect mechanisms; C wrongly blames the diagnostic tool.

</details>

---

**Q2. You run `ip addr add 10.0.0.5/24 dev eth0` successfully. After a reboot, the address is gone. Why?**

- A) `ip addr add` requires `--persistent` to survive reboot
- B) `ip` modifies the kernel's live network state only — persistence requires the distro's own network configuration system
- C) This is a bug specific to virtualized environments
- D) IPv4 addresses always expire after 24 hours by default

<details>
<summary>Answer</summary>

**B** — `ip` is a live inspection/modification tool, not a configuration-persistence layer; Netplan (Ubuntu) or NetworkManager (Rocky9) own persistent configuration. Trap: A invents a flag that doesn't exist; C and D invent unrelated explanations.

</details>

---

**Q3. An SSH key login fails and falls back to a password prompt. `ls -la ~/.ssh` shows `authorized_keys` at `664` permissions. What's happening?**

- A) `664` is fine; the problem must be the key itself
- B) `sshd` refuses to trust `authorized_keys` because it's group/world-writable, and falls back to password auth without an explicit error
- C) `664` permissions only affect read access, not authentication
- D) The user needs to run `chmod +x` on `authorized_keys`

<details>
<summary>Answer</summary>

**B** — `sshd` silently distrusts overly-permissive `authorized_keys`/`.ssh` permissions as a security measure, falling back to password auth rather than erroring loudly. Trap: A wrongly dismisses a real cause; C understates what the permissions actually gate; D suggests an irrelevant fix — `authorized_keys` doesn't need to be executable.

</details>

---

**Q4. What does `dig @8.8.8.8 novatech.internal` let you isolate that plain `dig novatech.internal` doesn't?**

- A) Whether the domain is registered at all
- B) Whether the problem is local resolver configuration (resolv.conf) versus the DNS server itself, by querying a known server directly
- C) The TTL of every record in the zone
- D) Whether IPv6 is enabled on the local machine

<details>
<summary>Answer</summary>

**B** — Querying a specific server directly bypasses `/etc/resolv.conf` entirely, isolating whether local resolver config or the actual DNS server is the source of a problem. Trap: A, C, and D describe things `dig` can show in other contexts, but not what this specific flag isolates.

</details>

---

**Q5. What does a `/24` CIDR suffix mean?**

- A) 24 total addresses are available on the network
- B) The first 24 bits of the address define the network portion, leaving 8 bits for hosts
- C) The address is valid for 24 hours
- D) 24 is the interface's MTU in a shorthand notation

<details>
<summary>Answer</summary>

**B** — CIDR notation specifies how many leading bits form the network portion; `/24` leaves 8 host bits (256 addresses, minus network/broadcast). Trap: A confuses the prefix length with an address count; C and D invent unrelated meanings for the number.

</details>

---

**Q6. An interface shows `inet6 fe80::42:acff:fe11:2/64 scope link`. What does `scope link` tell you?**

- A) This is a globally routable IPv6 address
- B) This is a link-local address, used only for on-link communication and never routed off the local network
- C) The interface is currently down
- D) IPv6 is misconfigured on this interface

<details>
<summary>Answer</summary>

**B** — `scope link` combined with the `fe80::` prefix identifies a link-local address, which is automatic and never routed beyond the local network segment. Trap: A misreads scope; C and D invent problems that aren't indicated by this output.

</details>

---

**Q7. Why would clock drift on a server cause TLS connections to that server to start failing?**

- A) TLS doesn't use time at all; this wouldn't happen
- B) Certificate validity is time-bounded, and drift can push the current time outside the certificate's valid window as seen by clients or the server
- C) TLS automatically disables itself if NTP isn't running
- D) Drift only affects UDP-based protocols, not TCP-based ones like TLS

<details>
<summary>Answer</summary>

**B** — TLS certificates have "not before"/"not after" validity windows; if a server's (or client's) clock drifts enough, time-based validation can fail even though the certificate itself is otherwise fine. Trap: A and C are factually wrong; D invents an irrelevant protocol distinction.

</details>

---

**Q8. Ubuntu defaults to `systemd-timesyncd` and Rocky9 defaults to `chrony`. What's the practical difference?**

- A) They are functionally identical, just named differently per distro
- B) `timesyncd` is lightweight and client-only; `chrony` is fuller-featured and can also serve time to others
- C) `chrony` only works on RHEL-family distros
- D) `timesyncd` doesn't support NTP at all

<details>
<summary>Answer</summary>

**B** — `timesyncd` syncs to NTP servers but can't act as a server itself; `chrony` can do both and handles larger clock jumps more gracefully. Trap: A understates a real functional difference; C is factually wrong — chrony runs on Ubuntu too, it's just not the default; D is false, timesyncd's entire purpose is NTP client sync.

</details>