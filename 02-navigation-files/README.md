# Demo 02 — Navigation, File Operations & Inodes

## Overview

In Demo 01 you learned the map — what lives where in the Linux
filesystem. In this demo you learn to move through that map and work
with files confidently. Navigation, file creation, copying, moving,
deletion, timestamps, and links are actions you perform hundreds of
times every day as a DevOps or sysadmin engineer.

Three concepts in this demo go deeper than most people realise:
**inodes** (the real identity of every file), **timestamps** (three
distinct values that tell you different things about a file's history),
and **hard vs soft links** (two fundamentally different ways to
reference the same data). These come up in interviews and in production
debugging.

---

## Demo Objectives

By the end of this demo you will be able to:

1. Navigate the filesystem confidently using `pwd`, `cd`, and `ls`
   with all important flags
2. Understand and use absolute vs relative paths
3. Create, copy, move, and delete files and directories safely
4. Read all three file timestamps — `atime`, `mtime`, `ctime` —
   and explain what changes each one
5. Use `stat` to read a file's complete inode information
6. Create and manage hard links and soft links — and explain
   the difference at the inode level
7. Understand what an inode is and why the link count in `ls -l`
   output matters
8. Understand what a directory entry is and why filenames are not
   stored in inodes

---

## Directory Structure

```
02-navigation-files/
├── README.md                          # this file
├── 02-navigation-files-anki.csv       # Anki flashcard deck
└── 02-navigation-files-quiz.md        # standalone quiz
```

---

## Environments

| Task | Ubuntu 24.04 | Rocky Linux 9.8 | EC2 Amazon Linux |
|---|---|---|---|
| All lab tasks | ✅ Primary | Optional | Not required |

The commands in this demo are standard across all Linux distros —
no meaningful output difference between Ubuntu and Rocky9 for basic
navigation, file operations, and inode concepts. Run on Ubuntu as
your primary environment. If you want to verify on Rocky9, the
output will be identical for these tasks.

---

## Recall Check

> Answer these from memory before reading further.
> These are from Demo 01.

1. A service's socket file is missing after a server reboot.
   The socket lived in `/run/myapp/`. Why is it missing and
   what does this tell you about `/run`?

2. You type `terraform` and get "command not found". The binary
   is at `/usr/local/bin/terraform`. What do you check next?

3. What is the difference between `/proc` (virtual filesystem)
   and `/run` (tmpfs)? Give one characteristic of each.

<details>
<summary>Answers</summary>

1. `/run` is tmpfs — RAM-backed, wiped completely on every reboot.
   The service must recreate its socket on startup. If the socket
   is missing, the service has not started yet.

2. Check `echo $PATH` — `/usr/local/bin` may not be in PATH.
   The binary exists but the shell cannot find it because
   `$PATH` does not include that directory.

3. `/proc` — virtual, kernel generates content on demand, size 0,
   nothing stored anywhere. `/run` — tmpfs, RAM storage, you
   write to it normally like a real filesystem, cleared on reboot.

</details>

---

## Part 1 — Concepts

### 1.1 pwd and cd — knowing where you are

`pwd` — Print Working Directory. Shows your exact current location.
Run it whenever you are unsure where you are, especially after
navigating with relative paths.

`cd` shortcuts every engineer must know:

```bash
cd          # go to home directory (~) — no argument means home
cd ~        # same as above — ~ always expands to your home path
cd -        # go to PREVIOUS directory (toggle between last two)
cd ..       # go up ONE level (.. = parent directory)
cd ../..    # go up TWO levels
cd /etc/ssh # absolute path — starts from root /
cd ssh      # relative path — from wherever you currently are
```

**`cd -` is one of the most useful shortcuts in daily work.**
During an incident you often switch between `/var/log` (logs),
`/etc/nginx` (config), and `/run` (runtime state). `cd -` takes
you back instantly without retyping the path.

**Absolute vs relative paths:**

```
Absolute path: starts with /
  /etc/nginx/nginx.conf     always means the same location
  /var/log/syslog           regardless of where you currently are

Relative path: starts from current directory
  ./nginx.conf              the file in the current directory
  ../logs/access.log        go up one level, then into logs/
  configs/app.conf          configs/ inside current directory
```

The `.` (dot) means current directory. `..` means parent directory.
These appear constantly in scripts, Docker build contexts, and
git operations.

### 1.2 ls — reading the full output

`ls` is the most-used command in Linux. Every flag serves a specific
purpose:

| Flag | Meaning of the letter | What it does |
|---|---|---|
| `-l` | long | Long format — permissions, owner, size, date |
| `-a` | all | Show hidden files (names starting with `.`) |
| `-h` | human-readable | Sizes as K, M, G instead of raw bytes |
| `-t` | time | Sort by modification time, newest first |
| `-S` | Size | Sort by file Size, largest first |
| `-r` | reverse | Reverse the sort order |
| `-R` | Recursive | List all subdirectories recursively |
| `-i` | inode | Show inode number before each entry |
| `-d` | directory | Show directory itself, not its contents |

**Common combinations:**

```bash
ls -lh      # standard detailed listing — daily use
ls -la      # include hidden dotfiles
ls -lht     # find recently modified files (time-sorted)
ls -lhS     # find largest files (size-sorted)
ls -lai     # show inode numbers — identify hard links
ls -ld /tmp # show /tmp itself, not what is inside it
```

**Hidden files (dotfiles):**

Files starting with `.` are hidden from plain `ls`. They are NOT
hidden for security — just to keep listings clean. Any user with
read permission can see them with `ls -la`:

```
~/.bashrc      shell configuration
~/.ssh/        SSH keys and config (important: permissions matter here)
~/.gitconfig   git configuration
.env           environment variables — never commit to git
.dockerignore  Docker build exclusions
```
These are not hidden for security — they are hidden to keep
directory listings clean. Any user with read permission can see them with `ls -la`.

### 1.3 File operations — mkdir, cp, mv, rm

**`mkdir` — make directory:**

```bash
mkdir mydir           # create one directory
mkdir -p a/b/c        # -p = parents — create full path including all
                      # intermediate parent directories
                      # no error if directory already exists
                      # essential in scripts — mkdir -p /etc/myapp
```

`-p` stands for **parents**. Without it, `mkdir a/b/c` fails if
`a/` does not already exist.

**`cp` — copy:**

```bash
cp file.txt backup.txt              # copy single file
cp -r /etc/nginx/ /tmp/nginx-bak/  # -r = recursive — copy directory
                                    # and all its contents
cp -p file.txt backup.txt          # -p = preserve — keep timestamps,
                                    # permissions, and ownership
                                    # without -p: new file gets current
                                    # time and your umask permissions
cp -a /src/ /dst/                  # -a = archive — equals -r + -p + -d
                                    # -d = preserve symlinks as symlinks
                                    # (without -d, cp follows symlinks
                                    # and copies the target content)
                                    # use -a for complete faithful copies
                                    # backups, migrations, staging
```

**`mv` — move or rename:**

```bash
mv old.txt new.txt           # rename in same directory
mv file.txt /tmp/            # move to different directory
mv app.conf app.conf.bak     # rename as backup before editing
                             # always do this before modifying configs
```

`mv` within the same filesystem is **atomic** — the rename happens
as a single operation. Moving across filesystems is copy + delete.

**`rm` — remove:**

```bash
rm file.txt             # remove single file
rm -r mydir/            # -r = recursive — remove directory and ALL
                        # contents: subdirectories, files, everything
rm -f file.txt          # -f = force — no error if file does not exist
rm -rf mydir/           # force + recursive — use with extreme care
```

**The most dangerous command in Linux:**
```bash
rm -rf /                       # deletes everything — DO NOT RUN
rm -rf /*                      # same effect
rm -rf /tmp/myapp/             # safe — specific path
```

Always double-check the path before `rm -rf`. A common safe practice:
```bash
ls -la /path/to/delete/        # verify what you are about to delete
rm -rf /path/to/delete/        # then delete
```

### 1.4 What is a directory entry?

Before understanding inodes, you need to understand what a
**directory entry** is — because this is fundamental to how
filenames, inodes, and links all connect.

A directory in Linux is itself a special file. Its contents are a
**table** that maps filenames to inode numbers. Each row in that
table is a directory entry:

```
Directory file contents (simplified):
  filename            inode number
  ──────────────────────────────────
  .                   12345        ← this directory itself
  ..                  12100        ← parent directory
  nginx.conf          67891        ← regular file
  sites-available     67892        ← subdirectory
  sites-enabled       67893        ← subdirectory
```

When you run `ls`, the kernel reads this table from the directory
file and looks up each inode number to get permissions, size, and
timestamps for display.

**Key operations explained by directory entries:**

```
Create file:    new inode allocated + new entry added to table
Delete file:    directory entry removed (inode freed only when
                link count reaches 0 and no process has file open)
Rename (mv):    directory entry's filename changed — same inode number
                (this is why rename is fast and atomic)
List (ls):      kernel reads the filename→inode table
```

**The critical point: the filename lives in the directory entry,
NOT in the inode itself.** This is why:
- Multiple names can point to the same inode (hard links)
- Renaming is instantaneous (just changes a table entry)
- An inode survives the deletion of any one name pointing to it

### 1.5 Inodes — the real identity of every file

Every file on a Linux filesystem has two parts:

```
1. The inode (index node) — stores ALL metadata about the file:
   - File type (regular, directory, symlink, device)
   - Permissions (rwxr-xr-x)
   - Owner UID and group GID
   - File size in bytes
   - Three timestamps (atime, mtime, ctime)
   - Link count (how many directory entries point here)
   - Pointers to the actual data blocks on disk
   - Does NOT contain: the filename

2. The directory entry — maps a filename to an inode number
   - Lives inside the directory file
   - "nginx.conf" → inode 127654
```

The filename you see is just a label in a table. The inode is the
real file. Understanding this explains:
- Hard links (multiple names, one inode)
- Why rename is instantaneous (just changes a table entry)
- Why a deleted file may still hold disk space (inode still alive)
- Inode exhaustion — a disk can be "full" with space remaining

**View inode numbers:**
```bash
ls -li /etc/hosts    # -i shows inode number as first column
stat /etc/hosts      # shows complete inode information
```

**Inode exhaustion — an important production scenario:**
A filesystem has a fixed number of inodes set at creation time.
If a process creates millions of tiny files (email servers, caches,
temp files), you can run out of inodes before running out of disk
space. `df -i` shows inode usage. `df -h` will show disk space
available but `df -i` shows 100% — no more files can be created
even though disk has space. We cover `df` fully in Demo 08.

### 1.6 File timestamps — atime, mtime, ctime

Linux tracks three separate timestamps per file. They measure
different things and are updated by different operations.
Confusing them is a very common mistake — including in interview
answers.

```
atime — Access time
        Updated when: file CONTENTS are read
        (cat, less, grep reading a file, a script reading it)
        NOT updated by: ls (listing does not read contents)
        Use case: find files not accessed in 30 days for cleanup

mtime — Modification time
        Updated when: file CONTENTS are changed
        (writing to file, truncating, appending)
        NOT updated by: chmod, chown, rename
        This is what ls -l shows by default
        Use case: find files changed recently, build systems,
                  deployment scripts

ctime — Change time (NOT creation time — a common mistake)
        Updated when: file METADATA changes
        (chmod, chown, rename/mv, any mtime change also updates ctime)
        CANNOT be set manually by users — kernel-controlled
        Use case: security auditing, detecting permission changes
```

**The common trap — ctime is NOT creation time:**

`ctime` stands for **change time** — the last time the inode
metadata changed. Linux does not traditionally store creation time
(called `btime` or `birth time`). Some modern filesystems (ext4,
xfs) do store birth time but it is not always exposed. `stat`
shows it as `Birth: -` if not available.

**Why does writing content also update ctime?**

When file contents change (write), two things happen:
1. The data blocks on disk change → `mtime` updates
2. The new `mtime` value must be written into the inode → the
   inode itself was modified → `ctime` updates

mtime changes always cause ctime to change, because the new mtime
value is stored inside the inode — and any inode modification
updates ctime.

**Why does `mv` change ctime if the filename is not in the inode?**

`mv` within the same filesystem modifies the directory entry (the
filename-to-inode table). Although the filename is not in the inode,
the rename operation still touches the inode — specifically the link
count is briefly adjusted and `ctime` is updated to record that a
rename occurred. The kernel records any structural reference change
in ctime.

**Operations and timestamps:**

```
Operation              atime   mtime   ctime
─────────────────────────────────────────────
Read file contents      ✓
Write/modify contents           ✓       ✓
chmod or chown                          ✓
Rename (mv)                             ✓
Create file             ✓       ✓       ✓
Hard link created                       ✓
touch (no flags)        ✓       ✓       ✓
```

**The ctime trap in interviews:**

`ctime` is called "change time", not "created time". Linux does
not traditionally store file creation time. `stat` shows it as
`Birth: -` on filesystems that do not support it. Modern ext4 and
xfs do store birth time (accessible via `stat`), but `ctime` is
not it.

**Note on atime and performance — what is `relatime`:**

By default, modern Linux mounts filesystems with the `relatime`
mount option. Without it, every file read would cause a disk write
to update atime — reading a directory with 10,000 files would
trigger 10,000 disk writes just to record "these were accessed".
This is significant I/O overhead.

`relatime` means: **atime is only updated if it is older than
mtime**. In practice:
```
File modified (mtime): Monday 10:00
Read Monday 11:00  → atime updated (atime was before mtime)
Read Monday 14:00  → atime NOT updated (atime is already after mtime)
Read Monday 16:00  → atime NOT updated
File modified Tuesday 09:00 (mtime advances)
Read Tuesday 10:00 → atime updated (atime is now before new mtime)
```

The result: atime still tracks "roughly when last accessed since
last modification" but avoids constant disk writes on every read.

Check your mount options:
```bash
cat /proc/mounts | grep " / " | grep -oE "(relatime|strictatime|noatime)"
```

`noatime` is more aggressive — never updates atime at all. Used
on high-performance or battery-sensitive systems where every disk
write matters.

### 1.7 Hard links and soft links

**Hard link — another name for the same inode:**

```bash
ln original.txt hardlink.txt     # ln = link, no -s flag = hard link
```

```
Directory table:
  original.txt  → inode 12345 → data blocks on disk
  hardlink.txt  → inode 12345 (same inode — same data)

ls -li shows:
  12345  -rw-r--r--  2  wadmin  wadmin  1024  original.txt
  12345  -rw-r--r--  2  wadmin  wadmin  1024  hardlink.txt
  ↑                  ↑
  same inode         link count = 2
```

Characteristics:
- Same inode number — verify with `ls -li`
- Same permissions, owner, timestamps — they are the same file
- Modifying one modifies both — same data blocks
- Deleting one does NOT delete the data — link count goes to 1
  Data only deleted when link count reaches 0
- Cannot cross filesystem boundaries
- Cannot link to directories (prevents circular loops)
- Link count in `ls -l` tells you how many hard links exist

**Soft link (symbolic link) — a file that stores a path:**

```bash
ln -s /path/to/original.txt symlink.txt    # -s flag = symbolic
ln -s /etc/nginx/nginx.conf nginx.conf     # common pattern
ln -s /usr/bin/python3.12 /usr/local/bin/python  # version alias
```

```
Directory table:
  original.txt  → inode 12345 → data blocks
  symlink.txt   → inode 99999 → stores the string "/path/to/original.txt"

ls -la shows:
  lrwxrwxrwx  1  wadmin  wadmin  12  symlink.txt -> original.txt
  ↑                               ↑
  l = symlink type                size = number of chars in target path
```

Characteristics:
- Different inode — it is a separate file containing a path string
- `l` at start of permissions in `ls -la`
- Can cross filesystem boundaries (stores a path, not an inode number)
- Can link to directories
- If original is deleted → symlink becomes a broken/dangling link
- The `->` arrow in `ls -la` shows what it points to
- Size shown = number of characters in the target path string
**Hard link vs soft link — when to use which:**

```
Use hard link when:
  - You want a file accessible from multiple locations
  - You need the data to survive deletion of the original name
  - Working within the same filesystem
  - Example: backup scripts that use hard links for efficiency

Use soft link when:
  - Pointing to a directory (required — hard links to dirs not allowed)
  - Crossing filesystem boundaries
  - Creating a stable name for a versioned binary
    /usr/bin/python -> python3.12 (easy to update to 3.13 later)
  - Making config available in multiple locations
    ln -s /etc/nginx/sites-available/myapp /etc/nginx/sites-enabled/
  - Example: kubectl version aliases, nginx site enabling
```

### 1.8 The nginx sites-available/sites-enabled pattern explained

nginx needs to know which website configurations are active.
The standard setup uses two directories:

```
/etc/nginx/sites-available/    — library of ALL config files stored here whether active or not
                                 nginx does NOT read this directly

/etc/nginx/sites-enabled/      — nginx reads this directory
                                 only symlinks live here
                                 each symlink points to a file in sites-available/
```

**Why symlinks instead of copies?**

If you copied `myapp.conf` from `sites-available/` to
`sites-enabled/`, you would have two files. Edit one — the other
is out of date. You would constantly need to sync them.

With a symlink: one file exists in `sites-available/`. The symlink
in `sites-enabled/` is just a pointer. nginx follows the pointer
and reads the original. Edit the file in `sites-available/` and
nginx sees the change immediately — there is only one file.

**Enable a site:**
```bash
ln -s /etc/nginx/sites-available/myapp.conf /etc/nginx/sites-enabled/
# Result: sites-enabled/myapp.conf -> ../sites-available/myapp.conf
```

**Disable a site — without losing the config:**
```bash
rm /etc/nginx/sites-enabled/myapp.conf
# Removes only the symlink pointer
# sites-available/myapp.conf is untouched — re-enable anytime
```

**Crossing filesystem boundaries in practice:**

If `sites-available/` and `sites-enabled/` were on different
filesystems (different partitions), a hard link would fail. A
symlink works regardless because it stores a path string, not an
inode reference.

**Making config available in multiple locations:**

The same pattern applies anywhere one file needs to appear in
multiple places:
```bash
# One config file, seen by two different tools
ln -s /etc/myapp/config.yaml /opt/legacy-tool/config.yaml

# One SSL certificate, used by nginx and postfix
ln -s /etc/letsencrypt/live/domain/cert.pem /etc/postfix/cert.pem
```

### 1.9 The `touch` command

`touch` has two purposes: create an empty file, or update timestamps.

```bash
touch newfile.txt      # create empty file if does not exist
                       # if file exists: update atime+mtime to now

touch -a file.txt      # -a = access — update atime only
touch -m file.txt      # -m = modification — update mtime only

touch -t 202601011200 file.txt   # -t = timestamp — set specific time
                                  # format: YYYYMMDDhhmm

touch -r ref.txt file.txt        # -r = reference — copy timestamps
                                  # from ref.txt to file.txt
```

**Brace expansion with touch (and any command):**

```bash
touch file{1..5}.txt   # creates file1.txt through file5.txt
```

This is **bash brace expansion** — not specific to `touch`. The
shell expands `{1..5}` before the command runs:

```bash
# The shell converts this:
touch file{1..5}.txt

# Into this before touch sees it:
touch file1.txt file2.txt file3.txt file4.txt file5.txt
```

Works with any command:
```bash
mkdir dir{a,b,c}           # creates dira, dirb, dirc
cp config{,.bak}           # copies config to config.bak
rm log{1..10}.txt          # removes log1.txt through log10.txt
```

Bash expansion is covered in full in Demo 12 (Bash Scripting).
Here it is shown so you can recognise the pattern.

---

## Part 2 — Hands-On Lab

### Commands used in this demo

| Command | Purpose |
|---|---|
| `pwd` | Print current directory |
| `cd` | Change directory |
| `ls -lhi` | Long + human sizes + inode numbers |
| `ls -lht` | Long + human sizes + time-sorted |
| `ls -la` | Long + hidden files |
| `mkdir -p` | Create directory tree including parents |
| `cp -a` | Archive copy — preserve all attributes + symlinks |
| `mv` | Move or rename |
| `rm -rf` | Remove recursively (verify path first) |
| `ln` | Create hard link |
| `ln -s` | Create soft link |
| `stat` | Show complete inode information |
| `touch` | Create file or update timestamps |

---

### Task 1 — Navigation and paths

```bash
# Where are you?
pwd

# cd shortcuts — run each and check pwd after
cd /etc && pwd
cd - && pwd        # back to previous
cd .. && pwd       # up one level
cd ~ && pwd        # home

# Absolute vs relative — same result
cd /var/log && pwd
cd ../../etc && pwd   # relative: up two levels, into etc
```
What you are building: muscle memory for paths. The `cd -` toggle
is one of the most useful shortcuts — bouncing between two directories
during an incident.

---

### Task 2 — ls flags in practice

```bash
mkdir -p ~/demo02/testdir
cd ~/demo02

# Create some test files with content
echo "config content" > app.conf
echo "log line 1" > app.log
echo "script content" > deploy.sh
mkdir -p subdir/nested
touch .hidden-file

# Now explore ls flags
echo "=== ls (basic) ===" && ls
echo "=== ls -la (long + hidden) ===" && ls -la
echo "=== ls -lh (human sizes) ===" && ls -lh
echo "=== ls -lht (time sorted) ===" && ls -lht
echo "=== ls -lhS (size sorted) ===" && ls -lhS
echo "=== ls -lai (with inodes) ===" && ls -lai
```

**Reading `ls -lai`:**
```
ls -lai output example:
  inode  perms      links  owner   group  size   date    name
  12345  -rw-r--r--  1    wadmin  wadmin  14    Jun  7  app.conf
  ↑                  ↑
  inode number       hard link count = 1 (only one name for this file)
  unique per file    directories always ≥ 2 (. and .. count as links)
```

Directories always show link count ≥ 2 because `.` (dot entry
inside itself) and the entry in the parent directory both count
as hard links.

**Paste your output.**

---

### Task 3 — stat and timestamps

```bash
cd ~/demo02

stat app.conf    # complete inode view

echo "new line" >> app.conf
stat app.conf    # mtime changed, ctime changed, atime unchanged

cat app.conf
stat app.conf    # atime may change (depends on relatime)

chmod 644 app.conf
stat app.conf    # only ctime changed — permissions = metadata

mv app.conf application.conf
stat application.conf   # ctime changed (rename touched inode)
                        # mtime unchanged (contents untouched)
mv application.conf app.conf  # rename back

# Check your mount options
cat /proc/mounts | grep " / " | grep -oE "(relatime|strictatime|noatime)"
```

**Paste stat output after each operation. Note which timestamps
change and which do not.**

---

### Task 4 — Hard links and inodes

```bash
cd ~/demo02

# Create a file and check its inode
echo "important data" > original.txt
ls -li original.txt
stat original.txt    # note the inode number and link count = 1

# Create a hard link
ln original.txt hardlink.txt

# Both show same inode number, link count = 2
ls -li original.txt hardlink.txt

# Prove they are the same file
echo "added via original" >> original.txt
cat hardlink.txt     # see the change — same data blocks

echo "added via hardlink" >> hardlink.txt
cat original.txt     # see this change too

# Delete original — hardlink still works
rm original.txt
cat hardlink.txt     # still readable — inode still has link count 1
ls -li hardlink.txt  # link count = 1 now

# Create hard link across directories
mkdir -p /tmp/demo02-links
ln hardlink.txt /tmp/demo02-links/another-name.txt
ls -li hardlink.txt /tmp/demo02-links/another-name.txt
# Same inode number — same file
```

**Paste the `ls -li` output showing both files with same inode
number. This proves they are the same file.**

---

### Task 5 — Soft links

```bash
cd ~/demo02

# Create a soft link
echo "target content" > target.txt
ln -s target.txt softlink.txt

# Observe the difference from hard link
ls -lai target.txt softlink.txt
# softlink shows: lrwxrwxrwx and -> target.txt
# DIFFERENT inode numbers
# size of softlink = number of chars in "target.txt" = 10

stat softlink.txt    # inode type: symbolic link

# Soft link follows the target
cat softlink.txt     # reads target.txt content

# What happens when target is deleted — dangling symlink
rm target.txt
cat softlink.txt     # fails: No such file or directory
ls -la softlink.txt  # still shows link but target is gone

# This is a dangling/broken symlink — common production issue
# Fix: recreate the target or update the symlink
echo "recreated" > target.txt  # fix by recreating target

# Real DevOps pattern: version alias
ln -s /usr/bin/python3 ~/demo02/python
ls -la ~/demo02/python   # shows -> /usr/bin/python3
~/demo02/python --version


# stat on symlink vs stat -L
stat softlink.txt          # symlink's OWN metadata — inode, timestamps
stat -L softlink.txt       # follows symlink — shows target.txt metadata
# Compare: different inodes, different timestamps, different sizes
```

**What to observe:**
```
stat symlink:      symlink's own inode and timestamps
stat -L symlink:   target's inode and timestamps
```

**Paste the `ls -lai` output showing different inodes.**

---

### Task 6  — cp with preserve and archive flags:**

```bash
cd ~/demo02

# Create a target file and a symlink to use for testing
echo "original content" > source.txt
ln -s source.txt source-link.txt
ls -lai source.txt source-link.txt   # note different inodes


# cp without -p — new timestamps, your umask permissions
cp source.txt copy-plain.txt
stat source.txt copy-plain.txt
# copy-plain.txt has NOW as mtime — original timestamps lost

# cp -p — preserve timestamps, permissions, ownership
cp -p source.txt copy-preserved.txt
stat source.txt copy-preserved.txt
# copy-preserved.txt has SAME mtime as source.txt

# cp with symlink — default behaviour FOLLOWS the symlink
cp source-link.txt copy-followed.txt
ls -lai copy-followed.txt   # regular file — symlink was followed
stat copy-followed.txt      # contains source.txt content

# Without -a: cp follows the symlink, copies target content
cp source-link.txt copy-followed.txt
ls -lai copy-followed.txt   # regular file — symlink was followed
file copy-followed.txt


# cp -a — archive mode preserves the symlink AS a symlink
cp -a source-link.txt copy-archived.txt
ls -lai copy-archived.txt   # still a symlink — not followed
stat copy-archived.txt      # type: symbolic link

# cp -a on a directory — faithful complete copy
mkdir -p testdir/sub
echo "file1" > testdir/file1.txt
ln -s ../file1.txt testdir/sub/link-to-file1
ls -lai testdir/sub/

cp -r testdir/ copy-r/      # -r follows symlinks
ls -lai copy-r/sub/         # link-to-file1 is now a regular file

cp -a testdir/ copy-a/      # -a preserves symlinks
ls -lai copy-a/sub/         # link-to-file1 is still a symlink
```

**What to observe:**
```
cp plain:       new timestamps, symlinks followed and content copied
cp -p:          original timestamps preserved, symlinks still followed
cp -a:          timestamps preserved + symlinks preserved as symlinks
                use -a for backups and migrations — faithful copy
```

### Task 7 — touch and timestamps

```bash
cd ~/demo02

# Create multiple files at once
touch file{1..5}.txt
ls -lht *.txt   # all have same timestamp

# Create file with specific timestamp
touch -t 202601011200 file1.txt
ls -lht *.txt   # file1.txt now appears with Jan 1 timestamp
stat file1.txt  # verify atime and mtime updated, ctime = now

# Force a file to appear modified without changing content
touch file2.txt
ls -lht *.txt   # file2.txt is now newest

# Set timestamps to match another file
touch -r file2.txt file3.txt
stat file2.txt file3.txt  # same atime and mtime, different ctime
```

---




## Part 3 — Break-Fix Scenario

### Scenario — NovaTech config mystery

> *"I edited the nginx config three days ago. When I run `stat` on
> `/etc/nginx/sites-enabled/app.conf` it shows mtime from last week
> — unchanged. Also I deleted a large backup file last week but
> `df -h` still shows the same disk usage."*

You investigate:

```bash
ls -lai /etc/nginx/sites-enabled/app.conf
ls -lai /etc/nginx/sites-available/app.conf

stat /etc/nginx/sites-enabled/app.conf
stat /etc/nginx/sites-available/app.conf

lsof | grep "app.conf.bak"
# nginx  1234  root  4r  REG  app.conf.bak (deleted)
```

**Your task:**

1. The developer ran `stat sites-enabled/app.conf` and saw last
   week's mtime. `sites-enabled/app.conf` is a symlink. What does
   `stat` show when run on a symlink — the symlink's own metadata
   or the target's? What should the developer have run instead to
   check whether their edit was saved?

2. When the developer edited through `sites-enabled/app.conf`,
   which file's mtime actually changed?

3. `df -h` shows no change after deleting `app.conf.bak`. Why
   does `df` still count it? Why does `du` not count it? What does
   the `lsof` output tell you?

4. What is the fix for the disk space issue?

<details>
<summary>Resolution — open after attempting</summary>

**1. What stat shows on a symlink:**

`stat` on a symlink shows the **symlink's own metadata** — its own
inode, its own timestamps, its own size. It does NOT follow the
symlink by default. This is confirmed by testing:

```bash
stat softlink.txt
# File: softlink.txt -> target.txt
# Inode: 9739          ← symlink's OWN inode
# Size: 10             ← length of "target.txt" path string
# Modify: <when symlink was created>   ← symlink's OWN mtime
```

The symlink's own mtime records when the symlink file itself was
last changed — i.e. when it was created or re-pointed. It does NOT
change when you edit through it. That is why the developer saw last
week's mtime — that is when the symlink was created, and it has not
changed since.

```
sites-enabled/app.conf  (symlink, inode A)
  own mtime: last week  ← when this symlink was created
  unchanged when you write through it

sites-available/app.conf  (target, inode B)
  own mtime: 3 days ago ← when contents last changed
  this is what confirms the edit was saved
```

**What the developer should have run:**

```bash
# Check the TARGET file directly — not the symlink
stat /etc/nginx/sites-available/app.conf
# mtime 3 days ago → edit was saved correctly
# mtime last week  → edit was NOT saved

# Or use -L to make stat follow the symlink
stat -L /etc/nginx/sites-enabled/app.conf
# -L = dereference — follows symlink, shows target metadata
```

**2. Which file's mtime changed after the edit:**

When the developer edited through `sites-enabled/app.conf`, the
editor followed the symlink and wrote to the actual target:
`sites-available/app.conf`. The target's mtime updated. The
symlink's own mtime did not change — the symlink file itself
(the path string it stores) was not modified.

**3. Why `df` counts the deleted file but `du` does not:**

`du` walks directory entries. When `app.conf.bak` was deleted,
its directory entry was removed — `du` cannot see it and does
not count its blocks.

`df` counts at the filesystem level — all allocated inodes and
their blocks, regardless of whether a directory entry exists.
nginx (PID 1234) still has the file open, so the inode is still
alive and its disk blocks remain allocated. `df` counts them.

```
For du:  directory entry removed → file invisible → not counted
For df:  inode alive (held open by nginx) → blocks allocated → counted
```

A file is truly deleted only when BOTH conditions are met:
- Link count = 0 (all directory entries removed)
- No process has it open (no open file descriptors)

Until both are true the kernel keeps the inode and blocks
allocated. `lsof | grep deleted` finds exactly these files —
gone at the directory level, still alive at the inode level.

This is covered in full in Demo 08 (Storage & Archives).

**4. Fix:**

```bash
systemctl restart nginx        # closes all file descriptors
df -h /etc                     # verify space freed
lsof | grep deleted            # confirm no more held-open files
```

</details>

---

## Interview Prep

**Q1. What is an inode? What information does it NOT contain?**

An inode is a data structure on the filesystem that stores all
metadata about a file: permissions, owner, group, size, timestamps
(atime/mtime/ctime), link count, and pointers to the data blocks
on disk. The inode does NOT contain the filename. The filename is
stored in the directory entry, which maps a name to an inode number.
This separation is what makes hard links possible — multiple names
can point to the same inode.

---

**Q2. What is the difference between mtime and ctime?
Give a concrete example where they differ.**

`mtime` (modification time) updates when the file contents change.
`ctime` (change time) updates when any inode metadata changes —
including permissions, ownership, rename, or mtime itself. They
differ when you run `chmod 755 script.sh` — ctime updates to now
because you changed metadata, but mtime stays unchanged because
you did not touch the file contents. A security audit tool
monitoring ctime detects this permission change even though the
file data was not modified.

---

**Q3. You delete a 10GB log file but `df -h` still shows the
same disk usage. What happened and how do you fix it?**

A process has the file open. Linux only frees disk blocks when
the link count is zero AND no process holds the file open. The
file's directory entry was removed (invisible to `ls` and `du`)
but the inode and data blocks remain until the process closes it.

Diagnose: `lsof | grep deleted` — shows the process and PID
holding the deleted file open.

Fix: restart the process (`systemctl restart servicename`) to
close all its file descriptors. The kernel then frees the blocks.

---

**Q4. What happens to a soft link when its target file is deleted?
What about a hard link?**

Soft link: becomes dangling/broken. The symlink file still exists
but reading it fails with "No such file or directory". The symlink
stores a path string — if nothing exists at that path, it is broken.

Hard link: nothing happens. The data survives. The deleted
"original" was just one directory entry pointing to the inode.
The hard link is another directory entry pointing to the same inode.
Link count goes from 2 to 1. Data persists until all hard links
are removed AND no process has it open.

---

## Key Takeaways

1. **`cd -` toggles between the last two directories — use it constantly during incidents instead of retyping paths.** Bouncing between `/var/log` and `/etc/nginx` during a debug session takes one keystroke, not a full path.

2. **An inode stores everything about a file except its name — permissions, owner, timestamps, size, data block pointers, and link count.** The filename lives in the directory entry (the table inside the directory file). This separation is what makes hard links and instant renames possible.

3. **`mtime` = file contents changed. `ctime` = inode metadata changed. `ctime` is NOT creation time — this is one of the most common wrong interview answers.** Writing content updates mtime and ctime together (because mtime is stored inside the inode — any inode write updates ctime).

4. **Hard links share one inode — same inode number, same data blocks, same permissions.** Deleting one directory entry reduces link count by 1. Data is freed only when link count reaches 0 AND no process has the file open.

5. **Soft links store a path string and have their own inode — different from the target.** Delete the target and the symlink becomes dangling. Can cross filesystem boundaries. Can link to directories. `stat` on a symlink shows the symlink's own metadata — use `stat -L` to follow it and show the target's metadata.

6. **A deleted file whose inode is still held open by a process is invisible to `du` but still counted by `df`.** `lsof | grep deleted` finds these. Restart the process to release the file descriptor and free the blocks.

7. **`cp -p` preserves timestamps, permissions, and ownership — without it the copy gets the current time and your umask permissions.** Use `-p` for backups. Use `-a` (= `-r` + `-p` + preserve symlinks) for complete directory migrations — it is the correct flag, not `-r` alone.

---

## Appendix A — Quick Reference

### ls flag combinations

| Command | Use case |
|---|---|
| `ls -lh` | Standard detailed listing |
| `ls -la` | Include hidden dotfiles |
| `ls -lht` | Find recently modified files |
| `ls -lhS` | Find largest files |
| `ls -lai` | Show inode numbers — identify hard links |
| `ls -ld /dir` | Show directory itself, not contents |

### Path notation

| Path | Meaning |
|---|---|
| `/` | Root directory |
| `~` | Your home directory |
| `.` | Current directory |
| `..` | Parent directory |
| `-` | Previous directory (with cd) |

### Timestamp reference

| Timestamp | Full name | Updated by | View with |
|---|---|---|---|
| `atime` | Access time | Reading file contents | `stat`, `ls -lu` |
| `mtime` | Modification time | Changing file contents | `stat`, `ls -l` |
| `ctime` | Change time | Any inode change | `stat`, `ls -lc` |
| `birth` | Birth/creation time | File creation | `stat` (if supported) |

### cp flag reference

| Flag | Meaning | Use when |
|---|---|---|
| `-r` | Recursive | Copying directories |
| `-p` | Preserve timestamps, permissions, owner | Backups, migrations |
| `-a` | Archive = -r + -p + preserve symlinks | Complete faithful copies |

---

## Appendix B — Anki Cards

**`02-navigation-files-anki.csv`:**

```
#deck:Linux Mastery::Module 1 - Filesystem & Navigation::Demo 02 - Navigation & Files
#separator:Comma
#columns:Front,Back,Tags

"ls -l shows link count 3 for a directory. What does 3 mean?","Every directory has minimum 2 hard links: the entry in its parent, and its own . (dot) entry. Each subdirectory adds 1 more via its .. (dotdot) entry. Link count 3 means exactly one subdirectory exists. Use ls -lai to see inode numbers.","demo02,inodes,links"

"You delete a 5GB file but df -h shows no change. What command diagnoses this and what is the fix?","lsof | grep deleted — shows files with directory entry removed but still held open by a process. The kernel keeps inode and data blocks allocated until the process closes the file. Fix: restart the process holding it. The kernel then frees the blocks.","demo02,inodes,disk,lsof"

"What is the difference between mtime and ctime? Why does writing content update both?","mtime updates when file contents change. ctime updates when any inode metadata changes. Writing content changes mtime — but the new mtime value must be stored in the inode — so the inode is modified — so ctime also updates. ctime always updates when mtime updates, but ctime can update without mtime (chmod, chown).","demo02,timestamps"

"ls -lai shows two files with the same inode number. What are they? What happens if you delete one?","Hard links — two directory entries pointing to the same inode and data blocks. Created with: ln original.txt hardlink.txt. Deleting one reduces link count by 1. Data survives until link count = 0 AND no process has file open.","demo02,hardlinks,inodes"

"ls -la shows: lrwxrwxrwx 1 root root 26 May 29 nginx.conf -> /etc/nginx/nginx.conf. What is the 26?","26 is the number of characters in the target path string /etc/nginx/nginx.conf. For symlinks, the size column shows the byte length of the stored path string — not data size. lrwxrwxrwx: l = symbolic link type.","demo02,softlinks,ls"

"chmod on a file updates ctime but not mtime. mv updates ctime but not mtime. Why?","chmod changes permissions — inode metadata. Any inode change updates ctime. mv renames the directory entry — the inode is touched during the rename operation. ctime records any inode change. mtime only changes when file contents change. Neither chmod nor mv touches file data.","demo02,timestamps,ctime"

"Explain the nginx sites-available/sites-enabled symlink pattern. Why symlinks instead of copies?","sites-available/ holds all config files. sites-enabled/ holds symlinks pointing to active configs in sites-available/. Enable: ln -s ../sites-available/app.conf sites-enabled/. Disable: rm sites-enabled/app.conf (symlink only, config preserved). One file exists — edits are immediately seen. Copies would go out of sync.","demo02,softlinks,nginx"

"What is a directory entry? What information does it contain?","A directory in Linux is a special file whose contents are a table mapping filenames to inode numbers. Each row is a directory entry: filename → inode number. The filename lives here, not in the inode. Create a file: new inode + new entry. Delete: entry removed (inode freed when count=0 and no open fd). Rename: entry's filename changed, same inode.","demo02,inodes,directory-entry"

"You run stat on a symlink. Does it show the symlink's own metadata or the target's? How do you make stat follow the symlink?","stat on a symlink shows the symlink's OWN metadata — its own inode, its own timestamps, its own size (= path string length). It does NOT follow the symlink by default. To follow the symlink and show the target's metadata: stat -L symlink.txt. The -L flag means dereference (follow the link). This distinction matters when checking whether an edit through a symlink was saved — always stat the target directly, not the symlink.","demo02,symlink,stat"

"What is the difference between cp, cp -p, and cp -a when copying a file that is a symlink?","cp (plain): follows the symlink and copies the target content as a regular file. New timestamps from now, your umask permissions. cp -p (preserve): follows the symlink, copies target content, but preserves original timestamps, permissions and ownership. Still a regular file. cp -a (archive): preserves the symlink AS a symlink — does not follow it. Also preserves timestamps and permissions. Use cp -a for complete faithful directory copies — backups, migrations, staging. cp -a = -r + -p + -d (preserve symlinks).","demo02,cp,symlink,flags"

"touch file{1..5}.txt creates 5 files. Is this a touch feature or something else? What other commands can use the same syntax?","This is bash brace expansion — a shell feature, not specific to touch. The shell expands {1..5} before touch even runs, converting it to: touch file1.txt file2.txt file3.txt file4.txt file5.txt. Works with any command: mkdir dir{a,b,c}, cp config{,.bak}, rm log{1..10}.txt. Covered fully in Demo 12 (Bash Scripting).","demo02,touch,bash,brace-expansion"

"You try to create a hard link to a directory: ln /etc/nginx/ /tmp/nginx-link. It fails. Why? What should you use instead?","Hard links to directories are not allowed — the kernel explicitly prevents it to avoid circular directory loops which would break filesystem traversal. Error: hard link not allowed for directory. Use a soft link instead: ln -s /etc/nginx/ /tmp/nginx-link. Soft links can point to directories and are the correct tool for directory aliasing.","demo02,hardlinks,directories"
```

---

## Appendix C — Quiz

**`02-navigation-files-quiz.md`:**

````markdown
# Quiz — Demo 02: Navigation, File Operations & Inodes

> One correct answer per question unless stated otherwise.
> Target: 80% or above before moving to Demo 03.

---

**Q1.** `ls -li` shows two files with inode 98765 and link count 2.
You delete one. What happens to the data?

- A) Data deleted immediately — both names reference the same data
- B) Data survives — link count drops to 1, freed only when count
  reaches 0 AND no process has file open
- C) Data moved to `/tmp` automatically
- D) Remaining file becomes read-only

<details>
<summary>Answer</summary>

**B** — Hard link = directory entry pointing to an inode. Deleting one entry reduces link count by 1. The inode and data blocks are freed only when link count = 0 AND no open file descriptor holds the file. Link count 2 → 1 means data survives completely.

Trap: A is wrong — the two names are separate directory entries, not copies of data. Deleting one name does not touch the inode or blocks.

</details>

---

**Q2.** You run `chmod 755` on a script. Which timestamps change?

- A) `mtime` — you modified the file
- B) `atime` — you accessed it
- C) `ctime` — metadata changed, contents unchanged
- D) All three

<details>
<summary>Answer</summary>

**C** — `chmod` changes permissions, which is inode metadata. Any inode change updates `ctime`. `mtime` only changes when file contents change. `atime` only changes when file contents are read. `chmod` does neither.

Trap: D is wrong — only `ctime` changes. A is a common mistake — "modified" in `chmod` refers to permission modification, but `mtime` only tracks content modification.

</details>

---

**Q3.** Soft link `sites-enabled/app` points to `sites-available/app.conf`.
You delete `sites-available/app.conf`. What happens to the symlink?

- A) Symlink is deleted automatically along with the target
- B) Symlink becomes dangling — it still exists but reading it fails with "No such file or directory"
- C) Symlink now points to `/dev/null`
- D) Symlink is automatically converted to a hard link

<details>
<summary>Answer</summary>

**B** — Soft links store a path string. When the path no longer resolves to anything, the link is broken (dangling). The symlink file itself still exists — `ls -la` still shows it — but any attempt to read through it fails.

Trap: A is wrong — the symlink is a separate file with its own inode. Deleting the target has no effect on the symlink file's existence.

</details>

---

**Q4.** `df -h` shows `/var` at 95% full. `du -sh /var/*` accounts for only 60%.
What most likely explains the 35% gap?

- A) `df` counts reserved filesystem blocks (~5% by default)
- B) `du` cannot count `/proc` files in `/var`
- C) Deleted files held open by running processes — `df` counts allocated inode blocks, `du` cannot see deleted directory entries
- D) `/var` uses transparent compression that `du` misreports

<details>
<summary>Answer</summary>

**C** — When a file is deleted (`rm`), its directory entry is removed — `du` walks directory entries and cannot see it. But if a process still has the file open, the kernel keeps the inode and data blocks allocated — `df` counts these. Diagnose with `lsof | grep deleted`. Fix by restarting the process holding the file.

Trap: A (reserved blocks) is real but only accounts for ~5%, not 35%. That makes A a plausible distractor but cannot explain this gap.

</details>

---

**Q5.** What does `stat` show that `ls -l` does not?

- A) File contents in hex
- B) All three timestamps (`atime`, `mtime`, `ctime`), inode number, block count, device number, and birth time if supported by the filesystem
- C) The MD5 checksum of the file
- D) The username of the last person to access the file

<details>
<summary>Answer</summary>

**B** — `ls -l` shows `mtime` only by default. `stat` shows all three timestamps at once (`atime`, `mtime`, `ctime`) plus the full inode metadata: inode number, blocks allocated, device, permissions in octal, and birth time when the filesystem supports it.

</details>

---

**Q6.** You want to make `python3.12` available as `python` in `/usr/local/bin/`.
Which command is correct?

- A) `cp /usr/bin/python3.12 /usr/local/bin/python`
- B) `ln /usr/bin/python3.12 /usr/local/bin/python`
- C) `ln -s /usr/bin/python3.12 /usr/local/bin/python`
- D) `mv /usr/bin/python3.12 /usr/local/bin/python`

<details>
<summary>Answer</summary>

**C** — Use a soft link for three reasons: (1) `/usr/bin` and `/usr/local/bin` may be on different filesystems — hard links cannot cross filesystem boundaries. (2) When Python upgrades to 3.13, update one symlink rather than re-copying. (3) `ls -la` shows the `->` arrow making the alias obvious to anyone reading the directory.

Trap: A creates a copy that goes stale on upgrade. D is dangerous — moves the original binary out of `/usr/bin`, breaking the package manager's record of where it installed it.

</details>

---

**Q7.** You create `file.txt`, then run `ln file.txt link1.txt` and `ln -s file.txt link2.txt`.
Then you delete `file.txt`. What happens to `link1.txt` and `link2.txt`?

- A) Both become inaccessible — deleting the original destroys all references
- B) `link1.txt` works normally; `link2.txt` becomes dangling (exists but unreadable)
- C) Both work — Linux automatically keeps a copy for all linked names
- D) `link2.txt` works; `link1.txt` is deleted along with the original

<details>
<summary>Answer</summary>

**B** — Hard link (`link1.txt`): shares the same inode as `file.txt`. `rm file.txt` removes that one directory entry, reducing link count from 2 to 1. The inode and data survive. `link1.txt` reads normally.

Soft link (`link2.txt`): stores the path string `"file.txt"`. That path no longer resolves — the symlink is dangling. It exists as a file but reading through it fails.

Trap: D reverses the answer — hard links survive target deletion, soft links do not.

</details>

---

| Score | Action |
|---|---|
| 7/7 | Import Anki cards, move to Demo 03 |
| 6/7 | Review wrong answer, proceed |
| 5/7 | Re-read relevant section, retry those questions |
| Below 5/7 | Re-read full demo and redo walkthrough before proceeding |
````