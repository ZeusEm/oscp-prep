# whois

## Description
WHOIS is a TCP-based protocol (port 43) used to query domain and IP ownership information.

## Basic Usage

`whois <domain>`
`whois <IP>`

---

## Specify WHOIS Server

`whois -h <server> <target>`

`-h <server>` → specify WHOIS server

Example:
`whois -h 192.168.50.251 example.com`

---

## Output Focus

Ignore noise. Extract:

- Registrant
- Organization
- Email
- Name Servers
- Net Range

---

## When to Use

- Early recon
- Domain-based targets
- Expanding attack surface

---

## ⚠️ Limitations

- Private registration may hide data
- Not always useful in internal lab environments
