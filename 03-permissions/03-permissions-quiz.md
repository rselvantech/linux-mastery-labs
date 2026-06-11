# Demo 03 — Quiz
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
