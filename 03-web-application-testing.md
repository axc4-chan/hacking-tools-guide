# 🌐 Web Application Testing

Tools specialized in finding and exploiting vulnerabilities in web applications.

## Contents

- [Burp Suite](#burp-suite--web-security-testing-platform)
- [OWASP ZAP](#owasp-zap--web-app-scanner)
- [Sqlmap](#sqlmap--sql-injection--database-takeover)
- [Gobuster](#gobuster--uri-directory--dns-bruteforcer)
- [Workflow](#-web-application-testing-workflow)

---

## Burp Suite — Web Security Testing Platform

Integrated platform combining an intercepting proxy, scanner, intruder,
repeater, decoder, and extensible module ecosystem (BApps). The industry
standard for manual web app testing.

### Installation (Kali)
```bash
sudo apt install burpsuite
# Community edition is free; Pro adds automated scanner + faster Intruder
```

### Setup
```
1. Start Burp → Proxy → Proxy settings → confirm 127.0.0.1:8080
2. Configure browser proxy to 127.0.0.1:8080 (or use FoxyProxy)
3. Import CA cert: visit http://burp while proxied → download & install cert
```

### Core Components

| Component | Purpose |
|-----------|---------|
| **Proxy** | Intercept and modify requests/responses between browser and server |
| **Repeater** | Manually replay and modify individual requests — ideal for testing auth bypass, IDOR, injection |
| **Intruder** | Automated fuzzing/brute-forcing (sniper, battering ram, pitchfork, cluster bomb attacks) |
| **Scanner** (Pro) | Automated vulnerability scanning (active + passive) |
| **Decoder** | Encode/decode/hash data (Base64, URL, hex, etc.) |
| **Comparer** | Diff two responses (great for auth-bypass testing) |
| **Extender** | Load BApp extensions (Logger++, Autorize, Turbo Intruder) |

### Common Techniques
```
# Intercept + modify a login request to test SQLi:
username: admin'-- -
password: anything

# Test IDOR: capture GET /api/orders/1001 → change to /1002

# Intruder cluster bomb: fuzz user:pass pairs with two payload sets

# Match/Replace rule: auto-inject cookies on every request
```

### Must-have extensions
- **Logger++** — log & grep all traffic
- **Autorize** — detect authorization flaws automatically
- **Turbo Intruder** — high-speed HTTP fuzzing
- **JWT Editor** — craft/tamper JWTs

**Resources:** [Burp Suite Docs](https://portswigger.net/burp/documentation) |
[Web Security Academy (free)](https://portswigger.net/web-security)

---

## OWASP ZAP — Web App Scanner

The world's most widely used free, open-source web app scanner. Great as a
proxy + automated scanner alternative to Burp, with strong CI/CD integration.

### Installation
```bash
sudo apt install zaproxy
# or download from https://www.zaproxy.org/download/
```

### Common Usage
```bash
# Launch GUI
zap-gui

# Quick scan from CLI
zap-cli quick-scan -s xss,sqli -r http://target.com

# Baseline scan (passive, safe for CI/CD) via Docker
docker run -t ghcr.io/zaproxy/zaproxy:stable zap-baseline.py -t https://target.com

# Full active scan via Docker
docker run -t ghcr.io/zaproxy/zaproxy:stable zap-full-scan.py -t https://target.com
```

### Core Components

| Component | Purpose |
|-----------|---------|
| **Quick Scan** | Automated scan of a single URL |
| **Spider** | Crawl the app discovering endpoints |
| **AJAX Spider** | Crawl JavaScript-heavy/SPA apps |
| **Active Scan** | Actively test discovered pages for vulns |
| **Fuzzer** | Fuzz parameters with payload lists |
| **Alerts** | Findings panel with OWASP risk ratings |

**Resources:** [ZAP Documentation](https://www.zaproxy.org/docs/)

---

## Sqlmap — SQL Injection & Database Takeover

Automatic SQL injection detection and exploitation: database enumeration,
data extraction, OS-level access, and out-of-band attacks.

### Installation
```bash
sudo apt install sqlmap
# or:
git clone --depth 1 https://github.com/sqlmapproject/sqlmap.git sqlmap
```

### Common Usage
```bash
# Test a GET parameter
sqlmap -u "http://target.com/item.php?id=1" --batch

# Test all parameters, enumerate databases
sqlmap -u "http://target.com/item.php?id=1" --dbs --batch

# Enumerate tables and dump data
sqlmap -u "http://target.com/item.php?id=1" -D sitedb --tables
sqlmap -u "http://target.com/item.php?id=1" -D sitedb -T users --dump

# Test POST data (captured from Burp)
sqlmap -r request.txt --dbs

# Use a specific injection technique
sqlmap -u "URL" --technique=BEUST   # B=Boolean, E=Error, U=Union, S=Stacked, T=Time

# Try OS shell via SQLi (file-write privileges required)
sqlmap -u "URL" --os-shell

# Tamper scripts for WAF bypass
sqlmap -u "URL" --tamper=space2comment,between --random-agent

# Use a cookie session
sqlmap -u "URL" --cookie="PHPSESSID=abc123" --level=3
```

### Key Flags

| Flag | Purpose |
|------|---------|
| `-u` / `-r` | Target URL / raw HTTP request file |
| `--dbs` / `-D` | Enumerate databases / select one |
| `--tables` / `-T` | Enumerate tables / select one |
| `--dump` / `--dump-all` | Extract data |
| `--batch` | Non-interactive (auto-accept defaults) |
| `--level` / `--risk` | Increase test coverage (1–5 / 1–3) |
| `--os-shell` | Attempt OS command execution |
| `--tamper` | WAF evasion payload transformations |
| `--threads` | Parallelize extraction |

**Resources:** [Sqlmap GitHub](https://github.com/sqlmapproject/sqlmap)

---

## Gobuster — Directory, URI & DNS Bruteforcer

Fast brute-forcing of URIs (directories/files), DNS subdomains, virtual hosts,
and cloud buckets using wordlists.

### Installation
```bash
sudo apt install gobuster
```

### Common Usage
```bash
# Directory bruteforce
gobuster dir -u http://target.com -w /usr/share/wordlists/dirb/common.txt

# With extensions + status codes
gobuster dir -u http://target.com -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt \
  -x php,html,txt,bak -o gobuster_dir.txt

# DNS subdomain enumeration
gobuster dns -d target.com -w /usr/share/wordlists/SecLists/DNS/subdomains-top1million-5000.txt

# VHOST discovery
gobuster vhost -u http://target.com -w /usr/share/wordlists/SecLists/Discovery/DNS/namelist.txt

# S3/GCS bucket enumeration
gobuster s3 -w bucket-names.txt
```

### Key Flags

| Flag | Purpose |
|------|---------|
| `dir` / `dns` / `vhost` / `s3` | Mode |
| `-u` / `-d` | Base URL / domain |
| `-w` | Wordlist |
| `-x` | File extensions to probe |
| `-t` | Threads (default 10) |
| `-b` | Blacklist status codes (e.g., `404,403`) |
| `-o` | Output file |
| `--no-tls-validation` | Skip TLS cert checks |

**Recommended wordlists:**
- `/usr/share/wordlists/dirb/common.txt` (installed with Kali)
- `/usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt`
- [SecLists](https://github.com/danielmiessler/SecLists) — install: `sudo apt install seclists`

**Resources:** [Gobuster GitHub](https://github.com/OJ/gobuster)

---

## 🧭 Web Application Testing Workflow

```
1. Recon (see 01): subdomains + ports → identify web services
2. Crawl:          Burp Spider / ZAP Spider → map app structure
3. Passive scan:   Burp/ZAP passive + Nuclei
4. Manual testing: Repeater — auth, IDOR, injection, business logic
5. Bruteforce:     Gobuster for hidden endpoints/backups
6. Exploit:        sqlmap for confirmed SQLi; escalate findings
7. Report:         screenshots + PoC + remediation per finding
```