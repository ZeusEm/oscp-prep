# Enumeration Methodology

## Step 1: Initial Scan
nmap -sC -sV -p- <IP> -oN initial.txt

## Step 2: Identify Services
- HTTP → web playbook
- SMB → smb playbook
- FTP → ftp playbook

## Step 3: Validate Access
- Anonymous login
- Default creds
- Version check

## Step 4: Expand
- More ports
- Subdomains
- Internal pivot

## Loop
Enumeration → Exploitation → PrivEsc → Repeat
