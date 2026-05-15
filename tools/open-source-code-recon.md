# Open Source Code Recon

# Objective

Use public source code repositories to discover:

- credentials
- API keys
- usernames
- internal URLs
- technologies
- frameworks
- configuration files
- developer mistakes
- sensitive information

---

# OSCP Mindset

Manual analysis is MORE important than automation.

Goal:
- understand what to look for
- build enumeration discipline
- identify exposed secrets manually

Automation helps later.

---

# Common Sources

- GitHub
- GitLab
- GitHub Gist
- SourceForge

---

# Recon Workflow

## 1. Identify Organization Repositories

Example:

https://github.com/megacorpone

Look for:
- web applications
- config repos
- infrastructure repos
- backup repos
- developer repos

---

# 2. Manual GitHub Search

Useful search operators:

## Search by Filename

```text
filename:users
```

---

## Search for Environment Files

```text
filename:.env
```

---

## Search for Config Files

```text
filename:config
```

---

## Search for Passwords

```text
password
```

---

## Search for API Keys

```text
api_key
```

---

## Search for AWS Keys

```text
AKIA
```

---

## Search for Private Keys

```text
BEGIN RSA PRIVATE KEY
```

---

# 3. Clone Repository Locally

## Clone Repo

```bash
git clone https://github.com/<org>/<repo>.git
```

Example:

```bash
git clone https://github.com/megacorpone/megacorpone.com.git
```

---

# 4. Manual Recursive Searching

Move into repo:

```bash
cd repo
```

---

## Search for Passwords

```bash
grep -Rni "password" .
```

---

## Search for Usernames

```bash
grep -Rni "user" .
```

---

## Search for Secrets

```bash
grep -RniE "secret|token|key" .
```

---

## Search for API Keys

```bash
grep -Rni "api" .
```

---

## Search for AWS Keys

```bash
grep -Rni "AKIA" .
```

---

## Search for Internal IPs

```bash
grep -RniE "10\.|172\.|192\.168" .
```

---

## Search for Emails

```bash
grep -RniE "@gmail|@company|@corp" .
```

---

## Search for SSH Keys

```bash
find . -name "*.pem"
```

---

## Search for Interesting Files

```bash
find . | grep -Ei "config|env|backup|bak|old|secret"
```

---

# 5. Inspect Commit History

IMPORTANT:
Secrets may have been deleted later.

Git preserves history.

---

## View Commit Log

```bash
git log
```

---

## View Specific Commit

```bash
git show <commit_hash>
```

---

## Search Entire History

```bash
git log -p | grep password
```

---

# 6. Identify Technologies

Look for:

| File | Technology |
|---|---|
| package.json | NodeJS |
| requirements.txt | Python |
| pom.xml | Java |
| composer.json | PHP |
| Dockerfile | Containers |
| .gitlab-ci.yml | CI/CD |
| Jenkinsfile | Jenkins |

---

# 7. Exploitation Value

Potential findings:

| Finding | Possible Use |
|---|---|
| usernames | password spraying |
| emails | phishing |
| passwords | direct login |
| API keys | cloud compromise |
| internal IPs | network mapping |
| source code | vulnerability discovery |
| config files | secrets / architecture |

---

# GitHub Dorks

## Search GitHub via Google

```text
site:github.com target
```

---

## Search for Env Files

```text
site:github.com ext:env
```

---

## Search for Passwords

```text
site:github.com password
```

---

# Automation Tools

## GitXray

Purpose:
- contributor analysis
- repo metadata
- secret discovery

Basic usage:

```bash
gitxray -r owner/repo
```

Example:

```bash
gitxray -r megacorpone/megacorpone.com
```

---

## Important Note

GitXray requires GitHub API access.

Without token:
- severe rate limiting
- unstable scans
- connection resets possible

---

# Optional GitHub Token Setup

Create free token:
https://github.com/settings/tokens

Export token:

```bash
export GH_ACCESS_TOKEN="TOKEN"
```

Verify:

```bash
echo $GH_ACCESS_TOKEN
```

---

# Gitleaks

Purpose:
- detect secrets
- scan commits
- identify exposed credentials

Install:

```bash
sudo apt install gitleaks
```

Basic usage:

```bash
gitleaks detect
```

---

# Trufflehog

Purpose:
- scan git history for secrets

Install:

```bash
pipx install trufflehog
```

Run:

```bash
trufflehog git https://github.com/org/repo.git
```

---

# OSCP-Relevant Reality

In OSCP:
- manual review matters more
- grep/find/git skills matter more
- methodology matters more than tools

Most useful skills:
- recursive grep
- git history inspection
- identifying sensitive files
- understanding developer mistakes

---

# High-Value Targets

Always inspect:

- .env
- config.php
- wp-config.php
- id_rsa
- backup files
- SQL dumps
- Dockerfiles
- Jenkins configs
- CI/CD configs
- credentials.json
- users.txt

---

# Important Enumeration Principle

Open-source code recon is NOT just about secrets.

It also reveals:
- technologies
- frameworks
- attack surface
- internal naming conventions
- usernames
- developer behavior
- infrastructure architecture

These become pivots later during exploitation.