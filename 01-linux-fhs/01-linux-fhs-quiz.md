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
