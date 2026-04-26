## 🌐 Domain Enumeration

IF target includes domain:

  → run:
    whois <domain>

  → extract:
    - names
    - emails
    - name servers
    - IP ranges

  → pivot:
    IF names → username list
    IF email → login attempts
    IF name servers → DNS enum
    IF IP range → expand scanning
