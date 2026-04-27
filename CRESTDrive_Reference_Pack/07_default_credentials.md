# Default Credentials Reference

A reference of default credentials for products commonly encountered in penetration tests. **Always try these first** before brute-forcing - they are responsible for an embarrassing fraction of real-world compromises.

## Top 25 Combinations to Try Universally

When confronted with any login prompt and no specific knowledge of the product, try these in roughly this order:

```
admin:admin
admin:password
admin:(blank)
root:root
root:toor
root:(blank)
administrator:administrator
administrator:(blank)
administrator:password
admin:Password1
admin:Password123
admin:Welcome1
admin:changeme
guest:guest
test:test
demo:demo
user:user
operator:operator
support:support
service:service
manager:manager
sysadmin:sysadmin
public:public
private:private
admin:admin123
```

The first three (`admin:admin`, `admin:password`, `admin:(blank)`) succeed surprisingly often.

---

## Network Devices

### Cisco

| Username | Password | Product |
|---|---|---|
| (none) | cisco | Standard IOS console |
| cisco | cisco | Default web admin on many devices |
| admin | admin | Various |
| admin | (blank) | Some default configs |
| root | attack | Some Cisco WLAN devices |
| (none) | admin | Various |
| Cisco | Cisco | Some IOS XE devices |
| admin | Cisco | Some CMS |

### Juniper

| Username | Password | Product |
|---|---|---|
| root | (blank) | JunOS factory default |
| netscreen | netscreen | NetScreen / SSG |

### HP

| Username | Password | Product |
|---|---|---|
| Administrator | (random) | iLO (printed on tag) |
| admin | admin | ProCurve switches |
| manager | (blank) | ProCurve old default |
| operator | (blank) | ProCurve operator account |
| admin | password | Various |
| anonymous | (blank) | JetDirect web admin |
| HP | (blank) | OpenView |

### Dell

| Username | Password | Product |
|---|---|---|
| root | calvin | DRAC / iDRAC |
| Administrator | calvin | Some iDRAC versions |
| admin | admin | Dell Sonicwall (older) |

### Mikrotik

| Username | Password | Product |
|---|---|---|
| admin | (blank) | RouterOS factory default |

### Ubiquiti

| Username | Password | Product |
|---|---|---|
| ubnt | ubnt | UniFi APs, AirOS |

### TP-Link

| Username | Password | Product |
|---|---|---|
| admin | admin | Most consumer routers |

### Netgear

| Username | Password | Product |
|---|---|---|
| admin | password | Most routers |
| admin | 1234 | Some older models |

### Linksys

| Username | Password | Product |
|---|---|---|
| (blank) | admin | Older routers |
| admin | admin | Modern |

### D-Link

| Username | Password | Product |
|---|---|---|
| admin | (blank) | Many models |
| admin | admin | Some newer |
| Admin | (blank) | Some |

### Fortinet

| Username | Password | Product |
|---|---|---|
| admin | (blank) | FortiGate factory default |
| admin | password | Some configs |
| maintainer | (model serial) | Recovery account |

### Palo Alto Networks

| Username | Password | Product |
|---|---|---|
| admin | admin | PAN-OS factory default |

### SonicWall

| Username | Password | Product |
|---|---|---|
| admin | password | Most appliances |

### F5 Networks

| Username | Password | Product |
|---|---|---|
| admin | admin | BIG-IP factory default |
| root | default | BIG-IP root shell |

---

## Databases

### Microsoft SQL Server

| Username | Password | Notes |
|---|---|---|
| sa | (blank) | Old default; still found |
| sa | sa | Common |
| sa | password | Common |
| sa | Password1 | |
| sa | sa@123 | |
| sa | admin | |
| (no SQL auth) | (Windows auth) | Modern installs default to Windows-only auth |

### MySQL / MariaDB

| Username | Password | Notes |
|---|---|---|
| root | (blank) | Old default |
| root | root | |
| root | toor | |
| root | password | |
| root | mysql | |
| mysql | mysql | |
| admin | admin | Some panels |

### Oracle (extensive default credentials list)

| Username | Password | Notes |
|---|---|---|
| SYS | change_on_install | The classic |
| SYS | manager | |
| SYSTEM | manager | The other classic |
| SYSTEM | oracle | |
| SCOTT | tiger | Demo schema |
| OUTLN | OUTLN | Stored outlines |
| HR | HR | Sample HR schema |
| OE | OE | Sample OE schema |
| SH | SH | Sample SH schema |
| MDSYS | MDSYS | Spatial / multimedia |
| CTXSYS | CTXSYS | Text indexing |
| ORDPLUGINS | ORDPLUGINS | Multimedia plugins |
| ORDSYS | ORDSYS | Multimedia |
| WKSYS | WKSYS | Workflow |
| WMSYS | WMSYS | Workspace manager |
| DBSNMP | DBSNMP | DB management |
| SYSMAN | OEM_TEMP | Enterprise Manager |
| ANONYMOUS | ANONYMOUS | XML DB |
| APEX_PUBLIC_USER | APEX_PUBLIC_USER | APEX |
| FLOWS_FILES | FLOWS_FILES | APEX |
| DIP | DIP | Directory integration |
| ORACLE_OCM | ORACLE_OCM | Configuration manager |
| LBACSYS | LBACSYS | Label security |
| TSMSYS | TSMSYS | Transparent session migration |
| XDB | (varies) | XML DB |
| MGMT_VIEW | (varies) | EM management view |
| NETWORK | NETWORK | |
| PERFSTAT | PERFSTAT | Statspack |

### PostgreSQL

| Username | Password | Notes |
|---|---|---|
| postgres | postgres | Common |
| postgres | (blank) | Default trust auth |
| postgres | password | |
| postgres | admin | |

### MongoDB

| Username | Password | Notes |
|---|---|---|
| (none) | (none) | **Default historically had no auth at all** |
| admin | admin | If auth enabled |
| root | root | |

### Redis

| Username | Password | Notes |
|---|---|---|
| (none) | (none) | **Default no auth** |
| (none) | foobared | Old default in some packages |

### CouchDB

| Username | Password | Notes |
|---|---|---|
| admin | admin | If admin set up |
| (none) | (none) | "Admin party" default - anyone is admin |

### Cassandra

| Username | Password | Notes |
|---|---|---|
| cassandra | cassandra | Default |

### Memcached

(No authentication; just check if reachable)

### InfluxDB

| Username | Password | Notes |
|---|---|---|
| (none) | (none) | v1 default has no auth |
| admin | admin | If enabled |

### Elasticsearch

| Username | Password | Notes |
|---|---|---|
| (none) | (none) | Default historically no auth |
| elastic | changeme | When X-Pack added auth |
| elastic | elastic | Some configs |

---

## Web Applications

### Tomcat (Manager / Host Manager)

| Username | Password | Notes |
|---|---|---|
| tomcat | tomcat | Documentation example |
| tomcat | s3cret | Documentation example |
| admin | admin | Common |
| admin | password | Common |
| admin | tomcat | Common |
| manager | manager | |
| role1 | tomcat | Documentation example |
| both | tomcat | Documentation example |

### Jenkins

| Username | Password | Notes |
|---|---|---|
| admin | admin | Sometimes |
| admin | password | |
| (anonymous read often default) | | Look for /asynchPeople, /script |
| jenkins | jenkins | Some installs |

### JBoss

| Username | Password | Notes |
|---|---|---|
| admin | admin | Common |
| jboss | jboss | |
| admin | password | |

### WebLogic

| Username | Password | Notes |
|---|---|---|
| weblogic | weblogic | Old default |
| weblogic | weblogic1 | |
| weblogic | welcome1 | |
| system | password | |
| system | weblogic | |

### WebSphere

| Username | Password | Notes |
|---|---|---|
| admin | admin | |
| websphere | websphere | |
| (no auth on old default config) | | |

### GlassFish

| Username | Password | Notes |
|---|---|---|
| admin | admin | |
| admin | (blank) | Old default |
| admin | glassfish | |

### Apache HTTP Server (.htaccess defaults vary - no universal)

### nginx (no default auth)

### Splunk

| Username | Password | Notes |
|---|---|---|
| admin | changeme | Default first-time login |
| admin | admin | Sometimes left |
| admin | password | |

### Grafana

| Username | Password | Notes |
|---|---|---|
| admin | admin | First-time setup |

### Kibana

| Username | Password | Notes |
|---|---|---|
| (none) | (none) | If Elasticsearch unauth |
| elastic | changeme | If X-Pack |
| kibana | changeme | |

### RabbitMQ

| Username | Password | Notes |
|---|---|---|
| guest | guest | Default - localhost only by default but often misconfigured |

### ActiveMQ

| Username | Password | Notes |
|---|---|---|
| admin | admin | Web console |

### Apache Solr

| Username | Password | Notes |
|---|---|---|
| (none) | (none) | Often unauthenticated |
| solr | SolrRocks | |

### phpMyAdmin

| Username | Password | Notes |
|---|---|---|
| root | (blank) | If MySQL root has no password |
| root | (matches MySQL root) | |
| pma | pma | Some installs |

### Webmin

| Username | Password | Notes |
|---|---|---|
| (matches system root) | | Uses Linux user accounts |
| admin | admin | If created |

### Cacti

| Username | Password | Notes |
|---|---|---|
| admin | admin | Default first login |

### Nagios

| Username | Password | Notes |
|---|---|---|
| nagiosadmin | nagiosadmin | |
| nagios | nagios | |
| admin | admin | |

### Nessus

| Username | Password | Notes |
|---|---|---|
| (set during install) | | No factory default |

### OpenVAS / Greenbone

| Username | Password | Notes |
|---|---|---|
| admin | admin | First setup |

### Zabbix

| Username | Password | Notes |
|---|---|---|
| Admin | zabbix | First default |

### Cobalt Strike Team Server

| Username | Password | Notes |
|---|---|---|
| (set during install) | | No default |

### Metasploit (msfrpcd)

| Username | Password | Notes |
|---|---|---|
| msf | (set during install) | No default |

### Drupal

| Username | Password | Notes |
|---|---|---|
| admin | admin | First-time |

### WordPress

| Username | Password | Notes |
|---|---|---|
| admin | admin | First setup, often left |
| admin | password | |
| admin | <site name> | Common |

### Joomla

| Username | Password | Notes |
|---|---|---|
| admin | admin | First setup |
| admin | password | |
| admin | <site name> | |

### Magento

| Username | Password | Notes |
|---|---|---|
| admin | password | |
| admin | admin123 | |
| admin | magento | |

---

## Containers and Orchestration

### Kubernetes

| Username | Password | Notes |
|---|---|---|
| (depends) | | Use service account tokens; check `~/.kube/config` |

### Docker Registry

| Username | Password | Notes |
|---|---|---|
| (none) | (none) | If unauthenticated registry |

### Portainer

| Username | Password | Notes |
|---|---|---|
| admin | admin | First setup |
| admin | password | |

### Rancher

| Username | Password | Notes |
|---|---|---|
| admin | admin | First setup |

---

## Repositories / DevOps

### GitLab

| Username | Password | Notes |
|---|---|---|
| root | 5iveL!fe | Old default until 8.17 |
| root | (set on first login) | Modern |
| root | password | Sometimes |

### GitHub Enterprise

| Username | Password | Notes |
|---|---|---|
| (set during setup) | | No factory default |

### Bitbucket

| Username | Password | Notes |
|---|---|---|
| (set during setup) | | |

### Atlassian Jira

| Username | Password | Notes |
|---|---|---|
| admin | admin | Documentation example |

### Atlassian Confluence

| Username | Password | Notes |
|---|---|---|
| admin | admin | Documentation example |

### SonarQube

| Username | Password | Notes |
|---|---|---|
| admin | admin | First setup |

### Nexus Repository Manager

| Username | Password | Notes |
|---|---|---|
| admin | admin123 | OSS default until 3.17 |
| admin | (in /opt/sonatype-work/nexus3/admin.password file) | Modern - random password generated |

### JFrog Artifactory

| Username | Password | Notes |
|---|---|---|
| admin | password | First setup |

---

## Mail Servers

### Postfix / Sendmail / Exim

(No default credentials; uses system accounts)

### Zimbra

| Username | Password | Notes |
|---|---|---|
| admin@<domain> | (set during install) | |

### iRedMail

| Username | Password | Notes |
|---|---|---|
| postmaster@<domain> | (set during install) | |

---

## Embedded Devices / IoT

### Generic IP cameras

| Username | Password | Notes |
|---|---|---|
| admin | admin | |
| admin | (blank) | |
| admin | 12345 | Common Hikvision |
| admin | 888888 | Some Dahua |
| root | root | |
| root | pass | |
| service | service | |

### Network printers (HP, Canon, Epson, etc.)

| Username | Password | Notes |
|---|---|---|
| admin | (blank) | Common |
| admin | admin | |
| admin | password | |
| operator | (blank) | |

### Brocade / Foundry switches

| Username | Password | Notes |
|---|---|---|
| admin | admin | |
| admin | password | |

### Arista switches

| Username | Password | Notes |
|---|---|---|
| admin | (blank) | |

### Polycom phones

| Username | Password | Notes |
|---|---|---|
| Polycom | 456 | |
| 456 | (blank) | |

---

## Storage / NAS

### Synology DSM

| Username | Password | Notes |
|---|---|---|
| admin | (blank) | First setup |

### QNAP NAS

| Username | Password | Notes |
|---|---|---|
| admin | admin | First setup |

### NetApp ONTAP

| Username | Password | Notes |
|---|---|---|
| admin | (set during install) | |

### EMC Isilon

| Username | Password | Notes |
|---|---|---|
| root | a | Some old Isilon |

---

## Hypervisors / Virtualization

### VMware ESXi

| Username | Password | Notes |
|---|---|---|
| root | (set during install) | No default password |

### vCenter

| Username | Password | Notes |
|---|---|---|
| administrator@vsphere.local | (set during install) | |

### Citrix XenServer

| Username | Password | Notes |
|---|---|---|
| root | (set during install) | |

### Proxmox VE

| Username | Password | Notes |
|---|---|---|
| root | (system root password) | Uses Linux PAM |

---

## SAN / Backup

### Veeam Backup & Replication

| Username | Password | Notes |
|---|---|---|
| (system admin) | | Uses Windows accounts |

### Commvault

| Username | Password | Notes |
|---|---|---|
| admin | (set during install) | |

### NetBackup

| Username | Password | Notes |
|---|---|---|
| (system admin) | | |

---

## ICS / SCADA

| Product | Username | Password |
|---|---|---|
| Siemens S7 | (default no auth or hardcoded) | |
| Schneider Modicon | USER / USER | |
| Allen Bradley | administrator / (blank) | |
| Wonderware | wwAdmin / wwAdmin | |
| Rockwell | guest / (blank) | |

---

## Methodology Notes

### When to try defaults

1. **Before any brute force** - defaults take seconds, brute force can lock accounts.
2. **When you identify a product** - search this reference (or PayloadsAllTheThings, or specific product docs) for that product's defaults.
3. **When facing a login prompt with no rate limiting** - cycle through the top 25.
4. **As post-exploitation pivot** - if you compromise a host running a service, that service may use the same default the admin never changed.

### Default-credential-prone services

These services account for the majority of default-credential findings in the wild:

- **Network device admin interfaces** (Cisco, HP, etc.) - often unchanged after install
- **Database servers** (MSSQL `sa`, MySQL `root`, Oracle SYS/SYSTEM) - "I'll change it later"
- **CI/CD systems** (Jenkins, Tomcat) - test/dev credentials in production
- **Monitoring** (Splunk, Grafana, Kibana, Cacti, Nagios) - often considered "internal"
- **Backup systems** - typically very privileged when compromised
- **Hypervisor management** (vCenter, iLO, iDRAC, IPMI) - lights-out management often forgotten

### Mass testing with NetExec/CrackMapExec

```bash
# SSH default credentials sweep
nxc ssh 10.10.10.0/24 -u root -p root --continue-on-success
nxc ssh 10.10.10.0/24 -u admin -p admin --continue-on-success

# Build a list and iterate
echo "admin
root
sa
postgres
guest
test" > users.txt

echo "admin
password
(blank)
root
toor
sa
12345
admin123" > passwords.txt

nxc ssh 10.10.10.0/24 -u users.txt -p passwords.txt --continue-on-success
```

### When defaults don't work

- Some products force password change on first login (especially modern installs)
- Some store defaults in a generated file (Nexus 3.17+, GitLab modern)
- Some require both factor authentication

If defaults fail, fall back to:
1. Username enumeration via the service
2. Password spraying common patterns (`Spring2025!`, `<Company>2025!`)
3. Examination of related systems for credential reuse
4. The product's documentation for newer default mechanisms

### A note on responsible disclosure

When pen testing finds default credentials in production systems, the finding writeup should be specific about:
- Which credentials were tested
- Which credentials worked
- The privilege level granted
- Recommended remediation (change password, restrict access, force change-on-login)

Default credentials are typically the easiest finding to fix and the easiest finding to verify is fixed - prioritise them in the report's "Critical / Quick Wins" section.
