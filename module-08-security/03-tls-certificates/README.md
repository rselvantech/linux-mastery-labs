# Demo M08-03 — TLS/Certificate Management

## Overview

This demo covers the certificate lifecycle problems that actually page
someone at 3 AM: a service presenting an incomplete chain, an expired
certificate nobody rotated, and mTLS failing between two internal
services in a way that looks like a network problem until you actually
read the TLS alert. Unlike most of this module, this demo's certificate
chain, its broken-chain error, and its expired-certificate error were all
**built and captured live in this sandbox** — `openssl` is genuinely
available here, so the failures shown are real output, not documentation
paraphrased into a plausible shape.

## Demo Objectives

1. Explain the CA chain model (root → intermediate → leaf) and why a
   server must present the full chain, not just its own certificate
2. Build and inspect a certificate chain with `openssl`
3. Diagnose the single most common TLS chain bug — a server sending only
   its leaf certificate — from the exact error it produces
4. Diagnose an expired-certificate failure and distinguish it from a
   chain-trust failure using the error text alone
5. Explain what mTLS adds over one-way TLS, and where a client-side
   certificate problem actually shows up
6. Explain what renewal automation (e.g. `certbot`) has to actually
   accomplish, and the classic "renewed, but nothing reloaded it" failure

## Directory Structure

```
03-tls-certificates/
├── README.md                         # this file
├── m08-03-tls-certificates-anki.csv  # Anki flashcard deck
└── m08-03-tls-certificates-quiz.md   # standalone quiz
```

## Environments

| Task | Ubuntu 24.04 | Rocky Linux 9.8 | EC2 Amazon Linux |
|---|---|---|---|
| `openssl` | preinstalled | preinstalled | Not required |
| `certbot` | `sudo apt install certbot` | `sudo dnf install certbot` | Not required |

> **Verification note:** §1.1–1.4 (the CA chain build, the missing-
> intermediate error, and the expired-certificate error) were built and
> tested live in this sandbox with real `openssl` commands — the error
> text shown is exactly what `openssl verify` produced, not a
> documentation paraphrase. §1.5's mTLS handshake behavior is described
> from stable, documented `openssl s_client`/`s_server` behavior; a live
> two-process handshake test in this sandbox repeatedly hung the tool
> session and was abandoned rather than force through — treat the exact
> `s_client` output shape there as `[Unverified — test before use]`.
> `certbot` isn't installed here (no network) and §1.6 is entirely
> doc-sourced; recommend confirming it on a real machine.

## Recall Check — 02-selinux-apparmor

Answer these from memory before continuing — no looking back at Demo M08-02.

1. Why can a script succeed when tested manually as a service account,
   but fail when the real systemd service runs the identical command as
   the identical user?
2. Why doesn't `chcon` permanently fix an SELinux context problem?
3. What should you check before generating a custom policy module with
   `audit2allow`?

<details>
<summary>Answers</summary>

1. SELinux context is tied to how the process was launched, not its Unix
   UID — a manual test typically runs unconfined, while the real service
   runs in its own confined domain.
2. `chcon` changes the file immediately, but the policy database's
   recorded rule for that path is unchanged; a relabel reverts it.
   `semanage fcontext` is what survives a relabel.
3. Whether an existing boolean (`getsebool -a`) already covers the exact
   behavior needed.

</details>

## Part 1 — Concepts

### 1.1 The CA chain model — why one certificate isn't enough

A certificate is a signed statement: "this public key belongs to this
name," signed by whoever issued it. That issuer is itself either trusted
directly, or it's a certificate signed by someone else — a chain that
eventually ends at a **root CA**, which every client already trusts
because its certificate ships in the OS/browser's trust store. Almost no
real deployment signs server certificates directly with a root CA;
instead there's an **intermediate CA**, signed by the root, which then
signs the actual server (**leaf**) certificate:

```
Root CA (in every client's trust store, never distributed with the app)
   └── Intermediate CA (signed by root)
          └── Server leaf certificate (signed by intermediate)
```

**The critical operational consequence:** a client only has the root CA
in its trust store — it does **not** already have your intermediate CA.
When your server presents its connection, it must send both its own leaf
certificate **and** the intermediate certificate, so the client can walk
the chain from the leaf it received up to a root it already trusts. If the
server sends only its leaf, the client has a certificate signed by an
issuer it's never seen, with no way to link it to anything it trusts —
even though the certificate itself is completely valid.

### 1.2 Building and inspecting a real chain

```bash
# Root CA — self-signed, long-lived
openssl genrsa -out root-ca.key 2048
openssl req -x509 -new -key root-ca.key -sha256 -days 3650 \
  -subj "/C=US/O=NovaTech/CN=NovaTech Root CA" -out root-ca.crt

# Intermediate CA — signed BY the root
openssl genrsa -out intermediate-ca.key 2048
openssl req -new -key intermediate-ca.key \
  -subj "/C=US/O=NovaTech/CN=NovaTech Intermediate CA" -out intermediate-ca.csr
openssl x509 -req -in intermediate-ca.csr -CA root-ca.crt -CAkey root-ca.key \
  -CAcreateserial -days 1825 -sha256 \
  -extfile <(printf "basicConstraints=critical,CA:TRUE,pathlen:0\nkeyUsage=critical,keyCertSign,cRLSign") \
  -out intermediate-ca.crt
```

Real output confirming the hierarchy:

```
$ openssl x509 -in root-ca.crt -noout -subject -issuer
subject=C = US, O = NovaTech, CN = NovaTech Root CA
issuer=C = US, O = NovaTech, CN = NovaTech Root CA
$ openssl x509 -in intermediate-ca.crt -noout -subject -issuer
subject=C = US, O = NovaTech, CN = NovaTech Intermediate CA
issuer=C = US, O = NovaTech, CN = NovaTech Root CA
```

The root's subject and issuer are identical (self-signed, the definition
of a root). The intermediate's issuer names the root — that link is what
`openssl verify` and every TLS client actually walk.

```bash
# Server leaf — signed BY the intermediate
openssl genrsa -out server.key 2048
openssl req -new -key server.key -subj "/CN=api.internal.novatech.example" -out server.csr
openssl x509 -req -in server.csr -CA intermediate-ca.crt -CAkey intermediate-ca.key \
  -CAcreateserial -days 90 -sha256 -out server.crt
```

The `-extfile`/`basicConstraints=CA:TRUE` on the intermediate is not
decoration: without `CA:TRUE`, nothing would let that certificate sign
anything else at all — a leaf certificate signed by another leaf
certificate is rejected outright by a correctly-behaving client.

### 1.3 The most common chain bug, reproduced

```bash
$ openssl verify server.crt
CN = api.internal.novatech.example
error 20 at 0 depth lookup: unable to get local issuer certificate
error server.crt: verification failed
```

This is real, captured output — verifying the leaf **alone**, with
nothing telling `openssl` about the intermediate. **This is exactly what
happens to a real client connecting to a misconfigured server that only
serves its leaf certificate.** The certificate itself is completely
valid; the client simply has no way to connect it to anything it trusts.

```bash
$ openssl verify -CAfile root-ca.crt -untrusted intermediate-ca.crt server.crt
server.crt: OK
```

Supplying the intermediate (`-untrusted`, meaning "not itself trusted,
but usable to complete a chain to something that is") alongside the
trusted root resolves it immediately — same leaf certificate, same key,
the only difference is whether the verifier was given the intermediate.

```bash
$ openssl verify -CAfile root-ca.crt server.crt
CN = api.internal.novatech.example
error 20 at 0 depth lookup: unable to get local issuer certificate
error server.crt: verification failed
```

Giving it the correct, trusted root **alone**, without the intermediate,
produces the identical `error 20` — proving the missing link specifically
is the intermediate, not a trust problem with the root. This is the
diagnostic signature to look for: `error 20` plus a certificate that
otherwise looks completely legitimate almost always means an incomplete
chain, not an invalid or untrusted certificate.

**The fix in production is almost never regenerating anything** — it's
serving a "full chain" file (leaf certificate, then intermediate,
concatenated) instead of the leaf alone:

```bash
cat server.crt intermediate-ca.crt > server-fullchain.crt
```

```
$ openssl pkcs7 -print_certs -noout < <(openssl crl2pkcs7 -nocrl -certfile server-fullchain.crt)
subject=CN = api.internal.novatech.example
issuer=C = US, O = NovaTech, CN = NovaTech Intermediate CA

subject=C = US, O = NovaTech, CN = NovaTech Intermediate CA
issuer=C = US, O = NovaTech, CN = NovaTech Root CA
```

Order matters: the leaf comes first, then the intermediate — most TLS
stacks expect the chain in that order and some fail or warn if it's
reversed.

### 1.4 Expired certificates — a different error, on purpose

```bash
# Issuing a certificate with an explicit, already-past validity window
openssl ca -config ca.cnf -in server.csr -out server-expired.crt \
  -startdate 20200101000000Z -enddate 20200201000000Z -batch
```

```
$ openssl x509 -in server-expired.crt -noout -dates
notBefore=Jan  1 00:00:00 2020 GMT
notAfter=Feb  1 00:00:00 2020 GMT
$ openssl verify -CAfile root-ca.crt -untrusted intermediate-ca.crt server-expired.crt
CN = api.internal.novatech.example
error 10 at 0 depth lookup: certificate has expired
error server-expired.crt: verification failed
```

Real, captured output — and notice the error **code is different**:
`error 10` (expired), not `error 20` (unable to get local issuer
certificate) from §1.3. **The two failures look superficially similar
("verification failed") but mean completely different things and need
completely different fixes**: `error 20` means the chain is incomplete —
serve the intermediate. `error 10` means the certificate's validity
window has actually passed — it needs to be reissued and the service
needs to actually start using the new one (§1.6's real failure mode).
Reading the specific error code, not just "verification failed," is what
tells you which of these you're actually looking at.

### 1.5 mTLS — verification in both directions

One-way TLS verifies the server to the client; **mutual TLS (mTLS)** adds
the reverse — the server also requires and verifies a certificate from
the client, typically for service-to-service authentication rather than
a human browser.

```bash
# A client certificate, from the SAME intermediate as the server
openssl genrsa -out client.key 2048
openssl req -new -key client.key -subj "/CN=service-b-client" -out client.csr
openssl x509 -req -in client.csr -CA intermediate-ca.crt -CAkey intermediate-ca.key \
  -CAcreateserial -days 90 -sha256 -out client.crt
```

```
$ openssl x509 -in client.crt -noout -subject -issuer
subject=CN = service-b-client
issuer=C = US, O = NovaTech, CN = NovaTech Intermediate CA
```

A server configured to require a client certificate
(`openssl s_server ... -Verify 1 -verify_return_error`) rejects a
connection that presents none, or one it can't verify, with a TLS alert
before any application data is exchanged — from the calling service's
point of view this often looks exactly like a generic connection
failure, not an obviously certificate-related one.
`[Unverified — test before use]`
The general shape a client sees on the failing side:

```
$ openssl s_client -connect internal-service:8443 -CAfile root-ca.crt
...
140234:error:...:SSL routines:...:sslv3 alert bad certificate:...
```

versus a working connection presenting the correct client certificate and
chain:

```bash
openssl s_client -connect internal-service:8443 -CAfile root-ca.crt \
  -cert client.crt -key client.key -cert_chain intermediate-ca.crt
```

**The practical debugging point**: if a service-to-service call fails and
one-way TLS (the server's own certificate) checks out fine, check whether
the far side actually requires a client certificate, and whether *this*
service is presenting one at all — an mTLS failure on the client side
produces no useful error from the client's own perspective beyond "the
connection failed," because the rejection happens inside the TLS
handshake before either side's application code runs.

### 1.6 Renewal automation — and the failure mode that actually matters

`certbot` (or an equivalent ACME client) automates *obtaining and
renewing* a certificate — it does not automatically make every service
using that certificate pick up the new one. This is the real-world
version of §1.4's "expired" scenario, and the most common form it
actually takes in an incident:

```
1. certbot renews the certificate file on disk — genuinely successful
2. nginx/the application was never told to re-read it — still serving
   the OLD certificate from memory, which is now expired
3. The renewal "succeeded" and the outage happens anyway
```

The fix isn't in the renewal step at all — it's ensuring the renewal
process also **reloads the consuming service** (`certbot renew
--deploy-hook "systemctl reload nginx"`, or equivalent), and confirming
that reload actually re-reads the certificate rather than just restarting
without picking up new files from a different path. Verifying a renewal
actually worked means checking what the **running service** currently
presents (`openssl s_client -connect host:443 -CAfile root-ca.crt |
openssl x509 -noout -dates`, or equivalent), not just checking the
certificate file's own expiry date on disk — those two can disagree for
exactly the reason above.

## Part 2 — Hands-On Lab

### Commands used in this demo

| Command | Purpose |
|---|---|
| `openssl genrsa` / `openssl req` | Generate a private key / certificate signing request |
| `openssl req -x509` | Self-sign — used only for a root CA |
| `openssl x509 -req -CA ... -CAkey ...` | Sign a CSR with a given CA, producing a certificate |
| `openssl verify -CAfile ROOT -untrusted INTERMEDIATE LEAF` | Validate a chain the way a real client does |
| `openssl x509 -noout -subject -issuer -dates` | Inspect a certificate's identity and validity window |
| `openssl ca -startdate ... -enddate ...` | Issue a certificate with an explicit (e.g. already-past) validity window |
| `openssl s_client -connect HOST:PORT` | Test a live TLS/mTLS connection |

### Task 1 — Build the chain

```bash
mkdir -p ~/tls-lab && cd ~/tls-lab
openssl genrsa -out root-ca.key 2048
openssl req -x509 -new -key root-ca.key -sha256 -days 3650 \
  -subj "/C=US/O=NovaTech/CN=NovaTech Root CA" -out root-ca.crt

openssl genrsa -out intermediate-ca.key 2048
openssl req -new -key intermediate-ca.key \
  -subj "/C=US/O=NovaTech/CN=NovaTech Intermediate CA" -out intermediate-ca.csr
printf "basicConstraints=critical,CA:TRUE,pathlen:0\nkeyUsage=critical,keyCertSign,cRLSign" > int_ext.cnf
openssl x509 -req -in intermediate-ca.csr -CA root-ca.crt -CAkey root-ca.key \
  -CAcreateserial -days 1825 -sha256 -extfile int_ext.cnf -out intermediate-ca.crt

openssl genrsa -out server.key 2048
openssl req -new -key server.key -subj "/CN=api.internal.novatech.example" -out server.csr
openssl x509 -req -in server.csr -CA intermediate-ca.crt -CAkey intermediate-ca.key \
  -CAcreateserial -days 90 -sha256 -out server.crt
```

Confirm the hierarchy with `openssl x509 -in FILE -noout -subject -issuer`
on all three files.

### Task 2 — Reproduce the missing-intermediate error, then fix it

```bash
openssl verify server.crt
openssl verify -CAfile root-ca.crt server.crt
openssl verify -CAfile root-ca.crt -untrusted intermediate-ca.crt server.crt
cat server.crt intermediate-ca.crt > server-fullchain.crt
```

Confirm the first two commands both fail with `error 20`, and the third
succeeds. This is the single most common real-world TLS misconfiguration
— reproduce it deliberately so the error text is recognizable later.

### Task 3 — Issue and diagnose an expired certificate

```bash
cat > ca.cnf << 'EOF'
[ ca ]
default_ca = CA_default
[ CA_default ]
dir = .
database = ./index.txt
new_certs_dir = .
certificate = ./intermediate-ca.crt
private_key = ./intermediate-ca.key
serial = ./ca-serial
default_md = sha256
policy = policy_any
[ policy_any ]
commonName = supplied
EOF
touch index.txt
echo 1000 > ca-serial

openssl ca -config ca.cnf -in server.csr -out server-expired.crt \
  -startdate 20200101000000Z -enddate 20200201000000Z -batch

openssl verify -CAfile root-ca.crt -untrusted intermediate-ca.crt server-expired.crt
```

Confirm this fails with `error 10`, not `error 20` — a genuinely
different failure than Task 2, even though both say "verification
failed."

### Task 4 — Build a client certificate for mTLS

```bash
openssl genrsa -out client.key 2048
openssl req -new -key client.key -subj "/CN=service-b-client" -out client.csr
openssl x509 -req -in client.csr -CA intermediate-ca.crt -CAkey intermediate-ca.key \
  -CAcreateserial -days 90 -sha256 -out client.crt
openssl x509 -in client.crt -noout -subject -issuer
```

Confirm the client certificate's issuer is the same intermediate CA as
the server's — in mTLS, both sides are commonly issued from the same
internal CA hierarchy.

### Task 5 — Classify three real incidents by error code

```
A: error 20 at 0 depth lookup: unable to get local issuer certificate
B: error 10 at 0 depth lookup: certificate has expired
C: sslv3 alert bad certificate (during a client's connection attempt to a service requiring mTLS)
```

For each, name the actual cause and the fix, without re-reading §1.3-1.5
first.

<details>
<summary>Answers</summary>

**A** — Incomplete chain: the server (or whoever configured the client's
trust store) is missing the intermediate certificate. Fix: serve the full
chain (leaf + intermediate), not the leaf alone.

**B** — The certificate's validity window has actually passed. Fix:
reissue the certificate AND confirm the consuming service actually reloads
it — per §1.6, a successful renewal on disk doesn't guarantee the running
service picked it up.

**C** — An mTLS rejection on the client side: the server required a
client certificate and either didn't receive one or couldn't verify the
one it received. Fix: confirm this service is actually presenting a valid
client certificate and full chain for the connection, not just that the
server's own certificate is fine.

</details>

## What You Learned

- ✅ Explained the CA chain model and why a server must present the full
  chain, not just its own certificate
- ✅ Built and inspected a real certificate chain with `openssl`
- ✅ Reproduced the most common TLS chain bug and identified it from its
  exact error code
- ✅ Issued and diagnosed an expired certificate, distinguishing its error
  code from a chain-trust failure
- ✅ Built a client certificate for mTLS and explained where a client-side
  certificate failure shows up
- ✅ Explained what renewal automation has to actually accomplish beyond
  just replacing the file on disk

## Part 3 — Break-Fix Scenario

### Scenario — NovaTech Ops: mTLS breaks between two internal services

> **Incident report — NovaTech Ops, ticket #5201**
>
> "service-a can't reach service-b. Both services' own certificates were
> checked and are valid and unexpired. The network team confirms the port
> is reachable. `curl` from service-a's host to service-b's port times out
> at the TLS layer specifically, not at the TCP layer."

```bash
$ openssl s_client -connect service-b.internal:8443 -CAfile root-ca.crt < /dev/null
...
verify return:1
...
140234:error:...:SSL alert number 42
```

Tasks:
1. Both services' own certificates are confirmed valid. Reference §1.5 —
   what category of failure does an "alert number 42" (bad certificate)
   during the handshake suggest, given the connection reaches the TLS
   layer before failing?
2. The test above connected without presenting a client certificate at
   all. Is that itself informative, or is it an incomplete test?
3. What's the correct next test, and what would distinguish "service-a's
   client certificate is the actual problem" from "the chain service-a
   presents is incomplete" (§1.3, applied to a client cert instead of a
   server cert)?

<details>
<summary>Answers</summary>

An alert during the handshake itself, after the connection reaches the
TLS layer, points at certificate verification specifically — not a
network, DNS, or general connectivity issue, which the ticket has already
ruled out (TCP connects fine). Given `service-b` requires mTLS, "bad
certificate" here most likely means service-b rejected whatever client
credential — or lack of one — it received from service-a. The test as run
is incomplete: it deliberately connected without presenting any client
certificate at all, which only proves the server enforces mTLS (expected)
— it says nothing about whether service-a's actual client certificate,
when it does present one, is itself the problem. The correct next test
repeats the connection presenting service-a's actual client certificate
and full chain (`-cert client.crt -key client.key -cert_chain
intermediate-ca.crt`, per Task 4): if that succeeds, the real service-a
process is misconfigured (not sending its cert, or sending the leaf
alone without the intermediate — §1.3's error 20 pattern, just on the
client side of an mTLS exchange this time); if it still fails with the
same bad-certificate alert even with the correct client credential
manually supplied, the problem is on service-b's side (an outdated trust
store, or a CA it no longer trusts).

</details>

## Interview Prep

**Q1. A server's certificate is completely valid and unexpired, but clients still reject it with an "unable to get local issuer certificate" error. What's the most likely cause?**

The server is presenting only its leaf certificate, without the
intermediate CA that signed it. The client's trust store only has the
root CA — it can't link a valid leaf certificate to a root it trusts
without the intermediate bridging the two. The fix is serving a full
chain (leaf + intermediate), not reissuing anything.

---

**Q2. What's the practical difference between `error 20` and `error 10` from `openssl verify`, and why does it matter which one you're looking at?**

`error 20` (unable to get local issuer certificate) means the chain is
incomplete — the fix is serving the missing intermediate. `error 10`
(certificate has expired) means the certificate's actual validity window
has passed — the fix is reissuing it and confirming the consuming service
reloads it. Both produce "verification failed," but treating one as the
other wastes time on the wrong fix entirely.

---

**Q3. A certificate renewal (`certbot renew`) reports success, but the service still serves the old, now-expired certificate. What went wrong?**

Renewal only replaces the certificate file on disk — it doesn't
automatically make a running service re-read it. The service needs an
explicit reload/restart tied to the renewal (a `--deploy-hook`, or
equivalent), and verifying the fix means checking what the *running*
service actually presents over the wire, not just the certificate file's
expiry date on disk.

---

**Q4. Two internal services fail to connect over mTLS, and the failure looks like a generic connection problem. What's the debugging approach?**

Confirm the failure is actually happening at the TLS handshake layer
(not TCP/DNS) first. Then test the connection while explicitly presenting
the calling service's actual client certificate and full chain — an mTLS
rejection on the client side produces no detailed error from the calling
service's own perspective, since the rejection happens inside the TLS
handshake before either side's application code ever runs.

---

## Key Takeaways

1. **A server must present its full certificate chain (leaf +
   intermediate), not just its own certificate** — clients only trust the
   root, and need the intermediate to bridge a valid leaf certificate up
   to it.
2. **`error 20` (missing issuer) and `error 10` (expired) are different
   failures needing different fixes**, even though both report
   "verification failed." Read the specific error, not just the summary.
3. **The most common real-world TLS misconfiguration is an incomplete
   chain, not an invalid certificate** — reproducing it (Task 2) makes the
   error text recognizable before it shows up in a real incident.
4. **mTLS adds server-side verification of the client** — a client-side
   certificate or chain problem produces a generic-looking connection
   failure from the calling service, with no detail beyond "the handshake
   failed," because rejection happens before any application code runs.
5. **Renewal automation replacing a file on disk doesn't mean the running
   service is using it.** Verify a renewal by checking what the live
   service actually presents over the wire, and tie renewal to an
   explicit reload of every consumer.

> **Demo scope:** Primary concept: diagnosing and managing the TLS
> certificate lifecycle — chain trust, expiry, and mTLS. Supporting
> concepts: CA hierarchy construction, `openssl verify` error codes,
> client-certificate failures, renewal automation's real failure mode.
> Estimated completion time: 38 minutes (reading + hands-on + verification).
> Checkpoints: 5 natural stopping points (after Task 1 through Task 5).

## Quick Commands Reference

### Building and inspecting a chain

| Command | Purpose |
|---|---|
| `openssl genrsa -out KEY 2048` | Generate a private key |
| `openssl req -x509 -new -key KEY ... -out CERT` | Self-sign (root CA only) |
| `openssl req -new -key KEY ... -out CSR` | Create a signing request |
| `openssl x509 -req -in CSR -CA CA.crt -CAkey CA.key -out CERT` | Sign a CSR with a CA |
| `openssl x509 -in CERT -noout -subject -issuer -dates` | Inspect identity and validity |

### Verifying and diagnosing

| Command | Purpose |
|---|---|
| `openssl verify -CAfile ROOT -untrusted INTERMEDIATE LEAF` | Validate the way a real client does |
| `openssl s_client -connect HOST:PORT -CAfile ROOT` | Test a live TLS connection |
| `openssl s_client ... -cert C -key K -cert_chain I` | Test with a client certificate (mTLS) |

## Appendix A — Anki Cards

**`m08-03-tls-certificates-anki.csv`:**

```
#deck:Linux Mastery::Module 8 - Security::Demo M08-03 - TLS/Certificate Management
#separator:Comma
#columns:Front,Back,Tags
"Why must a server present its intermediate CA certificate, not just its own leaf certificate?","Clients only trust the root CA directly. Without the intermediate, the client has no way to link a valid leaf certificate to any root it trusts — the intermediate is what bridges the two.","demo27b,tls,chain,fundamentals"
"What does openssl verify's error 20 (unable to get local issuer certificate) actually mean?","The certificate chain is incomplete — the verifier wasn't given the intermediate certificate needed to link the leaf to a trusted root. It does NOT mean the certificate itself is invalid or expired.","demo27b,tls,error-20,diagnosis"
"What does openssl verify's error 10 (certificate has expired) mean, and how does it differ from error 20?","The certificate's actual validity window has passed. Unlike error 20 (a chain problem, fixed by serving the intermediate), error 10 requires reissuing the certificate — a genuinely different fix.","demo27b,tls,error-10,diagnosis"
"What's the fix for a server sending only its leaf certificate instead of the full chain?","Concatenate the leaf certificate and intermediate certificate into one 'full chain' file (leaf first, then intermediate) and serve that instead of the leaf alone — no new certificates need to be issued.","demo27b,tls,fullchain,fix"
"A certificate renewal (e.g. certbot) reports success, but the service still serves the old expired certificate. What went wrong?","Renewal only replaces the certificate file on disk — it doesn't automatically make a running service re-read it. The renewal process needs an explicit reload/restart hook, and verification means checking what the RUNNING service presents, not the file's expiry date.","demo27b,tls,renewal,common-mistake"
"What does mTLS add over standard one-way TLS?","Standard TLS verifies only the server to the client. mTLS additionally requires and verifies a certificate FROM the client, typically for service-to-service authentication rather than a human browser.","demo27b,mtls,fundamentals"
"Why does an mTLS failure on the client side often look like a generic connection failure to the calling service?","The rejection happens inside the TLS handshake itself, before either side's application code runs — the calling service gets no detailed application-level error, just a failed connection.","demo27b,mtls,troubleshooting"
"Two services fail to connect over mTLS. What's the correct next diagnostic step, beyond confirming both services' own certificates are valid?","Test the connection while explicitly presenting the calling service's actual client certificate and full chain — this distinguishes a client-side certificate/chain problem from a server-side trust-store problem.","demo27b,mtls,troubleshooting,break-fix"
```

## Appendix B — Quiz

**`m08-03-tls-certificates-quiz.md`:**

````markdown
# Quiz — Demo M08-03: TLS/Certificate Management

> One correct answer per question unless stated otherwise.
> Target: 80% or above before moving to Demo M08-04.

---

**Q1.** A completely valid, unexpired server certificate is rejected by
clients with "unable to get local issuer certificate." What's the most
likely cause?

- A) The certificate's private key is corrupted
- B) The server is presenting only its leaf certificate, without the intermediate CA that signed it
- C) The client's system clock is wrong
- D) The certificate was issued for the wrong hostname

<details>
<summary>Answer</summary>

**B** — This is `error 20`'s signature: a chain problem, not a certificate
validity problem. The fix is serving the intermediate alongside the leaf.

</details>

---

**Q2.** `openssl verify` reports `error 10: certificate has expired` on
one host and `error 20: unable to get local issuer certificate` on
another. Do these need the same fix?

- A) Yes — both mean "verification failed" and are fixed by regenerating the certificate
- B) No — error 10 means the certificate's validity window has passed (reissue it); error 20 means the chain is incomplete (serve the missing intermediate). Different causes, different fixes
- C) Yes — both are always caused by clock skew
- D) No — error 20 is always more severe and requires revoking the certificate

<details>
<summary>Answer</summary>

**B** — Reading the specific error code, not just "verification failed,"
is what tells these two apart.

</details>

---

**Q3.** What does concatenating a leaf certificate and its intermediate
CA into one file accomplish, and in what order should they appear?

- A) It re-signs the leaf certificate with a longer validity period
- B) It creates a "full chain" file a server can serve so clients receive both certificates needed to verify trust; leaf certificate first, then the intermediate
- C) It merges the two into a single certificate with a combined subject
- D) Order doesn't matter — TLS stacks parse certificates in any order

<details>
<summary>Answer</summary>

**B** — Most TLS stacks expect leaf-first ordering and can fail or warn
if it's reversed.

</details>

---

**Q4.** What does mTLS require beyond standard one-way TLS?

- A) A longer certificate validity period
- B) The server also requires and verifies a certificate presented BY the client, not just the reverse
- C) A larger RSA key size
- D) DNS-based certificate validation instead of a CA chain

<details>
<summary>Answer</summary>

**B** — mTLS is verification in both directions; standard TLS only
verifies the server to the client.

</details>

---

**Q5.** A service-to-service call over mTLS fails, and the calling
service's own logs show only a generic "connection failed" with no
certificate-specific detail. Why might that be, even if the actual cause
IS a certificate problem?

- A) mTLS failures are always logged with full detail; this must be a different kind of failure entirely
- B) An mTLS rejection happens inside the TLS handshake itself, before either side's application code runs — so the calling application never gets a detailed, application-level error
- C) Generic errors only happen when DNS resolution fails
- D) This only happens when the client certificate has actually expired, never for a missing chain

<details>
<summary>Answer</summary>

**B** — This is exactly why mTLS failures are often misdiagnosed as
generic network problems — the rejection is invisible to the calling
application's own error handling.

</details>

---

**Q6.** `certbot renew` reports success, but a browser or client
connecting to the service still sees the old, expired certificate. What
should you check?

- A) Whether `certbot` actually has permission to write to the certificate directory
- B) Whether the service that actually serves TLS (nginx, etc.) was ever told to reload and re-read the newly renewed certificate file
- C) Whether the client's own browser cache needs clearing — this is always a client-side caching issue
- D) Whether the certificate was issued by the correct CA

<details>
<summary>Answer</summary>

**B** — Renewal replacing the file on disk doesn't mean the running
service picked it up — a reload/restart tied to renewal is required.

</details>

---

**Q7.** Two internal services fail an mTLS handshake. Both services' own
server-side certificates are confirmed valid. What's the correct next
test?

- A) Assume the problem is network-level and escalate to the network team
- B) Test the connection while explicitly presenting the calling service's actual client certificate and full chain, to distinguish a client-side certificate/chain problem from a server-side trust issue
- C) Regenerate both services' certificates from scratch, since that always resolves mTLS issues
- D) Disable mTLS temporarily as the permanent fix

<details>
<summary>Answer</summary>

**B** — Confirming each service's own server certificate says nothing
about whether the CALLING service is correctly presenting its own client
credential — that's the actual untested variable in an mTLS failure.

</details>

---

Score guide:

| Score | Action |
|---|---|
| 7/7 | Import Anki cards, move to Demo M08-04 |
| 6/7 | Review the wrong answer, then proceed |
| 5/7 | Re-read the relevant section, retry those questions |
| Below 5/7 | Re-read the full demo and redo the walkthrough before proceeding |
````