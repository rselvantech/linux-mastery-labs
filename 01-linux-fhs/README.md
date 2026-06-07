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
├── README.md                    # this file
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
├── proc/            VIRTUAL — live kernel and process data
├── sys/             VIRTUAL — live kernel hardware data
├── dev/             VIRTUAL — device files (disks, terminals)
├── boot/            Kernel and bootloader files
└── mnt/             Temporary manual mount points
```

### 1.3 Virtual filesystems vs tmpfs — critical distinction

This is one of the most misunderstood topics in Linux. These are
NOT the same thing.

**Virtual filesystems — kernel generates content on demand:**

```
/proc    Type: proc (kernelfs)
/sys     Type: sysfs
/dev     Type: devtmpfs
```

The kernel generates their contents dynamically — there is nothing
stored anywhere. When you `cat /proc/meminfo`, the kernel computes
current memory statistics at that exact moment and returns them as
if they were a file. The data does not exist until you ask for it.

Characteristics:
- Size = 0 — nothing is stored on disk or in RAM
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
delete files, set permissions. The only difference from disk storage:
everything lives in RAM and is completely gone on reboot.

Characteristics:
- Has actual size (allocated from RAM)
- You write to it normally — services create PID files, sockets here
- Cleared completely on every reboot
- Fast — RAM speed, no disk I/O

**Summary table:**

```
Filesystem  Type      Stored    Writable  Cleared    Example files
/proc       kernelfs  nowhere   kernel    N/A        /proc/meminfo
/sys        sysfs     nowhere   kernel    N/A        /sys/class/net/
/dev        devtmpfs  nowhere   kernel    N/A        /dev/sda
/run        tmpfs     RAM       yes       on reboot  sshd.pid, docker.sock
/tmp        tmpfs     RAM       yes       on reboot  build artifacts
```

### 1.4 Three things to know about /proc, /sys, /dev

These directories are **virtual** — generated by the kernel in RAM at
boot. They have zero disk size. They are not on your hard drive.

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

Important: Ubuntu and Rocky Linux log differently:
```
Ubuntu:   rsyslog writes text files → /var/log/syslog, auth.log, kern.log
Rocky9:   journald writes binary journal → /var/log/journal/
          Query with: journalctl (not cat)
```
This is why there is no `/var/log/syslog` on Rocky9. The logs exist
— they are in the journal. We cover `journalctl` in depth in Demo 07.

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

### 1.7 The `file` command — identifying files by content

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

| Command | Purpose |
|---|---|
| `ls -lh /` | List root with human-readable sizes |
| `ls -la /` | List including hidden files and symlinks |
| `tree -L 2 /var` | Visual directory tree, 2 levels deep |
| `file /path/to/file` | Identify file type by content |
| `df -h /proc` | Show filesystem size (proves /proc is virtual) |
| `cat /proc/version` | Kernel version from virtual filesystem |
| `cat /proc/uptime` | System uptime in seconds |
| `cat /proc/$$/status` | Status of current shell process |

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
     owner: rwx (read, write, execute)
     group: r-x (read, execute — no write)
     others: r-x (read, execute — no write)
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
dr-xr-x---  root      stricter permissions than Ubuntu's drwx------
```

---

### Task 2 — Prove /proc is virtual

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

597 seconds = 597 / 3600 = ~10 minutes uptime (Ubuntu just restarted)
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
  real=1000, effective=0, saved=1000, filesystem=0
  Kernel grants root-level permissions while tracking original user
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

- **`/etc`** configs. **`/var/log`** logs. **`/var/lib`** app state.
  **`/run`** runtime only. **`/usr/local/bin`** your manual tools.
  Know these five without thinking.
- `/proc`, `/sys`, `/dev` are virtual — kernel generates content
  on demand, nothing stored, size is zero. Every monitoring tool
  reads from `/proc`.
- `/run` and `/tmp` are tmpfs — RAM storage, writable, cleared on
  reboot. Different from virtual: you write files here normally.
- `/bin`, `/sbin`, `/lib` are symlinks to `/usr/` on all modern
  distros. This is UsrMerge. Scripts using `/bin/bash` still work.
- Ubuntu logs to text files in `/var/log/syslog`. Rocky Linux logs
  to the binary journal — query with `journalctl`, not `cat`.
- `file` command identifies what a file actually is from its content,
  not its name. `statically linked` means no dependencies — runs
  anywhere. `dynamically linked` means it needs shared libraries.
- `/usr/local/bin` is per-system. Tools installed in Ubuntu WSL2
  do not exist in Rocky9 — they are separate filesystems.

---

## Appendix A — Quick Reference

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

## Appendix B — Anki Cards

**`01-linux-fhs-anki.csv`:**

```
#deck:Linux Mastery::Module 1 - Filesystem::Demo 01 - FHS
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
```

---

## Appendix C — Quiz

**`01-linux-fhs-quiz.md`:**

```
# Quiz — Demo 01: Linux Filesystem Hierarchy

> One correct answer per question. Target: 80% before Demo 02.

---

Q1. You run df -h /proc and see Use%: - (a dash, not a number).
What is the correct interpretation?

A) /proc filesystem is full — the dash means overflow
B) /proc is a virtual filesystem with no physical storage — percentage
   is undefined for a zero-size filesystem — this is correct behaviour
C) df cannot access /proc due to permissions
D) /proc is unmounted and needs to be remounted

[Answer: B — /proc is virtual, kernel-generated, nothing stored.
Size 0, Use% undefined. df shows - for all virtual filesystems.
This is not an error. Trap: A is wrong — if /proc were full, df
would show 100%, not a dash.]

---

Q2. After a server reboot your application reports that its Unix
socket at /run/myapp/myapp.sock is missing. What is the cause?

A) The socket file was corrupted — restore from backup
B) /run is tmpfs — completely empty after every reboot, the service
   must recreate the socket on startup
C) Socket files cannot be stored in /run
D) /run is mounted read-only during boot

[Answer: B — /run is tmpfs, RAM-backed, wiped on every reboot.
Services create their PID files and sockets on startup. If the
socket is missing, the service has not started or failed to start.
Check: systemctl status myapp. Trap: D is wrong, /run is read-write.]

---

Q3. You need to write a bash script that installs a package
differently based on whether it is running on Ubuntu or Rocky Linux.
Which file and which field do you read?

A) /proc/version — check for "Ubuntu" or "Red Hat" in kernel string
B) /etc/os-release — use the ID_LIKE field
C) /etc/issue — read the first line
D) uname -a — check the build string

[Answer: B — /etc/os-release is the machine-readable OS identity file.
ID_LIKE=debian means apt/dpkg. ID_LIKE=fedora or rhel means dnf/rpm.
Use in scripts: source /etc/os-release and test $ID_LIKE.
Trap: A is unreliable — the WSL2 kernel string does not contain
distro names since both Ubuntu and Rocky9 share the same WSL2 kernel.]

---

Q4. A teammate put application logs in /tmp/applogs/.
What is the problem with this?

A) /tmp has wrong permissions for log files
B) /tmp is cleared on reboot — logs are gone after any restart,
   making post-incident analysis impossible
C) Log files must be owned by root
D) /tmp does not support directories

[Answer: B — /tmp is tmpfs, cleared on every reboot. If a server
restarts after an incident, all evidence is gone. Logs belong in
/var/log/appname/ — persistent, covered by logrotate, accessible
to monitoring agents. Trap: A is wrong, permissions in /tmp are fine
for log files from a technical standpoint.]

---

Q5. file /usr/bin/kubectl shows "statically linked".
file /usr/bin/nginx shows "dynamically linked". 
You want to copy just the binary to run in a FROM scratch Docker
container with no OS. Which binary works and why?

A) nginx — it is the standard installation format
B) kubectl — statically linked means all libraries are compiled in,
   no external dependencies needed, runs in any environment
C) Neither — both need a full Linux environment
D) Both — Docker provides all necessary libraries

[Answer: B — statically linked binaries include all their dependencies
compiled into the single file. They run on any Linux kernel regardless
of what libraries are installed. This is why kubectl can run in a
FROM scratch container. nginx is dynamically linked — it needs
specific shared library files (libssl, libc, etc.) to be present
in the container.]

---

Q6. Rocky Linux 9 has no /var/log/syslog. A colleague says
"logging is broken on this server." How do you respond?

A) Agree — syslog should always exist on a production server
B) Rocky Linux 9 uses journald as its primary logging system.
   Logs are in the binary journal. Query with journalctl, not cat.
   syslog is a Debian/Ubuntu convention.
C) Install rsyslog to fix the missing syslog file
D) /var/log/syslog exists but is hidden — use ls -la

[Answer: B — RHEL-family distros (Rocky, RHEL, Amazon Linux, Fedora)
use journald as the primary and often only logging system. The journal
is at /var/log/journal/ in binary format. Use journalctl to query it.
Ubuntu runs both journald and rsyslog. Neither is wrong — different
logging architectures. Trap: C would work but is unnecessary —
journald is fully functional.]

---

Q7. Based on FHS, where does each of these belong?
Match each to the correct directory.

nginx binary installed by apt       →  ?
kubectl downloaded manually         →  ?
nginx configuration file            →  ?
nginx runtime PID file              →  ?
nginx access log                    →  ?
Jenkins installed as vendor package →  ?
nginx cached data                   →  ?

[Answer:
nginx binary (apt)         → /usr/bin/nginx
kubectl (manual)           → /usr/local/bin/kubectl
nginx config               → /etc/nginx/
nginx PID file             → /run/nginx.pid (or /run/nginx/)
nginx access log           → /var/log/nginx/access.log
Jenkins (vendor package)   → /opt/jenkins/
nginx cached data          → /var/cache/nginx/

This is the FHS in practice. Knowing these locations is how you
navigate any server without documentation.]

---

Score guide:
Score   Action
7/7     Import Anki cards, move to Demo 02
6/7     Review wrong answer, proceed
5/7     Re-read relevant Part 1 section, retry those questions
4/7 -   Re-read full demo before proceeding
```