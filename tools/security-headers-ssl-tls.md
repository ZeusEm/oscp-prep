# Security Headers & SSL/TLS Recon

# Purpose

Assess target web security posture through third-party scanning services.

Goal:
- identify weak hardening
- discover insecure SSL/TLS configurations
- infer defensive maturity
- prioritize later attack vectors

This is considered passive reconnaissance because:
- third-party services perform the scans
- attacker system does not directly interact with target

---

# Services

## Security Headers

https://securityheaders.com/

Purpose:
- analyze HTTP response headers
- identify missing security controls
- assess web hardening maturity

---

## Qualys SSL Labs

https://www.ssllabs.com/ssltest/

Purpose:
- analyze SSL/TLS configuration
- identify weak protocols
- detect insecure cipher suites
- reveal SSL/TLS vulnerabilities

---

# Security Headers Workflow

## 1. Scan Target

Open:

```text
https://securityheaders.com/
````

Enter target:

```text
https://target.com
```

---

## 2. Analyze Grade

Possible grades:

|Grade|Meaning|
|---|---|
|A+|Strong hardening|
|B/C|Moderate security posture|
|D/F|Weak configuration|
|R|Redirect issue|
|T|Timeout|

---

## 3. Important Headers

### Content-Security-Policy (CSP)

Protects against:

- XSS
    
- malicious script execution
    

Missing CSP:

- weaker browser-side protection
    
- possible insecure development practices
    

---

### X-Frame-Options

Protects against:

- clickjacking attacks
    

Missing header:

- target may allow iframe embedding
    

---

### Strict-Transport-Security (HSTS)

Forces HTTPS usage.

Missing HSTS:

- downgrade attack potential
    
- weaker HTTPS enforcement
    

---

### X-Content-Type-Options

Protects against MIME sniffing.

---

### Referrer-Policy

Controls referrer leakage.

---

# OSCP Interpretation

Missing headers are NOT always direct vulnerabilities.

Instead, they may indicate:

- poor hardening
    
- outdated configurations
    
- inexperienced developers
    
- weak security maturity
    

This helps prioritize:

- web exploitation
    
- XSS testing
    
- authentication attacks
    
- session management testing
    

---

# Security Headers Note Template

```text
Target:
securityheaders.com Result:
Grade:

Missing Headers:
- CSP
- HSTS
- X-Frame-Options

Observations:
- weak hardening
- possible insecure deployment
```

---

# SSL Labs Workflow

## 1. Scan Target

Open:

```text
https://www.ssllabs.com/ssltest/
```

Enter:

```text
target.com
```

---

## 2. Review Important Sections

Focus on:

- TLS versions
    
- cipher suites
    
- certificate validity
    
- known vulnerabilities
    
- weak algorithms
    

Ignore excessive noise.

---

# Critical Findings

## Weak TLS Versions

### BAD

```text
TLS 1.0
TLS 1.1
SSLv3
```

Indicates:

- legacy configuration
    
- weak cryptography
    

---

## Strong Versions

### GOOD

```text
TLS 1.2
TLS 1.3
```

---

# Weak Cipher Suites

Look for:

```text
CBC
SHA1
RC4
3DES
EXPORT
NULL
```

These may indicate:

- downgrade opportunities
    
- weak encryption
    
- legacy support
    

---

# Vulnerabilities to Watch For

## Heartbleed

OpenSSL memory disclosure vulnerability.

---

## POODLE

SSLv3 downgrade vulnerability.

---

## Weak Diffie-Hellman

Small DH keys vulnerable to attacks.

---

# OSCP Interpretation

Weak SSL/TLS posture suggests:

- outdated infrastructure
    
- poor patching practices
    
- legacy compatibility requirements
    
- broader security weaknesses
    

This helps guide:

- exploit selection
    
- vulnerability research
    
- version-based attacks
    

---

# SSL Labs Note Template

```text
Target:
SSL Labs Grade:

Supported TLS Versions:
- TLS 1.0
- TLS 1.1
- TLS 1.2

Weak Ciphers:
- TLS_DHE_RSA_WITH_AES_256_CBC_SHA

Vulnerabilities:
- None observed

Observations:
- legacy TLS support enabled
- outdated hardening practices
```

---

# Important OSCP Insight

These tools rarely provide direct exploitation paths.

Their real value is:

- identifying weak operational security
    
- profiling infrastructure maturity
    
- guiding active enumeration priorities
    
- identifying likely outdated services
    

---

# Typical OSCP Workflow

```text
WHOIS
→ Google Dorking
→ Netcraft
→ GitHub/OSINT
→ Shodan
→ Security Headers
→ SSL Labs
→ Active Enumeration
```

---

# Common OSCP Notes

```text
Weak TLS configuration observed.
Legacy protocols enabled.
Potentially outdated infrastructure.
Further version-based enumeration recommended.
```

---

# Important Limitation

Third-party scans may:

- be cached
    
- outdated
    
- incomplete
    

Always verify findings manually during active enumeration later.

Never rely exclusively on passive recon results.

---

# Native Kali Alternatives (Preferred During Real Assessments)

Third-party web portals may fail due to:
- WAF protection
- IDS/IPS blocking
- rate limiting
- geofencing
- anti-scanner controls
- server timeout policies

Always validate findings manually using native tools.

---

# sslscan

## Purpose

Enumerate:
- supported TLS versions
- cipher suites
- certificate details
- weak protocols
- SSL vulnerabilities

---

## Basic Usage

```bash
sslscan <target>
```

Example:

```bash
sslscan www.joinindiannavy.gov.in
```

---

## Key Findings to Look For

### Weak Protocols

Bad:
- SSLv2
- SSLv3
- TLS 1.0
- TLS 1.1

Good:
- TLS 1.2
- TLS 1.3

---

### Weak Ciphers

Look for:
- RC4
- DES
- 3DES
- MD5
- SHA1

---

### Certificate Information

Extract:
- issuer
- validity
- SAN entries
- RSA key size

---

### Vulnerability Checks

Look for:
- Heartbleed
- compression enabled
- renegotiation issues

---

# openssl s_client

## Purpose

Manually inspect:
- TLS handshake
- certificates
- negotiated cipher
- certificate chain
- SAN entries

---

## Basic Usage

```bash
openssl s_client -connect <target>:443
```

Example:

```bash
openssl s_client -connect www.joinindiannavy.gov.in:443
```

---

## Useful Flags

### Specify SNI

```bash
openssl s_client -connect target:443 -servername target
```

---

### Show Certificates

```bash
openssl s_client -connect target:443 -showcerts
```

---

# Key Information to Extract

## Certificate Subject

```text
subject=
```

---

## Certificate Issuer

```text
issuer=
```

---

## Validity Period

```text
NotBefore
NotAfter
```

---

## Negotiated TLS Version

```text
Protocol : TLSv1.3
```

---

## Negotiated Cipher

```text
Cipher : TLS_AES_256_GCM_SHA384
```

---

## Subject Alternative Name (SAN) Enumeration

Look for:

```text
DNS:
```

May reveal:
- subdomains
- staging hosts
- alternate services

---

# OSCP Exam Relevance

Useful for:
- HTTPS enumeration
- SSL/TLS assessment
- certificate analysis
- discovering hidden hosts
- validating web security posture
- confirming third-party scan results

---

# Practical OSCP Mindset

Do not rely solely on:
- SSL Labs
- SecurityHeaders
- browser scanners

Always verify manually using:
- sslscan
- openssl
- curl
- nmap NSE scripts

Native tooling is more reliable during:
- labs
- exams
- real-world engagements