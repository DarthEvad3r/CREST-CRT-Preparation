# Hash Format Reference

A reference for identifying hash formats and looking up the corresponding hashcat mode and john format.

## Identifying by Length and Charset

| Length | Charset | Likely format |
|---|---|---|
| 16 hex chars | a-f0-9 | DES, MySQL323 |
| 32 hex chars | a-f0-9 | MD5, NTLM, MD4, MySQL3 |
| 40 hex chars | a-f0-9 | SHA-1, MySQL4.1+ (with `*` prefix) |
| 56 hex chars | a-f0-9 | SHA-224 |
| 64 hex chars | a-f0-9 | SHA-256, SHA3-256, Keccak-256 |
| 96 hex chars | a-f0-9 | SHA-384 |
| 128 hex chars | a-f0-9 | SHA-512, Whirlpool |
| 28 base64 | a-zA-Z0-9+/= | SHA-1 base64 (LDAP `{SHA}`) |
| 60 chars starting `$2a$`,`$2b$`,`$2y$` | | bcrypt |

## Quick identification with tools

```bash
# hashid (best general identifier)
hashid '5f4dcc3b5aa765d61d8327deb882cf99'
> Possible Hashs:
> [+] MD5
> [+] NTLM
> [+] MD4

# hash-identifier (older alternative)
hash-identifier
# (paste hash interactively)

# Useful for prefix-based formats
# $1$  -> md5crypt          mode 500
# $2a/2b/2y$ -> bcrypt      mode 3200
# $5$  -> sha256crypt       mode 7400
# $6$  -> sha512crypt       mode 1800
# $y$  -> yescrypt          mode 30001 (newer hashcat)
# $apr1$ -> Apache htpasswd MD5  mode 1600
# $argon2id$ / $argon2i$ -> Argon2  modes 32xxx
```

## Common Hashes by Hashcat Mode

The complete reference table. Sort order: most common first.

### Common Web/Application Hashes

| Mode | Format | Sample | Notes |
|---|---|---|---|
| 0 | MD5 | `5f4dcc3b5aa765d61d8327deb882cf99` | Unsalted MD5 |
| 100 | SHA-1 | `5baa61e4c9b93f3f0682250b6cf8331b7ee68fd8` | Unsalted SHA-1 |
| 1400 | SHA-256 | `e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855` | Unsalted SHA-256 |
| 1700 | SHA-512 | (128 hex) | Unsalted SHA-512 |
| 10 | MD5 + salt | `5f4dcc3b5aa765d61d8327deb882cf99:salt` | Salted append |
| 20 | salt + MD5 | (with salt-MD5 ordering) | |
| 110 | SHA-1 + salt | `hash:salt` | |
| 120 | salt + SHA-1 | | |
| 1410 | SHA-256 + salt | `hash:salt` | |
| 1510 | salt + SHA-256 | | |
| 1710 | SHA-512 + salt | `hash:salt` | |
| 1410 | sha256(pass.salt) | | |
| 1450 | HMAC-SHA-256 (key=$pass) | | |
| 1500 | descrypt | `MnE5LtTUA3GvE` (13 chars) | DES Unix crypt |
| 1600 | Apache `$apr1$` | `$apr1$salt$hash` | htpasswd MD5 |
| 1800 | sha512crypt `$6$` | `$6$rounds=5000$salt$hash` | Modern Linux /etc/shadow |
| 7400 | sha256crypt `$5$` | `$5$rounds=5000$salt$hash` | Linux /etc/shadow |
| 500 | md5crypt `$1$` | `$1$salt$hash` | Older Linux /etc/shadow |
| 3200 | bcrypt | `$2y$10$salt.hash` (60 chars) | Slow, modern best practice |
| 30001 | yescrypt | `$y$j9T$salt$hash` | Newest Linux default |

### Microsoft Windows Hashes

| Mode | Format | Sample | Notes |
|---|---|---|---|
| 1000 | NTLM | `8846f7eaee8fb117ad06bdd830b7586c` | The big one - SAM, NTDS |
| 3000 | LM | `b757bf5c0d87772faf6e1d6af6b9a5c6` | Legacy, weak |
| 5500 | NetNTLMv1 | `user::DOMAIN:hash:hash:nonce` | Captured network challenge-response |
| 5600 | NetNTLMv2 | `user::DOMAIN:nonce:hash:blob` | From Responder |
| 1100 | DCC (cached) | `M$user#hash` | Cached domain logon (legacy XP/2003) |
| 2100 | DCC2 / MSCash2 | `$DCC2$10240#user#hash` | Cached domain logon (modern) |
| 5300 | IKE-PSK MD5 | `id:nonce:nonce:nonce:nonce:hash:hash` | IKE Aggressive Mode PSK |
| 5400 | IKE-PSK SHA1 | (same shape) | IKE Aggressive Mode |
| 13100 | Kerberos TGS-REP RC4 | `$krb5tgs$23$*user$DOMAIN$spn*$cs$ed` | Kerberoast (default RC4) |
| 19600 | Kerberos TGS-REP AES128 | `$krb5tgs$17$user$DOMAIN$cs$ed` | Kerberoast AES128 |
| 19700 | Kerberos TGS-REP AES256 | `$krb5tgs$18$user$DOMAIN$cs$ed` | Kerberoast AES256 |
| 18200 | Kerberos AS-REP | `$krb5asrep$23$user@DOMAIN:cs$ed` | AS-REP roast |
| 7500 | Kerberos PreAuth (5/AS-REQ) | `$krb5pa$23$user$REALM$salt$blob` | (rarely used) |
| 19800 | Kerberos 5 PA-ETYPE-INFO2 (AES128) | | |
| 19900 | Kerberos 5 PA-ETYPE-INFO2 (AES256) | | |
| 13000 | RAR5 | `$rar5$16$salt$rounds$iv$hash` | |

### Database Hashes

| Mode | Format | Sample | Notes |
|---|---|---|---|
| 200 | MySQL pre-4.1 | (16 hex) | Very weak, deprecated |
| 300 | MySQL 4.1/5.x | `*00FA8C8A2E9F7CDDF4CDFB87CDCFAD5C8A2E9F7C` | The * prefix is required |
| 7401 | MySQL 8 caching_sha2 | `$mysql$$5$...` | Modern MySQL default |
| 12 | PostgreSQL md5 | `md5<32hex>` | Salt = username |
| 28600 | PostgreSQL SCRAM-SHA-256 | `SCRAM-SHA-256$4096$salt$server-key$client-key` | Modern PostgreSQL |
| 132 | MSSQL 2005-2008 | `0x0100<8B salt><40B hash>` | |
| 1731 | MSSQL 2012+ | `0x0200<8B salt><64B hash>` | |
| 112 | Oracle 11g (S:) | `S:<60 hex>` | Combined with username |
| 12300 | Oracle 12c+ T: | `H:<long>` or `S:<long>` | |
| 8200 | 1Password Cloud Keychain | | |

### Wireless and Network

| Mode | Format | Sample | Notes |
|---|---|---|---|
| 22000 | WPA-PBKDF2-PMKID+EAPOL | `WPA*<args>*<hash>` | Modern combined format |
| 2500 | WPA/WPA2 EAPOL (legacy) | (HCCAPX) | Older format |
| 16800 | WPA-PMKID-PBKDF2 | `<pmkid>*<bssid>*<sta_mac>*<essid>` | PMKID attack |
| 16801 | WPA-PMKID-PMK | (PMK directly) | If PMK known |
| 7300 | iSCSI CHAP | | |

### Web Tokens / Modern

| Mode | Format | Sample | Notes |
|---|---|---|---|
| 16500 | JWT HS256 | `eyJ...eyJ...sig` | Crackable HMAC secret |
| 16511 | JWT HS384 | (similar) | |
| 16512 | JWT HS512 | (similar) | |
| 31100 | CRAM-MD5 (Dovecot) | | |
| 31200 | Veeam VBR (Backup & Replication) | | |
| 1800 | Cisco IOS Type 5 | `$1$salt$hash` | Same as md5crypt |
| 9200 | Cisco IOS Type 8 | `$8$salt$hash` | PBKDF2-SHA-256 |
| 9300 | Cisco IOS Type 9 | `$9$salt$hash` | scrypt |
| 5700 | Cisco-IOS Type 4 | (64 hex) | PBKDF2-SHA-256 |
| 2400 | Cisco-PIX MD5 | `<16chars>` | |
| 2410 | Cisco-ASA MD5 | `<16chars>:<user>` | |
| (n/a) | Cisco Type 7 | `7<digits>` | Reversible - decode with online tools or `cisco7decode.py` |

### Encrypted Files / Archives

| Mode | Format | Sample | Notes |
|---|---|---|---|
| 9400 | Office 2007 | `$office$*2007$...` | |
| 9500 | Office 2010 | `$office$*2010$...` | |
| 9600 | Office 2013+ | `$office$*2013$...` | |
| 11600 | 7-Zip | `$7z$0$...` | |
| 12500 | RAR3 | `$RAR3$...` | |
| 13000 | RAR5 | `$rar5$...` | |
| 13600 | WinZip | `$zip2$...$/zip2$` | |
| 17225 | PKZIP (compressed) | | |
| 17220 | PKZIP (compressed multi-file) | | |
| 17200 | PKZIP (compressed) - mode 1 | | |
| 17210 | PKZIP (uncompressed) | | |
| 10500 | PDF 1.4-1.6 (40-bit) | `$pdf$1*2*40*-1060*1*16*hash*32*hash*32*hash` | |
| 10600 | PDF 1.7 Level 3 | | |
| 10700 | PDF 1.7 Level 8 | | |

### Disk / Container / Backup

| Mode | Format | Sample | Notes |
|---|---|---|---|
| 13400 | KeePass 2.x | `$keepass$*2*...` | |
| 14600 | LUKS | (from luks2john) | |
| 16700 | FileVault 2 | | |
| 6211 | TrueCrypt RIPEMD160 | | |
| 6212 | TrueCrypt SHA512 | | |
| 6213 | TrueCrypt Whirlpool | | |
| 13711-13772 | VeraCrypt various | | |

### SSH Keys

| Mode | Format | Sample | Notes |
|---|---|---|---|
| 22911 | RSA/DSA/EC/OpenSSH (>$3) | (from ssh2john) | Older format |
| 22921 | RSA/DSA/EC/OpenSSH bcrypt-pbkdf | | Modern OpenSSH default |
| 22931 | OpenSSH RSA bcrypt | | |
| 22941 | OpenSSH ECDSA bcrypt | | |
| 22951 | OpenSSH ED25519 bcrypt | | |

### Linux /etc/shadow Quick Reference

The string in field 2 of `/etc/shadow` directly tells you the format:

```
user:HASH:lastchange:min:max:warn:inactive:expire:reserved
```

| Hash field starts with | Algorithm | Hashcat mode |
|---|---|---|
| `$1$` | MD5 (md5crypt) | 500 |
| `$2a$`, `$2b$`, `$2y$` | bcrypt | 3200 |
| `$5$` | SHA-256 (sha256crypt) | 7400 |
| `$6$` | SHA-512 (sha512crypt) | 1800 |
| `$y$` | yescrypt | 30001 |
| `$argon2i$`, `$argon2id$` | Argon2 | (varies) |
| (just `*` or `!`) | Account locked | n/a |
| (numeric hash, no `$`) | Legacy DES | 1500 |

## Hashcat Quick Invocation

```bash
# Standard wordlist
hashcat -m <mode> hashes.txt /usr/share/wordlists/rockyou.txt

# With rule
hashcat -m <mode> hashes.txt rockyou.txt -r /usr/share/hashcat/rules/best64.rule
hashcat -m <mode> hashes.txt rockyou.txt -r /usr/share/hashcat/rules/dive.rule

# Mask attack
hashcat -m <mode> hashes.txt -a 3 ?u?l?l?l?l?l?d?d        # 8 chars: U L L L L L D D
hashcat -m <mode> hashes.txt -a 3 ?l?l?l?l?l?l?l?l        # 8 lowercase

# Hybrid (wordlist + mask suffix)
hashcat -m <mode> hashes.txt -a 6 rockyou.txt ?d?d?d?d    # word + 4 digits

# Hybrid (mask + wordlist prefix)
hashcat -m <mode> hashes.txt -a 7 ?d?d?d?d rockyou.txt    # 4 digits + word

# Show what was cracked
hashcat --show -m <mode> hashes.txt

# Resume an interrupted session
hashcat --restore --session=mysession

# Common masks
# ?l = a-z              ?u = A-Z              ?d = 0-9
# ?s = special chars    ?a = ?l?u?d?s         ?h = a-f0-9 (hex lower)
# ?H = A-F0-9 (hex up)  ?b = 0x00-0xff (raw byte)
```

## John Quick Invocation

```bash
# Standard
john --wordlist=rockyou.txt hash.txt
john --format=NT hash.txt --wordlist=rockyou.txt
john --format=raw-md5 hash.txt --wordlist=rockyou.txt
john --format=netntlmv2 capture.txt --wordlist=rockyou.txt
john --format=krb5tgs spn.txt --wordlist=rockyou.txt
john --format=krb5asrep asrep.txt --wordlist=rockyou.txt

# With rules
john --wordlist=rockyou.txt --rules=Single hash.txt
john --wordlist=rockyou.txt --rules=Wordlist hash.txt
john --wordlist=rockyou.txt --rules=Jumbo hash.txt

# Show cracked
john --show hash.txt
john --show --format=NT hash.txt

# Combine /etc/passwd and /etc/shadow into John format
unshadow /etc/passwd /etc/shadow > unshadowed.txt
john --wordlist=rockyou.txt unshadowed.txt
```

## Common file-to-hash converters

```bash
ssh2john id_rsa > id_rsa.hash                 # SSH key
zip2john secret.zip > zip.hash                 # ZIP password
rar2john secret.rar > rar.hash                 # RAR password
pdf2john secret.pdf > pdf.hash                 # PDF password
keepass2john Database.kdbx > kdbx.hash         # KeePass database
office2john document.docx > office.hash        # Office document password
luks2john disk.img > luks.hash                 # LUKS volume
truecrypt2john volume.tc > tc.hash             # TrueCrypt
1password2john *.opvault > 1pass.hash          # 1Password
ethereum2john wallet.json > eth.hash           # Ethereum wallet
bitcoin2john wallet.dat > btc.hash             # Bitcoin wallet
gpg2john secring.gpg > gpg.hash                # GPG keyring
mozilla2john key3.db > moz.hash                # Mozilla cert DB
hashcat-utils ...                              # WPA/etc.
```

## Specific scenarios

### Captured a NetNTLMv2 hash from Responder

```
admin::CORP:1122334455667788:7B8F1...0E:01010000000000...
```

Mode 5600. Save the entire line to a file.

```bash
hashcat -m 5600 hashes.txt /usr/share/wordlists/rockyou.txt
```

### Cracked a Kerberos TGS

After GetUserSPNs:
```
$krb5tgs$23$*svc_acc$DOMAIN.LOCAL$MSSQLSvc/sql.domain.local*$abc...
```

Mode 13100 (RC4). If you see `$krb5tgs$18$` instead, that's AES256 - mode 19700.

```bash
hashcat -m 13100 spn.txt /usr/share/wordlists/rockyou.txt
```

### AS-REP roast capture

```
$krb5asrep$23$user@DOMAIN.LOCAL:abc...
```

Mode 18200.

```bash
hashcat -m 18200 asrep.txt /usr/share/wordlists/rockyou.txt
```

### Linux /etc/shadow with `$6$` prefix

Mode 1800 (sha512crypt). These are slow - prioritise other paths if available.

### NTLM hash from secretsdump

```
Administrator:500:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
```

The first hash (LM) is empty here. The second (NTLM) is what you crack:

```bash
echo "31d6cfe0d16ae931b73c59d7e0c089c0" > nt.txt
hashcat -m 1000 nt.txt rockyou.txt
```

But often you don't need to crack - the NTLM hash IS the credential for pass-the-hash.

### Hash empty / well-known values

| Value | Plaintext |
|---|---|
| `aad3b435b51404eeaad3b435b51404ee` | Empty LM (always means "no LM hash stored") |
| `31d6cfe0d16ae931b73c59d7e0c089c0` | Empty NTLM (account has blank password!) |
| `5f4dcc3b5aa765d61d8327deb882cf99` | MD5 of "password" |
| `5baa61e4c9b93f3f0682250b6cf8331b7ee68fd8` | SHA-1 of "password" |
| `e10adc3949ba59abbe56e057f20f883e` | MD5 of "123456" |

If you see the empty NTLM, the user has a blank password - you can log in immediately.

## Hashcat Performance

Approximate cracking speeds on a single mid-range GPU (RTX 3080), for reference:

| Hash | Speed |
|---|---|
| MD5 | ~50 GH/s |
| NTLM | ~80 GH/s |
| SHA-1 | ~25 GH/s |
| SHA-256 | ~10 GH/s |
| SHA-512 | ~3 GH/s |
| md5crypt $1$ | ~12 MH/s |
| sha512crypt $6$ | ~150 KH/s |
| bcrypt cost 10 | ~25 KH/s |
| bcrypt cost 12 | ~6 KH/s |
| bcrypt cost 14 | ~1.5 KH/s |
| Kerberos AS-REP | ~600 MH/s |
| Kerberoast TGS-REP | ~600 MH/s |
| NetNTLMv2 | ~3 GH/s |

Implications:
- MD5 / NTLM / SHA-* on a wordlist: seconds for rockyou.
- Anything with a high cost (bcrypt 12+, sha512crypt with high rounds): only short passwords or weak ones reachable.
- Kerberos hashes are surprisingly fast - prioritise these if you find them.

In the exam VM (no GPU), expect speeds 100-1000x slower. Stick to wordlist attacks; mask attacks against complex passwords are not feasible CPU-only.

## Final Notes

- When a hash format is unfamiliar, run it through `hashid` or check `hashcat --help | grep -i <prefix>`.
- The `john --list=formats` command lists every format John supports.
- If hashcat refuses to recognise a hash, check it doesn't have spurious whitespace, BOM, or leading/trailing characters.
- Pre-compute the rockyou.txt + best64.rule combination - it cracks ~70% of common passwords in seconds for fast hashes.
