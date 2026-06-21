# Browser Developer Tools — Web Application Enumeration

## TL;DR
Every modern browser ships with a built-in suite of analysis tools that let
you inspect a web application's source code, JavaScript files, HTML structure,
hidden form fields, HTTP headers, cookies, and network requests — all without
sending a single additional probe to the server. On the OSCP exam, these tools
are open in a browser tab the entire time you are testing a web application.
They are passive, instant, and reveal things no scanner will find.

---

## What Are Browser Developer Tools?

Developer tools (DevTools) are a built-in debugging interface that browsers
expose for web developers to inspect and troubleshoot their own applications.
For penetration testers, they serve a different purpose: they let you see the
internal structure of an application as the server intended it — not the
rendered visual output, but the raw HTML, JavaScript logic, hidden fields,
API calls, and security headers underneath.

**Why this matters:** The browser renders a web page to look clean and
functional. DevTools lets you see what the browser is actually doing to
produce that result — including things the developer may not have intended
to expose.

Open DevTools in Firefox:
```
F12                          → opens DevTools (fastest)
Ctrl+Shift+I                 → alternative keyboard shortcut
Right-click any element → Inspect   → opens Inspector on that element
```

The DevTools panel has multiple tabs. The ones you will use for web app
enumeration:

| Tab | What It Shows | When to Use |
|-----|--------------|-------------|
| **Inspector** | Full HTML source, live DOM tree | Finding hidden fields, form structure |
| **Debugger** | All JavaScript files loaded by the page | Finding JS logic, API endpoints, secrets |
| **Network** | Every HTTP request/response the page makes | Watching API calls, finding endpoints |
| **Console** | JavaScript runtime environment | Running JS queries, reading errors |
| **Storage** | Cookies, localStorage, sessionStorage | Reading session tokens, client-side data |

---

## What is the DOM and Why Does "View Source" Miss Things?

When you press `Ctrl+U` in Firefox (View Page Source), you see the HTML the
server sent. That is static — it is what the server delivered before the
browser processed it.

The **DOM (Document Object Model)** is what the browser builds from that
HTML after running all JavaScript. JavaScript can add, remove, and modify
HTML elements after the page loads. A hidden field added by JavaScript after
page load will **not** appear in View Source — it only appears in the Inspector
which shows the live DOM.

**Exam implication:** Always use Inspector (live DOM) over View Source. Forms
frequently have hidden fields injected by JavaScript that control authentication
tokens, redirect destinations, or role parameters — and those fields are your
manipulation targets in Burp.

---

## Technique 1 — URL Inspection

Before opening any tool, read the URL. It reveals technology choices instantly.

### File Extensions in URLs

```
http://target/index.php          → PHP backend
http://target/login.asp          → Classic ASP (old Windows/IIS)
http://target/dashboard.aspx     → ASP.NET (modern Windows/IIS)
http://target/login.jsp          → Java (Tomcat, JBoss)
http://target/app.do             → Java Struts framework
http://target/api/v1/users.json  → REST API returning JSON
http://target/home.py            → Python backend
```

Each extension maps directly to a technology and its known vulnerability set:

```bash
# From URL extension to exploit search
URL shows .php    → searchsploit php <version>
URL shows .jsp    → searchsploit apache tomcat
URL shows .aspx   → searchsploit asp.net iis
```

### Routes vs Extensions

Modern frameworks use **routes** — clean URLs with no extension:

```
http://target/user/profile/1
http://target/admin/dashboard
http://target/api/orders
```

No extension visible. The framework maps the URL path to code internally.
When you see routes, fingerprint the framework from other sources (WhatWeb,
response headers, error pages) to know what technology is underneath.

### URL Parameters — Immediate Test Targets

Any URL parameter is an input point:

```
http://target/article.php?id=1           → test for SQLi: id=1'
http://target/page.php?file=about        → test for LFI: file=../../../etc/passwd
http://target/search.php?q=hello         → test for XSS: q=<script>alert(1)</script>
http://target/redirect.php?url=http://x  → test for open redirect
```

Write down every URL parameter you see while browsing. Each one is a
hypothesis to test in Burp Repeater.

---

## Technique 2 — Inspector Tool (HTML and DOM)

### How to Open

```
F12 → Inspector tab
OR
Right-click any element on the page → Inspect
```

Right-clicking an element opens Inspector directly at that element's HTML —
saves time finding it manually in the tree.

### What to Look For

#### Hidden Form Fields

HTML forms frequently contain hidden fields — inputs the user cannot see or
modify via the browser interface, but which are submitted with the form data:

```html
<input type="hidden" name="user_role" value="user">
<input type="hidden" name="redirect_to" value="/dashboard">
<input type="hidden" name="token" value="abc123def456">
<input type="hidden" name="price" value="99.99">
```

These are invisible to the user but fully visible in Inspector — and fully
modifiable in Burp Repeater. Common attacks:

| Hidden Field | Attack |
|---|---|
| `user_role=user` | Change to `admin` — privilege escalation |
| `price=99.99` | Change to `0.01` — price manipulation |
| `redirect_to=/dashboard` | Change to external URL — open redirect |
| `token=abc123` | Copy the value — CSRF token you need for Burp requests |

#### HTML Comments

Developers leave comments in HTML that they forget are publicly visible:

```html
<!-- TODO: remove debug mode before production -->
<!-- Admin panel at /admin_v2_final/ -->
<!-- DB: mysql://root:password@localhost/appdb -->
<!-- test credentials: admin/Summer2023! -->
```

These appear in Inspector (and View Source). To search all comments:

```bash
# Via curl — faster than manual browsing for comment hunting
curl -k -s https://<HOSTNAME>/ | grep -oE "<!--.*?-->" | head -20

# Multi-line comments
curl -k -s https://<HOSTNAME>/ | grep -A2 "<!--"
```

#### Form Action and Method

Always inspect the form element itself — not just the fields inside it:

```html
<form action="/en/account/login" method="POST" enctype="multipart/form-data">
```

- `action=` → exact URL the form submits to — this is what you target in Burp
- `method=` → POST or GET — determines where form data appears
- `enctype=` → how data is encoded:
  - `application/x-www-form-urlencoded` → standard form (default)
  - `multipart/form-data` → file upload form — note this for upload testing

#### Disabled and Read-Only Fields

```html
<input type="text" name="username" value="admin" disabled>
<input type="text" name="account_id" value="1001" readonly>
```

`disabled` and `readonly` are **client-side only controls**. The server
does not enforce them. In Burp Repeater you can submit any value for these
fields regardless of how the HTML marks them. A `readonly` account_id field
is an IDOR waiting to happen.

---

## Technique 3 — Debugger Tool (JavaScript Source)

### How to Open

```
F12 → Debugger tab
```

The left panel shows every file loaded by the page — HTML, JavaScript, CSS,
images. The JavaScript files are your focus.

### Why JavaScript Files Matter

Client-side JavaScript contains:
- **API endpoints** the application calls (often not linked anywhere in HTML)
- **Authentication logic** (token formats, session handling)
- **Input validation rules** (which you then bypass in Burp)
- **Hardcoded credentials or API keys** (developer mistakes)
- **Internal path references** and **commented-out debug code**
- **Framework and library versions** (for CVE matching)

### Minified JavaScript — What It Is and How to Read It

When a developer ships JavaScript to production, they typically **minify** it:
remove all whitespace, newlines, and comments, and shorten variable names to
single letters. This reduces file size and makes it harder to read but does
not protect the logic.

Minified code looks like:
```javascript
var a=document.getElementById("login"),b=function(){fetch("/api/auth",{method:"POST",body:JSON.stringify({user:a.value,pass:btoa(document.getElementById("pwd").value)})})};
```

**To make it readable — Pretty Print:**

In Firefox Debugger, with a JS file open:
```
Click the { } icon at the bottom left of the source panel
```

This reformats the code with proper indentation and line breaks:
```javascript
var a = document.getElementById("login"),
    b = function() {
        fetch("/api/auth", {
            method: "POST",
            body: JSON.stringify({
                user: a.value,
                pass: btoa(document.getElementById("pwd").value)
            })
        });
    };
```

Now you can see: the app calls `/api/auth` via POST with a JSON body, and
the password is base64-encoded (`btoa`) before sending.

**Exam implication:** This tells you the exact API endpoint and request
format. You can now craft this request manually in Burp Repeater with
full knowledge of the expected structure.

### What to Search For in JavaScript Files

With the file pretty-printed, use `Ctrl+F` to search for:

```
api           → API endpoint paths
fetch(        → JavaScript API calls — reveals endpoints and methods
XMLHttpRequest → older API call style
password      → any reference to password handling
token         → token format and where it is stored
secret        → hardcoded secrets or keys
key           → API keys
admin         → admin-specific paths or logic
http          → hardcoded URLs
/api/         → API route discovery
eval(         → dangerous function — potential injection point
```

### Finding API Endpoints in JavaScript

```javascript
// Common patterns that reveal hidden API endpoints
fetch('/api/v1/users')
$.ajax({url: '/internal/admin/reset'})
axios.get('/api/orders/' + userId)
const API_BASE = 'https://api.target.com/v2';
```

Every URL string in JavaScript is a potential endpoint to probe. Add them
all to your target list and test them in Burp.

---

## Technique 4 — Console Tab (Active JavaScript Queries)

The Console tab lets you run JavaScript in the context of the current page.
Use it for instant queries without writing any code in external tools.

```javascript
// Find jQuery version immediately
jQuery.fn.jquery

// Find all cookies readable by JavaScript (HttpOnly cookies won't appear)
document.cookie

// Find all form elements and their names
document.querySelectorAll('input').forEach(i => console.log(i.name, i.type, i.value))

// Find all hidden fields
document.querySelectorAll('input[type=hidden]').forEach(i => console.log(i.name, i.value))

// Find all links on the page (discover paths)
document.querySelectorAll('a').forEach(a => console.log(a.href))

// Find all form action URLs
document.querySelectorAll('form').forEach(f => console.log(f.action, f.method))

// Check what localStorage contains
JSON.stringify(localStorage)

// Check sessionStorage
JSON.stringify(sessionStorage)
```

These run instantly in the browser and give you structured output you can
read and act on. The hidden fields query is particularly fast — it dumps
every hidden field name and value in one line.

---

## Technique 5 — Page Source Search (Ctrl+U then Ctrl+F)

For a quick sweep of the raw server-sent HTML before JavaScript modifies it:

```
Ctrl+U → View Page Source → Ctrl+F to search
```

Search for:

```
password    → hardcoded or referenced passwords
admin       → admin paths or references
TODO        → developer notes
FIXME       → known issues the developer noted
api_key     → hardcoded API keys
secret      → any secret values
<!--        → all HTML comments
type="hidden"  → all hidden form fields
```

This is fast for initial recon. Follow up with Inspector for the live DOM
if you suspect JavaScript is adding fields after load.

---

## Exam-Day Browser Enumeration Sequence

Run this every time you land on a new web page of interest:

```
1. Read the URL
   → Note file extensions → technology hint
   → Note URL parameters → test each one in Burp

2. View Page Source (Ctrl+U)
   → Search: password, admin, TODO, <!--, api_key, type="hidden"
   → Note any comments or hardcoded values

3. Open Inspector (F12 → Inspector OR right-click → Inspect)
   → Find all hidden form fields → note name and value of each
   → Read form action URL and method → this is your Burp target
   → Look for disabled/readonly fields → test them in Burp regardless

4. Open Debugger (F12 → Debugger)
   → List all JavaScript files loaded
   → Open each .js file → pretty print → search for api, fetch, token, key
   → Note every URL string found → add to enumeration target list

5. Open Console (F12 → Console)
   → Run: document.querySelectorAll('input[type=hidden]')
            .forEach(i => console.log(i.name, i.value))
   → Run: jQuery.fn.jquery (if jQuery suspected)
   → Run: document.querySelectorAll('form')
            .forEach(f => console.log(f.action, f.method))

6. Feed all findings into Burp
   → Hidden fields → modify in Repeater
   → API endpoints → probe in Repeater
   → Form action URL → target for Intruder if login form
```

---

## What Each Finding Feeds Into

| Finding | Next Action |
|---|---|
| URL with `?id=` parameter | Burp Repeater → test SQLi, IDOR |
| URL with `?file=` parameter | Burp Repeater → test LFI/path traversal |
| Hidden field `role=user` | Burp Repeater → change to `admin` |
| Hidden field `price=99.99` | Burp Repeater → change to `0.01` |
| HTML comment with path | Add path to Gobuster wordlist, browse directly |
| HTML comment with credentials | Test immediately on all login panels |
| API endpoint in JavaScript | Probe in Burp Repeater → test auth, injection |
| Base64 encoded value in JS | Decode: `echo "<value>" \| base64 -d` |
| jQuery version | `searchsploit jquery <version>` |
| Hardcoded API key | Test against the API directly |
| Disabled input field | Burp Repeater → submit it anyway |

---

## Gotchas & Exam Tips

- **Inspector shows the live DOM, View Source shows the server response.**
  These are different. JavaScript adds hidden fields, modifies forms, and
  inserts content after page load. Always use Inspector for the definitive
  picture.

- **Pretty print everything.** Minified JavaScript is not unreadable —
  it is just unformatted. One click of `{ }` makes it searchable. Skipping
  this step means you miss API endpoints that are plainly visible once
  formatted.

- **Hidden fields are your highest-value Inspector finding.** On OSCP
  machines, hidden fields frequently contain: CSRF tokens (copy these into
  Burp requests or they fail), role parameters (change these for privilege
  escalation), price values (modify for logic flaws), and redirect
  destinations (manipulate for open redirects).

- **`disabled` and `readonly` mean nothing to Burp.** These are browser
  UI hints only. The server does not know you bypassed them. Always test
  whether the server actually enforces restrictions on fields marked this way.

- **JavaScript files sometimes contain staging/development endpoints.** A
  `.js` file might reference `https://dev.target.com/api/` or
  `/internal/debug/` — paths that were never removed before production
  deployment. These endpoints often have weaker authentication.

- **Search page source for version strings.** `jQuery v3.3.1`, `Bootstrap
  4.0.0`, `WordPress 5.7` frequently appear as comments in JS and CSS files.
  This confirms what WhatWeb detected and gives you exact versions to
  searchsploit.

---

## Quick Reference — Key Actions

```
Open DevTools:         F12
Open Inspector on element:  Right-click → Inspect
Open Console:          F12 → Console tab
View Page Source:      Ctrl+U
Search in source:      Ctrl+F

Pretty print JS:       Debugger → open .js file → click {} icon (bottom left)

Console queries:
  All hidden fields:   document.querySelectorAll('input[type=hidden]')
                         .forEach(i => console.log(i.name, i.value))
  All forms:           document.querySelectorAll('form')
                         .forEach(f => console.log(f.action, f.method))
  All links:           document.querySelectorAll('a')
                         .forEach(a => console.log(a.href))
  jQuery version:      jQuery.fn.jquery
  All cookies (non-HttpOnly): document.cookie

curl for comment hunting:
  curl -k -s https://<HOSTNAME>/ | grep -oE "<!--.*?-->"

Decode base64 value:
  echo "<value>" | base64 -d
```

---

## Next Steps

- Hidden API endpoints found → probe in `tools/burp-suite.md` Repeater
- Hidden `role` / `admin` fields found → modify in Burp → test access control
- JavaScript credentials found → test on all login panels
- Version strings confirmed → `searchsploit <component> <version>`
- Form action URL identified → Burp Intruder for login brute force
- URL parameters identified → `playbooks/sqli.md`, `playbooks/lfi.md`
