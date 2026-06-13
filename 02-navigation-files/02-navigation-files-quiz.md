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
