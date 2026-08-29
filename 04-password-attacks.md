# 🔑 Password Attacks

Tools for password auditing, brute-forcing, and hash cracking.

## Contents

- [Hashcat](#hashcat--gpu-password-recovery)
- [John the Ripper](#john-the-ripper--fast-password-cracker)
- [Hydra](#hydra--parallel-login-cracker)
- [Cracking Strategy](#-cracking-strategy)

---

## Hashcat — GPU Password Recovery

The world's fastest and most advanced password recovery utility, leveraging
GPU acceleration and rule-based mutation of wordlists.

### Installation
```bash
sudo apt install hashcat
# Verify GPU + OpenCL/CUDA working:
hashcat -I
```

### Common Usage
```bash
# Identify a hash type
hashcat --example-hashes | grep -i bcrypt
# or use: name-that-hash / hashid

# MD5 dictionary attack
hashcat -m 0 -a 0 hashes.txt /usr/share/wordlists/rockyou.txt

# SHA256 with rules (mutations like capitalization, leetspeak)
hashcat -m 1400 -a 0 hashes.txt rockyou.txt -r /usr/share/hashcat/rules/best64.rule

# Mask attack: 8-char password, digits (e.g., years)
hashcat -m 0 -a 3 hashes.txt ?d?d?d?d?d?d?d?d

# Combination attack: two wordlists
hashcat -m 0 -a 1 hashes.txt wordlist1.txt wordlist2.txt

# Show cracked passwords when done
hashcat -m 0 hashes.txt --show

# Benchmark your GPU
hashcat -b
```

### Common Hash Modes

| Mode | Hash Type |
|------|-----------|
| `-m 0` | MD5 |
| `-m 100` | SHA1 |
| `-m 1400` | SHA256 |
| `-m 1800` | sha512crypt ($6$) — Linux /etc/shadow |
| `-m 1000` | NTLM (Windows) |
| `-m 3200` | bcrypt ($2*$) |
| `-m 22000` | WPA-PBKDF2-PMKID+EAPOL (WiFi) |
| `-m 18200` | Kerberos AS-REQ (Kerberoasting) |

### Attack Modes

| Mode | Attack |
|------|--------|
| `-a 0` | Dictionary/straight |
| `-a 1` | Combination |
| `-a 3` | Mask/brute-force |
| `-a 6` | Hybrid dict + mask |
| `-a 7` | Hybrid mask + dict |

### Mask Placeholders
```
?l = lowercase   ?u = uppercase   ?d = digits   ?s = symbols
?a = all         ?h = hex         ?1-?4 = custom charsets (-1 ?l?d)
```

**Resources:** [Hashcat Wiki](https://hashcat.net/wiki/) | [Example Hashes](https://hashcat.net/wiki/doku.php?id=example_hashes)

---

## John the Ripper — Fast Password Cracker

CPU-based password cracker with automatic hash-format detection, rich format
support, and utils like `unshadow` and `john2cc` conversions.

### Installation
```bash
sudo apt install john
```

### Common Usage
```bash
# Crack a Linux /etc/shadow (combine passwd + shadow first)
unshadow passwd.txt shadow.txt > unshadowed.txt
john unshadowed.txt

# Crack with a specific wordlist
john --wordlist=/usr/share/wordlists/rockyou.txt hashes.txt

# Show cracked results
john --show hashes.txt

# Auto-detect format + incremental (brute force) mode
john --incremental hashes.txt

# Crack a ZIP file
zip2john protected.zip > zip_hash.txt
john --wordlist=rockyou.txt zip_hash.txt

# Kerberos / Office / PDF formats
john krb5tgs.txt --wordlist=rockyou.txt
office2john document.docx > office_hash.txt
john office_hash.txt

# Use rules for mutations
john --wordlist=rockyou.txt --rules=Best64 hashes.txt
```

### Key Flags

| Flag | Purpose |
|------|---------|
| `--wordlist=` | Dictionary attack |
| `--rules` | Apply mutation rules |
| `--format=` | Force hash format (e.g., `Raw-MD5`, `bcrypt`) |
| `--incremental` | Pure brute force |
| `--show` | Display cracked passwords |
| `--users=` / `--groups=` | Filter targets |
| `--restore` | Resume an interrupted session |

**Resources:** [OpenWall John Docs](https://www.openwall.com/john/doc/)

---

## Hydra — Parallel Login Cracker

Network login cracker supporting 50+ protocols: SSH, FTP, HTTP forms, RDP,
SMB, MySQL, SMTP, and more.

### Installation
```bash
sudo apt install hydra
```

### Common Usage
```bash
# SSH brute force
hydra -l admin -P /usr/share/wordlists/rockyou.txt ssh://10.10.10.5

# Username list + password list on FTP
hydra -L users.txt -P pass.txt ftp://10.10.10.5

# HTTP POST form login (analyse the form first!)
hydra -l admin -P rockyou.txt 10.10.10.5 http-post-form \
  "/login:user=^USER^&pass=^PASS^:F=incorrect" -t 16

# HTTP Basic Auth
hydra -L users.txt -P pass.txt 10.10.10.5 http-get /admin/

# SMB login
hydra -L users.txt -P pass.txt 10.10.10.5 smb

# MySQL / RDP
hydra -l root -P pass.txt 10.10.10.5 mysql
hydra -l administrator -P pass.txt 10.10.10.5 rdp

# Verbose + stop on first hit
hydra -l admin -P pass.txt -V -f ssh://10.10.10.5
```

### Key Flags

| Flag | Purpose |
|------|---------|
| `-l` / `-L` | Single username / username list |
| `-p` / `-P` | Single password / password list |
| `-t` | Threads (parallel tasks) |
| `-f` | Stop when a valid login is found |
| `-V` | Verbose — show each attempt |
| `-s` | Custom port |
| `F=string` | Failure condition (form login) |
| `S=string` | Success condition |

**Tip:** For HTTP forms, capture a login request (Burp), then set
`^USER^`/`^PASS^` placeholders and the failure string (e.g., `F=Invalid`).

**Resources:** [THC-Hydra GitHub](https://github.com/vanhauser-thc/thc-hydra)

---

## 🧭 Cracking Strategy

```
1. Harvest hashes (secretsdump, hashcat -m, /etc/shadow, DPAPI, AS-REP roast)
2. Identify format: hashid / name-that-hash / hashcat --example-hashes
3. Order attacks cheapest→costliest:
   a. Dictionary (rockyou)
   b. Dictionary + best64 rule
   c. Combinator / hybrid mask
   d. Mask attack (e.g., CompanyName?d?d?d?d)
   e. Rule-heavy (dive.rule) — last resort on GPU
4. For network logins: hydra — but only after rate-limit/lockout policy checks
5. Reuse cracked creds across services (password reuse is common)
```