# Active Information Gathering Playbook

# Objective

Directly interact with target systems to identify:

- live hosts
    
- open ports
    
- exposed services
    
- service versions
    
- network shares
    
- DNS information
    
- users and accounts
    
- potential attack vectors
    

This phase follows passive information gathering.

---

# Learning Objectives

- Perform Netcat and Nmap port scanning
    
- Conduct DNS Enumeration
    
- Conduct SMB Enumeration
    
- Conduct SMTP Enumeration
    
- Conduct SNMP Enumeration
    
- Understand Living off the Land techniques
    

---

# Methodology

```text
Target Validation
    ↓
Host Discovery
    ↓
Port Scanning
    ↓
Service Identification
    ↓
Protocol-Specific Enumeration
    ↓
Documentation
    ↓
Attack Surface Mapping
```

---

# Core Principles

## Enumeration is Iterative

Every finding may lead to:

- new hosts
    
- new ports
    
- credentials
    
- usernames
    
- shares
    
- trust relationships
    
- additional enumeration paths
    

Never stop after:

- one scan
    
- one tool
    
- one result
    

---

## Validate Tool Output

Do not blindly trust:

- automated detection
    
- OS fingerprinting
    
- NSE results
    
- banners
    
- service identification
    

Always manually verify important findings.

---

## Document Everything

Record:

- commands used
    
- discovered hosts
    
- open ports
    
- service versions
    
- usernames
    
- shares
    
- credentials
    
- screenshots
    
- observations
    
- attack ideas
    

Good notes directly improve exploitation speed.

---

# Phase 1 — Target Validation

Confirm:

- correct target
    
- VPN connectivity
    
- DNS resolution
    
- reachability
    

Initial goals:

- verify target is alive
    
- determine whether ICMP responses exist
    
- establish baseline connectivity
    

---

# Phase 2 — Host Discovery

Goal:  
Identify live systems.

Typical methods:

- ICMP discovery
    
- ARP discovery
    
- TCP discovery
    

Consider:

- firewalls may block ICMP
    
- hosts may still be alive even if ping fails
    
- internal and external environments behave differently
    

Outputs:

- live IPs
    
- reachable systems
    
- possible filtering devices
    

---

# Phase 3 — Port Scanning

Goal:  
Identify exposed services.

Focus Areas:

- TCP ports
    
- UDP ports
    
- filtered ports
    
- open ports
    
- closed ports
    

Important Concepts:

- full scans take time
    
- quick scans provide initial attack surface
    
- filtered ports often indicate firewall presence
    
- open ports guide enumeration direction
    

Common Targets:

- DNS
    
- SMB
    
- SMTP
    
- SNMP
    
- HTTP/HTTPS
    
- SSH
    
- RDP
    

Outputs:

- port lists
    
- service exposure
    
- possible technologies
    

---

# Phase 4 — Service Enumeration

Goal:  
Identify:

- service versions
    
- technologies
    
- configurations
    
- accessible functionality
    

Important:  
Service enumeration is often more valuable than vulnerability scanning.

Focus on:

- banners
    
- default configurations
    
- anonymous access
    
- weak permissions
    
- accessible data
    

---

# Phase 5 — DNS Enumeration

Goal:  
Gather domain and naming information.

Look for:

- nameservers
    
- mail servers
    
- subdomains
    
- internal naming conventions
    
- zone transfers
    

Potential Findings:

- internal infrastructure
    
- additional targets
    
- VPN hosts
    
- mail infrastructure
    
- development systems
    

Important Concept:  
A successful zone transfer can massively expand attack surface visibility.

---

# Phase 6 — SMB Enumeration

Goal:  
Enumerate Windows file sharing infrastructure.

Focus Areas:

- shares
    
- users
    
- groups
    
- permissions
    
- policies
    
- null sessions
    

Potential Findings:

- sensitive files
    
- usernames
    
- password policies
    
- writable shares
    
- domain information
    

Key Concept:  
SMB often leaks valuable information even before authentication.

---

# Phase 7 — SMTP Enumeration

Goal:  
Enumerate mail services and users.

Focus Areas:

- SMTP banners
    
- VRFY support
    
- EXPN support
    
- valid usernames
    

Potential Findings:

- user accounts
    
- internal naming conventions
    
- mail server software
    

Key Concept:  
Valid usernames become useful during password attacks and authentication testing.

---

# Phase 8 — SNMP Enumeration

Goal:  
Extract information from exposed SNMP services.

Focus Areas:

- community strings
    
- system information
    
- network interfaces
    
- processes
    
- users
    
- routing information
    

Important Concept:  
Misconfigured SNMP can expose extensive internal information.

Potential Findings:

- usernames
    
- installed software
    
- running processes
    
- network structure
    
- hostnames
    

---

# Phase 9 — Banner Grabbing

Goal:  
Identify service details directly.

Focus Areas:

- service banners
    
- protocol responses
    
- server software
    
- versions
    

Key Concept:  
Manual interaction often reveals more than automated scanning.

---

# Phase 10 — Living off the Land

Concept:  
Use native system functionality instead of external tools.

Advantages:

- reduced detection
    
- minimal tooling
    
- native trust relationships
    
- fewer artifacts
    

Examples include:

- built-in Windows utilities
    
- PowerShell
    
- native Linux networking tools
    

Important for:

- stealth
    
- restricted environments
    
- post-compromise operations
    

---

# Analysis Workflow

After enumeration:

```text
Open Port
    ↓
Identify Service
    ↓
Enumerate Service
    ↓
Identify Misconfiguration
    ↓
Identify Attack Path
    ↓
Document Findings
```

---

# OSCP Mindset

The goal is not:

- running many tools
    
- maximum automation
    
- noisy scanning
    

The goal is:

- understanding the target
    
- identifying attack paths
    
- extracting actionable information
    
- building exploitation opportunities
    

---

# Deliverables from Active Information Gathering

By the end of this phase you should ideally have:

- target IP inventory
    
- open ports inventory
    
- service inventory
    
- possible operating systems
    
- usernames
    
- domain information
    
- shares
    
- accessible services
    
- probable technologies
    
- attack surface map
    
- initial exploitation ideas
    

---

# Minimal Active Information Gathering Checklist

-  Connectivity verified
    
-  Live hosts identified
    
-  TCP ports identified
    
-  UDP ports checked where relevant
    
-  Services identified
    
-  DNS enumerated
    
-  SMB enumerated
    
-  SMTP enumerated
    
-  SNMP enumerated
    
-  Banners captured
    
-  Findings documented
    
-  Attack vectors identified