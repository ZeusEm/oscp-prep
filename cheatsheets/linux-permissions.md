# Linux File Permissions Cheatsheet

# View Permissions

```bash
ls -l
```

---

# Add Execute Permission

```bash
chmod +x file
```

---

# Remove Execute Permission

```bash
chmod -x file
```

---

# Numeric Modes

```text
644 = rw-r--r--
755 = rwxr-xr-x
777 = rwxrwxrwx
```

---

# Set Permissions

```bash
chmod 755 script.py
```

---

# Execute File

```bash
./script.py
```

---

# Common OSCP Fix

```bash
chmod +x exploit.py
```

---

# Rename File

```bash
mv old.txt new.txt
```

---

# Move File

```bash
mv file.txt /path/
```

---

# Permission Symbols

```text
r = 4
w = 2
x = 1
```

---

# Important OSCP Reminder

Many downloaded exploits lose executable permissions.

Fix immediately:

```bash
chmod +x exploit.py
```
