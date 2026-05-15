# OSCP Note-Taking Standard

# Objective

Notes must support:
- exact reproduction
- rapid review
- reporting
- operational execution

---

# Core Principles

## Record Everything

Document:
- commands
- outputs
- URLs
- credentials
- ports
- screenshots
- modified code
- GUI actions

---

# Eliminate Ambiguity

Bad:

```text
Got shell
```

Good:

```bash
python3 exploit.py 192.168.119.5 4444
nc -lvnp 4444
```

---

# Notes Must Be Reproducible

Another operator should be able to:
- follow notes
- reproduce compromise
- obtain same result

---

# Recommended Machine Structure

```text
IP Address
Hostname
OS Guess

Ports
Services
Versions

Potential Vulnerabilities

Exploitation

Privilege Escalation

Loot
Credentials
Hashes
Flags
```

---

# Screenshot Guidance

Capture:
- proof.txt
- local.txt
- exploit success
- privilege escalation
- important findings

---

# Important Reporting Principle

Poor notes can invalidate successful exploitation.

If exploitation cannot be reproduced:
- reporting becomes weak
- exam documentation suffers

---

# Operational Guidance

Notes should remain understandable when:
- tired
- stressed
- sleep deprived
- time constrained