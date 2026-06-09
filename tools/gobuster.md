# Gobuster — Directory and File Brute Forcing

## TL;DR
Gobuster brute forces web server paths using a wordlist — sending requests
for thousands of directory and file names and reporting which ones exist.
It finds hidden admin panels, login pages, upload directories, backup files,
and config files that are not linked anywhere on the visible site. Run it
against every web target on the exam. Always.

---

## What is Directory Brute Forcing and Why Does It Matter?

When a developer builds a web application, they create files and folders on
the server — admin panels, API endpoints, backup files, configuration files,
database dumps, upload directories. Not all of these are linked from the
visible pages. Some are forgotten. Some are intentionally hidden. But
"hidden" in web terms just means "not linked" — it does not mean protected.
If you know the path, the server will serve it.

Directory brute forcing is the process of **systematically guessing paths**
by sending HTTP GET requests for thousands of common names:

```
GET /admin/        → 403 Forbidden (exists, blocked)
GET /login.php     → 200 OK (exists, accessible)
GET /backup/       → 301 (exists, redirects)
GET /config.php    → 200 OK (exists — may contain credentials)
GET /notexist/     → 404 Not Found (does not exist)
```

Everything that is not 404 is a finding. Some of those findings are
your way into the system.

---

## What is Gobuster?

Gobuster is a purpose-built brute forcing tool written in Go. Go programs
are compiled to native binaries, which means Gobuster is extremely fast —
it can handle many concurrent requests without the overhead of interpreted
languages.

Gobuster operates in several modes. The one you will use on almost every
web target is:

```
dir mode  →  brute forces files and directories on a web server
```

Other modes exist (dns, vhost, fuzz) — we will cover them when relevant.

---

## What is a Wordlist?

A wordlist is a plain text file with one entry per line. In directory brute
forcing, each line is a path that Gobuster tries as a URL:

```
admin
login
uploads
backup
config
.htaccess
index.php
...
```

The quality of your results is directly proportional to the quality of your
wordlist. Kali Linux ships with several pre-built wordlists stored at:

```
/usr/share/wordlists/
```

**Key wordlists for web directory brute forcing:**

| Wordlist | Location | Size | When to Use |
|---|---|---|---|
| `common.txt` | `/usr/share/wordlists/dirb/common.txt` | ~4,600 entries | Default first pass — fast, good coverage |
| `big.txt` | `/usr/share/wordlists/dirb/big.txt` | ~20,000 entries | Second pass if common.txt misses something |
| `directory-list-2.3-medium.txt` | `/usr/share/wordlists/dirbuster/` | ~220,000 entries | Deep sweep — slow but thorough |
| `directory-list-2.3-small.txt` | `/usr/share/wordlists/dirbuster/` | ~87,000 entries | Balance of depth and speed |

**Exam strategy:** Start with `common.txt` — fast, immediate results while
other enumeration runs in parallel. If nothing useful found, escalate to
`directory-list-2.3-medium.txt`. Never start with the medium list on a fresh
target — you will wait 20 minutes while faster wordlists would have found
the same paths in 2 minutes.

---

## HTTP Status Codes — What Every Response Means

Gobuster reports the HTTP status code for every path it finds. Understanding
what each code means determines what action you take.

| Code | Name | Meaning | What You Do |
|---|---|---|---|
| `200` | OK | Path exists and is accessible | **High priority** — visit it immediately |
| `301` | Moved Permanently | Path exists, redirects to another URL | Follow the redirect — the → shows where |
| `302` | Found (Temporary Redirect) | Path exists, redirects temporarily | Follow redirect — often points to login page |
| `401` | Unauthorized | Path exists, requires HTTP Basic Auth | Try default credentials, brute force |
| `403` | Forbidden | Path exists, access denied by server | Note it — try bypass techniques |
| `404` | Not Found | Path does not exist | Gobuster hides these by default |
| `500` | Internal Server Error | Path exists, server crashed processing it | Potentially injectable — investigate |

**The 403 finding deserves special attention:**

A 403 means the server knows the path exists and is telling you "no." This
is more useful than a 404 which means "never heard of it." Common 403 bypass
techniques to attempt:

```bash
# Try adding a trailing slash
curl http://<TARGET_IP>/admin/

# Try different capitalisation
curl http://<TARGET_IP>/Admin/

# Try path traversal prefix
curl http://<TARGET_IP>/./admin/

# Try with HTTP method override headers
curl -H "X-Original-URL: /admin/" http://<TARGET_IP>/

# Try accessing a file inside the forbidden directory directly
curl http://<TARGET_IP>/admin/index.php
curl http://<TARGET_IP>/admin/config.php
```

---

## Core Execution — dir Mode

### Basic Command

```bash
gobuster dir -u http://<TARGET_IP> -w /usr/share/wordlists/dirb/common.txt
```

Flag breakdown:
- `dir` → directory and file enumeration mode
- `-u http://<TARGET_IP>` → target URL — must include `http://` or `https://`
- `-w <WORDLIST>` → path to wordlist file
- Without any other flags: uses 10 threads, 10 second timeout, reports
  all non-404 responses

---

### Standard Exam-Day Command

```bash
gobuster dir \
  -u http://<TARGET_IP> \
  -w /usr/share/wordlists/dirb/common.txt \
  -t 10 \
  -o gobuster-<TARGET_IP>.txt
```

- `-t 10` → 10 threads (default — good balance for exam lab networks)
- `-o gobuster-<TARGET_IP>.txt` → save output to file for later reference

---

### File Extension Brute Forcing — Critical for Exam

The most important flag you add once you know the backend language. Without
`-x`, Gobuster only looks for directories and extensionless paths. With `-x`,
it also tries every wordlist entry with the specified extensions appended.

**Why this matters:** Config files, backup files, and scripts all have
extensions. A wordlist entry of `config` with `-x php` also tries
`config.php`. The file `config.php` containing database credentials will
never be found without `-x php`.

```bash
# PHP target (most common on OSCP)
gobuster dir \
  -u http://<TARGET_IP> \
  -w /usr/share/wordlists/dirb/common.txt \
  -x php,txt,html,bak,zip,conf \
  -t 10 \
  -o gobuster-<TARGET_IP>.txt

# ASP.NET target (Windows/IIS servers)
gobuster dir \
  -u http://<TARGET_IP> \
  -w /usr/share/wordlists/dirb/common.txt \
  -x asp,aspx,txt,html,bak,config \
  -t 10 \
  -o gobuster-<TARGET_IP>.txt

# When you don't yet know the language (run after WhatWeb)
gobuster dir \
  -u http://<TARGET_IP> \
  -w /usr/share/wordlists/dirb/common.txt \
  -x php,asp,aspx,txt,html,bak,zip,conf,db,sql \
  -t 10 \
  -o gobuster-<TARGET_IP>.txt
```

**How WhatWeb informs your `-x` choices:**

```
WhatWeb says PHP   → -x php,txt,bak,zip,conf
WhatWeb says ASP   → -x asp,aspx,txt,bak,config
WhatWeb says Python/Django → -x py,txt,html
WhatWeb says Apache + nothing → try php first (Apache defaults to PHP)
```

---

### HTTPS Targets

```bash
# -k ignores SSL certificate errors (essential for self-signed certs)
gobuster dir \
  -u https://<HOSTNAME> \
  -w /usr/share/wordlists/dirb/common.txt \
  -x php,txt,bak \
  -k \
  -t 10 \
  -o gobuster-<TARGET_IP>.txt
```

For HTTPS virtual host targets where IP scanning gives wrong results:

```bash
# Use hostname as URL — sends correct Host header and SNI
gobuster dir \
  -u https://<HOSTNAME> \
  -w /usr/share/wordlists/dirb/common.txt \
  -k \
  -t 10 \
  -o gobuster-<TARGET_IP>.txt
```

---

### Scanning a Subdirectory

When Gobuster finds a directory (e.g. `/uploads/`), scan inside it too:

```bash
gobuster dir \
  -u http://<TARGET_IP>/uploads/ \
  -w /usr/share/wordlists/dirb/common.txt \
  -x php,txt,bak \
  -t 10 \
  -o gobuster-<TARGET_IP>-uploads.txt
```

Repeat for every interesting directory found in the first pass.

---

### Authenticated Scanning

When a web application requires authentication and you have obtained cookies
or credentials:

```bash
# Pass session cookie
gobuster dir \
  -u http://<TARGET_IP> \
  -w /usr/share/wordlists/dirb/common.txt \
  -c "sessionid=<COOKIE_VALUE>" \
  -t 10

# Pass HTTP Basic Auth credentials
gobuster dir \
  -u http://<TARGET_IP> \
  -w /usr/share/wordlists/dirb/common.txt \
  -U <USER> -P <PASS> \
  -t 10
```

---

### Filtering Noise from Output

Sometimes servers return unexpected status codes for non-existent pages
(e.g. returning 200 for everything, or 302 for everything). This breaks
Gobuster's ability to distinguish real findings from noise.

```bash
# Exclude specific status codes from output
gobuster dir \
  -u http://<TARGET_IP> \
  -w /usr/share/wordlists/dirb/common.txt \
  --exclude-length 0 \
  -t 10

# Only show specific status codes
gobuster dir \
  -u http://<TARGET_IP> \
  -w /usr/share/wordlists/dirb/common.txt \
  -s "200,301,302,401,403" \
  -t 10
```

**When every path returns 200 (wildcard response):** The server is returning
200 for everything including non-existent paths — a false-positive trap.
Gobuster includes a detection for this and will warn you. You may need to
inspect response sizes to distinguish real from fake 200s:

```bash
# Show response size in output — real pages have distinct sizes
# fake 200s often all return the same size (the error page size)
gobuster dir \
  -u http://<TARGET_IP> \
  -w /usr/share/wordlists/dirb/common.txt \
  -t 10
# Look at the [Size: XXX] column — identical sizes = fake positives
```

---

## Reading Gobuster Output — What Each Finding Means

```
/.hta         (Status: 403) [Size: 278]
/.htaccess    (Status: 403) [Size: 278]
/.htpasswd    (Status: 403) [Size: 278]
/css          (Status: 301) [Size: 312] [--> http://192.168.50.20/css/]
/db           (Status: 301) [Size: 311] [--> http://192.168.50.20/db/]
/images       (Status: 301) [Size: 315] [--> http://192.168.50.20/images/]
/index.php    (Status: 302) [Size: 0]  [--> ./login.php]
/js           (Status: 301) [Size: 311] [--> http://192.168.50.20/js/]
/server-status (Status: 403) [Size: 278]
/uploads      (Status: 301) [Size: 316] [--> http://192.168.50.20/uploads/]
```

**Triage this output in order of attack value:**

| Finding | Priority | Action |
|---|---|---|
| `/index.php` → 302 → `login.php` | **Critical** | Test default creds, SQLi, username enum |
| `/db/` → 301 | **Critical** | Browse it — likely contains database dumps |
| `/uploads/` → 301 | **Critical** | Check for file upload + directory listing → RCE |
| `/.htpasswd` → 403 | **High** | File exists — contains hashed credentials. Try 403 bypass. |
| `/.htaccess` → 403 | **High** | File exists — may expose access rules and paths |
| `/server-status` → 403 | **Medium** | Apache status page — try bypass, reveals active connections |
| `/css/`, `/js/` → 301 | **Low** | Check for directory listing, note any unusual files |
| `/images/` → 301 | **Low** | Check for directory listing, file upload residue |

**The `.htpasswd` finding deserves special attention:**

`.htpasswd` is Apache's file for storing HTTP Basic Auth credentials as
hashed values. A 403 means it exists but the server is blocking direct
access. If you bypass the 403 or find the file through another vector
(LFI, backup exposure), it contains usernames and password hashes ready
for offline cracking:

```bash
# If you obtain .htpasswd content:
hashcat -m 1600 htpasswd.txt /usr/share/wordlists/rockyou.txt
# or
john --wordlist=/usr/share/wordlists/rockyou.txt htpasswd.txt
```

---

## Exam-Day Gobuster Workflow

```bash
# 1. First pass — fast, common paths, no extensions
gobuster dir \
  -u http://<TARGET_IP> \
  -w /usr/share/wordlists/dirb/common.txt \
  -t 10 \
  -o gobuster-pass1-<TARGET_IP>.txt

# 2. While reviewing pass 1, launch pass 2 with extensions
#    (add -x based on WhatWeb/whatweb findings)
gobuster dir \
  -u http://<TARGET_IP> \
  -w /usr/share/wordlists/dirb/common.txt \
  -x php,txt,bak,zip,conf,db,sql \
  -t 10 \
  -o gobuster-pass2-<TARGET_IP>.txt

# 3. For any interesting directories found, scan inside them
gobuster dir \
  -u http://<TARGET_IP>/<FOUND_DIR>/ \
  -w /usr/share/wordlists/dirb/common.txt \
  -x php,txt,bak \
  -t 10 \
  -o gobuster-<FOUND_DIR>-<TARGET_IP>.txt

# 4. If pass 1 and 2 give nothing useful, escalate wordlist
gobuster dir \
  -u http://<TARGET_IP> \
  -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt \
  -x php,txt,bak \
  -t 10 \
  -o gobuster-medium-<TARGET_IP>.txt
```

**Parallelism tip:** Run pass 1 in one terminal, immediately start enumerating
other services in other terminals. When pass 1 finishes, review it and launch
pass 2. Never wait idle for Gobuster to complete.

---

## Wordlist Selection — Decision Guide

```
Target identified as WordPress?
→ Use wpscan instead of gobuster for plugin/theme enumeration
→ Still run gobuster for non-WP paths: -w common.txt

Target is PHP/Apache?
→ -x php,txt,bak,zip,conf

Target is IIS/ASP.NET (Windows)?
→ -x asp,aspx,txt,bak,config,zip

Target has no confirmed language?
→ -x php,asp,aspx,txt,html,bak

Time-constrained (exam, one machine left)?
→ common.txt only, no extensions, -t 20

Need maximum coverage?
→ directory-list-2.3-medium.txt + extensions, -t 10
→ Accept that this takes 20-40 min — run on background terminal
```

---

## Gotchas & Exam Tips

- **Always include `http://` or `https://` in the `-u` value.** Gobuster
  will error without the scheme prefix.

- **`-x` multiplies scan time.** Adding `-x php,txt,bak` triples the number
  of requests (every wordlist entry tried three additional times). On a
  220,000-entry wordlist with 5 extensions, you are making over a million
  requests. Balance thoroughness with time budget.

- **Start with `common.txt`, not the medium list.** On most OSCP machines
  the intended path is in the top 5,000 common paths. The medium list is
  for when you are stuck, not for first pass.

- **Rate limit `-t` on unstable targets.** Default 10 threads can overwhelm
  fragile web servers and cause 500 errors or service crashes. On a machine
  that seems unstable, drop to `-t 5` or `-t 3`.

- **403 is not the end.** `.htpasswd` at 403 means the file is there with
  hashed credentials. `/admin/` at 403 means there is something behind that
  door. Document 403 findings and attempt bypass.

- **Follow every 301/302 redirect target.** The `[-->]` arrow in Gobuster
  output shows exactly where the redirect goes. Browse that URL manually —
  the redirect destination is the real content.

- **Re-run with `-k` flag for HTTPS targets.** Without it, Gobuster fails
  on self-signed certs. On exam machines, SSL certs are almost never signed
  by a trusted CA.

- **`/server-status` at 403 on Apache = potential info leak.** This is
  Apache's built-in status page showing active connections, requested URLs,
  and worker status. Bypassing it reveals what other paths are being accessed
  on the server. Try: `curl http://<TARGET_IP>/server-status` — sometimes the
  403 is only enforced from external IPs and bypass methods work.

---

## Quick Reference — Key Commands

```bash
# Standard first pass
gobuster dir -u http://<TARGET_IP> \
  -w /usr/share/wordlists/dirb/common.txt \
  -t 10 -o gobuster-<TARGET_IP>.txt

# With file extensions (PHP target)
gobuster dir -u http://<TARGET_IP> \
  -w /usr/share/wordlists/dirb/common.txt \
  -x php,txt,bak,zip,conf -t 10 \
  -o gobuster-ext-<TARGET_IP>.txt

# HTTPS with cert bypass
gobuster dir -u https://<HOSTNAME> \
  -w /usr/share/wordlists/dirb/common.txt \
  -x php,txt,bak -k -t 10 \
  -o gobuster-https-<TARGET_IP>.txt

# Subdirectory scan
gobuster dir -u http://<TARGET_IP>/<DIR>/ \
  -w /usr/share/wordlists/dirb/common.txt \
  -x php,txt,bak -t 10

# Authenticated (cookie)
gobuster dir -u http://<TARGET_IP> \
  -w /usr/share/wordlists/dirb/common.txt \
  -c "SESSIONID=<VALUE>" -t 10

# Deep scan (when stuck)
gobuster dir -u http://<TARGET_IP> \
  -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt \
  -x php,txt,bak -t 10 \
  -o gobuster-deep-<TARGET_IP>.txt
```

---

## Next Steps After Gobuster

| Finding | Next Action |
|---|---|
| Login page (`login.php`, `admin/`) | Default creds → SQLi → `playbooks/web-login-attacks.md` |
| Upload directory | Check directory listing → file upload exploit → `playbooks/file-upload-exploitation.md` |
| Database directory (`/db/`) | Browse for `.sql` dumps → extract credentials |
| Backup files (`.bak`, `.zip`) | Download → read source for credentials and logic |
| Config files (`config.php`) | Fetch → database credentials likely inside |
| `.htpasswd` (403) | Attempt 403 bypass → crack hashes if obtained |
| WordPress paths found | `tools/wpscan.md` |
| Nothing found | Escalate wordlist → try `-x` extensions → try vhost enumeration |

---

## Addendum — Real-World Gobuster Behaviour and Scan Analysis

---

### Lesson 1 — Rate Limiting: What a Timeout Flood Looks Like and How to Fix It

In run 1, the scan produced clean results up to ~40% completion, then flooded
with consecutive errors:

```
[ERROR] Get "https://.../grants": context deadline exceeded
[ERROR] Get "https://.../Graphics": context deadline exceeded
[ERROR] Get "https://.../group_inlinemod": context deadline exceeded
... (dozens more)
```

This is **server-side rate limiting activating mid-scan**. The server or a
WAF (Web Application Firewall) in front of it detected the volume of requests,
identified the source as a scanner, and started dropping connections. Every
request after the threshold hit gets silently dropped — your client waits
for a response that never comes until the timeout fires.

**What makes this dangerous on exam day:** The scan still completes and
reports findings — but all results after the rate limit triggered are
unreliable. Paths that exist in the rate-limited portion of the wordlist
were never actually tested. You get a false sense of completion.

**Two indicators that rate limiting has triggered:**
- Consecutive `context deadline exceeded` errors on sequential wordlist
  entries (not random, but a burst of them together)
- `dial tcp <IP>:443: i/o timeout` errors (connection refused at TCP level
  — even more aggressive blocking)

**The fix — add delay between requests:**

```bash
# Add 500ms delay between requests — dramatically reduces rate limit triggers
gobuster dir \
  -u https://<HOSTNAME> \
  -w /usr/share/wordlists/dirb/common.txt \
  -k -t 5 \
  --delay 500ms \
  -o gobuster-<TARGET_IP>.txt
```

```bash
# More aggressive reduction for heavily protected targets
gobuster dir \
  -u https://<HOSTNAME> \
  -w /usr/share/wordlists/dirb/common.txt \
  -k -t 3 \
  --delay 1000ms \
  -o gobuster-<TARGET_IP>.txt
```

**Delay + thread reduction together.** Reducing threads alone (`-t 3`)
reduces parallelism but requests still arrive at maximum speed. Adding
`--delay` spaces out individual requests. For rate-limited targets, use
both. For exam lab machines (no WAF), default settings are fine.

**On exam day:** If you see a burst of timeout errors mid-scan, kill the
scan (`Ctrl+C`), relaunch with `--delay 500ms -t 5`, and restart. Do not
trust results from a scan that hit rate limiting without a delay.

---

### Lesson 2 — Extension Scanning Revealed SQL Files: Exactly Why `-x` Is Non-Negotiable

Comparing the two runs directly:

**Run 1 (no `-x` flag):** found `/en`, `/files`, `/logout`, `/robots.txt`,
`/sitemap.xml`, standard structure.

**Run 2 (with `-x` extensions):** found everything from run 1 PLUS:
```
/file.sql     (Status: 200) [Size: 10]
/wwwstat.sql  (Status: 503) [Size: 105]
/sitemap.html (Status: 303)
/403.html     (Status: 200) [Size: 276]
```

`/file.sql` and `/wwwstat.sql` are completely invisible without `-x sql`.
These are **database-related files sitting in the web root**. On an exam
machine this is one of the highest-value findings possible — SQL files
commonly contain:
- Full database schema with table names (maps the application's data model)
- `INSERT` statements with usernames and password hashes
- Database connection strings with credentials
- Backup dumps of entire databases

**`/file.sql` at size 10 bytes:** Very small — likely contains a single line
or is near-empty. Fetch it regardless:

```bash
curl -k https://<HOSTNAME>/file.sql
```

Even a 10-byte SQL file might contain a version marker, a comment with a
path, or a partial credential. On exam machines, nothing is planted without
purpose.

**`/wwwstat.sql` at Status 503:** Service Unavailable. The file likely exists
but the server is overloaded or the application is returning 503 for this
specific resource. Retry later:

```bash
# Retry a specific path manually
curl -k https://<HOSTNAME>/wwwstat.sql
curl -k -v https://<HOSTNAME>/wwwstat.sql 2>&1 | grep -E "HTTP|< "
```

**Lesson:** Always run a second Gobuster pass with `-x` extensions after
the initial pass. Use the language identified by WhatWeb to choose extensions.
If you see PHP, add `-x php,sql,txt,bak,zip`. SQL files in particular never
appear in directory-only scans.

---

### Lesson 3 — `robots.txt` and `sitemap.xml` at 403: How to Recover Value

Both `robots.txt` and `sitemap.xml` returned 403 — they exist but access is
blocked from your IP or externally.

**Why `robots.txt` at 403 is still a finding:**

`robots.txt` is a file webmasters create to tell search engine crawlers
which paths NOT to index. It is not a security control — it is a
politeness convention. The irony is that `robots.txt` commonly lists the
most sensitive paths on the application, because admins want those paths
excluded from Google results:

```
User-agent: *
Disallow: /admin/
Disallow: /backup/
Disallow: /internal-api/
Disallow: /config/
```

Every `Disallow:` entry is a path you should immediately test. On a public
target, you can read `robots.txt` via Google's cache:

```
https://webcache.googleusercontent.com/search?q=cache:https://<HOSTNAME>/robots.txt
```

Or check if a third-party archive has it:

```bash
# Check Wayback Machine for cached robots.txt
curl "https://web.archive.org/web/*/https://<HOSTNAME>/robots.txt" | grep -i "disallow"
```

On exam machines, `robots.txt` is often accessible (no production-level
WAF rules). Always try it directly:

```bash
curl -k https://<HOSTNAME>/robots.txt
```

**`sitemap.xml` at 403:**

The sitemap is an XML file listing every URL the site wants search engines
to know about. If you can access it, it hands you the complete URL map of
the application — every page, every endpoint. Same recovery approach:

```bash
curl -k https://<HOSTNAME>/sitemap.xml

# If 403, try common alternative locations
curl -k https://<HOSTNAME>/sitemap_index.xml
curl -k https://<HOSTNAME>/sitemap1.xml
curl -k https://<HOSTNAME>/en/sitemap.xml
```

---

### Lesson 4 — Language-Based Path Structure: Enumerate Each Language Root

The scan found `/en` (200, 86619 bytes) and `/hi` (200, 86622 bytes) — both
returning full pages. These are **language-specific roots** for the
application. The redirect chain from `/logout` to `/en/logout` confirms the
application structures all content under language prefixes:

```
/en/    → English content root
/hi/    → Hindi content root
/enc/   → Unknown (enc = encrypted content? encoded? needs investigation)
```

**The exam implication:** When an application uses language-path routing,
you need to run Gobuster against each language root separately — the content
under `/en/` may be completely different from what you find at the root `/`:

```bash
# Scan under each language root found
gobuster dir \
  -u https://<HOSTNAME>/en/ \
  -w /usr/share/wordlists/dirb/common.txt \
  -k -t 5 \
  -o gobuster-en-<TARGET_IP>.txt

gobuster dir \
  -u https://<HOSTNAME>/hi/ \
  -w /usr/share/wordlists/dirb/common.txt \
  -k -t 5 \
  -o gobuster-hi-<TARGET_IP>.txt
```

A login panel, admin path, or API endpoint under `/en/admin/` will not
appear in a root-level scan. Always recurse into every 200 and 301
directory you find.

**`/enc/` specifically** — the name is ambiguous and worth manual
investigation:

```bash
# Browse it directly
curl -k https://<HOSTNAME>/enc/
# If directory listing is on, you see contents immediately
```

---

### Lesson 5 — HTTP 303 "See Other": What It Means vs 301/302

`/logout` returned Status 303, which did not appear in the earlier notes.
This is distinct from 301 and 302:

| Code | Name | Meaning | Exam relevance |
|---|---|---|---|
| 301 | Moved Permanently | Resource permanently at new URL | Follow it — real content there |
| 302 | Found | Temporary redirect | Follow it — often points to login |
| 303 | See Other | After a POST, go GET this URL | Confirms form processing exists |
| 307 | Temporary Redirect | Like 302 but preserves POST method | Rare, treat like 302 |

**303 specifically means:** "You submitted something (POST request), now
go GET this other URL for the result." Finding `/logout` as 303 → `/en/logout`
confirms the application has **session management** — there is a POST-based
logout action, which means there is a login action somewhere producing
sessions. This confirms authentication functionality exists and is worth
finding with further enumeration.

---

### Lesson 6 — Custom Error Pages Masquerading as 200: The False Positive Trap

Run 2 found:
```
/403.html   (Status: 200) [Size: 276]
```

This is a custom error page that the server serves when access is forbidden.
The server has a file called `403.html` that it returns with HTTP 200 status
— the web framework is serving the error page content with the wrong status
code, or the file simply exists as static HTML.

**Why this matters for false positive detection:**

If a server returns HTTP 200 for its error pages (a misconfiguration where
custom error documents get served as 200 instead of their proper error code),
your Gobuster results will contain many false positives. Every path that
triggers this error page will show as 200 with the same file size.

**The tell:** All false positives will have **identical file sizes**. Real
pages have varied sizes. If you see 20 results all showing `[Size: 276]`,
they are all returning the same error page.

```bash
# Check for false positive pattern — identical sizes = same error page
grep "Status: 200" gobuster-output.txt | awk '{print $NF}' | sort | uniq -c | sort -rn
# High count of identical sizes = false positives
```

**Filtering by size to remove false positives:**

```bash
# Exclude responses of a specific size (the error page size)
gobuster dir \
  -u https://<HOSTNAME> \
  -w /usr/share/wordlists/dirb/common.txt \
  -k -t 5 \
  --exclude-length 276 \
  -o gobuster-filtered-<TARGET_IP>.txt
```

Replace `276` with the size of the false-positive response you identified.

---

### Lesson 7 — Use the Standard Kali Wordlist Path on Exam Day

Run 1 used `Downloads/common.txt` — a wordlist manually copied to the home
directory. This works but is fragile: the file may not be present on a fresh
Kali instance, the version may differ, and it is not in the expected location.

**On exam day, always use the pre-installed Kali paths:**

```bash
# Standard Kali wordlist locations — always present, no setup required
/usr/share/wordlists/dirb/common.txt          # ~4,600 entries — first pass
/usr/share/wordlists/dirb/big.txt             # ~20,000 entries — second pass
/usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt  # ~220k — deep
/usr/share/wordlists/dirbuster/directory-list-2.3-small.txt   # ~87k — medium
```

If `rockyou.txt` is not already unzipped on your Kali instance:

```bash
sudo gunzip /usr/share/wordlists/rockyou.txt.gz
```

---

### Revised Standard Command — Incorporating All Lessons

```bash
# Exam-day standard: delay added, standard wordlist path, extensions, output saved
gobuster dir \
  -u https://<HOSTNAME> \
  -w /usr/share/wordlists/dirb/common.txt \
  -x php,txt,sql,bak,zip,conf,html \
  -k \
  -t 5 \
  --delay 200ms \
  -o gobuster-<TARGET_IP>.txt

# If rate limiting triggers (burst of timeout errors) — reduce further
gobuster dir \
  -u https://<HOSTNAME> \
  -w /usr/share/wordlists/dirb/common.txt \
  -x php,txt,sql,bak \
  -k \
  -t 3 \
  --delay 500ms \
  -o gobuster-ratelimited-<TARGET_IP>.txt

# After first pass — recurse into every found directory
gobuster dir \
  -u https://<HOSTNAME>/<FOUND_DIR>/ \
  -w /usr/share/wordlists/dirb/common.txt \
  -x php,txt,sql,bak \
  -k -t 5 \
  -o gobuster-<FOUND_DIR>-<TARGET_IP>.txt
```
