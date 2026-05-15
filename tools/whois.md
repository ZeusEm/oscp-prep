# WHOIS Enumeration

## Purpose

WHOIS is a TCP-based protocol/service (port 43) used to retrieve:

- domain registration information
    
- registrar details
    
- registrant information
    
- name servers
    
- netblock ownership
    
- hosting/provider details
    

Useful during:

- passive reconnaissance
    
- OSINT
    
- attack surface expansion
    

---

# When to Use

Use during:

- initial reconnaissance
    
- domain-based assessments
    
- external pentests
    
- infrastructure profiling
    
- IP ownership analysis
    

---

# Basic Usage

## Domain Lookup

```bash
whois <domain>
```

Example:

```bash
whois megacorpone.com
```

---

## Specify WHOIS Server

```bash
whois <target> -h <server>
```

Example:

```bash
whois megacorpone.com -h 192.168.50.251
```

### Why This Matters

Some environments:

- use internal WHOIS servers
    
- return different datasets
    
- simulate enterprise infrastructure
    

PWK labs use custom/internal services frequently.

---

# Reverse Lookup (IP → Ownership)

```bash
whois <IP>
```

Example:

```bash
whois 38.100.193.70
```

OR

```bash
whois 8.8.8.8
```

## Reverse IP Lookup Intelligence

Reverse WHOIS on IP addresses often reveals more useful information than domain WHOIS.

Useful findings include:
- netblocks
- hosting providers
- ASN
- infrastructure ownership
- support emails
- abuse contacts
- physical datacenter locations

Example:

`whois <IP>`

Recon mindset:
One IP may reveal an entire infrastructure range.

---

# Important Output Fields

## Domain Enumeration

Focus on extracting:

|Field|Value|
|---|---|
|Registrant Name|employee names|
|Organization|target company|
|Email|username generation|
|Phone Number|OSINT|
|Name Servers|DNS pivot|
|Registrar|infrastructure insight|

---

## IP Reverse Lookup

Focus on:

|Field|Value|
|---|---|
|NetRange|scan scope|
|CIDR|subnet identification|
|OrgName|hosting provider|
|Address|geographic info|
|Abuse Contacts|attribution|

---

# Real OSCP/Pentest Value

## Registrant Names

Useful for:

- username generation
    
- password spraying
    
- phishing
    
- credential attacks
    

Example:

```text
Alan Grofield
```

Potential usernames:

```text
agrofield
alan.grofield
alan
```

---

## Name Servers

Example:

```text
NS1.MEGACORPONE.COM
```

Useful for:

- DNS enumeration
    
- subdomain discovery
    
- zone transfer attempts
    

---

## NetRange / CIDR

Example:

```text
38.0.0.0/8
```

Useful for:

- expanding scan scope
    
- identifying owned infrastructure
    
- discovering additional targets
    

---

## Hosting Provider Discovery

Example:

```text
OrgName: PSINet, Inc.
```

Useful for:

- understanding target hosting
    
- identifying cloud providers
    
- infrastructure mapping
    

---

# Enumeration Workflow

## IF domain discovered

```text
→ run whois
→ extract names
→ identify name servers
→ identify registrar
→ identify emails
```

---

## IF IP discovered (using ping for e.g.)

```text
→ reverse lookup
→ identify netblock
→ identify hosting provider
→ expand attack surface
```

---

# Example Commands

## Standard Domain Lookup

```bash
whois megacorpone.com
```

---

## PWK-Style WHOIS Query

```bash
whois megacorpone.com -h 192.168.50.251
```

---

## Reverse IP Lookup

```bash
whois 38.100.193.70
```

---

# Notes for OSCP

- WHOIS is passive reconnaissance
    
- No packets sent directly to target systems
    
- Often useful before DNS enumeration
    
- Private registration may hide registrant details
    
- Still valuable for:
    
    - infrastructure insight
        
    - registrar info
        
    - netblocks
        
    - DNS pivots
        

---

# Common Pivot Paths

|Discovery|Next Action|
|---|---|
|Names|username generation|
|Emails|credential attacks|
|Name Servers|DNS enumeration|
|NetRange|subnet scanning|
|OrgName|OSINT|
|Registrar|infrastructure profiling|

---

# Related Enumeration

After WHOIS:

```text
WHOIS
→ DNS Enumeration
→ Subdomain Enumeration
→ Port Scanning
→ Service Enumeration
```

## Important Reality

Modern WHOIS data is often redacted due to:
- GDPR
- ICANN privacy rules
- government protections
- registrar privacy services

Do NOT assume WHOIS is useless if fields are hidden.

Focus on remaining pivot points:
- name servers
- registrar
- organization
- IP ranges
- DNS infrastructure

Recon is about chaining intelligence sources together.
