# Web Application Testing — Methodology & Framework

## TL;DR
Web apps are the most common attack surface on the OSCP exam. Before you
touch a single tool, you need a mental model: what type of engagement are you
in, what information do you have, what are you looking for, and in what order.
This note is that mental model.

---

## Why Web Applications Are High-Value Targets

Web applications are attractive to attackers because:

- They are intentionally exposed to the internet — you can reach them without
  bypassing a network firewall
- They are complex: frontend code, backend logic, databases, third-party
  libraries, server configuration — every layer is a potential vulnerability
- They are frequently built under time pressure by developers whose primary
  goal is functionality, not security
- Even secure frameworks can be misconfigured or combined in insecure ways

On the OSCP exam, almost every machine with a web server running is an
intended attack path. A filtered SMB port with a running web server on port
80 is a signal: **the web app is your way in.**

---

## The Three Testing Methodologies

Before starting any web application assessment, establish which testing model
applies. This determines your starting posture and what you do in the first
60 seconds.

### White-Box Testing
You have **full access** to source code, infrastructure diagrams, and design
documentation. You can read the application's logic directly.

- **Strengths:** Most thorough. Can find logic flaws invisible to any scanner.
- **Weaknesses:** Time-intensive. Requires code review skills.
- **Exam relevance:** Rare on OSCP. If you find source code exposed (via
  directory traversal, `.git` folder leakage, backup files), treat it as an
  accidental white-box opportunity — read it.

### Black-Box Testing
You have **zero prior knowledge** of the application. You approach it exactly
as an external attacker would — the application's public interface is all you
have.

- **Strengths:** Realistic attacker simulation. Requires no special access.
- **Weaknesses:** Enumeration is the entire foundation. Skip enumeration steps
  and you miss attack surface.
- **Exam relevance:** This is your default posture on every OSCP web target.
  You arrive at a URL with no credentials, no source code, no hints. Everything
  you discover, you discover through enumeration.

### Grey-Box Testing
You have **partial information** — maybe credentials, maybe knowledge of the
framework, maybe a user account but not an admin one.

- **Strengths:** More efficient than black-box. Credentials allow authenticated
  scanning and access to more app functionality.
- **Weaknesses:** Can create blind spots in areas outside the provided context.
- **Exam relevance:** Occurs when you find credentials through other attack
  vectors (SNMP, FTP, SMB) and can then authenticate to a web app. Shifts your
  approach — now test authenticated functionality too.

---

## OWASP Top 10 — Your Vulnerability Targeting Framework

OWASP (Open Web Application Security Project) is a non-profit that researches
and publishes the most critical web application security risks. Their Top 10
list is the industry standard reference for what to look for when assessing
any web application.

**Think of this list as your ordered checklist of hypotheses.** For every web
app you encounter, mentally run through these categories and ask: "could this
app be vulnerable to this?" The answer guides your tool selection and manual
testing approach.

### OWASP Top 10 (2021 — Current Standard)

| Rank | Vulnerability | What It Means in Practice |
|------|--------------|--------------------------|
| A01 | **Broken Access Control** | Users can access resources or functions they shouldn't — other users' data, admin panels, restricted files |
| A02 | **Cryptographic Failures** | Sensitive data (passwords, tokens, PII) stored or transmitted without proper encryption |
| A03 | **Injection** | Unsanitised user input is passed to an interpreter — SQL, OS commands, LDAP. Your primary attack vector on OSCP |
| A04 | **Insecure Design** | Flaws in the application's architecture and business logic — no technical exploit, just logical abuse |
| A05 | **Security Misconfiguration** | Default credentials, directory listing enabled, verbose error messages, unnecessary services |
| A06 | **Vulnerable and Outdated Components** | The app uses libraries, frameworks, or plugins with known CVEs |
| A07 | **Identification and Authentication Failures** | Weak passwords, broken session management, no account lockout, predictable tokens |
| A08 | **Software and Data Integrity Failures** | App trusts unverified updates or data — insecure deserialisation |
| A09 | **Security Logging and Monitoring Failures** | Not directly exploitable but means attacks go undetected — relevant in real engagements |
| A10 | **Server-Side Request Forgery (SSRF)** | App can be made to make HTTP requests on your behalf — can reach internal services |

### OSCP Exam Focus — Which OWASP Categories Matter Most

On the OSCP exam, you will encounter these most frequently and they have the
highest return on time invested:

```
HIGH PRIORITY (exam staples):
├── A03 Injection          → SQL injection, command injection
├── A05 Misconfiguration   → Default creds, directory listing, exposed admin panels
├── A06 Outdated Components → Known CVE on a plugin/library/framework version
└── A07 Auth Failures      → Weak passwords, credential reuse, broken login logic

MEDIUM PRIORITY (appear regularly):
├── A01 Broken Access Control → IDOR, forced browsing, privilege escalation via URL
└── A10 SSRF               → Appears in more advanced machines

LOWER PRIORITY (less common on OSCP but know the concept):
└── A04 Insecure Design    → Logic flaws — requires deep app understanding
```

---

## Black-Box Web App Testing — Mental Model for Exam Day

Since black-box is your default OSCP posture, here is the mental model to
load every time you hit a web application target. This is the thinking process
before any tool runs.

```
Step 1 — OBSERVE
  What does the application do? (login portal, CMS, custom app, API)
  What technologies are visible? (form fields, file extensions, headers, error messages)
  Is there a login page? A file upload? A search bar? A URL with parameters?

Step 2 — FINGERPRINT
  What web server? (Apache, Nginx, IIS — version matters)
  What language/framework? (PHP, Python/Django, ASP.NET, WordPress, Drupal)
  What is the URL structure? (static files, dynamic parameters, REST API endpoints)

Step 3 — ENUMERATE
  What pages and directories exist? (Gobuster)
  What does the robots.txt or sitemap.xml expose?
  Are there backup files, .git folders, config files in accessible paths?

Step 4 — IDENTIFY INPUTS
  Every input field is a potential injection point (GET/POST parameters,
  HTTP headers, cookies, file uploads)
  Map every place where user-controlled data enters the application

Step 5 — TEST HYPOTHESES
  For each input: test for SQLi, command injection, path traversal
  For each page: test for broken access control (change IDs, access
  admin URLs without authentication)
  For the server: test for known CVEs on the detected version/framework

Step 6 — EXPLOIT
  Confirmed vulnerability → follow the relevant exploitation playbook
```

---

## Web Application Attack Surface — What to Look For Immediately

When a web app loads, scan visually before touching any tool:

| Observation | Why It Matters |
|---|---|
| Login page present | Test default creds; check for SQLi; check for account enumeration |
| URL contains `?id=1` or `?page=about` | Parameter-based input — test for SQLi, LFI, path traversal |
| File upload functionality | Test for unrestricted upload — potential RCE |
| Search bar | Test for SQLi and XSS |
| `.php`, `.asp`, `.aspx` extensions | Language-specific vulnerabilities and exploit paths |
| Error messages with stack traces | Verbose errors reveal framework, paths, database type |
| `admin/`, `login/`, `dashboard/` in URLs | Forced browsing targets — try accessing without auth |
| Copyright year in footer | Old app = outdated components = CVE hunting |
| Powered by X / built with Y | Framework fingerprint → searchsploit for known vulns |

---

## Tools Overview — What Each Tool Does in This Phase

These tools will have their own dedicated notes. This section orients you to
what role each plays in the methodology so you reach for the right tool at
the right moment.

| Tool | Phase | What It Does |
|------|-------|--------------|
| `nmap` (http scripts) | Fingerprint | Service version, HTTP headers, common web vulns |
| Wappalyzer | Fingerprint | Browser extension — identifies tech stack instantly |
| `gobuster` | Enumerate | Brute forces directories and files on the web server |
| Burp Suite | Intercept + Test | Intercepts HTTP traffic, replays requests, manipulates parameters |
| `curl` | Manual probe | Fetch headers, test endpoints, examine raw responses |
| `nikto` | Sweep | Fast scan for known web server misconfigurations and CVEs |

The workflow order is always: **Fingerprint → Enumerate → Manually test →
Exploit**. Tools do not replace the thinking in between steps.

---

## Key Terms Quick Reference

| Term | Definition |
|------|------------|
| White-box | Full source code and infrastructure access |
| Black-box | Zero prior knowledge — attacker's default view |
| Grey-box | Partial information (credentials, framework hints) |
| OWASP | Open Web Application Security Project |
| Attack surface | Every input, endpoint, and parameter a user can interact with |
| Injection | User input passed unsanitised to an interpreter (SQL, OS shell, etc.) |
| IDOR | Insecure Direct Object Reference — changing an ID to access others' data |
| LFI | Local File Inclusion — reading local server files via path parameter |
| RCE | Remote Code Execution — running arbitrary commands on the server |
| Directory traversal | Using `../` sequences to navigate outside the web root |

---

## Next Steps
- Web server fingerprinting → `tools/web-enumeration.md`
- Directory and file discovery → `tools/gobuster.md`
- HTTP interception and manipulation → `tools/burp-suite.md`
- SQL injection → `playbooks/sqli.md`
- Command injection → `playbooks/command-injection.md`
- File inclusion attacks → `playbooks/file-inclusion.md`
