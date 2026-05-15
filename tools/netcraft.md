# Netcraft

# Purpose

Netcraft is a passive reconnaissance platform used to identify:
- subdomains
- hosting providers
- web technologies
- operating systems
- server history
- shared hosting infrastructure
- SSL/TLS information

Passive recon means:
- no packets sent directly to target
- low detection risk
- useful during early enumeration

---

# Primary Use Cases in OSCP

Use Netcraft to:

- map external attack surface
- identify technologies before scanning
- discover hidden subdomains
- identify hosting providers
- identify WAF/CDN presence
- discover shared infrastructure
- collect intelligence for later exploitation

---

# Access

Website:

```text
https://searchdns.netcraft.com
````

---

# Core Workflow

## 1. Search Target Domain

Search:

```text
*.target.com
```

Example:

```text
*.megacorpone.com
```

Goal:

- enumerate subdomains
    
- identify internet-facing assets
    

---

## 2. Review Site Reports

For each discovered host:

- open Site Report
    

Extract:

- server technologies
    
- operating system
    
- web server
    
- CDN/WAF
    
- SSL details
    
- hosting provider
    
- historical changes
    

---

# High-Value Data to Extract

## Subdomains

Examples:

```text
dev.target.com
vpn.target.com
mail.target.com
test.target.com
intranet.target.com
```

Potential value:

- staging environments
    
- forgotten applications
    
- admin panels
    
- VPN portals
    

---

## Web Technologies

Examples:

- Apache
    
- Nginx
    
- IIS
    
- PHP
    
- ASP.NET
    
- WordPress
    

OSCP usage:

- version-specific exploits
    
- targeted enumeration
    
- exploit selection
    

---

## Hosting Provider

Examples:

- AWS
    
- Azure
    
- Cloudflare
    
- DigitalOcean
    

Useful for:

- identifying cloud exposure
    
- detecting CDN/WAF
    
- infrastructure mapping
    

---

## Operating System Clues

Examples:

- Ubuntu
    
- Windows Server
    
- FreeBSD
    

Useful for:

- tailoring exploits
    
- payload selection
    
- privilege escalation planning
    

---

# Important OSCP Insight

Passive recon should guide active recon.

Example:

IF Netcraft shows:

- Apache + PHP  
    THEN:
    
- prioritize web enumeration
    
- check PHP vulnerabilities
    
- run directory brute force
    

IF Netcraft shows:

- IIS + ASP.NET  
    THEN:
    
- prioritize Windows attack paths
    
- enumerate .aspx endpoints
    

---

# Example Workflow

## Step 1 — Search Domain

```text
*.target.com
```

---

## Step 2 — Record Subdomains

Document:

- dev.target.com
    
- mail.target.com
    
- vpn.target.com
    

---

## Step 3 — Open Site Reports

Extract:

- web server
    
- technologies
    
- hosting provider
    
- OS clues
    

---

## Step 4 — Feed Into Enumeration

Example pivots:

IF WordPress detected:  
→ run WPScan

IF IIS detected:  
→ enumerate ASP.NET

IF VPN portal found:  
→ investigate authentication surface

IF admin subdomain found:  
→ prioritize attack path

---

# Netcraft → Enumeration Pivot Mapping

|Netcraft Finding|Next Action|
|---|---|
|WordPress|Run WPScan|
|Apache|Web enumeration|
|IIS|ASP.NET enumeration|
|Cloudflare|Expect filtering/WAF|
|VPN portal|Credential attack surface|
|Dev environment|Look for weak security|
|Shared hosting|Identify neighboring assets|

---

# Important Limitations

- Data may be outdated
    
- Technology fingerprinting may be inaccurate
    
- Hidden/internal assets will not appear
    
- CDN/WAF may obscure origin infrastructure
    

Always validate findings manually.

---

# OSCP Notes Template

```text
Target:
target.com

Subdomains:
- vpn.target.com
- dev.target.com
- mail.target.com

Technologies:
- Apache
- PHP
- WordPress

Hosting:
- Cloudflare

Potential Attack Surface:
- WordPress enumeration
- VPN authentication
- Admin portal discovery
```

---

# OSCP Relevance

Netcraft helps:

- reduce blind scanning
    
- identify likely technologies
    
- prioritize attack vectors
    
- improve enumeration efficiency
    

Passive recon done properly makes exploitation significantly faster.

# IP Delegation Interpretation

Netcraft may display IP delegation hierarchy.

Purpose:
- understand ownership chain
- identify hosting provider
- identify infrastructure relationships

Example Flow:

IANA
→ Regional Registry (ARIN/APNIC/RIPE)
→ Organization Allocation
→ Datacenter/Subnet
→ Target IP

Useful For:
- infrastructure mapping
- identifying shared ownership
- identifying hosting providers
- subnet expansion
- attack surface analysis

OSCP Insight:
Delegation hierarchy helps build target context and improves recon quality.