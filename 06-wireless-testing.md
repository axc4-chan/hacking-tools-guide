# 📡 Wireless Testing

Tools for assessing the security of wireless networks.

## Contents

- [Aircrack-ng](#aircrack-ng--wifi-security-suite)
- [Kismet](#kismet--wireless-detection--ids)
- [Workflow](#-wireless-assessment-workflow)

> **⚠️ Note:** Wireless testing requires monitor-mode-capable adapters
> (e.g., Alfa AWUS036ACH, Panda PAU09). Only test networks you own or are
> explicitly authorized to assess.

---

## Aircrack-ng — WiFi Security Suite

Complete suite to capture, inject, crack, and test WiFi networks:
WEP, WPA/WPA2-PSK, WPA3 (via downgrade/PMKID), and more.

### Installation
```bash
sudo apt install aircrack-ng
```

### Core Tools

| Tool | Purpose |
|------|---------|
| `airmon-ng` | Enable monitor mode |
| `airodump-ng` | Capture packets, discover networks/clients |
| `aireplay-ng` | Packet injection (deauth, replay) |
| `aircrack-ng` | Crack captured handshakes |
| `airbase-ng` | Fake access points |

### Common Workflow — WPA/WPA2-PSK

```bash
# 1. Kill interfering processes, enable monitor mode
sudo airmon-ng check kill
sudo airmon-ng start wlan0
# → creates wlan0mon

# 2. Discover nearby networks
sudo airodump-ng wlan0mon

# 3. Capture a specific AP (note BSSID + channel), save handshakes
sudo airodump-ng -c 6 --bssid AA:BB:CC:DD:EE:FF -w capture wlan0mon

# 4. Deauth a client to force a reconnection (captures new handshake)
sudo aireplay-ng -0 2 -a AA:BB:CC:DD:EE:FF -c CLIENT_MAC wlan0mon

# 5. Crack the handshake (dictionary attack)
aircrack-ng -w /usr/share/wordlists/rockyou.txt -b AA:BB:CC:DD:EE:FF capture-01.cap

# 6. Stop monitor mode when done
sudo airmon-ng stop wlan0mon
```

### PMKID attack (clientless — no deauth needed)
```bash
sudo hcxdumptool -i wlan0mon -o pmkid.pcap --enable_status=1
hcxpcapngtool -o hash.hc22000 pmkid.pcap
hashcat -m 22000 hash.hc22000 /usr/share/wordlists/rockyou.txt
```

### WEP (legacy)
```bash
sudo airodump-ng -c 6 --bssid AA:BB:CC:DD:EE:FF -w wep_cap wlan0mon
sudo aireplay-ng -3 -b AA:BB:CC:DD:EE:FF wlan0mon    # ARP replay
aircrack-ng -b AA:BB:CC:DD:EE:FF wep_cap-01.cap
```

**Resources:** [Aircrack-ng Docs](https://www.aircrack-ng.org/doku.php)

---

## Kismet — Wireless Detector, Sniffer & IDS

Passive wireless network detector, packet sniffer, and intrusion detection
system for WiFi, Bluetooth, SDR, and more. Great for RF survey and rogue AP
detection.

### Installation
```bash
sudo apt install kismet
# edit /etc/kismet/kismet.conf (source interface + logging)
# default web UI: http://localhost:2501
```

### Common Usage
```bash
# Start with a capture source
sudo kismet -c wlan0

# Headless server (then browse http://localhost:2501)
sudo kismet -c wlan0 --no-ncurses

# Configuration: /etc/kismet/kismet_site.conf
#   ncsource=wlan0
#   logdir=/tmp/kismet/
```

### Key Capabilities

- Detects hidden networks, rogue APs, and spoofed/deauth attacks
- Channel hopping across 2.4 & 5 GHz
- Live device tracking (clients, APs, Bluetooth, BTLE)
- Alerts (IDS): deauth floods, AP impersonation, new devices
- Logging: pcapng, kismetdb, JSON — exportable for reporting
- REST API for automation (`/devices/summary.json` etc.)

**Resources:** [Kismet Docs](https://www.kismetwireless.net/docs/)

---

## 🧭 Wireless Assessment Workflow

```
1. Survey:         kismet / airodump-ng → map SSIDs, channels, clients
2. Identify:       open, WEP, WPA2-PSK, WPA2-Enterprise, WPA3
3. Capture:        handshakes (deauth) or PMKID
4. Crack:          aircrack-ng / hashcat -m 22000
5. Rogue/evil-twin analysis, misconfig review (weak PSKs, WPS, old TKIP)
6. Report:         SSID, auth type, findings, recommendations
```