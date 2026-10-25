# Demo 00e — Remote Transfer & Verification

## Overview

Moving a file to or from another machine is one command. Knowing
*which* command, and whether what arrived is actually the right file,
is where real incidents happen. `scp` is the quick one-off copy,
`sftp` is a session you can script, `rsync` (Demo 14b) is for keeping
folders in sync, and `curl` and `wget` pull from a URL with no SSH at
all. None of them tell you the file you got is the file you meant to
get. That is a separate step, a checksum, and it is easy to skip and
expensive to have skipped.

The theme of this demo: **a command exiting `0` means the command
finished, not that the right file arrived.**

| Tool | Use it for |
|---|---|
| `scp` | A quick, scripted one-off copy over SSH |
| `sftp` | A session where you browse and move several files, or a scripted batch |
| `rsync` | Syncing folders and repeated backups (Demo 14b) |
| `curl`, `wget` | Downloading from a URL (HTTP/HTTPS), no SSH |
| `sha256sum`, `md5sum` | Proving what arrived matches what was sent |

---

## Demo Objectives

By the end of this demo you will be able to:

1. Choose between `scp`, `sftp` and `rsync` for a given task
2. Copy directories with `scp -r`, and avoid the nested-copy gotcha
3. Use the right port flag: `scp -P` and `sftp -P` (capital) vs. `ssh -p` (lowercase)
4. Run an `sftp` batch with `-b`, and read its exit code
5. Download with `curl` and `wget`, and explain what each does on an HTTP error (404)
6. Use `curl -f` so a failed download fails loudly
7. Generate and verify checksums with `sha256sum` and `md5sum`, including `-c` and its exit codes
8. Explain what a checksum match proves and does not prove, and apply the verified-transfer workflow

---

## Directory Structure

```
00e-remote-transfer/
├── README.md                       # this file
├── 00e-remote-transfer-anki.csv    # Anki flashcard deck (Appendix A)
├── 00e-remote-transfer-quiz.md     # standalone quiz (Appendix B)
└── src/                            # lab working directories (git-ignored)
    └── transfer-lab/
        ├── www/                    # files served by the local web server (Tasks 5-6)
        ├── source/                 # the "source side" for checksums (Tasks 7-8)
        └── dest/                   # the "destination side" (Task 7)
```

All lab work happens inside `src/`. The lab commands create the
folders (`mkdir -p`), so you only need to be in the demo directory
(`00e-remote-transfer/`) when you start.

> **Note:** `src/` must be in the repository's root `.gitignore`. If
> you did not add it in Demo 00a, add this line once:
>
> ```
> src/
> ```

---

## Environments

| Task | Ubuntu 24.04 | Rocky Linux 9.8 | EC2 Amazon Linux |
|---|---|---|---|
| All lab tasks | ✅ | ✅ | Not required |

`scp`, `sftp`, `curl`, `wget`, `sha256sum` and `md5sum` ship by
default on both distros. The lab needs two things:

- **An SSH server** on the machine you run the lab on. It uses your own machine as the "remote" (`localhost`), so you need no second VM. Copying to yourself exercises exactly the same code as copying to another host.
- **`python3`**, to run a tiny local web server for the download tasks. Nothing needs the internet.

---

## Recall Check — Demo 00d

Before starting, answer from memory:

1. A cron job cannot find a tool whose path is only in `.bashrc`, but the script works by hand. Why, and where should the fix go?
2. Which two commands tell you whether your current shell is a login shell and whether it is interactive?
3. A login shell finds both `.bash_profile` and `.profile`. Which loads?

<details>
<summary>Answers</summary>

1. Cron runs the job with `/bin/sh` and a minimal environment and reads no startup file, while your interactive shell does read `.bashrc`. Set `PATH` where the job runs: the crontab or the script itself.
2. `shopt login_shell` (`on` means login) and `echo $-` (an `i` in the flags means interactive).
3. Only `.bash_profile`. Bash uses the first found of `.bash_profile`, `.bash_login`, `.profile` and stops.

</details>

---

## Part 1 — Concepts

### 1.1 `scp`: quick one-off copy over SSH

```bash
scp localfile.txt user@host:/remote/path/uploaded.txt   # upload
scp user@host:/remote/path/file.txt downloaded.txt       # download
```

The remote side is written `host:path` (or `user@host:path`). `scp`
uses the same authentication as `ssh`: it opens an SSH connection,
transfers, and closes it. Recent OpenSSH (9.0 and later) makes `scp`
speak the SFTP protocol internally. The command line is unchanged, but
some error messages differ between versions.

### 1.2 `scp -r`: directories need the recursive flag

```bash
scp localdir user@host:/remote/path/        # fails: a directory is not a regular file
scp -r localdir user@host:/remote/path/     # works: copies the whole tree
```

Plain `scp` on a directory prints an error saying it is not a regular
file and exits non-zero. `-r` copies the directory and everything
inside it, at any depth.

**Gotcha: whether the destination already exists changes the result.**

| Destination | Result of `scp -r localdir host:/tmp/copy` |
|---|---|
| `/tmp/copy` does **not** exist | `/tmp/copy` becomes the copy: `/tmp/copy/a.txt` |
| `/tmp/copy` **already exists** | The directory is placed *inside* it: `/tmp/copy/localdir/a.txt` |

Running the same `scp -r` twice therefore gives a different layout the
second time.

### 1.3 Port flags: `scp -P`, `sftp -P` and `ssh -p`

```bash
ssh  -p 2222 user@host          # lowercase p
scp  -P 2222 file user@host:/tmp/   # capital P
sftp -P 2222 user@host          # capital P
```

The letters differ. In `scp`, lowercase `-p` means "preserve file
times and modes", so `scp -p 2222 file host:` reads `2222` as a
filename and fails with a confusing error.

### 1.4 `sftp`: a session, or a scripted batch

```bash
sftp user@host
sftp> put localfile.txt remote-name.txt
sftp> ls
sftp> get remote-name.txt downloaded.txt
sftp> bye
```

Unlike `scp`, `sftp` opens a persistent session where you can browse
(`ls`, `cd`), move several files, and check state before transferring.
Use `put -r` and `get -r` for directories.

For scripts, `sftp -b batchfile host` runs the commands in a file,
one per line, with no prompt:

```bash
sftp -b batch.txt user@host
```

In batch mode, if a transfer command such as `get` or `put` fails,
`sftp` stops and exits non-zero, so a script can detect it.

### 1.5 `curl` vs. `wget`: downloading from a URL

```bash
curl -o output.txt https://example.com/file.txt
wget -O output.txt https://example.com/file.txt
```

Both fetch a URL over HTTP or HTTPS with no SSH involved. For a plain
download they give identical results. `wget` is the natural choice for
"save this URL to disk", including recursive site mirroring. `curl`
is the natural choice when you need headers, authentication, POST
bodies, or to pipe the response into another command.

| Flag | Tool | Meaning |
|---|---|---|
| `-o file` | `curl` | Save to `file` |
| `-O file` | `wget` | Save to `file` (capital O) |
| `-L` | `curl` | Follow redirects |
| `-s` | `curl` | Silent (hide the progress meter) |
| `-S` | `curl` | With `-s`, still show errors |
| `-f` | `curl` | **Fail** on an HTTP error instead of saving the error page |
| `-q` | `wget` | Quiet |

#### What each does on an HTTP error (404)

This is where "exit `0`" misleads:

| Command | Exit code | What ends up on disk |
|---|---|---|
| `curl -o f URL` | **`0`** | An HTML "Not Found" page saved as `f` |
| `wget -O f URL` | `8` | An empty file `f` |
| `curl -fsS -o f URL` | `22` | Nothing |

Plain `curl` considers the transfer a success, because the server
answered. The answer was just a 404 page. Always use `-f` (usually as
`curl -fsSL`) in scripts.

### 1.6 Checksums: proving what arrived matches what was sent

`sha256sum` and `md5sum` compute a fixed-length fingerprint of a
file's exact contents. Change one byte and the fingerprint changes
completely:

```bash
printf 'release build 1.0\n' | sha256sum | cut -c1-24
# 64dd4b4d4e390eef5bdc81c4
printf 'release build 1.1\n' | sha256sum | cut -c1-24
# 21b9d13416fc5d58b44a228c      ← no resemblance
```

**The `-c` workflow:** generate a checksum file once, verify against it
later:

```bash
sha256sum app.bin > app.bin.sha256     # at the source
sha256sum -c app.bin.sha256            # at the destination
# app.bin: OK                          exit code 0
```

| Situation | Output | Exit code |
|---|---|---|
| File matches | `app.bin: OK` | `0` |
| File differs | `app.bin: FAILED` and `WARNING: 1 computed checksum did NOT match` | `1` |
| File missing | `app.bin: FAILED open or read` | `1` |

`--quiet` prints only the failures. The exit code makes it scriptable:
`if sha256sum -c app.bin.sha256; then ...`.

**`sha256sum` vs. `md5sum`:** both detect accidental corruption.
`md5sum` is broken against a deliberate attacker (collisions can be
manufactured), so prefer `sha256sum` for anything that matters.

#### What a checksum does and does not prove

- A match proves the bytes are identical to **whatever the checksum was generated from**. It says nothing about whether that original was ever correct.
- Generate the checksum from the known-good source, **before** transfer, never from a copy you already suspect.
- A checksum downloaded **from the same server as the file** only catches accidental damage. An attacker who can replace the file can replace the checksum too. For authenticity you need a signature, or a checksum obtained through a separate trusted channel.
- SSH already detects corruption on the wire, so a damaged `scp` is rare. What checksums catch in practice is the **wrong file**: a partial file, a stale one, a different build than the one tested, or an error page saved as the file.

### 1.7 The verified-transfer workflow

| Where | Step |
|---|---|
| Source | Generate the checksum from the known-good file: `sha256sum app.bin > app.bin.sha256` |
| Transfer | Send **both** the file and the checksum file |
| Destination | Verify: `sha256sum -c app.bin.sha256`. Continue only if it exits `0` |

Treat "the transfer command succeeded" and "the file is correct" as
two separate things to confirm.

---

## Part 2 — Hands-On Lab

### Commands used in this demo

| Command | Flag meaning | Purpose |
|---|---|---|
| `scp -r` | `r` = recursive | Copy a directory tree over SSH |
| `scp -P` | `P` = port | Connect to a non-default SSH port |
| `sftp -b` | `b` = batch file | Run sftp commands from a file |
| `ssh -G` | `G` = print config | Show the settings ssh would use (including the port) |
| `curl -fsSL` | `f` = fail, `s` = silent, `S` = show errors, `L` = follow redirects | A safe download in scripts |
| `wget -O` | `O` = output file | Save a download to a specific name |
| `sha256sum -c` | `c` = check | Verify files against a checksum file |
| `python3 -m http.server` | n/a | A tiny local web server for the lab |
| `&` | n/a | Run a command in the background (Demo 00f covers it fully) |

### Step 0 — Set up (about a minute)

```bash
echo $0                              # if this says zsh, run:  bash
REMOTE=localhost
RDIR=/tmp/transfer-lab-remote

ssh "$REMOTE" 'echo connected as $(whoami) on $(hostname)'
ssh "$REMOTE" "mkdir -p $RDIR"
```

The two `ssh` lines use different quotes on purpose (Demo 00a): in
single quotes the **remote** shell runs `$(whoami)`; in double quotes
**your** shell expands `$RDIR` before sending.

If the first command prints `connected as ... on ...`, you are ready.
If not:

| What you see | Fix |
|---|---|
| `Are you sure you want to continue connecting?` | Type `yes` (first connection only) |
| A password prompt on every command | Expected. To stop it: `ssh-keygen -t ed25519` (press Enter at each question), then `ssh-copy-id "$REMOTE"` |
| `Connection refused` | Start the SSH server: `sudo systemctl start ssh` (Ubuntu) or `sudo systemctl start sshd` (Rocky). Not installed? `sudo apt install openssh-server` or `sudo dnf install openssh-server` |
| Refused, but the server is running | It uses a **non-default port**. Find it with `sudo ss -tlnp \| grep sshd`, then create an alias (below) |

**Alias for a non-default port** (only if you need it). Set `PORT` to
your server's port, run these four lines once, and carry on with
`REMOTE=lab`:

```bash
PORT=2404
mkdir -p ~/.ssh && chmod 700 ~/.ssh
printf 'Host lab\n    HostName localhost\n    Port %s\n' "$PORT" >> ~/.ssh/config
chmod 600 ~/.ssh/config
REMOTE=lab
```

### Task 1 — One-off transfer with `scp` (Concepts §1.1)

```bash
# from the demo directory: 00e-remote-transfer/
mkdir -p src/transfer-lab && cd src/transfer-lab

echo "test content" > localfile.txt
scp localfile.txt "$REMOTE:$RDIR/uploaded.txt"
ssh "$REMOTE" "cat $RDIR/uploaded.txt"
# test content                   ← the file arrived

scp "$REMOTE:$RDIR/uploaded.txt" roundtrip.txt
diff localfile.txt roundtrip.txt; echo "diff exit: $?"
# diff exit: 0                   ← byte-identical round trip
```

### Task 2 — Directories need `-r`, and the nesting gotcha (Concepts §1.2)

```bash
mkdir -p localdir/subdir
echo "a" > localdir/a.txt
echo "b" > localdir/subdir/b.txt

scp localdir "$REMOTE:$RDIR/"; echo "exit: $?"
# an error saying localdir is not a regular file (wording varies by version)
# exit: 1                        ← non-zero

scp -r localdir "$REMOTE:$RDIR/localdir_copy"; echo "exit: $?"
# exit: 0
ssh "$REMOTE" "find $RDIR/localdir_copy -type f | sort"
# …/localdir_copy/a.txt
# …/localdir_copy/subdir/b.txt
```

**Now run the same `scp -r` again:**

```bash
scp -r localdir "$REMOTE:$RDIR/localdir_copy"
ssh "$REMOTE" "find $RDIR/localdir_copy -type f | sort"
# …/localdir_copy/a.txt
# …/localdir_copy/localdir/a.txt             ← nested inside, because it now exists
# …/localdir_copy/localdir/subdir/b.txt
# …/localdir_copy/subdir/b.txt
```

### Task 3 — Port flags (Concepts §1.3)

`ssh -G` prints the settings ssh would use, so this picks out your
port whether it is 22 or set in an alias:

```bash
PORT=$(ssh -G "$REMOTE" | awk '$1=="port"{print $2}')
echo "$PORT"                               # 22 (or your port)

ssh -p "$PORT" "$REMOTE" hostname          # prints this machine's hostname
scp -P "$PORT" localfile.txt "$REMOTE:$RDIR/port-test.txt"; echo "exit: $?"
# exit: 0                                  ← capital P for scp

scp -p "$PORT" localfile.txt "$REMOTE:$RDIR/oops.txt"; echo "exit: $?"
# an error about a file named 22 (or your port); non-zero exit
#                                          ← lowercase p means "preserve times"

scp -P 2 localfile.txt "$REMOTE:$RDIR/x.txt"; echo "exit: $?"
# connection refused (wording varies); non-zero exit
```

### Task 4 — An `sftp` batch (Concepts §1.4)

```bash
printf 'cd %s\nput localfile.txt sftp-uploaded.txt\nls\nget sftp-uploaded.txt sftp-downloaded.txt\n' "$RDIR" > batch.txt
cat batch.txt
# cd /tmp/transfer-lab-remote
# put localfile.txt sftp-uploaded.txt
# ls
# get sftp-uploaded.txt sftp-downloaded.txt

sftp -b batch.txt "$REMOTE"
# the commands are echoed as they run, and ls lists the uploaded file
diff localfile.txt sftp-downloaded.txt; echo "diff exit: $?"
# diff exit: 0
```

**A failing batch exits non-zero:**

```bash
printf 'get nosuchfile.txt\n' > bad-batch.txt
sftp -b bad-batch.txt "$REMOTE"; echo "exit: $?"
# a "not found" error
# exit: 1                          ← non-zero: a script can detect it
```

### Task 5 — `curl` and `wget` from a local web server (Concepts §1.5)

Start a tiny web server in the background, so no second terminal or
internet is needed:

```bash
mkdir -p www
echo "hello release" > www/app.txt

python3 -m http.server 8000 --directory www > /dev/null 2>&1 &
SERVER_PID=$!
sleep 1
```

`&` runs the server in the background, `> /dev/null 2>&1` silences its
log (Demo 00b), and `$!` holds its process ID so you can stop it later.

```bash
curl -sL -o via_curl.txt http://localhost:8000/app.txt
wget -q -O via_wget.txt http://localhost:8000/app.txt

diff via_curl.txt via_wget.txt; echo "diff exit: $?"
# diff exit: 0                       ← identical results
cat via_curl.txt
# hello release
```

### Task 6 — HTTP errors: what exit `0` hides (Concepts §1.5)

Ask the server for a file that does not exist:

```bash
curl -s -o curl-404.txt http://localhost:8000/missing.txt; echo "curl exit: $?"
# curl exit: 0                       ← "success"!
wc -c < curl-404.txt                 # a few hundred bytes
head -n 3 curl-404.txt
# <!DOCTYPE HTML>
# <html lang="en">
#     <head>                         ← an HTML error page saved as your file

wget -q -O wget-404.txt http://localhost:8000/missing.txt; echo "wget exit: $?"
# wget exit: 8
wc -c < wget-404.txt                 # 0   ← an empty file

curl -fsS -o curl-f.txt http://localhost:8000/missing.txt; echo "curl -f exit: $?"
# curl: (22) The requested URL returned error: 404
# curl -f exit: 22
ls curl-f.txt
# ls: cannot access 'curl-f.txt': No such file or directory   ← nothing saved
```

### Task 7 — Checksums (Concepts §1.6)

```bash
mkdir -p source dest
printf 'release build 1.0\n' > source/app.bin

cd source
sha256sum app.bin > app.bin.sha256
cat app.bin.sha256
# 64dd4b4d4e390eef5bdc81c422b1d62d5c0c0d6db1d72d99f6af26493a2d4fdb  app.bin
cd ..

md5sum source/app.bin
# 0ff52c162a431c3c4a043ce1fef5cffa  source/app.bin     ← shorter, but weaker
```

"Transfer" both files to the destination side and verify:

```bash
cp source/app.bin source/app.bin.sha256 dest/
cd dest

sha256sum -c app.bin.sha256; echo "exit: $?"
# app.bin: OK
# exit: 0
```

Damage the destination copy and verify again:

```bash
echo "x" >> app.bin

sha256sum -c app.bin.sha256; echo "exit: $?"
# app.bin: FAILED
# sha256sum: WARNING: 1 computed checksum did NOT match
# exit: 1

sha256sum -c --quiet app.bin.sha256; echo "exit: $?"
# app.bin: FAILED                    ← --quiet prints only failures
# exit: 1

rm app.bin
sha256sum -c app.bin.sha256; echo "exit: $?"
# sha256sum: app.bin: No such file or directory
# app.bin: FAILED open or read
# sha256sum: WARNING: 1 listed file could not be read
# exit: 1
cd ..
```

### Task 8 — The full verified transfer over SSH (Concepts §1.7)

```bash
cd source
scp app.bin app.bin.sha256 "$REMOTE:$RDIR/"
ssh "$REMOTE" "cd $RDIR && sha256sum -c app.bin.sha256"
# app.bin: OK
```

Simulate damage on the remote copy, then verify again. `ssh` passes
the remote command's exit code back to you:

```bash
ssh "$REMOTE" "echo x >> $RDIR/app.bin"
ssh "$REMOTE" "cd $RDIR && sha256sum -c app.bin.sha256"; echo "exit: $?"
# app.bin: FAILED
# sha256sum: WARNING: 1 computed checksum did NOT match
# exit: 1
cd ..
```

**Clean up (optional, since `src/` is git-ignored):**

```bash
kill "$SERVER_PID"                   # stop the web server
ssh "$REMOTE" "rm -rf $RDIR"         # remove the remote working directory
cd ../..                             # back to 00e-remote-transfer/
rm -rf src/transfer-lab
```

---

## What You Learned

- ✅ Copied files with `scp`, confirmed a byte-identical round trip, and saw plain `scp` refuse a directory
- ✅ Reproduced the `scp -r` nesting gotcha, and the `-P` vs. `-p` mix-up
- ✅ Ran an `sftp` batch with `-b` and saw a failing batch exit non-zero
- ✅ Compared `curl` and `wget`, and saw that on a 404 `curl` exits `0` with an error page, `wget` exits `8` with an empty file, and `curl -f` exits `22` with nothing saved
- ✅ Generated and verified checksums with `-c`, and saw `OK` (exit 0) vs. `FAILED` (exit 1)
- ✅ Ran the full source → transfer → verify workflow over SSH

**Key Takeaway:** a transfer command exiting `0` means the command
finished. It does not mean the right bytes arrived. Use `curl -f` to
make HTTP errors fail, and a checksum comparison to prove the file is
the one you meant to get.

---

## Part 3 — Break-Fix Scenario

### Scenario — NovaTech: the deploy "succeeded" but the service will not start

You are handed this ticket:

> *"Last night's deploy reported success. This morning the service
> will not start: the log says `cannot execute binary file: Exec format
> error`. The build server's copy of the binary runs fine. Nobody
> touched the code."*

The deploy script contains:

```bash
curl -s -o /opt/app/app.bin https://artifacts.novatech.example/app/app-v2.bin
chmod +x /opt/app/app.bin
systemctl restart app
```

You find that the artifact was renamed to `app-v2.1.bin` two days ago.

**Reproduce it** (from `src/transfer-lab`; start the server again with
the Task 5 commands if you stopped it):

```bash
curl -s -o app.bin http://localhost:8000/app-v2.bin; echo "exit: $?"
head -n 2 app.bin
```

**Your task (answer before opening the resolution):**

1. `curl` exited `0`. Why did the script think everything worked, and what is actually in `app.bin`?
2. Which two safeguards would each have caught this before the service was restarted?
3. What is the correct download-and-deploy process?

<details>
<summary>Resolution — open after attempting</summary>

**1. Why the script saw success:**

The server answered the request (with a 404 page), so as far as `curl`
is concerned the transfer completed. Without `-f`, `curl` does not
treat an HTTP error as a failure, so it saved the error page and
exited `0`. `app.bin` is a few hundred bytes of HTML, and `chmod +x`
works on any file.

**2. Two safeguards, either of which would have caught it:**

- `curl -fsSL`: the 404 makes `curl` exit `22` and save nothing, so the script can stop.
- A checksum check: `sha256sum -c` against the published checksum prints `FAILED` and exits `1`, because an HTML page does not match a binary's checksum.

**3. The correct process:**

```bash
curl -fsSL -o /tmp/app.bin.new https://artifacts.example/app/app-v2.1.bin
echo "<published sha256>  /tmp/app.bin.new" | sha256sum -c -
mv /tmp/app.bin.new /opt/app/app.bin
```

Download to a temporary name, check the exit code, verify the
checksum, and only then move the file into place. Never overwrite the
live file until the new one is proven good.

**The lesson:** exit `0` is a claim about the command, not about the
file. `-f` makes HTTP errors loud, and a checksum proves the bytes
are the ones you meant to get.

</details>

---

## Interview Prep

**Q1. When would you choose `sftp` over `scp`, and when `rsync` over both?**

`sftp` when you want to browse an unfamiliar remote filesystem, move
several files in one session, or run a scripted batch (`sftp -b`).
`scp` for a single scripted copy where you already know the source
and destination. `rsync` (Demo 14b) when you are syncing a folder or
repeating the transfer, because it sends only what changed.

---

**Q2. A deploy script downloads an artifact with `curl -s -o` and checks `$?` before continuing. Is that enough?**

No. Without `-f`, `curl` exits `0` even on an HTTP error such as 404
and saves the error page as your file. Use `curl -fsSL` so the error
fails the command, and verify a checksum (`sha256sum -c`) before using
the file.

---

**Q3. The checksum file you downloaded matches the artifact you downloaded. Does that prove the artifact is authentic?**

No. If both came from the same server, whoever could replace the
artifact could replace its checksum. The match only proves nothing
damaged the file in transit. For authenticity, verify a signature, or
get the checksum through a separate trusted channel.

---

**Q4. `scp -r mydir host:/data/backup` runs nightly. The first night the layout is right. After that you find `/data/backup/mydir/mydir/...`. Why?**

On the first run `/data/backup` did not exist, so it became the copy.
On later runs it existed, so `scp -r` placed `mydir` *inside* it. For
repeated syncs use `rsync` (Demo 14b), or copy into an explicit
target such as `host:/data/backup/` and keep the layout consistent.

---

## Key Takeaways

1. **`scp` refuses a bare directory with an error and a non-zero exit; `-r` copies the tree.** If the destination already exists, `scp -r` nests the directory inside it, so repeated runs change the layout.
2. **Port flags differ: `scp -P` and `sftp -P` (capital) vs. `ssh -p` (lowercase).** In `scp`, lowercase `-p` means "preserve times", so the port number is read as a filename.
3. **`sftp -b batchfile` runs commands from a file, and a failing `get` or `put` makes it exit non-zero.**
4. **On an HTTP 404, plain `curl` exits `0` and saves the error page; `wget` exits `8` and leaves an empty file; `curl -f` exits `22` and saves nothing.** Use `curl -fsSL` in scripts.
5. **`sha256sum -c` prints `OK` and exits `0` on a match, `FAILED` and exits `1` on a mismatch or missing file.** Build it into scripts instead of eyeballing hashes.
6. **A checksum match proves the bytes equal whatever the checksum was made from, not that the original was correct, and not that it is authentic if the checksum came from the same server.** Generate it from the known-good source before transfer.
7. **SSH detects corruption on the wire; checksums catch the wrong, partial or stale file.** Prefer `sha256sum` to `md5sum` for anything that matters.
8. **A transfer command exiting `0` means the command finished, not that the right file arrived.** Confirm those separately.

---

## Quick Commands Reference

### Transfer tools compared

| Tool | Best for | Directories |
|---|---|---|
| `scp` | Quick scripted one-off copy | `-r` required (nests if the destination exists) |
| `sftp` | Browsing, multi-file sessions, scripted batches | `put -r` / `get -r` |
| `rsync` | Syncing and repeated backups (Demo 14b) | Built in |
| `curl` | Headers, auth, POST, piping; use `-fsSL` in scripts | n/a (HTTP) |
| `wget` | Saving a URL to disk, mirroring | n/a (HTTP) |

### Port flags

| Tool | Port flag |
|---|---|
| `ssh` | `-p` (lowercase) |
| `scp` | `-P` (capital) |
| `sftp` | `-P` (capital) |

### HTTP 404 behavior

| Command | Exit | On disk |
|---|---|---|
| `curl -o f URL` | `0` | HTML error page |
| `wget -O f URL` | `8` | Empty file |
| `curl -fsS -o f URL` | `22` | Nothing |

### Checksum workflow

```bash
sha256sum file > file.sha256     # at the source, from the known-good file
sha256sum -c file.sha256         # at the destination: OK / exit 0, or FAILED / exit 1
sha256sum -c --quiet file.sha256 # print only failures
```

---

## Appendix A — Anki Cards

**`00e-remote-transfer-anki.csv`:**

```
#deck:Linux Mastery::Module 1 - Filesystem & Navigation::Demo 00e - Remote Transfer & Verification
#separator:Comma
#columns:Front,Back,Tags

"What happens when you run plain scp on a directory, and what is the fix?","scp prints an error saying it is not a regular file and exits non-zero. scp -r copies the whole directory tree, at any depth.","demo00e,scp,directories,recursive"

"scp -r localdir host:/tmp/copy is run twice. Why is the layout different the second time?","The first time /tmp/copy does not exist, so it becomes the copy. The second time it exists, so scp -r places localdir INSIDE it (/tmp/copy/localdir/...). Repeated syncs should use rsync.","demo00e,scp,nesting,gotcha"

"What are the port flags for ssh, scp and sftp?","ssh uses lowercase -p; scp and sftp use capital -P. In scp, lowercase -p means preserve times and modes, so scp -p 2222 file host: reads 2222 as a filename and fails.","demo00e,scp,ssh,port"

"What is the practical difference between scp and sftp, and when do you reach for rsync instead?","scp is a single scripted copy. sftp is a session (browse with ls/cd, move several files) or a scripted batch with -b. rsync (Demo 14b) is for syncing folders and repeated transfers because it sends only what changed.","demo00e,scp,sftp,rsync"

"A failing command inside an sftp -b batch file (for example get of a missing file). What happens?","sftp stops and exits non-zero, so a script can detect the failure.","demo00e,sftp,batch,exitcode"

"curl -o file URL is run and the server returns 404. What is the exit code, and what is in the file?","Exit code 0, and the file contains the HTML error page. Without -f, curl treats any server response as a successful transfer.","demo00e,curl,404,exitcode"

"wget -O file URL is run and the server returns 404. What is the exit code and what is in the file?","Exit code 8 (server error response), and the file is empty (0 bytes).","demo00e,wget,404,exitcode"

"What does curl -f do, and what flags make a safe script download?","-f (--fail) makes curl exit 22 on an HTTP error and save nothing. A safe download is curl -fsSL: fail on errors, silent progress, still show errors, follow redirects.","demo00e,curl,fail,safe"

"sha256sum -c file.sha256 is run against a modified file. What is printed and what is the exit code?","'file: FAILED' plus 'WARNING: 1 computed checksum did NOT match', and exit code 1. A match prints 'file: OK' and exits 0. A missing file prints 'FAILED open or read' and exits 1.","demo00e,sha256sum,verify,exitcode"

"Does a matching checksum prove the original file was correct?","No. It proves the bytes equal whatever the checksum was generated from. Generate the checksum from the known-good source, before transfer.","demo00e,checksum,limitations"

"A checksum file downloaded from the same server as the artifact matches the artifact. Does that prove authenticity?","No. Whoever can replace the artifact can replace its checksum. It only catches accidental damage. For authenticity, verify a signature or get the checksum through a separate trusted channel.","demo00e,checksum,authenticity"

"What is the verified-transfer workflow, in order?","1) At the source, generate the checksum from the known-good file. 2) Transfer both the file and the checksum file. 3) At the destination run sha256sum -c and continue only if it exits 0.","demo00e,checksum,workflow"

"Why prefer sha256sum over md5sum, and what do checksums catch in practice if SSH already protects the wire?","md5 is broken against deliberate attackers (collisions can be manufactured), though fine for accidental corruption. SSH detects corruption in transit, so in practice checksums catch the wrong, partial or stale file, a different build than the one tested, or an error page saved as the file.","demo00e,checksum,md5,sha256"
```

---

## Appendix B — Quiz

**`00e-remote-transfer-quiz.md`:**

```
# Quiz — Demo 00e: Remote Transfer & Verification

> One correct answer per question. Target: 80% (8/10) before Demo 00f.
> Assume bash.

---

**Q1.** `scp mydir user@host:/tmp/` is run where `mydir` is a
directory. What happens?

- A) It copies only the files directly inside `mydir`
- B) It prints a "not a regular file" error and exits non-zero: `-r` is required
- C) It creates an empty directory on the remote side
- D) It copies everything but flattens the structure

<details>
<summary>Answer</summary>

**B**. Plain `scp` refuses a directory. `-r` copies the whole tree.

</details>

---

**Q2.** `scp -r localdir host:/tmp/copy` is run twice. What is the
layout after the second run?

- A) Identical to the first run
- B) `localdir` is nested inside `/tmp/copy`, because it now exists
- C) The second run fails
- D) The first copy is deleted and replaced

<details>
<summary>Answer</summary>

**B**. When the destination exists, `scp -r` copies the directory
inside it. For repeated syncs, use `rsync`.

</details>

---

**Q3.** Which command connects to an SSH server on port 2222?

- A) `scp -p 2222 file host:/tmp/`
- B) `scp -P 2222 file host:/tmp/`
- C) `ssh -P 2222 host`
- D) `sftp -p 2222 host`

<details>
<summary>Answer</summary>

**B**. `scp` and `sftp` use capital `-P`; `ssh` uses lowercase `-p`.
In `scp`, lowercase `-p` means "preserve times".

</details>

---

**Q4.** A `get nosuchfile.txt` line is inside a file run with
`sftp -b batch.txt host`. What is the exit status?

- A) `0`, because `sftp` itself ran
- B) Non-zero: a failing transfer command stops the batch
- C) Always `255`
- D) `sftp` hangs waiting for input

<details>
<summary>Answer</summary>

**B**. In batch mode a failing command such as `get` stops `sftp` and
gives a non-zero exit, so scripts can detect it.

</details>

---

**Q5.** `curl -s -o app.bin URL` runs against a URL that returns 404.
What is `$?`, and what is in `app.bin`?

- A) `22`, and nothing was saved
- B) `0`, and `app.bin` holds the HTML error page
- C) `8`, and `app.bin` is empty
- D) `1`, and `app.bin` holds the old version

<details>
<summary>Answer</summary>

**B**. Without `-f`, `curl` treats any server response as success.
`22` is `curl -f`; `8` is what `wget` returns.

</details>

---

**Q6.** The same 404 is fetched with `wget -q -O app.bin URL`. What
happens?

- A) Exit `0` and an error page saved
- B) Exit `8` and an empty file
- C) Exit `22` and nothing saved
- D) Exit `0` and an empty file

<details>
<summary>Answer</summary>

**B**. `wget` exits `8` on a server error response, and `-O` leaves an
empty file behind.

</details>

---

**Q7.** Which flag makes `curl` fail on an HTTP error?

- A) `-s`
- B) `-L`
- C) `-f`
- D) `-o`

<details>
<summary>Answer</summary>

**C**. `-f` (`--fail`) gives exit `22` and saves nothing. A safe script
download is `curl -fsSL`.

</details>

---

**Q8.** `sha256sum -c app.bin.sha256` runs against a corrupted file.
What happens?

- A) Prints nothing, exits `0`
- B) Prints `app.bin: FAILED` and exits `1`
- C) Prints the new hash and exits `0`
- D) Crashes with no exit code

<details>
<summary>Answer</summary>

**B**. The exit code makes it usable in scripts
(`if sha256sum -c app.bin.sha256; then ...`).

</details>

---

**Q9.** A checksum matches after transfer. What does that prove?

- A) The original file was correct
- B) The file is authentic, even if the checksum came from the same server
- C) The transferred bytes equal whatever the checksum was generated from
- D) The transfer used a secure protocol

<details>
<summary>Answer</summary>

**C**. It does not prove the original was correct, and a checksum from
the same server as the file does not prove authenticity.

</details>

---

**Q10.** Which order is the verified-transfer workflow?

- A) Transfer the file, then generate the checksum on the destination
- B) Generate the checksum at the source from the known-good file, transfer both, verify at the destination
- C) Generate the checksum on both sides and compare by eye
- D) Verify first, then transfer

<details>
<summary>Answer</summary>

**B**. A checksum made from an already-transferred (possibly damaged)
copy proves nothing.

</details>

---

Score guide:

| Score | Action |
|---|---|
| 10/10 | Import Anki cards, move to Demo 00f |
| 9/10 | Review the wrong answer, then proceed |
| 7-8/10 | Re-read the relevant section, retry those questions |
| Below 7/10 | Re-read the full demo and redo the lab |
```