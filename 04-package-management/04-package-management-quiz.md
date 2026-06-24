# Quiz — Demo 04: Package Management

> One correct answer per question unless stated otherwise.
> Target: 80% or above before moving to Demo 05.

---

**Q1.** You add the Elastic repo on Rocky9 and run `dnf install filebeat`.
The package downloads successfully but install fails with
`Error: GPG check FAILED`. What is the cause?

- A) The package is corrupted — download again
- B) `gpgcheck=1` in the repo file is too strict — set it to 0
- C) The GPG key URL in the repo file is incorrect
- D) The GPG key was never imported into the RPM database —
  `rpm --import` must be run explicitly before install

<details>
<summary>Answer</summary>

**D** — The repo file's `gpgkey=` line tells dnf where to find the key but
does not import it. `rpm --import URL` must be run explicitly.
The package downloaded fine — the failure is at signature verification.

Trap: B would disable signature checking entirely — a serious security
regression. Never set `gpgcheck=0` in production.

</details>

---

**Q2.** `rpm -V curl` returns no output. What does this mean?

- A) The `curl` package is not installed
- B) `rpm -V` requires root to run — rerun with sudo
- C) All files in the `curl` package match the RPM database record —
  the package is intact
- D) The RPM database is corrupted and cannot verify

<details>
<summary>Answer</summary>

**C** — `rpm -V` prints nothing when every file passes all checks
(size, MD5, permissions, owner, group, mtime). Output only appears
when something differs. Silence is the success state.

Trap: A is wrong — if the package were not installed, rpm -V would
print an error, not silence.

</details>

---

**Q3.** `rpm -V openssh-server` returns `S.5....T.  c  /etc/ssh/sshd_config`.
A junior engineer flags this as a security incident. How do you respond?

- A) Agree — any rpm -V output indicates compromise
- B) The `c` tag means this is a config file. Config files are
  expected to be modified after install. This is normal — the
  sshd config was edited by an administrator.
- C) The file must be restored from the package immediately
- D) This means the package was installed incorrectly

<details>
<summary>Answer</summary>

**B** — The `c` tag marks a config file. `rpm -V` flags any difference
from the install-time state, including intentional admin edits.
Modified config files with a `c` tag are expected and normal.

The concern is `S.5....T.` on a **binary** with no `c` tag —
that would indicate the binary itself was modified, which is a
security indicator.

</details>

---

**Q4.** A Dockerfile has these two instructions:
```
RUN apt-get update
RUN apt-get install -y curl git
```
What is wrong with this pattern?

- A) `apt-get update` and `apt-get install` must be run as separate users
- B) Docker caches each RUN layer — a future build may use the cached
  update layer with a stale index, installing old package versions.
  Both must be in one RUN instruction.
- C) `-y` flag is not valid in Dockerfiles
- D) `apt-get` must be replaced with `apt` in containers

<details>
<summary>Answer</summary>

**B** — Docker layer caching causes the update layer to be reused on
future builds. The install layer runs with a stale package index.
Fix: `RUN apt-get update && apt-get install -y --no-install-recommends curl git && apt-get clean && rm -rf /var/lib/apt/lists/*`

Trap: D is wrong — `apt-get` is preferred over `apt` in Dockerfiles
and scripts because its interface is stable; `apt` is for interactive use.

</details>

---

**Q5.** After running `apt remove nginx`, `dpkg -l nginx` shows `rc`.
What does `rc` mean?

- A) Package is removed and nothing remains
- B) Package removal failed — config still in use
- C) Package binaries removed, config files still on disk
- D) Package is on hold at current version

<details>
<summary>Answer</summary>

**C** — `r` = desired state is remove. `c` = config files remain on disk.
`apt remove` keeps config files. To remove config files too: `apt purge nginx`.
After purge, dpkg shows `pn` (purged, nothing) or the entry disappears.

Trap: D would show `hi` (hold, installed) — not `rc`.

</details>

---

**Q6.** On Ubuntu, you want to prevent `kubelet` from being upgraded
by `apt upgrade`. Which command achieves this?

- A) `apt-mark hold kubelet`
- B) `apt pin kubelet`
- C) `dpkg --hold kubelet`
- D) `apt-get lock kubelet`

<details>
<summary>Answer</summary>

**A** — `apt-mark hold kubelet` places the package in hold state.
`apt upgrade` skips held packages and lists them under
"The following packages have been kept back". Release with:
`apt-mark unhold kubelet`. View held packages: `apt-mark showhold`.

Trap: B and D are not valid commands. C uses wrong syntax —
`dpkg` does not have a `--hold` flag; holds are managed through
`apt-mark` or `dpkg --set-selections`.

</details>

---

**Q7.** You need to find which installed package provides the file
`/usr/bin/jq` on both Ubuntu and Rocky9. Which commands are correct?

- A) Ubuntu: `apt show jq` — Rocky9: `dnf info jq`
- B) Ubuntu: `dpkg -S /usr/bin/jq` — Rocky9: `rpm -qf /usr/bin/jq`
- C) Ubuntu: `apt search jq` — Rocky9: `dnf search jq`
- D) Ubuntu: `dpkg -L jq` — Rocky9: `rpm -ql jq`

<details>
<summary>Answer</summary>

**B** — `dpkg -S path` (Search) and `rpm -qf path` (query file) both
answer "which installed package owns this file?" given a path.

Trap: A and C search for packages by name/description — they require
you to already know the package name. D lists all files in a named
package — the reverse direction.

</details>

---

Score guide:

| Score | Action |
|---|---|
| 7/7 | Import Anki cards, move to Demo 05 |
| 6/7 | Review the wrong answer, then proceed |
| 5/7 | Re-read the relevant section, retry those questions |
| Below 5/7 | Re-read the full demo and redo the walkthrough before proceeding |
