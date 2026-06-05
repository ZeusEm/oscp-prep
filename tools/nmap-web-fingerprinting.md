# Nmap — Web Server Fingerprinting

## TL;DR
Before attacking a web application, you need to know what is running it.
Nmap's `-sV` flag grabs the web server banner (name + version). The
`http-enum` NSE script then probes for directories, files, and known paths.
Together they give you version intelligence for CVE hunting and a map of
the application's structure — both in under 60 seconds.

---

## What is a Web Server Banner?

When you connect to a web server, it announces itself. This announcement is
called a **banner** — a string the server sends in its HTTP response headers
that typically contains:

- The web server software name (Apache, Nginx, IIS, LiteSpeed)
- The version number (2.4.41, 1.18.0, 10.0)
- Sometimes the underlying OS (Ubuntu, Debian, Win64)

Example banner as seen in an HTTP response header:
```
Server: Apache/2.4.41 (Ubuntu)
```

**Why does this matter?**

The banner is your entry point into the CVE pipeline. Once you know the
exact version, you can:

```
Apache 2.4.41 detected
        ↓
searchsploit apache 2.4.41
        ↓
Cross-reference with nmap vulners output
        ↓
Identify exploitable CVEs for this exact version
```

A version number is not just a label — it is a key that unlocks the list
of known vulnerabilities for that software. This is why fingerprinting
always comes before any exploitation attempt.

**What if the banner is missing or suppressed?**

Some hardened servers disable or falsify their banner (`Server: Apache`
with no version, or `Server: webserver`). This is intentional obfuscation.
In that case, fall back to behaviour-based fingerprinting: error page
format, header ordering, default file paths, and response timing can all
reveal the true server even without a banner.

---

## What is http-enum?

`http-enum` is an NSE script that performs **automated web path discovery**
against a target web server. It works by sending HTTP GET requests for a
built-in list of commonly known paths — admin panels, login pages, config
files, backup directories, database folders, upload directories — and
reporting which ones exist (return HTTP 200 or similar response) versus
which ones don't (return HTTP 404).

Think of it as an automated version of manually typing:
```
http://TARGET/admin/
http://TARGET/login.php
http://TARGET/uploads/
http://TARGET/.git/
... (and thousands more)
```

It doesn't brute force with a massive wordlist like Gobuster does — it uses
a curated list of **high-probability paths** known to exist in common web
applications and frameworks. This makes it fast and low-noise as a first
pass. Use Gobuster for deeper directory enumeration afterwards.

---

## Execution Workflow

### Step 1 — Banner Grab (Web Server Version)

```bash
sudo nmap -p80 -sV <TARGET_IP>
```

For HTTPS targets:
```bash
sudo nmap -p443 -sV <TARGET_IP>
```

For both HTTP and HTTPS simultaneously:
```bash
sudo nmap -p80,443 -sV <TARGET_IP>
```

**What you get:**
```
PORT   STATE SERVICE VERSION
80/tcp open  http    Apache httpd 2.4.41 ((Ubuntu))
```

Note down immediately:
- Web server name: `Apache`
- Version: `2.4.41`
- OS hint: `Ubuntu`

This feeds directly into your CVE search:
```bash
searchsploit apache 2.4.41
```

---

### Step 2 — NSE http-enum (Path Discovery + Extended Fingerprint)

```bash
sudo nmap -p80 --script=http-enum <TARGET_IP>
```

For HTTPS:
```bash
sudo nmap -p443 --script=http-enum <TARGET_IP>
```

Combine with `-sV` to get version AND enumeration in one pass:
```bash
sudo nmap -p80,443 -sV --script=http-enum <TARGET_IP>
```

Save output for reference:
```bash
sudo nmap -p80 -sV --script=http-enum <TARGET_IP> | tee nmap-web-<TARGET_IP>.txt
```

---

## Reading http-enum Output — What Each Finding Means

```
| http-enum:
| /login.php: Possible admin folder
| /db/: BlogWorx Database
| /css/: Potentially interesting directory w/ listing on 'apache/2.4.41'
| /db/: Potentially interesting directory w/ listing on 'apache/2.4.41'
| /images/: Potentially interesting directory w/ listing on 'apache/2.4.41'
| /js/: Potentially interesting directory w/ listing on 'apache/2.4.41'
|_ /uploads/: Potentially interesting directory w/ listing on 'apache/2.4.41'
```

Break this output into two categories and act on them differently:

### Category 1 — High-Value Paths (Act Immediately)

| Path Found | What It Signals | What to Do |
|---|---|---|
| `/login.php` | Authentication page | Test default credentials; check for SQLi in login form; check for username enumeration |
| `/admin/` | Admin panel | Access it — does it require auth? If no auth, you're in. If auth, test default creds. |
| `/db/` | Database directory | Browse it directly. May contain `.sql` dump files with credentials |
| `/uploads/` | File upload directory | Can you upload files? Can you upload `.php`? If yes → RCE path |
| `/backup/` | Backup files | May contain source code, config files, credentials |
| `/.git/` | Git repository | Source code exposed → download entire repo → read for credentials, logic flaws |
| `/config.php` | Configuration file | May be readable — often contains database credentials |
| `/phpinfo.php` | PHP info page | Reveals PHP version, loaded modules, server paths, environment variables |

### Category 2 — Directory Listing (Understand What This Is)

The phrase `w/ listing on 'apache/2.4.41'` in the output is critical.

**What is directory listing?**

Normally when you navigate to `http://TARGET/images/` in a browser, the
web server should either show you a specific webpage or return a 403
Forbidden. It should NOT show you the contents of that folder.

Directory listing is a **misconfiguration** where the web server has been
told to display the actual file contents of a directory when no index file
exists — like Windows Explorer, but in your browser. You see every file in
that folder, and you can click any of them to download or view it.

This is dangerous because:
- `/uploads/` with directory listing shows every uploaded file — and if
  you can upload a `.php` file, you can now navigate to it and execute it
- `/db/` with directory listing may show `.sql` backup files containing
  full database dumps including usernames and password hashes
- `/css/`, `/js/` listing seems harmless but confirms misconfiguration —
  if listing is on everywhere, look for sensitive files everywhere

**Every directory in the http-enum output marked with `w/ listing` should
be manually browsed in a web browser or with curl:**

```bash
curl -s http://<TARGET_IP>/db/
curl -s http://<TARGET_IP>/uploads/
```

Read the file listing returned. Look for `.sql`, `.bak`, `.zip`, `.conf`,
`.php`, `.txt` files. Download and read anything that looks sensitive.

---

## What the Output Also Tells You — Framework Fingerprinting

The string `BlogWorx Database` appearing next to `/db/` is not generic
nmap output — it is the name of the application or framework. When http-enum
identifies a specific product name, that is your cue to:

```bash
# Search for known exploits for this specific application
searchsploit blogworx
searchsploit <framework-name>
```

Combined with the web server version and OS from `-sV`, you now have a
multi-layer fingerprint:
```
Web server:   Apache 2.4.41 (Ubuntu)
Application:  BlogWorx
```

Each layer has its own CVE history. Check both.

---

## Putting It Together — Exam-Day Fingerprinting Sequence

```bash
# 1. Version detection — grab banner, feed into CVE search
sudo nmap -p80,443 -sV <TARGET_IP>

# 2. Path discovery — find the application's structure
sudo nmap -p80 --script=http-enum <TARGET_IP>

# 3. Combined one-liner — run both in a single pass, save output
sudo nmap -p80,443 -sV --script=http-enum <TARGET_IP> | tee nmap-web-<TARGET_IP>.txt

# 4. Version intel — search for exploits immediately
searchsploit <server-name> <version>
# e.g. searchsploit apache 2.4.41

# 5. Browse every interesting path manually
curl -s http://<TARGET_IP>/<path>/
# OR open in browser for visual directory listing

# 6. For HTTPS targets — same commands, swap port
sudo nmap -p443 -sV --script=http-enum <TARGET_IP>
```

---

## What http-enum Does NOT Cover

`http-enum` uses a curated list — it is fast but not exhaustive. After
running it, always follow up with Gobuster for deeper discovery:

```bash
# Gobuster with a larger wordlist after http-enum
gobuster dir -u http://<TARGET_IP> -w /usr/share/wordlists/dirb/common.txt
```

Think of it as two passes:
- `http-enum` → quick high-probability sweep (fast, good first intel)
- `gobuster` → thorough brute-force sweep (slower, deeper coverage)

Always run both. http-enum is not a replacement for Gobuster, it is a
complement to it.

---

## Gotchas & Exam Tips

- **Always fingerprint before exploiting.** A wrong assumption about the
  server version wastes time building exploits that won't work.

- **The version in the banner can be wrong.** Backporting means the banner
  says `2.4.41` but a security patch from a later version has been applied.
  If an exploit fails against a version the scanner confirmed, backporting
  is likely the cause. Move on to other vectors.

- **Directory listing is misconfiguration, not exploitation.** Finding it
  does not mean you are done — it means you have read access to file names.
  You still need to find something sensitive in those directories.

- **`/uploads/` with directory listing + file upload functionality = RCE
  path.** This is one of the most common web app attack chains on OSCP.
  Note it immediately and follow the file upload exploitation playbook.

- **`/login.php` is not just a brute force target.** It is also a SQLi
  target, a username enumeration target, and an information disclosure
  target (error messages). Look at it from all angles.

- **Run http-enum on non-standard ports too.** Web apps frequently run on
  ports 8080, 8443, 8000, 3000. If nmap finds an HTTP service on any port,
  run http-enum against that port:
```bash
  sudo nmap -p8080 -sV --script=http-enum <TARGET_IP>
```

- **http-enum is relatively noisy.** It sends many requests rapidly. On
  production systems in a real engagement this would trigger IDS alerts.
  On the OSCP exam environment it is expected and acceptable.

---

## Quick Reference — Key Commands

```bash
# Banner grab only
sudo nmap -p80 -sV <TARGET_IP>

# http-enum only
sudo nmap -p80 --script=http-enum <TARGET_IP>

# Both combined — standard first-pass command for any web target
sudo nmap -p80,443 -sV --script=http-enum <TARGET_IP> | tee nmap-web-<TARGET_IP>.txt

# Non-standard web ports
sudo nmap -p8080,8443,8000 -sV --script=http-enum <TARGET_IP>

# Browse a discovered directory
curl -s http://<TARGET_IP>/<PATH>/

# Exploit search from version
searchsploit <software> <version>
```

---

## Next Steps After Web Fingerprinting

- Version found → `searchsploit <server> <version>`
- Login page found → `playbooks/web-login-attacks.md`
- Upload directory found → `playbooks/file-upload-exploitation.md`
- Database directory found → browse it, look for `.sql` dumps → extract
  credentials
- Framework identified → `searchsploit <framework-name>`
- Deeper directory enumeration needed → `tools/gobuster.md`
- Manual parameter testing → `tools/burp-suite.md`

---

## Addendum — HTTPS Fingerprinting Failures and How to Recover

The following lessons come from live scan behaviour against a hardened HTTPS
target. These scenarios appear frequently on exam machines.

---

### Lesson 1 — HTTP Filtered, HTTPS Open: Redirect Everything

Port 80 showed `filtered`. Port 443 showed `open`. This is a deliberate
security configuration — the server is forcing all traffic to HTTPS and
silently dropping plain HTTP.

**What not to do:** Spend time trying to get http-enum working on port 80.
The firewall will drop every packet. There is nothing there for you.

**What to do:** The moment you see this pattern, redirect your entire web
enumeration workflow to HTTPS:

```bash
# All further web commands use https:// and port 443
curl -k https://<TARGET_IP>/
sudo nmap -p443 -sV --script=http-enum <TARGET_IP>
gobuster dir -u https://<TARGET_IP> -w <WORDLIST> -k
```

The `-k` flag in curl and gobuster tells the tool to **ignore SSL certificate
errors**. On exam machines and lab environments, SSL certificates are
frequently self-signed or expired. Without `-k`, curl refuses to connect.
`-k` bypasses certificate validation so you can still interact with the
service.

**Pattern to memorise:**
```
Port 80 filtered + Port 443 open = HTTPS-only target
→ Add -k to all web tools
→ Replace http:// with https:// everywhere
→ Never waste time on the filtered port
```

---

### Lesson 2 — `ssl/https?` Means Version Detection Failed

Port 443 returned this:
```
443/tcp open  ssl/https?
```

That question mark means: nmap detected a TLS/SSL handshake occurred (the
port is genuinely open and something is listening), but it could not identify
what application is running behind the SSL layer. The version detection probes
got no recognisable response.

**Consequence:** Without a confirmed service type and version, the vulners
script produces nothing, http-enum produces nothing, and `-sV` gives you no
actionable intelligence. This is not a dead end — it means nmap's automated
approach failed and you need to manually take over.

**Recovery Step 1 — Manual banner grab with curl:**

```bash
# -k ignores cert errors, -v shows full response headers including Server:
curl -k -v https://<TARGET_IP>/ 2>&1 | head -50
```

The `-v` (verbose) flag makes curl print every HTTP request and response
header. The `Server:` header in the response is your banner. You will see
something like:
```
< Server: Apache/2.4.41 (Ubuntu)
```
or
```
< Server: nginx/1.18.0
```

Even if nmap failed, curl almost always gets a response because it speaks
HTTP natively at the application layer, not just TCP/SSL at the network layer.

```bash
# Quieter version — just print response headers
curl -k -I https://<TARGET_IP>/
```

`-I` sends a HEAD request — asks for headers only, no body. Fast and clean
for banner grabbing.

---

### Lesson 3 — SNI: Why Scanning by IP Fails on HTTPS

This is a fundamental HTTPS concept that directly affects how you scan
on the exam.

**What is SNI (Server Name Indication)?**

A single server can host dozens of different websites, all on the same IP
address and the same port 443. When your browser connects to `bank.com`, the
server needs to know which website you want before it can present the right
SSL certificate and serve the right content. SNI is the mechanism that solves
this — your browser sends the hostname (`bank.com`) at the start of the TLS
handshake so the server knows which site to serve.

**The problem:** When nmap scans by IP address only (`164.100.161.148`), it
does not send an SNI hostname during the TLS handshake. The server receives
a TLS connection with no indication of which site is being requested. Depending
on the server configuration, it may:
- Serve a default/empty response that nmap cannot fingerprint → `ssl/https?`
- Return a certificate mismatch error
- Serve a completely different site than the one you expect

**The fix:** If you have a hostname for the target, always include it. Use
`--script-args` to pass the hostname to NSE scripts, or scan using the
hostname directly:

```bash
# Scan using hostname instead of IP — SNI is sent correctly
sudo nmap -p443 -sV --script=http-enum <HOSTNAME>
# e.g. sudo nmap -p443 -sV --script=http-enum joinindiannavy.gov.in
```

```bash
# Force nmap to use a specific hostname for HTTP scripts via script args
sudo nmap -p443 -sV --script=http-enum \
  --script-args="http.host=<HOSTNAME>" <TARGET_IP>
```

```bash
# curl with explicit hostname resolving to specific IP
# (useful when DNS is unreliable or you want to confirm a specific IP)
curl -k --resolve <HOSTNAME>:443:<TARGET_IP> https://<HOSTNAME>/
# e.g. curl -k --resolve joinindiannavy.gov.in:443:164.100.161.148 \
#   https://joinindiannavy.gov.in/
```

**On the OSCP exam:** Targets have entries in `/etc/hosts` mapping hostnames
to IPs. Always check `/etc/hosts` for the target's hostname and use it for
web enumeration. Scanning HTTPS targets by IP alone frequently produces
degraded results exactly like the `ssl/https?` output above.

```bash
# Check /etc/hosts for known hostnames on exam day
cat /etc/hosts
```

---

### Lesson 4 — SSL Certificate Intelligence

When nmap fails to fingerprint an HTTPS service, the SSL certificate itself
is a rich intelligence source that costs nothing to read. Every HTTPS server
presents its certificate during the TLS handshake and you can read it without
any credentials.

**What SSL certificates reveal:**
- **Common Name (CN):** The primary hostname the cert was issued for
- **Subject Alternative Names (SANs):** Every other hostname the cert is valid
  for — subdomains, internal hostnames, service names
- **Organisation (O):** The company or entity that requested the cert
- **Issuer:** Who signed the cert (Let's Encrypt = recent, self-signed = lab/
  internal, internal CA = corporate environment)
- **Validity dates:** When it was issued and when it expires

On an exam machine, the CN and SANs frequently reveal the target's hostname
that you need to add to `/etc/hosts`, or reveal additional subdomains that
are in scope.

```bash
# Read the full SSL certificate
openssl s_client -connect <TARGET_IP>:443 </dev/null 2>/dev/null \
  | openssl x509 -noout -text
```

```bash
# Faster — just the key fields (CN, SANs, dates)
openssl s_client -connect <TARGET_IP>:443 </dev/null 2>/dev/null \
  | openssl x509 -noout -subject -issuer -dates -ext subjectAltName
```

**Reading the output — what to extract:**

```
subject=CN=joinindiannavy.gov.in         ← primary hostname
issuer=CN=Let's Encrypt Authority X3     ← publicly trusted cert
Not Before: Jan  1 00:00:00 2024 GMT
Not After : Apr  1 00:00:00 2024 GMT     ← expired = possibly outdated server
X509v3 Subject Alternative Names:
    DNS:joinindiannavy.gov.in
    DNS:www.joinindiannavy.gov.in        ← additional hostnames to enumerate
    DNS:internal-admin.gov.in            ← internal hostname exposed
```

Any hostname found in the certificate that you did not already know about is a
new enumeration target. Add it to `/etc/hosts` and run your web enumeration
against it.

```bash
# Add discovered hostname to /etc/hosts
echo "<TARGET_IP> <DISCOVERED_HOSTNAME>" | sudo tee -a /etc/hosts
```

---

### Lesson 5 — http-enum Silence Is Not Proof of Nothing

When http-enum runs against a target and produces no output at all (not even
"nothing found"), it may mean:

| Reason | How to Confirm |
|--------|----------------|
| Service type unconfirmed (`https?`) | Use curl first to verify HTTP is actually served |
| SNI required — IP scan got wrong response | Rescan using hostname |
| NSE scripts did not fire due to uncertain service | Use `--script-args`

---

### Lesson 6 — curl Error 52: What "Empty Reply from Server" Actually Means

Running `curl -k -I https://<TARGET_IP>/` returned:
```
curl: (52) Empty reply from server
```

This error means: the TLS handshake completed successfully (the connection
was established) but the server sent zero bytes in response to your HTTP
request and then closed the connection. It is not a network failure — it is
the server deliberately ignoring your request at the HTTP layer.

**Why this happens:** When curl uses an IP address as the target, it sends
that IP in the HTTP `Host:` header:
```
GET / HTTP/1.1
Host: 164.100.161.148
```

A server running virtual hosts (multiple websites on one IP) uses the `Host:`
header to decide which site to serve. When it receives an IP address instead
of a hostname it recognises, it has no matching virtual host and closes the
connection with no response.

The TLS connection succeeding but HTTP returning nothing is the fingerprint
of a **virtual host misconfiguration from your side** — you are speaking to
the right server but asking for the wrong host.

**The fix is always `--resolve`:**
```bash
curl -k --resolve <HOSTNAME>:443:<TARGET_IP> https://<HOSTNAME>/
```

This forces curl to connect to `<TARGET_IP>` but send `<HOSTNAME>` in the
`Host:` header and the TLS SNI field simultaneously. The server receives a
request it recognises and responds.

**Diagnosis table for curl failures on HTTPS:**

| Error | Meaning | Fix |
|-------|---------|-----|
| `curl: (52) Empty reply` | Server got request, sent nothing back — virtual host issue | Use `--resolve` with correct hostname |
| `curl: (60) SSL certificate problem` | Cert validation failed | Add `-k` flag |
| `curl: (7) Failed to connect` | Port closed or filtered | Confirm port is open with nmap |
| `curl: (35) SSL handshake failed` | TLS version or cipher mismatch | Try `--tlsv1.2` or `--tlsv1` |

---

### Lesson 7 — Server Identity From Response Body, Not Headers

When scanning by IP produced no HTTP response, the server type was unknown.
But when `--resolve` was used with the correct hostname, the 301 redirect
response body revealed it immediately:

```html
<center>nginx</center>
```

This is a critical habit: **when HTTP response headers are absent or
uninformative, read the response body**. Web servers embed their identity in:
- Default error pages (`403 Forbidden` pages, `404 Not Found` pages)
- Redirect response bodies (as above)
- Default index pages when no application is present

This matters because a server that suppresses its `Server:` header in normal
responses (a common hardening measure) will still leak its identity in error
and redirect pages it generates automatically.

```bash
# Get the body of a redirect response without following it
curl -k --resolve <HOSTNAME>:443:<TARGET_IP> https://<HOSTNAME>/

# Grep for common server identities in any response
curl -k --resolve <HOSTNAME>:443:<TARGET_IP> https://<HOSTNAME>/ \
  | grep -i "nginx\|apache\|iis\|server\|powered"
```

In this case: nmap said `ssl/https?` (unknown). curl said `nginx`. curl won.
When nmap cannot fingerprint a service, curl with the correct hostname is your
fallback fingerprinting tool.

---

### Lesson 8 — Always Follow 301/302 Redirects

The `--resolve` command returned a `301 Moved Permanently` — but the
destination of that redirect was not shown. A redirect is not a dead end,
it is a pointer to where the actual content lives. Always follow it:

```bash
# -L tells curl to follow redirects automatically
curl -k -L --resolve <HOSTNAME>:443:<TARGET_IP> https://<HOSTNAME>/
```

On the exam, redirects commonly point to:
- A different hostname (reveals a new virtual host to enumerate)
- A specific path like `/index.php` or `/app/` (reveals the app entry point)
- A login page (your primary target)
- An entirely different port

```bash
# Show each redirect hop explicitly — useful when chains are long
curl -k -L -v --resolve <HOSTNAME>:443:<TARGET_IP> https://<HOSTNAME>/ \
  2>&1 | grep -E "^< Location:|^> Host:|HTTP/"
```

This prints every redirect destination and the final Host header being sent —
the full redirect chain in one readable output.

---

### Lesson 9 — `--script-args http.host` Does Not Fix `-sV` Detection

Passing `--script-args="http.host=<HOSTNAME>"` to nmap still returned
`ssl/https?` with no http-enum output. This is a technical limitation worth
understanding so you do not waste time on it during the exam.

Nmap's operation has two separate layers:

```
Layer 1: Service Detection (-sV)
  → Sends its own binary probes to fingerprint the service
  → Does NOT use --script-args settings
  → If it cannot confirm "this is HTTP", it marks service as unknown

Layer 2: NSE Scripts (--script)
  → Only fires against services that -sV successfully identified
  → http-enum will only run if -sV confirmed an HTTP service
  → --script-args http.host affects how scripts construct their Host header
     but only matters if the scripts actually run
```

If `-sV` cannot confirm the service type, NSE HTTP scripts do not run
regardless of what `--script-args` you pass. `--script-args http.host` is
useful when nmap already knows the service is HTTP but needs the correct
hostname for scripts to get real responses — it does not rescue failed
service detection.

**Conclusion:** When nmap returns `ssl/https?` and http-enum produces nothing,
nmap's automated approach has hit its limit. Stop trying to fix nmap and
switch to curl + manual enumeration with the correct hostname:

```bash
# This is more reliable than any nmap workaround for HTTPS+virtual host targets
curl -k -L --resolve <HOSTNAME>:443:<TARGET_IP> https://<HOSTNAME>/
gobuster dir -u https://<HOSTNAME> -w <WORDLIST> -k
```

Gobuster handles SNI and virtual hosts correctly when given the hostname as
the target URL — it sends the proper `Host:` header on every request.
