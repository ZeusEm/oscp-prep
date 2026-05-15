# PWK Environment & Exam Rules

# VPN Environment

Upon VPN connection:
- tun0 interface is created

Check:

```bash
ip a
```

Example:

```text
192.168.119.X
```

---

# Lab Addressing

Lab machines follow:

```text
192.168.X.Y
```

X matches your VPN subnet.

---

# SSH Recommendation

Use:

```bash
ssh -o "UserKnownHostsFile=/dev/null" -o "StrictHostKeyChecking=no" learner@IP
```

---

# Why These Options Matter

## StrictHostKeyChecking=no

Prevents SSH warnings when machines are reset/reused.

---

## UserKnownHostsFile=/dev/null

Prevents known_hosts corruption.

Useful because:
- PWK machines are frequently reverted
- SSH fingerprints change often

---

# Flag Behavior

Flags:
- regenerate after machine revert
- expire on shutdown/revert

IMPORTANT:
Submit BEFORE reverting machine.

---

# Exam Flags

## local.txt
User-level proof.

## proof.txt
Root/SYSTEM proof.

---

# Restrictions

Client-to-client VPN attacks are forbidden.

Meaning:
- do NOT attack other students
- only attack assigned targets

Violation may terminate lab access.

---

# Important OSCP Insight

The VPN subnet structure helps:
- standardize scanning
- simplify targeting
- reduce confusion during enumeration
