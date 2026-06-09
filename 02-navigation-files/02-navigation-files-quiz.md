# Quiz — Demo 02: Navigation, File Operations & Inodes

> One correct answer per question. Target: 80% before Demo 03.

---

Q1. ls -li shows two files with inode 98765 and link count 2.
    You delete one. What happens to the data?

A) Data deleted immediately
B) Data survives — link count drops to 1, freed only when count
   reaches 0 AND no process has file open
C) Data moved to /tmp
D) Remaining file becomes read-only

[Answer: B — Hard link = directory entry pointing to inode.
Deleting one entry reduces link count. Inode and data freed only
when count = 0 AND no open file descriptor.]

---

Q2. You run chmod 755 on a script. Which timestamps change?

A) mtime — you modified the file
B) atime — you accessed it
C) ctime — metadata changed, contents unchanged
D) All three

[Answer: C — chmod changes permissions (inode metadata). ctime
updates for any inode change. mtime only changes when contents
change. atime only when contents are read.]

---

Q3. Soft link sites-enabled/app → sites-available/app.conf.
    You delete sites-available/app.conf. What happens to the link?

A) Symlink deleted automatically
B) Symlink becomes dangling — exists but reading it fails
C) Symlink now points to /dev/null
D) Converted to hard link

[Answer: B — Soft links store a path string. When the target path
no longer resolves, the link is broken. ls -la still shows it
but cat fails: No such file or directory.]

---

Q4. df -h shows /var at 95% full. du -sh /var/* shows 60%.
    What most likely explains the gap?

A) df counts reserved filesystem blocks
B) du cannot count /proc files in /var
C) Deleted files held open by running processes — df counts
   allocated blocks, du cannot see deleted files
D) /var uses compression du misreports

[Answer: C — Deleted-but-held-open inode scenario. lsof | grep
deleted to find. Restart the process to free space.
Trap: A (reserved blocks ~5%) cannot explain a 35% gap.]

---

Q5. What does stat show that ls -l does not?

A) File contents
B) All three timestamps (atime, mtime, ctime), inode number,
   block count, device number, and birth time if available
C) File checksum
D) Username of last accessor

[Answer: B — ls -l shows mtime only by default. stat shows
all three timestamps at once plus complete inode metadata.]

---

Q5b. A developer runs stat /etc/nginx/sites-enabled/app.conf and
     sees mtime from last week. They edited the config three days
     ago through this symlink. What went wrong and what should
     they have run?

A) stat follows symlinks — if mtime is last week the edit was lost
B) stat shows the symlink's OWN metadata by default. The symlink's
   mtime records when the symlink itself was created, not when
   content was edited through it. Run stat -L or stat the target
   directly: stat /etc/nginx/sites-available/app.conf
C) The editor did not save the file correctly
D) mtime on a symlink always shows the creation date

[Answer: B — stat on a symlink shows the symlink's own inode and
timestamps. The symlink's mtime = when the symlink file was created
or re-pointed. It does not change when you write through it to the
target. To check whether the edit was saved, always stat the target
directly or use stat -L to dereference. The nginx sites-enabled/
sites-available pattern makes this confusion very common in practice.]

---

Q6. Make python3.12 available as python in /usr/local/bin/.
    Which command is correct and why?

A) cp /usr/bin/python3.12 /usr/local/bin/python
B) ln /usr/bin/python3.12 /usr/local/bin/python
C) ln -s /usr/bin/python3.12 /usr/local/bin/python
D) mv /usr/bin/python3.12 /usr/local/bin/python

[Answer: C — Soft link. Reasons: (1) /usr/bin and /usr/local/bin
may be different filesystems — hard links cannot cross. (2) When
Python upgrades to 3.13, update one symlink. (3) ls -la shows ->
arrow making the alias obvious.
Trap: A works but wastes space and goes stale on upgrade.
D is dangerous — moves the original, breaking package manager.]

---

Q7. You create file.txt. Run: ln file.txt link1.txt
    and: ln -s file.txt link2.txt
    Then: rm file.txt
    What happens to link1.txt and link2.txt?

A) Both inaccessible
B) link1.txt works normally, link2.txt becomes dangling
C) Both work — Linux keeps copies for all links
D) link2.txt works, link1.txt deleted with original

[Answer: B — Hard link (link1.txt): shares inode with file.txt.
rm removes that directory entry, link count goes to 1. Inode and
data survive. Soft link (link2.txt): stores path "file.txt" —
that path no longer resolves. Dangling — exists but unreadable.]

---

Q8. You run cp source-link.txt copy.txt where source-link.txt is
    a symlink pointing to source.txt. What does copy.txt contain?

A) copy.txt is a symlink pointing to source.txt
B) copy.txt is a regular file containing the content of source.txt
   — cp follows the symlink by default
C) cp fails — you cannot copy a symlink
D) copy.txt is a symlink pointing to source-link.txt

[Answer: B — cp follows symlinks by default and copies the target
content as a new regular file. The symlink is not preserved.
To preserve the symlink: cp -a source-link.txt copy.txt
copy.txt then becomes a symlink pointing to source.txt.
cp -a = archive mode = -r + -p + -d (preserve symlinks as symlinks).]

---

Score guide:
Score   Action
9/9     Import Anki cards, move to Demo 03
8/9     Review wrong answer, proceed
7/9     Re-read relevant section, retry those questions
6/9 -   Re-read full demo before proceeding
