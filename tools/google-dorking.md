# Google Dorking

# Purpose

Use search engines to identify:
- exposed files
- sensitive documents
- login portals
- backups
- directory listings
- configuration files
- credentials
- source code exposure
- technology stack indicators

---

# Core Principle

Google Dorking is:
- passive reconnaissance
- low-noise intelligence gathering
- attack surface expansion

Search engines often index:
- forgotten files
- exposed directories
- backup archives
- developer artifacts
- sensitive metadata

---

# Primary Resources

Google Hacking Database (GHDB):
https://www.exploit-db.com/google-hacking-database

DorkSearch:
https://dorksearch.com/

Google Search Operators:
https://support.google.com/websearch/answer/2466433

---

# Important Search Operators

| Operator | Purpose |
|---|---|
| site: | Limit results to domain |
| filetype: | Search specific file types |
| ext: | Alternative to filetype |
| intitle: | Search page title |
| inurl: | Search URL |
| - | Exclude results |
| "" | Exact phrase |

---

# Basic Workflow

## 1. Identify Web Presence

```text
site:target.com
````

Goal:

- identify indexed pages
    
- discover subdomains
    
- map public web exposure
    

---

## 2. Discover Sensitive Files

### PDFs

```text
site:target.com filetype:pdf
```

Potential Findings:

- internal documentation
    
- usernames
    
- email addresses
    
- software versions
    

---

### Text Files

```text
site:target.com filetype:txt
```

Potential Findings:

- notes
    
- credentials
    
- config remnants
    

---

### Office Documents

```text
site:target.com (filetype:doc OR filetype:docx OR filetype:xls OR filetype:ppt)
```

Potential Findings:

- metadata
    
- usernames
    
- internal naming conventions
    

---

## 3. Discover Login Portals

```text
site:target.com inurl:login
```

Additional Variants:

```text
site:target.com admin
site:target.com portal
site:target.com inurl:auth
```

Goal:

- identify authentication surfaces
    
- enumerate admin panels
    

---

## 4. Find Directory Listings

```text
site:target.com intitle:"index of"
```

OR

```text
intitle:"index of" "parent directory"
```

Potential Findings:

- backups
    
- source code
    
- internal files
    
- exposed shares
    

---

## 5. Identify Exposed Backups

```text
site:target.com ext:bak
```

```text
site:target.com ext:old
```

```text
site:target.com ext:zip
```

```text
site:target.com ext:tar
```

Potential Findings:

- archived web roots
    
- database exports
    
- source code backups
    

---

## 6. Find Configuration Files

```text
site:target.com ext:env
```

```text
site:target.com ext:ini
```

```text
site:target.com ext:conf
```

Potential Findings:

- API keys
    
- credentials
    
- database connection strings
    

---

## 7. Technology Stack Enumeration

### PHP

```text
site:target.com ext:php
```

### ASPX

```text
site:target.com ext:aspx
```

### Python

```text
site:target.com ext:py
```

Goal:

- identify backend technologies
    
- tailor exploitation approach
    

---

## 8. Git Exposure

```text
site:target.com ".git"
```

Potential Findings:

- exposed repositories
    
- source code leakage
    

---

# Excluding Results

Use `-` to filter noise.

Example:

```text
site:target.com -filetype:html
```

Purpose:

- remove standard web pages
    
- isolate interesting files
    

---

# robots.txt Enumeration

Search engines may reveal:

- hidden directories
    
- disallowed paths
    
- sensitive endpoints
    

Example Finding:

```text
User-agent: *
Allow: /
Allow: /nanites.php
```

Potential Pivot:

- hidden application functionality
    
- admin endpoints
    
- internal tools
    

---

# Recon Mindset

Do NOT search randomly.

Each finding should create pivots.

Example:

PDF found  
→ extract usernames  
→ generate user list

Backup found  
→ download archive  
→ search credentials

Directory listing found  
→ enumerate contents  
→ locate configs/source

Technology identified  
→ search CVEs/exploits

---

# Important OSCP Insight

Google Dorking:

- reduces noisy scanning
    
- expands attack surface
    
- identifies hidden content
    
- reveals sensitive files before active enumeration begins
    

Passive recon often makes later exploitation significantly easier.

---

# Common OSCP-Relevant Dorks

## Exposed SQL Dumps

```text
site:target.com ext:sql
```

---

## Open Directories

```text
site:target.com intitle:"index of"
```

---

## Environment Files

```text
site:target.com ext:env
```

---

## Login Portals

```text
site:target.com inurl:login
```

---

## Config Files

```text
site:target.com ext:conf
```

---

## Backup Archives

```text
site:target.com ext:zip
```

---

# Limitations

- Results depend on search engine indexing
    
- Some pages may be removed/de-indexed
    
- Modern applications may block indexing
    
- Internal systems usually invisible
    

---

# Exam Strategy

During OSCP:

- spend limited time on passive recon
    
- quickly identify:
    
    - login portals
        
    - exposed files
        
    - technology stack
        
    - hidden directories
        

Then pivot into:

- active enumeration
    
- service exploitation
    
- web attacks