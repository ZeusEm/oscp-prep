# TCP/UDP Port Scanning Theory

## Objective

Identify:

- Open TCP ports
    
- Open UDP ports
    
- Running services
    
- Potential attack vectors
    
- Firewall filtering behavior
    

Port scanning is foundational OSCP reconnaissance.

---

# Legal & Operational Warning

Port scanning:

- is intrusive
    
- may trigger IDS/IPS
    
- may overload systems
    
- may cause outages
    

Never scan systems without explicit authorization.

---

# OSCP Port Scanning Methodology

## Core Principle

Port scanning is iterative.

```text
Initial Scan
    ↓
Identify Services
    ↓
Targeted Enumeration
    ↓
Focused Follow-Up Scans
    ↓
Exploit Research
```

---

# Smart Enumeration Strategy

Avoid:

```text
Blind full-range scans immediately
```

Instead:

1. Start small
    
2. Identify likely targets
    
3. Expand gradually
    

Example:

```text
Scan 80/443 first
    ↓
Identify web servers
    ↓
Run deeper scans selectively
```

---

# TCP vs UDP

|Protocol|Connection-Oriented|Reliable|Handshake|
|---|---|---|---|
|TCP|Yes|Yes|3-Way Handshake|
|UDP|No|No|Stateless|

---

# TCP Three-Way Handshake

## Process

```text
Client → SYN
Server → SYN-ACK
Client → ACK
```

If completed successfully:

```text
Port is OPEN
```

---

# TCP CONNECT Scanning

## Concept

Attempt full TCP connection.

If handshake succeeds:

```text
OPEN
```

If connection refused:

```text
CLOSED
```

---

# Netcat TCP Port Scanning

## Basic TCP Scan

```bash
nc -nvv -w 1 -z 192.168.50.152 3388-3390
```

---

# Important Options

|Option|Purpose|
|---|---|
|-n|No DNS resolution|
|-v|Verbose|
|-vv|Extra verbose|
|-w 1|1 second timeout|
|-z|Zero-I/O mode (scan only)|

---

# Example Output

```text
3389 open
3388 Connection refused
3390 Connection refused
```

---

# Interpretation

|Result|Meaning|
|---|---|
|open|Service listening|
|Connection refused|Port closed|
|timeout|Filtered/firewalled|

---

# TCP Packet Behavior

## Open Port

```text
SYN → SYN-ACK → ACK
```

Successful handshake.

---

## Closed Port

```text
SYN → RST-ACK
```

Target actively rejects connection.

---

# FIN Connection Closure

After successful connection:

```text
FIN-ACK
```

Connection terminated cleanly.

---

# Wireshark TCP Analysis

## Typical Packet Sequence

|Packet|Purpose|
|---|---|
|SYN|Connection attempt|
|SYN-ACK|Open port response|
|ACK|Handshake completion|
|FIN-ACK|Connection closure|
|RST-ACK|Port closed|

---

# UDP Scanning Theory

## Key Difference

UDP is:

```text
Stateless
```

No handshake exists.

---

# UDP Scanning Mechanism

Scanner sends:

```text
Empty UDP packet
```

---

# Possible Outcomes

|Response|Meaning|
|---|---|
|No response|Possibly open|
|ICMP Port Unreachable|Closed|
|Application response|Open|

---

# Netcat UDP Scan

## Basic UDP Scan

```bash
nc -nv -u -z -w 1 192.168.50.149 120-123
```

---

# UDP Options

|Option|Purpose|
|---|---|
|-u|UDP mode|
|-z|Scan only|
|-w 1|Timeout|

---

# Example Output

```text
123 (ntp) open
```

---

# UDP Packet Behavior

## Open UDP Port

```text
UDP packet sent
No ICMP unreachable returned
```

May indicate:

```text
OPEN
```

---

## Closed UDP Port

Target returns:

```text
ICMP Port Unreachable
```

---

# UDP Scanning Problems

UDP scanning is notoriously unreliable.

---

# Common UDP Issues

## 1. Firewalls Drop ICMP

Result:

```text
False positives
```

Closed ports may appear open.

---

## 2. Routers Filter ICMP

Scanner receives no response.

Again:

```text
Possibly false OPEN state
```

---

## 3. Many Scanners Skip UDP

Most scanners prioritize:

```text
TCP only
```

Leading to:

```text
Missed attack surface
```

---

# Important OSCP UDP Insight

Many high-value services use UDP.

Examples:

|Port|Service|
|---|---|
|53|DNS|
|67/68|DHCP|
|69|TFTP|
|123|NTP|
|161|SNMP|
|500|IKE/IPsec|

---

# Why UDP Matters in OSCP

Open UDP ports may expose:

- SNMP information leakage
    
- DNS misconfigurations
    
- TFTP file access
    
- NTP amplification
    
- VPN infrastructure
    

---

# TCP vs UDP Reliability

|Aspect|TCP|UDP|
|---|---|---|
|Reliable Results|High|Low|
|Handshake|Yes|No|
|Firewall Visibility|Easier|Harder|
|False Positives|Lower|Higher|

---

# OSCP Enumeration Workflow

## Recommended Sequence

### 1. Quick TCP Discovery

```bash
nc -nvv -w 1 -z target 1-1000
```

---

### 2. Identify High-Value Ports

Examples:

```text
22
80
443
445
3389
```

---

### 3. Begin Service Enumeration

Examples:

```bash
nmap -sV
curl
enum4linux
smbclient
```

---

### 4. Run Broader Scans In Background

Expand scope gradually.

---

### 5. Perform UDP Checks

Especially:

```text
53
69
123
161
500
```

---

# Important Port Scanning Mindset

Do not think:

```text
Scan everything immediately
```

Think:

```text
Scan intelligently
```

---

# Practical OSCP Scanning Philosophy

## Goal

Minimize:

- noise
    
- detection
    
- wasted time
    

Maximize:

- actionable intelligence
    
- service discovery
    
- attack surface visibility
    

---

# Common OSCP Mistakes

## Ignoring UDP

Very common beginner mistake.

---

## Blind Full-Range Scans

Can waste:

- time
    
- bandwidth
    
- exam time
    

---

## Ignoring Filtered Ports

Filtered ports may still:

- exist
    
- be reachable internally
    
- reveal firewall behavior
    

---

# Netcat Limitations

Netcat is useful for:

- quick checks
    
- lightweight scans
    
- living-off-the-land
    

But:

```text
Netcat is NOT a full port scanner
```

---

# Why We Still Learn Netcat Scanning

Understanding Netcat scanning teaches:

- TCP handshake mechanics
    
- UDP behavior
    
- port state logic
    
- how scanners work internally
    

Critical for OSCP troubleshooting.

---

# OSCP Notes Checklist

Document:

- Open TCP ports
    
- Open UDP ports
    
- Filtered ports
    
- Service banners
    
- Firewall behavior
    
- Timeouts
    
- Interesting protocols
    
- High-value services
    

---

# High-Value OSCP Ports

|Port|Service|
|---|---|
|21|FTP|
|22|SSH|
|25|SMTP|
|53|DNS|
|80|HTTP|
|110|POP3|
|111|RPCbind|
|135|MSRPC|
|139|NetBIOS|
|161|SNMP|
|389|LDAP|
|443|HTTPS|
|445|SMB|
|1433|MSSQL|
|2049|NFS|
|3306|MySQL|
|3389|RDP|
|5985|WinRM|
|8080|Alternate HTTP|

---

# Minimal Practical OSCP Command Set

## TCP Scan

```bash
nc -nvv -w 1 -z target 1-1000
```

---

## UDP Scan

```bash
nc -nv -u -z -w 1 target 53-161
```

---

## Focused Port Scan

```bash
nc -nvv -w 1 -z target 22 80 443 445 3389
```

---

# Core OSCP Takeaways

## TCP

Reliable and easy to interpret.

---

## UDP

Unreliable but extremely valuable.

Never ignore it.

---

## Port Scanning Is Dynamic

Each scan determines:

```text
What to scan next
```

---

# Advanced Recon Principle

Beginner mindset:

```text
Find open ports
```

Advanced mindset:

```text
Understand the target’s exposed attack surface and prioritize intelligently
```

# Addendum — Practical Observation From Today’s Netcat Scan

## Example Command

```bash
nc -nvv -w 1 -z 164.100.161.148 1-1000
```

---

# Observed Result

```text
Only TCP 443 was open
```

---

# Practical Interpretation

This demonstrates a very common real-world scenario:

```text
Minimal exposed attack surface
```

The target exposed only:

| Port | Service |
| ---- | ------- |
| 443  | HTTPS   |

All other scanned ports were either:

- closed
    
- filtered
    
- non-responsive
    

---

# Important Enumeration Insight

A scan result with:

```text
Only 443 open
```

does NOT mean:

```text
No further enumeration possible
```

It means:

```text
Enumeration focus shifts toward the exposed service
```

---

# Enumeration Principle Reinforced

Port scanning is NOT only about:

```text
Finding many open ports
```

Sometimes:

```text
A single exposed service becomes the primary attack surface
```

---

# Practical Recon Workflow Learned

Today’s scan reinforces the methodology:

```text
Initial Scan
    ↓
Identify Open Service
    ↓
Focus Enumeration Around That Service
```

Example:

```text
443 open
    ↓
HTTPS/web-focused enumeration later
```

---

# Netcat Limitation Observed

Today’s scan also demonstrated that Netcat primarily provides:

```text
Port state visibility
```

Example:

- open
    
- closed
    
- timeout/filtered
    

But Netcat does NOT provide:

- detailed service identification
    
- banner analysis
    
- deeper enumeration
    

---

# Firewall Behavior Observation

During the scan:

| Behavior                        | Likely Meaning                   |
| ------------------------------- | -------------------------------- |
| Open response on 443            | Service listening                |
| No other successful connections | Restricted exposure or filtering |

---

# Important Practical Lesson

Modern systems commonly expose:

```text
Very few externally reachable services
```

Therefore:

```text
Even one open port is important
```

---

# OSCP Enumeration Mindset Reinforced

Do not think:

```text
Few open ports = nothing interesting
```

Think:

```text
Identify the exposed surface and investigate it methodically
```