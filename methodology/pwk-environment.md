# PWK Environment & Exam Rules

## Flag Behavior
- Flags are dynamically generated per machine boot
- Flags expire on machine revert or shutdown
- Must submit flag BEFORE reverting machine

---

## SSH Access to Lab Machines

Recommended command:

`ssh -o "UserKnownHostsFile=/dev/null" -o "StrictHostKeyChecking=no" learner@<IP>`

Purpose:
- Prevents SSH key conflicts when machines are reset/reused

---

## VPN & Network Structure

- VPN provides tun0 interface
- IP format: 192.168.119.X (your machine)

Lab machines:
- Format: 192.168.X.Y
- X matches your tun0 third octet

Example:
Your IP: 192.168.119.5  
Target: 192.168.119.23

---

## Restrictions

- Client-to-client VPN traffic is NOT allowed
- No attacking other students' machines

---

## Flags in OSCP

- local.txt → user-level access
- proof.txt → root/admin access

These are required for exam scoring.
