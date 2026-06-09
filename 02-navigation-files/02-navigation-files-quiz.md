# Quiz — Demo 02: Navigation, File Operations & Inodes

> One correct answer per question. Target: 80% before Demo 03.

---

**Q1.** `ls -li` shows two files with inode 98765 and link count 2.
You delete one. What happens to the data?

- A) Data deleted immediately
- B) Data survives — link count drops to 1, freed only when count
  reaches 0 AND no process has the file open
- C) Data moved to `/tmp`
- D) Remaining file becomes read-only

<details>
<summary>Answer</summary>

**B** — A hard link is a directory entry pointing to an inode.
Deleting one entry reduces link count. Inode and data are freed
only when count = 0 AND no open file descriptor exists.

</details>

---

**Q2.** You run `chmod 755` on a script. Which timestamps change?

- A) mtime — you modified the file
- B) atime — you accessed it
- C) ctime — metadata changed, contents unchanged
- D) All three

<details>
<summary>Answer</summary>

**C** — `chmod` changes permissions which is inode metadata. ctime
updates for any inode change. mtime only changes when file contents
change. atime only when file contents are read.

</details>

---

**Q3.** Soft link `sites-enabled/app` → `sites-available/app.conf`.
You delete `sites-available/app.conf`. What happens to the symlink?

- A) Symlink deleted automatically
- B) Symlink becomes dangling — exists but reading it fails
- C) Symlink now points to `/dev/null`
- D) Converted to a hard link automatically

<details>
<summary>Answer</summary>

**B** — Soft links store a path string. When the target path no
longer resolves, the link is broken/dangling. `ls -la` still shows
it but `cat` fails: No such file or directory.

</details>

---

**Q4.** `df -h` shows `/var` at 95% full. `du -sh /var/*` totals 60%.
What most likely explains the gap?

- A) `df` counts reserved filesystem blocks that `du` ignores
- B) `du` cannot count `/proc` files in `/var`
- C) Deleted files held open by running processes — `df` counts
  allocated blocks, `du` cannot see deleted files
- D) `/var` uses compression that `du` misreports

<details>
<summary>Answer</summary>

**C** — Deleted-but-held-open inode scenario. `lsof | grep deleted`
finds them. Restart the process holding the file to free the space.
Trap: A (reserved blocks ~5%) cannot explain a 35% gap.

</details>

---

**Q5.** What does `stat` show that `ls -l` does not?

- A) File contents
- B) All three timestamps (atime, mtime, ctime), inode number,
  block count, device number, and birth time if available
- C) File checksum
- D) Username of last accessor

<details>
<summary>Answer</summary>

**B** — `ls -l` shows mtime only by default. `stat` shows all three
timestamps at once plus complete inode metadata: inode number,
link count, block count, device, and birth time if the filesystem
supports it.

</details>

---

**Q6.** A developer runs `stat /etc/nginx/sites-enabled/app.conf`
and sees mtime from last week. They edited the config three days
ago through this symlink. What went wrong and what should they
have run instead?

- A) `stat` follows symlinks — if mtime is last week the edit was lost
- B) `stat` shows the symlink's OWN metadata by default. The symlink's
  mtime records when the symlink itself was created, not when content
  was edited through it. Run `stat -L` or stat the target directly.
- C) The editor did not save the file correctly
- D) mtime on a symlink always shows the creation date

<details>
<summary>Answer</summary>

**B** — `stat` on a symlink shows the symlink's own inode and
timestamps — it does NOT follow the symlink by default. The symlink's
mtime = when the symlink file was created or re-pointed. It does not
change when you write through it to the target.

To check whether the edit was saved:
```bash
stat /etc/nginx/sites-available/app.conf   # stat the target directly
stat -L /etc/nginx/sites-enabled/app.conf  # -L = dereference symlink
```
The nginx sites-enabled/sites-available pattern makes this confusion
very common in production.

</details>

---

**Q7.** Make `python3.12` available as the command `python`
in `/usr/local/bin/`. Which command is correct and why?

- A) `cp /usr/bin/python3.12 /usr/local/bin/python`
- B) `ln /usr/bin/python3.12 /usr/local/bin/python`
- C) `ln -s /usr/bin/python3.12 /usr/local/bin/python`
- D) `mv /usr/bin/python3.12 /usr/local/bin/python`

<details>
<summary>Answer</summary>

**C** — Use a soft link. Reasons: (1) `/usr/bin` and `/usr/local/bin`
may be on different filesystems — hard links cannot cross filesystem
boundaries. (2) When Python upgrades to 3.13, update one symlink.
(3) `ls -la` shows the `->` arrow making the alias relationship visible.
Trap: A works but wastes space and goes stale on upgrade.
D is dangerous — moves the original, breaking the package manager.

</details>

---

**Q8.** You create `file.txt`. Then run:
`ln file.txt link1.txt` and `ln -s file.txt link2.txt`.
Then: `rm file.txt`. What happens to `link1.txt` and `link2.txt`?

- A) Both inaccessible
- B) `link1.txt` works normally, `link2.txt` becomes dangling
- C) Both work — Linux keeps copies for all links
- D) `link2.txt` works, `link1.txt` is deleted with the original

<details>
<summary>Answer</summary>

**B** — Hard link (`link1.txt`): shares inode with `file.txt`. `rm`
removes that directory entry, link count goes to 1. Inode and data
survive — `link1.txt` is fully accessible.
Soft link (`link2.txt`): stores the path string `"file.txt"` — that
path no longer resolves. Dangling — exists but unreadable.

</details>

---

**Q9.** You run `cp source-link.txt copy.txt` where `source-link.txt`
is a symlink pointing to `source.txt`. What does `copy.txt` contain?

- A) `copy.txt` is a symlink pointing to `source.txt`
- B) `copy.txt` is a regular file containing the content of
  `source.txt` — `cp` follows the symlink by default
- C) `cp` fails — you cannot copy a symlink
- D) `copy.txt` is a symlink pointing to `source-link.txt`

<details>
<summary>Answer</summary>

**B** — `cp` follows symlinks by default and copies the target content
as a new regular file. The symlink is not preserved.
To preserve the symlink as a symlink:
```bash
cp -a source-link.txt copy.txt
```
`cp -a` = archive mode = `-r` + `-p` + `-d` (preserve symlinks).

</details>

---

Score guide:

| Score | Action |
|---|---|
| 9/9 | Import Anki cards, move to Demo 03 |
| 8/9 | Review wrong answer, proceed |
| 7/9 | Re-read relevant section, retry those questions |
| Below 7/9 | Re-read Demo 02 before proceeding |
