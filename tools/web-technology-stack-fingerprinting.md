# Technology Stack Fingerprinting

## TL;DR
Before exploiting a web application you need to know what it is built with —
the web server, programming language, framework, CMS, JavaScript libraries,
and OS underneath. Each component has its own CVE history. Wappalyzer does
this passively via a browser extension or web lookup. WhatWeb does the same
from the command line and is your primary exam-day tool since it requires no
browser or account.

---

## What is a Technology Stack?

A web application is not a single piece of software — it is a layered assembly
of components, each built by different teams, each with its own version history
and known vulnerabilities. This assembly is called the **technology stack**.

A typical stack looks like this, from bottom to top:

```
Operating System         → Ubuntu 20.04, Windows Server 2019
        ↓
Web Server               → Apache 2.4.41, Nginx 1.18, IIS 10.0
        ↓
Application Runtime      → PHP 7.4, Python 3.8, ASP.NET 4.8, Node.js
        ↓
Application Framework    → WordPress, Drupal, Laravel, Django, Rails
        ↓
JavaScript Libraries     → jQuery 1.8.3, Angular 9.0, React 16.8
        ↓
Frontend UI Framework    → Bootstrap 4.0, Materialize
```

**Why every layer matters:**

Each layer has its own CVE database. jQuery 1.8.3 has known XSS
vulnerabilities. WordPress 5.1 has known RCE paths. PHP 7.4 has known type
juggling flaws. Apache 2.4.49 has a path traversal CVE.

Identifying the stack means you know which CVE databases to search in. Without
it, you are guessing. With it, you have a structured list of hypotheses to
test — one per layer, working from the most commonly exploitable downward.

---

## Passive vs Active Fingerprinting

This distinction matters because it determines risk and detectability.

**Active fingerprinting:** You send probes to the target (nmap, http-enum,
gobuster). The target receives your traffic. Logs are generated. IDS may
alert. This is what you have been doing.

**Passive fingerprinting:** You observe information the target already makes
publicly available without sending any unusual or probing traffic. The target
sees only normal browsing behaviour or nothing at all (if a third-party
service fetches the data for you).

Wappalyzer works both ways:
- **Browser extension:** Passively reads the page your browser loads normally
  — examines response headers, HTML structure, JavaScript files, cookies. Your
  browsing is the only traffic generated.
- **Web lookup on wappalyzer.com:** A third-party server fetches the target
  page on your behalf. The target sees a request from Wappalyzer's IP, not
  yours. Fully passive from your perspective.

---

## What Wappalyzer Is

Wappalyzer is a **technology fingerprinting service** available as:
- A browser extension (Chrome, Firefox) — analyses pages as you browse them
- A web-based lookup at `https://www.wappalyzer.com/`

It works by matching observable evidence from the page against a database of
known fingerprints. Observable evidence includes:

| Evidence Source | What It Reveals |
|---|---|
| HTTP response headers | Web server, framework, caching layer |
| HTML meta tags | CMS, generator tags, framework hints |
| JavaScript file names and content | JS libraries, versions, frameworks |
| Cookie names | Framework (e.g. `PHPSESSID` = PHP, `JSESSIONID` = Java) |
| URL structure patterns | CMS routing conventions |
| HTML comments | Framework or generator comments left by developers |

**What Wappalyzer reports:** OS, web server, programming language, CMS,
JavaScript libraries with version numbers, UI frameworks, analytics tools,
CDN providers, and more — all from a single page load.

---

## Using Wappalyzer

### Method 1 — Browser Extension (Passive, Preferred for OSCP Labs)

1. Install from Chrome Web Store or Firefox Add-ons (search "Wappalyzer")
2. Navigate to the target web application in your browser
3. Click the Wappalyzer icon in your browser toolbar
4. Read the detected technologies — categorised and version-labelled

No account required for the extension. No traffic beyond normal page load.

### Method 2 — Web Lookup (Fully Passive from Your Machine)

1. Go to `https://www.wappalyzer.com/`
2. Enter the target domain in the lookup bar
3. Read the results — same categories as the extension

**Limitation:** Requires a free account for more than a few lookups.
**Also note:** Only works on publicly accessible targets. Exam machines are
on a private VPN network — Wappalyzer's servers cannot reach them. The
browser extension is what you use on exam machines.

---

## WhatWeb — The Exam-Day Command Line Alternative

Wappalyzer's browser extension is useful but has a critical limitation for the
exam: it requires a browser session and manual navigation. **WhatWeb** is the
command-line equivalent — it performs the same technology fingerprinting
automatically, runs from your terminal, outputs greppable text, and is
**pre-installed on Kali Linux**.

For the OSCP exam, WhatWeb is your primary stack fingerprinting tool.

### What WhatWeb Is

WhatWeb sends HTTP requests to the target and analyses the response against
a database of over 1,800 technology fingerprints. It identifies the same
categories as Wappalyzer — web server, CMS, frameworks, libraries, OS hints
— and outputs them with version information where detectable.

### WhatWeb Commands

```bash
# Basic scan — fast, low noise, immediate intel
whatweb http://<TARGET_IP>

# HTTPS target
whatweb https://<TARGET_IP>

# Scan by hostname (correct for virtual host targets)
whatweb https://<HOSTNAME>

# Verbose output — shows every matched fingerprint and the evidence
whatweb -v http://<TARGET_IP>

# Aggressive mode — sends additional probes for more complete detection
whatweb -a 3 http://<TARGET_IP>

# Ignore SSL certificate errors
whatweb --no-check-certificate https://<TARGET_IP>

# Combine: aggressive + no cert check + verbose, save to file
whatweb -a 3 -v --no-check-certificate https://<TARGET_IP> \
  | tee whatweb-<TARGET_IP>.txt
```

**Aggression levels explained:**

| Level | Flag | Behaviour |
|-------|------|-----------|
| 1 | `-a 1` | Single request only — stealthy |
| 2 | `-a 2` | No 404 pages fetched |
| 3 | `-a 3` | Default aggressive — fetches additional pages |
| 4 | `-a 4` | Makes requests for every URL found — very noisy |

Default is level 1. Level 3 is the best balance of depth and noise for
exam lab environments.

### Reading WhatWeb Output

```
http://192.168.50.20 [200 OK]
Apache[2.4.41],
Bootstrap[4.3.1],
Country[RESERVED][ZZ],
HTML5,
HTTPServer[Ubuntu Linux][Apache/2.4.41 (Ubuntu)],
IP[192.168.50.20],
JQuery[3.3.1],
PHP[7.4.3],
WordPress[5.7],
X-Powered-By[PHP/7.4.3]
```

Each bracketed value is the detected version. Read this as a structured
list of CVE search targets:

```
Apache 2.4.41    → searchsploit apache 2.4.41
WordPress 5.7    → searchsploit wordpress 5.7
PHP 7.4.3        → searchsploit php 7.4.3
jQuery 3.3.1     → searchsploit jquery 3.3.1
Bootstrap 4.3.1  → low risk, skip unless nothing else found
```

---

## What to Do With Every Component You Identify

This is the workflow that converts a technology stack into an attack path:

```bash
# For every component + version found, run searchsploit
searchsploit <component> <version>

# Examples from a typical WordPress stack
searchsploit apache 2.4.41
searchsploit wordpress 5.7
searchsploit php 7.4
searchsploit jquery 3.3.1

# Also search for the CMS specifically
searchsploit wordpress       # all WordPress exploits
searchsploit wordpress theme  # theme-specific vulns (very common on exam)
searchsploit wordpress plugin # plugin vulns (extremely common on exam)
```

**Priority order for exploitation research:**

```
1. CMS core version (WordPress, Drupal, Joomla) → most likely RCE path
2. CMS plugins and themes                        → second most common
3. Web server version (Apache, Nginx, IIS)       → check for path traversal, CVEs
4. Application runtime (PHP, ASP.NET version)    → check for type juggling, CVEs
5. JavaScript libraries                          → jQuery old versions → XSS
```

JavaScript library vulnerabilities (jQuery, Angular) are almost exclusively
client-side XSS — they are lower priority on OSCP where the goal is server
access, not XSS. However they confirm the app is not regularly updated, which
implies the server-side components are also likely outdated.

---

## Cookie Names as Passive Fingerprints

One of the fastest ways to identify the backend technology without any tools
is reading the session cookie name in your browser's developer tools
(`F12 → Application → Cookies`):

| Cookie Name | Technology |
|---|---|
| `PHPSESSID` | PHP |
| `JSESSIONID` | Java (Tomcat, JBoss, Spring) |
| `ASP.NET_SessionId` | ASP.NET |
| `rack.session` | Ruby on Rails |
| `connect.sid` | Node.js (Express) |
| `wordpress_logged_in_...` | WordPress |
| `csrftoken` + `sessionid` | Django (Python) |

This takes 5 seconds and requires no tools — open browser dev tools, check
the cookie name, and you know the backend language immediately.

---

## HTTP Response Headers as Fingerprints

Beyond the `Server:` header you already know from nmap, other response headers
reveal framework and platform:

```bash
# Print all response headers
curl -k -I https://<TARGET_IP>/
curl -k -I --resolve <HOSTNAME>:443:<TARGET_IP> https://<HOSTNAME>/
```

Headers to look for:

| Header | What It Reveals |
|---|---|
| `Server: Apache/2.4.41` | Web server + version |
| `X-Powered-By: PHP/7.4.3` | Runtime + version (often left on by default) |
| `X-Generator: WordPress 5.7` | CMS + version |
| `X-AspNet-Version: 4.0.30319` | ASP.NET version |
| `X-Frame-Options: DENY` | Security header — server is somewhat hardened |
| `Set-Cookie: PHPSESSID=...` | PHP backend confirmed |
| `Via: 1.1 varnish` | Caching layer (Varnish) in front of server |

`X-Powered-By` is particularly valuable — developers frequently forget to
disable it, and it gives you the exact runtime version without any probing.

---

## Exam-Day Fingerprinting Sequence — Complete Stack ID Workflow

```bash
# Step 1 — WhatWeb for automated stack identification
whatweb -a 3 --no-check-certificate http://<TARGET_IP>
whatweb -a 3 --no-check-certificate https://<HOSTNAME>

# Step 2 — HTTP headers for version leakage
curl -k -I https://<HOSTNAME>/ 2>/dev/null | grep -iE \
  "server|x-powered|x-generator|x-aspnet|set-cookie"

# Step 3 — Browser: Wappalyzer extension (while manually browsing the app)
# Check: browser toolbar → Wappalyzer icon → read all detected components

# Step 4 — Browser: Dev tools cookie check (5 seconds)
# F12 → Application → Cookies → read session cookie name

# Step 5 — searchsploit every version found
searchsploit <component> <version>   # repeat for every identified component

# Step 6 — Cross-reference with nmap vulners output
# Components confirmed by WhatWeb + CVEs flagged by vulners = high confidence
```

---

## Gotchas & Exam Tips

- **Wappalyzer web lookup won't work on exam machines.** They are on a
  private VPN network unreachable from the internet. Use the browser extension
  or WhatWeb from your Kali machine.

- **WhatWeb `-a 3` is your default.** Level 1 frequently misses components
  that require a second request to detect. Level 3 gives you a complete picture
  in exam lab environments where stealth is not a concern.

- **Old jQuery ≠ easy win.** jQuery XSS vulnerabilities require interaction
  from another user (admin clicking a link). On OSCP where there is no
  simulated victim, XSS rarely leads to flags. But old jQuery confirms the
  app is unpatched — that same negligence applies to server-side components.
  Use it as a confidence signal, not a direct exploit path.

- **WordPress plugins and themes beat WordPress core.** OffSec loves putting
  vulnerable plugins on WordPress installations. After identifying WordPress,
  always enumerate installed plugins:
```bash
  # Manual plugin enumeration via URL
  curl -k https://<TARGET>/wp-content/plugins/
  # If directory listing is on, you see every plugin name and can searchsploit each
```

- **`X-Powered-By` is frequently disabled in hardened configs but still
  appears in error pages.** If headers look clean, trigger a 404 and read
  the error page headers:
```bash
  curl -k -I https://<TARGET>/nonexistent-path-that-returns-404
```

- **Stack fingerprinting informs all subsequent decisions.** The output of
  WhatWeb is not one answer — it is the input to 5–10 parallel searchsploit
  searches. Run it first on every web target, every time, before any other
  web enumeration.

---

## Quick Reference — Key Commands

```bash
# WhatWeb — primary exam tool
whatweb http://<TARGET_IP>
whatweb -a 3 --no-check-certificate https://<HOSTNAME>
whatweb -a 3 -v https://<HOSTNAME> | tee whatweb-<TARGET_IP>.txt

# Header grab
curl -k -I https://<TARGET_IP>/
curl -k -I --resolve <HOSTNAME>:443:<TARGET_IP> https://<HOSTNAME>/

# searchsploit from WhatWeb findings
searchsploit <component> <version>
searchsploit wordpress plugin
searchsploit drupal 7

# WordPress plugin directory (if directory listing enabled)
curl -k https://<TARGET>/wp-content/plugins/
```

---

## Next Steps After Stack Identification

- CMS version found → `searchsploit <cms> <version>`
- WordPress found → enumerate plugins → `tools/wpscan.md`
- Apache/Nginx version found → `searchsploit <server> <version>`
- PHP version found → check for known type juggling or injection CVEs
- Old jQuery found → note it, prioritise server-side components first
- Stack fully identified → feeds into `tools/gobuster.md` (now you know
  what extensions to add: `.php`, `.asp`, `.aspx` etc.)

  ---

## Addendum — Real-World WhatWeb Behaviour and Header Analysis

---

### Correction — `--no-check-certificate` is Not a WhatWeb Flag

The flag `--no-check-certificate` belongs to `wget`, not WhatWeb. Passing it
to WhatWeb throws an unrecognised option error. Remove it from every WhatWeb
command.

WhatWeb handles HTTPS connections using its own Ruby HTTP implementation and
in most cases connects to HTTPS targets without any additional SSL flag. If
you encounter SSL verification errors with WhatWeb specifically, the correct
workaround is:

```bash
# WhatWeb on HTTPS — no SSL flag needed in most cases
whatweb -a 3 -v https://<HOSTNAME>

# If WhatWeb throws SSL errors, downgrade to HTTP first to confirm the app
# then use curl -k for the HTTPS version
curl -k https://<HOSTNAME>/
```

**Corrected standard WhatWeb commands (remove --no-check-certificate):**

```bash
whatweb http://<TARGET_IP>
whatweb -a 3 -v https://<HOSTNAME>
whatweb -a 3 -v https://<HOSTNAME> | tee whatweb-<TARGET_IP>.txt
```

---

### Lesson 1 — WhatWeb Automatically Follows Redirects

When run against `joinindiannavy.gov.in`, WhatWeb produced two separate
reports without being told to follow redirects:

```
WhatWeb report for https://joinindiannavy.gov.in     → 301
WhatWeb report for https://www.joinindiannavy.gov.in → 200 OK
```

WhatWeb follows HTTP 301 and 302 redirects automatically and reports on every
hop. This is useful because:

- It reveals the **canonical hostname** the server expects — here
  `www.joinindiannavy.gov.in` rather than `joinindiannavy.gov.in`
- The 301 hop itself leaks intel (redirect destination, HSTS headers)
- The final 200 response is where all technology detection happens

**Exam implication:** Always use the canonical hostname (the one that returns
200) for all further enumeration — gobuster, curl, nmap, everything. The
non-www version in this case returns only a redirect with minimal headers.
WhatWeb hands you the correct target hostname automatically.

```bash
# Let WhatWeb find the canonical hostname for you
whatweb -a 3 -v https://<TARGET_DOMAIN>

# Then use the 200-returning hostname for all subsequent commands
gobuster dir -u https://<CANONICAL_HOSTNAME> -w <WORDLIST> -k
```

---

### Lesson 2 — Suppressed Server Header: What To Do

The WhatWeb output and curl headers both showed no `Server:` field in the
200 response. The web server is deliberately stripping its identity from
responses — a common hardening measure. WhatWeb detected no server type.

**This is not the end of fingerprinting.** The server already leaked its
identity earlier in the engagement — the nginx 301 redirect body from the
previous curl session:

```html
<center>nginx</center>
```

**The lesson:** Server headers can be stripped from application responses
but are harder to suppress in automatically generated error and redirect
pages. Your fingerprinting toolkit therefore has a fallback order:

```
1. WhatWeb / response headers (fast, passive)
        ↓ if suppressed
2. curl body of 301/302/403/404 error pages (server generates these directly)
        ↓ if still nothing
3. Browser dev tools → Network tab → response headers on specific requests
        ↓ if still nothing
4. Nmap -sV (TCP fingerprinting, independent of HTTP headers)
        ↓ if still nothing
5. SSL certificate issuer and CN (confirms domain, sometimes org)
```

```bash
# Deliberately trigger error pages to catch header leakage
curl -k https://<HOSTNAME>/nonexistent-page-404
curl -k https://<HOSTNAME>/admin/nonexistent-page-404

# Check the 301 redirect response specifically
curl -k -v https://<HOSTNAME>/ 2>&1 | grep -iE "server|nginx|apache|iis"
```

---

### Lesson 3 — Security Headers as Attack Surface Intelligence

The response headers revealed a well-hardened server. Reading security headers
is not just defensive analysis — it directly tells you which attack vectors
are blocked before you waste time on them.

**Headers present and what they mean for your attack surface:**

```
Content-Security-Policy: default-src 'self'; script-src 'self' 'nonce-...'
  https://www.google-analytics.com ...
```
CSP restricts where scripts can load from and execute. A nonce-based CSP
means reflected XSS that injects `<script>` tags will be blocked by the
browser even if the injection succeeds server-side. **XSS via script
injection is unlikely to be exploitable here.** Move this lower on your
priority list.

```
X-Frame-Options: SAMEORIGIN
frame-ancestors 'none'  (inside CSP)
```
Clickjacking is mitigated. Not an OSCP-relevant attack vector regardless,
but confirms attention to security configuration.

```
X-Content-Type-Options: nosniff
```
Prevents MIME-type sniffing. Relevant if you were attempting to serve a
JavaScript file with a non-JS content type. Low impact on OSCP attack paths.

```
X-XSS-Protection: 1; mode=block
```
Legacy browser XSS filter. Deprecated in modern browsers but present —
indicates the application is old enough to still include this header. Older
codebases have more bugs.

```
Strict-Transport-Security: max-age=31536000; includeSubDomains
```
HTTPS enforced for 1 year across all subdomains. HTTP will always redirect.
All your requests must be HTTPS or you will get the 301 redirect, not the
application.

```
Referrer-Policy: strict-origin
Permission-Policy: camera=(), microphone=(), geolocation=()
```
Modern privacy headers. No attack value, but confirm this is an actively
maintained deployment.

**Hardening signal summary for this target:**

| Attack Vector | Status |
|---|---|
| Script injection XSS | Blocked by nonce-based CSP |
| HTTP traffic interception | Blocked by HSTS |
| Clickjacking | Blocked by frame-ancestors + X-Frame-Options |
| MIME confusion attacks | Blocked by nosniff |
| Backend version fingerprint | Suppressed — no Server header |

This is a well-configured public-facing server. On the OSCP exam, targets
are intentionally less hardened than this. But this analysis gives you
the muscle memory to read headers and immediately know which vectors are
worth pursuing versus which are dead ends before you spend time on them.

---

### Lesson 4 — WhatWeb Extracts Emails Automatically

WhatWeb identified and extracted:
```
Email: ddit.dmpr@navy.gov.in
```

This happened passively from the page HTML — WhatWeb scans for `mailto:`
links in the source. On an OSCP exam machine or a real engagement this has
two uses:

- **Username derivation:** `ddit.dmpr` is likely a valid username format
  for this organisation. If you find a login panel, try `ddit.dmpr` as
  a username with common passwords.
- **Domain confirmation:** `navy.gov.in` confirms the internal domain
  structure. On exam machines with Active Directory, email domains often
  match the AD domain name — critical for Kerberos attacks later.

Always add extracted emails to your target notes and derive the username
format from them (first.last, first_last, flast, firstl, etc.).

---

### Lesson 5 — jQuery Detected Without Version: Manual Recovery

WhatWeb detected jQuery but reported no version:
```
JQuery  (no version shown)
```

This happens when jQuery is loaded in a way WhatWeb's regex cannot match
to a version string — minified inline, loaded from a CDN path without a
version number, or embedded without a standard comment block.

**Version matters for CVE hunting.** jQuery below 1.9 has XSS
vulnerabilities. jQuery below 3.5 has prototype pollution issues. Without
the version you cannot assess risk.

**How to find the jQuery version manually:**

```bash
# Method 1: Check the JS file URL in the page source for version in path
curl -k -s https://<HOSTNAME>/ | grep -i "jquery"
# Look for: jquery-3.3.1.min.js or jquery/3.3.1/jquery.min.js
```

```bash
# Method 2: Fetch the jQuery file itself and read its header comment
# First find the URL from page source, then:
curl -k -s https://<HOSTNAME>/js/jquery.min.js | head -5
# First line of unminified jQuery: /*! jQuery v3.3.1 | ... */
```

Browser method (fastest):
```
Browser → F12 → Console → type: jQuery.fn.jquery → press Enter
```
The console returns the exact version string immediately. No tool needed.

---

### Lesson 6 — Reading Browser Cookies Correctly

The browser dev tools screenshot showed three cookies in the Application tab:

```
_ga        → GA1.3.1605075705.17788...
_ga_1BJCE  → GS2.3.s17806214305o3$g...
_gid       → GA1.3.1070061073.17806...
```

These are **Google Analytics tracking cookies**, not session cookies. Their
`_ga` prefix is Google's standard naming. They tell you the site uses Google
Analytics (which WhatWeb already confirmed) — they have zero attack value.

The actual application session cookie (`indiannavy`) does not appear in the
dev tools cookie list because it was set with the **HttpOnly flag**:

```
Set-Cookie: indiannavy=...; HttpOnly
```

**What HttpOnly means:** A cookie flagged HttpOnly is intentionally invisible
to JavaScript. It cannot be read by `document.cookie` in the browser console
or through any client-side script. This is a defence against XSS-based
cookie theft — even if XSS is achieved, the session token cannot be stolen
via JavaScript.

**Exam implication:** If you achieve XSS on a target and the session cookie
is HttpOnly, you cannot steal it with `document.cookie`. Your XSS payload
must do something else — redirect the victim, perform actions on their behalf
(CSRF-style), or keylog. On OSCP, XSS leading to session theft is rare
anyway — but knowing HttpOnly blocks that path stops you wasting time
building a cookie-stealing payload that will never work.

**How to see HttpOnly cookies:** They are visible in:
- The `Set-Cookie` response header (curl -v or dev tools Network tab)
- Dev tools → Application → Cookies (they show as ticked in the HttpOnly
  column, but the value is still visible here — only JavaScript cannot read
  them, the browser DevTools can)

```bash
# Confirm HttpOnly on session cookies via response headers
curl -k -v https://<HOSTNAME>/ 2>&1 | grep -i "set-cookie"
```

---

### Quick Reference — Header Analysis Commands

```bash
# Full header dump for analysis
curl -k -I --resolve <HOSTNAME>:443:<TARGET_IP> https://<HOSTNAME>/

# Find server identity in error pages
curl -k https://<HOSTNAME>/fakepath404 2>&1 | grep -iE "nginx|apache|iis|server"

# Extract emails from page source
curl -k -s https://<HOSTNAME>/ | grep -oE "[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}"

# Find jQuery version in page source
curl -k -s https://<HOSTNAME>/ | grep -i "jquery"

# Check Set-Cookie flags
curl -k -v https://<HOSTNAME>/ 2>&1 | grep -i "set-cookie"
```
