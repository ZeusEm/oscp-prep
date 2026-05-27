# Nmap Port Scanning

## Objective

Use Nmap for:

- Host discovery
    
- TCP/UDP port scanning
    
- Service detection
    
- OS fingerprinting
    
- NSE scripting
    
- Network sweeping
    
- Enumeration prioritization
    

Nmap is the de-facto OSCP reconnaissance tool.

---

# What Makes Nmap Important

Nmap provides:

- Fast scanning
    
- Flexible scan types
    
- Service/version detection
    
- OS fingerprinting
    
- NSE scripting
    
- Host discovery
    
- Greppable output
    
- Large-scale network scanning
    

---

# Raw Sockets & Privileges

Many Nmap scan types require:

```text
Raw socket access
```

Therefore:

```bash
sudo nmap
```

is often required.

---

# Why Raw Sockets Matter

Raw sockets allow Nmap to:

- Craft custom TCP packets
    
- Manipulate flags directly
    
- Perform SYN scans
    
- Perform UDP scans efficiently
    

Without raw sockets:

```text
Nmap falls back to TCP Connect scanning
```

using the Berkeley sockets API.

---

# Default Nmap Behavior

Basic scan:

```bash
nmap target
```

Default behavior:

- Scans top 1000 TCP ports
    
- Performs host discovery
    
- Attempts service identification
    

---

# Basic Nmap Scan

## Syntax

```bash
nmap 192.168.50.149
```

---

# Typical Output

```text
53/tcp   open  domain
88/tcp   open  kerberos-sec
135/tcp  open  msrpc
445/tcp  open  microsoft-ds
```

---

# Important Observation

Even a simple scan may reveal:

- Host role
    
- Likely operating system
    
- Authentication infrastructure
    
- Exposed attack surface
    

Example:

```text
Kerberos + LDAP + SMB
```

often suggests:

```text
Windows Domain Controller
```

---

# Nmap Traffic Awareness

Port scanning creates measurable traffic.

---

# Default Scan vs Full Port Scan

## Top 1000 Scan

```bash
nmap target
```

- Faster
    
- Lower traffic
    
- May miss services
    

---

## Full TCP Scan

```bash
nmap -p 1-65535 target
```

- Much slower
    
- Much noisier
    
- Finds additional ports/services
    

---

# Important OSCP Tradeoff

Balance:

- Speed
    
- Noise
    
- Bandwidth
    
- Detection risk
    
- Thoroughness
    

---

# Scanning Philosophy

Do NOT always:

```text
Scan all 65535 ports immediately
```

Instead:

1. Start small
    
2. Identify interesting hosts
    
3. Expand selectively
    

---

# SYN / Stealth Scanning

## Most Common Nmap Scan

```bash
sudo nmap -sS target
```

---

# SYN Scan Mechanism

Nmap sends:

```text
SYN
```

If target replies:

```text
SYN-ACK
```

Nmap identifies:

```text
OPEN
```

but DOES NOT complete:

```text
ACK
```

---

# SYN Scan Packet Logic

## Open Port

```text
SYN → SYN-ACK
```

---

## Closed Port

```text
SYN → RST
```

---

# Why SYN Scans Matter

Advantages:

- Faster
    
- Efficient
    
- Reduced traffic
    
- Less application-layer logging
    

---

# Important Reality Check

“Stealth” does NOT mean invisible.

Modern firewalls and IDS systems often detect SYN scans.

---

# TCP Connect Scanning

## Syntax

```bash
nmap -sT target
```

---

# TCP Connect Scan Behavior

Nmap performs:

```text
Full TCP three-way handshake
```

using the OS socket API.

---

# When Connect Scans Occur

Used when:

- No raw socket privileges
    
- Non-root user
    
- Certain proxy scenarios
    

---

# TCP Connect Characteristics

|Aspect|TCP Connect|
|---|---|
|Privileges Required|No|
|Speed|Slower|
|Handshake|Complete|
|Application Logging|More likely|

---

# SYN vs TCP Connect

|Feature|SYN Scan (-sS)|TCP Connect (-sT)|
|---|---|---|
|Raw Sockets|Required|Not required|
|Full Handshake|No|Yes|
|Speed|Faster|Slower|
|Noise|Lower|Higher|

---

# UDP Scanning

## Syntax

```bash
sudo nmap -sU target
```

---

# UDP Scan Logic

Nmap sends:

```text
UDP packets
```

Possible outcomes:

|Response|Meaning|
|---|---|
|ICMP Port Unreachable|Closed|
|No response|Possibly open|
|Application response|Open|

---

# Nmap UDP Intelligence

For common UDP services:

- SNMP
    
- DNS
    
- NTP
    

Nmap may send:

```text
Protocol-specific payloads
```

instead of empty packets.

---

# Combined TCP + UDP Scanning

## Syntax

```bash
sudo nmap -sS -sU target
```

---

# Why Combined Scans Matter

Combined scans provide:

- Better visibility
    
- More accurate service mapping
    
- Larger attack surface discovery
    

---

# High-Value UDP Ports

|Port|Service|
|---|---|
|53|DNS|
|69|TFTP|
|123|NTP|
|161|SNMP|
|500|IKE/IPsec|

---

# Network Sweeping

## Purpose

Identify:

```text
Live hosts
```

across a subnet.

---

# Ping Sweep

## Syntax

```bash
nmap -sn 192.168.50.1-253
```

---

# Important Note

Nmap host discovery is NOT just ICMP.

It may also use:

- TCP SYN to 443
    
- TCP ACK to 80
    
- ICMP timestamp requests
    

---

# Greppable Output

## Save Sweep Results

```bash
nmap -sn 192.168.50.1-253 -oG ping-sweep.txt
```

---

# Extract Live Hosts

```bash
grep Up ping-sweep.txt | cut -d " " -f 2
```

---

# Service Sweeping

## Scan Entire Network for HTTP

```bash
nmap -p 80 192.168.50.1-253 -oG web-sweep.txt
```

---

# Extract Hosts with Open HTTP

```bash
grep open web-sweep.txt | cut -d " " -f 2
```

---

# Top Ports Scanning

## Scan Top 20 Ports

```bash
nmap --top-ports=20 target
```

---

# Aggressive Enumeration

## Syntax

```bash
nmap -sT -A --top-ports=20 target
```

---

# What -A Enables

Aggressive mode includes:

- OS detection
    
- Version detection
    
- NSE scripts
    
- Traceroute
    

---

# Important Warning About -A

Aggressive scans:

- Generate significant traffic
    
- Increase noise
    
- Take longer
    
- Trigger detections more easily
    

Use carefully.

---

# Nmap Service Detection

## Version Detection

```bash
nmap -sV target
```

---

# Purpose

Identify:

- Software name
    
- Service version
    
- Possible technologies
    

---

# Example

```text
Apache httpd 2.4.41
OpenSSH 8.2p1
FileZilla Server 1.2.0
```

---

# Banner Grabbing

Nmap may retrieve:

- Service banners
    
- Error messages
    
- Supported commands
    
- SSL information
    

---

# Important Banner Warning

Banners can be:

```text
Fake or intentionally misleading
```

Never trust banners blindly.

---

# OS Fingerprinting

## Syntax

```bash
sudo nmap -O target --osscan-guess
```

---

# Purpose

Identify likely:

- Operating system
    
- OS family
    
- Network stack behavior
    

---

# OS Fingerprinting Indicators

Nmap analyzes:

- TTL values
    
- TCP window sizes
    
- TCP/IP stack behavior
    
- Packet responses
    

---

# Important OS Detection Limitation

Firewalls/proxies may alter packets and reduce accuracy.

OS fingerprinting is:

```text
Best-effort estimation
```

not certainty.

---

# NSE (Nmap Scripting Engine)

## Purpose

Automate:

- Enumeration
    
- Discovery
    
- Information gathering
    
- Vulnerability checks
    

---

# NSE Script Location

```bash
/usr/share/nmap/scripts
```

---

# Running an NSE Script

## Example

```bash
nmap --script http-headers target
```

---

# What http-headers Does

Performs:

```text
HTTP HEAD request
```

and displays returned headers.

---

# NSE Script Help

## Syntax

```bash
nmap --script-help http-headers
```

---

# Why Script Help Matters

Displays:

- Script purpose
    
- Categories
    
- Usage details
    
- Arguments
    
- Documentation URL
    

---

# Important NSE Mindset

NSE scripts can save enormous time during:

- Enumeration
    
- Service analysis
    
- Reconnaissance
    

---

# Practical OSCP Nmap Workflow

## 1. Host Discovery

```bash
nmap -sn subnet
```

---

## 2. Quick TCP Discovery

```bash
nmap target
```

---

## 3. Focused SYN Scan

```bash
sudo nmap -sS target
```

---

## 4. Service Detection

```bash
nmap -sV target
```

---

## 5. UDP Enumeration

```bash
sudo nmap -sU target
```

---

## 6. OS Fingerprinting

```bash
sudo nmap -O --osscan-guess target
```

---

## 7. NSE Enumeration

```bash
nmap --script script-name target
```

---

# Practical OSCP Scanning Strategy

## Small → Focused → Deep

Start with:

- Sweeps
    
- Top ports
    
- Lightweight scans
    

Then escalate toward:

- Full TCP scans
    
- UDP scans
    
- Aggressive enumeration
    
- NSE scripting
    

---

# Traffic & Detection Awareness

Remember:

Every scan generates:

- Packets
    
- Logs
    
- Detection opportunities
    
- Network load
    

---

# Nmap vs Fast Scanners

## MASSCAN / RustScan

Advantages:

- Extremely fast
    

Disadvantages:

- High traffic
    
- Very noisy
    

---

# Why Nmap Remains Preferred

Nmap offers:

- Better control
    
- Lower noise
    
- Better enumeration
    
- More flexibility
    
- More covert behavior
    

---

# Windows Living-Off-The-Land Scanning

## Test-NetConnection

### Syntax

```powershell
Test-NetConnection -Port 445 192.168.50.151
```

---

# Important Output Field

```powershell
TcpTestSucceeded : True
```

indicates:

```text
Port is OPEN
```

---

# PowerShell Port Scanning One-Liner

## Scan First 1024 TCP Ports

```powershell
1..1024 | % {echo ((New-Object Net.Sockets.TcpClient).Connect("192.168.50.151", $_)) "TCP port $_ is open"} 2>$null
```

---

# Why This Matters

Useful when:

- Nmap unavailable
    
- No internet access
    
- Restricted Windows environment
    
- Living-off-the-land required
    

---

# Important OSCP Mindset

Nmap is NOT just a scanner.

It is an:

```text
Attack surface mapping framework
```

---

# Core OSCP Takeaways

## SYN Scans

Fast and efficient.

---

## TCP Connect

Reliable fallback without root privileges.

---

## UDP Scans

Slow but extremely valuable.

---

## NSE

Powerful automation capability.

---

## Enumeration Philosophy

Do not scan blindly.

Always ask:

```text
What information am I trying to obtain next?
```

## Addendum — Real-World Nmap Behaviour & Interpretation Notes

---

# Real-World Firewall Behaviour

## Important Observation

A host may:

```text
Respond on one port
```

while simultaneously:

```text
Dropping probes for all other ports
```

Example from lab execution:

```text
443/tcp open
999 filtered tcp ports (no-response)
```

---

# Filtered vs Closed Ports

## CLOSED Port

Typical response:

```text
RST / Connection Refused
```

Meaning:

```text
Host actively rejected connection
```

---

## FILTERED Port

Typical response:

```text
No response
```

Meaning:

```text
Firewall or filtering device silently dropped packets
```

---

# Important OSCP Interpretation

## CLOSED

Usually indicates:

```text
Host reachable
No service listening
```

---

## FILTERED

Usually indicates:

```text
Firewall present
ACLs applied
IDS/IPS filtering
```

Filtered ports are often:

```text
More interesting than closed ports
```

because they reveal defensive controls.

---

# Nmap Timing Reality

## Important Practical Observation

Heavy filtering dramatically slows scans.

Example observed:

|Scan Type|Approximate Time|
|---|---|
|Top 1000 TCP scan|~15 minutes|
|UDP scan|~28 minutes|
|Combined TCP+UDP|~30 minutes|

---

# OSCP Practical Lesson

When ports are filtered:

```text
Nmap waits for timeouts
```

instead of receiving immediate rejection packets.

Result:

```text
Much slower scans
```

---

# UDP Scan Reality

## Important Observation

UDP scans can produce:

```text
open|filtered
```

state.

Meaning:

```text
Nmap cannot reliably determine whether:
- UDP port is open
OR
- Firewall silently dropped packets
```

---

# Critical UDP Insight

```text
No response ≠ confirmed open UDP port
```

This is one of the most important UDP concepts in OSCP.

---

# Host Discovery vs Port Scanning

## Important Observation

Host discovery may fail even when host is alive.

Example:

```text
Host seems down.
If it is really up, but blocking our ping probes, try -Pn
```

---

# Why This Happens

Some systems/firewalls block:

- ICMP Echo
    
- TCP discovery probes
    
- ARP responses
    
- ACK/SYN discovery packets
    

---

# Using -Pn

## Purpose

```bash
nmap -Pn target
```

Tells Nmap:

```text
Assume host is alive
Skip host discovery
```

---

# Important OSCP Usage

Use `-Pn` when:

- ping blocked
    
- ICMP filtered
    
- host discovery unreliable
    
- target obviously exists but appears down
    

---

# Important Syntax Mistake Learned

Incorrect:

```bash
nmap -Pn 1-65535 target
```

Result:

```text
Failed to resolve "1-65535"
```

---

# Correct Syntax

```bash
nmap -Pn -p 1-65535 target
```

---

# OSCP Lesson

Always remember:

```text
-p specifies ports
```

Without `-p`:

```text
Nmap interprets values as targets
```

---

# Network Sweep Practical Insight

## Observation

Host discovery scans:

```bash
nmap -sn subnet
```

may discover:

- gateways
    
- routers
    
- phones
    
- IoT devices
    
- workstations
    
- firewalled systems
    

---

# Important Enumeration Principle

A live host does NOT imply:

```text
Open ports
```

Example observed:

```text
Host up
1000 closed ports
```

---

# OSCP Enumeration Mindset

Even systems with no open ports may still:

- reveal MAC vendors
    
- reveal firewall behavior
    
- expose services later
    
- respond differently internally
    

---

# MAC Vendor Enumeration

## Observation

Nmap may identify vendor using MAC address.

Example:

```text
MAC Address: 20:05:05:60:86:5C
(Radmax Communication Private Limited)
```

---

# Practical Value

MAC vendors may help identify:

- routers
    
- IoT devices
    
- embedded systems
    
- virtualization platforms
    
- appliance vendors
    

---

# OS Detection Limitations

## Important Observation

OS fingerprinting may fail when:

- insufficient open ports
    
- insufficient responses
    
- aggressive filtering
    
- identical TCP/IP fingerprints
    

Example:

```text
Too many fingerprints match this host
```

---

# OSCP Interpretation

Reliable OS fingerprinting generally improves when:

- multiple ports open
    
- diverse services exposed
    
- less filtering present
    

---

# Service Detection Limitation

## Important Observation

Running:

```bash
nmap -sV target
```

against fully closed systems provides little value.

Example observed:

```text
All 1000 scanned ports closed
```

---

# OSCP Practical Rule

Run:

```bash
-sV
-A
NSE scripts
```

primarily against:

```text
confirmed open ports
```

to reduce:

- scan time
    
- noise
    
- unnecessary traffic
    

---

# Top Ports Scanning Insight

## Important Observation

Top-port scanning is extremely fast.

Example:

```bash
nmap --top-ports=20 target
```

completed in:

```text
~0.10 seconds
```

---

# OSCP Enumeration Strategy

Recommended workflow:

```text
Host Discovery
    ↓
Top Ports Scan
    ↓
Service Detection
    ↓
Targeted Deep Enumeration
    ↓
Selective Full Port Scan
```

---

# HTTPS Enumeration Insight

## Observation

Running:

```bash
nmap --script http-headers target
```

against HTTPS-only services may not produce useful results unless:

- correct port specified
    
- SSL/TLS handled correctly
    

---

# Practical OSCP Lesson

For HTTPS services:

Prefer:

```bash
curl -k https://target
```

or:

```bash
nmap -sV -p 443 164.100.161.148
```

before running web NSE scripts.

---

# Real-World Scan Behaviour Summary

## Your Observed Behaviours Included

|Behavior|Meaning|
|---|---|
|open|Service listening|
|closed|No service listening|
|filtered|Firewall silently dropping|
|open\|filtered|UDP ambiguity|
|Host seems down|Host discovery blocked|
|Slow scans|Timeout-heavy filtering|
|OS detection unreliable|Insufficient fingerprint data|

---

# Critical OSCP Takeaway

Advanced enumeration is NOT:

```text
Just running scans
```

It is:

```text
Interpreting network behavior correctly
```

---

# Final Practical OSCP Principle

When scanning:

```text
Every response matters
```

Including:

- silence
    
- resets
    
- timeouts
    
- latency
    
- filtering patterns
    
- inconsistent responses
    

All of them reveal information about the target environment.

# Most Important Practical Insight

Your target is behaving like:

```
Internet-exposed HTTPS servicebehind restrictive firewall/WAF
```

Therefore:

```
Web enumeration is now higher prioritythan broad port scanning
```

This is a critical OSCP mindset shift.

Once you identify:

- only 80/443 exposed
- heavy filtering
- likely hardened perimeter

you pivot from:

```
network enumeration
```

to:

```
application-layer enumeration
```

---

# Recommended Next Command Sequence

Exactly in this order:

```
nmap -sV -p 443 164.100.161.148
```

```
curl -vk https://164.100.161.148
```

```
curl -I -k https://164.100.161.148
```

```
nmap -p 443 --script ssl-cert,ssl-enum-ciphers 164.100.161.148
```

Then:

```
whatweb https://164.100.161.148
```
or:

```
nikto -h https://164.100.161.148
```

This is a very realistic OSCP-style progression.