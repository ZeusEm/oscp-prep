# Nmap NSE — Vulnerability Scanning

## TL;DR
Nmap has a built-in scripting engine (NSE) that extends it beyond port
scanning into lightweight vulnerability detection. It is **fully allowed on
the OSCP exam** (unlike Nessus). It uses small Lua scripts to test specific
services for known CVEs. Your two primary use cases: run the `vuln` category
for a broad sweep, or run a specific CVE script for a targeted check.

---

## What is the Nmap Scripting Engine (NSE)?

When nmap scans a target, by default it tells you: port is open, service is
X, version is Y. That is enumeration. NSE extends nmap to then **act on that
information** — querying the service, sending crafted packets, checking
specific conditions — and report back findings.

NSE scripts are written in a language called **Lua** and live on your Kali
machine at:

```
/usr/share/nmap/scripts/
```

Each script is a `.nse` file. Examples:
```
http-shellshock.nse
smb-vuln-ms17-010.nse
ftp-anon.nse
ssl-heartbleed.nse
```

The name tells you exactly what it does — `smb-vuln-ms17-010.nse` checks
whether the target is vulnerable to MS17-010 (EternalBlue) over SMB.

Think of NSE scripts as plugins. Nmap is the engine. Scripts are attachments
that give it specific detection abilities for specific vulnerabilities.

---

## NSE Script Categories — Why They Matter

Every NSE script is tagged with one or more categories. Categories tell you
**what the script does and whether it is safe to run**. This is critical
knowledge — running the wrong category against a fragile target can crash a
service.

| Category    | What It Means                                                |
|-------------|--------------------------------------------------------------|
| `safe`      | No risk to stability. Read-only operations.                  |
| `intrusive` | May crash services or alter state. Use with caution.         |
| `vuln`      | Checks for known vulnerabilities. May be safe OR intrusive.  |
| `exploit`   | Actively attempts exploitation. High impact.                 |
| `discovery` | Enumerates additional info about the target.                 |
| `brute`     | Brute forces credentials.                                    |
| `dos`       | Denial of service. Almost never run this.                    |
| `external`  | Makes requests to third-party services (e.g. Vulners API).   |

A single script can have **multiple categories**. For example a script tagged
`exploit, intrusive, vuln` will attempt to exploit the vulnerability and may
crash the service in the process.

**Exam rule:** Before running any NSE script you downloaded from the internet
or are unfamiliar with, check its category and read its header comments. A
script tagged `intrusive` or `exploit` that crashes a machine you need to
solve costs you points and a revert request.

---

## script.db — What It Is and How to Use It

Inside `/usr/share/nmap/scripts/` lives a file called `script.db`. This is
an **index file** — a text-based catalogue of every script nmap knows about,
mapping each filename to its categories.

When you run `--script vuln`, nmap reads `script.db`, finds all entries
tagged `vuln`, and loads those scripts. It does not scan the directory
directly — it reads the index. This is why when you add a new script manually,
you must update this index, or nmap will not know the script exists.

**Browsing the vuln category:**

```bash
grep "\"vuln\"" /usr/share/nmap/scripts/script.db
```

This shows you every script that includes `vuln` in its category list —
exactly what gets executed when you run `--script vuln`.

**Counting available vuln scripts:**

```bash
grep "\"vuln\"" /usr/share/nmap/scripts/script.db | wc -l
```

---

## NSE Script Documentation

Two ways to check what any script does before running it:

**Online:**
```
https://nmap.org/nsedoc/scripts/<script-name-without-.nse>
```
Example: `https://nmap.org/nsedoc/scripts/smb-vuln-ms17-010`

**Locally:**

```bash
# Read the script file itself — top section always has description and categories
head -40 /usr/share/nmap/scripts/smb-vuln-ms17-010.nse
```

---

## Execution Workflow

### Step 1 — Broad Vuln Sweep (vuln category)

Run all scripts in the `vuln` category against a target. Use this when you
want a general "what is broken on this service?" answer without knowing which
specific CVE to look for.

```bash
sudo nmap -sV -p <PORT> --script "vuln" <TARGET_IP>
```

Flag breakdown:
- `-sV` → service + version detection (mandatory — without version info,
  the vulners script cannot match CVEs and returns nothing)
- `-p <PORT>` → target a specific port. Avoids wasting time running every
  vuln script against every port. Run this per-service as you enumerate.
- `--script "vuln"` → loads all NSE scripts tagged with the `vuln` category
- No `-sV` means no version → no CVE matches → wasted scan. Always pair
  these two flags.

**Common targeted examples:**

```bash
# Web server vulnerability sweep
sudo nmap -sV -p 80,443 --script "vuln" <TARGET_IP>

# SMB vulnerability sweep (EternalBlue, etc.)
sudo nmap -sV -p 445 --script "vuln" <TARGET_IP>

# FTP vulnerability sweep
sudo nmap -sV -p 21 --script "vuln" <TARGET_IP>

# Full port + vuln sweep (slow but thorough — run in background)
sudo nmap -sV --script "vuln" <TARGET_IP> -oA vuln-scan-<TARGET_IP>
```

---

### Step 2 — Reading vulners Output

The **vulners** script is the most important script in the `vuln` category.
It takes the detected service name + version number, queries the Vulners
vulnerability database, and returns a list of every known CVE that affects
that version.

**What raw output looks like:**

```
PORT    STATE SERVICE VERSION
443/tcp open  http    Apache httpd 2.4.49 ((Unix))
| vulners:
|   cpe:/a:apache:http_server:2.4.49:
|       CVE-2021-41773  4.3   https://vulners.com/cve/CVE-2021-41773
|       CVE-2021-42013  9.8   https://vulners.com/cve/CVE-2021-42013  *EXPLOIT*
|       ...
```

**How to read it:**

| Field              | Meaning                                              |
|--------------------|------------------------------------------------------|
| `CVE-YYYY-NNNNN`   | The vulnerability identifier                         |
| The number (e.g. `9.8`) | CVSS score — higher = more severe              |
| URL                | Link to full CVE details on Vulners                  |
| `*EXPLOIT*`        | A public proof-of-concept exploit exists for this    |

**Triage logic:**
1. Filter for `*EXPLOIT*` entries first — these have working public exploits
2. Sort remaining by CVSS score — 9.0+ is critical, investigate immediately
3. For each candidate: note the CVE number, look it up on NVD, find the
   matching exploit

```bash
# Save output and grep for exploits immediately
sudo nmap -sV -p <PORT> --script "vuln" <TARGET_IP> | tee nmap-vuln-<TARGET_IP>.txt
grep "EXPLOIT" nmap-vuln-<TARGET_IP>.txt
```

> **Important:** vulners requires `-sV` and makes external requests to
> vulners.com (category: `external`). If the exam network blocks outbound
> internet access, vulners will return no results. In that case rely on
> targeted CVE scripts instead (Step 3).

---

### Step 3 — Targeted Single-CVE Scripts

When you know or suspect a specific vulnerability, use or find a dedicated
NSE script for that exact CVE. This is faster and gives a clean
VULNERABLE / NOT VULNERABLE result instead of a wall of output to parse.

**Nmap has several built-in CVE-specific scripts. High-value ones for OSCP:**

```bash
# MS17-010 EternalBlue (Windows SMB — extremely common on exam)
sudo nmap -sV -p 445 --script "smb-vuln-ms17-010" <TARGET_IP>

# MS08-067 (Windows XP/2003 SMB — older targets)
sudo nmap -sV -p 445 --script "smb-vuln-ms08-067" <TARGET_IP>

# Heartbleed (OpenSSL — HTTPS services)
sudo nmap -sV -p 443 --script "ssl-heartbleed" <TARGET_IP>

# Shellshock (Apache CGI / bash)
sudo nmap -sV -p 80,443 --script "http-shellshock" <TARGET_IP>

# Anonymous FTP login check
sudo nmap -sV -p 21 --script "ftp-anon" <TARGET_IP>

# SMB null session / dangerous settings
sudo nmap -sV -p 445 --script "smb-vuln-*" <TARGET_IP>
```

The wildcard `smb-vuln-*` runs all scripts whose names start with
`smb-vuln-` — useful for sweeping all known SMB CVEs at once.

---

### Step 4 — Installing Custom NSE Scripts

Sometimes no built-in script exists for a specific CVE. In that case you
find one online (usually GitHub) and install it manually. The workflow is:

**1. Find the script**

Search for: `CVE-YYYY-NNNNN nse site:github.com`

Always check the script source code before running it. Malicious scripts
disguised as CVE checkers exist. Read the file — it should do one clear,
specific thing. Reject any script that opens reverse shells, makes unexpected
outbound connections, or has obfuscated code.

**2. Download and place it in the correct directory**

```bash
# Download directly to the scripts directory
sudo wget https://raw.githubusercontent.com/<repo>/script-name.nse \
  -O /usr/share/nmap/scripts/http-vuln-cve<YEAR>-<NUMBER>.nse
```

Naming convention: `<protocol>-vuln-cve<YEAR>-<NUMBER>.nse`
Example: `http-vuln-cve2021-41773.nse`

Following the convention lets you find scripts easily later with tab
completion and grep.

**3. Update script.db**

```bash
sudo nmap --script-updatedb
```

This re-indexes all scripts in the directory. Without this step, nmap does
not know your new script exists and will throw a "no such script" error.

**4. Run the script**

```bash
sudo nmap -sV -p <PORT> --script "<script-name-without-.nse>" <TARGET_IP>
```

Example:
```bash
sudo nmap -sV -p 443 --script "http-vuln-cve2021-41773" 192.168.50.124
```

**Reading targeted script output:**

```
| http-vuln-cve2021-41773:
|   VULNERABLE:
|   Path traversal and file disclosure in Apache HTTP Server 2.4.49
|     State: VULNERABLE
|     Disclosure date: 2021-10-05
|     Check results:
|       Verify arbitrary file read:
|       https://<TARGET>/cgi-bin/.%2e/%2e%2e/etc/passwd
```

The output gives you: confirmation of vulnerability, a description, and
often a ready-made proof-of-concept request to verify it manually.

---

## Complete Exam-Day Workflow

```
For each open service found during enumeration:
        ↓
1. Run broad vuln sweep per service
   sudo nmap -sV -p <PORT> --script "vuln" <TARGET_IP>
        ↓
2. Grep output for *EXPLOIT* entries
   grep "EXPLOIT" output.txt
        ↓
3. Note all CVE numbers and CVSS scores
   Prioritise: CVSS 9.0+ and *EXPLOIT* flagged first
        ↓
4. Look up each CVE on NVD
   https://nvd.nist.gov/vuln/detail/CVE-YYYY-NNNNN
        ↓
5. Search for exploit or dedicated NSE script
   searchsploit <software> <version>
   Google: CVE-YYYY-NNNNN nse site:github.com
        ↓
6. If dedicated script found → install → run → confirm VULNERABLE
        ↓
7. Feed confirmed CVE into exploitation playbook
```

---

## Safe vs Intrusive — Decision Table

| Scenario                              | Use This                       |
|---------------------------------------|--------------------------------|
| Initial sweep, stability unknown      | `--script "safe and vuln"`     |
| Lab / exam machine (can revert)       | `--script "vuln"`              |
| Specific suspected CVE                | `--script "<specific-script>"` |
| Never in any scenario                 | `--script "dos"`               |
| Need version-matched CVE list         | `--script "vulners"` + `-sV`   |

---

## Gotchas & Exam Tips

- **`-sV` is mandatory with vulners.** Version detection is what vulners
  uses to query the CVE database. Without it, the script runs but returns
  nothing. This is the single most common reason vulners gives blank output.

- **vulners makes external requests.** On isolated exam/lab networks with
  no internet access, it will silently return no results. This is not a
  bug — fall back to specific CVE scripts that work offline.

- **`--script-updatedb` after every new script install.** If you skip this,
  nmap cannot find your script and fails with a cryptic error.

- **Never blindly run scripts from GitHub.** Read the source first. The
  script directory is on your attack machine — a malicious NSE script runs
  with sudo privileges. Verify it does exactly what the filename claims.

- **Wildcard syntax is powerful:** `--script "smb-vuln-*"` runs all SMB
  vuln scripts at once. Combine with port specification:
  `--script "smb-*" -p 445`.

- **Script output goes to file:** Always use `| tee filename.txt` or
  `-oA filename` so you can grep and re-read without re-running the scan.

- **Intrusive scripts on the exam:** The `vuln` category contains both safe
  and intrusive scripts. If a machine becomes unstable after a `--script vuln`
  run, the intrusive scripts in the category may be the cause. To play it
  safe against fragile targets: `--script "safe and vuln"`.

---

## Quick Reference — Key Commands

```bash
# Browse vuln category scripts
grep "\"vuln\"" /usr/share/nmap/scripts/script.db

# Broad vuln sweep on a specific port
sudo nmap -sV -p <PORT> --script "vuln" <TARGET_IP>

# Safe-only vuln sweep (won't crash services)
sudo nmap -sV -p <PORT> --script "safe and vuln" <TARGET_IP>

# Specific CVE script
sudo nmap -sV -p <PORT> --script "<script-name>" <TARGET_IP>

# All SMB vuln scripts at once
sudo nmap -sV -p 445 --script "smb-vuln-*" <TARGET_IP>

# Install custom script + update index
sudo cp <downloaded-script>.nse /usr/share/nmap/scripts/<name>.nse
sudo nmap --script-updatedb

# Save output + grep exploits
sudo nmap -sV -p <PORT> --script "vuln" <TARGET_IP> | tee nmap-vuln-<TARGET_IP>.txt
grep "EXPLOIT\|VULNERABLE" nmap-vuln-<TARGET_IP>.txt
```

---

## Next Steps After NSE Vuln Scan

- CVE confirmed → `searchsploit <software> <version>` for matching exploit
- CVE confirmed → look up on `https://nvd.nist.gov`
- EternalBlue (MS17-010) confirmed → SMB exploitation playbook
- Heartbleed confirmed → SSL exploitation playbook
- Web vuln confirmed → web application exploitation playbook

---

## Addendum — Real-World Scan Behaviour (Exam-Relevant Lessons)

The following lessons come from live scan outputs and are directly applicable
to exam day. The theory section above tells you how things work when everything
cooperates. This section tells you what to do when they don't.

---

### Lesson 1 — Port States: open vs closed vs filtered

When nmap scans a port, it doesn't just say "open" or "not open". It returns
one of three states, each with a different meaning and a different required
response from you.

| State      | What It Means                                               | What You Do                       |
|------------|-------------------------------------------------------------|-----------------------------------|
| `open`     | Port is reachable, a service is accepting connections       | Enumerate and scan                |
| `closed`   | Port is reachable, but no service is listening              | Note it; move on                  |
| `filtered` | A firewall is silently dropping your packets — no response  | Try different techniques (below)  |

**The critical distinction:** `closed` means the host responded and said "nothing
here". `filtered` means the host (or a firewall in front of it) said nothing at
all — your packet was dropped. NSE scripts cannot run against filtered ports
because there is no service to talk to. If you see `filtered` across most ports,
you are hitting a firewall, not an unresponsive host.

**From the scan output above:** ports 80, 443, 445 all showed `filtered` — a
firewall is present in front of this host blocking those ports entirely. This is
why no NSE script output was produced for those ports. There was nothing to
talk to.

---

### Lesson 2 — ICMP Blocked Does Not Mean the Host is Down

The ping test showed 100% packet loss — the host dropped every ICMP echo
request. A naive interpretation: "host is down, skip it." That would be wrong.
The subsequent nmap scans showed the host was up.

**Why this happens:** Firewalls commonly block ICMP (ping) as a first line of
defence. But the host is still reachable on other protocols.

**Why nmap still found it up:** nmap's default host discovery doesn't rely
solely on ICMP. It also sends TCP SYN packets to common ports (80, 443) and
checks for any response. If any probe gets a response, the host is marked alive.

**Exam implication:** On the exam, if a host appears down but your gut says
it shouldn't be, bypass host discovery entirely with `-Pn`:

```bash
# Skip host discovery — treat the host as up regardless of ping response
# and go straight to port scanning
sudo nmap -Pn -sV -p- <TARGET_IP>
```

`-Pn` tells nmap: "I know this host exists, don't waste time confirming it,
just scan." Use this when a target shows as down but is listed in your scope,
or when you know ICMP is being filtered.

```bash
# Combine with vuln scanning on specific ports
sudo nmap -Pn -sV -p 80,443,445 --script "vuln" <TARGET_IP>
```

---

### Lesson 3 — The `?` in Service Detection and Why It Destroys Your Time

Port 21 returned this:

```
21/tcp open  ftp?
```

That question mark is nmap saying: "something is listening on this port, but
it didn't respond the way I expected FTP to respond, so I'm not sure." The
scan of that single port took **343 seconds** — nearly 6 minutes.

**Why it took so long:** nmap's service detection (`-sV`) works by sending a
series of probe packets and waiting for a response that matches a known pattern.
When the service doesn't respond as expected, nmap exhausts its full probe list
and all their individual timeouts before giving up. With NSE scripts on top,
every script also times out waiting for valid responses. The result is a massive
time sink for a single uncertain port.

**What the `?` means for you:**
- The version detection failed — no version = vulners cannot match CVEs
- NSE scripts likely produced no output because they couldn't establish a
  proper session with the service
- The service may not be FTP at all — something else may be running on port 21,
  or it may be a honeypot

**What to do on exam day when you see `ftp?` or `service?`:**

```bash
# Step 1: Try connecting manually — bypass all tool timeouts
nc -nv <TARGET_IP> 21
```

Connect with netcat and read the banner manually. Almost every service sends
a greeting banner when you connect. Read it. It will tell you what is actually
running, or confirm there is nothing coherent there.

```bash
# Step 2: If it is FTP, grab the exact version string from the banner
# Then manually search: searchsploit vsftpd 2.3.4
# (or whatever version the banner shows)
```

```bash
# Step 3: If banner is empty or garbage, it may be a filtered port
# that nmap is misreading as open due to firewall behaviour.
# Note it and move on — don't let one ambiguous port consume 6 minutes.
```

**Exam time management rule:** If a service scan is taking longer than
2 minutes on a single port, kill it (`Ctrl+C`), manually banner-grab with
netcat, and continue scanning other ports in parallel. Never let nmap time
out silently while you wait idle.

---

### Lesson 4 — All Ports Filtered: What It Tells You

When the full scan returned:

```
All 1000 scanned ports on 164.100.161.148 are in ignored states.
Not shown: 1000 filtered tcp ports (no-response)
```

This means a perimeter firewall is blanket-dropping all TCP traffic on the
standard 1000 ports. On the exam, this typically means one of:

| Scenario | What to Try Next |
|---|---|
| Host is behind a stateful firewall | Try UDP scan: `sudo nmap -sU <TARGET_IP>` |
| Non-standard ports in use | Full port scan: `sudo nmap -p- <TARGET_IP>` |
| Host discovery is failing | Add `-Pn` to assume host is up |
| Firewall allows only specific source IPs | Check your exam scope — you may need a VPN or pivot |
| Wrong target IP | Verify the IP — scope document is ground truth |

```bash
# When standard 1000 ports are all filtered — escalate to full port range
sudo nmap -Pn -p- --min-rate 1000 <TARGET_IP> -oA fullscan-<TARGET_IP>
```

`--min-rate 1000` tells nmap to send at least 1000 packets per second.
This makes the full 65535-port scan tolerable in time. Without it, a `-p-`
scan can take 20+ minutes. Adjust down if the network is unstable.

```bash
# Also run a UDP sweep — services may be available on UDP when TCP is blocked
sudo nmap -Pn -sU --top-ports 100 <TARGET_IP>
```

---

### Quick Diagnosis Flow for Uncooperative Targets

```
Scan returns all filtered / no service version detected
        ↓
Is ICMP blocked?
  → Add -Pn to all subsequent scans
        ↓
Are only standard ports filtered?
  → Run full port scan: nmap -Pn -p- --min-rate 1000 <TARGET_IP>
  → Run UDP scan: nmap -Pn -sU --top-ports 100 <TARGET_IP>
        ↓
Service shows with ? (uncertain detection)?
  → Manual banner grab: nc -nv <TARGET_IP> <PORT>
  → Read banner → identify service → searchsploit manually
        ↓
Still nothing?
  → Confirm you have the right IP from scope
  → Request a revert if exam machine (may be crashed)
  → Move to next target — come back later
```
