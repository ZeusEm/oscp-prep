# HTTP Response Headers and Sitemaps — Web Enumeration

## TL;DR
Every HTTP response from a web server contains headers — metadata that
reveals the technology stack, security posture, infrastructure, and sometimes
credentials. `robots.txt` and `sitemap.xml` are files web servers publish
intentionally for search engines — and they frequently list the most sensitive
paths on the application. Both are passive, zero-noise intelligence sources
you check on every web target before touching any attack tool.

---

## What Are HTTP Response Headers?

When your browser requests a web page, the server sends back two parts:

```
Part 1 — Response Headers (metadata about the response)
Part 2 — Response Body (the actual HTML/JSON/content)
```

Headers are key-value pairs that travel in front of every response. They
are invisible in the rendered browser page but fully visible in DevTools
and curl. They tell you how the server is configured, what technology
generated the response, what security controls are in place, and sometimes
what platform the application runs on.

**A raw HTTP response with headers looks like this:**

```
HTTP/1.1 200 OK
Server: Apache/2.4.41 (Ubuntu)
X-Powered-By: PHP/7.4.3
Content-Type: text/html; charset=UTF-8
Set-Cookie: PHPSESSID=abc123; HttpOnly; Secure
X-Frame-Options: SAMEORIGIN
Content-Length: 45231

<html>...body starts here...
```

Everything above the blank line is headers. Everything below is body.
The headers are your intelligence source. The body is what the user sees.

---

## Two Ways to Inspect Response Headers

### Method 1 — Firefox Network Tool (Visual, Good for Browsing)

The Network tool captures every HTTP transaction the browser makes —
requests and their responses — as you browse.

```
F12 → Network tab → refresh the page (Ctrl+R)
```

The tool only shows traffic that occurs while it is open. Refreshing
the page after opening it populates the list.

**Reading the Network tool:**
- Each row is one HTTP request
- Click a row → right panel shows request and response details
- Click **Headers** sub-tab → see all response headers

**What to look at first:**
- The first request (the page itself) — contains the most revealing headers
- Any XHR/Fetch requests (API calls) — often have different headers revealing
  backend infrastructure
- Redirect responses (301/302) — headers on redirects often expose internal
  hostnames or paths

### Method 2 — curl (Fast, Scriptable, Best for Exam)

curl fetches headers directly from the terminal without opening a browser.
This is your primary exam-day method — faster, greppable, and scriptable.

```bash
# HEAD request — fetch headers only, no response body (fastest)
curl -k -I https://<HOSTNAME>/

# Verbose GET — full headers + body, useful when HEAD suppressed
curl -k -v https://<HOSTNAME>/ 2>&1 | grep "^<"

# Save full response headers to file
curl -k -D headers-<TARGET>.txt https://<HOSTNAME>/ -o /dev/null

# Follow redirects and show headers at each hop
curl -k -IL https://<HOSTNAME>/
```

Flag breakdown:
- `-k` → ignore SSL certificate errors (mandatory for exam lab machines)
- `-I` → HEAD request — server sends headers only, no body
- `-v` → verbose — shows both request and response headers
- `-D <file>` → dump response headers to a file
- `-L` → follow redirects
- `-o /dev/null` → discard the body (when combined with -D)

---

## Header Intelligence — What Every Header Reveals

This is the complete reference for reading response headers as attack
surface intelligence. Check every header against this table.

### Technology Fingerprinting Headers

| Header | Example Value | What It Reveals |
|--------|--------------|-----------------|
| `Server` | `Apache/2.4.41 (Ubuntu)` | Web server + version + OS → searchsploit immediately |
| `X-Powered-By` | `PHP/7.4.3` | Runtime + exact version → searchsploit |
| `X-AspNet-Version` | `4.0.30319` | ASP.NET version → searchsploit |
| `X-Generator` | `WordPress 5.7` | CMS + version → searchsploit |
| `X-Drupal-Cache` | any value | Drupal CMS confirmed |

```bash
# Immediately searchsploit everything version-specific
searchsploit apache 2.4.41
searchsploit php 7.4.3
searchsploit wordpress 5.7
```

### Infrastructure and Hosting Headers

| Header | Example Value | What It Reveals |
|--------|--------------|-----------------|
| `x-amz-cf-id` | any value | Amazon CloudFront CDN in front of server |
| `x-amz-request-id` | any value | Direct AWS S3 or service — no CDN |
| `CF-RAY` | any value | Cloudflare CDN — real IP hidden |
| `Via` | `1.1 varnish` | Varnish cache layer in front of server |
| `X-Cache` | `HIT from proxy` | Caching proxy present |
| `X-Forwarded-For` | `10.0.0.5` | Internal IP address of original client or proxy |

**`X-Forwarded-For` deserves special attention:**

This header is added by proxies and load balancers to tell the web server
the real client IP address. When present in a response, it means a proxy
or load balancer sits in front of the actual server. More importantly, some
applications trust this header without validation — meaning you can spoof it:

```bash
# Some applications use X-Forwarded-For to determine if you are internal
# Spoofing it to an internal IP can sometimes bypass IP-based access controls
curl -k -H "X-Forwarded-For: 127.0.0.1" https://<HOSTNAME>/admin/
curl -k -H "X-Forwarded-For: 10.0.0.1" https://<HOSTNAME>/admin/
```

If an admin panel returns 403 from your IP but the application trusts
X-Forwarded-For, spoofing an internal address may grant access.

### Security Headers — Attack Surface Mapping

These headers tell you which attack vectors are pre-blocked before you
waste time on them:

| Header | Value | Implication |
|--------|-------|-------------|
| `Content-Security-Policy` | `default-src 'self'` | Strict CSP — XSS script injection blocked |
| `Content-Security-Policy` | absent | No CSP — XSS is viable |
| `X-Frame-Options` | `DENY` or `SAMEORIGIN` | Clickjacking blocked |
| `X-XSS-Protection` | `1; mode=block` | Legacy XSS filter — present on older apps |
| `Strict-Transport-Security` | any value | HTTPS enforced — all requests must be HTTPS |
| `X-Content-Type-Options` | `nosniff` | MIME sniffing blocked |
| `Access-Control-Allow-Origin` | `*` | CORS open to all origins — potential for CSRF-like attacks |
| `Access-Control-Allow-Origin` | absent | No CORS headers — API may reject cross-origin requests |

```
CSP present and strict  → deprioritise XSS, focus on injection and auth
CSP absent              → XSS is worth testing
HSTS present            → all URLs must use https://
No security headers     → application is likely old/poorly maintained → more vulns
```

### Session and Authentication Headers

| Header | Value | Implication |
|--------|-------|-------------|
| `Set-Cookie: PHPSESSID=...` | any | PHP backend confirmed |
| `Set-Cookie: JSESSIONID=...` | any | Java backend confirmed |
| `Set-Cookie: ...;HttpOnly` | flag present | Cookie theft via XSS not possible |
| `Set-Cookie: ...;Secure` | flag present | Cookie only sent over HTTPS |
| `Set-Cookie: ...;SameSite=Lax` | flag present | CSRF partly mitigated |
| `Set-Cookie: ...` | no flags | Cookie stealable via XSS, sendable cross-site |
| `WWW-Authenticate: Basic` | any | HTTP Basic Auth — base64 encoded creds |
| `WWW-Authenticate: NTLM` | any | Windows NTLM auth — domain environment confirmed |

`WWW-Authenticate: NTLM` on a 401 response is significant on the exam —
it confirms a Windows Active Directory environment and reveals the domain.

### Redirect and Routing Headers

| Header | Value | Implication |
|--------|-------|-------------|
| `Location` | `/en/login` | Redirect destination — reveals path structure |
| `Location` | `https://internal.corp/` | Internal hostname leaked in redirect |

Internal hostnames in `Location` headers are valuable — add them to
`/etc/hosts` and enumerate them as additional targets.

---

## Exam-Day Header Grep — One Command to Extract All Intel

```bash
# Pull all headers and grep for the high-value ones in one pass
curl -k -IL https://<HOSTNAME>/ 2>/dev/null | grep -iE \
  "server:|x-powered|x-aspnet|x-generator|set-cookie|location|\
x-forwarded|content-security|www-authenticate|x-frame|access-control"
```

This prints only the headers that carry actionable intelligence and
ignores noise headers (date, content-length, cache-control etc.).

---

## robots.txt — The Accidental Attack Surface Map

### What robots.txt Is

`robots.txt` is a plain text file placed at the root of a web server
(`https://target.com/robots.txt`). It is part of the **Robots Exclusion
Protocol** — a convention (not enforced technically) that tells automated
web crawlers (Googlebot, Bingbot, etc.) which pages to index and which
to skip.

**The critical insight for attackers:**

Administrators add pages to `robots.txt` specifically because they do NOT
want those pages appearing in Google search results. The reason they want
them hidden from search engines is almost always because those pages are
sensitive:

```
Disallow: /admin/           → admin panel
Disallow: /backup/          → backup files
Disallow: /internal/        → internal tools
Disallow: /api/private/     → private API
Disallow: /config/          → configuration
Disallow: /wp-admin/        → WordPress admin
Disallow: /phpmyadmin/      → database admin panel
```

Every `Disallow:` entry is a path you should **immediately go test**.
The administrator inadvertently published a list of their most sensitive
paths to the entire internet.

### Reading robots.txt

```
User-agent: *           → applies to all crawlers
Disallow: /search       → do NOT crawl /search
Allow: /search/about    → exception — /search/about IS allowed
Disallow: /admin/       → do NOT crawl /admin/
```

**From an attack perspective:**
- `Disallow:` = "interesting paths I want to hide" → test these
- `Allow:` = explicitly permitted paths → lower priority
- `User-agent: Googlebot` with different rules = path only hidden from
  specific crawlers → still accessible to you

### Fetching robots.txt

```bash
# Standard fetch
curl -k https://<HOSTNAME>/robots.txt

# Extract only Disallow lines — your target list
curl -k -s https://<HOSTNAME>/robots.txt | grep -i "disallow"

# Also try HTTP if HTTPS fails
curl http://<TARGET_IP>/robots.txt

# Save for reference
curl -k -s https://<HOSTNAME>/robots.txt -o robots-<TARGET>.txt
```

### When robots.txt Returns 403

The file exists but is blocked. Try recovery methods:

```bash
# Google's cached version (for public targets)
# Search in Google: site:target.com filetype:txt robots

# Wayback Machine archive
curl "https://web.archive.org/web/*/https://<HOSTNAME>/robots.txt"

# Check if accessible via HTTP even when HTTPS is blocked
curl http://<TARGET_IP>/robots.txt
```

---

## sitemap.xml — The Complete URL Map

### What a Sitemap Is

A sitemap is an XML file that lists every URL the website wants search
engines to know about. Unlike `robots.txt` which is a text file with
path patterns, sitemaps list **exact URLs** — often with metadata about
when they were last updated.

While `robots.txt` tells you what to hide, the sitemap tells you what
exists — the complete map of the application's publicly intended content.
On a complex application, the sitemap can reveal:

- Every page in the application (including rarely-linked ones)
- API endpoint URLs
- File download paths
- Administrative sections the developer intended to be public but forgot
  to protect
- Language or locale-specific paths (`/en/`, `/fr/`, `/admin/`)

### Fetching and Parsing Sitemaps

```bash
# Standard fetch
curl -k https://<HOSTNAME>/sitemap.xml

# Extract just the URLs from the XML
curl -k -s https://<HOSTNAME>/sitemap.xml | grep -oE "<loc>.*?</loc>" \
  | sed 's/<\/*loc>//g'

# Some sites use sitemap indexes (multiple sitemaps)
curl -k https://<HOSTNAME>/sitemap_index.xml
curl -k https://<HOSTNAME>/sitemap1.xml
curl -k https://<HOSTNAME>/sitemap-0.xml

# Save and parse
curl -k -s https://<HOSTNAME>/sitemap.xml -o sitemap-<TARGET>.xml
grep -oE "<loc>.*?</loc>" sitemap-<TARGET>.xml | sed 's/<\/*loc>//g'
```

### Common Sitemap Locations to Try

```bash
# Try all common sitemap paths — servers vary
for path in sitemap.xml sitemap_index.xml sitemap1.xml \
            sitemap-0.xml sitemap.html sitemapindex.xml; do
  echo -n "$path: "
  curl -k -s -o /dev/null -w "%{http_code}" https://<HOSTNAME>/$path
  echo
done
```

---

## Combining Both Sources — Complete Passive URL Enumeration

```bash
# Step 1 — Headers: technology and security posture
curl -k -IL https://<HOSTNAME>/ 2>/dev/null | grep -iE \
  "server:|x-powered|x-aspnet|set-cookie|location|x-forwarded|\
content-security|www-authenticate"

# Step 2 — robots.txt: hidden sensitive paths
curl -k -s https://<HOSTNAME>/robots.txt | grep -i "disallow\|allow"

# Step 3 — sitemap: complete URL inventory
curl -k -s https://<HOSTNAME>/sitemap.xml \
  | grep -oE "<loc>.*?</loc>" | sed 's/<\/*loc>//g'

# Step 4 — Feed all Disallow paths and sitemap URLs into Gobuster or manual testing
```

---

## Exam-Day Workflow

```
Web target identified
        ↓
1. Pull headers → grep for technology, security posture, auth method
   curl -k -IL https://<HOSTNAME>/ 2>/dev/null | grep -iE "server:|x-powered|set-cookie|..."
        ↓
2. searchsploit every version found in headers
   searchsploit <server> <version>
        ↓
3. Fetch robots.txt → extract Disallow paths
   curl -k -s https://<HOSTNAME>/robots.txt | grep -i disallow
        ↓
4. Test every Disallow path manually
   curl -k https://<HOSTNAME>/<disallow-path>
        ↓
5. Fetch sitemap → extract all URLs
   curl -k -s https://<HOSTNAME>/sitemap.xml | grep -oE "<loc>.*?</loc>"
        ↓
6. Add any new paths to Gobuster run and browse manually
        ↓
7. Feed all findings into Burp Repeater for parameter testing
```

---

## Gotchas & Exam Tips

- **`robots.txt` Disallow = your test list.** Every admin who added a path
  to Disallow did so because it is sensitive. They have handed you a
  prioritised target list. Never skip this file.

- **Sitemaps sometimes list API endpoints.** A sitemap intended for a web
  application can accidentally include API routes that were never meant to
  be public. Parse the full list and look for anything with `/api/`, `/v1/`,
  `/internal/` in the path.

- **`X-Forwarded-For` spoofing bypasses IP-based access controls.** Some
  applications restrict admin panels to internal IPs only and check the
  `X-Forwarded-For` header to determine this. If you see `403` on an admin
  path and the server is behind a proxy, try spoofing the header.

- **`WWW-Authenticate: NTLM` = Windows AD environment.** This one header
  on a 401 response tells you more about the target than most enumeration
  — you are facing a Windows Active Directory domain. The NTLM challenge
  also leaks the NetBIOS and DNS domain names in the response.

- **Multiple `Set-Cookie` headers = multiple application layers.** Seeing
  `PHPSESSID` alongside `ASP.NET_SessionId` on the same response means a
  reverse proxy or load balancer is adding its own cookie on top of the
  application's. Map each cookie to its source.

- **The Network tool only captures after it is opened.** If you open it
  after the page loaded, you see nothing. Always hit `Ctrl+R` to refresh
  after opening the Network tab.

---

## Quick Reference — Key Commands

```bash
# Headers only (fast)
curl -k -I https://<HOSTNAME>/

# Headers following redirects (shows all hops)
curl -k -IL https://<HOSTNAME>/

# Headers + grep for intel
curl -k -IL https://<HOSTNAME>/ 2>/dev/null | grep -iE \
  "server:|x-powered|set-cookie|location|x-forwarded|content-security|\
www-authenticate|x-frame|x-aspnet|x-generator"

# X-Forwarded-For bypass attempt
curl -k -H "X-Forwarded-For: 127.0.0.1" https://<HOSTNAME>/admin/

# robots.txt — extract disallow paths
curl -k -s https://<HOSTNAME>/robots.txt | grep -i disallow

# sitemap — extract all URLs
curl -k -s https://<HOSTNAME>/sitemap.xml \
  | grep -oE "<loc>.*?</loc>" | sed 's/<\/*loc>//g'

# Test all common sitemap locations
for p in sitemap.xml sitemap_index.xml sitemap1.xml sitemap.html; do
  echo -n "$p: "; curl -k -s -o /dev/null -w "%{http_code}\n" https://<HOSTNAME>/$p
done
```

---

## Next Steps After Header and Sitemap Enumeration

| Finding | Next Action |
|---|---|
| Server version in header | `searchsploit <server> <version>` |
| PHP/ASP.NET version | `searchsploit <runtime> <version>` |
| Disallow paths in robots.txt | Browse each path → Gobuster on each |
| Sitemap URLs | Add to manual browse list, check for parameters |
| `WWW-Authenticate: NTLM` | Note domain name → feeds into AD enumeration |
| `X-Forwarded-For` trusted | Attempt header spoofing for access control bypass |
| No security headers | Application likely poorly maintained → aggressive testing |
| Internal hostname in Location | Add to `/etc/hosts` → enumerate as new target |

Here’s the final appendix, directly informed by your latest hands-on session. It’s formatted to match your existing notes and covers every gotcha you encountered.

---

## Appendix: Real‑World Enumeration Gotchas & Exam‑Ready Workarounds

This appendix is the direct result of enumerating `joinindiannavy.gov.in`.  
Every lesson below came from a failed command, an unexpected output, or a subtle misconfiguration that turned into actionable intel.  
Add this to the end of your HTTP Response Headers and Sitemaps notes.

---

### A2. When `robots.txt` Returns 403 (or No Output)
Your `robots.txt` hit a 403 – the file exists but is forbidden.  
An empty grep result does **not** mean there’s nothing useful.

**What to do when you get 403:**
1. Try over HTTP (may bypass WAF or weird rules):
   ```bash
   curl -k -s -o /dev/null -w "%{http_code}" http://TARGET/robots.txt
   ```
   If it returns `000`, the server likely doesn’t listen on plain HTTP at all. Move on.
2. Check the Wayback Machine:
   ```bash
   curl -k "https://web.archive.org/web/*/https://TARGET/robots.txt"
   ```
3. Search for `robots.txt` leaks in Google dorking (only for allowed targets):
   ```
   site:target.com filetype:txt robots
   ```

**Takeaway:**  
A `403` on `robots.txt` is a signal, not a dead end. The file exists – find another way to read it.

---

### A3. Error Pages Leak CMS and Version – Even on 404s
Your `X-Forwarded-For: 127.0.0.1` attempt on `/admin/` returned a 404, but the HTML source contained:
```html
<!--
    This website is powered by QuickAppsCMS 2.0.0-beta2, Licensed under GNU/LGPL
//-->
```
This was identical to the leak from `sitemap_index.xml`.  
**Why this matters:**  
- The CMS and exact version are now confirmed.
- Even if `searchsploit QuickAppsCMS` returns *no results* (as happened here), you still have a precise entry for further manual research (default creds, known vulnerable plugins, configuration files, etc.).
- In the exam, **always inspect the body of every error response** – 403, 404, 500. Look for comments, `<meta generator>`, or server‑specific HTML snippets.

**Command to capture this every time:**
```bash
curl -k -v https://TARGET/nonexistent 2>&1 | /bin/grep -iE '<!--|generator|powered|cms|version'
```

---

### A4. When `searchsploit` Returns Nothing
```
searchsploit QuickAppsCMS
Exploits: No Results
Shellcodes: No Results
Papers: No Results
```
This is common for niche or custom CMSs. Your next steps:
- Search for the exact version + "vulnerability" / "exploit" / "bypass" in a browser.
- Look for default credentials: `admin:admin`, `admin:password`, `QuickAppsCMS:QuickAppsCMS`
- Check for publicly accessible configuration files: `/config/`, `/settings.php`, `/core/config/`
- Try common QuickAppsCMS entry points: `/admin`, `/user/login`, `/dashboard`

**Note for exam:**  
Even without a ready exploit, a known CMS + version is a solid finding that demonstrates information disclosure and may be chained later.

---

### A5. Sitemap File Variants – Follow the 303
Your sitemap enumeration returned:
```
sitemap.xml: 403
sitemap.html: 303
```
A `303` (or `301`/`302`) means “go look over there”. Always follow it:
```bash
curl -k -L -v https://TARGET/sitemap.html 2>&1 | /bin/grep -iE '<|<!--|sitemap'
```
In many cases the redirected location contains the full sitemap index. If the redirect loops or lands on a login page, you’ve still identified a valid endpoint worth investigating.

---

### A8. Absolute‑Path Command Templates for Broken Environments
When only absolute paths work, use these compact, tested one‑liners:

```bash
# Header intelligence (all high‑value headers)
/bin/curl -k -IL https://TARGET/ 2>/dev/null | /bin/grep -iE \
  "server:|x-powered|x-aspnet|set-cookie|location|x-forwarded|content-security|www-authenticate|x-frame|x-generator"

# Extract disallowed paths from robots.txt
/bin/curl -k -s https://TARGET/robots.txt | /bin/grep -i disallow

# Extract all URLs from a sitemap
/bin/curl -k -s https://TARGET/sitemap.xml | /bin/grep -oE "<loc>.*?</loc>" | /bin/sed 's/<\/*loc>//g'

# Check all common sitemap filenames
for p in sitemap.xml sitemap_index.xml sitemap1.xml sitemap-0.xml sitemap.html sitemapindex.xml; do
  echo -n "$p: "
  /bin/curl -k -s -o /dev/null -w "%{http_code}" https://TARGET/$p
  echo
done

# Spoof X-Forwarded-For for access control bypass
/bin/curl -k -H "X-Forwarded-For: 127.0.0.1" https://TARGET/admin/
```

---
