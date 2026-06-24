# Demo 04 — Package Management

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

1. Query what is installed, search for packages, and inspect
   package metadata on both Ubuntu and Rocky9
2. Install, update, and remove packages cleanly on both distros
3. Hold a package at a specific version and release the hold
4. Add a third-party repository with its GPG key on both distros
5. Verify the integrity of installed package files using `rpm -V`
6. Read and interpret `rpm -V` output flags
7. Write a correct `Dockerfile` `RUN` layer for package installation —
   pinned versions, no cache, no unnecessary packages
8. Explain the relationship between the high-level tool (`apt`/`dnf`)
   and the low-level tool (`dpkg`/`rpm`)

---

## Directory Structure

```
04-package-management/
├── README.md                          # this file
├── 04-package-management-anki.csv     # Anki flashcard deck
├── 04-package-management-quiz.md      # standalone quiz
└── src/
    └── Dockerfile                     # DevOps base image — correct layer pattern
```

---

## Environments

| Task | Ubuntu 24.04 | Rocky Linux 9.8 | EC2 Amazon Linux 2023 |
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

> Answer from memory before reading further. These are from Demo 03.

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

### 1.1 What a package manager does

A **package** is an archive containing:
- The compiled binary or script files
- Default configuration files
- Metadata: name, version, description, maintainer, dependencies
- Pre/post install scripts (run automatically during install)
- A file manifest: every file this package owns and where it goes

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

### 1.2 apt and dpkg — Debian family

**`apt`** = **A**dvanced **P**ackage **T**ool. The standard interface
for package management on Ubuntu and all Debian-family systems.

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

**Reading `dpkg -l` output:**

```
dpkg -l nginx
Desired  Status  Error
||/          Name            Version
ii           nginx           1.24.0-...
↑↑
││
│└── Installed status: i = installed, c = config only, n = not installed
└─── Desired state:    i = install, h = hold, r = remove, p = purge

ii = desired:install + status:installed = package is installed and should be
```

### 1.3 dnf and rpm — RHEL family

**`dnf`** = **D**andified **Y**um. The standard package manager for
RHEL 8+, Rocky Linux 9, Fedora, and Amazon Linux 2023. Replaced
`yum` (which remains as a compatibility alias on some systems).

**Core dnf commands:**

```bash
dnf install nginx               # install nginx and dependencies
dnf install nginx-1.24.0        # install specific version
dnf install -y nginx            # -y = skip confirmation

dnf remove nginx                # remove package (may leave config files)
dnf autoremove                  # remove unused dependency packages

dnf upgrade                     # upgrade all packages
dnf upgrade nginx               # upgrade one package

dnf search nginx                # search package names and summaries
dnf info nginx                  # detailed package information
dnf list installed              # list all installed packages
dnf list installed | grep nginx # check specific package

dnf provides /usr/sbin/nginx    # which package provides this file?
                                # equivalent to dpkg -S on Debian

dnf history                     # list all past transactions (installs, removes)
dnf history info 5              # detail of transaction 5
dnf history undo 5              # undo transaction 5 — reinstall removed or
                                # remove installed packages from that transaction
```

**`dnf history undo`** — the ability to undo a transaction — is a
significant advantage over `apt`. On an RHEL system you can roll
back a botched `dnf upgrade` with one command.

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

**Alternative — pin with `/etc/apt/preferences.d/`:**

```
# /etc/apt/preferences.d/nginx-pin
Package: nginx
Pin: version 1.24.*
Pin-Priority: 1001
```

Pin-Priority > 1000 means "prefer this version even if a newer
one is available and would normally be upgraded." Pinning is more
flexible than hold — you can pin to a version pattern (`1.24.*`)
rather than one exact version.

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

When a locked package would be upgraded, `dnf upgrade` skips it:
```
Skipping packages with conflicts:
  nginx  (versionlock)
```

### 1.5 Repository management

A **repository** (repo) is a server hosting packages and metadata.
The package manager reads an index from the repo, resolves what to
download, verifies signatures, and installs.

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

**GPG key import — why it is mandatory:**

Every package in a signed repository is signed with the repo
maintainer's GPG private key. The package manager verifies this
signature against the public key before installing. If the key
is not imported, installation fails with `NO_PUBKEY` or
`GPG check FAILED`.

This is not optional ceremony — it is the only guarantee that
the package you are installing is the one the maintainer published
and has not been tampered with in transit.

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

### 1.6 rpm -V — verifying package integrity

`rpm -V` (**V**erify) compares every file installed by a package
against the RPM database record of what it should look like
at install time: size, MD5 hash, permissions, owner, group,
modification time, and device.

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
Layers are cached. Getting this wrong either breaks reproducibility
or bloats the image.

**The correct pattern:**

```dockerfile
RUN apt-get update \
    && apt-get install -y --no-install-recommends \
       curl=8.5.0-2ubuntu10.6 \
       git=1:2.43.0-1ubuntu7.1 \
       jq=1.7.1-3build1 \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/*
```

**Why every part of this pattern matters:**

```
apt-get update                   Refresh the package index. Without this,
                                 the base image's cached index may be stale —
                                 you might install an old version or fail to
                                 find the package at all.

&& (all in one RUN layer)        apt-get update and apt-get install must be
                                 in the SAME RUN instruction. If they are in
                                 separate RUN layers, Docker caches each layer.
                                 A future build may use the cached update layer
                                 (stale index) but re-run install — installing
                                 whatever version the stale index knows about.
                                 This is called cache busting failure.

-y                               Non-interactive. Required in Dockerfiles —
                                 no terminal to answer prompts.

--no-install-recommends          apt installs recommended packages by default.
                                 Recommended packages are not required for the
                                 tool to function. In a container, skip them.
                                 Typical image size reduction: 20–40%.

package=version                  Pin exact versions. Without pinning, a fresh
                                 build 6 months later may install a different
                                 version than today. Pinned versions make builds
                                 reproducible.

apt-get clean                    Delete downloaded .deb files from the cache.
                                 The files in /var/cache/apt/archives/ are no
                                 longer needed after install.

rm -rf /var/lib/apt/lists/*      Delete the downloaded package index. Not needed
                                 at runtime. Must be in the SAME RUN layer —
                                 if in a separate layer, the files still exist
                                 in the earlier cached layer and are included in
                                 the image even if invisible in later layers.
```

**What NOT to do:**

```dockerfile
# WRONG — separate update and install layers
RUN apt-get update
RUN apt-get install -y curl git

# WRONG — no version pinning
RUN apt-get update && apt-get install -y curl git jq

# WRONG — apt-get upgrade in a container
RUN apt-get update && apt-get upgrade -y && apt-get install -y curl
# apt-get upgrade in a container replaces the base image's packages
# with newer versions — defeating the purpose of a pinned base image.
# Security patches should come from rebuilding with a newer base image,
# not from running upgrade inside the container.

# WRONG — cleaning in a separate layer
RUN apt-get update && apt-get install -y curl
RUN apt-get clean && rm -rf /var/lib/apt/lists/*
# The cache still exists in the install layer — Docker includes it.
# The clean layer only hides it. Image is the same size.
```

> **Best Practice —** In production Dockerfiles, pin both the base image
> tag (`ubuntu:24.04`, never `ubuntu:latest`) and each package version.
> This ensures that rebuilding the image 6 months later produces
> bit-for-bit identical installed software. Demo only: package versions
> shown are illustrative — verify current versions with
> `apt-cache policy packagename` on a live Ubuntu 24.04 system before
> pinning in a real Dockerfile.

---

## Part 2 — Hands-On Lab

### Commands used in this demo

| Command | Letter meaning | Purpose |
|---|---|---|
| `apt update` | Advanced Package Tool | Refresh package index |
| `apt install` | — | Install packages |
| `apt show` | — | Show package metadata |
| `apt-mark hold` | — | Hold package at current version |
| `dpkg -l` | l = list | List installed packages |
| `dpkg -S` | S = Search | Find which package owns a file |
| `dpkg -L` | L = List files | List files owned by a package |
| `dnf install` | Dandified Yum | Install packages (RHEL family) |
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
# First: find current pinnable versions for our packages
apt-cache policy curl git unzip jq ca-certificates gnupg

# Note the Installed and Candidate versions shown
# Use the Candidate version for pinning in Dockerfile
```

The Dockerfile below uses the correct pattern with realistic
DevOps base tooling — single `RUN` layer, pinned versions,
`--no-install-recommends`, cache cleaned in the same layer.
Save it as `src/Dockerfile`, then build and verify the result.


**`src/Dockerfile`:**

```dockerfile
# DevOps base image — Demo 04: Package Management
#
# Demonstrates the correct Dockerfile RUN layer pattern for apt:
#   - apt-get update and install in one RUN instruction (cache busting)
#   - --no-install-recommends (smaller image)
#   - Pinned package versions (reproducible builds)
#   - Cache cleaned in the same layer (no hidden bloat)
#
# Base image pinned to specific digest tag — never use :latest
FROM ubuntu:24.04

# Prevent interactive prompts during package install
ENV DEBIAN_FRONTEND=noninteractive

# Single RUN layer: update + install + clean
# All three steps must be in one instruction — see Part 1 section 1.7
RUN apt-get update \
    && apt-get install -y --no-install-recommends \
       # Core utilities — present in virtually every DevOps base image
       curl=8.5.0-2ubuntu10.6 \
       ca-certificates=20240203 \
       gnupg=2.4.4-2ubuntu17 \
       unzip=6.0-28ubuntu4 \
       # Data processing — jq is used in almost every shell script
       # that interacts with APIs, Kubernetes, or cloud CLIs
       jq=1.7.1-3build1 \
       # Version control — required for fetching code in CI environments
       git=1:2.43.0-1ubuntu7.1 \
       # Process utilities — useful for health checks and debugging
       procps=2:4.0.4-4ubuntu3 \
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
    && gnupg2 --version \
    && unzip -v | head -1

# Default to bash — explicitly set for clarity
CMD ["/bin/bash"]

# Build:   docker build -t devops-base:demo04 .
# Run:     docker run --rm devops-base:demo04 jq --version
# History: docker history devops-base:demo04
#          (should show one RUN layer for all package install steps)
```

```bash
cd ~/path/to/04-package-management/src

# Build the image
docker build -t devops-base:demo04 .

# Verify installed packages inside the container
docker run --rm devops-base:demo04 curl --version
docker run --rm devops-base:demo04 git --version
docker run --rm devops-base:demo04 jq --version

# Inspect image layers — see the single RUN layer
docker history devops-base:demo04

# Check image size
docker images devops-base:demo04

# Clean up
docker rmi devops-base:demo04
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

**Paste your `docker history` output and the final image size.**

---

## What You Learned

- ✅ Queried installed packages, searched for packages, and inspected package metadata on both Ubuntu and Rocky9
- ✅ Installed, updated, and removed packages cleanly on both distros
- ✅ Held a package at a specific version and released the hold
- ✅ Added a third-party repository with its GPG key on both distros
- ✅ Verified the integrity of installed package files using `rpm -V`
- ✅ Read and interpreted `rpm -V` output flags
- ✅ Wrote a correct `Dockerfile` `RUN` layer for package installation — pinned versions, no cache, no unnecessary packages
- ✅ Explained the relationship between the high-level tool (`apt`/`dnf`) and the low-level tool (`dpkg`/`rpm`)

**Key Takeaway:** Package management failures almost always trace back to one of three things: a stale index, a missing GPG key, or a Docker layer caching the wrong state. Knowing which of the three you're looking at turns a confusing error into a two-command fix.

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

Docker caches each `RUN` layer. If `apt-get update` and
`apt-get install` are in separate layers, a future build can
use the cached `update` layer (with a stale package index)
while re-running `install` — installing whatever version the
stale index knows about rather than the latest. This is called
cache busting failure and causes non-reproducible builds.
Combining them in one `RUN` ensures the index is always fresh
when packages are installed.

---

## Key Takeaways

1. **`apt`/`dnf` are the high-level tools that resolve dependencies and fetch from repos; `dpkg`/`rpm` are the low-level tools that manage the package database and install files.** Use the high-level tools for install/remove, the low-level tools for queries and verification.

2. **Always run `apt update` before `apt install` — never assume the local package index is current.** A stale index silently installs old versions or fails to find packages added after your last update.

3. **Hold packages that have strict version coupling: `apt-mark hold` on Ubuntu, `dnf versionlock` on RHEL.** Kubernetes nodes, database servers, and anything with cluster-wide version requirements need this — an unintended upgrade is a production incident.

4. **GPG key import is mandatory before installing from a third-party repo — skipping it causes `GPG check FAILED` at install time, not at repo-add time.** On RHEL: `rpm --import URL`. On Debian: `gpg --dearmor | tee /etc/apt/keyrings/name.gpg`. The repo file points to the key but does not import it automatically.

5. **`rpm -V` prints nothing when a package is intact — any output means a file differs from the database record.** `c` in the file type column means config file (expected to be modified). No tag on a modified binary is a security indicator requiring immediate investigation.

6. **In a Dockerfile: `apt-get update && apt-get install` in one `RUN` layer, `--no-install-recommends`, pinned versions, and cleanup in the same layer.** Separate layers cause stale index builds; cleanup in a separate layer does not reduce image size because the files exist in the earlier layer.

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

**`04-package-management-anki.csv`:**

```
#deck:Linux Mastery::Module 2 - System Administration::Demo 04 - Package Management
#separator:Comma
#columns:Front,Back,Tags

"You run dnf install filebeat and get 'GPG check FAILED'. The repo file exists and gpgcheck=1. What was skipped and what is the fix?","The GPG key was never imported. The repo file's gpgkey= line tells dnf where to find the key but does not import it automatically. Fix: sudo rpm --import https://repo-url/GPG-KEY then retry install. On Debian: curl -fsSL URL | gpg --dearmor | sudo tee /etc/apt/keyrings/name.gpg","demo04,repos,gpg,security"

"rpm -V openssh-clients returns: S.5....T.  /usr/bin/ssh (no c tag). What does this mean and how serious is it?","Size (S), MD5 hash (5), and mtime (T) all differ from the package database record. No 'c' tag means this is a regular binary — not a config file. A modified ssh binary with no distro update to explain it is a serious security indicator. Remediation: check dnf history for a legitimate update. If none found, isolate host, preserve for forensics, reinstall: dnf reinstall openssh-clients","demo04,rpm-verify,security"

"What is the difference between apt remove and apt purge? When do you use each?","apt remove uninstalls binaries but leaves config files. apt purge removes binaries AND config files. Use remove when you may reinstall and want to keep config. Use purge for a clean slate — reinstalling fresh, or permanent removal. In containers and CI, always purge to avoid config residue affecting subsequent installs.","demo04,apt,remove,purge"

"A Kubernetes worker node's kubelet upgraded unexpectedly from 1.29 to 1.30 during dnf upgrade. Node is NotReady. What should have been done after the initial install?","dnf versionlock add kubelet kubeadm kubectl (RHEL) or apt-mark hold kubelet kubeadm kubectl (Ubuntu). All three packages must be version-locked together — they must stay in lockstep with the control plane. The lock is released, upgrade performed at planned version, then re-applied.","demo04,versionlock,kubernetes"

"Why must apt-get update and apt-get install be in the same RUN instruction in a Dockerfile?","Docker caches each RUN layer. If they are separate, a future build uses the cached update layer (stale index) but re-runs install — installing whatever version the stale index knows, not the latest. This is cache busting failure causing non-reproducible builds. One RUN layer ensures the index is always fresh when packages install.","demo04,dockerfile,apt,caching"

"What does --no-install-recommends do in apt-get install? Why use it in containers?","Prevents installation of recommended packages — packages not strictly required for the tool to function. apt installs recommended packages by default. In a container, recommended packages add size with no benefit. Typical image size reduction: 20-40%. Always use --no-install-recommends in Dockerfiles.","demo04,dockerfile,apt,containers"

"rpm -V nginx returns nothing. rpm -V nginx shows: .M.......  /usr/sbin/nginx. What does each mean?","No output: all files match the package database — package is intact. .M.......: the M position (mode/permissions) differs from the database record. Permissions on the nginx binary were changed after install. Investigate: was this intentional hardening? Run stat /usr/sbin/nginx to see current permissions. May be benign but warrants review.","demo04,rpm-verify,permissions"

"What is the relationship between apt and dpkg? Between dnf and rpm?","apt is the high-level tool: resolves dependencies, fetches packages from repos, then calls dpkg to install. dpkg is the low-level tool: installs/removes .deb files, manages the package database, does NOT resolve deps. Same relationship: dnf (high-level) calls rpm (low-level). Use high-level for install/remove. Use low-level for queries (dpkg -S, rpm -qf) and verification (rpm -V).","demo04,apt,dpkg,dnf,rpm"
```

---

## Appendix B — Quiz

**`04-package-management-quiz.md`:**

````markdown
# Quiz — Demo 04: Package Management

> One correct answer per question unless stated otherwise.
> Target: 80% or above before moving to Demo 05.

---

**Q1.** You add the Elastic repo on Rocky9 and run `dnf install filebeat`.
The package downloads successfully but install fails with
`Error: GPG check FAILED`. What is the cause?

- A) The package is corrupted — download again
- B) `gpgcheck=1` in the repo file is too strict — set it to 0
- C) The GPG key URL in the repo file is incorrect
- D) The GPG key was never imported into the RPM database —
  `rpm --import` must be run explicitly before install

<details>
<summary>Answer</summary>

**D** — The repo file's `gpgkey=` line tells dnf where to find the key but
does not import it. `rpm --import URL` must be run explicitly.
The package downloaded fine — the failure is at signature verification.

Trap: B would disable signature checking entirely — a serious security
regression. Never set `gpgcheck=0` in production.

</details>

---

**Q2.** `rpm -V curl` returns no output. What does this mean?

- A) The `curl` package is not installed
- B) `rpm -V` requires root to run — rerun with sudo
- C) All files in the `curl` package match the RPM database record —
  the package is intact
- D) The RPM database is corrupted and cannot verify

<details>
<summary>Answer</summary>

**C** — `rpm -V` prints nothing when every file passes all checks
(size, MD5, permissions, owner, group, mtime). Output only appears
when something differs. Silence is the success state.

Trap: A is wrong — if the package were not installed, rpm -V would
print an error, not silence.

</details>

---

**Q3.** `rpm -V openssh-server` returns `S.5....T.  c  /etc/ssh/sshd_config`.
A junior engineer flags this as a security incident. How do you respond?

- A) Agree — any rpm -V output indicates compromise
- B) The `c` tag means this is a config file. Config files are
  expected to be modified after install. This is normal — the
  sshd config was edited by an administrator.
- C) The file must be restored from the package immediately
- D) This means the package was installed incorrectly

<details>
<summary>Answer</summary>

**B** — The `c` tag marks a config file. `rpm -V` flags any difference
from the install-time state, including intentional admin edits.
Modified config files with a `c` tag are expected and normal.

The concern is `S.5....T.` on a **binary** with no `c` tag —
that would indicate the binary itself was modified, which is a
security indicator.

</details>

---

**Q4.** A Dockerfile has these two instructions:
```
RUN apt-get update
RUN apt-get install -y curl git
```
What is wrong with this pattern?

- A) `apt-get update` and `apt-get install` must be run as separate users
- B) Docker caches each RUN layer — a future build may use the cached
  update layer with a stale index, installing old package versions.
  Both must be in one RUN instruction.
- C) `-y` flag is not valid in Dockerfiles
- D) `apt-get` must be replaced with `apt` in containers

<details>
<summary>Answer</summary>

**B** — Docker layer caching causes the update layer to be reused on
future builds. The install layer runs with a stale package index.
Fix: `RUN apt-get update && apt-get install -y --no-install-recommends curl git && apt-get clean && rm -rf /var/lib/apt/lists/*`

Trap: D is wrong — `apt-get` is preferred over `apt` in Dockerfiles
and scripts because its interface is stable; `apt` is for interactive use.

</details>

---

**Q5.** After running `apt remove nginx`, `dpkg -l nginx` shows `rc`.
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

**Q6.** On Ubuntu, you want to prevent `kubelet` from being upgraded
by `apt upgrade`. Which command achieves this?

- A) `apt-mark hold kubelet`
- B) `apt pin kubelet`
- C) `dpkg --hold kubelet`
- D) `apt-get lock kubelet`

<details>
<summary>Answer</summary>

**A** — `apt-mark hold kubelet` places the package in hold state.
`apt upgrade` skips held packages and lists them under
"The following packages have been kept back". Release with:
`apt-mark unhold kubelet`. View held packages: `apt-mark showhold`.

Trap: B and D are not valid commands. C uses wrong syntax —
`dpkg` does not have a `--hold` flag; holds are managed through
`apt-mark` or `dpkg --set-selections`.

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
| 7/7 | Import Anki cards, move to Demo 05 |
| 6/7 | Review the wrong answer, then proceed |
| 5/7 | Re-read the relevant section, retry those questions |
| Below 5/7 | Re-read the full demo and redo the walkthrough before proceeding |
````