# Shodan

# Objective

Use Shodan to passively identify:
- internet-facing systems
- exposed services
- open ports
- banners
- technologies
- vulnerable services
- exposed infrastructure

WITHOUT directly interacting with the target.

---

# What Shodan Is

Shodan is a search engine for:
- servers
- routers
- firewalls
- IoT devices
- industrial systems
- exposed services

Unlike Google:
- Google indexes web content
- Shodan indexes internet-connected devices and service banners

---

# OSCP Relevance

Useful for:
- passive recon
- attack surface discovery
- service enumeration
- version identification
- identifying public exposures

---

# Access

Website:

https://www.shodan.io/

Free account:
- limited searches
- enough for learning/basic recon

---

# Basic Searches

## Search by Hostname

```text
hostname:target.com
```

Example:

```text
hostname:megacorpone.com
```

---

## Search by Organization

```text
org:"Target Company"
```

---

## Search by IP

```text
ip:1.2.3.4
```

---

## Search by Netblock

```text
net:192.168.1.0/24
```

---

## Search by Port

```text
port:22
```

---

## Search Apache Servers

```text
apache
```

---

## Search OpenSSH

```text
product:"OpenSSH"
```

---

## Search Specific Version

```text
product:"OpenSSH" "7.2"
```

---

# Useful Filters

| Filter | Purpose |
|---|---|
| hostname: | search by domain |
| org: | organization search |
| port: | specific ports |
| country: | geographic filtering |
| product: | service/product |
| version: | software version |
| net: | subnet search |
| os: | operating system |

---

# Enumeration Workflow

## 1. Search Target

```text
hostname:target.com
```

---

## 2. Identify Hosts

Extract:
- IP addresses
- exposed systems
- cloud infrastructure
- VPN gateways

---

## 3. Review Open Ports

Look for:
- SSH
- RDP
- SMB
- FTP
- databases
- web services

---

## 4. Inspect Service Banners

Extract:
- software versions
- operating systems
- technologies
- server names

---

## 5. Identify Vulnerabilities

Shodan may show:
- known CVEs
- vulnerable services
- outdated software

---

# Important Data to Extract

| Information | Value |
|---|---|
| IPs | target expansion |
| ports | attack surface |
| banners | version detection |
| technologies | exploit selection |
| SSL info | certificates/domains |
| hosting providers | infrastructure mapping |

---

# Example Findings

## SSH Exposure

```text
OpenSSH 7.2p2 Ubuntu
```

Potential value:
- version-based exploits
- username enumeration
- weak configuration testing

---

## Web Technologies

```text
Apache
PHP
WordPress
```

Potential value:
- CMS exploitation
- plugin attacks
- version enumeration

---

# Important OSCP Insight

Shodan data is:
- historical
- passive
- sometimes outdated

Always verify later during active enumeration.

---

# Common OSCP Usage

Shodan is most useful for:
- external recon
- bug bounty
- real-world assessments

Less useful for:
- isolated lab environments
- internal networks
- exam machines

---

# CLI Usage

Install:

```bash
sudo apt install shodan
```

---

# Initialize API Key

```bash
shodan init API_KEY
```

---

# Basic CLI Search

```bash
shodan search apache
```

---

# Search Target Domain

```bash
shodan search hostname:target.com
```

---

# Host Lookup

```bash
shodan host IP
```

Example:

```bash
shodan host 8.8.8.8
```

---

# Download Results

```bash
shodan download results hostname:target.com
```

---

# Parse Downloaded Results

```bash
shodan parse results.json.gz
```

---

# High-Value Targets

Look carefully for:
- RDP exposed to internet
- Jenkins
- Elasticsearch
- Kibana
- MongoDB
- Redis
- Grafana
- Open FTP
- NAS devices
- VPN portals

---

# Common Recon Pivot

Shodan Finding:
- SSH exposed
- OpenSSH version identified

Next Steps:
- searchsploit
- Nmap verification
- username enumeration
- brute-force assessment

---

# Important Passive Recon Principle

Passive recon:
- reduces noise
- avoids detection
- improves targeting
- accelerates exploitation later

---

# Limitations

- Some results stale
- Limited free searches
- Internal hosts invisible
- Data depends on Shodan crawl timing

---

# OSCP Reality

For OSCP:
- understand concept
- know basic usage
- understand banner intelligence

But:
- active enumeration matters far more
- Nmap remains primary weapon

Shodan is supplementary reconnaissance.