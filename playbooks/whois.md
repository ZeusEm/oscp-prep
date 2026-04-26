# WHOIS Enumeration

## 🔥 When to Use
- Target is a domain (not just IP)
- During initial recon phase
- Before web enumeration

---

## 1. Basic Domain Lookup

`whois <domain>`

Example:
`whois megacorpone.com`

---

## 2. Specify WHOIS Server

`whois -h <whois_server> <domain>`

Example:
`whois -h 192.168.50.251 megacorpone.com`

---

## 3. Reverse Lookup (IP → Owner Info)

`whois <IP>`

Example:
`whois 8.8.8.8`

---

## 4. Key Data to Extract ⚠️

- Registrar
- Registrant Name
- Organization
- Email Address
- Name Servers
- IP Range / Netblock
- Hosting Provider

---

## Real Example Findings

### google.com
- Organization: Google LLC
- Registrar: MarkMonitor
- Name Servers: ns1–ns4.google.com

### 8.8.8.8
- NetRange: 8.8.8.0/24
- Organization: Google LLC
- Location: US

---

## 5. Exploitation Value 🎯

### 🔥 High Probability OSCP
- Names → username generation
- Emails → password spraying / login attempts
- Name servers → DNS enumeration pivot
- Hosting provider → attack surface understanding

---

## 6. Pivot Actions 🔁

IF names found:
  → add to username wordlist

IF email found:
  → try login portals (web, VPN, OWA)

IF name servers found:
  → perform DNS enumeration

IF IP range found:
  → expand scan scope

Use netblock for:
  → subnet scanning

Use org name for:
  → OSINT

---

## 7. Notes

- Data may be hidden (private registration)
- Still useful for:
  - registrar patterns
  - infrastructure hints
