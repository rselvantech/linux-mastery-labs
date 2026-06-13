# Demo 03 — File Permissions, Ownership & Access Control

## Overview

File permissions are the foundation of Linux security. Every
deployment failure caused by "Permission denied", every CI/CD
pipeline that cannot write to a directory, every service that
cannot read its config file — all trace back to permissions and
ownership being wrong.

This demo covers the complete Linux permission model: standard
`rwx` permissions for owner/group/other, octal and symbolic
notation with `chmod`, ownership management with `chown` and
`chgrp`, default permissions with `umask`, the three special bits
(setuid, setgid, sticky), and Access Control Lists (ACLs) for
when the standard model is not flexible enough.

This is one of the most tested topics in LFCS, RHCSA, and DevOps
interviews. Getting permissions right separates engineers who
debug by trial and error from those who diagnose in 30 seconds.

---

## Demo Objectives

By the end of this demo you will be able to:

1. Read and interpret any permission string from `ls -l` output
2. Set permissions using both octal notation and symbolic notation
3. Change file ownership and group with `chown` and `chgrp`
4. Explain and calculate `umask` and its effect on new files
5. Identify and set the three special bits: setuid, setgid, sticky
6. Find dangerous permissions using `find` with `-perm`
7. Use ACLs (`getfacl`/`setfacl`) when standard permissions are
   not granular enough
8. Apply the correct permissions for common DevOps scenarios:
   web server files, SSH keys, scripts, shared directories

---

## Directory Structure

```
03-permissions/
├── README.md                    # this file
├── 03-permissions-anki.csv      # Anki flashcard deck
└── 03-permissions-quiz.md       # standalone quiz
```

---

## Environments

| Task | Ubuntu 24.04 | Rocky Linux 9.8 | EC2 Amazon Linux |
|---|---|---|---|
| All lab tasks | ✅ | ✅ | Not required |
| ACL filesystem support | ✅ ext4 | ✅ xfs | ✅ xfs |

---

## Recall Check

> Answer from memory before reading further. From Demo 02.

1. You delete a file but disk usage does not decrease.
   What command diagnoses this and what does it show?

2. `ls -li` shows two files with the same inode number.
   What are they and what happens if you delete one?

3. What does `ctime` measure? Why is it NOT creation time?

<details>
<summary>Answers</summary>

1. `lsof | grep deleted` — shows files deleted from the directory
   but still held open by a running process. The inode and disk
   blocks remain allocated until the process closes the file.

2. Hard links — two directory entries pointing to the same inode.
   Deleting one reduces the link count by 1. Data survives until
   link count reaches 0 and no process has it open.

3. `ctime` is change time — the last time the file's inode
   metadata changed (permissions, ownership, rename, any mtime
   change also updates ctime). It is NOT creation time. Linux
   does not traditionally store file creation time. The common
   mistake of reading `ctime` as "created time" is wrong.

</details>

---

## Part 1 — Concepts

### 1.1 Knowing your identity first — id and whoami

Before working with permissions, always confirm who you are.
The kernel makes all permission decisions based on your UID and
GIDs — not your username. These two commands tell you exactly
what identity the kernel sees for your current process.

**`whoami`** — **who am i** — prints your current effective
username. Simple, one word output.

```bash
whoami
# wadmin
```

**`id`** — **id**entity — prints your full identity: UID, primary
GID, and all supplementary groups. This is what the kernel
actually uses for permission checks.

```bash
id
# uid=1000(wadmin) gid=1000(wadmin) groups=1000(wadmin),10(wheel)
#  ↑                ↑                ↑
#  your UID         primary GID      all groups including supplementary
```

**Syntax:**
```bash
id                    # your full identity
id username           # identity of a specific user
id -u                 # UID only (useful in scripts: if [ $(id -u) -eq 0 ])
id -g                 # primary GID only
id -G                 # all GIDs (numbers)
id -Gn                # all group names
```

**When this matters for permissions:**
When you access a file, the kernel checks in order:
1. Does your UID match the file owner? → apply owner permissions
2. Does any of your GIDs match the file group? → apply group permissions
3. Neither matches → apply other permissions

Run `id` before debugging any "Permission denied" error — it
tells you immediately which category (owner/group/other) applies
to you for the file in question.

**How groups are assigned when a user is created:**

When `useradd` creates a user, two things happen by default on
most Linux systems:

**Primary group** — a new group with the same name and GID as
the user is created automatically. This is called the
**User Private Group (UPG)** scheme. The new user's primary GID
is set to this private group.

```bash
useradd wadmin
# Creates: user wadmin (UID=1000)
# Also creates: group wadmin (GID=1000)  ← UPG
# /etc/passwd: wadmin:x:1000:1000:...
#                               ↑ primary GID = 1000 (wadmin group)
```

**Supplementary groups** — none are assigned by default. The
user only belongs to their primary group until an administrator
explicitly adds them to other groups.

```bash
usermod -aG wheel wadmin    # add to wheel (sudo access)
usermod -aG docker wadmin   # add to docker
```

The full details of `useradd`, `usermod`, group management,
and the `/etc/passwd` / `/etc/group` / `/etc/shadow` files are
covered in **Demo 05 — Users, Groups & Authentication**.

---

### 1.2 The permission model — three levels, three types

Every file and directory has permissions for three categories:

```
u — user (owner)     the user who owns the file
g — group            the group assigned to the file
                     (a property of the FILE — set with chown/chgrp)
                     (not "the group the user belongs to")
o — other            everyone else — neither owner nor in file's group
a — all              shortcut for u+g+o used in chmod symbolic notation
```

And three permission types:

```
r — read     4    file: read contents    directory: list contents (ls)
w — write    2    file: modify contents  directory: create/delete files
x — execute  1    file: run as program   directory: enter (cd into it)
```

**Permissions on directories behave differently from files:**

```
Directory r (read):    can list the contents with ls
                       without r you cannot see what is inside
Directory w (write):   can create, delete, rename files inside
                       even if you cannot read those files' contents
Directory x (execute): can enter the directory with cd
                       can access files inside if you know their names
```

This is why web server document roots are typically `755`:
```
drwxr-xr-x   /var/www/html
   rwx        owner (root/www-data) can do everything
      r-x     group can list and enter
         r-x  others (the web browser users) can list and enter
```

### 1.2 Reading permission strings

The full permission string from `ls -l`:

```
-rwxr-xr-x   2   root   root   4096   Jun 7   filename
↑↑↑↑↑↑↑↑↑↑   ↑   ↑      ↑
│└─────────┘  │   │      └── group
│ permissions │   └── owner
│             └── link count
└── file type
    - = regular file
    d = directory
    l = symbolic link
    c = character device (terminal, serial port)
    b = block device (disk)
    p = named pipe
    s = socket
```

The 9 permission characters broken down:

```
rwxr-xr-x
│││└─────── other permissions (r-x)
│││
│└┘──────── group permissions (r-x)
│
└─┘──────── owner permissions (rwx)

Position 1: owner read    (r or -)
Position 2: owner write   (w or -)
Position 3: owner execute (x or -)  or s (setuid) or S (setuid, no x)
Position 4: group read    (r or -)
Position 5: group write   (w or -)
Position 6: group execute (x or -)  or s (setgid) or S (setgid, no x)
Position 7: other read    (r or -)
Position 8: other write   (w or -)
Position 9: other execute (x or -)  or t (sticky) or T (sticky, no x)
```

### 1.3 chmod — changing permissions

**`chmod`** = **ch**ange **mod**e. Mode is the Linux term for
the permission bits on a file or directory.

**Syntax:**
```bash
chmod [options] mode file
chmod [options] mode file1 file2 ...
```

**Key options:**

| Option | Meaning | Use |
|---|---|---|
| `-R` | Recursive | Apply to directory and all contents |
| `-v` | Verbose | Print each file changed |
| `--reference=ref` | Reference | Copy permissions from another file |

**Two notation modes** — octal (numeric) and symbolic. Both
achieve the same result. Octal is faster when you know the
exact value. Symbolic is clearer when adding/removing one bit.

**Octal notation:**

Each permission type has a numeric value: r=4, w=2, x=1.
Add the values for each category (owner/group/other) to get a
three-digit number:

```
7 = rwx = 4+2+1    full permissions
6 = rw- = 4+2+0    read and write
5 = r-x = 4+0+1    read and execute
4 = r-- = 4+0+0    read only
0 = --- = 0+0+0    no permissions

chmod 755 file    rwxr-xr-x   owner:rwx  group:r-x  other:r-x
chmod 644 file    rw-r--r--   owner:rw-  group:r--  other:r--
chmod 600 file    rw-------   owner:rw-  group:---  other:---
chmod 700 file    rwx------   owner:rwx  group:---  other:---
chmod 777 file    rwxrwxrwx   everyone:rwx  (DANGEROUS — avoid)
```

**Common permission values you must know:**

```
Permission  Octal  Typical use
─────────────────────────────────────────────────────
rwxr-xr-x   755   executables, scripts, directories
rw-r--r--   644   regular files, configs (world-readable)
rw-r-----   640   configs with sensitive data (group read)
rw-------   600   private keys, sensitive configs
rwx------   700   private directories
rwxrwxrwx   777   AVOID — world-writable is a security risk
r-xr-xr-x   555   read-only executables
```

**Symbolic notation:**

```
chmod u+x file       add execute for owner
chmod g-w file       remove write from group
chmod o= file        set other to no permissions
chmod a+r file       add read for all (owner+group+other)
chmod u=rwx,g=rx,o=r file   set exact permissions symbolically
chmod +x script.sh   add execute for all — common for scripts
chmod -R 755 /dir    recursive — apply to directory and all contents
```

**The danger of recursive chmod:**

Applying 755 recursively to a directory will also give execute permission to all files, which is usually not what you want for regular files. A safer approach for web directories:

```bash
# Correct way: different permissions for files vs directories
find /var/www/html -type d -exec chmod 755 {} \;
find /var/www/html -type f -exec chmod 644 {} \;
```

We cover `find` in depth in Demo 08. The pattern above is
referenced here so you know the correct approach — not
`chmod -R 755 /var/www/html` which makes all files executable.

### 1.4 chown and chgrp — changing ownership

**`chown`** = **ch**ange **own**er. Changes the user owner and/or
group assigned to a file or directory.

**`chgrp`** = **ch**ange **gr**ou**p**. Changes only the group.
`chown :group file` does the same thing — `chgrp` is the older
dedicated command.

**Syntax:**
```bash
chown [options] user[:group] file
chgrp [options] group file
```

**Key options:**

| Option | Meaning | Use |
|---|---|---|
| `-R` | Recursive | Apply to directory and all contents |
| `-v` | Verbose | Print each file changed |
| `--reference=ref` | Reference | Copy ownership from another file |

**Usage patterns:**

```bash
chown user file              # change owner only
chown user:group file        # change owner and group
chown :group file            # change group only (colon with no user before it)
                             # empty user field = leave owner unchanged
chgrp group file             # change group only (same result as chown :group)
chown -R user:group /dir     # -R = Recursive — apply to all contents
```

**Reading ownership in ls -l output:**
```
-rw-r--r--  1  root  nginx  /etc/nginx/nginx.conf
               ↑     ↑
               owner group
               (chown changes owner, chgrp changes group)
```

**Real DevOps scenarios:**

```bash
# After deploying app files as root, give ownership to app user
sudo chown -R www-data:www-data /var/www/myapp

# SSH key file must be owned by the user who will use it
chown wadmin:wadmin ~/.ssh/id_ed25519

# After mounting an NFS share, fix ownership to local user
sudo chown -R wadmin:wadmin /mnt/nfs-share
```

### 1.5 umask — default permission mask

**`umask`** = **u**ser file creation **mask**. Controls the default
permissions for newly created files and directories by masking
(removing) bits from the system defaults.

**The system defaults — same across all Linux distributions:**

The kernel always starts from these defaults when creating files:
```
Files:       666  (rw-rw-rw-)   — no execute by default (security)
Directories: 777  (rwxrwxrwx)   — execute needed to enter directories
```
These are hardcoded in the kernel's `open()` and `mkdir()` system
calls — not in a config file. The umask then removes bits from
these defaults.

**Where to check the system default umask:**
```bash
umask                          # your current session umask
cat /etc/login.defs | grep UMASK   # system-wide default for new users
grep umask /etc/profile        # shell-level default
grep umask /etc/bashrc         # bash-specific default
```

**How umask is calculated:**

```
System default for new files:       666  (rw-rw-rw-)
System default for new directories: 777  (rwxrwxrwx)

umask 022 removes:  ---w--w-  (write from group and other)
                                          
Result for files:   644  (rw-r--r--)
Result for dirs:    755  (rwxr-xr-x)
```

The subtraction is bitwise — umask bits that are set get removed
from the default:

```
umask    meaning              file result   dir result
─────────────────────────────────────────────────────
022      remove g-w, o-w      644           755   (standard)
027      remove g-w, o-rwx    640           750   (secure server)
077      remove g-rwx, o-rwx  600           700   (very restrictive)
002      remove o-w only      664           775   (developer default)
```

```bash
# View current umask
umask           # shows current value (e.g. 0022)
umask -S        # shows symbolic form (u=rwx,g=rx,o=rx)

# Set umask temporarily (current session only)
umask 027

# Set umask permanently — add to ~/.bashrc or /etc/profile
echo "umask 027" >> ~/.bashrc
```

**DevOps use case:**
CI/CD pipelines running as a service account should use `umask 027`
or `umask 077` to prevent accidentally creating world-readable
artifact files.

### 1.6 Special permissions — setuid, setgid, sticky bit

These are the three special bits beyond standard `rwx`. They
share the execute position in the permission string — the
character shown depends on whether the execute bit is also set.

**Setuid (SUID) — run as file owner:**

**`setuid`** = **set** **u**ser **id** on execution.

**Applies to: executable binary files only.**
The kernel ignores setuid on shell scripts — this is a deliberate
security measure.

```
-rwsr-xr-x  /usr/bin/passwd
   ↑
   s in owner execute position = setuid SET + execute SET
```

When a binary has setuid, it runs with the **owner's UID**
regardless of who executes it. Without setuid, a process runs
with the caller's UID.

**Real examples of setuid binaries:**
```bash
ls -l /usr/bin/passwd    # -rwsr-xr-x  root — writes to /etc/shadow
ls -l /usr/bin/sudo      # -rwsr-xr-x  root — runs commands as root
ls -l /usr/bin/ping      # -rwsr-xr-x  root — needs raw socket
ls -l /usr/bin/su        # -rwsr-xr-x  root — changes user identity
```

**How to set and remove setuid:**
```bash
chmod u+s file       # add setuid
chmod u-s file       # remove setuid
chmod 4755 file      # octal: 4=setuid, 755=rwxr-xr-x
chmod 0755 file      # octal: 0=no special bits, removes setuid
```

**lowercase s vs uppercase S:**

The execute bit and setuid share position 3 in the string.
What appears depends on whether execute is also set:

```
chmod u+s file        (no execute bit set first)
ls -l → -rwSr--r--
           ↑
           S (uppercase) = setuid IS set, but execute is NOT set
           The binary cannot execute at all — setuid is useless here
           This is almost always a mistake

chmod u+s file && chmod +x file   (or: chmod 4755 file)
ls -l → -rwsr-xr-x
           ↑
           s (lowercase) = setuid IS set AND execute IS set
           Correct — binary executes and runs as owner's UID
```

The same uppercase/lowercase rule applies to setgid (position 6)
and sticky (position 9 shows `T` when sticky set but no execute).

**Security implication for DevOps/containers:**
Setuid binaries are a container escape vector. A container image
with unexpected root-owned setuid binaries lets non-root container
processes gain root. Container scanners and Dockerfiles explicitly
check for and remove unexpected setuid bits.

---

**Setgid (SGID) — two different behaviours depending on target:**

**`setgid`** = **set** **g**roup **id** on execution.

**On executable files:**
```
-rwxr-sr-x  /usr/bin/wall
      ↑
      s in group execute position = setgid on file
```
Binary runs with the **group's GID** regardless of caller's group.
Less commonly used than setuid — mainly for programs needing
specific group access.

**On directories (the important DevOps use case):**
```bash
chmod g+s /shared/project   # or: chmod 2775 /shared/project
ls -ld /shared/project
# drwxrwsr-x  root  developers  /shared/project
#        ↑ s = setgid on directory
```
New files created inside inherit the directory's group — not the
creating user's primary group.

```
Without setgid on /shared/project:
  alice (primary group: alice) creates file
  → file group: alice    ← her primary group, not useful for team

With setgid on /shared/project:
  alice creates file
  → file group: developers  ← directory's group, correct for team
```

**How to set and remove setgid:**
```bash
chmod g+s /dir       # add setgid to directory
chmod g-s /dir       # remove setgid
chmod 2775 /dir      # octal: 2=setgid, 775=rwxrwxr-x
```

---

**Sticky bit — restrict deletion in shared directories:**

**Applies to: directories only.**
On regular files, the sticky bit is obsolete — modern Linux
kernels ignore it completely on files.

```
drwxrwxrwt  /tmp
         ↑
         t in other execute position = sticky bit
```

On a world-writable directory, the sticky bit means **only the
file's owner (and root) can delete their own files**.

```
/tmp without sticky:  any user can delete any other user's files
/tmp with sticky:     you can only delete files you own
```

Used on: `/tmp`, `/var/tmp`, shared project directories.

**How to set and remove sticky bit:**
```bash
chmod +t /dir        # add sticky bit
chmod -t /dir        # remove sticky bit
chmod 1777 /tmp      # octal: 1=sticky, 777=rwxrwxrwx
```

---

**Octal prefix summary for all three special bits:**

```
4000  setuid   chmod 4755 file  →  -rwsr-xr-x
2000  setgid   chmod 2755 dir   →  drwxr-sr-x
1000  sticky   chmod 1777 dir   →  drwxrwxrwt

Combined: chmod 6755 file  →  setuid + setgid + rwxr-xr-x
```

**Finding special permission files — security audit:**

```bash
find / -perm -4000 -type f 2>/dev/null   # setuid files
find / -perm -2000 -type f 2>/dev/null   # setgid files
find / -perm -o+w -type f 2>/dev/null    # world-writable files
find / -perm -o+w -type d 2>/dev/null    # world-writable dirs
```

Run these on your systems — the output is educational and this
exact pattern is used in real security audits.

### 1.7 ACLs — when standard permissions are not enough

**The problem standard permissions cannot solve:**

Standard Unix permissions have exactly three principals:
- **owner** — the one user who owns the file
- **group** — one group assigned to the file
- **other** — everyone else

What if you have a file owned by `root:sysadmin` and you need:
- `sysadmin` group members: read+write
- `auditor` user (not in sysadmin group): read only
- Everyone else: no access

There is no way to express this with standard `rwx` permissions
alone — you would have to make it world-readable or add `auditor`
to the `sysadmin` group (a security escalation). This is the
exact problem ACLs solve.

**What ACLs are:**

ACL = **Access Control List**. A list of additional permission
entries on a file beyond the standard owner/group/other. Each
entry specifies a named user or group with its own permissions.

**Where ACL data is stored:**

ACLs are stored in **extended attributes** on the filesystem
inode — specifically in `system.posix_acl_access`. They are
part of the inode's extended attribute block, separate from the
standard 9 permission bits. This is why `ls -l` shows a `+`
after permissions — it signals "there is more data on this inode
than the standard permission bits show."

Both ext4 (Ubuntu) and xfs (Rocky9/EC2) support ACLs natively.

**Install ACL tools if needed:**

```bash
sudo apt install acl -y       # Ubuntu
sudo dnf install acl -y       # Rocky9
```

**Recognising ACLs in ls output:**

```bash
ls -l /etc/nginx/nginx.conf
# -rw-r--r--   root  root  nginx.conf    ← no ACL
# -rw-r--r--+  root  root  nginx.conf    ← + means ACL is set
#            ↑
#            + = extended ACL exists — use getfacl to see it
```

**`getfacl` — read ACLs:**

**`getfacl`** = **get** **f**ile **a**ccess **c**ontrol **l**ist.

```bash
getfacl /etc/nginx/nginx.conf
```

**Reading getfacl output field by field:**

```
# file: nginx.conf          ← filename
# owner: root               ← file owner (same as ls -l)
# group: sysadmin           ← file group (same as ls -l)
user::rw-                   ← owner permissions (:: = owner, no name)
user:auditor:r--            ← named user ACL entry: auditor gets r--
group::rw-                  ← owning group permissions
group:devops:r--            ← named group ACL entry: devops group gets r--
mask::rw-                   ← ACL mask (ceiling for named entries)
other::---                  ← other permissions
```

The `::` (double colon with no name) entries are the standard
permissions — they mirror what you see in `ls -l`. The named
entries (`user:auditor:r--`, `group:devops:r--`) are the ACL
additions.

**`setfacl` — set ACLs:**

**`setfacl`** = **set** **f**ile **a**ccess **c**ontrol **l**ist.

```bash
# Grant a specific user read access
setfacl -m u:auditor:r /etc/nginx/nginx.conf
# -m = modify — add or update ACL entry
# u:auditor:r — u=user, auditor=username, r=read permission

# Grant a specific group read+write access
setfacl -m g:developers:rw /srv/project/
# g:developers:rw — g=group, developers=group name, rw=permissions

# Recursive — apply to directory and all contents
setfacl -R -m u:auditor:rX /var/log/nginx/
# -R = Recursive
# X = conditional execute: adds execute only if already executable
#     use X (not x) for mixed file/directory trees

# Remove one specific ACL entry
setfacl -x u:auditor /etc/nginx/nginx.conf
# -x = remove this specific entry

# Remove ALL ACLs — return to standard permissions only
setfacl -b /etc/nginx/nginx.conf
# -b = blank all ACLs — + disappears from ls -l

# Default ACL on directory — new files inherit this ACL
setfacl -d -m g:developers:rw /srv/project/
# -d = default — applied automatically to new files created inside
#     does NOT change existing files
```

**Flag summary:**

| Flag | Meaning | Use |
|---|---|---|
| `-m` | modify | Add or update ACL entry |
| `-x` | remove | Remove one specific entry |
| `-b` | blank | Remove all ACL entries |
| `-R` | Recursive | Apply to directory tree |
| `-d` | default | Set default ACL (inherited by new files) |

**The ACL mask — effective permission ceiling:**

The mask limits the maximum effective permission for ALL named
user and group entries (except the file owner). It does not
affect the owner or other.

```
user::rw-           ← owner — mask does NOT apply
user:auditor:rwx    ← auditor wants rwx
mask::r--           ← mask limits to r--
                       effective permission for auditor = r-- (not rwx)
other::---          ← other — mask does NOT apply
```

The mask is automatically recalculated by `setfacl` when you
add entries. Set it explicitly if needed:
```bash
setfacl -m m:rw file    # allow up to rw for all named entries
```

**Real DevOps scenario — the correct use case:**

```bash
# Problem:
# /etc/nginx/nginx.conf owned by root:root, permissions 644
# ci-runner user needs read access for audit
# Cannot add ci-runner to root group (security risk)
# Cannot make world-readable (security risk)

# Solution: ACL grants exactly what is needed
sudo setfacl -m u:ci-runner:r-- /etc/nginx/nginx.conf
sudo setfacl -m u:ci-runner:--x /etc/nginx/

# Verify
getfacl /etc/nginx/nginx.conf
# user::rw-
# user:ci-runner:r--   ← ci-runner can read, nothing else
# group::r--
# mask::r--
# other::r--
```

---

### 1.8 When you create a file — which group is assigned?

When you create a new file or directory, Linux assigns the group
automatically. The rule is simple but has one important exception.

**Default rule — primary group:**

The new file gets the **creator's primary group** — the GID shown
in the first position of `id` output.

```bash
id
# uid=1000(wadmin) gid=1000(wadmin) groups=1000(wadmin),10(wheel)
#                  ↑
#                  primary GID = 1000 (wadmin group)

touch newfile.txt
ls -l newfile.txt
# -rw-r--r--  1  wadmin  wadmin  newfile.txt
#                         ↑
#                         group = primary group (wadmin)
```

**The exception — setgid directory:**

If the directory you are creating the file *inside* has the
**setgid bit** set, the new file inherits the **directory's group**
instead of your primary group.

```bash
# Directory with setgid:
ls -ld /srv/project/
# drwxrwsr-x  root  developers  /srv/project/
#        ↑ s = setgid set, group = developers

# You are wadmin, your primary group is wadmin
touch /srv/project/myfile.txt
ls -l /srv/project/myfile.txt
# -rw-r--r--  1  wadmin  developers  myfile.txt
#                          ↑
#                          group = directory's group (developers)
#                          NOT your primary group (wadmin)
```

**Practical example — why this matters:**

```bash
# Scenario: /var/www/project/ owned by root:webteam
# WITHOUT setgid:
ls -ld /var/www/project/
# drwxrwxr-x  root  webteam  /var/www/project/   ← no s

# alice (primary group: alice) creates a file
touch /var/www/project/index.html
ls -l /var/www/project/index.html
# -rw-r--r--  alice  alice  index.html   ← group is alice, not webteam
# nginx (running as www-data in webteam group) cannot write to it

# WITH setgid:
sudo chmod g+s /var/www/project/
touch /var/www/project/style.css
ls -l /var/www/project/style.css
# -rw-r--r--  alice  webteam  style.css   ← group is webteam
# nginx can now access it correctly
```

**same rule applies to directories as to files:**

```bash
# Regular directory — mkdir assigns your primary group
mkdir /srv/project/mysubdir
ls -ld /srv/project/mysubdir
# drwxr-xr-x  wadmin  wadmin  mysubdir
#                      ↑
#                      primary group (wadmin)

# Inside a setgid directory — mkdir inherits directory's group
ls -ld /srv/project/
# drwxrwsr-x  root  developers  /srv/project/  ← setgid set

mkdir /srv/project/assets/
ls -ld /srv/project/assets/
# drwxr-sr-x  wadmin  developers  assets/
#                       ↑           ↑
#                       inherited   notice: setgid bit is also
#                       from parent inherited by the new subdir
```

The setgid bit propagates — new subdirectories created inside a
setgid directory also get the setgid bit set automatically. This
means the entire tree maintains the same group assignment behaviour
without manual intervention. Files do NOT inherit the setgid bit —
only subdirectories do.

**Summary:**

```
When creating inside a directory:

                         Without setgid      With setgid (g+s)
                         ──────────────────────────────────────
New file — group         your primary group  directory's group
New file — setgid bit    not set             not set
New subdir — group       your primary group  directory's group
New subdir — setgid bit  not set             inherited (set automatically)
```

This is why the break-fix scenario in Part 3 uses `chmod g+s` —
not to change existing files, but to control what group all
*future* files get automatically.

---

## Part 2 — Hands-On Lab

### Commands used in this demo

| Command | Letter meaning | Purpose |
|---|---|---|
| `ls -l` | l = long | View permissions in long format |
| `chmod` | ch = change, mod = mode | Change file permissions |
| `chown` | ch = change, own = owner | Change file owner and/or group |
| `chgrp` | ch = change, grp = group | Change group only |
| `umask` | u = user, mask = permission mask | View or set default permission mask |
| `stat` | stat = status | View complete file metadata including permissions |
| `find -perm` | perm = permissions | Find files matching a permission pattern |
| `getfacl` | get + facl = file ACL | Read ACL entries on a file |
| `setfacl` | set + facl = file ACL | Set ACL entries on a file |
| `id` | id = identity | Show current user's UID, GID, and groups |
| `whoami` | who am i | Show current effective username |

---

### Task 1 — Know your identity, then read permissions

**Run on Ubuntu, then Rocky9:**

```bash
# Always start with your identity — the kernel uses these numbers
# for every permission check
whoami
id
```

**Reading `id` output:**
```
uid=1000(wadmin) gid=1000(wadmin) groups=1000(wadmin),10(wheel)
 ↑                ↑                ↑
 your UID         primary GID      all groups: primary + supplementary

When you access a file the kernel checks in order:
1. Does your UID (1000) match the file owner UID?  → owner permissions
2. Does any of your GIDs match the file group GID? → group permissions
3. Neither matches?                                 → other permissions
```

```bash
# Create test area
mkdir -p ~/demo03
cd ~/demo03

# Create files to work with
echo "config data" > app.conf
echo '#!/bin/bash' > deploy.sh
echo "log entry" > app.log
mkdir -p shared-dir

# View all permissions
ls -la ~/demo03/
```

**What to look for in the output:**

> New files will show your username as owner and your **primary
> group** as group. On a standard install using the User Private
> Group scheme, the primary group name matches your username —
> which is why both columns show the same name. This is not always
> the case when users are created with a shared primary group.

The default permissions come from your current umask — typically
`644` for files and `755` for directories with `umask 022`.

```bash
# Inspect real system files — special bits in the wild
ls -l /usr/bin/passwd     # -rwsr-xr-x — setuid in action
ls -l /usr/bin/wall 2>/dev/null || ls -l /usr/bin/write 2>/dev/null
ls -ld /tmp               # drwxrwxrwt — sticky bit in action
                          # -d = directory itself, not its contents
```

**Paste your full output from both systems.**

---

### Task 2 — chmod in both modes

**Run on Ubuntu, then Rocky9:**

```bash
cd ~/demo03

# View current permissions
ls -la

# Octal mode — set exact permissions
chmod 755 deploy.sh      # executable script
chmod 644 app.conf       # readable config
chmod 600 app.log        # private log
chmod 700 shared-dir     # private directory

ls -la

# Symbolic mode — add/remove/set
chmod +x deploy.sh       # add execute for all
chmod g-w app.conf       # remove group write
chmod o= app.log         # set other to nothing
chmod u=rwx,g=rx,o= deploy.sh   # set exact symbolically

ls -la

# Verify with stat — see octal value explicitly
stat app.conf | grep "Access: ("
# Output: Access: (0644/-rw-r--r--)  ← octal on left, symbolic on right
```

**The stat output shows octal directly** — useful when you need
to confirm exact permission value without mental arithmetic.

**Paste your output after each `ls -la`.**

---

### Task 3 — umask

**Run on Ubuntu, then Rocky9:**

```bash
# Check current umask
umask
umask -S

# Create files and see what permissions they get
touch umask-test-file
mkdir umask-test-dir
ls -la umask-test-file umask-test-dir

# Change umask and observe the difference
umask 077

touch restricted-file
mkdir restricted-dir
ls -la restricted-file restricted-dir
# Owner only — group and other have nothing

# Change to developer-friendly umask
umask 002
touch group-writable-file
ls -la group-writable-file
# Owner and group can write

# Restore default
umask 022

# Show umask values from /etc/profile and /etc/bashrc
grep -r "umask" /etc/profile /etc/bashrc /etc/profile.d/ 2>/dev/null
```

**Paste the output showing permissions change with different umask values.**

---

### Task 4 — Special bits in action

**Run on Ubuntu, then Rocky9:**

```bash
cd ~/demo03

# --- Setuid ---
# Find real setuid binaries on your system
find /usr/bin -perm -4000 -type f 2>/dev/null | head -5
ls -l /usr/bin/passwd    # classic setuid example

# Create a test script and set setuid (educational — no real effect on scripts)
cp /bin/cat ~/demo03/mycat 2>/dev/null || cp /usr/bin/cat ~/demo03/mycat
chmod 4755 ~/demo03/mycat
ls -l ~/demo03/mycat     # should show -rwsr-xr-x

# --- Setgid on directory ---
mkdir ~/demo03/team-project
chmod 2775 ~/demo03/team-project
ls -ld ~/demo03/team-project     # should show drwxrwsr-x
# Any file created here will inherit the directory's group

# --- Sticky bit ---
mkdir ~/demo03/shared-tmp
chmod 1777 ~/demo03/shared-tmp
ls -ld ~/demo03/shared-tmp       # should show drwxrwxrwt

# --- Security audit: find all setuid files ---
echo "=== Setuid files on this system ==="
find /usr -perm -4000 -type f 2>/dev/null
find /bin /sbin -perm -4000 -type f 2>/dev/null

# --- Find world-writable directories (excluding /proc /sys) ---
echo "=== World-writable directories ==="
find / -perm -o+w -type d \
  -not -path "/proc/*" \
  -not -path "/sys/*" \
  -not -path "/dev/*" \
  2>/dev/null | head -10
```

**Paste the output. Note any unexpected setuid files.**

---

### Task 5 — ACLs

**Scenario for this task:**

You have a config file owned by `root:sysadmin` with permissions
`640`. A new user `auditor` needs read access. You cannot make
it world-readable (security risk) and you cannot add `auditor`
to the `sysadmin` group (privilege escalation). ACLs are the
correct solution.

**Run on Ubuntu, then Rocky9:**

```bash
# Install acl tools if needed
sudo apt install acl -y 2>/dev/null || sudo dnf install acl -y 2>/dev/null

# --- Setup: create the scenario ---
# Create test group and user (the "other user" who needs access)
sudo groupadd sysadmin-team 2>/dev/null || true
sudo useradd -m -s /bin/bash -c "Auditor User" auditor 2>/dev/null || true
echo "auditor:AuditPass123!" | sudo chpasswd

# Create the protected file owned by root:sysadmin-team
sudo mkdir -p /opt/acl-demo
echo "sensitive config data" | sudo tee /opt/acl-demo/app.conf
sudo chown root:sysadmin-team /opt/acl-demo/app.conf
sudo chmod 640 /opt/acl-demo/app.conf

# Verify the setup
ls -la /opt/acl-demo/app.conf
# -rw-r-----  root  sysadmin-team  app.conf
# owner=root: rw-  group=sysadmin-team: r--  other: ---

id auditor
# auditor is NOT in sysadmin-team — falls into "other" — no access

# --- Prove the problem ---
sudo -u auditor cat /opt/acl-demo/app.conf
# cat: /opt/acl-demo/app.conf: Permission denied  ← confirmed

# --- Step 1: Read the current ACL (no extra entries yet) ---
getfacl /opt/acl-demo/app.conf
# # file: app.conf
# # owner: root
# # group: sysadmin-team
# user::rw-       ← owner (root)
# group::r--      ← sysadmin-team
# other::---      ← everyone else including auditor
# No + in ls output yet

# --- Step 2: Grant auditor read access via ACL ---
sudo setfacl -m u:auditor:r-- /opt/acl-demo/app.conf
sudo setfacl -m u:auditor:--x /opt/acl-demo/   # need x to enter dir

# --- Step 3: Verify ACL is set ---
ls -la /opt/acl-demo/app.conf
# -rw-r-----+  root  sysadmin-team  app.conf   ← + appeared

getfacl /opt/acl-demo/app.conf
# user::rw-
# user:auditor:r--    ← new ACL entry
# group::r--
# mask::r--
# other::---

# --- Step 4: Confirm auditor can now read ---
sudo -u auditor cat /opt/acl-demo/app.conf
# sensitive config data  ← access granted

# --- Step 5: Default ACL — new files inherit automatically ---
sudo setfacl -d -m u:auditor:r-- /opt/acl-demo/
getfacl /opt/acl-demo/
# default:user:auditor:r--  ← future files will automatically get this

echo "new config entry" | sudo tee /opt/acl-demo/new.conf
getfacl /opt/acl-demo/new.conf
# user:auditor:r--  ← inherited automatically from directory default ACL

# --- Step 6: Remove the ACL entries ---
sudo setfacl -x u:auditor /opt/acl-demo/app.conf
getfacl /opt/acl-demo/app.conf   # auditor entry gone

sudo setfacl -b /opt/acl-demo/app.conf
ls -la /opt/acl-demo/app.conf    # + is gone — back to standard permissions

# --- Cleanup ---
sudo userdel -r auditor 2>/dev/null
sudo groupdel sysadmin-team 2>/dev/null
sudo rm -rf /opt/acl-demo
```

**What to observe at each stage:**
```
Before setfacl:    ls -l shows no +,  auditor gets Permission denied
After setfacl:     ls -l shows +,     auditor can read
After setfacl -b:  ls -l shows no +,  back to standard permissions only
```

**Paste the getfacl output after Step 2 and after Step 6.**

---

## Part 3 — Break-Fix Scenario

### Scenario — NovaTech deployment pipeline failure

Two related incidents reported together:

> *"Incident 1: Our deploy script at `/opt/novatech/deploy.sh`
> fails with 'Permission denied' when run by the CI runner user
> `ci-runner`. Works fine when I run it manually as my user."*

> *"Incident 2: The web team says new files created in
> `/var/www/project/` always end up owned by the wrong group.
> Each developer's files show their own username as group instead
> of `webteam`. We have to manually `chgrp` everything."*

You investigate:

```bash
# Incident 1 — deploy script
ls -l /opt/novatech/deploy.sh
# -rw-r--r--  1  wadmin  wadmin  deploy.sh

id ci-runner
# uid=1002(ci-runner) gid=1002(ci-runner) groups=1002(ci-runner)

# Incident 2 — web directory
ls -ld /var/www/project/
# drwxrwxr-x  2  root  webteam  /var/www/project/

touch /var/www/project/testfile
ls -l /var/www/project/testfile
# -rw-rw-r--  1  wadmin  wadmin  testfile  ← group is wadmin, not webteam
```

**Your task:**

1. Why does the deploy script fail for `ci-runner`?
   What is the minimum permission change needed?

2. Why do new files in `/var/www/project/` inherit the creator's
   group instead of `webteam`? What single permission fixes this
   permanently for all future files?

3. Write the exact commands to fix both issues.

<details>
<summary>Resolution — open after attempting</summary>

**Incident 1 — Permission denied on deploy.sh:**

```bash
ls -l /opt/novatech/deploy.sh
# -rw-r--r--  1  wadmin  wadmin  deploy.sh
```

The file has permissions `644` — no execute bit for anyone.
`ci-runner` is not the owner and not in the `wadmin` group, so
falls into "other" which has `r--` — read only, no execute.

A script requires the execute bit to run. Without it, the shell
cannot execute it as a program.

**Minimum fix — add execute permission:**
```bash
chmod 755 /opt/novatech/deploy.sh
# or more restrictive if only ci-runner needs to run it:
chmod 750 /opt/novatech/deploy.sh
chown wadmin:ci-runner /opt/novatech/deploy.sh
```

**Incident 2 — Wrong group on new files:**

`/var/www/project/` does not have the setgid bit. When a user
creates a file, it gets the user's primary group (wadmin in this
case), not the directory's group (webteam).

**Fix — set the setgid bit on the directory:**
```bash
sudo chmod g+s /var/www/project/
# or with octal: chmod 2775 /var/www/project/
ls -ld /var/www/project/
# drwxrwsr-x  2  root  webteam  /var/www/project/
#        ↑ s = setgid now active
```

From this point, all new files created inside inherit `webteam`
as their group automatically. No more manual `chgrp` needed.

**Both fixes together:**
```bash
# Fix 1: deploy script executable
sudo chmod 755 /opt/novatech/deploy.sh

# Fix 2: setgid on web directory
sudo chmod g+s /var/www/project/

# Verify
ls -l /opt/novatech/deploy.sh     # should show -rwxr-xr-x
ls -ld /var/www/project/           # should show drwxrwsr-x
touch /var/www/project/newtest
ls -l /var/www/project/newtest    # group should now be webteam
```

</details>

---

## Interview Prep

**Q1. A web server returns 403 Forbidden even though the file
exists and is readable by root. What do you check?**

Check the execute bit on every directory in the path. A web
server process (e.g., `www-data`) needs execute permission on
every directory from `/` down to the file to traverse the path.
If any directory in the chain is missing execute for the web
server's user or group, access is denied — even if the file
itself has `644` permissions.

```bash
ls -ld /var/www/
ls -ld /var/www/html/
ls -ld /var/www/html/myapp/
ls -l /var/www/html/myapp/index.html
```

Also check the process identity: `ps aux | grep nginx` —
what user is the web server running as?

---

**Q2. What is the difference between `chmod 4755` and `chmod 4555`?
When would setuid be dangerous on a shell script?**

`chmod 4755` sets setuid + `rwxr-xr-x` — owner (often root) can
read, write, execute; group and others can read and execute.
`chmod 4555` sets setuid + `r-xr-xr-x` — no one can write,
including the owner. Used on very sensitive binaries.

Setuid on shell scripts is **ignored by the Linux kernel** as a
security measure — the kernel does not honour setuid on interpreted
scripts, only on compiled ELF binaries. If you set setuid on a
bash script, it has no effect. However, some older `sudo` misuse
or wrapper binaries can achieve the same effect — this is why
security audits flag unexpected setuid binaries for review.

---

**Q3. What does `umask 027` mean in a production server context?
Where would you set it?**

`umask 027` removes write from group and all permissions from
other. New files get `640` (owner rw, group r, others nothing).
New directories get `750`.

This means: service accounts running applications cannot
accidentally create world-readable files. Useful for CI/CD
runners, app service accounts, log-writing services.

Set it in `/etc/profile.d/security.sh` for all interactive users,
or in the service's systemd unit file with `UMask=0027` for
non-interactive services.

---

**Q4. You need to give one specific user read access to a file
without making it world-readable and without adding that user
to the file's group. What do you use?**

ACL (Access Control List) with `setfacl`:

```bash
setfacl -m u:username:r /path/to/file
getfacl /path/to/file   # verify effective permissions
```

This grants read access to exactly one user without changing
ownership, group membership, or standard permissions. The file
remains locked down for everyone else.

---

## Key Takeaways

1. **Permissions have three levels (owner/group/other) and three types (read/write/execute) — on directories, `x` means the right to enter, not just to execute a binary.** A web server missing execute on any directory in the path to a file returns 403 even if the file itself is readable.

2. **Memorise the four permission values you use every day: `755` for executables and directories, `644` for regular files, `600` for private keys, `640` for sensitive configs readable by a group.** These cover 90% of daily use without mental arithmetic.

3. **Octal: `r=4`, `w=2`, `x=1`. Add per category. `755 = rwxr-xr-x`, `644 = rw-r--r--`, `600 = rw-------`.** The execute bit in position 3/6/9 shows `s` (setuid/setgid with execute), `S` (setuid/setgid without execute — almost always a mistake), or `t`/`T` (sticky with/without execute).

4. **`umask 022` is standard (new files get `644`). `umask 027` is hardened (new files get `640`, no access for others).** Service accounts and CI/CD runners should use `umask 027` or stricter — set in the systemd unit with `UMask=0027` so it is enforced regardless of shell config.

5. **Setgid on a directory makes all new files and subdirectories inherit the directory's group — not the creator's primary group.** New subdirectories also inherit the setgid bit itself. New files do not. This is the correct fix for shared team directories where files keep showing the wrong group.

6. **`chmod -R 755` on a directory also makes every regular file executable — this is wrong for web content and configs.** The correct approach uses `find -type d -exec chmod 755` and `find -type f -exec chmod 644` separately.

7. **`+` after permissions in `ls -l` means an ACL is set — `getfacl filename` shows the full picture.** Standard `ls -l` cannot display ACL entries. Use ACLs when you need to grant one specific user or group access without changing the file's owner, group, or making it world-readable.

---

## Appendix A — Quick Reference

### Permission values — memorise these

| Octal | Symbolic | Meaning | Typical use |
|---|---|---|---|
| `755` | `rwxr-xr-x` | Owner full, others read+exec | Scripts, dirs, binaries |
| `644` | `rw-r--r--` | Owner rw, others read | Config files, web files |
| `600` | `rw-------` | Owner only | SSH private keys, secrets |
| `700` | `rwx------` | Owner only, executable | Private directories |
| `640` | `rw-r-----` | Owner rw, group read | Sensitive configs |
| `664` | `rw-rw-r--` | Owner+group rw | Shared development files |
| `777` | `rwxrwxrwx` | Everyone everything | **Avoid — security risk** |

### chmod symbolic notation

| Syntax | Meaning |
|---|---|
| `u+x` | Add execute to owner |
| `g-w` | Remove write from group |
| `o=` | Set other to nothing |
| `a+r` | Add read to all |
| `+x` | Add execute to all (shorthand for `a+x`) |
| `u=rwx,g=rx,o=r` | Set exact permissions |

### Special bits — octal prefix

| Prefix | Bit | Effect |
|---|---|---|
| `4000` | setuid | File runs as owner |
| `2000` | setgid on file | File runs as group |
| `2000` | setgid on dir | New files inherit dir's group |
| `1000` | sticky | Only owner can delete in dir |

### find with permissions — security audit patterns

```bash
find / -perm -4000 -type f 2>/dev/null   # setuid files
find / -perm -2000 -type f 2>/dev/null   # setgid files
find / -perm -o+w -type f 2>/dev/null    # world-writable files
find / -perm -o+w -type d 2>/dev/null    # world-writable dirs
find / -nouser -type f 2>/dev/null       # orphaned files (no owner)
```

---

## Appendix B — Anki Cards

**`03-permissions-anki.csv`:**

```
#deck:Linux Mastery::Module 1 - Filesystem & Navigation::Demo 03 - Permissions
#separator:Comma
#columns:Front,Back,Tags

"ls -l shows -rwsr-xr-x on /usr/bin/passwd. What does the s mean and why is it needed?","The s in the owner execute position is the setuid bit. When any user runs passwd, the process temporarily runs with root's permissions (passwd is owned by root). This allows writing to /etc/shadow which only root can access. Without setuid, users could never change their own passwords.","demo03,setuid,permissions"

"You create a file in /var/www/project/. It shows your username as group instead of webteam. How do you fix this permanently for all future files?","Set the setgid bit on the directory: chmod g+s /var/www/project/ or chmod 2775 /var/www/project/. With setgid, all new files created inside inherit the directory's group (webteam) instead of the creator's primary group. Existing files are unaffected — only new files follow the rule.","demo03,setgid,permissions"

"What is umask 027? What permissions does it give to new files and directories?","umask 027 removes write from group and all permissions from others. New files: 640 (rw-r-----). New dirs: 750 (rwx r-x ---). Production use: service accounts should use umask 027 so they cannot accidentally create world-readable files. Set in systemd unit with UMask=0027.","demo03,umask,security"

"A CI/CD pipeline gets Permission denied on deploy.sh. ls -l shows -rw-r--r--. What is wrong and what is the minimum fix?","No execute bit. The script has 644 — read/write for owner, read for group and others, no execute for anyone. Shell cannot execute a script without the execute bit. Fix: chmod 755 deploy.sh (or chmod +x deploy.sh). The x bit is required for any user to run the script as a program.","demo03,chmod,permissions,troubleshooting"

"What does the + at end of permissions mean in ls -l? Example: -rw-r--r--+","The + indicates an ACL (Access Control List) is set on this file beyond standard permissions. Standard ls -l cannot show the full picture. Use getfacl filename to see all ACL entries including per-user and per-group permissions.","demo03,acl,permissions"

"You need to give user ci-runner read access to /etc/nginx/nginx.conf without making it world-readable and without adding ci-runner to the root group. What do you use?","ACL: setfacl -m u:ci-runner:r /etc/nginx/nginx.conf. Then verify: getfacl /etc/nginx/nginx.conf. This grants exactly read permission to one specific user without changing standard permissions, group membership, or world-readability. This is the primary use case for ACLs over standard permissions.","demo03,acl,setfacl,security"

"What is wrong with chmod -R 755 /var/www/html/?","It applies execute permission to ALL files including regular HTML, CSS, JS files — not just directories. Regular files do not need the execute bit. The correct approach: find /var/www/html -type d -exec chmod 755 {} \\; && find /var/www/html -type f -exec chmod 644 {} \\;","demo03,chmod,recursive,security"

"A file has permissions rw-r--r-- and you run chmod o=. What are the new permissions?","rw-r----- (640). chmod o= sets other permissions to nothing (empty). Owner permissions (rw-) unchanged. Group permissions (r--) unchanged. Other permissions cleared to ---. Equivalent to: chmod 640 filename","demo03,chmod,symbolic"
```

---

## Appendix C — Quiz

**`03-permissions-quiz.md`:**

````markdown
# Quiz — Demo 03: File Permissions, Ownership & Access Control
> One correct answer per question unless stated otherwise.
> Target: 80% or above before moving to Demo 04.

---

**Q1.** `ls -l` shows `-rwxr-xr-x` on `/usr/bin/somecmd`
and `-rwsr-xr-x` on `/usr/bin/passwd`.
What is the security difference?

- A) passwd is read-only, somecmd is executable
- B) passwd runs with root's permissions when executed by any user
  due to the setuid bit. somecmd runs with the calling user's UID.
- C) passwd can only be run by root
- D) The `s` means passwd requires a password to run

<details>
<summary>Answer</summary>

**B** — The `s` in owner execute position is the setuid bit.
When any user runs `passwd`, the process effective UID becomes 0
(root) — allowing it to write `/etc/shadow`. When running `somecmd`,
the process runs with the caller's UID. Unexpected setuid on a
shell or editor would give any user root access.

</details>

---

**Q2.** What does `umask 027` produce for a newly created file?

- A) rwxrwxrwx (777)
- B) rw-r--r-- (644)
- C) rw-r----- (640)
- D) rwx------ (700)

<details>
<summary>Answer</summary>

**C** — Files start at system default 666. umask 027 removes:
0=owner nothing, 2=group write, 7=other all.
666 - 027 = 640 (rw-r-----)
Directories start at 777: 777 - 027 = 750 (rwxr-x---)

</details>

---

**Q3.** New files in `/srv/project/` always get the creator's
personal group instead of `projectteam`. What single command
fixes this permanently for all future files?

- A) `chown -R :projectteam /srv/project/`
- B) `chmod 775 /srv/project/`
- C) `chmod g+s /srv/project/`
- D) `setfacl -d -m g:projectteam:rw /srv/project/`

<details>
<summary>Answer</summary>

**C** — `chmod g+s` sets the setgid bit on the directory. New
files created inside inherit the directory's group (projectteam)
instead of the creator's primary group. Permanent for all future
files.
Trap: A changes existing files but not future ones. D (ACL
default) also works but is more complex than necessary here.

</details>

---

**Q4.** `ls -l` shows `-rw-r--r--+` on a file. What does the
`+` mean and what command shows the full permissions?

- A) `+` means the file is executable by all users
- B) `+` means an ACL is set — run `getfacl filename` to see all entries
- C) `+` means the file is owned by multiple users
- D) `+` means the file is locked

<details>
<summary>Answer</summary>

**B** — The `+` indicates an ACL (Access Control List) is set
beyond standard permissions. ACL entries are stored in extended
attributes on the inode. `ls -l` cannot show them — use
`getfacl filename` to see all named user and group entries,
the mask, and any default ACLs.

</details>

---

**Q5.** You want `chmod 755` on all directories under `/var/www`
and `chmod 644` on all files. Which approach is correct?

- A) `chmod -R 755 /var/www`
- B) `chmod -R 644 /var/www`
- C) `find /var/www -type d -exec chmod 755 {} \;` and
  `find /var/www -type f -exec chmod 644 {} \;`
- D) `chmod 755 /var/www` then `chmod 644 /var/www/*`

<details>
<summary>Answer</summary>

**C** — `find` applies different permissions to files and
directories separately. `chmod -R 755` (A) makes all regular
files executable — wrong. `chmod -R 644` (B) removes execute
from directories — you could not `cd` into them. D only works
for the top level, not subdirectories.

</details>

---

**Q6.** `/tmp` has permissions `drwxrwxrwt`. What does the `t`
mean and why is it critical?

- A) `t` means `/tmp` is temporary and cleared on reboot
- B) `t` is the sticky bit — only the file's owner can delete it
  even though everyone has write permission on the directory
- C) `t` means the directory is mounted as tmpfs
- D) `t` requires two-factor authentication

<details>
<summary>Answer</summary>

**B** — The `t` (sticky bit) on a world-writable directory means
each user can only delete their own files. Without it, any user
with write permission could delete any other user's files.
Note: the sticky bit is only meaningful on directories on modern
Linux — it is ignored on regular files.
Trap: A is true about `/tmp` (it IS tmpfs) but `t` does not
represent that — `t` means sticky bit only.

</details>

---

**Q7.** A security audit finds `/usr/local/bin/backdoor`
with permissions `-rwsr-xr-x` owned by root. Why is this critical?

- A) Execute permission is a risk on its own
- B) setuid + root owner — any user who runs this gets a process
  with root UID, enabling privilege escalation
- C) Files in `/usr/local/bin` should not be executable
- D) root-owned files in `/usr/local/bin` are always suspicious

<details>
<summary>Answer</summary>

**B** — setuid + root owner means the process runs as root
regardless of who invokes it. If this binary can spawn a shell
or write files, any user gains root privileges.
Detection: `find / -perm -4000 -type f 2>/dev/null`
Response: investigate origin, remove if unauthorised, verify
with `rpm -qf /path` (Rocky) or `dpkg -S /path` (Ubuntu) to
check if it belongs to any package.

</details>

---

Score guide:

| Score | Action |
|---|---|
| 7/7 | Import Anki cards, move to Demo 04 |
| 6/7 | Review wrong answer, proceed |
| 5/7 | Re-read relevant section, retry those questions |
| Below 5/7 | Re-read Demo 03 before proceeding |
````