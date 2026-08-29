# 🔍 Information Gathering

Tools used for reconnaissance, footprinting, and gathering intelligence about a target.

## Contents

- [Nmap](#nmap--network-mapper)
- [Recon-ng](#recon-ng--web-reconnaissance-framework)
- [TheHarvester](#theharvester--osint-e-mail--subdomain-harvester)
- [Amass](#amass--attack-surface-mapping)
- [Workflow](#-information-gathering-workflow)

---

## Nmap — Network Mapper

The de facto standard for network discovery, port scanning, service
fingerprinting, and OS detection. Uses raw packets to determine hosts, ports,
services, and firewall rules on a network.

### Installation (Kali/Debian)
```bash
sudo apt install nmap
```

### Common Usage
```bash
# Host discovery (ping sweep) on a subnet
nmap -sn 192.168.1.0/24

# Fast top-1000 TCP port scan
nmap 10.10.10.5

# Full TCP port scan with service + OS + version detection
sudo nmap -sS -sV -O -p- -T4 -A 10.10.10.5

# Scan specific ports with NSE scripts
sudo nmap -sV -p 80,443,8080 --script "http-*" 10.10.10.5

# UDP scan (top 100 UDP ports)
sudo nmap -sU --top-ports 100 10.10.10.5

# Evade basic firewalls: fragment packets, decoy scan
sudo nmap -f -D RND:5 -sS 10.10.10.5

# Save output in all formats (grepable + normal + XML)
nmap -sV -oA scan_results 10.10.10.5
```

### Key Flags

| Flag | Purpose |
|------|---------|
| `-sS` | SYN (half-open/stealth) scan — requires root |
| `-sV` | Version detection of services |
| `-O` | OS fingerprinting |
| `-p-` | Scan all 65,535 ports |
| `-A` | Aggressive (OS + version + scripts + traceroute) |
| `--script=` | Run NSE scripts (e.g., `vuln`, `default`, `smb-vuln*`) |
| `-oN/-oX/-oG` | Output normal / XML / grepable |
| `--rate` | Adjust packet rate for speed tuning |

### Useful NSE scripts
```bash
nmap --script vuln 10.10.10.5              # vulnerability category
nmap --script smb-vuln-ms17-010 10.10.10.5 # EternalBlue check
nmap -sV --script default 10.10.10.5       # safe defaults
```

**Resources:** [Nmap Official Docs](https://nmap.org/book/man.html) | `nmap --help`

---

## Recon-ng — Web Reconnaissance Framework

A full-featured, Metasploit-style framework for web-based OSINT. Modular,
database-driven, with marketplace-installed modules.

### Installation
```bash
sudo apt install recon-ng
# or from source:
git clone https://github.com/lanmaster53/recon-ng.git
cd recon-ng && pip install -r REQUIREMENTS && ./recon-ng
```

### Basic Workflow
```
recon-ng
┌──> marketplace install all          # install modules
├──> workspaces create targetcorp     # create a workspace
├──> db insert companies              # add seed data
├──> modules search domains
├──> modules load recon/domains-hosts/hackertarget
├──> options set SOURCE target.com
├──> run
└──> dashboard / show hosts           # view collected data
```

### Key Commands

| Command | Purpose |
|---------|---------|
| `marketplace search <keyword>` | Find modules |
| `marketplace install <module>` | Install a module |
| `modules load <module>` | Load a module |
| `options set SOURCE <value>` | Configure the module input |
| `run` | Execute the loaded module |
| `show hosts / contacts / domains` | Query the workspace database |
| `snapshot` / `reporting` | Generate an HTML report (`modules load reporting/html`) |

### Popular modules
- `recon/domains-hosts/google_site_web` — subdomains via Google dorking
- `recon/domains-contacts/hunter_io` — email discovery (needs API key)
- `recon/companies-multi/shodan_org` — Shodan asset discovery
- `recon/hosts-hosts/resolve` — resolve hosts to IPs

**Resources:** [Recon-ng Wiki](https://github.com/lanmaster53/recon-ng/wiki)

---

## TheHarvester — OSINT E-mail & Subdomain Harvester

Gathers e-mails, subdomains, names, URLs, and IPs from public sources
(search engines, certificate transparency, threat intel APIs).

### Installation
```bash
sudo apt install theharvester
# or:
pip install theHarvester
```

### Common Usage
```bash
# Basic enumeration via multiple engines
theHarvester -d example.com -b all

# Subdomains via crt.sh (certificate transparency)
theHarvester -d example.com -b crtsh

# Limit results and output to file
theHarvester -d example.com -b bing -l 500 -f results.html

# Specific data gathering
theHarvester -d example.com -b dnsdumpster
```

### Key Flags

| Flag | Purpose |
|------|---------|
| `-d` | Domain to search |
| `-b` | Data source (`all`, `bing`, `crtsh`, `dnsdumpster`, `github`, `linkedin`, etc.) |
| `-l` | Limit number of results |
| `-s` | Start result number |
| `-f` | Save output (HTML/XML) |
| `-v` | Verify hostnames via DNS resolution |

**Resources:** [GitHub — laramies/theHarvester](https://github.com/laramies/theHarvester)

---

## Amass — Attack Surface Mapping

OWASP Amass performs in-depth network mapping of attack surfaces and external
asset discovery through DNS enumeration, scraping archives, APIs, and
certificate transparency.

### Installation
```bash
sudo apt install amass
# or from source:
go install -v github.com/owasp-amass/amass/v4/...@master
```

### Common Usage
```bash
# Passive subdomain discovery (no packets sent to target)
amass enum -passive -d example.com

# Active enumeration with DNS resolution
amass enum -active -d example.com -p 80,443,8080

# Combine multiple data sources + save results
amass enum -d example.com -o amass_results.txt

# Full intel mode — find related assets/ASN info
amass intel -d example.com -whois

# Track changes over time (find NEW subdomains since last run)
amass enum -df old_subs.txt -nf new_subs.txt
```

### Key Flags

| Flag | Purpose |
|------|---------|
| `enum` | Perform DNS enumeration and network mapping |
| `intel` | Collect open-source intel (ASNs, WHOIS, related orgs) |
| `-passive` | Never touch the target directly |
| `-active` | Cert probing + DNS resolution against targets |
| `-d` | Target domain |
| `-brute` | DNS brute-forcing |
| `-o` / `-json` | Output file / structured output |

**Resources:** [OWASP Amass Docs](https://owasp.org/www-project-amass/)

---

## 🧭 Information Gathering Workflow

```
1. Passive OSINT:      theHarvester + Amass (passive) + Recon-ng
2. Active DNS:         amass enum -active
3. Network mapping:    nmap -sn (hosts) → nmap -sS -sV -p- (services)
4. Correlate:          import results into a workspace / report
5. Feed findings into: Vulnerability Analysis phase → 02-vulnerability-analysis.md
```