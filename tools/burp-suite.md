# Burp Suite — Web Application Proxy and Testing Platform

## TL;DR
Burp Suite is a man-in-the-middle proxy that sits between your browser and
the target web server, letting you intercept, read, modify, and replay every
HTTP request and response. It is **fully allowed on the OSCP exam**. Your
three core tools inside it are Proxy (intercept traffic), Repeater (manually
modify and resend individual requests), and Intruder (automate payloads
across a parameter). Master these three and you can test any web application
manually without relying on automated scanners.

---

## What is a Web Proxy and Why Does it Matter?

When your browser loads a web page, it constructs an HTTP request and sends
it to the server. The server processes it and sends back a response. Normally
you only see the rendered result — the page in your browser. You do not see
the raw request your browser sent, the exact headers, the parameters, or the
raw response.

A **web proxy** is software that inserts itself into this communication path:

```
Normal flow:
Browser → Web Server

With Burp proxy:
Browser → Burp Suite → Web Server
               ↓
         (intercepts, shows you, lets you modify)
               ↓
         Web Server → Burp Suite → Browser
```

Every request your browser makes passes through Burp first. Burp shows you
the raw request, lets you modify any part of it — headers, parameters, body,
cookies — and then forwards it. The server receives your modified request and
responds. Burp captures that response too.

**Why this matters for web application testing:**

Browsers are designed to send well-formed requests. Vulnerability testing
requires sending malformed, unexpected, or crafted requests that browsers
would never generate. Burp lets you:
- Change a parameter value from `id=1` to `id=1 OR 1=1` (SQL injection test)
- Add a header the browser doesn't normally send
- Submit a form with data that bypasses client-side length limits
- Replay a request hundreds of times with different payloads
- Read the raw server response including hidden fields and comments

Every web vulnerability you will test on the OSCP exam requires you to
manipulate HTTP requests. Burp is the tool that makes that possible.

---

## What is HTTP? — The Protocol Burp Intercepts

HTTP (HyperText Transfer Protocol) is the language browsers and web servers
use to communicate. Every interaction is a **request** from the client and a
**response** from the server.

**A raw HTTP request looks like this:**
```
POST /wp-login.php HTTP/1.1
Host: 192.168.50.16
User-Agent: Mozilla/5.0 ...
Content-Type: application/x-www-form-urlencoded
Content-Length: 78
Cookie: wordpress_test_cookie=WP%20Cookie%20check

log=admin&pwd=password123&wp-submit=Log+In&redirect_to=%2Fwp-admin%2F
```

Breaking this down:
- Line 1: HTTP method (`POST`), path (`/wp-login.php`), HTTP version
- `Host:` — which server/virtual host you're talking to
- `Content-Type:` — how the body data is encoded
- `Cookie:` — session tokens and tracking data
- Blank line — separates headers from body
- Body — the actual data submitted (form fields and values)

**A raw HTTP response looks like this:**
```
HTTP/1.1 302 Found
Location: http://192.168.50.16/wp-admin/
Set-Cookie: wordpress_logged_in_...=admin...; path=/
Content-Type: text/html

<html>...
```

- Line 1: HTTP version, **status code** (302), status text
- Headers: cookies being set, redirect destination, content type
- Body: the actual HTML content

**Every field in both request and response is a potential attack surface.**
Burp lets you see and modify all of them.

---

## First-Time Setup

### Step 1 — Launch Burp Suite

```bash
burpsuite &
```

The `&` runs it in the background so your terminal stays usable.

On launch:
- Ignore the JRE warning — Kali's Java is tested and compatible
- Select **Temporary project** → click **Next**
- Select **Use Burp defaults** → click **Start Burp**

---

### Step 2 — Configure Firefox to Use Burp as Proxy

By default, Burp listens for proxy connections on `127.0.0.1:8080`.
You need to tell Firefox to send all its traffic through this address.

**In Firefox:**
1. Navigate to `about:preferences#general`
2. Scroll down to **Network Settings** → click **Settings**
3. Select **Manual proxy configuration**
4. Set HTTP Proxy: `127.0.0.1` Port: `8080`
5. Tick **Also use this proxy for HTTPS**
6. Click **OK**

**Verify it is working:** Browse to any HTTP site. Switch to Burp and check
`Proxy → HTTP History` — you should see requests appearing.

**Quick toggle tip:** Install the **FoxyProxy** Firefox extension. It lets
you toggle the proxy on/off with one click instead of navigating menus. On
the exam, you will switch proxy on/off frequently. FoxyProxy saves time.

---

### Step 3 — Import Burp's CA Certificate (for HTTPS Interception)

By default, Firefox refuses to trust Burp's certificate when it intercepts
HTTPS traffic. You will see SSL errors when trying to proxy HTTPS sites.
Fix this once:

1. With Firefox proxying through Burp, navigate to `http://burp`
2. Click **CA Certificate** in the top right to download `cacert.der`
3. In Firefox: `about:preferences` → search "certificates" →
   **View Certificates** → **Authorities** tab → **Import**
4. Select the downloaded `cacert.der`
5. Tick **Trust this CA to identify websites** → OK

HTTPS traffic will now flow through Burp without SSL warnings.

---

### Step 4 — Add Target Hostnames to /etc/hosts

On the OSCP exam, machines have hostnames that need to resolve to their IPs.
The exam provides these — add them before you start:

```bash
# Add a target hostname
echo "<TARGET_IP> <HOSTNAME>" | sudo tee -a /etc/hosts

# Example — always do this for every exam machine with a web app
echo "192.168.50.16 offsecwp" | sudo tee -a /etc/hosts
```

Burp then intercepts traffic to the hostname correctly because DNS resolves
locally through `/etc/hosts`.

---

### Step 5 — Suppress Firefox Noise in Proxy History

Firefox makes background requests (captive portal checks, telemetry) that
clutter your HTTP History. Disable them:

```
address bar → about:config → accept risk
search: network.captive-portal-service.enabled
double-click → set to false
```

---

## Core Tool 1 — Proxy

### What Intercept Mode Does

When **Intercept is ON**, every request your browser makes is held in Burp
and frozen until you manually forward or drop it. The browser hangs waiting.
Use this when you want to modify a specific request before it hits the server.

When **Intercept is OFF**, requests flow through Burp automatically. The
browser works normally. Burp silently logs everything to HTTP History.

**Default working mode:** Keep Intercept **OFF** while browsing. Turn it
**ON** only when you want to capture and modify a specific request.

```
Proxy tab → Intercept sub-tab → button shows "Intercept is off" = OFF (good)
                                              "Intercept is on" = ON (blocking)
```

**If your browser freezes while browsing through Burp:** Intercept is ON.
Switch it off. This is the most common Burp confusion for new users.

### HTTP History — Your Primary Intelligence Source

`Proxy → HTTP History` shows every request your browser made and every
response the server returned. This is your raw intelligence feed.

**What to look for in HTTP History:**

| Observation | What It Means |
|---|---|
| POST requests to login pages | Capture these → send to Intruder for brute forcing |
| Requests with `?id=`, `?page=`, `?file=` | Parameter-based input → test for SQLi, LFI |
| Requests with cookies in headers | Session tokens — analyse for predictability |
| Large POST request bodies | Form submissions — inspect all field names and values |
| Requests to `/api/` endpoints | API surface — test each endpoint |
| Response headers with `Set-Cookie:` | Session management — look at token format |
| Responses containing hidden HTML fields | Hidden parameters — often bypassed by client-side only |

**How to use HTTP History:**
1. Browse the target application normally (Intercept OFF)
2. Click through every page, form, and function
3. Review History — every request is now visible and actionable
4. Right-click any request → **Send to Repeater** or **Send to Intruder**

---

## Core Tool 2 — Repeater

### What Repeater Is

Repeater lets you take a single HTTP request, modify it manually, send it,
read the raw response, modify it again, send again — as many times as you
want. It is your primary tool for **manual vulnerability testing**.

**Why not just use the browser?** The browser constructs requests according
to HTML rules. It will not let you submit a negative quantity, inject SQL
characters into a hidden field, or send a request with a tampered cookie. Burp
Repeater lets you do all of these because it sends raw HTTP directly.

### Repeater Workflow

**Step 1 — Capture the request in History**
Browse to the target page/form. Submit it normally. Find the request in
`Proxy → HTTP History`.

**Step 2 — Send to Repeater**
Right-click the request → **Send to Repeater**

**Step 3 — Modify and test**
Click the **Repeater** tab. The request appears on the left. Modify any part
of it — change a parameter value, add a header, alter a cookie.

**Step 4 — Send and read response**
Click **Send**. The raw server response appears on the right.

**Step 5 — Iterate**
Modify the request again. Send again. Compare responses. A change in response
(different size, different status code, different content) indicates the
server behaved differently — potentially triggered by your modification.

### What to Modify in Repeater (Common Tests)

```
# Test for SQL injection — modify a parameter value
id=1          →    id=1'
id=1          →    id=1 OR 1=1--
id=1          →    id=1; DROP TABLE users--

# Test for path traversal — modify a file parameter
file=image.png  →  file=../../../../etc/passwd

# Test for command injection — modify any parameter
name=test       →  name=test; whoami
name=test       →  name=test | id

# Test authentication bypass — modify cookie values
session=abc123  →  session=admin
role=user       →  role=administrator

# Test for IDOR (accessing other users' data) — change ID values
id=1001   →   id=1002
id=1001   →   id=1
```

For each modification: send → read response → compare to original response.
Different response = something changed on the server = investigate further.

---

## Core Tool 3 — Intruder

### What Intruder Is

Intruder automates sending a request repeatedly with different values
substituted into a **position** you define. Think of it as Repeater with a
wordlist — instead of manually changing `pwd=test` to each password one by
one, Intruder does it automatically for every entry in a list.

**Primary use cases on OSCP:**
- Password brute forcing against login forms
- Username enumeration
- Fuzzing parameter values for injection vulnerabilities
- Testing every ID value in a range (IDOR)

### Intruder Attack Types

| Attack Type | How It Works | When to Use |
|---|---|---|
| **Sniper** | One position, one wordlist — tries each word in the position | Password brute force, single parameter fuzzing |
| **Battering Ram** | One wordlist, same word inserted into ALL positions simultaneously | When username = password scenarios |
| **Pitchfork** | Multiple positions, multiple wordlists — one entry per list used together | Username:password pair lists |
| **Cluster Bomb** | Multiple positions, every combination tried | Credential stuffing with small lists |

**For OSCP exam login attacks:** Sniper is what you use when you know the
username and want to brute force the password. Pitchfork is what you use
when you have a list of username:password pairs.

### Intruder Workflow — Password Brute Force

**Step 1 — Capture a failed login request**

With Intercept OFF, navigate to the login page. Submit a fake login
(`admin` / `test`). Find the POST request in `Proxy → HTTP History`.
Right-click → **Send to Intruder**.

**Step 2 — Set the attack position**

`Intruder tab → Positions sub-tab`

Burp auto-highlights all detected parameters with `§` markers. Clear all
of them first:

→ Click **Clear §** on the right sidebar

Now select only the password field value. Highlight the value after `pwd=`
(or `password=`) and click **Add §**.

The request should look like:
```
log=admin&pwd=§test§&wp-submit=Log+In
```

Only the password value is wrapped in `§` markers — Intruder will only
substitute values there.

**Step 3 — Set the payload wordlist**

`Intruder tab → Payloads sub-tab`

Under **Payload Options [Simple list]**:
- Click **Load** → select your wordlist file
- OR click **Paste** → paste values directly

**Wordlists for login brute forcing:**
```bash
/usr/share/wordlists/rockyou.txt          # 14 million passwords — large
/usr/share/wordlists/metasploit/unix_passwords.txt  # smaller, faster
```

For quick tests with known password likely in top entries:
```bash
head -100 /usr/share/wordlists/rockyou.txt > /tmp/top100.txt
```

**Step 4 — Launch the attack**

Click **Start Attack** (top right). Accept the Community Edition warning.
A results window opens showing each request and its response details.

**Step 5 — Read the results**

Look for the request where something is **different** from all others:

| Difference | Meaning |
|---|---|
| Different **Status code** (e.g. 302 instead of 200) | Redirect after login = success |
| Different **Length/Size** | Server responded differently = potentially correct value |
| Different **Response content** (grep for "Welcome", "Dashboard") | Login succeeded |

Click on the anomalous request → check the Response tab → confirm login
success or failure from the response body.

### Community Edition Limitation — Intruder Throttling

**This is critical to know for the exam:** Burp Suite Community Edition
deliberately throttles Intruder. Requests are rate-limited to approximately
one per second. Against a 10,000-word wordlist, that is nearly 3 hours.

**For serious password brute forcing on the exam, use these instead:**

```bash
# Hydra — purpose-built login brute forcer, no throttle
hydra -l <USER> -P /usr/share/wordlists/rockyou.txt \
  <TARGET_IP> http-post-form \
  "/wp-login.php:log=^USER^&pwd=^PASS^&wp-submit=Log+In:Invalid username"

# ffuf — web fuzzer, extremely fast
ffuf -w /usr/share/wordlists/rockyou.txt \
  -u http://<TARGET_IP>/wp-login.php \
  -d "log=admin&pwd=FUZZ&wp-submit=Log+In" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -fs <SIZE_OF_FAILED_RESPONSE>
```

**Use Intruder for:** Small lists (under 100 entries), manual testing,
proof of concept. **Use hydra/ffuf for:** Full wordlist attacks.

---

## Exam-Day Burp Workflow

```
Target web app found
        ↓
1. Launch Burp (burpsuite &)
2. Set Firefox proxy → 127.0.0.1:8080
3. Add target to /etc/hosts if needed
4. Ensure Intercept is OFF
5. Browse entire application — click every page, form, link
        ↓
6. Review HTTP History
   → Find POST requests (login forms, data submission)
   → Find GET requests with parameters (?id=, ?page=, ?file=)
   → Note all cookies being set
        ↓
7. For each interesting request:
   → Right-click → Send to Repeater
   → Manually test: SQLi, path traversal, command injection
   → Compare responses to baseline
        ↓
8. For login forms:
   → Send POST to Intruder
   → If small list: use Intruder
   → If large wordlist: use hydra or ffuf instead
        ↓
9. Document every anomalous response
   → Different size, status code, or content = finding to investigate
```

---

## Key Keyboard Shortcuts (Exam Speed)

| Action | Shortcut |
|---|---|
| Send to Repeater from History | Right-click → R |
| Send request in Repeater | Ctrl+Enter |
| Switch between Repeater tabs | Ctrl+Shift+← / → |
| Search in request/response | Ctrl+F |
| Go to Proxy History | Ctrl+Shift+H |

---

## Reading Response Differences — The Core Testing Skill

Every test in Burp comes down to comparing responses. Train yourself to look
for these signals:

```
Baseline (normal parameter):  HTTP 200, Size: 4521, contains "Invalid login"
Modified (injected payload):   HTTP 302, Size: 0, Location: /admin/

→ 302 redirect = application logged you in or processed the payload
→ Completely different status code = your input changed server behaviour
→ Always means: investigate this payload further
```

```
Baseline:  HTTP 200, Size: 4521
Modified:  HTTP 200, Size: 4489   ← 32 bytes shorter

→ Server stripped something from the response
→ Could indicate filtering, could indicate a condition evaluated differently
→ Try more variations of this payload
```

```
Baseline:  HTTP 200, Size: 4521, fast response (50ms)
Modified:  HTTP 200, Size: 4521, slow response (5000ms)

→ Time-based blind injection — server is executing something that takes time
→ Classic time-based SQL injection signal
→ Investigate with time-based SQLi payloads
```

---

## Gotchas & Exam Tips

- **Browser frozen?** Intercept is ON. Turn it off immediately.

- **HTTPS sites not loading through proxy?** CA certificate not imported.
  Go to `http://burp` → download cert → import to Firefox.

- **`detectportal.firefox.com` flooding History?** Disable captive portal
  checking: `about:config` → `network.captive-portal-service.enabled` → false.

- **Intruder is too slow for large wordlists.** Use hydra or ffuf for
  anything over 100 passwords. Intruder is for small targeted lists.

- **Always add exam machine hostnames to `/etc/hosts` first.** Without this,
  Burp sees IP addresses in the Host header, which breaks virtual host routing
  and causes the same empty-reply issues you saw with curl.

- **Send everything interesting to Repeater immediately.** Even if you are
  not ready to test it yet. You can always go back to a Repeater tab. If you
  lose the request from History (it scrolls off, or you clear it) and you
  did not send it to Repeater, you have to navigate back to that page and
  reproduce the request.

- **Right-click → Copy as curl command.** Any request in Burp History or
  Repeater can be exported as a ready-to-run curl command. Useful for
  putting requests into scripts or notes:
  `Right-click → Copy as curl command → paste into terminal`

- **Check the Response tab for hidden fields.** Rendered pages in Firefox
  hide HTML comments, hidden input fields, and disabled form elements. Burp's
  raw response shows everything. Developer comments frequently contain internal
  paths, version numbers, or credentials left in the code.

---

## Quick Reference — Key Commands and Locations

```
Launch Burp:
  burpsuite &

Firefox proxy setting:
  about:preferences#general → Network Settings → Manual → 127.0.0.1:8080

Burp CA cert download:
  http://burp  (with Firefox proxied through Burp)

Disable captive portal noise:
  about:config → network.captive-portal-service.enabled → false

Add target to /etc/hosts:
  echo "<TARGET_IP> <HOSTNAME>" | sudo tee -a /etc/hosts

Intercept toggle:
  Proxy → Intercept → click to toggle on/off

Proxy listener location:
  Proxy → Options → Proxy Listeners (default: 127.0.0.1:8080)

Send to Repeater:
  HTTP History → right-click request → Send to Repeater

Send to Intruder:
  HTTP History → right-click request → Send to Intruder

Intruder positions:
  Intruder → Positions → Clear § → highlight value → Add §

Intruder wordlist:
  Intruder → Payloads → Payload Options → Load / Paste
```

---

## Next Steps

- Login brute force (large wordlist) → use `hydra` or `ffuf` instead of
  Intruder
- SQL injection confirmed in Repeater → `playbooks/sqli.md`
- File parameter found → test for LFI → `playbooks/file-inclusion.md`
- Command injection confirmed → `playbooks/command-injection.md`
- Admin panel accessed → `playbooks/web-post-auth-exploitation.md`
