# Linux Permissions Cheatsheet

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

# Common Numeric Modes

```text
644 = rw-r--r--
755 = rwxr-xr-x
777 = rwxrwxrwx
```

---

# Set Numeric Permissions

```bash
chmod 755 script.py
```

---

# Run Executable

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