# Common Port Reference

A quick lookup table of services by port. The first table is sorted by port number; the second is sorted by service category for finding ports when you know what you want to attack.

## By Port Number

| Port | TCP/UDP | Service | Notes |
|---|---|---|---|
| 7 | TCP/UDP | Echo | Diagnostic; rare |
| 13 | TCP/UDP | Daytime | Diagnostic; rare |
| 19 | TCP/UDP | Chargen | Char generator; DDoS amplification source |
| 20 | TCP | FTP-data | Active mode FTP data |
| 21 | TCP | FTP | File Transfer Protocol control |
| 22 | TCP | SSH | Secure Shell |
| 23 | TCP | Telnet | Cleartext interactive shell |
| 25 | TCP | SMTP | Mail relay/submission |
| 37 | TCP/UDP | Time | RFC 868 time protocol |
| 42 | TCP | WINS | Legacy Windows name resolution |
| 43 | TCP | WHOIS | Domain registration data |
| 49 | TCP/UDP | TACACS | Terminal access controller (auth) |
| 53 | TCP/UDP | DNS | UDP for queries, TCP for >512B and AXFR |
| 67 | UDP | DHCP server | Bootstrap protocol server |
| 68 | UDP | DHCP client | Bootstrap protocol client |
| 69 | UDP | TFTP | Trivial FTP - no auth, often Cisco config |
| 70 | TCP | Gopher | Used in SSRF for raw byte injection |
| 79 | TCP | Finger | User information disclosure |
| 80 | TCP | HTTP | World Wide Web (cleartext) |
| 88 | TCP | Kerberos | Authentication; **DC indicator** |
| 102 | TCP | MS Exchange | Older message API |
| 110 | TCP | POP3 | Post Office Protocol v3 |
| 111 | TCP/UDP | rpcbind | RPC portmapper - precedes NFS, NIS |
| 113 | TCP | ident | Authentication service (legacy) |
| 119 | TCP | NNTP | Network News Transfer |
| 123 | UDP | NTP | Network Time Protocol |
| 135 | TCP | MS RPC | Endpoint mapper - **Windows indicator** |
| 137 | UDP | NetBIOS-NS | NetBIOS Name Service |
| 138 | UDP | NetBIOS-DGM | NetBIOS Datagram |
| 139 | TCP | NetBIOS-SSN | SMB over NetBIOS (legacy) |
| 143 | TCP | IMAP | Internet Message Access Protocol |
| 161 | UDP | SNMP | Simple Network Management Protocol |
| 162 | UDP | SNMP-trap | SNMP trap receiver |
| 177 | UDP | XDMCP | X Display Manager Control |
| 179 | TCP | BGP | Border Gateway Protocol |
| 194 | TCP | IRC | Internet Relay Chat |
| 264 | TCP | Check Point FW1 | Check Point firewall management |
| 389 | TCP/UDP | LDAP | Directory access |
| 427 | TCP/UDP | SLP | Service Location Protocol |
| 443 | TCP | HTTPS | HTTP over TLS |
| 444 | TCP | SNPP | Simple Network Paging Protocol |
| 445 | TCP | SMB | SMB direct (modern) |
| 464 | TCP/UDP | kpasswd | Kerberos password change |
| 465 | TCP | SMTPS | SMTP over TLS (implicit) |
| 500 | UDP | IKE | IPsec Internet Key Exchange |
| 502 | TCP | Modbus | Industrial control protocol |
| 512 | TCP | rexec | Remote execution (cleartext) |
| 513 | TCP | rlogin | Remote login (cleartext) |
| 513 | UDP | rwho | Who is logged in |
| 514 | TCP | rsh | Remote shell (cleartext) |
| 514 | UDP | syslog | System logging |
| 515 | TCP | LPD | Line Printer Daemon |
| 520 | UDP | RIP | Routing Information Protocol |
| 521 | UDP | RIPng | RIP for IPv6 |
| 540 | TCP | UUCP | Unix-to-Unix Copy |
| 543 | TCP | klogin | Kerberos login |
| 544 | TCP | kshell | Kerberos shell |
| 548 | TCP | AFP | Apple Filing Protocol |
| 554 | TCP | RTSP | Real Time Streaming |
| 587 | TCP | SMTP submission | Mail client → server |
| 593 | TCP | RPC over HTTP | DCOM over HTTP |
| 623 | UDP | IPMI | Intelligent Platform Management |
| 631 | TCP | IPP | Internet Printing Protocol / CUPS |
| 636 | TCP | LDAPS | LDAP over TLS |
| 666 | TCP | doom | Often used by malware |
| 749 | TCP | Kerberos admin | Kerberos administration |
| 783 | TCP | SpamAssassin | Spam filter daemon |
| 853 | TCP | DNS-over-TLS | Encrypted DNS |
| 873 | TCP | rsync | rsync daemon |
| 902 | TCP | VMware authd | VMware ESXi authentication |
| 989/990 | TCP | FTPS | FTP over TLS |
| 992 | TCP | telnets | Telnet over TLS |
| 993 | TCP | IMAPS | IMAP over TLS |
| 995 | TCP | POP3S | POP3 over TLS |
| 1025-49151 | TCP | RPC dynamic ports | Windows RPC services |
| 1080 | TCP | SOCKS proxy | |
| 1098 | TCP | Java RMI | Remote Method Invocation registry |
| 1099 | TCP | Java RMI | Object Activation |
| 1194 | UDP | OpenVPN | VPN protocol |
| 1241 | TCP | Nessus | Nessus daemon (legacy) |
| 1352 | TCP | Lotus Notes | Notes RPC |
| 1433 | TCP | MSSQL | Microsoft SQL Server |
| 1434 | UDP | MSSQL Browser | Instance discovery |
| 1521 | TCP | Oracle TNS | Oracle Net Services |
| 1604 | UDP | Citrix ICA | Citrix client |
| 1701 | UDP | L2TP | Layer 2 Tunneling |
| 1723 | TCP | PPTP | Point-to-Point Tunneling |
| 1812 | UDP | RADIUS auth | Authentication |
| 1813 | UDP | RADIUS acct | Accounting |
| 1883 | TCP | MQTT | IoT messaging |
| 1900 | UDP | SSDP/UPnP | Service discovery |
| 2000 | TCP | Cisco SCCP | Skinny Client Control |
| 2049 | TCP/UDP | NFS | Network File System |
| 2080-2083 | TCP | (various) | Sometimes web |
| 2100 | TCP | Oracle XML DB | |
| 2181 | TCP | ZooKeeper | |
| 2222 | TCP | (alt SSH) | Common alternate SSH port |
| 2375 | TCP | Docker REST | **Cleartext, no auth by default** |
| 2376 | TCP | Docker REST | TLS-enabled |
| 2483/2484 | TCP | Oracle DB SSL | |
| 2525 | TCP | (alt SMTP) | Common alternate SMTP |
| 2638 | TCP | Sybase | |
| 3000 | TCP | (various) | Grafana, Node.js, GitLab pages |
| 3128 | TCP | Squid proxy | |
| 3260 | TCP | iSCSI | |
| 3268 | TCP | LDAP GC | Global Catalog (AD) |
| 3269 | TCP | LDAPS GC | Global Catalog over TLS (AD) |
| 3306 | TCP | MySQL | |
| 3389 | TCP/UDP | RDP | Remote Desktop Protocol |
| 3478 | UDP | STUN | NAT traversal |
| 3632 | TCP | distccd | Distributed compiler - RCE if exposed |
| 3690 | TCP | SVN | Subversion server |
| 3702 | UDP | WS-Discovery | Web Services Dynamic Discovery |
| 3724 | TCP | World of Warcraft | Game |
| 3780 | TCP | Nessus | Newer Nessus port |
| 3784 | TCP | Bfd-control | |
| 3790 | TCP | Quick-im | |
| 4000 | TCP | Various | Sometimes used by Diablo II/games |
| 4040 | TCP | Yahoo IM | |
| 4369 | TCP | epmd | Erlang Port Mapper Daemon |
| 4444 | TCP | Metasploit | Default Meterpreter port (and reverse shells) |
| 4505/4506 | TCP | Salt master | SaltStack control channel |
| 4664 | TCP | Google Desktop | |
| 4786 | TCP | Cisco SMI | Smart Install (often abusable) |
| 4848 | TCP | GlassFish admin | Java app server admin |
| 4899 | TCP | Radmin | Remote admin (Win) |
| 5000 | TCP | UPnP / Flask | |
| 5001 | TCP | Slingbox | |
| 5050 | TCP | Yahoo Messenger | |
| 5060 | TCP/UDP | SIP | VoIP signalling |
| 5061 | TCP | SIP-TLS | |
| 5222 | TCP | XMPP client | Jabber |
| 5269 | TCP | XMPP server | Jabber server-to-server |
| 5353 | UDP | mDNS | Multicast DNS / Bonjour / Avahi |
| 5355 | UDP | LLMNR | Link-Local Multicast Name Resolution |
| 5432 | TCP | PostgreSQL | |
| 5500 | TCP | VNC reverse | VNC reverse connection |
| 5555 | TCP | Various | HP Data Protector, ADB (Android) |
| 5601 | TCP | Kibana | Elasticsearch UI |
| 5666 | TCP | NRPE | Nagios Remote Plugin Executor |
| 5672 | TCP | AMQP | RabbitMQ |
| 5800-5806 | TCP | VNC HTTP | VNC over HTTP |
| 5900-5906 | TCP | VNC | Virtual Network Computing |
| 5938 | TCP | TeamViewer | |
| 5984 | TCP | CouchDB | |
| 5985 | TCP | WinRM HTTP | Windows Remote Management |
| 5986 | TCP | WinRM HTTPS | Windows Remote Management TLS |
| 6000-6063 | TCP | X11 | X Window System |
| 6379 | TCP | Redis | Often unauthenticated |
| 6443 | TCP | Kubernetes API | |
| 6543 | TCP | Pyramid (Python) | |
| 6660-6669 | TCP | IRC | |
| 6697 | TCP | IRC over TLS | |
| 7000-7002 | TCP | WebLogic | Oracle WebLogic |
| 7077 | TCP | Spark | Apache Spark master |
| 7080 | TCP | EmpowerID | |
| 7100 | TCP | X font server | |
| 7218 | TCP | (FTP) | Sometimes |
| 7474 | TCP | Neo4j HTTP | Graph database |
| 7687 | TCP | Neo4j Bolt | |
| 8000 | TCP | (various) | Splunk, dev servers |
| 8001 | TCP | (various) | |
| 8008 | TCP | HTTP-alt | |
| 8009 | TCP | Apache AJP | **Ghostcat (CVE-2020-1938)** |
| 8014 | TCP | (various) | |
| 8025 | TCP | (alt SMTP) | |
| 8060 | TCP | (various) | |
| 8080 | TCP | HTTP-alt | Tomcat, Jenkins, generic |
| 8081 | TCP | (various) | MongoDB Express |
| 8086 | TCP | InfluxDB | |
| 8088 | TCP | (various) | Hadoop ResourceManager |
| 8089 | TCP | Splunk forwarder | |
| 8090 | TCP | Confluence | Atlassian |
| 8112 | TCP | Deluge | |
| 8161 | TCP | ActiveMQ | |
| 8200 | TCP | Vault | HashiCorp Vault |
| 8333 | TCP | Bitcoin | |
| 8400 | TCP | cvd | |
| 8443 | TCP | HTTPS-alt | Tomcat HTTPS |
| 8500 | TCP | Consul | HashiCorp Consul |
| 8530 | TCP | WSUS HTTP | Windows Server Update Services |
| 8531 | TCP | WSUS HTTPS | |
| 8649 | TCP | Ganglia | |
| 8666 | TCP | Various | |
| 8834 | TCP | Nessus web | Modern Nessus UI |
| 8880 | TCP | (various) | WebSphere |
| 8888 | TCP | (various) | Jupyter, dev |
| 9000 | TCP | (various) | PHP-FPM, SonarQube |
| 9001 | TCP | HSQLDB / Tor | |
| 9042 | TCP | Cassandra | |
| 9080 | TCP | WebSphere HTTP | |
| 9090 | TCP | (various) | Cockpit, Prometheus |
| 9091 | TCP | (various) | |
| 9092 | TCP | Kafka | |
| 9100 | TCP | JetDirect | Network printing |
| 9200 | TCP | Elasticsearch | HTTP API |
| 9300 | TCP | Elasticsearch | Transport (cluster) |
| 9418 | TCP | git | Git daemon |
| 9999 | TCP | (various) | Often custom apps |
| 10000 | TCP | Webmin | Linux web admin |
| 10243 | TCP | WMF | Windows Media Format |
| 11211 | TCP/UDP | Memcached | Often no auth |
| 11214/11215 | TCP | Memcached SSL | |
| 12345 | TCP | NetBus | Trojan |
| 13720/13721 | TCP | NetBackup | |
| 15672 | TCP | RabbitMQ Mgmt | |
| 17070 | TCP | iLO | HP iLO XML |
| 17988 | TCP | iLO Virtual Console | |
| 18091 | TCP | Couchbase | |
| 22000 | TCP | (alt SSH) | |
| 25565 | TCP | Minecraft | |
| 27015 | UDP | Source engine games | |
| 27017/27018 | TCP | MongoDB | |
| 28015 | TCP | RethinkDB | |
| 31337 | TCP | Back Orifice | Trojan / 1337 reference |
| 32400 | TCP | Plex | |
| 50030 | TCP | Hadoop JobTracker | |
| 50050 | TCP | Cobalt Strike | C2 default |
| 50070 | TCP | Hadoop NameNode | |
| 50443 | TCP | (Cobalt Strike alt) | |
| 50461 | TCP | (various) | |
| 54321 | UDP | Bo2k | Trojan |
| 60000+ | TCP | Mosh | Mobile shell |

## By Service Category

### Domain Controller / Active Directory

```
TCP 53, 88, 135, 139, 389, 445, 464, 593, 636, 3268, 3269, 5985
UDP 53, 88, 123, 137, 138, 389
```

If most/all of these are open on a host, it is a Domain Controller. If only TCP 135, 139, 445 are open, it is a member server or workstation.

### Web

```
80, 443                      # standard
81, 591, 8000, 8008, 8080, 8081, 8088, 8888    # alternate HTTP
8443, 4443, 9443             # alternate HTTPS
3000                         # Grafana, Node.js
4848                         # GlassFish
5000, 5001                   # Flask, .NET dev
5601                         # Kibana
7001, 7002                   # WebLogic
7474                         # Neo4j
8009                         # Tomcat AJP (Ghostcat CVE-2020-1938)
8086                         # InfluxDB
8090                         # Confluence
8161                         # ActiveMQ
8530, 8531                   # WSUS
9090, 9091                   # Cockpit, Prometheus
10000                        # Webmin
```

### Databases

```
1433     MSSQL
1434/UDP MSSQL Browser
1521     Oracle TNS
3306     MySQL / MariaDB
5432     PostgreSQL
5984     CouchDB
6379     Redis (often unauthenticated!)
7000     Cassandra cluster
9042     Cassandra CQL
9200     Elasticsearch HTTP
9300     Elasticsearch transport
11211    Memcached (often unauthenticated!)
27017    MongoDB
28015    RethinkDB
50000    DB2
```

### Remote Access / Management

```
22       SSH
23       Telnet (cleartext)
512/513/514  r-services (rexec/rlogin/rsh - cleartext)
3389     RDP
5800-5806    VNC over HTTP
5900-5906    VNC
5985/5986    WinRM HTTP/HTTPS
4899     Radmin
5938     TeamViewer
10000    Webmin
```

### Mail

```
25       SMTP
110      POP3
143      IMAP
465      SMTPS (legacy implicit TLS)
587      SMTP submission (with STARTTLS)
993      IMAPS
995      POP3S
```

### File Transfer / Sharing

```
21       FTP control
20       FTP data (active mode)
69/UDP   TFTP
115      SFTP
139      SMB over NetBIOS
445      SMB direct
873      rsync
989/990  FTPS
2049     NFS
3260     iSCSI
4877     ?
```

### DNS / Name Resolution

```
53       DNS
137/UDP  NetBIOS-NS
138/UDP  NetBIOS-DGM
5353/UDP mDNS
5355/UDP LLMNR
```

### Routing / Network Management

```
161/UDP  SNMP query
162/UDP  SNMP trap
179      BGP
500/UDP  IKE / IPsec
1812/UDP RADIUS auth
1813/UDP RADIUS acct
4500/UDP IPsec NAT-T
4789/UDP VXLAN
```

### Containers / Orchestration

```
2375     Docker REST (cleartext - DANGER)
2376     Docker REST TLS
2379     etcd
2380     etcd peer
6443     Kubernetes API
8080     Kubernetes (legacy insecure port)
8443     Kubernetes Dashboard
10250    Kubelet
10255    Kubelet read-only (deprecated)
50051    gRPC
```

### CI/CD / DevOps

```
8080     Jenkins
3000     GitLab pages, Grafana
4000     Phabricator
5984     CouchDB
7990     Bitbucket
8000     Splunk Enterprise
8089     Splunk forwarder
8200     Vault
8500     Consul
8161     ActiveMQ
8181     (various)
9092     Kafka
9418     Git daemon
```

### IoT / Embedded

```
554      RTSP (cameras)
1883     MQTT
2323     Telnet (alt)
5683     CoAP
8554     RTSP-alt
9100     JetDirect (printers)
49152+   UPnP / Roku / Sonos
```

### VoIP

```
1720     H.323 (call setup)
2000     Cisco SCCP (Skinny)
4569     IAX2
5060     SIP (cleartext)
5061     SIP-TLS
10000-20000  RTP (media)
```

### Industrial / SCADA

```
102      Siemens S7
502      Modbus
1089-1091  Foundation Fieldbus
2222     EtherNet/IP
4840     OPC UA
20000    DNP3
44818    EtherNet/IP explicit
47808    BACnet
```

## Port Discovery Tips

### Default nmap port set vs full scan

```bash
# Default top 1000 - fast but misses non-standard ports
nmap -sCV 10.10.10.50

# Full TCP - find services on weird ports
nmap -sS -p- --min-rate 5000 10.10.10.50

# UDP top 100 - UDP is slow so use top
nmap -sU --top-ports 100 10.10.10.50

# Specifically for AD discovery
nmap -p 53,88,135,139,389,445,464,593,636,3268,3269,5985 10.10.10.0/24
```

### Quick web port sweep

```bash
nmap -p 80,81,443,591,4848,5000,7001,7002,8000,8008,8009,8080,8081,8088,8090,8161,8443,8888,9000,9090,9443,10000 10.10.10.50
```

### Service identification when port is unusual

```bash
# Banner only
nc -nv 10.10.10.50 4567

# nmap version detection on a specific port
nmap -sV -p 4567 --version-all 10.10.10.50

# AmAP (more aggressive than nmap -sV - rarely needed)
amap -bqv 10.10.10.50 4567
```

## High-Value Default-Cred Ports

The ports most worth checking with default credentials first:

| Port | Service | Common defaults |
|---|---|---|
| 22 | SSH | root/root, root/toor, root:(blank), msfadmin/msfadmin |
| 23 | Telnet | admin/admin, root:(blank), Cisco: cisco/cisco |
| 1433 | MSSQL | sa:(blank), sa/sa, sa/Password1 |
| 3306 | MySQL | root:(blank), root/root |
| 5432 | PostgreSQL | postgres/postgres, postgres:(blank) |
| 6379 | Redis | (no auth by default) |
| 9200 | Elasticsearch | (no auth historically) |
| 27017 | MongoDB | (no auth historically) |
| 8080 | Tomcat | tomcat/tomcat, admin/admin |
| 8080 | Jenkins | admin/admin (often anonymous read) |
| 8443 | various | admin/admin |
| 10000 | Webmin | (system root credentials) |
| 5985 | WinRM | (any AD user, often) |
| 2375 | Docker | (no auth!) |
| 3389 | RDP | administrator:(blank), Guest |
| 5900 | VNC | password "password" or 8-char weak |

When you scan a network and see any of the unauthenticated services (2375 Docker, 6379 Redis, 9200 Elasticsearch, 11211 Memcached, 27017 MongoDB), they are almost always misconfigured and often the fastest path to compromise.
