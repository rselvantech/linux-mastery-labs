# Demo M03-03 — nftables & Production Firewall Patterns

## Overview

Demo M03-02 covered firewall troubleshooting with the tools already present on
a system. This demo covers writing firewall rules from scratch with
`nftables` — the modern replacement for `iptables` on both Ubuntu and Rocky
Linux — and the production patterns that show up in almost every real
firewall configuration: default-deny, rule ordering, rate limiting, and
logging before dropping.

---

## Demo Objectives

By the end of this demo you will be able to:

1. Explain `nftables`'s table/chain/rule structure, and how it differs
   conceptually from `iptables`'s separate chains-per-purpose model
2. Write a default-deny inbound chain with explicit accept rules for
   loopback, established connections, and specific services
3. Explain why rule *order* determines the outcome for overlapping rules —
   first match wins, not "most specific wins"
4. Add rate limiting to a rule with `limit rate`, and explain what `burst`
   controls
5. Log traffic before dropping it, and explain why logging and the action
   are two separate things in the same rule
6. Inspect, test, and safely flush an `nftables` ruleset without locking
   yourself out

---

## Directory Structure

```
03-nftables-firewall/
├── README.md                          # this file
├── m03-03-nftables-firewall-anki.csv  # Anki flashcard deck
└── m03-03-nftables-firewall-quiz.md   # standalone quiz
```

---

## Environments

| Task | Ubuntu 24.04 | Rocky Linux 9.8 | EC2 Amazon Linux |
|---|---|---|---|
| All lab tasks | ✅ | ✅ | Not required |

`nftables` ships as the default firewall backend on both current Ubuntu
and Rocky releases (`iptables` commands are transparently translated to
`nftables` rules under the hood on most current installs, rather than
being a genuinely separate legacy system). Install with `apt install
nftables` / `dnf install nftables` if not already present.

> ⚠️ **Root/capability note:** every command in this demo needs root (or
> `NET_ADMIN` capability) — firewall rules operate on the kernel's own
> packet-filtering tables, not user-space state.

---

## Recall Check

> Answer from memory before reading further. From Demo M03-02.

1. A connection to a service works from the server itself but not from
   another machine. What's the first thing to check?
2. What's the difference between a connection being `REFUSED` and one that
   just times out, in terms of what's happening at the network/firewall level?
3. Which command shows active listening ports along with the process
   holding each one?

<details>
<summary>Answers</summary>

1. Whether a firewall rule is actually blocking the port from outside,
   even though the service itself is listening correctly (bound to the
   right interface, on the right port) — `ss -tlnp` confirms the service
   side, then check the firewall rules separately.
2. `REFUSED` means a packet reached the host and something actively
   rejected it (nothing listening, or an explicit reject rule) — you get a
   response. A timeout means packets are being silently dropped somewhere
   (a `drop` firewall rule, or blocked upstream) — no response at all.
3. `ss -tlnp` — `t` = TCP, `l` = listening, `n` = numeric, `p` = show the
   owning process.

</details>

---

## Part 1 — Concepts

### 12a.1 The table/chain/rule structure

`nftables` organizes rules in three layers:

```
table    A namespace/family container — e.g. `inet` (both IPv4 and IPv6
         in one table, unlike iptables' separate ip/ip6 commands)
  chain  A named list of rules, optionally hooked into a kernel
         netfilter point (input, output, forward)
    rule Individual match-and-action statements, evaluated in order
```

```bash
nft add table inet filter
nft add chain inet filter input '{ type filter hook input priority 0; policy drop; }'
nft add rule inet filter input tcp dport 22 accept
```

A chain's `hook` (`input`, `output`, `forward`) is what actually wires it
into the kernel's packet path — a table/chain with no hook is just an
inert container until something references it.

### 12a.2 Default-deny — build the base first, then add exceptions

The production pattern almost every real firewall follows: start by
dropping everything, then explicitly allow only what's needed:

```bash
nft add table inet filter
nft add chain inet filter input '{ type filter hook input priority 0; policy drop; }'
nft add rule inet filter input ct state established,related accept
nft add rule inet filter input iif lo accept
nft add rule inet filter input tcp dport 22 accept
```

> ⚠️ [VERIFY] Confirmed directly against real kernel enforcement, not just
> syntax: with `policy drop` and only an explicit `iif lo accept` rule
> present, a real loopback connection (netcat client/server on `127.0.0.1`)
> succeeded — traffic that matched the accept rule actually flowed; nothing
> else was allowed.

**Order of the exceptions matters for a practical reason, even under
`policy drop`:** `ct state established,related accept` is placed early so
return traffic for connections you initiated doesn't have to be
individually allowed by port — without it, every outbound connection's
*response* traffic would also need an explicit inbound rule, which defeats
the point of a stateful firewall.

### 12a.3 Rule order — first match wins, not "most specific"

```bash
nft add rule inet filter input tcp dport 8080 counter drop
nft add rule inet filter input tcp dport 8080 counter accept
```

> ⚠️ [VERIFY] Confirmed directly: with the `drop` rule listed first, it's
> the one that actually takes effect for matching traffic — nftables
> evaluates rules top to bottom and stops at the first match within a
> chain, unless a later mechanism explicitly continues evaluation. A
> second, later rule for the identical match never gets a chance to run.

This is a common real mistake: adding a more specific rule *after* a
broader one that already matched, assuming specificity wins. It doesn't —
position in the chain wins. `nft insert rule ... index 0` (or explicitly
rebuilding the chain in the intended order) is how you fix a
wrongly-ordered rule, not by adding another one after it.

### 12a.4 Rate limiting

```bash
nft add rule inet filter input tcp dport 22 limit rate 5/minute accept
```

> ⚠️ [VERIFY] Confirmed directly: this syntax is accepted and appears in
> `nft list ruleset` as `limit rate 5/minute burst 5 packets accept` —
> nftables applies an implicit default `burst` (matching the rate value)
> when none is specified explicitly.

`limit rate 5/minute` allows an average of 5 matching packets per minute;
`burst` (settable explicitly, e.g. `limit rate 5/minute burst 10 packets`)
allows a short burst above the steady rate before throttling kicks in —
useful for legitimate traffic that's naturally bursty (an SSH client
retrying a few times quickly) without allowing sustained abuse.

### 12a.5 Logging before dropping — two separate actions in one rule

```bash
nft add rule inet filter input tcp dport 22 log prefix "SSH-ATTEMPT " counter drop
```

> ⚠️ [VERIFY] Confirmed directly: this rule syntax is accepted and listed
> correctly — `log` and the terminating action (`drop`, `accept`, etc.)
> are separate statements evaluated in sequence within the same rule, not
> alternatives.

`log` doesn't decide the packet's fate — it's a side effect (writes to the
kernel log, visible via `dmesg` or `journalctl -k`) that happens in
addition to whatever action follows it in the same rule. This is why you
can log traffic you're about to drop without needing two separate rules.

### 12a.6 Inspecting and safely resetting a ruleset

```bash
nft list ruleset              # show everything currently active
nft list table inet filter    # show just one table
nft flush ruleset             # remove ALL rules from ALL tables — use with care
```

> ⚠️ **A note specific to this demo's verification:** `nft flush ruleset`
> was used repeatedly while building and testing the examples above, in a
> disposable sandboxed environment with no other services depending on
> existing rules. On a real server, flushing the ruleset removes every
> protection currently in place, instantly — never run this on a system
> you're remotely connected to without a safety net (a scheduled rule
> re-application, or being on console access) in case the new rules lock
> out your own connection.

---

## Part 2 — Hands-On Lab

### Commands used in this demo

| Command | Flag meaning | Purpose |
|---|---|---|
| `nft add table` | — | Create a new table (namespace) |
| `nft add chain` | — | Create a chain, optionally hooked into the kernel packet path |
| `nft add rule` | — | Append a rule to a chain |
| `nft insert rule ... index N` | — | Insert a rule at a specific position, not just append |
| `nft list ruleset` | — | Show every table/chain/rule currently active |
| `nft flush ruleset` | — | Remove everything — use with care, especially on a live system |
| `nft delete table NAME` | — | Remove one specific table |

### Task 1 — Build a default-deny inbound chain

```bash
sudo nft add table inet filter
sudo nft add chain inet filter input '{ type filter hook input priority 0; policy drop; }'
sudo nft add rule inet filter input ct state established,related accept
sudo nft add rule inet filter input iif lo accept
sudo nft add rule inet filter input tcp dport 22 accept
sudo nft list ruleset
```

### Task 2 — Prove loopback traffic works, everything else doesn't

```bash
# In one terminal:
nc -l -p 9999

# In another:
echo "hello" | nc -w1 127.0.0.1 9999
# Should succeed — loopback is explicitly allowed

# Try a port that's NOT in your ruleset from another host on your network,
# if you have one available, and confirm it does NOT connect — this is
# the practical proof default-deny is actually doing something
```

### Task 3 — Reproduce the rule-order footgun

```bash
sudo nft flush ruleset
sudo nft add table inet ordertest
sudo nft add chain inet ordertest input '{ type filter hook input priority 0; policy accept; }'
sudo nft add rule inet ordertest input tcp dport 8080 counter drop
sudo nft add rule inet ordertest input tcp dport 8080 counter accept
sudo nft list chain inet ordertest input
# The drop rule (listed first) is the one that actually applies —
# the accept rule after it never gets evaluated for matching traffic
```

### Task 4 — Rate limiting and logging together

```bash
sudo nft flush ruleset
sudo nft add table inet filter
sudo nft add chain inet filter input '{ type filter hook input priority 0; }'
sudo nft add rule inet filter input tcp dport 22 limit rate 5/minute accept
sudo nft add rule inet filter input tcp dport 22 log prefix "SSH-ATTEMPT " counter drop
sudo nft list ruleset
```

### Task 5 — Safe inspection and cleanup

```bash
sudo nft list ruleset
sudo nft list table inet filter
sudo nft delete table inet filter
sudo nft list ruleset   # confirm it's gone
```

---

## What You Learned

- ✅ Built a default-deny inbound chain and confirmed loopback traffic
  actually worked under it, using a real netcat connection
- ✅ Reproduced the rule-order footgun directly — a `drop` rule listed
  first wins over an `accept` rule for the identical match listed after it
- ✅ Added rate limiting and confirmed nftables' implicit default burst
- ✅ Combined logging and dropping in a single rule, and confirmed `log`
  is a side effect, not an alternative action
- ✅ Inspected, scoped, and safely removed rulesets

**Key Takeaway:** `nftables` evaluates rules top to bottom and stops at
the first match within a chain — specificity doesn't override position.
A default-deny policy plus explicit, correctly-ordered accept rules is the
production pattern for a reason: it's the only structure where the
absence of a rule is automatically safe.

---

## Part 3 — Break-Fix Scenario

### Scenario — NovaTech: the new firewall rule didn't do anything

You're handed this ticket:

> *"We added a rule to block port 8080 from the public interface — a
> service running there had an unpatched vulnerability. The rule is
> definitely in the ruleset (`nft list ruleset` shows it), but traffic on
> 8080 is still getting through."*

```bash
$ nft list chain inet filter input
table inet filter {
	chain input {
		type filter hook input priority filter; policy drop;
		ct state established,related accept
		iif "lo" accept
		tcp dport 8080 accept
		tcp dport 8080 counter drop
	}
}
```

**Your task — answer from what this demo covered before revealing the
resolution:**

1. Why does traffic on port 8080 still get through, given the `drop` rule
   is clearly present in the output?
2. What's the actual fix — and is it adding another rule, or something else?
3. What command would have caught this immediately, before assuming the
   rule "didn't work"?

<details>
<summary>Resolution — open after attempting</summary>

**1. Why it doesn't work:**

Rule order. The `accept` rule for port 8080 is listed *before* the `drop`
rule for the identical match. `nftables` evaluates top to bottom and stops
at the first match — the `accept` rule fires first, and the `drop` rule
below it never gets evaluated for that traffic at all. The rule isn't
broken; it's unreachable given its position.

**2. The fix — reorder, don't just add:**

```bash
sudo nft delete rule inet filter input handle <handle-of-accept-rule>
# then re-add tcp dport 8080 drop BEFORE any accept rule for the same port,
# or use nft insert rule ... index 0 to place it at a specific position
```

Adding yet another rule after the existing two wouldn't fix anything —
the already-first-matching `accept` rule would still win. The fix has to
change the *order*, not add more rules to an already-wrong order.

**3. What would have caught this immediately:**

`nft list chain inet filter input` — actually reading the full rule list
in order, rather than just confirming the new rule's *presence* with a
grep or a quick scan. The rule was there; its position, not its existence,
was the actual problem.

**The lesson:** confirming a firewall rule exists is not the same as
confirming it's effective. Position in the chain determines outcome for
any overlapping match — always read the full ordered list, not just check
for the rule's presence.

</details>

---

## Interview Prep

**Q1. A firewall rule to block a port is confirmed present in the
ruleset, but traffic on that port still gets through. What's the first
thing you check?**

Rule order, not rule existence. `nftables` (and `iptables`) evaluate rules
top to bottom and stop at the first match within a chain — if an `accept`
rule for the same match is listed earlier, it wins regardless of a
correctly-written `drop` rule appearing later. Read the full ordered rule
list, not just confirm the rule's presence.

---

**Q2. Why does a default-deny policy need an explicit `ct state
established,related accept` rule near the top, rather than just allowing
specific inbound ports?**

Without it, return traffic for connections *you* initiated outbound would
also need individual inbound rules by port — defeating the point of a
stateful firewall. `ct state established,related accept` lets response
traffic for connections you started flow back automatically, while still
requiring explicit rules for genuinely new inbound connections.

---

**Q3. What does `limit rate 5/minute` actually guarantee, and what does
`burst` change about that guarantee?**

It caps the *average* rate of matching traffic at 5 per minute over time —
not a hard "never more than 5 in any given minute" ceiling. `burst`
(defaulting to match the rate value if unspecified) allows a short spike
above the steady rate before throttling kicks in, accommodating naturally
bursty legitimate traffic without permitting sustained abuse at a higher
rate.

---

## Key Takeaways

1. **`nftables` structure is table → chain → rule, with a chain's `hook` (input/output/forward) being what actually wires it into the kernel's packet path.** A table/chain with no hook is inert until referenced.
2. **`nftables` evaluates rules top to bottom and stops at the first match within a chain — position determines outcome, not specificity.** Confirmed directly: a `drop` rule listed before an identical `accept` rule wins; the reverse also holds.
3. **Default-deny (`policy drop` plus explicit accepts) is the production pattern because it's the only structure where the absence of a rule is automatically safe** — anything not explicitly allowed is blocked by default, rather than relying on remembering to add a block rule for everything unwanted.
4. **`ct state established,related accept` near the top of a default-deny chain lets return traffic for outbound connections flow automatically** — without it, every response would need its own inbound rule.
5. **`log` is a side effect within a rule, not an alternative to the rule's action — `log prefix "..." drop` logs AND drops the same packet, in sequence, confirmed directly.**
6. **Confirming a firewall rule exists in the ruleset is not the same as confirming it's effective — always read the full ordered rule list**, since an earlier overlapping rule can make a correctly-written later rule completely unreachable.

---

## Quick Commands Reference

### Structure

| Layer | Command |
|---|---|
| Table | `nft add table inet NAME` |
| Chain (hooked) | `nft add chain inet NAME CHAINNAME '{ type filter hook input priority 0; policy drop; }'` |
| Rule | `nft add rule inet NAME CHAINNAME MATCH ACTION` |

### Common rule patterns

| Pattern | Purpose |
|---|---|
| `ct state established,related accept` | Allow return traffic for connections you initiated |
| `iif lo accept` | Allow loopback traffic |
| `tcp dport N accept` | Allow inbound to a specific port |
| `limit rate N/minute accept` | Rate-limit matching traffic |
| `log prefix "TEXT " drop` | Log then drop, in one rule |

### Inspection & cleanup

| Command | Purpose |
|---|---|
| `nft list ruleset` | Show everything active |
| `nft list chain inet TABLE CHAIN` | Show one chain, in order |
| `nft flush ruleset` | Remove everything — use with care |

---

## Appendix A — Anki Cards

**`m03-03-nftables-firewall-anki.csv`:**

```
#deck:Linux Mastery::Module 3 - Networking::Demo M03-03 - nftables and Production Firewall Patterns
#separator:Comma
#columns:Front,Back,Tags

"An nftables chain has an accept rule for tcp dport 8080 listed BEFORE a drop rule for the identical match. Which one actually applies to matching traffic?","The accept rule - nftables evaluates rules top to bottom within a chain and stops at the first match. Position determines the outcome, not specificity - the drop rule below never gets evaluated for that traffic, confirmed directly by testing this exact ordering.","demo12a,nftables,rule-order,footgun"

"Why does a default-deny (policy drop) inbound chain need an explicit ct state established,related accept rule near the top?","Without it, response traffic for connections YOU initiated outbound would also need individual inbound rules by port - defeating the point of a stateful firewall. This rule lets return traffic for already-established connections flow back automatically, while new inbound connections still need explicit rules.","demo12a,nftables,established,stateful"

"What's the nftables table/chain/rule structure, and what does a chain's 'hook' actually do?","table = namespace/family container (e.g. inet covers both IPv4 and IPv6). chain = a named ordered list of rules. A chain's hook (input/output/forward) is what wires it into the kernel's actual packet-filtering path - a table/chain with no hook is just an inert container until referenced by something with a hook.","demo12a,nftables,structure,hook"

"limit rate 5/minute accept is added to a rule. Does this guarantee never more than 5 matches in any given minute?","No - it caps the AVERAGE rate over time, not a hard per-minute ceiling. burst (defaulting to match the rate value if unspecified) allows a short spike above the steady rate before throttling kicks in, accommodating naturally bursty legitimate traffic.","demo12a,nftables,rate-limit,burst"

"A rule reads: tcp dport 22 log prefix \"SSH-ATTEMPT \" counter drop. Does log override drop, or do both happen?","Both happen, in sequence. log is a side effect (writes to the kernel log, visible via dmesg/journalctl -k) that occurs in addition to whatever action follows it in the same rule - it's not an alternative action, so a single rule can log AND drop the same packet.","demo12a,nftables,log,drop"

"A firewall rule blocking a port is confirmed present via nft list ruleset, but traffic still gets through. What should you check FIRST, before assuming the rule is broken?","Rule ORDER, not just presence. Confirm the rule's exact position relative to any other rule matching the same traffic - an earlier accept rule for the identical match makes a correctly-written later drop rule completely unreachable. Read the full ordered chain listing, not just grep for the rule's existence.","demo12a,nftables,troubleshooting,order"

"How do you fix a firewall rule that's correctly written but positioned after a conflicting rule that matches the same traffic first?","Reorder it - don't just add another rule. Delete the misordered rule (nft delete rule ... handle N) and re-add it in the correct position, or use nft insert rule ... index 0 to place it before the conflicting rule. Adding yet another rule after the existing wrong order doesn't fix anything, since the already-first-matching rule still wins.","demo12a,nftables,fix,reorder"

"What's the practical difference between nft list ruleset and nft flush ruleset?","nft list ruleset shows every table/chain/rule currently active with no side effects. nft flush ruleset REMOVES every rule from every table immediately - on a live system this removes all current firewall protection instantly, so it should never be run on a remote system without a safety net in case new rules lock out the connection.","demo12a,nftables,list,flush,safety"

"Why does nftables use an inet table family instead of separate ip/ip6 tables like older iptables usage?","inet covers both IPv4 and IPv6 in ONE table/chain/rule set, rather than requiring separate, duplicated rule sets for each protocol family the way iptables traditionally required ip and ip6 to be managed as genuinely separate command invocations.","demo12a,nftables,inet,ipv4-ipv6"

"A rule is added with policy drop as the chain's default. What happens to traffic that doesn't match ANY rule in the chain?","It's dropped - the chain's policy is the fallback action applied when no individual rule matches. This is exactly why default-deny is the production pattern: anything not explicitly allowed is blocked automatically, rather than relying on remembering to write an explicit block rule for every unwanted case.","demo12a,nftables,policy,default-deny"
```

---

## Appendix B — Quiz

**`m03-03-nftables-firewall-quiz.md`:**

````markdown
# Quiz — Demo M03-03: nftables & Production Firewall Patterns

> One correct answer per question. Target: 80% before Demo M03-04.

---

**Q1.** A chain has `tcp dport 8080 accept` listed before `tcp dport 8080
drop`. What happens to traffic on port 8080?

- A) It's dropped — the more specific/later rule wins
- B) It's accepted — nftables evaluates top to bottom and stops at the
  first match; the drop rule below never gets evaluated
- C) Both rules apply — traffic is accepted then immediately dropped
- D) nftables refuses to load a ruleset with a conflicting duplicate rule

<details>
<summary>Answer</summary>

**B** — Confirmed directly: position determines outcome, not
specificity or which rule was added more recently. The first matching
rule wins and evaluation for that chain stops there.

</details>

---

**Q2.** Why does a default-deny inbound chain need an explicit `ct state
established,related accept` rule?

- A) It's optional — modern nftables handles established connections
  automatically with no rule needed
- B) Without it, response traffic for connections initiated outbound
  would also need individual inbound rules by port, defeating the point
  of a stateful firewall
- C) It only matters for UDP traffic, not TCP
- D) It replaces the need for any other accept rules entirely

<details>
<summary>Answer</summary>

**B** — This rule is what lets return traffic for connections you
started flow back automatically, while genuinely new inbound connections
still require their own explicit rules.

</details>

---

**Q3.** A firewall rule to block a port is confirmed present in `nft list
ruleset`, but traffic still gets through. What should you check first?

- A) Whether the rule syntax has a typo
- B) The rule's exact position relative to any other rule matching the
  same traffic — an earlier overlapping accept rule makes a correct later
  drop rule unreachable
- C) Whether the interface name is spelled correctly
- D) Whether the table name matches the chain name

<details>
<summary>Answer</summary>

**B** — Confirming a rule's presence is not the same as confirming its
effectiveness. Rule order is the first thing to check when a
seemingly-correct rule doesn't appear to work.

</details>

---

**Q4.** `limit rate 5/minute accept` is applied to a rule. What does this
actually guarantee?

- A) Exactly 5 matches per minute, evenly spaced
- B) An average rate capped at 5/minute, with a burst allowance
  (defaulting to match the rate) permitting a short spike above that
  steady rate before throttling
- C) A hard ceiling of 5 matches in any given minute, no exceptions
- D) 5 matches total, then the rule stops matching entirely

<details>
<summary>Answer</summary>

**B** — `limit rate` caps the average over time, not a strict per-minute
ceiling; `burst` is what allows short-term spikes above that average.

</details>

---

**Q5.** A rule reads `tcp dport 22 log prefix "X" drop`. Does `log`
override `drop`, or do both occur?

- A) `log` overrides `drop` — logged packets are not actually dropped
- B) Both occur, in sequence — `log` is a side effect that happens in
  addition to the rule's actual action, not an alternative to it
- C) This syntax is invalid; `log` and `drop` cannot appear in the same rule
- D) Only the first matching packet is logged; subsequent ones are silently dropped

<details>
<summary>Answer</summary>

**B** — Confirmed directly: `log` writes to the kernel log as a side
effect while the rule's terminating action (`drop` here) still applies to
the same packet.

</details>

---

**Q6.** What's the practical difference between `nft list ruleset` and
`nft flush ruleset`?

- A) They're aliases for the same command
- B) `list` shows the current ruleset with no side effects; `flush`
  immediately removes every rule from every table, with no confirmation
- C) `list` requires root; `flush` does not
- D) `flush` only removes rules older than 24 hours

<details>
<summary>Answer</summary>

**B** — `flush ruleset` is a genuinely destructive, immediate action —
on a live system it removes all current firewall protection at once,
which is why it needs to be used carefully, especially on remote systems.

</details>

---

**Q7.** A misordered rule (a correct `drop` rule positioned after a
conflicting `accept` rule for the same match) needs fixing. What's the
correct approach?

- A) Add another `drop` rule after both existing rules
- B) Reorder the existing rules — delete and re-add in the correct
  position, or use `nft insert rule ... index N` — adding more rules
  after an already-wrong order doesn't change which one matches first
- C) Restart the `nftables` service to force re-evaluation
- D) Nothing can be done; the ruleset must be deleted and rebuilt from scratch

<details>
<summary>Answer</summary>

**B** — The fix has to change rule *order*, not just add more rules —
the already-first-matching rule would still win regardless of what's
appended afterward.

</details>

---

| Score | Action |
|---|---|
| 7/7 | Import Anki cards, move to Demo M03-04 |
| 6/7 | Review the wrong answer, then proceed |
| 5/7 | Re-read the relevant section, retry those questions |
| Below 5/7 | Re-read the full demo and redo the walkthrough before proceeding |
````