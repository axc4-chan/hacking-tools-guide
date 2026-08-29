---
tags:
  - redteam
  - recon
  - nmap
  - enumeration
created: 2026-08-29
status: complete
---

# 🗺️ Nmap — Enumeration Guide

> [!abstract] TL;DR
> Host discovery → port scan → service/version → scripts. Speed matters:
> start broad (`-sS -p- --min-rate`), then deep-dive interesting ports.

---

## 1. Host Discovery

```bash
nmap -sn 10.10.20.0/24                       # ping sweep, no port scan
nmap -sn -PR 10.10.20.0/24                   # ARP (local net, most reliable)
nmap -sP -PS443,80 -PU -PE 10.10.20.0/24     # mixed probes for filter-y hosts
nmap -sn --disable-arp-ping -PS80,443,22 10.10.20.0/24   # when ARP is blocked
```

| Flag | Meaning |
|---|---|
| `-sn` | host discovery only |
| `-PE / -PP / -PM` | ICMP echo/timestamp/netmask |
| `-PS / -PA / -PU / -PY` | TCP SYN / TCP ACK / UDP / SCTP ping |
| `-PR` | ARP ping (local segment) |
| `-n` | never resolve DNS (speed) |

---

## 2. Port Scans

```bash
nmap -sS -p- --min-rate 10000 -oA allports 10.10.20.10    # fast full TCP sweep (SYN, needs root)
nmap -sT -p- 10.10.20.10                                  # no root / unprivileged
nmap -sU --top-ports 50 10.10.20.10                       # UDP — slow, always rate-limit
nmap -sS -sV -p 22,80,443,445 -sC -A 10.10.20.10          # deep dive on found ports
```

| Flag | Meaning |
|---|---|
| `-sS` | SYN (half-open) — default root scan, stealthier |
| `-sT` | full connect — fallback, generates logs |
| `-sU` | UDP (pair with `--top-ports N`, add `-sV` for good version info) |
| `-p-` | all 65535 ports |
| `-p-` + `--min-rate` | fast SYN sweep |
| `--open` | show only open ports |
| `-sV` | service/version detection |
| `-sC` | default NSE scripts (safe ones) |
| `-A` | aggressive: `-sV -sC -O --traceroute` (loud) |
| `-O` | OS detection (needs 1 open + 1 closed port) |
| `-6` | IPv6 |

> [!tip] The two-phase pattern
> ```bash
> # Phase 1 — fast, find open ports:
> nmap -sS -p- --min-rate 10000 -oA fast 10.10.20.10
> # Phase 2 — thorough on just the open ones:
> nmap -sCV -p <open,ports> -oA deep 10.10.20.10
> ```
> UDP phase: `nmap -sU --top-ports 100 -sV --open 10.10.20.10`

---

## 3. Timing & Evasion

| Flag | Effect |
|---|---|
| `-T0`–`-T5` | paranoid(5min/probe) → insane(0.3s) — use `-T4` normally, `-T2+` for fragile hosts |
| `--max-rate` / `--min-rate` | hard bandwidth caps |
| `--max-retries` | lower = faster, more false negative risk |
| `-f` | fragment packets (weak vs modern IPS) |
| `--data-length <n>` | pad packets to break signature-based rules |
| `--source-port 53` | spoof source port (sometimes slips badly-configured filters) |
| `-D decoy1,decoy2,ME` | add decoy sources (noisy, mostly old gear) |
| `-S <ip>` / `-e <iface>` | spoof source address (no replies back — useful for log pollution only) |
| `--proxies` | chain through proxies |

Firewall/host IDS sensitivity: SYN scan + `-T3` + real source IP is the
baseline; anything flashier trades reliability for evasion.

---

## 4. NSE Scripts

```bash
# discovery
ls /usr/share/nmap/scripts/ | grep <service>
nmap --script-help http-enum

# categories: auth, broadcast, default, discovery, dos, exploit,
#             external, fuzzer, intrusive, malware, safe, version, vuln

nmap --script vuln 10.10.20.10                        # all vuln-category scripts
nmap --script smb-vuln* -p445 10.10.20.10             # MS17-010 etc.
nmap --script http-enum,http-headers -p80,443 10.10.20.10
nmap --script mysql-empty-password -p3306 10.10.20.10
nmap --script ftp-anon -p21 10.10.20.10
nmap --script smtp-enum-users -p25 10.10.20.10
nmap --script dns-zone-transfer --script-args dns-zone-transfer.domain=corp.local -p53 ns.corp.local
```

Script args:

```bash
nmap --script http-brute --script-args http-brute.path=/admin -p80 target
```

---

## 5. Output

```bash
-oA base                # all formats: base.nmap, base.gnmap, base.xml
-oN / -oX / -oG         # human / xml / greppable
--stats-every 10s       # progress while long scans run
```

Greppable trick:

```bash
grep "open" fast.gnmap | awk '{print $2, $4}'
```

---

## 6. Service-Specific Quickies

```bash
nmap -p445 --script smb-enum-shares,smb-enum-users,smb-os-discovery 10.10.20.10
nmap -p139,445 --script-args smbusername=svc,smbpassword=pass --script smb-enum-shares 10.10.20.10
nmap -p389,636 --script ldap-rootdse 10.10.20.10
nmap -p3389 --script rdp-ntlm-info 10.10.20.10
nmap -p161 --script snmp-info,snmp-interfaces -sU 10.10.20.10
nmap --script broadcast-dhcp-discover -e eth0
nmap --script nbstat,dsquery -sU -p137 10.10.20.10
```

---

## 7. Troubleshooting

> [!warning] Common issues
> - **"Host seems down"** → firewalled ICMP/ARP; try `-Pn` (assume up, scan anyway) + `-PS`/`-PA` pings
> - **All ports "filtered"** → egress/host firewall on YOUR side, or inline IPS; try `-sT -Pn --source-port 53`
> - **UDP shows `open|filtered`** → normal; add service-specific probe (`-sV`, NSE for snmp/dns)
> - **Version detection too slow** → `--version-intensity 0` (light) or `--version-light`
> - **Scans from VPN fail** → verify `tun0` routes, use `nmap -e tun0`

Related: [[Metasploit Framework]] · [[Aircrack-ng Guide]] · [[msfvenom — Payload Generation Guide]]