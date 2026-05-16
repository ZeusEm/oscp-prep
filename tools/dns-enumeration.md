# DNS Enumeration

## Objective

Actively enumerate DNS infrastructure to identify:

- Valid subdomains
- Mail infrastructure
- Internal naming conventions
- Additional IP ranges
- Potential attack surface

DNS enumeration is one of the highest-value early-stage reconnaissance activities in OSCP.

---

# DNS Record Types

| Record | Purpose | Example |
|---|---|---|
| A | Hostname → IPv4 | www.target.com |
| AAAA | Hostname → IPv6 | ipv6.target.com |
| MX | Mail infrastructure | mail.target.com |
| NS | Authoritative DNS servers | ns1.target.com |
| PTR | Reverse DNS lookup | 10.10.10.5 → mail.target.com |
| CNAME | Alias records | dev → server01 |
| TXT | Arbitrary text / SPF / verification | SPF, DKIM |

---

# OSCP DNS Enumeration Workflow

---

# Phase 1 — Basic DNS Resolution

## Resolve Main Host

```bash
host target.com
host www.target.com
```

---

# Phase 2 — Enumerate Core DNS Records

## MX Records

```bash
host -t mx target.com
```

### Purpose

Identify:

- Mail servers
- External providers
- Internal naming conventions
- Additional attack surface

---

## TXT Records

```bash
host -t txt target.com
```

### Purpose

Identify:

- SPF records
- Verification tokens
- Email infrastructure
- Hidden IPs
- Internal notes

---

## NS Records

```bash
host -t ns target.com
```

### Purpose

Identify:

- Authoritative DNS servers
- Infrastructure ownership
- Potential zone transfer targets

---

# Interpreting Common Findings

---

# Valid Host

```bash
host vpn.target.com
```

Example:

```text
vpn.target.com has address 10.10.10.15
```

Indicates:

- Valid DNS entry
- Potentially reachable service

---

# Invalid Host

```bash
host doesnotexist.target.com
```

Example:

```text
Host doesnotexist.target.com not found: 3(NXDOMAIN)
```

### NXDOMAIN

Indicates:

- DNS record does not exist

---

# Phase 3 — Manual Subdomain Brute Force

---

# Create Initial Wordlist

```bash
cat > list.txt << EOF
www
mail
vpn
dev
test
admin
intranet
router
proxy
owa
ftp
api
staging
backup
git
jira
EOF
```

---

# Manual Enumeration

```bash
for sub in $(cat list.txt); do
    host $sub.target.com
done
```

---

# Filter Valid Results Only

```bash
for sub in $(cat list.txt); do
    host $sub.target.com
done | grep -v "not found"
```

---

# High-Value Subdomains

| Hostname | Possible Value |
|---|---|
| admin | Admin portal |
| vpn | Remote access |
| intranet | Internal applications |
| mail | Mail infrastructure |
| owa | Outlook Web Access |
| dev | Development environment |
| test | Weakly secured systems |
| router | Network infrastructure |
| siem | Security tooling |
| snmp | Monitoring |
| syslog | Logging infrastructure |
| git | Source code repositories |
| jira | Internal ticketing |

---

# Phase 4 — Reverse DNS Lookups

## Purpose

Identify hostnames associated with discovered IP ranges.

---

# Reverse Lookup Single IP

```bash
host 10.10.10.15
```

Example:

```text
15.10.10.10.in-addr.arpa domain name pointer vpn.target.com
```

---

# Reverse Lookup Range

```bash
for ip in $(seq 1 254); do
    host 10.10.10.$ip
done | grep -v "not found"
```

---

# PTR Record Intelligence

PTR records frequently reveal:

- Internal naming conventions
- SIEM infrastructure
- Monitoring systems
- VPN gateways
- Backup systems
- Logging infrastructure

---

# Phase 5 — dnsrecon

---

# Standard Enumeration

```bash
dnsrecon -d target.com -t std
```

---

# What dnsrecon Enumerates

- SOA records
- NS records
- MX records
- TXT records
- SRV records
- DNSSEC configuration

---

# Important Findings

## DNSSEC Not Configured

Example:

```text
ERROR No answer for DNSSEC query
```

Meaning:

- DNSSEC not enabled
- Informational finding

---

## Bind Version Disclosure

Example:

```text
Bind Version for 10.10.10.2 "local"
```

Meaning:

- Version disclosure restricted
- Proper hardening

---

# Subdomain Brute Force

## Install SecLists

```bash
sudo apt install seclists
```

---

# Recommended OSCP Wordlist

```bash
/usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt
```

Best balance of:

- Speed
- Coverage
- Signal-to-noise ratio

---

# dnsrecon Brute Force

```bash
dnsrecon -d target.com \
-D /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt \
-t brt
```

---

# Important Observation

Example:

```text
INFO 1 Records Found
```

Possible meanings:

- Minimal public attack surface
- Hardened DNS exposure
- Limited external infrastructure

---

# Phase 6 — dnsenum

---

# Basic Usage

```bash
dnsenum target.com
```

---

# What dnsenum Performs

- Subdomain brute force
- Reverse lookups
- WHOIS extraction
- Range identification
- Zone transfer attempts

---

# Zone Transfer Testing

Example:

```text
AXFR record query failed: REFUSED
```

Meaning:

- Zone transfer blocked
- Proper DNS hardening

---

# Successful AXFR

If successful:

```text
Huge finding
```

Possible exposure:

- Entire DNS zone
- Internal hosts
- Internal naming structure
- Mail systems
- VPN gateways

---

# Class C Netblocks

Example:

```text
10.10.10.0/24
```

Useful for:

- Future Nmap scans
- Reverse lookups
- Host discovery

---

# Phase 7 — SPF / TXT Infrastructure Correlation

---

# Enumerate SPF Records

```bash
host -t txt target.com
```

Example:

```text
v=spf1 ip4:10.10.10.5 ip4:10.10.10.6 -all
```

---

# Important OSCP Insight

## No MX Record ≠ No Mail Infrastructure

Even without MX records:

- SPF may expose mail relays
- SMTP gateways may still exist
- Third-party providers may be used

Always enumerate TXT records.

---

# Extract Infrastructure IPs

Example:

```text
10.10.10.5
10.10.10.6
```

---

# Reverse Lookup SPF IPs

```bash
host 10.10.10.5
```

---

# What PTR Results Reveal

PTR records may expose:

- Vendor infrastructure
- Shared hosting
- Third-party providers
- Mail systems
- Internal service roles

---

# Shared Infrastructure Indicators

Multiple unrelated domains on same IP may indicate:

- Shared hosting
- Outsourced infrastructure
- Multi-tenant mail relays

---

# Phase 8 — Service Enumeration Pivot

---

# Service Detection

```bash
nmap -Pn -sV <IP>
```

---

# SMTP Enumeration

```bash
nmap --script smtp-enum-users <IP>
```

---

# Banner Grabbing

```bash
nc <IP> 25
```

---

# Phase 9 — Windows Enumeration

---

# nslookup

Windows-native DNS enumeration utility.

Useful during:

- Active Directory environments
- Restricted shells
- Living-off-the-land operations

---

# Basic Lookup

```powershell
nslookup target.com
```

---

# Query Specific DNS Server

```powershell
nslookup target.com 10.10.10.2
```

---

# MX Query

```powershell
nslookup -type=MX target.com
```

---

# NS Query

```powershell
nslookup -type=NS target.com
```

---

# TXT Query

```powershell
nslookup -type=TXT target.com
```

---

# PTR Query

```powershell
nslookup 10.10.10.15
```

---

# Important nslookup Observation

If no MX records exist:

```powershell
nslookup -type=MX target.com
```

may return SOA information instead.

This is normal behavior.

---

# OSCP DNS Enumeration Sequence

## Recommended Real-World Workflow

### 1. Resolve Target

```bash
host target.com
host www.target.com
```

---

### 2. Enumerate Core Records

```bash
host -t mx target.com
host -t txt target.com
host -t ns target.com
```

---

### 3. Standard dnsrecon Scan

```bash
dnsrecon -d target.com -t std
```

---

### 4. Subdomain Brute Force

```bash
dnsrecon -d target.com \
-D /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt \
-t brt
```

---

### 5. Automated Enumeration

```bash
dnsenum target.com
```

---

### 6. Reverse Lookup Discovered Ranges

```bash
for ip in $(seq 1 254); do
    host 10.10.10.$ip
done | grep -v "not found"
```

---

### 7. Extract SPF Infrastructure

```bash
host -t txt target.com
```

---

### 8. Reverse Lookup SPF IPs

```bash
host <spf_ip>
```

---

### 9. Enumerate Discovered Services

```bash
nmap -Pn -sV <IP>
```

---

# DNS Enumeration Mindset

DNS enumeration is cyclical.

Every discovery may reveal:

- New IP ranges
- Additional hosts
- Third-party providers
- VPN infrastructure
- Internal naming standards
- Additional attack surface

Continue iterating:

```text
Discover → Enumerate → Correlate → Expand Scope → Repeat
```

---

# OSCP Notes Checklist

Document all:

- A records
- MX records
- TXT records
- NS records
- PTR results
- Netblocks
- VPN hosts
- Mail infrastructure
- Third-party providers
- Internal naming conventions
- Security tooling hosts
- SPF IPs
- Zone transfer behavior

---

# Key OSCP Takeaways

## host

Best for:

- Quick manual enumeration
- Record validation
- Lightweight reconnaissance

---

## dnsrecon

Best for:

- Structured enumeration
- Fast automation
- Subdomain brute force

---

## dnsenum

Best for:

- Large-scale enumeration
- Reverse lookups
- Netblock discovery
- AXFR testing

---

## nslookup

Best for:

- Windows environments
- Active Directory
- LOLBAS-style operations

---

# Minimal Practical OSCP Command Set

```bash
host target.com
host -t mx target.com
host -t txt target.com
host -t ns target.com

dnsrecon -d target.com -t std

dnsrecon -d target.com \
-D /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt \
-t brt

dnsenum target.com

for sub in $(cat list.txt); do
    host $sub.target.com
done | grep -v "not found"

for ip in $(seq 1 254); do
    host 10.10.10.$ip
done | grep -v "not found"
```