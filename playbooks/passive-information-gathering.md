# Passive Information Gathering

# Objective

Gather intelligence WITHOUT directly interacting with target systems.

---

# Goals

Collect:
- domains
- subdomains
- employee names
- email addresses
- exposed files
- technologies
- metadata
- SSL/TLS information

---

# Workflow

## 1. WHOIS Enumeration

Purpose:
- identify ownership
- discover name servers
- discover netblocks

Reference:
[[whois]]

---

## 2. Google Dorking

Purpose:
- identify exposed files
- locate login portals
- discover backups

Reference:
[[google-dorking]]

---

## 3. Netcraft

Purpose:
- identify hosting provider
- discover technologies
- fingerprint infrastructure

Reference:
[[netcraft]]

---

## 4. OSINT

Sources:
- GitHub
- LinkedIn
- public repositories
- paste sites

Goal:
- identify usernames
- locate secrets
- build target profile

Reference:
[[open-source-code-recon]]

---

## 5. Shodan

Purpose:
- identify exposed services
- discover internet-facing systems

---

## 6. Security Headers

Resource:
https://securityheaders.com/

Check:
- CSP
- HSTS
- X-Frame-Options

---

## 7. SSL/TLS Analysis

Resource:
https://www.ssllabs.com/ssltest/

Review:
- TLS versions
- weak ciphers
- certificate issues

---

# Operational Notes

Passive recon:
- reduces noisy scanning
- improves phishing realism
- identifies technologies
- expands attack surface

---

# OSCP Mindset

Passive recon builds targeting intelligence BEFORE active enumeration begins.