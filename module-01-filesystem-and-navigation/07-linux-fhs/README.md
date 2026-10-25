# Demo 01 — Linux Filesystem Hierarchy (FHS)

## Overview

Every action you take on a Linux system — editing a config, reading a
log, installing a tool, debugging a failing service — happens inside a
specific directory. That directory is not random. Linux follows a
standard called the **Filesystem Hierarchy Standard (FHS)** that
defines what lives where and why.

Without this knowledge you spend time searching for files instead of
fixing problems. With it, you know immediately:

- Service broken → check `/etc` for config, `/var/log` for logs
- Tool not found → check `/usr/local/bin` and `$PATH`
- Disk full → check `/var/lib` and `/var/log`
- Runtime socket missing → check `/run`

This is the map you carry into every other demo in this series.

---

## Demo Objectives

By the end of this demo you will be able to:

1. Name the purpose of every major directory under `/`
2. Explain the difference between virtual filesystems (`/proc`,
   `/sys`, `/dev`) and tmpfs filesystems (`/run`, `/tmp`)
3. Know immediately where configs, logs, binaries, runtime state,
   and temporary files live — and why
4. Explain the three binary installation patterns: package manager,
   manual install, and self-contained vendor package
5. Navigate and explore the filesystem using `ls`, `cd`, `tree`,
   and `file`
6. Read and understand the output of `ls -lh`, `df -h /proc`,
   `/proc/version`, `/proc/uptime`, and `/proc/$$/status`

---

## Directory Structure

```
01-linux-fhs/
├── README.md                              # this file
├── 01-linux-fhs-anki.csv        # Anki flashcard deck
└── 01-linux-fhs-quiz.md         # standalone quiz
```

---

## Environments

| Task | Ubuntu 24.04 | Rocky Linux 9.8 | EC2 Amazon Linux |
|---|---|---|---|
| All lab tasks | ✅ | ✅ | Not required |
| SELinux behaviour | ❌ AppArmor | Limited | ✅ Full SELinux |

Run every lab task on **both Ubuntu and Rocky9**. Note the differences
— they are real production differences you will encounter in jobs.
EC2 is not required for this demo.

---

## Recall Check — Demo 00f

Before starting, answer from memory:

1. What specific signal kills a plain background job when its
   terminal session ends?
2. A job is started with `cmd &`, then `disown` is run (no
   `nohup`). It's later sent an actual `SIGHUP` directly. Does it
   survive?
3. When would you reach for `tmux` over `nohup` for a long-running
   task?

<details>
<summary>Answers</summary>

1. `SIGHUP` — sent by the kernel to processes still attached to a
   terminal when that terminal disconnects.
2. No — `disown` only removes the job from the current shell's own
   job table; it does not block `SIGHUP` arriving from any other
   source. Only `nohup` makes the process itself immune to the signal.
3. When you'll want to reattach and actively interact with the
   session again later, not just have a command finish unattended —
   `tmux` gives you a full session to return to; `nohup` just keeps
   one command alive with no way back in.

</details>

---

## Part 1 — Concepts

### 1.1 The FHS principle

Linux follows one rule: **everything is a file**. Processes, hardware
devices, kernel settings, network sockets — all exposed as files you
can read and sometimes write. The FHS organises these files into a
predictable tree so any engineer on any Linux system knows where
to look.

### 1.2 The directory map

```
/                    Root — the top of everything
├── etc/             System-wide configuration (static, human-edited)
├── var/             Variable data — changes at runtime
│   ├── log/         Log files from OS and all services
│   ├── lib/         Persistent app state (docker, kubelet, mysql)
│   ├── tmp/         Temp files that SURVIVE reboots
│   └── cache/       App caches (apt, dnf, pip)
├── run/             Runtime state — wiped on every reboot (tmpfs)
├── tmp/             Temporary files — wiped on every reboot (tmpfs)
├── home/            User home directories (/home/username)
├── root/            Root user's home — NOT /home/root (see 1.5)
├── bin/  → usr/bin  User binaries — symlink on modern Linux
├── sbin/ → usr/sbin Admin binaries — symlink on modern Linux
├── usr/             All installed software
│   ├── bin/         User commands managed by package manager
│   ├── sbin/        Admin commands managed by package manager
│   ├── local/       Manually installed software
│   │   └── bin/     kubectl, terraform, helm, aws-cli go here
│   └── share/       Man pages, docs, shared data
├── opt/             Self-contained third-party vendor packages
├── proc/            VIRTUAL — kernel exposes process and system data
├── sys/             VIRTUAL — kernel exposes hardware and driver data
├── dev/             VIRTUAL — kernel exposes device files (disks, terminals)
├── boot/            Kernel and bootloader files
└── mnt/             Temporary manual mount points
```

### 1.3 Virtual filesystems vs tmpfs — critical distinction

This is one of the most misunderstood topics in Linux. These are
NOT the same thing.

**Virtual filesystems — kernel generates content on demand:**

```
/proc    Type: proc (procfs)
/sys     Type: sysfs
/dev     Type: devtmpfs
```

The kernel generates their contents **on demand** — nothing is
stored anywhere between reads. When you `cat /proc/meminfo`, the
kernel computes current memory statistics at that exact moment and
returns them as if they were a file. The data does not exist until
you ask for it. It is not pre-populated at boot and held in RAM —
it is computed when accessed and discarded immediately after.

An important distinction for `/dev`: the kernel does create some
device node files in `devtmpfs` as hardware is detected during boot
(your disk appears as `/dev/sda`, your terminal as `/dev/tty`), but
these nodes are tiny metadata entries, not stored data. The actual
hardware data still flows through the kernel, not through files on
disk.

Characteristics of all three:
- `df -h /proc` → Size 0, Use% shows `-` (undefined for zero-size)
- Contents are controlled entirely by the kernel
- You cannot create arbitrary files here
- Permissions shown by `ls` are kernel-controlled, not standard Unix
- `df -h /proc` shows: Size 0, Use% shown as `-`

**tmpfs — RAM-backed storage you can write to normally:**

```
/run     Type: tmpfs — runtime state
/tmp     Type: tmpfs — temporary files
/dev/shm Type: tmpfs — shared memory between processes
```

tmpfs behaves exactly like a normal filesystem — you create files,
delete files, set permissions. The difference from disk: everything
lives in RAM and is completely gone on every reboot.

Characteristics:
- Has actual allocated size (from RAM)
- You write to it normally — services create PID files and sockets
- Cleared completely on every reboot
- Fast — RAM speed, no disk I/O

**Summary table:**

```
Path     Type       Contents      Writable   Cleared      Example
/proc    procfs     on demand     kernel     N/A          /proc/meminfo
/sys     sysfs      on demand     kernel     N/A          /sys/class/net/
/dev     devtmpfs   at detection  kernel     on reboot    /dev/sda
/run     tmpfs      you write     yes        on reboot    sshd.pid
/tmp     tmpfs      you write     yes        on reboot    build artifacts
```

### 1.4 Three things to know about /proc, /sys, /dev

These directories are **virtual** — They have zero disk size.

```
/proc    Live window into the kernel and every running process
         /proc/meminfo       — current memory usage (what `free` reads)
         /proc/cpuinfo       — CPU details (what `lscpu` reads)
         /proc/1234/         — directory for process with PID 1234
         /proc/1234/cmdline  — the command that started that process
         /proc/1234/fd/      — open file descriptors of that process

/sys     Kernel's hardware and driver interface
         /sys/block/         — block devices (disks)
         /sys/class/net/     — network interfaces

/dev     Device files — your hardware as files
         /dev/sda            — your first hard disk
         /dev/null           — the black hole (discard anything written)
         /dev/random         — random number generator
         /dev/tty            — your terminal
```

The practical value: every monitoring tool you will ever use (`top`,
`free`, `ss`, `ps`) reads from `/proc`. Understanding this means you
can go directly to the source when a tool gives you unexpected output.

### 1.5 The six directories you use every day

**`/etc` — configuration**
Every service's config lives here. `sshd_config`, `nginx.conf`,
`fstab`, `hosts`, `passwd`, `sudoers`. If it configures something
on this host, it is in `/etc`. These files are static — they do not
change unless a human or automation script edits them.

Key files you will use in this course:
```
/etc/hosts          hostname to IP mappings
/etc/passwd         user account database
/etc/fstab          filesystem mount configuration
/etc/os-release     OS identity (distro, version, family)
/etc/resolv.conf    DNS server configuration
/etc/sudoers        sudo permission rules
/etc/ssh/           SSH daemon and client configuration
```

**`/var/log` — logs**
When something breaks, you start here. Every service writes its logs
here. Sorted by modification time (`ls -lht`), the most recently
changed file shows what was active last.

Important difference between Ubuntu and Rocky Linux:
```
Ubuntu:  rsyslog daemon writes traditional text files
         /var/log/syslog    — general system log
         /var/log/auth.log  — authentication events
         /var/log/kern.log  — kernel messages

Rocky9:  journald is the primary logging system
         Logs go to binary journal at /var/log/journal/
         No /var/log/syslog — logs exist but in binary format
         Read with: journalctl (not cat)
```

This difference — and how to use `journalctl` — is covered in
full detail in Demo 07 (systemd & Service Management).

**`/var/lib` — persistent app state**
Services that need to remember things between restarts store data
here. This is the most common source of disk-full incidents:
```
/var/lib/docker/    All Docker images, containers, volumes
/var/lib/kubelet/   Kubernetes node state and pod definitions
/var/lib/mysql/     MySQL database files
/var/lib/apt/       apt package database
```
On a busy CI/CD server, `/var/lib/docker` grows silently until
the disk fills and everything stops.

**`/run` — runtime state**
tmpfs cleared on every boot. Services write three types of files:
```
PID files:   /run/sshd.pid      — contains the PID of the running sshd
             Used by systemd and scripts to send signals to services

Sockets:     /run/docker.sock   — Unix socket for Docker CLI ↔ daemon
             /run/systemd/      — systemd's internal communication

Lock files:  /run/lock/         — prevent two instances running at once
```
If a socket or PID file is missing after reboot, the service has not
started yet — it is not a filesystem problem.

**`/usr/local/bin` — your manually installed tools**
The package manager (`apt`, `dnf`) never touches this directory.
This is where you put tools you install yourself:
```
kubectl, helm, terraform, aws-cli, custom scripts
```
Always put manual binary installs here, never in `/usr/bin`.

**`/tmp` — temporary files**
tmpfs, cleared on reboot. For short-lived scratch space in scripts.
Never store logs here — they disappear on reboot, making post-
incident analysis impossible.

### 1.6 Three things that confuse everyone

**Why `/root` is not `/home/root`**

`/home` is often a separate partition, sometimes NFS-mounted,
sometimes mounted late in boot. If root's home were under `/home`
and that mount failed, you would be locked out of your own recovery
console. `/root` sits directly on the root filesystem — always
accessible, even in single-user recovery mode.

**The `/bin → usr/bin` symlink (UsrMerge)**

On all modern distros (Ubuntu 20.04+, Rocky Linux 9, RHEL 9+),
`/bin`, `/sbin`, and `/lib` are symlinks pointing to their `/usr/`
equivalents. This is called UsrMerge. The historical reason (early
Unix used a separate disk for `/usr`) is gone. Scripts that still
use `/bin/bash` work fine — the symlink resolves correctly.

```
Ubuntu shows:  bin.usr-is-merged/  ← explicit marker directory
Rocky9:        no marker, silent merge
Both:          bin -> usr/bin
```

**`/opt` — self-contained vendor packages**

`/opt` is for third-party software that installs as a complete,
self-contained unit — binary, config, data, and libraries all in
one directory tree, not split across `/usr`, `/etc`, `/var`.

```
/opt/novatech/
├── bin/        ← the executable
├── config/     ← its configuration
├── data/       ← its data files
└── lib/        ← its libraries
```

This contrasts with how the package manager installs software:
```
Package manager install (apt/dnf):
  binary   → /usr/bin/nginx
  config   → /etc/nginx/
  logs     → /var/log/nginx/
  data     → /var/lib/nginx/
  Split across multiple FHS locations — the standard way

Manual binary install:
  binary   → /usr/local/bin/kubectl
  Just a single binary file, no config or data

Vendor/self-contained install:
  everything → /opt/vendorname/
  Complete package in one place
  Jenkins, Atlassian tools, some commercial software
```

The problem `/opt` installs cause: deployment scripts written
expecting FHS convention (config in `/etc`) break when a vendor
installer put config in `/opt`. Neither is wrong — but consistency
matters.

### 1.7 ls flags and tree — the tools you use to walk the map

Every task in Part 2 uses `ls` with flags and `tree` to explore the
directory map from section 1.2. Both are introduced here, before
you use them, rather than explained the first time they appear in
a task.

**`ls` — list directory contents.** The name is self-evident; what
matters is its flags:

| Flag | Meaning of the letter | What it does |
|---|---|---|
| `-l` | long | Long format — permissions, owner, size, date |
| `-a` | all | Show hidden files (names starting with `.`) |
| `-h` | human-readable | Sizes as K, M, G instead of raw bytes |

```bash
ls -lh /     # long format + human-readable sizes — daily-use combination
ls -la /     # long format + hidden files — see everything, including dotfiles
```

Flags combine: `-lh` and `-la` are `-l` plus one more flag, not
separate options — `ls -l -h` and `ls -lh` are identical.

**`tree` — display a directory as a visual tree, not a flat
list.** Not installed by default on either distro (install
instructions are in Part 2). The one flag you need here:

| Flag | Meaning of the letter | What it does |
|---|---|---|
| `-L` | Level | Limit recursion depth — `-L 2` shows 2 levels deep, not the entire subtree |

```bash
tree -L 2 /var   # /var and its immediate children's children — not
                  # every file in every log directory, which would
                  # be unreadably long on a real system
```

Without `-L`, `tree` recurses into every subdirectory it finds —
on `/var` specifically that means every file under `/var/log`,
`/var/lib/docker`, etc., which is why this demo always constrains
depth explicitly.

### 1.8 The `file` command — identifying files by content

`file` reads the first bytes of a file (magic bytes) and identifies
what it actually is, regardless of name or extension:

```bash
file /usr/bin/ls
file /usr/bin/python3
file /usr/local/bin/kubectl
```

Output types you will see:

```
ELF 64-bit LSB pie executable    ← compiled binary, position-independent
                                    (pie = can load at any memory address,
                                    security feature, harder to exploit)

ELF 64-bit LSB executable        ← compiled binary, fixed address
statically linked                ← all libraries built in, no dependencies
                                    kubectl is statically linked —
                                    one file, runs anywhere, no library issues

dynamically linked               ← depends on external .so library files
                                    most system binaries are dynamic

symbolic link to python3.12      ← this file is just a pointer
                                    follow it to find the real binary

ASCII text executable            ← a script (shell, Python, etc.)
                                    or Rocky9 coreutils multi-call binary
```

**Ubuntu vs Rocky9 difference in your output:**
```
Ubuntu:  file /usr/bin/ls → ELF 64-bit LSB pie executable
Rocky9:  file /usr/bin/ls → a /usr/bin/coreutils --coreutils-prog-shebang=ls script
```

Rocky Linux 9 uses a **multi-call binary** for coreutils. All
commands (`ls`, `cp`, `mv`, `cat`, `echo`) are compiled into one
single binary called `coreutils`. Each command is a tiny wrapper
script that calls `coreutils` with its name as argument. This saves
disk space. Ubuntu compiles each command as a separate ELF binary.
Functionally identical — just a packaging decision.

**Python versions:**
```
Ubuntu:  python3 → python3.12   ← current
Rocky9:  python3 → python3.9    ← older, RHEL ships conservative versions
```
This is a real production consideration — scripts assuming Python 3.10+
features will fail on RHEL/Rocky9 without version checking.

---

## Part 2 — Hands-On Lab

### Commands used in this demo

| Command | Flag meaning | Purpose |
|---|---|---|
| `ls -lh /` | `l` = long, `h` = human-readable | List root with human-readable sizes |
| `ls -la /` | `l` = long, `a` = all | List including hidden files and symlinks |
| `tree -L 2 /var` | `L` = Level | Visual directory tree, 2 levels deep |
| `file /path/to/file` | — | Identify file type by content |
| `df -h /proc` | `h` = human-readable | Show filesystem size (proves /proc is virtual) |
| `cat /proc/version` | — | Kernel version from virtual filesystem |
| `cat /proc/uptime` | — | System uptime in seconds |
| `cat /proc/$$/status` | — | Status of current shell process |

Install `tree` if missing:
```bash
# Ubuntu
sudo apt install tree -y

# Rocky9
sudo dnf install tree -y
```

---

### Task 1 — Map the root directory

**Run on Ubuntu, then Rocky9:**

```bash
ls -lh /
```

**What to look for:**

```
lrwxrwxrwx  bin -> usr/bin    l = symlink, UsrMerge confirmed
dr-xr-xr-x  proc             dr-x = no write bit, kernel-controlled virtual fs
dr-xr-xr-x  sys              same — virtual filesystem
drwxrwxrwt  tmp              t at end = sticky bit (explained below)
drwx------  root             only root can enter — intentional security
```

**Reading `ls -lh` output — full field decode:**

```
drwxr-xr-x   3   root   root   4.0K   Jun 7 02:48   etc
│             │   │      │      │      │              │
│             │   │      │      │      │              └─ name
│             │   │      │      │      └─ last modified
│             │   │      │      └─ size (4K = one disk block)
│             │   │      └─ group owner
│             │   └─ user owner
│             └─ hard link count
└─ type + permissions
   d=directory  l=symlink  -=regular file
   rwxr-xr-x:
     owner(u): rwx — read, write, execute
     group(g): r-x — read, execute, no write
     other(o): r-x — read, execute, no write
```


**The sticky bit on `/tmp`:**
```
drwxrwxrwt   tmp
         ↑
         t = sticky bit
```
Everyone can write to `/tmp` (world-writable: `rwxrwxrwx`).
The sticky bit means you can only delete your own files — not
anyone else's. Without it, any user could delete any other user's
temp files on a shared system.

**Ubuntu-specific entries not on Rocky9:**
```
bin.usr-is-merged/    Ubuntu's explicit UsrMerge marker — empty dir
lib.usr-is-merged/    Rocky9 does this silently with no marker
sbin.usr-is-merged/
snap/                 Ubuntu snap package runtime — not on RHEL systems
init (2.8M file)      Microsoft WSL2 PID 1 binary — not on bare metal
```

**Rocky9-specific entries not on Ubuntu:**
```
afs/                  Andrew Filesystem mount point — RHEL legacy
dr-xr-x---  root      different from Ubuntu's drwx------ on /root —
                      Rocky9 drops owner write and adds group r-x;
                      not simply "stricter," a different tradeoff
                      (verify against your own system before relying
                      on this — permissions here can vary by install)
```

---

### Task 2 — Prove /proc is virtual, then contrast with /sys, /dev, and tmpfs

**Run on Ubuntu, then Rocky9:**

```bash
df -h /proc
cat /proc/version
cat /proc/uptime
```

**Reading `df -h /proc` output:**

```
Filesystem   Size   Used   Avail   Use%   Mounted on
proc            0      0       0      -   /proc
                ↑             ↑      ↑
                0 = nothing   0      - means Use% cannot be calculated
                on disk             for a filesystem with no size
                or in RAM           Any monitoring script parsing df
                                    output must handle this dash
```

**Reading `/proc/uptime`:**

```
597.31   9520.95
  ↑         ↑
seconds   total idle CPU time across all cores
since     9520 / 597 = ~15.9 cores worth of idle time
boot      Divide by your CPU count to get idle percentage

597 seconds = 597 / 60 = ~10 minutes uptime (Ubuntu just restarted)
```

**Reading `/proc/version`:**

```
Linux version 6.6.114.1-microsoft-standard-WSL2
              ↑         ↑
              kernel    build tag — "microsoft-standard-WSL2" means
              version   this is Microsoft's custom kernel, not upstream.
                        Some modules available on bare metal servers
                        are missing here.

(root@507f3e43091d)    the build system hostname — Microsoft's CI
(gcc 13.2.0)           compiler used to build this kernel
#1 SMP                 SMP = Symmetric Multi-Processing (multi-core)
PREEMPT_DYNAMIC        can switch preemption modes at runtime
Mon Dec 1 20:46:23 UTC 2025   when this kernel was compiled
```

Both Ubuntu and Rocky9 show the same `/proc/version` — they share
the same WSL2 kernel. This is unique to WSL2. On real servers each
distro runs its own kernel.

**Now confirm `/sys` and `/dev` share the same virtual characteristics:**

```bash
ls /sys/class/net/              # network interfaces exposed as virtual files
cat /sys/class/net/*/mtu 2>/dev/null | head -3
ls -l /dev/null /dev/random /dev/tty
df -h /sys /dev
# /sys shows the same Size 0 / Use% - pattern as /proc — same kernel
# mechanism, different subject matter (hardware/drivers instead of
# processes). /dev is closer to zero but not necessarily identical —
# devtmpfs holds tiny metadata entries for detected hardware, still
# fundamentally different from real disk usage.
```

**Now contrast with tmpfs — `/run` and `/tmp` are NOT virtual in the
same sense, even though both are "not really on disk":**

```bash
df -h /run /tmp
# Unlike /proc/sys/dev, these show a REAL allocated size (from RAM),
# not Size 0 — this is the concrete proof that tmpfs and virtual
# filesystems are different mechanisms, not just two names for the
# same thing.

mount | grep -E " /run | /tmp "
# Confirms the filesystem type is tmpfs, not proc/sysfs/devtmpfs

# Prove /run and /tmp are writable exactly like a normal filesystem
touch /tmp/demo01-tmpfs-test
echo "hello" > /tmp/demo01-tmpfs-test
cat /tmp/demo01-tmpfs-test
ls -l /tmp/demo01-tmpfs-test
rm -f /tmp/demo01-tmpfs-test
# You just created, wrote to, read, and deleted a file normally —
# try the same sequence against /proc/meminfo and it will fail,
# because /proc's contents are kernel-computed, not something you
# can create arbitrary new files inside.
```

**Paste the `df -h` output for `/proc`, `/sys`, `/dev`, `/run`, and
`/tmp` side by side — this is the clearest possible proof of the
distinction section 1.3 describes.**

---

### Task 3 — Look inside your own process

Every running process has a directory in `/proc` named by its PID.
Your shell is a process. This task makes `/proc` concrete.

**Run on Ubuntu, then Rocky9:**

```bash
echo "My shell PID: $$"
cat /proc/$$/status | grep -E "^(Name|State|Pid|PPid|Uid)"
```

**Reading the status output:**

```
Name:   zsh              ← process name (Ubuntu uses zsh, Rocky9 uses bash)
State:  S (sleeping)     ← S = sleeping, waiting for input. Normal.
                            Other states: R=running, D=uninterruptible sleep,
                            Z=zombie, T=stopped
Pid:    632              ← this process's ID
PPid:   631              ← parent PID — process that started this shell
                            (your terminal emulator or sshd)
Uid:    1000 1000 1000 1000
        ↑    ↑    ↑    ↑
        real eff  saved filesystem
```

**The four UID fields — why they matter:**

```
real      Who you actually are (1000 = wadmin)
effective Who you are for permission checks right now
saved     Stored for privilege restoration
filesystem Who you are for filesystem operations

All four equal = normal user, no privilege escalation active

When you run sudo bash:
  real=0, effective=0, saved=0, filesystem=0
  sudo replaces the process's identity outright — all four fields
  become root's, not just effective. (sudo does log which original
  user invoked it, but that's a separate audit trail, not one of
  these four kernel-tracked UID fields.)

Contrast with a setuid binary like passwd, mid-execution — this is
the actual case where the four fields differ from each other:
  real=1000, effective=0, saved=0, filesystem=0
  real stays as the calling user (1000); effective becomes the
  binary's owner (root) for the duration of the setuid call. This
  is the mechanism that lets passwd write to /etc/shadow on your
  behalf without making you root generally.
```

**Ubuntu vs Rocky9 difference:**
```
Ubuntu:  Name: zsh   (you installed zsh with powerlevel10k)
Rocky9:  Name: bash  (default shell, zsh not installed)
```

---

### Task 4 — Explore /var structure

**Run on Ubuntu, then Rocky9:**

```bash
tree -L 2 /var
```

**Key differences you will see:**

```
Ubuntu /var/log:                    Rocky9 /var/log:
  syslog         ← text log file      dnf.log      ← package manager log
  auth.log       ← auth events        dnf.rpm.log  ← RPM transactions
  kern.log       ← kernel messages    hawkey.log   ← dependency resolver
  dmesg          ← boot messages      wtmp         ← login history
  dpkg.log       ← package events     lastlog      ← last logins
  apt/           ← apt transactions   sa/          ← sysstat data (sar)
                                      journal/     ← binary journal (main)

Why Ubuntu has syslog and Rocky9 does not:
Ubuntu  → rsyslog daemon writes traditional text log files
Rocky9  → journald only, binary format, query with journalctl
```

```
Ubuntu /var/lib:                    Rocky9 /var/lib:
  docker/        ← Docker data        rpm/         ← RPM package database
  containerd/    ← container runtime  dnf/         ← dnf metadata
  apt/           ← apt database       selinux/     ← SELinux policy data
  snapd/         ← snap packages      systemd/     ← systemd state
  dpkg/          ← dpkg database      tpm2-tss/    ← TPM library state
```

```
Ubuntu /usr/local/bin has tools:    Rocky9 /usr/local/bin is empty:
  kubectl, helm, terraform            ← tools installed in Ubuntu WSL2
  argocd, aws, eksctl                 are NOT available in Rocky9
  minikube, skaffold                  Each WSL2 instance = separate filesystem
```

---

### Task 5 — Identify file types

**Run on Ubuntu, then Rocky9:**

```bash
file /usr/bin/ls
file /usr/bin/python3
file /usr/local/bin/kubectl 2>/dev/null || echo "kubectl not installed"
```

**Expected outputs and what they mean:**

```
Ubuntu:
/usr/bin/ls → ELF 64-bit LSB pie executable, x86-64, dynamically linked
              ELF = Linux binary format
              64-bit = 64-bit architecture
              LSB = little-endian byte order
              pie = position-independent, security hardening
              dynamically linked = needs shared libraries at runtime

/usr/bin/python3 → symbolic link to python3.12
                   file tells you it is a symlink — follow it:
                   file /usr/bin/python3.12 → ELF binary

/usr/local/bin/kubectl → ELF 64-bit LSB executable, statically linked
                         statically linked = all libraries bundled inside
                         kubectl is one file, no dependencies
                         runs on any Linux system regardless of libraries

Rocky9:
/usr/bin/ls → a /usr/bin/coreutils --coreutils-prog-shebang=ls script
              Multi-call binary — all coreutils in one executable
              ls, cp, mv, cat are all scripts calling coreutils
              Saves disk space, functionally identical to separate binaries

/usr/bin/python3 → symbolic link to python3.9
                   RHEL ships Python 3.9, Ubuntu ships 3.12
                   Write portable scripts — avoid 3.10+ only features

/usr/local/bin/kubectl → not installed
                          Tools installed in Ubuntu WSL2 do not exist
                          in Rocky9 — separate filesystems
```

---

## What You Learned

- ✅ Named the purpose of every major directory under `/`
- ✅ Explained the difference between virtual filesystems (`/proc`, `/sys`, `/dev`) and tmpfs filesystems (`/run`, `/tmp`) — and proved it directly by comparing `df -h` output across all five
- ✅ Identified immediately where configs, logs, binaries, runtime state, and temporary files live
- ✅ Explained the three binary installation patterns: package manager, manual install, vendor self-contained
- ✅ Navigated and explored the filesystem using `ls`, `cd`, `tree`, and `file`
- ✅ Read and interpreted output of `ls -lh`, `df -h /proc`, `/proc/version`, `/proc/uptime`, `/proc/$$/status`

**Key Takeaway:** The FHS is not arbitrary — every directory's location encodes information about how data behaves (static vs runtime, persistent vs ephemeral). Knowing this map turns "where is the problem" from a search into a lookup.

---

## Part 3 — Break-Fix Scenario

### Scenario — NovaTech deployment failure

You join NovaTech's on-call rotation. First alert:

> *"The deployment pipeline is failing. The app cannot find its
> configuration file. Everything was working yesterday. Nobody
> changed the script."*

You SSH in and investigate:

```bash
# Script expects config here
ls /etc/novatech/app.conf
# ls: cannot access '/etc/novatech/app.conf': No such file or directory

# Search for the file
find / -name "app.conf" 2>/dev/null
# /opt/novatech/config/app.conf
```

**Your task — answer from FHS knowledge covered in this demo:**

1. Which location is FHS-correct for application configuration:
   `/etc/novatech/app.conf` or `/opt/novatech/config/app.conf`?

2. Based on what you learned about `/opt` in Part 1, how did the
   file end up there — what type of installer puts things in `/opt`?

3. What is the immediate fix to get the pipeline working?
   What is the proper long-term fix?

Write your answers before revealing the resolution.

<details>
<summary>Resolution — open after attempting</summary>

**1. FHS-correct location: `/etc/novatech/app.conf`**

`/etc` is for host-specific system configuration. Application configs
that control how a service behaves on this particular server belong
here. Every sysadmin, automation script, and monitoring tool looks
in `/etc` first.

**2. How it ended up in `/opt`:**

As explained in Part 1 (section 1.5), `/opt` is for self-contained,
third-party vendor packages that install everything — binary, config,
data — in one directory tree. The file landed in `/opt/novatech/config/`
because a vendor installer or Docker-style install script packaged
the entire application as a self-contained unit under `/opt`.

This is not technically wrong for `/opt`-style installations. The
problem is that the deployment script was written expecting FHS
convention — config in `/etc`. The mismatch between installer
convention and script expectation caused the failure.

**3. Fixes:**

Immediate — get the pipeline working now:
```bash
sudo mkdir -p /etc/novatech
sudo cp /opt/novatech/config/app.conf /etc/novatech/app.conf
```

Proper long-term fix — pick one approach and document it:

Option A: Move to FHS standard
```bash
sudo mv /opt/novatech/config/app.conf /etc/novatech/app.conf
# Update installer to put config in /etc going forward
```

Option B: Update the deployment script to use /opt path
```bash
# Change the script to look in /opt/novatech/config/app.conf
# Keep the vendor's self-contained layout intact
```

The choice depends on whether this is a vendor-managed application
(keep it in /opt, update the script) or an in-house application
(move to /etc, the FHS standard). Document the decision so the
next engineer does not face the same mystery.

**The FHS lesson:**
Knowing FHS let you diagnose this in 30 seconds — you knew
immediately that `/etc` was where config should be, you used `find`
to locate the actual file, and you understood why `/opt` told you
it was a vendor-style install. Without FHS knowledge, you would
be reading the script source code trying to understand what went wrong.

</details>

---

## Interview Prep

**Q1. Where do you look first when a service fails to start?**

`/var/log` — specifically `journalctl -u servicename` on modern
systems, or the service's text log file under `/var/log/` on older
systems. The log shows the exact error. Look here before touching
any config. Then check `/etc/servicename/` to review the config
if the log points to a configuration issue. We cover `journalctl`
in full in Demo 07.

---

**Q2. What is the difference between `/tmp` and `/var/tmp`?**

Both hold temporary files but differ in persistence. `/tmp` is
cleared on every reboot. `/var/tmp` is for temporary data that
must survive a reboot — a long upload in progress, a build artifact
being assembled across sessions. Use `/tmp` for short-lived scratch
space in scripts. Use `/var/tmp` when the data needs to outlast a
reboot.

---

**Q3. You installed `terraform` manually. Two weeks later: command
not found. What do you check, in order?**

```bash
which terraform                # is it findable at all?
ls -la /usr/local/bin/terraform  # is the binary there?
echo $PATH                     # is /usr/local/bin in PATH?
```

Most common cause: binary is in `/usr/local/bin/` but PATH does not
include it. Or it was placed in `/usr/bin/` and overwritten by a
package manager upgrade. Rule: manual installs always go in
`/usr/local/bin/`, which the package manager never touches.

---

**Q4. What is the difference between `/proc` and `/run`?
Why does `/proc` show size 0 in `df` output?**

`/proc` is a virtual filesystem — the kernel generates its contents
dynamically on demand. Nothing is stored on disk or in RAM. Size is
0 because there is nothing to measure. `df` shows `-` for Use%
because percentage is undefined for a zero-size filesystem.

`/run` is tmpfs — RAM-backed storage where services write PID files,
Unix sockets, and lock files. It has actual size (allocated from RAM),
behaves like a normal filesystem, and is cleared on every reboot.

Key operational difference: if a file in `/proc` is missing, the
kernel process or feature does not exist. If a file in `/run` is
missing after reboot, the service has not started yet.

---

## Key Takeaways

1. **Know the five directories you use every day without thinking: `/etc` configs, `/var/log` logs, `/var/lib` app state, `/run` runtime only, `/usr/local/bin` your manual tools.** Getting to the right place in 5 seconds separates engineers who debug quickly from those who `find` their way around.
2. **`/proc`, `/sys`, and `/dev` are virtual — the kernel generates their contents on demand and nothing is stored on disk or in RAM.** Size is zero, `df` shows `-` for Use%, and you cannot create arbitrary files there. Every monitoring tool (`top`, `free`, `ps`, `ss`) reads from `/proc`.
3. **`/run` and `/tmp` are tmpfs — RAM-backed storage that you write to normally, cleared completely on every reboot.** A missing socket or PID file after reboot means the service has not started yet, not a filesystem problem.
4. **`/bin`, `/sbin`, and `/lib` are symlinks to `/usr/` equivalents on all modern distros (UsrMerge).** Scripts using `/bin/bash` still work via the symlink. Ubuntu marks this explicitly with `bin.usr-is-merged/`; Rocky9 does it silently.
5. **Ubuntu logs to text files in `/var/log/syslog` via rsyslog; Rocky Linux logs to the binary journal — query with `journalctl`, not `cat`.** Writing a monitoring script that greps syslog will silently miss all logs on RHEL-family systems.
6. **The `file` command identifies what a file actually is from its content (magic bytes), not its name or extension.** `statically linked` means no external library dependencies — runs on any Linux. `dynamically linked` means it needs specific shared libraries present.
7. **`/usr/local/bin` is per-system and per-WSL2-instance.** Tools installed in Ubuntu WSL2 do not exist in Rocky9 — separate filesystems, no sharing.

---

## Quick Commands Reference

### Shell operators used in this demo

| Operator | Meaning | Example |
|---|---|---|
| `&&` | Run right side only if left succeeded (exit 0) | `mkdir /etc/app && cp conf /etc/app/` |
| `\|\|` | Run right side only if left failed (non-zero exit) | `ls /etc/app \|\| echo "not found"` |
| `2>/dev/null` | Discard error messages (stderr to /dev/null) | `find / -name file 2>/dev/null` |
| `$$` | PID of the current shell process | `cat /proc/$$/status` |

### grep patterns used in this demo

| Pattern | Matches | Use |
|---|---|---|
| `^Name` | Lines starting with "Name" | Extract field from /proc/status |
| `-E "^(Name\|Pid)"` | Lines starting with Name OR Pid | Multiple field extract |

### /proc/$$  — key files in any process directory

| File | What it contains |
|---|---|
| `cmdline` | The full command that started this process |
| `status` | Process state, PID, UID, memory usage |
| `exe` | Symlink to the binary being executed |
| `cwd` | Symlink to current working directory |
| `fd/` | Directory of open file descriptors |
| `environ` | Environment variables of the process |

---

## Appendix A — Anki Cards

**`01-linux-fhs-anki.csv`:**

```
#deck:Linux Mastery::Module 1 - Filesystem & Navigation::Demo 01 - FHS
#separator:Comma
#columns:Front,Back,Tags

"A service fails after a config change. Before touching anything, what is the first place you check and why?","Check /var/log first — journalctl -u servicename or the service text log file. The log shows the exact error. Look here before changing any config — the actual problem may be different from what you assume.","demo01,fhs,troubleshooting"

"You run df -h /proc and see Size: 0, Use%: -. What does this tell you and why can you not calculate a percentage?","/proc is a virtual filesystem. The kernel generates its contents dynamically on demand — nothing is stored on disk or in RAM. Size is literally 0. Use% shows - because percentage is mathematically undefined for a zero-size filesystem. This is correct, not an error.","demo01,fhs,proc,virtual"

"What is the difference between /proc (virtual) and /run (tmpfs)? Give one operational example of each.","Virtual (/proc, /sys, /dev): kernel generates content on demand, nothing stored, size 0, you cannot create arbitrary files. Operational: cat /proc/meminfo — kernel computes memory stats at that moment. tmpfs (/run, /tmp): RAM-backed storage, you write normally, cleared on reboot. Operational: /run/docker.sock — Docker daemon creates this socket on startup, gone after reboot.","demo01,fhs,virtual,tmpfs"

"Install kubectl manually. Where does it go and why? What happens if you put it in /usr/bin instead?","Goes in /usr/local/bin/ — the package manager (apt/dnf) never touches this directory. If you put it in /usr/bin/, a future apt upgrade or dnf update can overwrite it with a packaged version, breaking your pinned version silently.","demo01,fhs,usrlocalbin,tools"

"Rocky9 has no /var/log/syslog. Where are the logs and how do you read them?","Rocky Linux 9 (RHEL-family) uses journald as the only logging system. Logs go to the binary journal at /var/log/journal/. Read them with: journalctl (all logs), journalctl -u nginx (service logs), journalctl -f (follow live). Ubuntu runs both journald AND rsyslog, which writes text files.","demo01,fhs,logging,rocky,ubuntu"

"file /usr/bin/ls shows ELF on Ubuntu but shows a coreutils script on Rocky9. Why? Are they functionally different?","Rocky Linux uses a multi-call binary — all coreutils commands (ls, cp, mv, cat) are compiled into one binary called coreutils. Each command is a wrapper script that calls coreutils with its name. Ubuntu compiles each command as a separate ELF binary. Functionally identical — just a packaging decision for space efficiency.","demo01,fhs,file,coreutils,rocky"

"A deploy script fails because it cannot find config at /etc/app/app.conf. find locates it at /opt/app/config/app.conf. What does the /opt location tell you about how this was installed?","/opt is for self-contained third-party vendor packages that install everything (binary, config, data, libraries) in one directory tree. The file is in /opt because a vendor installer packaged it that way — not split across /etc, /usr, /var as FHS standard expects. The script was written expecting FHS convention. One must be updated to match the other.","demo01,fhs,opt,break-fix"

"What does statically linked mean in file command output? Why does kubectl being statically linked matter for DevOps?","Statically linked means all required libraries are compiled into the binary itself. No external dependencies. kubectl runs on any Linux system regardless of which shared libraries are installed. This is why you can drop the kubectl binary into a minimal container or a fresh server and it just works — no library version conflicts.","demo01,fhs,file,static,kubectl"

"Why does root's home directory live at /root instead of /home/root?","/home is often a separate partition, sometimes NFS-mounted, sometimes mounted late in boot. If root's home were under /home and that mount failed, you would be locked out of your own recovery console. /root sits directly on the root filesystem — always accessible, even in single-user recovery mode.","demo01,fhs,root,recovery"

"ls -lh / shows bin -> usr/bin as a symlink. What is this called, and does it break scripts that use #!/bin/bash?","This is UsrMerge — on all modern distros (Ubuntu 20.04+, Rocky Linux 9, RHEL 9+), /bin, /sbin, and /lib are symlinks to their /usr/ equivalents. Scripts using /bin/bash still work fine because the symlink resolves correctly to the real binary under /usr/bin. Ubuntu marks this explicitly with a bin.usr-is-merged/ marker directory; Rocky9 does it silently with no marker.","demo01,fhs,usrmerge,symlink"
```

---

## Appendix B — Quiz

**`01-linux-fhs-quiz.md`:**

```
# Quiz — Demo 01: Linux Filesystem Hierarchy

> One correct answer per question. Target: 80% before Demo 02.

---

**Q1.** Why does root's home directory live at `/root` instead of
`/home/root`, given that regular users' homes are under `/home`?

- A) `/root` is faster to type, which is the only reason for the convention
- B) `/home` is often a separate partition, sometimes NFS-mounted or
  mounted late in boot — if root's home lived there and that mount
  failed, root would be locked out of the recovery console; `/root`
  sits directly on the root filesystem and is always accessible
- C) `/root` is a historical leftover from early Unix with no
  functional reason today
- D) `/home/root` is reserved for a different purpose and cannot be used

<details>
<summary>Answer</summary>

**B** — This is a deliberate reliability decision, not a naming
convention or historical accident. If `/home` fails to mount (a
separate partition, an NFS share that's unreachable), root still
needs a working home directory to log in and fix the problem.

Trap: C dismisses a design choice that has a clear, current
functional reason — losing recovery access is a real, serious
consequence, not a cosmetic one.

</details>

---

**Q2.** After a server reboot your application reports that its Unix
socket at `/run/myapp/myapp.sock` is missing. What is the cause?

- A) The socket file was corrupted — restore from backup
- B) `/run` is tmpfs — completely empty after every reboot, the
  service must recreate the socket on startup
- C) Socket files cannot be stored in `/run`
- D) `/run` is mounted read-only during boot

<details>
<summary>Answer</summary>

**B** — `/run` is tmpfs, RAM-backed, wiped on every reboot. Services
create their PID files and sockets on startup. Missing socket means
the service has not started or failed to start.
Check: `systemctl status myapp`.
Trap: D is wrong — `/run` is read-write.

</details>

---

**Q3.** You need to write a bash script that installs a package
differently based on whether it is running on Ubuntu or Rocky Linux.
Which file and which field do you read?

- A) `/proc/version` — check for "Ubuntu" or "Red Hat" in kernel string
- B) `/etc/os-release` — use the `ID_LIKE` field
- C) `/etc/issue` — read the first line
- D) `uname -a` — check the build string

<details>
<summary>Answer</summary>

**B** — `/etc/os-release` is the machine-readable OS identity file.
`ID_LIKE=debian` → apt/dpkg. `ID_LIKE=fedora` → dnf/rpm.
Usage: `source /etc/os-release` then test `$ID_LIKE`.
Trap: A is unreliable — Ubuntu and Rocky9 share the same WSL2
kernel so `/proc/version` does not contain distro names.

</details>

---

**Q4.** A teammate put application logs in `/tmp/applogs/`.
What is the problem?

- A) `/tmp` has wrong permissions for log files
- B) `/tmp` is cleared on reboot — logs are gone after any restart,
  making post-incident analysis impossible
- C) Log files must be owned by root
- D) `/tmp` does not support directories

<details>
<summary>Answer</summary>

**B** — `/tmp` is tmpfs, cleared on every reboot. If a server
restarts after an incident, all evidence is gone. Logs belong in
`/var/log/appname/` — persistent, covered by logrotate, accessible
to monitoring agents.
Trap: A is wrong — permissions in `/tmp` are fine technically.

</details>

---

**Q5.** `cat /proc/$$/status` shows `Uid: 1000 1000 1000 1000` — four
identical numbers. What do these four fields represent, and under
what circumstance would they differ from each other?

- A) They represent four separate users who can access this
  process; they differ when a process is shared between users
- B) real, effective, saved, and filesystem UID — normally identical
  for an ordinary process; they differ mid-execution of a setuid
  binary like `passwd`, where real stays as the calling user but
  effective becomes the binary owner (root) for the duration of the call
- C) They are redundant copies for error-correction and never
  differ under any circumstance
- D) They represent the UID at four different points in the
  process's history (creation, last exec, last fork, current)

<details>
<summary>Answer</summary>

**B** — This is the exact mechanism that lets `passwd` write to
`/etc/shadow` on your behalf without making you root generally:
effective UID changes to the binary owner's for the duration of the
setuid call, while real UID still correctly reflects who actually
invoked it.

Trap: D invents a time-series meaning that isn't what these fields
track — they're about privilege context, not history.

</details>

---

**Q6.** `ls -lh /` shows `bin -> usr/bin` as a symlink target. What
is this called, and does it break scripts that use `#!/bin/bash`?

- A) This is called UsrMerge; scripts using `/bin/bash` still work
  correctly because the symlink resolves to the real binary under
  `/usr/bin` — no breakage
- B) This is a Docker-specific artifact and does not appear on bare-metal servers
- C) Scripts must be rewritten to use `/usr/bin/bash` directly, or
  they will fail on any modern distro
- D) This only exists on Ubuntu; Rocky9 keeps `/bin` as a real directory

<details>
<summary>Answer</summary>

**A** — UsrMerge is standard on all modern distros (Ubuntu 20.04+,
Rocky Linux 9, RHEL 9+). The symlink resolves transparently, so
existing scripts using `/bin/bash` need no changes.

Trap: D is wrong — Rocky9 does UsrMerge too, just silently, without
Ubuntu's explicit `bin.usr-is-merged/` marker directory.

</details>

---

**Q7.** You are deploying nginx on a fresh server. Which of the
following correctly maps nginx's runtime PID file, access log,
package-manager binary, and configuration file to their FHS locations?

- A) PID: `/tmp/nginx.pid` — Log: `/opt/nginx/logs/access.log` — Binary: `/usr/local/bin/nginx` — Config: `/var/nginx/`
- B) PID: `/run/nginx.pid` — Log: `/var/log/nginx/access.log` — Binary: `/usr/bin/nginx` — Config: `/etc/nginx/`
- C) PID: `/var/run/nginx.pid` — Log: `/var/log/nginx/access.log` — Binary: `/usr/bin/nginx` — Config: `/etc/nginx/nginx.conf`
- D) PID: `/run/nginx.pid` — Log: `/var/log/nginx/access.log` — Binary: `/usr/local/bin/nginx` — Config: `/etc/nginx/`

<details>
<summary>Answer</summary>

**B** — PID files belong in `/run/` (tmpfs, runtime state). Logs belong in `/var/log/servicename/`. A binary installed by the package manager (`apt install nginx`) goes in `/usr/bin/`. Configuration for any service goes in `/etc/servicename/`.

Trap: C has correct paths for PID, log, and config — but `/var/run` is a legacy symlink to `/run/` on modern systems, not the canonical path. D is wrong because `/usr/local/bin` is for manually installed tools, not package manager installs. A fails on every location.

</details>

---

Score guide:

| Score | Action |
|---|---|
| 7/7 | Import Anki cards, move to Demo 02 |
| 6/7 | Review the wrong answer, then proceed |
| 5/7 | Re-read the relevant section, retry those questions |
| Below 5/7 | Re-read the full demo and redo the walkthrough before proceeding |
```