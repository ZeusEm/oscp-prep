# SMTP Enumeration — OSCP Style Execution Notes (Corrected + Real-World Addendum)

## Overview

SMTP (Simple Mail Transfer Protocol) commonly operates on:

|Port|Service|
|---|---|
|25|SMTP|
|465|SMTPS|
|587|SMTP Submission|

SMTP enumeration focuses on:

- User enumeration
    
- Mail server fingerprinting
    
- Open relay testing
    
- Valid email discovery
    
- Internal infrastructure discovery
    

Common SMTP enumeration methods:

|Method|Purpose|
|---|---|
|VRFY|Verify if user exists|
|EXPN|Expand mailing list|
|RCPT TO|Validate recipient|
|Banner Grabbing|Identify mail server|
|STARTTLS|Check encryption support|

---

# 1. Initial SMTP Discovery

## Fast Targeted SMTP Scan

```bash
nmap -sV -p 25,465,587 <target>
```

Example:

```bash
nmap -sV -p 25,465,587 192.168.50.8
```

### Purpose

- Detect SMTP services
    
- Identify version/banner
    
- Check filtered/open ports
    

---

# 2. Aggressive SMTP Enumeration

```bash
nmap -sV -sC -p 25,465,587 <target>
```

Example:

```bash
nmap -sV -sC -p 25,465,587 192.168.50.8
```

### Purpose

- Run default NSE scripts
    
- Gather banners
    
- Detect SMTP capabilities
    

---

# 3. Manual Banner Grabbing

## Using Netcat

```bash
nc -nv <target> 25
```

Example:

```bash
nc -nv 192.168.50.8 25
```

### Typical Successful Output

```bash
220 mail ESMTP Postfix (Ubuntu)
```

---

# 4. Manual SMTP User Enumeration

After connecting:

```bash
VRFY root
```

Possible response:

```bash
252 2.0.0 root
```

Invalid user example:

```bash
550 5.1.1 User unknown
```

---

# 5. SMTP NSE Scripts

## Enumerate SMTP Capabilities

```bash
nmap --script smtp-commands -p 25 <target>
```

---

## SMTP Open Relay Check

```bash
nmap --script smtp-open-relay -p 25 <target>
```

---

## SMTP User Enumeration (Correct Syntax)

```bash
nmap --script smtp-enum-users \
--script-args smtp-enum-users.methods={VRFY,EXPN,RCPT} \
-p 25 <target>
```

Example:

```bash
nmap --script smtp-enum-users \
--script-args 'smtp-enum-users.methods={VRFY,EXPN,RCPT}' \
-p 25 192.168.50.8
```

## IMPORTANT

The earlier syntax failed because:

```bash
smtp-enum-users.methods=EXPN
smtp-enum-users.methods=RCPT
```

were interpreted as separate targets.

Correct usage requires:

- Entire argument in quotes  
    OR
    
- No spaces inside braces
    

---

# 6. STARTTLS Enumeration

## Check TLS Support

```bash
openssl s_client -connect <target>:25 -starttls smtp
```

Example:

```bash
openssl s_client -connect 192.168.50.8:25 -starttls smtp
```

### Purpose

- Check STARTTLS support
    
- Extract certificates
    
- Identify mail server info
    

---

# 7. smtp-user-enum Tool

## Single User Check

```bash
smtp-user-enum -M VRFY -u root -t <target>
```

Example:

```bash
smtp-user-enum -M VRFY -u root -t 192.168.50.8
```

---

## Multiple User Enumeration

```bash
smtp-user-enum -M VRFY -U <wordlist> -t <target>
```

Example:

```bash
smtp-user-enum -M VRFY -U users.txt -t 192.168.50.8
```

---

# 8. Preloaded Kali Username Wordlists (VERY IMPORTANT)

Instead of manually creating users.txt every time, use built-in Kali wordlists.

## Common Username Lists

### SecLists

```bash
/usr/share/seclists/Usernames/
```

Examples:

```bash
/usr/share/seclists/Usernames/top-usernames-shortlist.txt
```

```bash
/usr/share/seclists/Usernames/Names/names.txt
```

```bash
/usr/share/seclists/Usernames/xato-net-10-million-usernames.txt
```

---

## Example Usage

```bash
smtp-user-enum -M VRFY \
-U /usr/share/seclists/Usernames/top-usernames-shortlist.txt \
-t 192.168.50.8
```

---

# 9. RCPT TO Enumeration

Useful when VRFY disabled.

```bash
smtp-user-enum -M RCPT \
-U /usr/share/seclists/Usernames/top-usernames-shortlist.txt \
-t 192.168.50.8
```

---

# 10. Windows SMTP Enumeration

## Test SMTP Connectivity

```powershell
Test-NetConnection -Port 25 <target>
```

Example:

```powershell
Test-NetConnection -Port 25 192.168.50.8
```

---

## Enable Telnet Client

```powershell
dism /online /Enable-Feature /FeatureName:TelnetClient
```

---

## Manual SMTP Interaction

```powershell
telnet <target> 25
```

Then:

```text
VRFY root
```

---

# 11. Useful SMTP NSE Scripts

## List All SMTP Scripts

```bash
ls /usr/share/nmap/scripts/smtp*
```

---

## Commonly Useful Scripts

|Script|Purpose|
|---|---|
|smtp-commands|Enumerate SMTP commands|
|smtp-enum-users|User enumeration|
|smtp-open-relay|Open relay testing|
|smtp-ntlm-info|NTLM information|
|smtp-strangeport|Detect SMTP on unusual ports|

---

# 12. OSCP Workflow

## Recommended Enumeration Order

### Step 1

```bash
nmap -sV -p 25,465,587 <target>
```

### Step 2

```bash
nc -nv <target> 25
```

### Step 3

```bash
nmap --script smtp-commands -p 25 <target>
```

### Step 4

```bash
smtp-user-enum -M VRFY -U <wordlist> -t <target>
```

### Step 5

```bash
openssl s_client -connect <target>:25 -starttls smtp
```

### Step 6

Attempt RCPT enumeration if VRFY disabled.

---

# 13. Interpretation Guide

|Result|Meaning|
|---|---|
|open|Service accessible|
|closed|Service reachable but inactive|
|filtered|Firewall dropping packets|
|Connection refused|Service not listening|
|Timed out|Firewall silently dropping|

---

# Addendum — Real World Execution Analysis (Your Results)

# 1. Key Real-World Observation

Your target:

```text
164.100.161.148
```

showed:

```text
25/tcp filtered smtp
465/tcp filtered smtps
587/tcp filtered submission
```

This is extremely important operationally.

---

# 2. What “filtered” REALLY Means

Filtered ≠ service confirmed.

Filtered usually indicates:

- Firewall dropping packets
    
- ACL protection
    
- IPS/IDS filtering
    
- Geo/IP restrictions
    
- SMTP disabled externally
    

This means:

```text
Enumeration cannot proceed directly.
```

This is a critical OSCP mindset point.

---

# 3. Why Netcat Failed

You observed:

```bash
nc -nv 164.100.161.148 25
Connection refused
```

Meaning:

- TCP handshake blocked/refused
    
- SMTP daemon inaccessible externally
    

Therefore:

- VRFY impossible
    
- EXPN impossible
    
- RCPT impossible
    
- Banner grabbing impossible
    

---

# 4. Why smtp-user-enum Failed

Because:

```text
smtp-user-enum requires:
1. Reachable SMTP service
2. Username source
```

Your target failed condition #1.

So even with perfect wordlists:

```text
Enumeration still impossible externally.
```

This is an important exam insight.

---

# 5. Correct OSCP Thinking

## VERY IMPORTANT

Do NOT force enumeration endlessly.

Instead document:

```text
SMTP ports filtered.
Direct SMTP enumeration not possible externally.
Likely firewall filtering or service restriction.
```

Then pivot.

---

# 6. What You SHOULD Do Next in OSCP

When SMTP blocked:

## Pivot to:

|Alternative|Reason|
|---|---|
|Web enumeration|Most exposed attack surface|
|HTTPS recon|Often leaks usernames/emails|
|DNS recon|MX/SPF/DMARC records|
|Subdomain enumeration|Alternate mail gateways|
|TLS cert inspection|Internal naming|
|OSINT|Employee usernames|

---

# 7. Real OSCP-Level Adaptation

If SMTP inaccessible:

## Extract usernames elsewhere

Examples:

- Web pages
    
- GitHub
    
- LinkedIn
    
- SSL certificates
    
- Email patterns
    
- SMB shares
    
- SNMP
    
- Kerberos
    
- LDAP
    
- FTP banners
    

Then reuse during:

- Password spraying
    
- SMB auth
    
- SSH auth
    
- Web login testing
    

---

# 8. Critical Exam Mindset

## Enumeration Failure IS Enumeration

This is a massive OSCP concept.

These findings are valuable:

```text
- ICMP blocked
- SMTP filtered
- SMB filtered
- HTTPS only exposed
- Likely hardened edge firewall
```

This already profiles the target security posture.

---

# 9. Your Corrected Command That Failed Earlier

Incorrect:

```bash
--script-args smtp-enum-users.methods={VRFY,EXPN,RCPT}
```

Correct:

```bash
--script-args 'smtp-enum-users.methods={VRFY,EXPN,RCPT}'
```

OR

```bash
--script-args smtp-enum-users.methods={VRFY,EXPN,RCPT}
```

without shell splitting.

---

# 10. Extremely Important OSCP Addendum

## Use “top-usernames-shortlist.txt” FIRST

Instead of giant lists.

Reason:

- Faster
    
- Less noisy
    
- Exam efficient
    

Recommended:

```bash
/usr/share/seclists/Usernames/top-usernames-shortlist.txt
```

Then escalate to:

```bash
xato-net-10-million-usernames.txt
```

only if needed.

---

# 11. Final OSCP Takeaway

Your execution demonstrates proper methodology:

- Port discovery
    
- Service validation
    
- NSE usage
    
- Manual testing
    
- TLS testing
    
- Enumeration attempts
    
- Windows + Kali testing
    
- Error interpretation
    

This is exactly how strong OSCP candidates operate.

The important evolution now is:

```text
Recognize dead ends early.
Document clearly.
Pivot aggressively.
```