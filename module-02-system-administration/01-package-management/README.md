# Demo M02-01 — Package Management

## Overview

Every tool you install on a Linux server — nginx, Docker, kubectl,
Python, OpenSSL — arrives through a package manager. The package
manager resolves dependencies, verifies integrity, tracks what is
installed, and handles upgrades and removal cleanly.

This demo covers both package management families you will work
with in production: the Debian family (`apt`/`dpkg` on Ubuntu) and
the RHEL family (`dnf`/`rpm` on Rocky Linux 9 and Amazon Linux
2023). The commands differ but the mental model is the same —
knowing both makes you effective on any server you land on.

The demo also covers two topics that separate junior from senior
engineers: holding packages at a specific version (critical for
Kubernetes nodes, database servers, anything where an unexpected
upgrade breaks the system), and writing correct `Dockerfile`
`RUN` layers for package installation.

---

## Demo Objectives

By the end of this demo you will be able to:

1. Explain what a package actually contains, the package types
   you'll encounter (binary, source, meta, virtual), and why
   `.deb`/`.rpm` are not just zip/tar archives
2. Explain the relationship between a package's own embedded
   metadata and a repository's package index — two related but
   distinct records
3. Query what is installed, search for packages, and inspect
   package metadata on both Ubuntu and Rocky9
4. Install, update, and remove packages cleanly on both distros
5. Hold a package at a specific version and release the hold
6. Add a third-party repository with its GPG key on both distros
7. Verify the integrity of installed package files using `rpm -V`
8. Read and interpret `rpm -V` output flags
9. Write a correct `Dockerfile` `RUN` layer for package
   installation — correct layer ordering, no cache, no unnecessary
   packages, and know when (and how) to safely pin exact versions
10. Explain the relationship between the high-level tool
    (`apt`/`dnf`) and the low-level tool (`dpkg`/`rpm`), and look
    up the equivalent command on either distro family using the
    side-by-side reference
11. Connect package manager install locations back to the FHS
    knowledge from Demo M01-07 — explain why package-manager installs
    split files across `/usr`, `/etc`, `/var`, and `/run`, in
    contrast to the manual and vendor self-contained patterns

---

## Directory Structure

```
01-package-management/
├── README.md                           # this file
├── m02-01-package-management-anki.csv  # Anki flashcard deck
├── m02-01-package-management-quiz.md   # standalone quiz
└── src/
    └── Dockerfile                      # DevOps base image — correct layer pattern
```

---

## Environments

| Task | Ubuntu 24.04 | Rocky Linux 9.8 | EC2 Amazon Linux |
|---|---|---|---|
| apt / dpkg tasks | ✅ | ❌ | ❌ |
| dnf / rpm tasks | ❌ | ✅ Primary | ✅ Same family |
| rpm -V | ❌ | ✅ | ✅ |
| Dockerfile task | ✅ (Ubuntu base image) | ❌ | ❌ |
| Third-party repo (HashiCorp) | ✅ | ✅ | ✅ |

Run Debian-family tasks on Ubuntu. Run RHEL-family tasks on Rocky9.
Note the differences — they reflect real production environments.

---

## Recall Check

> Answer from memory before reading further. These are from Demo M01-10.

1. A new file is created inside `/var/www/project/` by user `alice`.
   The file ends up owned by group `alice` instead of `webteam`.
   What is missing on the directory and what single command fixes it?

2. `ls -l` shows `-rw-r--r--+` on a config file. What does the `+`
   mean and what command do you run to see the full permissions?

3. A CI pipeline fails with "Permission denied" on `deploy.sh`.
   `ls -l` shows `-rw-r--r--`. What is wrong and what is the fix?

<details>
<summary>Answers</summary>

1. The setgid bit is missing on the directory. Fix:
   `chmod g+s /var/www/project/`. With setgid set, new files
   created inside inherit the directory's group (`webteam`)
   instead of the creator's primary group.

2. `+` means an ACL (Access Control List) is set on the file.
   Standard permissions are only part of the picture. Run:
   `getfacl /path/to/file` to see all named user and group entries,
   the mask, and any default ACLs.

3. No execute bit. `644` gives read/write to owner and read to
   others — no execute for anyone. A shell cannot run a script
   without the execute bit. Fix: `chmod 755 deploy.sh` or
   `chmod +x deploy.sh`.

</details>

---

## Part 1 — Concepts

### 1.0 What is a package

A **package** is a single archive file that bundles everything
needed to install one piece of software: the program's files, the
information describing it, and instructions for installing or
removing it cleanly. This is the fundamental unit every package
manager works with — understanding what's actually inside one
makes every later topic (dependency resolution, verification,
`rpm -V`, GPG signing) make sense as "operations on this archive,"
rather than separate unrelated features.

**What a package contains:**

- The compiled binary or script files — the actual program
- Default configuration files
- **Metadata** — structured information *about* the package itself:
  name, version, architecture, description, maintainer, and its
  dependency list (what it needs to run). This is distinct from
  the **repository index** covered later in this demo (section 1.2)
  — that index is a separate manifest file the repository
  publishes, built by aggregating this kind of metadata across
  every package it hosts, plus a checksum for each entry. This
  package's own copy of its metadata travels embedded inside the
  package file itself and remains there after install, readable
  later from the local install database.
- Pre/post install and removal scripts — run automatically by the
  package manager at the right moment (e.g. creating a service
  account before files are placed, restarting a service after
  upgrade)
- A **file manifest** — the complete list of every file this
  package owns and the exact path each one installs to

**Package archive formats — and why they are not just zip/tar:**

A `.deb` file is an `ar` archive; a `.rpm` file is a custom binary
format wrapping a `cpio` archive. Both bundle the file content
together with the metadata, scripts, and manifest above — which is
what lets a package manager resolve dependencies and track
ownership. A plain `.zip` or `.tar` only stores files; it has no
concept of dependencies, install scripts, or "which package owns
this file later." Full breakdown of each format's internal
structure: **Appendix D, section D.1**.

**Package types you will encounter:**

```
Binary package    Pre-compiled, ready to install as-is — what
                   apt/dnf install by default (nginx, curl, jq).
                   Fastest path; what this entire demo focuses on.

Source package     Contains source code, not compiled binaries —
                   you compile it yourself for your exact system.
                   Debian: .dsc + .orig.tar.* + .debian.tar.*
                   RHEL:   .src.rpm ("SRPM")
                   Rare in day-to-day DevOps work; mostly used when
                   no binary package exists for your architecture,
                   or you need a custom compile-time option.

Meta-package        Contains no files of its own — exists purely to
                   pull in a group of other packages as dependencies.
                   Example: a "build-essential"-style package that
                   depends on gcc, make, and friends, just to give
                   you one thing to install.

Virtual package     A name a package manager recognizes that isn't a
                   real package — represents a capability multiple
                   real packages can satisfy (e.g. several different
                   mail-server packages can all "provide" mail-transport-agent).
```

**Package-related data — where it lives, beyond the package file
itself:**

The package **file** (`.deb`/`.rpm`) is only half the picture. Once
a package is installed, its metadata and manifest are also written
into the package manager's own **database** — a separate, persistent
record on disk, independent of the original package file:

```
dpkg database     /var/lib/dpkg/status      (plain text, one
                   stanza per installed package) and
                   /var/lib/dpkg/info/      (per-package scripts
                   and file lists)

rpm database       /var/lib/rpm/rpmdb.sqlite  (a real SQLite
                   database — full structure in Appendix D,
                   section D.3)
```

This local database is what every query command in this demo
actually reads — `dpkg -l`, `rpm -qi`, `rpm -V` — none of them
re-read the original `.deb`/`.rpm` file after install; they all
read this database. This distinction matters later: the package
**index** (section 1.2/1.5) describes what's *available in a
repository*; this **database** describes what's *actually installed
on this system right now*. Two different records, two different
purposes — covered side by side once both are introduced.

### 1.1 What a package manager does

A package manager is the tool that acts on packages — it finds
them in a repository, resolves what else they need, installs them
correctly, and keeps a record of what's on the system so it can
update or remove them cleanly later. Everything below is that job
broken into its specific responsibilities.

A **package manager** handles:

```
Dependency resolution   You ask for nginx. nginx needs pcre2, zlib, openssl.
                        The package manager resolves and installs the full tree.

Integrity verification  Every package is GPG-signed by the repo maintainer.
                        The package manager verifies the signature before
                        installing — ensures you are installing what you intended.

Version tracking        Keeps a database of every installed package and version.
                        dpkg database: /var/lib/dpkg/
                        rpm database:  /var/lib/rpm/

Clean removal           Knows which files belong to which package — removes
                        exactly those files, leaves files from other packages alone.

Upgrade management      Can upgrade one package, all packages, or hold specific
                        ones at a pinned version while others upgrade freely.
```

**High-level vs low-level tools — the relationship:**

```
Debian family:
  apt        High-level — resolves deps, fetches from repos, calls dpkg
  apt-get    Older high-level — same as apt, less user-friendly output
  dpkg       Low-level — installs/removes .deb files, manages database
             Does NOT resolve dependencies — install deps manually or use apt

RHEL family:
  dnf        High-level — resolves deps, fetches from repos, calls rpm
  yum        Legacy — replaced by dnf in RHEL 8+. Still available as alias.
  rpm        Low-level — installs/removes .rpm files, manages database
             Does NOT resolve dependencies
```

Think of it as: `apt` drives `dpkg`. `dnf` drives `rpm`. You use
the high-level tool for day-to-day work. You use the low-level tool
for queries, verification, and situations where the high-level tool
cannot help (e.g. verifying which package owns a specific file).

**Connecting back to the filesystem — where package managers actually put things:**

Every directory a package manager writes to is one you already
learned the purpose of in Demo M01-07 (FHS). A package manager doesn't
invent new locations — it follows the same map, and specifically
uses the **package manager install pattern** from Demo M01-07's three
installation patterns (section 1.6) — the one where files are
split across multiple FHS locations by purpose, rather than kept
together in one place:

```
/usr/bin/, /usr/sbin/    Binaries the package manager installs
                         (e.g. nginx, jq, curl) — Demo M01-07: this is
                         where package-manager-installed binaries
                         belong, as opposed to /usr/local/bin/
                         for manually installed tools, or /opt/
                         for vendor self-contained installs (below).

/etc/<package-name>/     Configuration files the package ships
                         with defaults for (e.g. /etc/nginx/) —
                         Demo M01-07: every service's config lives
                         under /etc/, regardless of which package
                         manager installed it.

/var/log/<package-name>/ Logs the installed service writes —
                         Demo M01-07: /var/log/ is where persistent,
                         growing log data belongs.

/var/lib/<package-name>/ Application state the service needs to
                         persist (databases, caches that must
                         survive a reboot) — Demo M01-07: /var/lib/
                         is for this kind of persistent app data,
                         distinct from /var/log/.

/run/<package-name>.pid  PID files for running services — Demo M01-07:
                         /run/ is tmpfs, cleared on every reboot,
                         which is exactly right for a PID file —
                         if the service isn't running, the PID
                         file shouldn't survive a reboot either.

/var/lib/dpkg/           The package manager's OWN database — also
/var/lib/rpm/            under /var/lib/, for the same FHS reason:
                         this is persistent application state, in
                         this case the package manager's own
                         record of what's installed.
```

**Why apt/dnf installs land split across FHS, while some software doesn't — the `/opt` contrast from Demo M01-07:**

Demo M01-07 covered three installation patterns side by side. The table above is entirely about the first:

```
Package manager install (apt/dnf) — what this entire demo covers:
  binary   → /usr/bin/nginx
  config   → /etc/nginx/
  logs     → /var/log/nginx/
  data     → /var/lib/nginx/
  Split across multiple FHS locations, by purpose — the standard way

Manual binary install — covered in Demo M01-07, not this demo:
  binary   → /usr/local/bin/kubectl
  Just a single binary file, no separate config or data location

Vendor/self-contained install — covered in Demo M01-07, not this demo:
  everything → /opt/vendorname/
  binary, config, data, libraries all kept together in one tree
  Jenkins, Atlassian tools, some commercial software
```

A package you install with `apt`/`dnf` will essentially never put
anything in `/opt` — that location is reserved for vendor
installers that deliberately bypass the FHS split (their own
installer script, a `.tar.gz` you extract yourself, a Docker-style
all-in-one bundle) rather than going through a package manager at
all. If you ever see `apt install` or `dnf install` putting files
in `/opt`, that's unusual and worth investigating — it isn't the
package manager's normal behavior.

**Why this matters in practice:** when something goes wrong with
an installed service, you don't need to memorize a separate set of
"package manager locations" — you already know the FHS map from
Demo M01-07, and the package manager simply respects it. Quiz Q7 in
Demo M01-07 tested exactly this: nginx's config, PID file, log, and
binary locations — all four are predictable once you know FHS,
regardless of whether `apt` or `dnf` put them there. The same
quiz question also included a vendor-install example (Jenkins →
`/opt/jenkins/`) specifically to contrast against the package
manager pattern this demo focuses on.

### 1.2 apt and dpkg — Debian family

**`apt`** = **A**dvanced **P**ackage **T**ool. The standard interface
for package management on Ubuntu and all Debian-family systems.

**The repository index — what it is, how it goes stale, and where it lives:**

A **repository index** (also called the **package index** — both
terms refer to the same file) is a manifest listing every package
the repository offers: its version, dependencies, and a checksum
for each one — for Debian this is the `Packages` file, wrapped in
a signed `Release`/`InRelease` file. This is distinct from the
metadata embedded inside an individual package (introduced in
section 1.0) — the index is the repository's combined catalog
built from many packages' metadata, not the metadata of any one
package by itself. `apt` does not query the repo live for every
command — it keeps a **local cached copy** of the index at
`/var/lib/apt/lists/`, and reads only that copy for `search`,
`show`, and `install`.

This copy goes stale the moment the maintainer publishes a new
package version after your last `apt update`. Concretely: if
`nginx 1.26.1` is published an hour after your last `update`, your
local index still says `1.25.x` is newest. Running `apt install
nginx` now installs `1.25.x` — not because `apt` is wrong, but
because it is working from outdated information it has no way to
know is outdated. Impact: you believe you are getting the latest
release and you are not; in a Dockerfile this is exactly the
cache-busting failure covered in section 1.7.

**`apt update` vs `apt upgrade` — these do different things, and
the names invite confusion:**

```
apt update    Refreshes the LOCAL INDEX only. Installs or upgrades nothing.
              Re-downloads Packages/Release files to /var/lib/apt/lists/.

apt upgrade   Installs newer versions of packages ALREADY INSTALLED,
              based on whatever the local index currently says —
              which may itself be stale if you have not run update first.
```

Running `apt update` alone will often report something like
`17 packages can be upgraded` — it found that newer versions exist,
but it installs nothing. A separate `apt upgrade` is required to
actually install them. Skipping `update` entirely means every
`install`/`search`/`show` operates on whatever index already
happens to be on disk — possibly weeks old, or simply absent on a
fresh container, which is why a Dockerfile's first apt instruction
is always `apt-get update`.

**Does `apt install nginx` get the latest version? How does it know?**

Yes — it installs the highest version available in the local index
that satisfies any pinning rules in place (default: highest
available, called the **candidate** version). `apt-cache policy
packagename` shows exactly this: a `***` marks the candidate
version, with all other known versions listed below it by priority.

**Does `apt search` hit the network?**

No — `apt search` is entirely local, querying the same cached
index in `/var/lib/apt/lists/` that `install` uses. No network call
happens during `search` itself; only `update` touches the network.
Use case: quickly find a package name you don't know exactly
(`apt search "json processor"`) without a network round-trip each
time, safe to run repeatedly.

**Core apt commands:**

```bash
apt update                      # refresh local package index from repos
                                # run this before any install — index may be stale
                                # does NOT upgrade anything — only updates the list

apt install nginx               # install nginx and all its dependencies
apt install nginx=1.24.0*       # install specific version
apt install -y nginx            # -y = yes — skip confirmation prompt
                                # use in scripts and Dockerfiles

apt remove nginx                # remove binary — leave config files
apt purge nginx                 # remove binary AND config files
                                # use purge when reinstalling fresh

apt upgrade                     # upgrade all packages to latest available version
apt upgrade nginx               # upgrade one specific package

apt search nginx                # search package names and descriptions
apt show nginx                  # show package metadata, dependencies, description
apt list --installed            # list all installed packages
apt list --installed | grep nginx   # check if specific package installed

apt autoremove                  # remove packages installed as dependencies
                                # that are no longer needed by anything
                                # run after remove/purge to clean up

apt clean                       # delete downloaded .deb files from cache
                                # /var/cache/apt/archives/ — frees disk space
```

**`dpkg`** = **D**ebian **P**ack**a**ge. Low-level tool — direct
access to the package database without network or dependency resolution.

**Core dpkg commands:**

```bash
dpkg -l                         # list all installed packages
dpkg -l nginx                   # list specific package (shows version, status)
dpkg -i package.deb             # install a local .deb file
dpkg -r nginx                   # remove (keep config files)
dpkg -P nginx                   # purge (remove + config files)
dpkg -S /usr/sbin/nginx         # which package owns this file?
dpkg -L nginx                   # list all files owned by this package
dpkg --get-selections           # get all package states (installed/hold/etc)
```

**`dpkg --get-selections` explained:**

This lists every package dpkg knows about along with its
**desired state** — `install`, `hold`, `deinstall`, or `purge` —
not its current actual status. This is the mechanism `apt-mark
hold` is built on: running `apt-mark hold curl` and then `dpkg
--get-selections | grep hold` shows `curl   hold` — `apt-mark` is
a friendly wrapper around `dpkg --set-selections`.

**Package states — the codes you will see in real `dpkg -l` output:**

Every `dpkg -l` listing begins with this legend, then shows three
status characters per package:

```
Desired=Unknown/Install/Remove/Purge/Hold
| Status=Not/Inst/Conf-files/Unpacked/halF-conf/Half-inst/trig-aWait/Trig-pend
|/ Err?=(none)/Reinst-required (Status,Err: uppercase=bad)
||/ Name                Version              Architecture  Description
+++-===================-====================-============-============
ii  curl                8.5.0-2ubuntu10.9    amd64         command line tool...
```

Read the three leading characters as **three independent codes
stacked vertically**, not one symbol:

| Position | Meaning | Common values |
|---|---|---|
| 1 | Desired state (what *should* happen) | `i`=install, `h`=hold, `r`=remove, `p`=purge |
| 2 | Actual status (what state it *is in*) | `i`=installed, `c`=config files only, `n`=not installed |
| 3 | Error flag | usually blank — uppercase letters mean something is broken |

**The states you will actually encounter:**

| Code | Meaning |
|---|---|
| `ii` | Desired install + actually installed = healthy, normal — almost every package |
| `rc` | Desired remove + config files remain — the state right after `apt remove` (before `purge`) |
| `pn` | Purged, nothing left — after `apt purge`, or the entry disappears from `dpkg -l` entirely |
| `hi` | Held + installed — a package under `apt-mark hold` |

### 1.3 dnf and rpm — RHEL family

**`dnf`** = **D**andified **Y**um. The standard package manager for
RHEL 8+, Rocky Linux 9, Fedora, and Amazon Linux 2023. Replaced
`yum` (which remains as a compatibility alias on some systems).

**Why "Dandified"?** `yum` itself stood for **Y**ellowdog
**U**pdater, **M**odified. "Dandified" is a deliberately
tongue-in-cheek name signaling `dnf` is a cleaner, modernized
rewrite of `yum` — a faster dependency solver (`libsolv`), better
plugin architecture, more maintainable codebase — not a
fundamentally different tool philosophically. Same job, better
engineering underneath.

**No `dnf purge`?** Correct — there is no separate purge command.
`dnf remove` typically removes config files along with the binary
for most packages, because RPM packaging convention does not split
remove-vs-purge the way Debian's packaging convention does. There
is no missing feature here, just a different default behavior.

**Core dnf commands:**

```bash
dnf install nginx               # install nginx and dependencies
dnf install nginx-1.24.0        # install specific version
dnf install -y nginx            # -y = skip confirmation

dnf remove nginx                # remove package (may leave config files)
dnf autoremove                  # remove unused dependency packages
                                # Why/when: installing a package pulls in
                                # dependencies automatically. Removing the
                                # package later leaves those dependencies
                                # behind, unused. Run autoremove right after
                                # any remove to clean them up — same purpose
                                # as apt autoremove.

dnf upgrade                     # upgrade all packages
dnf upgrade nginx               # upgrade one package
                                # NOTE: "dnf update" also works — it is an
                                # accepted ALIAS for upgrade on dnf. This is
                                # a common point of confusion coming from apt,
                                # where update and upgrade are STRICTLY
                                # different operations (see section 1.2).
                                # On dnf, "update" can install/upgrade things.

dnf search nginx                # search package names and summaries
dnf info nginx                  # detailed package information
dnf list installed              # list all installed packages
dnf list installed | grep nginx # check specific package

dnf provides /usr/sbin/nginx    # which package provides this file?
                                # More powerful than dpkg -S: dnf provides can
                                # search repo metadata for a NOT-YET-installed
                                # package too, not just installed files.
                                # apt has no direct equivalent for the
                                # not-yet-installed case (apt-file exists but
                                # is a separate tool, not installed by default).

dnf history                     # list all past transactions (installs, removes)
dnf history info 5              # detail of transaction 5
dnf history undo 5              # undo transaction 5 — reinstall removed or
                                # remove installed packages from that transaction
```

**`dnf history` — no apt equivalent.** apt keeps no interactive
transaction browser; the closest you can do on Debian/Ubuntu is
manually reading `/var/log/apt/history.log`.

**`dnf history undo`** — the ability to undo a transaction — is a
significant advantage over `apt`. "A botched upgrade" simply means
an upgrade that broke something — a kernel update that fails to
boot, or a package bump that breaks a dependent service. `dnf
history undo N` reverses that specific transaction with one
command. **apt has no equivalent at all** — real-world recovery on
Debian/Ubuntu relies on filesystem snapshots (LVM, ZFS, Timeshift)
or manually reinstalling known-good version numbers, not a
built-in rollback.

**`rpm`** = **R**PM **P**ackage **M**anager (recursive). Low-level
tool on RHEL family.

**Core rpm commands:**

```bash
rpm -qa                         # query all installed packages
rpm -qa | grep nginx            # check if specific package installed
rpm -qi nginx                   # query info — metadata for installed package
rpm -ql nginx                   # query list — all files in this package
rpm -qf /usr/sbin/nginx         # query file — which package owns this file?
rpm -qp package.rpm             # query a local .rpm file (not installed)
rpm -ivh package.rpm            # install local .rpm file (verbose + progress)
rpm -e nginx                    # erase (remove) package
rpm -V nginx                    # verify — check package file integrity
```

### 1.4 Holding packages at a specific version

**Why this matters in production:**

Kubernetes requires all nodes in a cluster to run the exact same
`kubelet`, `kubeadm`, and `kubectl` version as the control plane.
An unintended `apt upgrade` or `dnf upgrade` on a worker node
that bumps `kubelet` from `1.29` to `1.30` will cause that node
to be rejected by the control plane. The node stops accepting pods.

Database servers (MySQL, PostgreSQL), message brokers (Kafka,
RabbitMQ), and any service with strict version compatibility
requirements need the same protection.

**Debian family — `apt-mark`:**

```bash
# Hold a package at its current version
apt-mark hold nginx
# nginx set on hold.

# View all held packages
apt-mark showhold

# Release the hold (allow upgrades again)
apt-mark unhold nginx

# What held packages look like in dpkg
dpkg --get-selections | grep hold
# nginx    hold
```

When a package is on hold, `apt upgrade` skips it and prints:
```
The following packages have been kept back:
  nginx
```

**What happens to `apt update` when a package is held?** Nothing —
`update` only refreshes the local index; it never touches installed
packages. A hold only affects `upgrade`. This trips people up
because it's natural to assume "held" means "frozen from all apt
activity," but the index for a held package still refreshes
normally — only the upgrade step is blocked.

**Alternative — pin with `/etc/apt/preferences.d/`:**

```
# /etc/apt/preferences.d/nginx-pin
Package: nginx
Pin: version 1.24.*
Pin-Priority: 1001
```

**What the priority number actually means:**

```
>= 1000   install/upgrade this version even if it means a downgrade
990–1000  default priority for the target release
500       default priority for any version — normal apt behavior
< 0       never install this version
```

Pin-Priority > 1000 means "prefer this version even if a newer
one is available and would normally be upgraded." Pinning is more
flexible than hold — you can pin to a version pattern (`1.24.*`)
rather than one exact version, and `apt-mark`/preferences-pinning
are the only two tools you need on Debian — there is no separate
plugin to install, unlike dnf below.

**RHEL family — `dnf versionlock`:**

```bash
# Install the versionlock plugin (if not present)
dnf install -y python3-dnf-plugin-versionlock

# Lock a package at its current installed version
dnf versionlock add nginx

# List all locked packages
dnf versionlock list

# Remove the lock (allow upgrades again)
dnf versionlock delete nginx

# Or remove all locks
dnf versionlock clear
```

**Why is this a separate plugin, not built in?** `dnf` ships a
minimal core; advanced features like versionlock are implemented
as **plugins** — separately packaged (`python3-dnf-plugin-
versionlock`, written in Python since dnf itself is Python-based).
This is an architectural difference from apt, where holding is a
built-in dpkg feature with no extra install step.

When a locked package would be upgraded, `dnf upgrade` skips it:
```
Skipping packages with conflicts:
  nginx  (versionlock)
```

**Versionlock vs hold — Debian vs RHEL, side by side:**

| | Debian (`apt-mark hold`) | RHEL (`dnf versionlock`) |
|---|---|---|
| Built-in? | Yes, no extra install | No — separate plugin install required |
| Granularity | Whole package locked at current version | Version pattern (e.g. `7.76.1-40.el9.*`) |
| Storage | dpkg selections database | Separate versionlock plugin config file |
| List locked | `apt-mark showhold` | `dnf versionlock list` |
| Release | `apt-mark unhold` | `dnf versionlock delete` |

Similarity: both ultimately block `upgrade` from touching the named
package; both operate per-package, not system-wide.

### 1.5 Repository management

A **repository** (repo) is a server hosting packages and metadata.
The package manager reads an index from the repo, resolves what to
download, verifies signatures, and installs.

**What "the index" means at the repo level:** the index is the
manifest file the repo publishes — for Debian, the `Packages`
file wrapped in a signed `Release`/`InRelease`; for RHEL,
`repodata/repomd.xml` plus the files it references. This is the
same index concept introduced in section 1.2 for apt specifically
— here it's the general repo-side concept both families implement
differently. The index is what `search`/`show`/`info` actually
query — never the live repo file listing itself.

**Debian — repo configuration:**

```
/etc/apt/sources.list           legacy single file (avoid editing directly)
/etc/apt/sources.list.d/        directory — one .list file per repo (correct approach)
/etc/apt/trusted.gpg.d/         GPG keys for repo signature verification
/etc/apt/keyrings/              modern alternative for GPG keys (Ubuntu 22.04+)
```

A `.list` file format:
```
deb [signed-by=/etc/apt/keyrings/hashicorp.gpg] https://apt.releases.hashicorp.com noble main
↑   ↑                                            ↑                               ↑     ↑
│   signed-by= tells apt which key to use        URL of the repo                 dist  component
│   without signed-by, apt uses the global
│   trusted.gpg keyring (less secure)
deb = binary packages (deb-src = source packages)
```

**RHEL — repo configuration:**

```
/etc/yum.repos.d/               directory — one .repo file per repository
```

A `.repo` file format:
```ini
[hashicorp]
name=HashiCorp Stable - $basearch
baseurl=https://rpm.releases.hashicorp.com/RHEL/$releasever/$basearch/stable
enabled=1
gpgcheck=1
gpgkey=https://rpm.releases.hashicorp.com/gpg
```

**`.list` vs `.repo` — content difference, not just syntax:**

A `.repo` file is **self-contained** — the GPG key URL, the
enable/disable flag, and the metadata source all live in one file.
A `.list` file only has the repo URL; the GPG key reference points
to a *separate* keyring file you create yourself (the `signed-by=`
target), and there is no per-repo `gpgcheck`-style toggle the way
RHEL has it — Debian's verification is implicit via whether
`signed-by=` resolves to a valid key.

**GPG key import — why it is mandatory:**

Every package in a **signed repository** — one whose index is
GPG-signed by the maintainer, true of essentially all production
repos — is checked against that signature before install. GPG
(GNU Privacy Guard) implements public-key cryptography: the
maintainer holds a private key and signs the index with it; anyone
holding the matching public key can verify that signature, proving
the index has not been tampered with and really came from that
maintainer. GPG does not encrypt the package — it signs the index.
Full mechanics of how signing/verification actually works
cryptographically: **Appendix D, section D.2**.

If the key is not imported, installation fails with `NO_PUBKEY` or
`GPG check FAILED` — this is not optional ceremony, it is the only
guarantee that the package you are installing is the one the
maintainer published and has not been tampered with in transit.

**How does a system get the public key in the first place?** Two
ways: pre-bundled with the distro (Ubuntu and Rocky ship with their
own official repo keys already trusted out of the box), or fetched
manually for third-party repos — exactly the `curl`/`rpm --import`
commands below. There is no central global keyserver that
package managers query automatically; you, or the vendor's install
script, must explicitly fetch and import the key.

```bash
# Debian — import GPG key to a named keyring file
curl -fsSL https://apt.releases.hashicorp.com/gpg \
  | gpg --dearmor \
  | sudo tee /etc/apt/keyrings/hashicorp.gpg > /dev/null
# gpg --dearmor converts ASCII armored key to binary format apt expects

# RHEL — rpm imports keys directly into the RPM database
sudo rpm --import https://rpm.releases.hashicorp.com/gpg
# or from a local file:
sudo rpm --import /path/to/key.gpg
```

**`curl -fsSL` flags, decoded:** `-f` fails silently on HTTP errors
instead of printing an HTML error page as if it were the key; `-s`
silences the progress meter; `-S` shows the error anyway if one
occurs even with `-s`; `-L` follows redirects. This exact
combination is the de facto standard for "fetch this URL cleanly
inside a script."

**What does `gpg --dearmor` do?** GPG keys are often distributed
ASCII-armored (the human-readable `-----BEGIN PGP PUBLIC KEY
BLOCK-----` text format). `apt` needs the key in **binary** format
to use as a keyring file, so `--dearmor` converts ASCII-armored to
binary. `rpm --import` accepts both formats internally, which is
why Debian needs this explicit conversion step and RHEL does not.

**Is `rpm --import` the only way to import on RHEL? Debian
equivalent?** Yes, `rpm --import` is the standard mechanism on
RHEL family — see the rpm command reference table in Quick
Commands Reference, where it is listed alongside `-qa`/`-qi`/etc.
Debian's conceptual equivalent is the `gpg --dearmor | tee
/etc/apt/keyrings/name.gpg` sequence above — a different mechanism
(file-based keyring vs database import) achieving the same purpose.

**Where to find the correct repo URL and GPG key URL:** always
from the vendor's **official install documentation** — never
guessed or improvised. This is universal practice: HashiCorp,
Docker, Kubernetes, Elastic, and every other vendor publish their
own official repo-setup instructions with the exact URLs to use.
Never reverse-engineer or assume these URLs from a pattern.

### 1.6 rpm -V — verifying package integrity

**GPG signature vs MD5 hash in `rpm -V` — two different jobs, two
different times:**

GPG signature verification happens **once, at install time** — it
proves the package file you are about to install is authentic and
untampered, before anything is written to disk (section 1.5). The
MD5 hashing inside `rpm -V` happens **after** install, any time
later — it checks whether files **already on disk right now** still
match what was originally installed. GPG answers "is this incoming
package legitimate?" `rpm -V`'s MD5 check answers "has anything
changed since I trusted it?" — different questions, different
points in time, not two ways of doing the same check.

**What is the "RPM database record," concretely, and where does
it live?** When a package installs, RPM writes a record into its
own database containing the **expected** size, MD5 hash,
permissions, owner, group, and modification time for every file
that package installs. This database is a real file on disk — on
modern systems, `/var/lib/rpm/rpmdb.sqlite` (older systems use
Berkeley DB format at the same path). `rpm -V` does not compare a
file to the original `.rpm` package file, and does not compare
files to each other — it compares the files currently on disk
against this stored expected-state record from install time.
Internal structure of the RPM database: **Appendix D, section D.3**.

`rpm -V` (**V**erify) compares every file installed by a package
against that database record of what it should look like at
install time: size, MD5 hash, permissions, owner, group,
modification time, and device.

**What does "verify" concretely check, e.g. in `rpm -V nginx`?**
For every file the `nginx` package owns, RPM recomputes its
current size, MD5 hash, permissions, owner, group, and mtime, and
diffs each against the database record from install time. Nothing
is printed for a file that matches on every check; a difference on
any check produces one output line per flagged file, using the
flag positions covered below.

**When to use it:**

- Security incident: did an attacker modify a binary or config?
- Troubleshooting: was a config file changed since install?
- Compliance audit: are binaries on this server as shipped?

```bash
rpm -V nginx        # verify all files in the nginx package
rpm -Va             # verify ALL installed packages (slow — takes minutes)
rpm -Vf /usr/sbin/nginx   # verify the package that owns this specific file
```

**Reading `rpm -V` output:**

When a file passes all checks, nothing is printed. Output only
appears when something differs from the database record:

```
S.5....T.  c  /etc/nginx/nginx.conf
↑↑↑↑↑↑↑↑↑  ↑  ↑
││││││││└─  │  └── file path
│││││││└──  │  size check
││││││└───  │  MD5 hash
│││││└────  │  symlink
││││└─────  │  owner
│││└──────  │  group
││└───────  │  mtime (modification time)
│└────────  │  device
└─────────  │  mode (permissions)
            └── file type:
                c = config file
                d = documentation
                g = ghost file (not installed, but tracked)
                  = (blank) regular file
```

Each position in the 9-character string shows either `.` (pass)
or a letter (fail):

```
S  size differs
M  mode (permissions) differs
5  MD5 hash differs (contents changed)
D  device major/minor number differs
L  symlink target differs
U  owner (User) differs
G  group differs
T  mtime differs
P  capabilities differ
```

**Common output patterns:**

```
S.5....T.  c  /etc/nginx/nginx.conf
```
Size changed, MD5 changed, mtime changed — the config file was
edited. This is expected for a `c` (config) file. Not alarming.

```
.M.......  /usr/sbin/nginx
```
Permissions changed on the binary. Was `755`, now something else.
Investigate — may be intentional hardening or may be an attack.

```
S.5....T.    /usr/sbin/nginx
```
No `c` tag — this is a regular binary, not a config. Size, hash,
and mtime all changed. A modified binary with no distro update to
explain it is a serious security indicator. Investigate immediately.

```
missing   /usr/share/nginx/html/index.html
```
File is missing entirely — was installed by the package, now gone.

### 1.7 Dockerfile RUN layer — correct package installation pattern

Every `RUN` instruction in a Dockerfile creates a new image layer.
Per Docker's own documentation, the cache for a `RUN` instruction
is keyed on the exact text of that instruction — "the cache for
RUN instructions isn't invalidated automatically between builds...
rebuilding the image one week later will still get you the same
packages as before" (docs.docker.com/build/cache/invalidation/).
If the instruction text hasn't changed, the layer is reused
regardless of how much time has passed or what's changed on the
actual package repository.


**Why this section exists — the actual problem being solved:**

The goal is not "always install the newest available package." A
Dockerfile is typically rebuilt many times during normal
development — adding a `COPY` line, changing an `ENV`, fixing
something unrelated — long before anyone deliberately touches the
package list. The real risk is the **install list and the package
index silently drifting apart from each other** across routine
rebuilds.

**The bug, shown step by step — two separate `RUN` lines:**

```
DAY 1 — first build:

  RUN apt-get update              ──▶  [Layer A: fresh index cached]
  RUN apt-get install -y curl     ──▶  [Layer B: installs curl 8.5.0-10.6]


LATER — curl 8.5.0-10.9 is now published. You make an UNRELATED
change elsewhere in the Dockerfile and rebuild:

  RUN apt-get update              ──▶  text unchanged ──▶ CACHE HIT
                                        [Layer A: reused, same index
                                         as Day 1]
  RUN apt-get install -y curl     ──▶  text unchanged ──▶ CACHE HIT
                                        [Layer B: reused, still 10.6]

  Nothing wrong yet — both layers are simply reused, as expected.


NOW — you actually edit the install line, adding git:

  RUN apt-get update              ──▶  text STILL unchanged
                                        ──▶  CACHE HIT
                                        [Layer A: reused — same index
                                         as Day 1, now weeks/months old]
  RUN apt-get install -y curl git ──▶  text CHANGED (added git)
                                        ──▶  cache MISS, reruns fresh
                                        ──▶  runs against the STALE
                                              index from Layer A
                                        ──▶  installs curl 8.5.0-10.6
                                              (old), not the current
                                              repository's version
```

**The same edit, with one combined `RUN` line:**

```
DAY 1 — first build:

  RUN apt-get update && apt-get install -y curl
                                   ──▶  [Layer A: one layer, fresh
                                         index, installs curl 10.6]

LATER — unrelated change, rebuild:

  RUN apt-get update && apt-get install -y curl
                                   ──▶  text unchanged ──▶ CACHE HIT
                                   ──▶  [Layer A: reused as-is]

  Same as before — reused, fine, expected.

NOW — you edit the install line, adding git:

  RUN apt-get update && apt-get install -y curl git
                                   ──▶  text CHANGED
                                   ──▶  the ENTIRE instruction is one
                                         cache key ──▶ cache MISS
                                   ──▶  update AND install both
                                         rerun together, fresh,
                                         against a current index
```

This is the official recommended fix, stated directly in Docker's
own caching guide: "Combine apt-get update and apt-get install in a
single instruction to ensure that update always runs when you
change the installed packages."

**What this does and does NOT fix — stated plainly:**

```
Does NOT do:                          Does do:
- Refresh the index automatically     - Guarantees that whenever you
  on a schedule or timer                edit the install list, the
- Make an untouched Dockerfile          index step is FORCED to rerun
  "self-update" over time                alongside it — never split,
- Replace --no-cache when you           never silently mismatched
  genuinely want a full refresh
  right now
```

**Where `--no-cache` fits — a different problem, not a competing
fix:**

```bash
docker build --no-cache -t myimage .
```

`--no-cache` forces every layer in the build to rerun from scratch,
regardless of whether its instruction text changed. This is the
right tool for a deliberate, occasional full refresh — a scheduled
rebuild to pick up security patches, or a one-off "give me today's
latest packages." It's the wrong default for routine development
builds, where you want unrelated, unchanged layers to stay cached
for speed. The combined `RUN` line and `--no-cache` solve different
problems and are typically both used in production: the combined
line as the permanent pattern in the file, `--no-cache` invoked
periodically (e.g. a weekly CI rebuild) for a genuine refresh.

**A related caching mechanism, outside this demo's scope —
`COPY`/`ADD` invalidate differently than `RUN`:**

Everything above describes `RUN`'s cache key (instruction text).
`COPY` and `ADD` work differently — per Docker's own documentation,
"the builder calculates a cache checksum from all the files to
determine if it can reuse the layer. If any file changes, the
builder invalidates the cache at that step"
(docs.docker.com/build/cache/optimize/). And critically, once any
layer invalidates, "all other layers that come after it are also
affected" (docs.docker.com/build/cache/) — so a `COPY` of
application source code placed *before* an `apt-get` `RUN` line
would force that `RUN` line to rerun on every source-code change,
even with zero edits to the `RUN` line itself. This demo's
Dockerfile has no `COPY` instruction, so this doesn't apply here —
full coverage of `COPY`/`ADD` caching and instruction-ordering
strategy is deferred to a future Docker-focused demo.

**The correct pattern:**

```dockerfile
RUN apt-get update \
    && apt-get install -y --no-install-recommends \
       curl \
       git \
       jq \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/*
```

**Why every part of this pattern matters:**

```
apt-get update                   Refresh the package index. Without this,
                                 the base image's cached index may be stale —
                                 you might install an old version or fail to
                                 find the package at all.

&& (all in one RUN layer)        Ties apt-get update and apt-get install to
                                 ONE cache key. Per Docker's documentation,
                                 RUN's cache is keyed on instruction text —
                                 combining them means any edit to the install
                                 list forces both to rerun together. This does
                                 not refresh anything automatically on its
                                 own; see --no-cache above for that.

-y                               Non-interactive. Required in Dockerfiles —
                                 no terminal to answer prompts.

--no-install-recommends          Debian packages declare three relationship
                                 tiers: Depends (required), Recommends (commonly
                                 useful but not required), Suggests (optional).
                                 Plain "apt-get install" pulls in Depends AND
                                 Recommends. This flag skips the Recommends
                                 tier. Typical image size reduction: 20–40%.

apt-get clean                    Delete downloaded .deb files from the cache.
                                 The files in /var/cache/apt/archives/ are no
                                 longer needed after install.

rm -rf /var/lib/apt/lists/*      Delete the downloaded package index. Must be
                                 in the SAME RUN layer — if in a separate
                                 layer, the files still exist in the earlier
                                 cached layer and are included in the image
                                 even if invisible in later layers.
```

**A note on pinning exact package versions (`package=version`):**

Pinning the exact version string (e.g. `curl=8.5.0-2ubuntu10.6`)
can make a build perfectly reproducible — but only if that exact
string still exists in the repo when you build. Ubuntu publishes
frequent security patch rebuilds that bump the patch suffix every
few weeks, sometimes days. A version copied from documentation, or
from an earlier build attempt, can simply no longer exist by the
time you build again:

```
E: Version '8.5.0-2ubuntu10.6' for 'curl' was not found
```

If you need this level of reproducibility, the version string must
be fetched **immediately before the build, every time**:

```bash
apt-cache policy curl | grep Candidate
# Candidate: 8.5.0-2ubuntu10.9     ← use exactly this, copied fresh
```

For most DevOps tooling images, pinning only the base image tag
(`FROM ubuntu:24.04`, never `:latest`) and leaving package versions
unpinned is the more maintainable default — the pattern used in
Task 6 of this demo.

**What NOT to do:**

```dockerfile
# WRONG — separate update and install layers
RUN apt-get update
RUN apt-get install -y curl git
# Not wrong on the FIRST build. Becomes wrong the next time someone
# edits the install line: the update layer gets silently reused
# (possibly stale) while install reruns fresh against that stale
# index. See the step-by-step walkthrough above.

# WRONG — apt-get upgrade in a container
RUN apt-get update && apt-get upgrade -y && apt-get install -y curl
# Replaces the base image's carefully chosen package set with
# whatever's newest at build time, defeating the purpose of a
# pinned base image. Security patches should come from rebuilding
# with a newer base image, not running upgrade inside the container.

# WRONG — cleaning in a separate layer
RUN apt-get update && apt-get install -y curl
RUN apt-get clean && rm -rf /var/lib/apt/lists/*
# The cache still exists in the install layer. The clean layer
# only hides it. Image is the same size.

# RISKY — hardcoded exact version pins copied from elsewhere
RUN apt-get update && apt-get install -y curl=8.5.0-2ubuntu10.6
# This exact patch-suffix version may no longer exist in the repo
# by the time you build. Fails with "Version ... was not found"
# unless fetched fresh, immediately before this exact build.
```

**Can you use `apt` instead of `apt-get` here? Short answer: don't, even though it would often work.**

```
apt        Designed for interactive human use — friendlier output,
           colorized, progress bars. Its own man page explicitly
           warns: "the apt command is meant to be pleasant for
           end users and does not need to be backward compatible
           like apt-get." Output FORMAT is not guaranteed stable
           between versions.

apt-get    The older, scriptable interface. Output format and
           flag behavior are stable across versions by design —
           this is the actual contract Debian maintains, precisely
           BECAUSE so much automation (Dockerfiles, Ansible,
           shell scripts) depends on it not changing.
```

Nothing will obviously break if you write `RUN apt install -y curl`
today — it works. The risk is a future Ubuntu release changing
`apt`'s output or prompt behavior in a way that's irrelevant to a
human typing interactively, but breaks a script that happened to
depend on it. `apt-get` carries no such risk because backward
compatibility for scripts is its explicit design goal. This is why
every official Docker base image, and Debian's own documentation,
uses `apt-get` in scripts and Dockerfiles, reserving `apt` for
humans at a terminal.

---

## Part 2 — Hands-On Lab

### Commands used in this demo

| Command | Flag meaning | Purpose |
|---|---|---|
| `apt update` | — | Refresh package index |
| `apt install` | — | Install packages |
| `apt show` | — | Show package metadata |
| `apt-mark hold` | — | Hold package at current version |
| `dpkg -l` | l = list | List installed packages |
| `dpkg -S` | S = Search | Find which package owns a file |
| `dpkg -L` | L = List files | List files owned by a package |
| `dnf install` | — | Install packages (RHEL family) |
| `dnf info` | — | Show package metadata |
| `dnf history` | — | Show transaction history |
| `dnf versionlock` | — | Lock package version |
| `rpm -qa` | q = query, a = all | List all installed packages |
| `rpm -qf` | q = query, f = file | Find package owning a file |
| `rpm -ql` | q = query, l = list | List files in a package |
| `rpm -V` | V = Verify | Verify package file integrity |

---

### Task 1 — Query installed packages

**Ubuntu:**

```bash
# List all installed packages
dpkg -l | head -20

# Check if a specific package is installed
dpkg -l curl

# Find what package owns a file
dpkg -S /usr/bin/curl

# List all files installed by a package
dpkg -L curl | head -20

# Show package metadata
apt show curl
```

**Reading `dpkg -l` output:**

```
||/ Name          Version        Architecture   Description
+++-==============-==============-==============-================================
ii  curl          8.5.0-2ubuntu  amd64          command line tool for ...
↑↑
││
│└─ Status: i=installed  c=config-only  n=not-installed  H=half-installed
└── Desired: i=install  h=hold  r=remove  p=purge
ii = correctly installed
```

**Paste your `dpkg -l curl` output.**

---

**Rocky9:**

```bash
# List all installed packages
rpm -qa | sort | head -20

# Check if specific package is installed
rpm -qa | grep curl

# Show package metadata
dnf info curl

# Find what package owns a file
rpm -qf /usr/bin/curl

# List all files installed by a package
rpm -ql curl | head -20

# Show transaction history
dnf history | head -10
```

**Paste your `rpm -qf /usr/bin/curl` output.**

---

### Task 2 — Install, inspect, and remove

**Ubuntu:**

```bash
# Install jq (JSON processor — used constantly in DevOps scripts)
sudo apt update
sudo apt install -y jq

# Verify installation
jq --version
dpkg -l jq
dpkg -L jq | head -10

# Remove and purge
sudo apt purge -y jq
dpkg -l jq   # should show 'rc' status (removed, config files remain after -r; purge removes all)
```

**What `rc` means in dpkg output:**

```
rc  jq   (not installed, config files remain)
↑↑
│└─ c = config files still exist on disk
└── r = desired state is removed
```

After `apt purge`, you should see `pn` (purged, nothing left)
or the package line disappears entirely.

**If `jq` was already installed before you ran this task:** `apt
install -y jq` simply reports it as already at the newest version
and does nothing — this is expected, not an error. The task still
works correctly through the purge/reinstall cycle either way.

---

**Rocky9:**

```bash
# Install jq
sudo dnf install -y jq

# Verify
jq --version
rpm -qi jq          # detailed package info
rpm -ql jq          # all installed files

# Remove
sudo dnf remove -y jq
rpm -qa | grep jq   # should return nothing
```

**Paste your `rpm -qi jq` output — note the Signature field.**

**Why Rocky9's `jq` version may be noticeably older than
Ubuntu's:** RHEL-family distros prioritize long-term stability over
shipping the newest upstream release — packages are often frozen
at a tested version for the life of the major release, receiving
only security backports. Seeing `jq 1.6` on Rocky9 next to `jq
1.7.1` on Ubuntu 24.04 is normal packaging philosophy, not a
broken install or a stale mirror.

**If you tried `sudo apt install -y jq` on Rocky9 by habit:** this
fails with `sudo: apt: command not found` — expected and correct.
`apt` simply does not exist on RHEL-family systems; this is the
same distro-family muscle-memory mistake covered for sudo groups
in Demo M02-02, just showing up here for package managers instead.

---

### Task 3 — Hold / lock a package

**Ubuntu:**

```bash
# Hold curl at its current version
sudo apt-mark hold curl

# Verify the hold
apt-mark showhold
dpkg --get-selections | grep hold

# Attempt upgrade — curl should be kept back
sudo apt upgrade --dry-run | grep -A2 "kept back" || echo "no held packages shown in dry-run"

# Release the hold
sudo apt-mark unhold curl
apt-mark showhold   # should be empty
```

**Paste the `apt-mark showhold` output showing curl held.**

---

**Now the alternative — pin via `/etc/apt/preferences.d/`, never
exercised above:**

```bash
# Check curl's current version first
apt-cache policy curl | head -3

# Create a pin file — restrict curl to its current major.minor,
# with a priority high enough to override normal upgrade behavior
CURRENT_VER=$(apt-cache policy curl | grep Installed | awk '{print $2}')
echo "Package: curl
Pin: version ${CURRENT_VER%-*}*
Pin-Priority: 1001" | sudo tee /etc/apt/preferences.d/curl-pin

# Confirm apt now sees this pin
apt-cache policy curl
# The candidate version line should now reflect the pin,
# and a line noting the 1001 priority should appear

# Clean up
sudo rm /etc/apt/preferences.d/curl-pin
apt-cache policy curl   # confirm pin removed, normal candidate returns
```

**Paste the `apt-cache policy curl` output before and after
creating the pin file — note where the `1001` priority appears.**

---

**Rocky9:**

```bash
# Install versionlock plugin if not present
sudo dnf install -y python3-dnf-plugin-versionlock

# Lock curl at current version
sudo dnf versionlock add curl

# Verify
dnf versionlock list

# Show that upgrade would skip it
sudo dnf upgrade --assumeno curl 2>&1 | head -5

# Remove the lock
sudo dnf versionlock delete curl
dnf versionlock list   # should be empty
```

**Paste `dnf versionlock list` output showing curl locked.**

---

### Task 4 — Add a third-party repository

This task adds the HashiCorp repository on both distros and installs
Terraform. HashiCorp is chosen because it uses the same repo-add
pattern you will use for Docker, Kubernetes, and other third-party
tools — GPG key import, repo file, then install.

**Ubuntu:**

```bash
# Step 1: Import GPG key to a named keyring
curl -fsSL https://apt.releases.hashicorp.com/gpg \
  | gpg --dearmor \
  | sudo tee /etc/apt/keyrings/hashicorp.gpg > /dev/null

# Verify the key was saved
ls -la /etc/apt/keyrings/hashicorp.gpg

# Step 2: Add the repository
echo "deb [signed-by=/etc/apt/keyrings/hashicorp.gpg] \
  https://apt.releases.hashicorp.com $(lsb_release -cs) main" \
  | sudo tee /etc/apt/sources.list.d/hashicorp.list

# Step 3: Update index and install
sudo apt update
sudo apt install -y terraform

# Verify
terraform version
dpkg -S $(which terraform)   # confirms which package owns the binary
```

**Note what `lsb_release -cs` produces on your system** —
it prints the codename (`noble` for Ubuntu 24.04, `jammy` for 22.04).
The repo URL uses the codename to serve the right packages.

**Rocky9:**

```bash
# Step 1: Import GPG key into RPM database
sudo rpm --import https://rpm.releases.hashicorp.com/gpg

# Verify key was imported
rpm -qa gpg-pubkey | xargs -I{} rpm -qi {} | grep -A2 "HashiCorp"

# Step 2: Add repository
sudo dnf config-manager --add-repo \
  https://rpm.releases.hashicorp.com/RHEL/hashicorp.repo

# Verify repo was added
ls /etc/yum.repos.d/ | grep hashicorp
cat /etc/yum.repos.d/hashicorp.repo

# Step 3: Install
sudo dnf install -y terraform

# Verify
terraform version
rpm -qf $(which terraform)
```

**Paste `terraform version` output on both systems.**

---

### Task 5 — rpm -V: verify package integrity

**Run on Rocky9:**

```bash
# Verify a specific package — clean state first
rpm -V curl
# No output = all files match database — package is intact

# Now deliberately modify a config-adjacent file to see output
# (We use a package with a config file for this demonstration)
sudo dnf install -y openssh-server
rpm -V openssh-server
# Likely no output if freshly installed

# Change permissions on a file to trigger a verification failure
sudo chmod 600 /usr/sbin/sshd   # should be 755
rpm -V openssh-server
# .M.......  /usr/sbin/sshd  ← mode changed

# Restore correct permissions
sudo chmod 755 /usr/sbin/sshd
rpm -V openssh-server           # should be silent again

# Verify a specific file directly
rpm -Vf /usr/sbin/sshd

# Run a broader check — all packages (this takes time)
# rpm -Va 2>/dev/null | head -20   # uncomment to run — takes ~60 seconds
```

**Reading the output:**

```
.M.......  /usr/sbin/sshd
↑↑↑↑↑↑↑↑↑
12345678 9

Position 1: S — size
Position 2: M — mode (permissions)  ← M appears here
Position 3: 5 — MD5 hash
Position 4: D — device
Position 5: L — symlink
Position 6: U — user/owner
Position 7: G — group
Position 8: T — mtime
Position 9: P — capabilities
. = pass at that position
Letter = that check failed
```

**Paste the output before and after restoring permissions.**

---

### Task 6 — Dockerfile: correct package installation layer

**Run on Ubuntu:**

```bash
# Find current pinnable versions for our packages
apt-cache policy curl git unzip jq ca-certificates gnupg

# The "Candidate:" line for each package is the version apt would
# install right now. Note it carefully — see the warning below
# before using it in a Dockerfile.
```

**⚠️ Why exact version pins go stale fast — and why this demo
does not hardcode them:**

Ubuntu ships frequent security patch rebuilds, which bump the
patch suffix on a package version (e.g. `8.5.0-2ubuntu10.6` →
`8.5.0-2ubuntu10.9`) every few weeks, sometimes days. A version
string hardcoded into a Dockerfile today can be **gone from the
repo** by the time you build again — `apt-get` will fail outright
with:

```
E: Version '8.5.0-2ubuntu10.6' for 'curl' was not found
```

This is not a hypothetical — it is the exact failure you get if
you copy a version string from documentation or from a teammate's
machine instead of querying the live repo immediately before your
build. The correct workflow when you genuinely need pinned,
reproducible versions:

```bash
# Run this immediately before every build, not once and reused
apt-cache policy curl | grep Candidate
# Candidate: 8.5.0-2ubuntu10.9     ← use exactly this, copied fresh
```

For this demo's image — a general-purpose DevOps toolbox, not a
compliance-pinned production base — we pin only the **base image
tag** (`ubuntu:24.04`, never `:latest`) and leave individual
package versions unpinned. This keeps the build reproducible at
the OS-version level while avoiding a build that breaks every time
Ubuntu ships a security patch. If your use case genuinely requires
pinning every package (regulated environments, strict change
control), pin using the `apt-cache policy` output fetched fresh at
build time — never a version string copied from elsewhere.

**`src/Dockerfile`:**

```dockerfile
# DevOps base image — Demo M02-01: Package Management
#
# Demonstrates the correct Dockerfile RUN layer pattern for apt:
#   - apt-get update and install in one RUN instruction (cache busting)
#   - --no-install-recommends (smaller image)
#   - Cache cleaned in the same layer (no hidden bloat)
#
# Base image tag is pinned — never use :latest. Individual package
# versions are intentionally NOT pinned here — see Part 1 section 1.7
# for why hardcoded patch-suffix versions go stale within weeks.
FROM ubuntu:24.04

# Prevent interactive prompts during package install
ENV DEBIAN_FRONTEND=noninteractive

# Single RUN layer: update + install + clean
# All three steps must be in one instruction — see Part 1 section 1.7
RUN apt-get update \
    && apt-get install -y --no-install-recommends \
       curl \
       ca-certificates \
       gnupg \
       unzip \
       jq \
       git \
       procps \
    # Remove downloaded .deb files — not needed after install
    && apt-get clean \
    # Remove package index — regenerated by apt-get update when needed
    # Must be in this same RUN layer — a separate RUN would not reduce
    # image size because the files already exist in an earlier layer
    && rm -rf /var/lib/apt/lists/*

# Verify tools are present and callable
# Build fails here if any package was not installed correctly
RUN curl --version \
    && git --version \
    && jq --version \
    && gpg --version | head -1 \
    && unzip -v | head -1

# Default to bash — explicitly set for clarity
CMD ["/bin/bash"]

# Build:   docker build -t devops-base:demo04 .
# Run:     docker run --rm devops-base:demo04 jq --version
# History: docker history devops-base:demo04
#          (should show one RUN layer for all package install steps)
```

```bash
cd ~/path/to/01-package-management/src

# Build the image — first build, everything fresh
docker build -t devops-base:demo04 .

# Verify installed packages inside the container
docker run --rm devops-base:demo04 curl --version
docker run --rm devops-base:demo04 git --version
docker run --rm devops-base:demo04 jq --version

# Inspect image layers — see the single RUN layer
docker history devops-base:demo04
```

**What to observe in `docker history` output:**

```
IMAGE          CREATED BY                                      SIZE
<hash>         RUN /bin/sh -c apt-get update && apt-get ...   ~80MB
<hash>         /bin/sh -c #(nop)  CMD ["/bin/bash"]           0B
<hash>         /bin/sh -c #(nop) ADD file:...                 ~80MB

One RUN layer = update + install + clean all in one step.
If you see two RUN layers for apt, the pattern is wrong.
```

**Now — reproduce the cache mechanism from section 1.7 live:**

```bash
# Rebuild with NO changes — confirm the layer is fully cached
docker build -t devops-base:demo04 .
# Look for "CACHED" next to the RUN line in the build output —
# this confirms Docker reused the layer because the instruction
# text didn't change, exactly as section 1.7 describes

docker history devops-base:demo04
# SIZE for the RUN layer should be identical to the first build —
# nothing re-executed
```

```bash
# Now edit the Dockerfile — add "vim" to the install list
sed -i 's/       procps \\/       procps \\\n       vim \\/' src/Dockerfile
cat src/Dockerfile | grep -A2 procps
# Confirm vim was added to the install line

# Rebuild
docker build -t devops-base:demo04 .
# This time the RUN line should NOT say "CACHED" — the instruction
# text changed (vim was added), so the entire layer reruns,
# including apt-get update, fresh — exactly the guarantee section
# 1.7 describes: editing the install list forces update to rerun
# alongside it, never silently split

docker run --rm devops-base:demo04 vim --version | head -1
# Confirms vim installed successfully in the same rebuild
```

```bash
# Finally — force a full refresh regardless of cache, the
# DIFFERENT tool for the DIFFERENT problem covered in section 1.7
docker build --no-cache -t devops-base:demo04 .
# Every layer reruns from scratch this time, including the base
# image setup — not because anything in the Dockerfile changed,
# but because --no-cache was explicitly requested
```

**Paste your build output from the unchanged rebuild (showing
`CACHED`), the edited rebuild (showing the layer re-executing), and
the `--no-cache` build — in that order, with a one-line note on
what triggered each one to behave differently.**

---

## What You Learned

- ✅ Explained what a package contains, the four package types, and why `.deb`/`.rpm` formats enable dependency resolution and verification in ways a plain zip/tar cannot
- ✅ Distinguished a package's own embedded metadata from a repository's package index
- ✅ Queried installed packages, searched for packages, and inspected package metadata on both Ubuntu and Rocky9
- ✅ Installed, updated, and removed packages cleanly on both distros
- ✅ Held a package at a specific version two ways — `apt-mark`/`dnf versionlock` and, on Debian, `/etc/apt/preferences.d/` pinning — and released the hold
- ✅ Added a third-party repository with its GPG key on both distros
- ✅ Verified the integrity of installed package files using `rpm -V`
- ✅ Read and interpreted `rpm -V` output flags
- ✅ Wrote a correct `Dockerfile` `RUN` layer for package installation — correct layer ordering, no cache, no unnecessary packages — and learned why hardcoded version pins go stale and how to pin safely when truly required
- ✅ Explained the relationship between the high-level tool (`apt`/`dnf`) and the low-level tool (`dpkg`/`rpm`), with a side-by-side command reference for both distro families
- ✅ Connected package manager install locations back to the FHS map from Demo M01-07

**Key Takeaway:** Package management failures almost always trace back to one of three things: a stale index, a missing GPG key, or a Docker layer caching the wrong state. Knowing which of the three you're looking at turns a confusing error into a two-command fix. And once you see that a package manager is just placing files into the same FHS locations you already mapped in Demo M01-07 — `/usr`, `/etc`, `/var`, `/run` — you stop needing a separate mental model for "where package managers put things."

---

## Part 3 — Break-Fix Scenario

### Scenario — NovaTech Ops: two package management failures

> *"Incident 1: The automation team added the Elastic repository to
> our Rocky9 build servers for Filebeat. The install keeps failing.
> Error appears after the package downloads. Works fine on Ubuntu."*

> *"Incident 2: Security team flagged `/usr/bin/ssh` on our Rocky9
> bastion host. They ran `rpm -V openssh-clients` and got output.
> They do not know how to read it and neither does the on-call
> engineer. Need it interpreted and a remediation plan."*

**Investigation — Incident 1:**

```bash
# Rocky9 — attempting Elastic repo install
sudo dnf install -y filebeat

# Output:
# Downloading Packages:
# filebeat-8.14.0-x86_64.rpm    3.1 MB/s | 31 MB
# error: Verifying a signature using certificate ... (key ID d27d666cd88e42b4):
#   NOKEY (status)
# The downloaded packages were saved in cache till the next
# successful transaction.
# Error: GPG check FAILED
```

```bash
# Check what keys are imported
rpm -qa gpg-pubkey

# Check the repo file
cat /etc/yum.repos.d/elastic.repo
# [elasticsearch-8.x]
# name=Elasticsearch repository for 8.x packages
# baseurl=https://artifacts.elastic.co/packages/8.x/yum
# gpgcheck=1
# gpgkey=https://artifacts.elastic.co/GPG-KEY-elasticsearch
# enabled=1
# autorefresh=1
# type=rpm-md
```

**Investigation — Incident 2:**

```bash
# Output from security team's rpm -V run:
rpm -V openssh-clients

# S.5....T.    /usr/bin/ssh
```

**Your tasks:**

1. What is the root cause of Incident 1? What command was skipped
   and what is the exact fix?

2. For Incident 2: read the `rpm -V` output character by character.
   What does `S.5....T.` tell you about `/usr/bin/ssh`? Is this
   likely benign or serious? What is the remediation?

3. After fixing Incident 1, what command would you run to verify
   the Filebeat package files are intact after install?

<details>
<summary>Resolution — open after attempting</summary>

**Incident 1 — GPG check FAILED:**

Root cause: the GPG key for the Elastic repository was never
imported into the RPM database. The `.repo` file has `gpgcheck=1`
(correct — should always be 1) and points to `gpgkey=` URL, but
`gpgkey=` in a repo file tells dnf where to find the key — it
does not automatically import it. The key must be explicitly
imported before or during the first install.

**Fix — import the key:**
```bash
sudo rpm --import https://artifacts.elastic.co/GPG-KEY-elasticsearch

# Verify import
rpm -qa gpg-pubkey | xargs -I{} rpm -qi {} | grep -i elastic

# Retry install
sudo dnf install -y filebeat
```

The difference from what was done: `dnf config-manager --add-repo`
or manually writing a `.repo` file does not import the key.
`rpm --import` must be run explicitly. On Ubuntu, `apt` handles
this differently — the `signed-by=` field in the `.list` file
points to a pre-imported keyring file. Both approaches require
the key to be present before install.

**Incident 2 — reading `S.5....T.    /usr/bin/ssh`:**

```
S . 5 . . . . T .
↑   ↑         ↑
│   │         └── mtime changed
│   └── MD5 hash changed (contents changed)
└── Size changed
```

Position 2 (M = mode/permissions): `.` — permissions unchanged.
Position 6 (U = user): `.` — owner unchanged.
Position 7 (G = group): `.` — group unchanged.

The binary `/usr/bin/ssh` has changed in size, content (MD5),
and modification time. There is no file type tag (`c`, `d`) —
this is a **regular binary file**, not a config file.

A modified binary on a bastion host with no corresponding distro
update is a **serious security indicator**. This is not a config
edit — this is the `ssh` client binary itself.

**Remediation:**

```bash
# Step 1: Do not run the binary until resolved
# Step 2: Check if there was a legitimate update
dnf history | head -20          # was openssh-clients upgraded recently?
rpm -qi openssh-clients         # check install date vs current date

# Step 3: If no legitimate update explains it — incident response
# Isolate the host. Preserve disk image for forensics.
# Reinstall the package from a known-good source:
sudo dnf reinstall openssh-clients

# Step 4: Verify after reinstall
rpm -V openssh-clients          # should be silent
```

**Incident 3 — post-install verification (question 3):**

```bash
rpm -V filebeat
# Silent output = all files match the package database = intact
# Any output = investigate each flagged file
```

</details>

---

## Interview Prep

**Q1. What is the difference between `apt remove` and `apt purge`?
When do you use each?**

`apt remove` uninstalls the package binaries but leaves configuration
files on disk. `apt purge` removes binaries and all configuration
files. Use `remove` when you may want to reinstall later and keep
your configuration — for example, temporarily removing a service
during maintenance. Use `purge` when you want a completely clean
slate — reinstalling fresh, or removing a package permanently.
In containers and CI environments, always use `purge` to avoid
config file residue affecting subsequent installs.

---

**Q2. A `dnf upgrade` on a Kubernetes worker node upgrades `kubelet`
from 1.29 to 1.30 unexpectedly. The node becomes not ready.
How do you prevent this in future?**

Use `dnf versionlock add kubelet kubeadm kubectl` after the
initial install at the target version. This locks all three
packages together — they must be upgraded in lockstep with the
control plane following the Kubernetes upgrade procedure.
On Ubuntu use `apt-mark hold kubelet kubeadm kubectl`.
The lock must be released, the upgrade performed, and the lock
re-applied at the new version — not done with a blanket
`dnf upgrade`.

---

**Q3. `rpm -V nginx` returns `S.5....T.  /etc/nginx/nginx.conf`.
Is this a problem?**

No — this is expected. The `c` tag in the output marks this as
a config file. Config files are modified after install — that is
their purpose. `rpm -V` treats modifications to config files as
informational, not as integrity failures. The concern would be
`S.5....T.` on a binary file with no `c` tag, which would indicate
the binary itself was modified.

---

**Q4. Why must `apt-get update` and `apt-get install` be in the
same `RUN` instruction in a Dockerfile?**

Docker's cache key For `RUN` instruction is the exact text of an instruction — an
unchanged instruction is reused regardless of how much time has
passed. The real risk is on the next edit: if `update` and
`install` are separate lines and someone later edits only the
`install` line (adding a package, say), Docker reuses the
unchanged `update` layer — a possibly stale cached index — while
re-running `install` fresh against that stale data, installing
whatever the old index knew about rather than the current
repository. Combining them into one instruction means any edit to
the install list changes the entire instruction's text, forcing
`update` and `install` to rerun together every time, never split
apart. This is not automatic staleness-busting on a schedule —
if the Dockerfile is never touched again, neither version
refreshes the index on its own; both still need `--no-cache` or a
new base image to force that.

---

## Key Takeaways

1. **`apt`/`dnf` are the high-level tools that resolve dependencies and fetch from repos; `dpkg`/`rpm` are the low-level tools that manage the package database and install files.** Use the high-level tools for install/remove, the low-level tools for queries and verification.

2. **Always run `apt update` before `apt install` — never assume the local package index is current.** A stale index silently installs old versions or fails to find packages added after your last update.

3. **Hold packages that have strict version coupling: `apt-mark hold` on Ubuntu, `dnf versionlock` on RHEL.** Kubernetes nodes, database servers, and anything with cluster-wide version requirements need this — an unintended upgrade is a production incident. On Debian, `/etc/apt/preferences.d/` pinning is more flexible than a hold when you need a version *pattern* rather than one exact version.

4. **GPG key import is mandatory before installing from a third-party repo — skipping it causes `GPG check FAILED` at install time, not at repo-add time.** On RHEL: `rpm --import URL`. On Debian: `gpg --dearmor | tee /etc/apt/keyrings/name.gpg`. The repo file points to the key but does not import it automatically.

5. **`rpm -V` prints nothing when a package is intact — any output means a file differs from the database record.** `c` in the file type column means config file (expected to be modified). No tag on a modified binary is a security indicator requiring immediate investigation.

6. **In a Dockerfile: `apt-get update && apt-get install` in one `RUN` layer, `--no-install-recommends`, and cleanup in the same layer.** Docker's cache key for `RUN` is the instruction's exact text, not elapsed time — splitting update/install risks a stale-index install only when someone later edits the install line and the update layer gets silently reused. Combining them ties both to one cache key, so any edit to the install list forces both to rerun together. Cleanup in a separate layer never reduces image size, regardless of caching, because the files already exist in an earlier layer.

7. **`dnf history undo N` can roll back a transaction on RHEL systems** — a capability `apt` does not have. Record transaction numbers before major upgrades so you have a rollback target.

---

## Quick Commands Reference

### apt command reference

| Command | Purpose |
|---|---|
| `apt update` | Refresh package index |
| `apt install -y pkg` | Install package |
| `apt install pkg=version` | Install specific version |
| `apt remove pkg` | Remove (keep config) |
| `apt purge pkg` | Remove + config files |
| `apt upgrade` | Upgrade all packages |
| `apt search term` | Search packages |
| `apt show pkg` | Show metadata |
| `apt-mark hold pkg` | Hold at current version |
| `apt-mark unhold pkg` | Release hold |
| `apt-mark showhold` | List held packages |

### dpkg command reference

| Command | Purpose |
|---|---|
| `dpkg -l` | List all installed |
| `dpkg -l pkg` | List specific package |
| `dpkg -S /path/to/file` | Which package owns this file |
| `dpkg -L pkg` | All files in this package |
| `dpkg --get-selections` | All packages with states |

### dnf command reference

| Command | Purpose |
|---|---|
| `dnf install -y pkg` | Install package |
| `dnf remove pkg` | Remove package |
| `dnf upgrade` | Upgrade all packages |
| `dnf search term` | Search |
| `dnf info pkg` | Show metadata |
| `dnf provides /path` | Which package provides file |
| `dnf history` | Transaction history |
| `dnf history undo N` | Undo transaction N |
| `dnf versionlock add pkg` | Lock version |
| `dnf versionlock delete pkg` | Remove lock |

### rpm command reference

| Command | Purpose |
|---|---|
| `rpm -qa` | List all installed |
| `rpm -qi pkg` | Package info |
| `rpm -ql pkg` | Files in package |
| `rpm -qf /path` | Package owning file |
| `rpm -V pkg` | Verify package integrity |
| `rpm -Va` | Verify all packages |
| `rpm --import URL` | Import GPG key |

### rpm -V output flags

| Position | Letter | Meaning |
|---|---|---|
| 1 | S | Size differs |
| 2 | M | Mode (permissions) differs |
| 3 | 5 | MD5 hash differs |
| 4 | D | Device number differs |
| 5 | L | Symlink target differs |
| 6 | U | User/owner differs |
| 7 | G | Group differs |
| 8 | T | Modification time differs |
| 9 | P | Capabilities differ |
| — | c | Config file (modifications expected) |
| — | d | Documentation file |
| — | . | Check passed |

---

## Appendix A — Anki Cards

**`m02-01-package-management-anki.csv`:**

```
#deck:Linux Mastery::Module 2 - System Administration::Demo M02-01 - Package Management
#separator:Comma
#columns:Front,Back,Tags

"You run dnf install filebeat and get 'GPG check FAILED'. The repo file exists and gpgcheck=1. What was skipped and what is the fix?","The GPG key was never imported. The repo file's gpgkey= line tells dnf where to find the key but does not import it automatically. Fix: sudo rpm --import https://repo-url/GPG-KEY then retry install. On Debian: curl -fsSL URL | gpg --dearmor | sudo tee /etc/apt/keyrings/name.gpg","demo04,repos,gpg,security"

"rpm -V openssh-clients returns: S.5....T.  /usr/bin/ssh (no c tag). What does this mean and how serious is it?","Size (S), MD5 hash (5), and mtime (T) all differ from the package database record. No 'c' tag means this is a regular binary — not a config file. A modified ssh binary with no distro update to explain it is a serious security indicator. Remediation: check dnf history for a legitimate update. If none found, isolate host, preserve for forensics, reinstall: dnf reinstall openssh-clients","demo04,rpm-verify,security"

"What is the difference between apt remove and apt purge? When do you use each?","apt remove uninstalls binaries but leaves config files. apt purge removes binaries AND config files. Use remove when you may reinstall and want to keep config. Use purge for a clean slate — reinstalling fresh, or permanent removal. In containers and CI, always purge to avoid config residue affecting subsequent installs.","demo04,apt,remove,purge"

"A Kubernetes worker node's kubelet upgraded unexpectedly from 1.29 to 1.30 during dnf upgrade. Node is NotReady. What should have been done after the initial install?","dnf versionlock add kubelet kubeadm kubectl (RHEL) or apt-mark hold kubelet kubeadm kubectl (Ubuntu). All three packages must be version-locked together — they must stay in lockstep with the control plane. The lock is released, upgrade performed at planned version, then re-applied.","demo04,versionlock,kubernetes"

"Why must apt-get update and apt-get install be in the same RUN instruction in a Dockerfile?","Docker's cache key is the exact TEXT of an instruction, not elapsed time - an unchanged line is reused no matter how stale the real repo is. The actual risk: if update/install are split and someone later edits only the install line, Docker reuses the unchanged (possibly stale) update layer while installing fresh against that stale index. Combining them means any edit to the install list invalidates the whole instruction, forcing both to rerun together - never split. This is NOT automatic refresh on a timer; an untouched Dockerfile stays cached either way without --no-cache or a new base image.","demo04,dockerfile,apt,caching"

"What does --no-install-recommends do in apt-get install? Why use it in containers?","Prevents installation of recommended packages — packages not strictly required for the tool to function. apt installs recommended packages by default. In a container, recommended packages add size with no benefit. Typical image size reduction: 20-40%. Always use --no-install-recommends in Dockerfiles.","demo04,dockerfile,apt,containers"

"rpm -V nginx returns nothing. rpm -V nginx shows: .M.......  /usr/sbin/nginx. What does each mean?","No output: all files match the package database — package is intact. .M.......: the M position (mode/permissions) differs from the database record. Permissions on the nginx binary were changed after install. Investigate: was this intentional hardening? Run stat /usr/sbin/nginx to see current permissions. May be benign but warrants review.","demo04,rpm-verify,permissions"

"What is the relationship between apt and dpkg? Between dnf and rpm?","apt is the high-level tool: resolves dependencies, fetches packages from repos, then calls dpkg to install. dpkg is the low-level tool: installs/removes .deb files, manages the package database, does NOT resolve deps. Same relationship: dnf (high-level) calls rpm (low-level). Use high-level for install/remove. Use low-level for queries (dpkg -S, rpm -qf) and verification (rpm -V).","demo04,apt,dpkg,dnf,rpm"

"What is the difference between a binary package and a source package? Which does apt/dnf install by default?","Binary package: pre-compiled, ready to install as-is — what apt/dnf install by default (e.g. nginx, curl, jq). Source package: contains source code you compile yourself for your exact system (Debian: .dsc + .orig.tar.* + .debian.tar.*; RHEL: .src.rpm). Source packages are rare in day-to-day DevOps work — mainly used when no binary exists for your architecture or you need a custom compile-time option.","demo04,package-types"

"What is a meta-package? Give an example use case.","A meta-package contains no files of its own — it exists purely to pull in a group of other packages as dependencies. Example: a build-essential-style package that depends on gcc, make, and related tools, giving you one thing to install instead of several.","demo04,package-types,meta-package"

"What is a virtual package?","A name a package manager recognizes that isn't a real package itself — it represents a capability that multiple real packages can satisfy. Example: several different mail-server packages can all 'provide' mail-transport-agent, letting other packages depend on the capability rather than one specific implementation.","demo04,package-types,virtual-package"

"Is a package's embedded metadata the same thing as a repository's package index? Explain the difference.","No. A package's metadata (name, version, dependencies, maintainer) is embedded inside that one package file and describes only itself. The repository index (Packages/Release for Debian, repodata for RHEL) is a separate manifest the repository publishes, built by aggregating metadata across every package it hosts, plus a checksum for each entry. The index is what search/show/info query before install; the package's own metadata is read from the local install database after install.","demo04,package-index,metadata"

"A package manager installs nginx via apt. Where do the binary, config, logs, and persistent data each go, and why does this match Demo M01-07's FHS map rather than a separate 'package manager' convention?","Binary: /usr/bin/nginx (package-manager-installed binaries). Config: /etc/nginx/ (host-specific service config). Logs: /var/log/nginx/ (persistent growing log data). Data: /var/lib/nginx/ (persistent app state). The package manager doesn't invent new locations — it follows the exact FHS map from Demo M01-07. This is the 'package manager install' pattern, distinct from manual install (/usr/local/bin only) and vendor self-contained install (/opt, everything together).","demo04,fhs,package-locations"

"Why does this demo's Dockerfile NOT pin exact package versions like curl=8.5.0-2ubuntu10.6, even though pinning is described as good practice for reproducibility?","Ubuntu publishes frequent security patch rebuilds that change the patch suffix every few weeks. A hardcoded version string copied from documentation or an earlier build can simply no longer exist in the repo, causing 'Version ... was not found' errors. For general-purpose tooling images, pinning only the base image tag (ubuntu:24.04, never :latest) and leaving package versions unpinned is more maintainable. True version pinning, when genuinely required, must use a version string fetched fresh via apt-cache policy immediately before that exact build — never reused from an earlier session.","demo04,dockerfile,versioning,reproducibility"
```

---

## Appendix B — Quiz

**`m02-01-package-management-quiz.md`:**

````markdown
# Quiz — Demo M02-01: Package Management

> One correct answer per question unless stated otherwise.
> Target: 80% or above before moving to Demo M02-02.

---

**Q1.** `apt-cache policy nginx` shows several version numbers with
one marked `***`. What does that marked version represent, and
where does `apt install nginx` (with no version specified) get its
version from?

- A) `***` marks the oldest available version — apt always defaults to the most conservative choice
- B) `***` marks the **candidate** version — the highest version in
  the local index satisfying any pinning rules — and this is exactly
  what a plain `apt install nginx` installs
- C) `***` marks a version flagged as insecure and to be avoided
- D) `apt install` always ignores `apt-cache policy` and queries the live repo directly instead

<details>
<summary>Answer</summary>

**B** — The candidate version is the one apt would install or
upgrade to right now, computed from the local index and any pin
priorities in effect. `apt-cache policy` is exactly how you inspect
this before committing to an install.

</details>

---

**Q2.** A `dnf upgrade` breaks a service — a kernel update fails to
boot, say. What RHEL-specific capability lets you cleanly reverse
just that one transaction, and why does apt have no equivalent?

- A) `dnf history undo N` — reverses a specific numbered transaction;
  apt keeps no interactive transaction browser or built-in undo,
  so real-world recovery on Debian/Ubuntu relies on filesystem
  snapshots or manually reinstalling known-good versions
- B) `apt rollback` — both apt and dnf support this identically
- C) Neither package manager can reverse a bad transaction; a full reinstall is always required
- D) `dnf downgrade --auto` reverses any transaction automatically without needing a transaction number

<details>
<summary>Answer</summary>

**A** — This is a genuine, notable asymmetry between the two
families. Recording transaction numbers (`dnf history`) before a
major upgrade is what makes `history undo` actually useful when
something breaks.

</details>

---

**Q3.** Why does this series consistently use `apt-get` rather than
`apt` inside Dockerfiles and scripts, even though `apt install -y
curl` would usually work today?

- A) `apt` is deprecated and will be removed in a future Ubuntu release
- B) `apt`'s own documentation explicitly states it is designed for
  interactive human use and its output format is not guaranteed
  stable between versions; `apt-get`'s scriptability and stable
  output are its explicit design contract, which automation depends on
- C) `apt-get` runs faster than `apt` for identical operations
- D) `apt` cannot be used with the `-y` flag in non-interactive contexts

<details>
<summary>Answer</summary>

**B** — Nothing breaks today by using `apt` in a script, but a
future Ubuntu release changing `apt`'s interactive-oriented output
is a real, documented risk that `apt-get` is specifically designed
not to have.

</details>

---

**Q4.** An `/etc/apt/preferences.d/` pin file sets `Pin-Priority:
1001` for `nginx`. What does a priority of `1001` actually cause
apt to do that a normal install wouldn't?

- A) Nothing different — priorities above 1000 have no special
  meaning versus the default of 500
- B) It causes apt to prefer this pinned version even if it means a
  **downgrade** from what's normally the candidate — priority ≥
  1000 is the one tier that overrides apt's default "always prefer
  newest" behavior
- C) It only affects `apt search`, never `apt install`
- D) `1001` is invalid; valid priorities only run from 0-999

<details>
<summary>Answer</summary>

**B** — This is the specific behavior that separates the ≥1000
tier from every other priority band: normal pin priorities (500,
990-1000) still respect "don't downgrade," but ≥1000 explicitly
overrides even that.

</details>

---

**Q5.** What is the concrete content difference between a Debian
`.list` repo file and an RHEL `.repo` file — not just syntax, but
what information each actually contains?

- A) They contain identical information in different syntax, nothing more
- B) A `.repo` file is self-contained — GPG key URL, enable/disable
  flag, and metadata source all in one file. A `.list` file only
  has the repo URL; the GPG key reference points to a *separate*
  keyring file you create yourself, with no per-repo `gpgcheck`-style toggle
- C) `.list` files are self-contained; `.repo` files require a separate keyring file
- D) Only `.repo` files can reference third-party repositories; `.list` files are Ubuntu-official repos only

<details>
<summary>Answer</summary>

**B** — This is a genuine content asymmetry, not just a syntax
difference: RHEL's `.repo` format bundles everything about a
repository's trust and location in one file, while Debian splits
the GPG key into a separate keyring file referenced by `signed-by=`.

</details>

---

**Q6.** After `apt remove nginx`, `dpkg -l nginx` shows `rc`.
What does `rc` mean?

- A) Package is removed and nothing remains
- B) Package removal failed — config still in use
- C) Package binaries removed, config files still on disk
- D) Package is on hold at current version

<details>
<summary>Answer</summary>

**C** — `r` = desired state is remove. `c` = config files remain on disk.
`apt remove` keeps config files. To remove config files too: `apt purge nginx`.
After purge, dpkg shows `pn` (purged, nothing) or the entry disappears.

Trap: D would show `hi` (hold, installed) — not `rc`.

</details>

---

**Q7.** You need to find which installed package provides the file
`/usr/bin/jq` on both Ubuntu and Rocky9. Which commands are correct?

- A) Ubuntu: `apt show jq` — Rocky9: `dnf info jq`
- B) Ubuntu: `dpkg -S /usr/bin/jq` — Rocky9: `rpm -qf /usr/bin/jq`
- C) Ubuntu: `apt search jq` — Rocky9: `dnf search jq`
- D) Ubuntu: `dpkg -L jq` — Rocky9: `rpm -ql jq`

<details>
<summary>Answer</summary>

**B** — `dpkg -S path` (Search) and `rpm -qf path` (query file) both
answer "which installed package owns this file?" given a path.

Trap: A and C search for packages by name/description — they require
you to already know the package name. D lists all files in a named
package — the reverse direction.

</details>

---

Score guide:

| Score | Action |
|---|---|
| 7/7 | Import Anki cards, move to Demo M02-02 |
| 6/7 | Review the wrong answer, then proceed |
| 5/7 | Re-read the relevant section, retry those questions |
| Below 5/7 | Re-read the full demo and redo the walkthrough before proceeding |
````

---

## Appendix C — apt vs dnf and dpkg vs rpm: Side-by-Side Reference

This appendix exists so you stop trying to memorize two parallel
command sets independently. Learn the concept once; use this table
to look up the syntax for whichever distro you're on right now.

### C.1 High-level tool: apt vs dnf

| Action | apt (Debian/Ubuntu) | dnf (RHEL/Rocky) | Notes |
|---|---|---|---|
| Refresh package index | `apt update` | *(no separate command)* | dnf checks/refreshes automatically on most operations |
| Install package | `apt install -y pkg` | `dnf install -y pkg` | Direct equivalent |
| Install specific version | `apt install pkg=version` | `dnf install pkg-version` | Syntax differs: `=` vs `-` |
| Remove, keep config | `apt remove pkg` | *(no equivalent — dnf remove typically removes config too)* | RPM packaging convention differs |
| Remove + config files | `apt purge pkg` | `dnf remove pkg` | dnf's remove ≈ apt's purge in effect |
| Upgrade all packages | `apt upgrade` | `dnf upgrade` (alias: `dnf update`) | dnf accepts "update" as upgrade — apt does NOT |
| Upgrade one package | `apt upgrade pkg` | `dnf upgrade pkg` | Direct equivalent |
| Search package name/desc | `apt search term` | `dnf search term` | Direct equivalent |
| Show package metadata | `apt show pkg` | `dnf info pkg` | Direct equivalent |
| List installed | `apt list --installed` | `dnf list installed` | Direct equivalent |
| Remove unused deps | `apt autoremove` | `dnf autoremove` | Direct equivalent |
| Clean download cache | `apt clean` | *(dnf cleans automatically; `dnf clean all` for manual)* | |
| Which package owns a file | *(no apt-level command — use `dpkg -S`)* | `dnf provides /path` | dnf can also query NOT-YET-installed packages; `dpkg -S` cannot |
| Transaction history | *(no built-in — read `/var/log/apt/history.log`)* | `dnf history` | No apt equivalent |
| Undo a transaction | *(no equivalent — manual recovery only)* | `dnf history undo N` | No apt equivalent — notable gap |
| Hold/lock version | `apt-mark hold pkg` | `dnf versionlock add pkg` (requires plugin install) | apt: built-in. dnf: separate plugin |

### C.2 Low-level tool: dpkg vs rpm

| Action | dpkg (Debian/Ubuntu) | rpm (RHEL/Rocky) | Notes |
|---|---|---|---|
| List all installed | `dpkg -l` | `rpm -qa` | Direct equivalent |
| List specific package | `dpkg -l pkg` | `rpm -q pkg` | Direct equivalent |
| Package metadata | *(no single dpkg flag — use `apt show`)* | `rpm -qi pkg` | dpkg has no direct metadata-display flag |
| List files in package | `dpkg -L pkg` | `rpm -ql pkg` | Direct equivalent |
| Which package owns a file | `dpkg -S /path` | `rpm -qf /path` | Direct equivalent (installed files only, both sides) |
| Install local package file | `dpkg -i file.deb` | `rpm -ivh file.rpm` | Direct equivalent |
| Remove package | `dpkg -r pkg` | `rpm -e pkg` | Direct equivalent |
| Purge (+ config) | `dpkg -P pkg` | *(no equivalent)* | See apt/dnf table — RPM has no purge concept |
| Verify file integrity | *(no equivalent)* | `rpm -V pkg` | Notable gap — dpkg has no built-in verify |
| Import GPG key | *(not a dpkg function — handled via apt keyring files)* | `rpm --import URL` | Different mechanisms entirely, not just different syntax |
| View/set selections (hold etc.) | `dpkg --get-selections` | *(handled by dnf versionlock plugin instead)* | Different underlying systems |

### C.3 What has no equivalent at all

Memorize these as gaps, not commands — knowing what *doesn't* exist
on one side is exactly what prevents confusion when moving between
distro families.

| Capability | Exists on | Missing on |
|---|---|---|
| Built-in transaction undo | RHEL (`dnf history undo`) | Debian — no equivalent |
| Built-in file integrity verification | RHEL (`rpm -V`) | Debian — no equivalent |
| Query not-yet-installed package by file | RHEL (`dnf provides`) | Debian — `dpkg -S` only works on installed files |
| Native purge concept | Debian (`apt purge` / `dpkg -P`) | RHEL — `dnf remove` already removes config in most cases |
| Hold without installing a plugin | Debian (`apt-mark`, built-in) | RHEL — `versionlock` requires separate plugin install |

---

## Appendix D — Deep Dives

Optional. The main walkthrough gives you enough to work
confidently day to day. This appendix exists for the "but how does
that actually work underneath" questions — read it when curious,
skip it if you just need to get things done.

### D.1 Package archive formats — internal structure

A `.deb` file is an `ar` archive (the same generic archive format
`ar` has used for decades, originally for static libraries) — not
a tar or zip file. It contains exactly three members, in order:

```
debian-binary      A single line stating the format version (e.g. "2.0")
control.tar.xz     Package metadata: name, version, dependencies,
                   maintainer scripts (preinst, postinst, prerm, postrm)
data.tar.xz        The actual files, in the exact paths they'll occupy
                   once installed (e.g. ./usr/bin/curl)
```

A `.rpm` file is a different, custom binary format with four
sections, in order:

```
Lead               Legacy header for old RPM version identification
                   (mostly vestigial on modern systems)
Signature header   The GPG signature and size/checksum info used
                   for the install-time verification covered in §1.5
Header             All package metadata: name, version, dependencies,
                   file list, and — critically — the size/MD5/
                   permission/owner/mtime values rpm -V checks
                   against later (§1.6)
Payload            A compressed cpio archive containing the actual files
```

**Why this matters practically:** both formats are inspectable
with generic tools, because they're built on standard archive
formats underneath:

```bash
# Inspect a .deb without installing it
ar x package.deb            # extracts the three members listed above
tar -tJf control.tar.xz     # list metadata contents
tar -tJf data.tar.xz        # list files that would be installed

# Inspect a .rpm without installing it
rpm -qlp package.rpm        # list files (query, local Package, before install)
rpm2cpio package.rpm | cpio -t   # list files via the raw cpio payload
```

The reason a `.deb`/`.rpm` can support dependency resolution,
post-install scripts, and "which package owns this file" queries
— none of which a plain `.zip`/`.tar` can do — is precisely
because the metadata sections above travel embedded with the
files, not because of anything special about the file compression
itself.

### D.2 GPG signature verification — the actual mechanism

GPG (GNU Privacy Guard) is an implementation of asymmetric
(public-key) cryptography. Two mathematically related keys exist:
a **private key**, kept secret by the repo maintainer, and a
**public key**, freely distributed to anyone (the key you `curl`
or `rpm --import` in section 1.5).

**Note on the description below:** it describes the RSA-style
mental model of signing — "encrypt the hash with the private key" —
which is the simplest correct picture and the one worth having by
default. GPG's other supported algorithms (DSA, ECDSA) use a
related but mathematically distinct signing operation, not literally
"encryption" of the hash — the verify/reject logic below (steps and
outcome) holds for all of them regardless of which algorithm signed
a given key.

**Signing, at the maintainer's end, before publishing:**

1. The maintainer computes a cryptographic hash of the index file
   (the `Release`/`InRelease` for Debian, `repomd.xml` for RHEL)
2. The maintainer encrypts that hash using their **private** key —
   this encrypted hash is "the signature"
3. The signature is published alongside the index file itself

**Verifying, at your end, before installing:**

1. Your package manager re-computes the hash of the index file it
   just downloaded
2. It decrypts the published signature using the **public** key
   you imported, recovering the maintainer's original hash
3. If your freshly computed hash matches the decrypted one, the
   index is provably (a) unmodified since signing and (b) signed
   by whoever holds the matching private key

**Why this can't be faked without the private key:** only the
private key can produce a signature that the corresponding public
key will successfully decrypt back to a valid hash. An attacker
who modifies the index file changes its hash; without the private
key, they cannot produce a new signature that the public key
verification step would accept. This is why `NO_PUBKEY`/`GPG check
FAILED` is a hard failure, not a warning — the package manager has
no way to know whether the content is legitimate without
successfully completing this check.

**Why GPG signing, not encryption:** the repo content (package
files, index) is meant to be publicly downloadable by anyone —
encrypting it would prevent that. Signing instead proves
authenticity and integrity without restricting access.

### D.3 The RPM database — internal structure

`/var/lib/rpm/rpmdb.sqlite` (on modern RPM versions; older systems
use a Berkeley DB file at the same path) is a real SQLite database
file. It stores, per installed package, every field needed for the
queries and verification covered throughout this demo:

```
Per-package record includes:
  - Package metadata: name, version, release, architecture,
    description, maintainer, install time
  - Dependency graph: what this package requires, what it provides
  - GPG signature info captured at install time
  - File manifest: every file path this package owns, plus for
    each file — the expected size, MD5 hash, permission bits,
    owner, group, mtime, and device number recorded at install time
```

This file manifest with expected per-file values is exactly what
every `rpm -q*` query and `rpm -V` verification reads from — `rpm
-ql` reads the file path list, `rpm -qf` searches the file→package
mapping in reverse, and `rpm -V` is the only command that actually
**recomputes** current values and diffs them against this stored
record rather than simply displaying it.

You can inspect the database's existence and basic health
directly:

```bash
# Confirm the database file and its size
ls -lh /var/lib/rpm/rpmdb.sqlite

# Rebuild the database if it becomes corrupted (rare, but happens
# after an interrupted install or filesystem issue)
sudo rpm --rebuilddb
```

`dpkg`'s equivalent database is simpler by comparison — plain text
files at `/var/lib/dpkg/status` (one large file, one stanza per
package) and `/var/lib/dpkg/info/` (per-package maintainer
scripts and file lists) — not a structured database file. This is
part of why `rpm -V`'s integrity-check capability has no direct
dpkg equivalent (see the comparison gap table above): the
structured per-file expected-value storage that makes verification
possible is built into RPM's database design from the start.

## Appendix E — End-to-End Install/Upgrade Flow

This appendix walks the complete request → disk sequence for both
`apt install`/`apt upgrade` and `dnf install`/`dnf upgrade`,
side by side — every file touched, in what order, and which value
wins when more than one source could apply.

### E.1 Install flow — `apt install curl` (Ubuntu)

```
1. Read local index cache
   /var/lib/apt/lists/*Packages
   (NOT the live repo — this is the cached copy from your last
   `apt update`. If you haven't run update, this may be stale or
   missing entirely on a fresh system.)

2. Resolve dependencies
   Check Depends:/Recommends: fields in the index entry for curl.
   Walk this recursively for each dependency.

3. Check pin priority, if any pin files exist
   /etc/apt/preferences.d/*
   Default priority is 500 for any available version, unless a
   pin file raises/lowers it for specific packages (section 1.4).
   Highest-priority candidate wins.

4. Download the .deb file(s)
   From the repo URL in /etc/apt/sources.list.d/*.list
   Verify GPG signature using the key referenced by signed-by=
   in that .list file (section 1.5) — fails here with
   NO_PUBKEY/GPG check FAILED if the key isn't imported.

5. Unpack and run pre-install scripts
   dpkg unpacks the .deb's data.tar.xz, runs any preinst script
   from control.tar.xz (Appendix D, section D.1)

6. Place files on disk
   Following the FHS pattern from Demo M01-07/this demo's section 1.1
   connecting content — binary to /usr/bin/, config to /etc/,
   etc.

7. Run post-install scripts
   postinst script — may create a service account, enable a
   systemd unit, etc.

8. Write the install record
   /var/lib/dpkg/status     — plain text, one stanza per package
   /var/lib/dpkg/info/      — per-package file lists, scripts

   This is what every later dpkg -l/-L/-S query reads — NOT the
   original .deb file, which is discarded after step 4 unless you
   keep it manually.
```

### E.2 Install flow — `dnf install curl` (Rocky9)

```
1. Read local index cache
   /var/cache/dnf/<repo-id>/repodata/*.xml (or .solv, dnf's own
   compiled cache format)
   dnf checks repo metadata freshness automatically — unlike apt,
   there's no separate "update" step you must remember to run
   first (section 1.3's "no separate update command" point).

2. Resolve dependencies
   dnf's solver (libsolv) computes the full dependency graph,
   including any virtual package "provides" matches.

3. Check versionlock, if the plugin is installed and active
   dnf versionlock's config file — restricts candidate versions
   to whatever was locked (section 1.4).

4. Download the .rpm file(s)
   From the baseurl= in the matching /etc/yum.repos.d/*.repo file
   Verify GPG signature against keys already imported via
   rpm --import (section 1.5) — fails here with
   "GPG check FAILED" if the key isn't present.

5. Unpack and run pre-install scriptlets
   rpm unpacks the cpio payload, runs any %pre scriptlet from the
   Header section (Appendix D, section D.1)

6. Place files on disk
   Following the same FHS pattern as the apt flow above —
   package manager doesn't change where files land, only how
   they get there.

7. Run post-install scriptlets
   %post scriptlet — same purpose as Debian's postinst

8. Write the install record
   /var/lib/rpm/rpmdb.sqlite — a real SQLite database (Appendix
   D, section D.3), not plain text files.

   This is what every later rpm -qa/-ql/-qf/-V query reads — NOT
   the original .rpm file, which is discarded after step 4 unless
   kept manually.
```

### E.3 Upgrade flow — what's different from install

```
apt upgrade curl                      dnf upgrade curl
─────────────────────                 ─────────────────────
Reads /var/lib/dpkg/status to         Reads /var/lib/rpm/rpmdb.sqlite
find the CURRENTLY installed          to find the CURRENTLY installed
version first                         version first

Compares against the candidate        Compares against the candidate
version in the local index            version in the local index
(same staleness risk as install       (dnf checks freshness itself,
if apt update wasn't run recently     reducing but not eliminating
— section 1.2)                        the same risk)

If held (apt-mark hold), SKIPS        If versionlocked, SKIPS this
this package silently, printing       package, printing "Skipping
"kept back" in the summary            packages with conflicts"
(section 1.4)                         (section 1.4)

Old .deb's files are removed by       Old .rpm's files are removed by
dpkg using the OLD install record's   rpm using the OLD install record's
file manifest, then the NEW           file manifest, then the NEW
package's files are placed and        package's files are placed and
the database record is REPLACED,      the database record is REPLACED,
not appended                          not appended

A failed/interrupted upgrade can      dnf history (section 1.3) lets
leave dpkg in a "half-configured"     you see exactly which
state — dpkg --configure -a           transaction an upgrade was,
attempts to finish a stalled          and dnf history undo N can
configuration step                    reverse it entirely — no dpkg
                                       equivalent exists
```

### E.4 Priority — which value wins, when more than one source applies

```
For a package version, in order of what's actually checked first:

1. A versionlock/hold entry, if one exists       → wins outright,
   (dnf versionlock / apt-mark hold)                upgrade refuses
                                                     to touch it

2. A pin file with Pin-Priority > 1000            → wins even over
   (apt only — /etc/apt/preferences.d/)              a newer version
                                                     that would
                                                     otherwise install

3. The highest version available in the           → normal default
   local index (apt) / repo metadata (dnf)            behavior

4. Pin files with priority 1-999                   → only break ties
   between multiple AVAILABLE repos offering
   the same package, doesn't override #3 alone


For a CONFIG FILE, which copy wins after install (relevant for both
distros, mechanism differs slightly):

apt:  if you've locally modified a config file and a package
      upgrade ships a new default version, apt PROMPTS you
      interactively (in a terminal) to keep yours, take the new
      one, or view a diff — never silently overwrites a modified
      config file by default

dnf:  ships the new default as filename.rpmnew alongside your
      existing (preserved) file, rather than prompting — you must
      manually diff and merge .rpmnew files yourself; nothing
      prompts you that this happened unless you check for .rpmnew
      files explicitly after an upgrade
```