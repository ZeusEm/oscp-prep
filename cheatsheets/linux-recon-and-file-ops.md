# Linux Recon & File Operations Cheatsheet

# Objective

Rapid Linux command reference for:
- enumeration
- file operations
- repo analysis
- privilege escalation
- loot extraction
- OSCP workflows

Focus:
- speed
- operational usage
- copy-paste readiness

---

# File Operations

## List Files

```bash
ls
```

---

## Detailed Listing

```bash
ls -la
```

---

## Create File

```bash
touch file.txt
```

---

## Create Directory

```bash
mkdir dirname
```

---

## Copy File

```bash
cp source.txt dest.txt
```

---

## Move File

```bash
mv file.txt /tmp/
```

---

## Rename File

```bash
mv old.txt new.txt
```

---

## Remove File

```bash
rm file.txt
```

---

## Remove Directory

```bash
rm -rf directory/
```

---

# Permissions

## View Permissions

```bash
ls -l
```

---

## Add Execute Permission

```bash
chmod +x script.py
```

---

## Remove Execute Permission

```bash
chmod -x script.py
```

---

## Numeric Permissions

```bash
chmod 755 script.py
```

---

## Change Ownership

```bash
sudo chown user:user file
```

---

# Viewing Files

## Print File

```bash
cat file.txt
```

---

## Scroll File

```bash
less file.txt
```

---

## First 10 Lines

```bash
head file.txt
```

---

## Last 10 Lines

```bash
tail file.txt
```

---

## Follow Log File

```bash
tail -f logfile.log
```

---

# Searching

## Find Files

```bash
find / -name file.txt 2>/dev/null
```

---

## Find by Extension

```bash
find . -name "*.conf"
```

---

## Find Writable Directories

```bash
find / -writable 2>/dev/null
```

---

## Find SUID Binaries

```bash
find / -perm -4000 2>/dev/null
```

---

## Find Interesting Files

```bash
find . | grep -Ei "config|env|backup|secret|pass"
```

---

# grep

## Search String

```bash
grep "password" file.txt
```

---

## Recursive Search

```bash
grep -Rni "password" .
```

Flags:
- R → recursive
- n → line numbers
- i → case insensitive

---

## Multiple Patterns

```bash
grep -RniE "password|secret|token" .
```

---

## Search Internal IPs

```bash
grep -RniE "10\.|172\.|192\.168" .
```

---

## Search Emails

```bash
grep -Rni "@gmail\|@corp\|@company" .
```

---

# Filtering & Parsing

## Unique Results

```bash
sort file.txt | uniq
```

---

## Cut Columns

```bash
cut -d ":" -f1 file.txt
```

---

## awk

Print first column:

```bash
awk '{print $1}'
```

---

## sed Replace

```bash
sed 's/root/admin/g' file.txt
```

---

# Networking

## Check IP Address

```bash
ip a
```

---

## Show Routes

```bash
ip route
```

---

## Ping Host

```bash
ping 192.168.1.10
```

---

## Netcat Listener

```bash
nc -lvnp 4444
```

---

## Connect with Netcat

```bash
nc 192.168.1.10 4444
```

---

## Download File

```bash
wget http://IP/file
```

---

## Download with curl

```bash
curl -O http://IP/file
```

---

## Port Enumeration

```bash
ss -tulpn
```

---

# Archives

## Extract tar.gz

```bash
tar -xvf file.tar.gz
```

---

## Create tar.gz

```bash
tar -cvf archive.tar dir/
```

---

## Unzip

```bash
unzip file.zip
```

---

# Processes

## View Processes

```bash
ps aux
```

---

## Interactive Process View

```bash
top
```

---

## Kill Process

```bash
kill -9 PID
```

---

# Git Operations

## Clone Repository

```bash
git clone https://github.com/org/repo.git
```

---

## Git Log

```bash
git log
```

---

## Show Commit

```bash
git show <commit_hash>
```

---

## Search Git History

```bash
git log -p | grep password
```

---

# Python Quick Server

## Python3 HTTP Server

```bash
python3 -m http.server 80
```

Useful for:
- file transfer
- payload hosting
- exploit delivery

---

# File Transfers

## Download via wget

```bash
wget http://IP/file
```

---

## Download via curl

```bash
curl -O http://IP/file
```

---

## Transfer with Netcat

Listener:

```bash
nc -lvnp 4444 > file
```

Sender:

```bash
nc IP 4444 < file
```

---

# Useful OSCP One-Liners

## Find Password Files

```bash
find / -iname "*pass*" 2>/dev/null
```

---

## Find Config Files

```bash
find / -iname "*.conf" 2>/dev/null
```

---

## Find SSH Keys

```bash
find / -name "id_rsa*" 2>/dev/null
```

---

## Search for Credentials

```bash
grep -RniE "password|user|token|secret" .
```

---

## Find World Writable Files

```bash
find / -perm -2 -type f 2>/dev/null
```

---

## Find World Writable Directories

```bash
find / -perm -2 -type d 2>/dev/null
```

---

# Important OSCP Principle

Linux command fluency:
- saves time
- improves enumeration
- reduces mistakes
- improves pivoting
- accelerates privilege escalation

Most OSCP failures are methodology failures, not tool failures.