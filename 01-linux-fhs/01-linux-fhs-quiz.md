# Quiz — Demo 01: Linux Filesystem Hierarchy

> One correct answer per question. Target: 80% before Demo 02.

---

**Q1.** You run `df -h /proc` and see `Use%: -` (a dash, not a number).
What is the correct interpretation?

- A) `/proc` is full — the dash means overflow
- B) `/proc` is a virtual filesystem with no physical storage —
  percentage is undefined for a zero-size filesystem — this is
  correct behaviour
- C) `df` cannot access `/proc` due to permissions
- D) `/proc` is unmounted and needs to be remounted

<details>
<summary>Answer</summary>

**B** — `/proc` is virtual, kernel-generated on demand, nothing stored.
Size 0, Use% undefined. `df` shows `-` for all virtual filesystems.
Not an error.
Trap: A is wrong — a full filesystem shows 100%, not a dash.

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

**Q5.** `file /usr/bin/kubectl` shows `statically linked`.
`file /usr/bin/nginx` shows `dynamically linked`.
You want to copy just the binary to run in a `FROM scratch` Docker
container with no OS. Which binary works and why?

- A) nginx — it is the standard installation format
- B) kubectl — statically linked means all libraries are compiled in,
  no external dependencies needed, runs in any environment
- C) Neither — both need a full Linux environment
- D) Both — Docker provides all necessary libraries

<details>
<summary>Answer</summary>

**B** — Statically linked binaries include all dependencies compiled
into the single file. They run on any Linux kernel regardless of what
libraries are installed. This is why kubectl can run in a `FROM scratch`
container. nginx is dynamically linked — it needs specific shared
library files (libssl, libc, etc.) to be present.

</details>

---

**Q6.** Rocky Linux 9 has no `/var/log/syslog`. A colleague says
"logging is broken on this server." How do you respond?

- A) Agree — syslog should always exist on a production server
- B) Rocky Linux 9 uses journald as its primary logging system.
  Logs are in the binary journal. Query with `journalctl`, not `cat`.
  syslog is a Debian/Ubuntu convention.
- C) Install rsyslog to fix the missing syslog file
- D) `/var/log/syslog` exists but is hidden — use `ls -la`

<details>
<summary>Answer</summary>

**B** — RHEL-family distros (Rocky, RHEL, Amazon Linux) use journald
as the primary logging system. Journal is at `/var/log/journal/` in
binary format. Use `journalctl` to query it. Ubuntu runs both journald
and rsyslog. Neither is wrong — different logging architectures.
Trap: C would work but is unnecessary — journald is fully functional.

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
| 6/7 | Review wrong answer, proceed |
| 5/7 | Re-read relevant Part 1 section, retry |
| Below 5/7 | Re-read Demo 01 before proceeding |
