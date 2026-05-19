---
hidden: true
---

# Network Scanning and Enumeration

**Methodology**

```
Scope definition → Host discovery → Port scanning → Service fingerprinting → Vulnerability mapping
```

**Host Discovery**

bash

```bash
# ARP scan — fastest on local segment, works even when ICMP blocked
arp-scan -l                         # local network
arp-scan -I eth0 192.168.1.0/24

# nmap ping sweep
nmap -sn 192.168.1.0/24             # ICMP + TCP ACK 80 + TCP SYN 443 + ICMP timestamp
nmap -sn -PE 192.168.1.0/24        # ICMP echo only
nmap -sn -PS22,80,443 192.168.1.0/24  # TCP SYN to specific ports
nmap -sn -PA80,443 192.168.1.0/24     # TCP ACK (bypasses some firewalls)
nmap -sn -PU53,161 192.168.1.0/24     # UDP probes

# netdiscover — passive ARP monitoring + active scanning
netdiscover -r 192.168.1.0/24
netdiscover -p   # passive only

# masscan — for large networks fast
masscan 10.0.0.0/8 -p 80,443 --rate 10000
```

**Port Scanning**

bash

```bash
# Default scan (top 1000 TCP ports)
nmap 192.168.1.10

# Full TCP port scan
nmap -p- 192.168.1.10

# SYN scan (requires root, doesn't complete handshake)
nmap -sS 192.168.1.10

# Connect scan (no root needed, more detectable)
nmap -sT 192.168.1.10

# UDP scan (slow but important)
nmap -sU 192.168.1.10
nmap -sU --top-ports 100 192.168.1.10

# Combine TCP and UDP
nmap -sS -sU -p T:1-1000,U:53,67,68,69,123,161 192.168.1.10
```

**Port scan types explained:**

| Type    | Flag  | Mechanics                                                            | Notes                                |
| ------- | ----- | -------------------------------------------------------------------- | ------------------------------------ |
| SYN     | `-sS` | SYN → (SYN-ACK = open, RST = closed)                                 | Fast, somewhat stealthy, needs root  |
| Connect | `-sT` | Full TCP handshake                                                   | No root needed, fully logged         |
| UDP     | `-sU` | UDP probe → (no response = open/filtered, ICMP unreachable = closed) | Slow, unreliable                     |
| FIN     | `-sF` | FIN → (no response = open, RST = closed)                             | Bypasses some packet filters         |
| NULL    | `-sN` | No flags → (no response = open, RST = closed)                        | Same as FIN                          |
| XMAS    | `-sX` | FIN+PSH+URG → (no response = open, RST = closed)                     | Same as FIN                          |
| ACK     | `-sA` | ACK → (RST = unfiltered, no response = filtered)                     | Maps firewall rules, not open/closed |

Note: FIN/NULL/XMAS scans only work against RFC-compliant stacks. Windows returns RST for all ports regardless of state — these scans are useless against Windows.

**Service and Version Detection**

bash

```bash
# Version detection
nmap -sV 192.168.1.10

# Aggressive (more probes, slower, noisier)
nmap -sV --version-intensity 9 192.168.1.10

# OS detection
nmap -O 192.168.1.10

# Everything at once
nmap -A 192.168.1.10   # version + OS + scripts + traceroute

# Specific NSE scripts
nmap --script smb-vuln-ms17-010 192.168.1.10
nmap --script http-title,http-headers 192.168.1.10
nmap --script ftp-anon,ftp-bounce 192.168.1.21
nmap --script dns-zone-transfer --script-args dns-zone-transfer.domain=target.com 192.168.1.53
```

**Evasion During Scanning**

bash

```bash
# Slow timing (T0 = paranoid, T5 = insane — default is T3)
nmap -T1 192.168.1.0/24

# Decoys — mix real scans with spoofed sources
nmap -D RND:10 192.168.1.10
nmap -D 192.168.1.100,192.168.1.101,ME 192.168.1.10

# Fragment packets
nmap -f 192.168.1.10           # 8-byte fragments
nmap -ff 192.168.1.10          # 16-byte fragments

# Spoof source port
nmap --source-port 53 192.168.1.10

# Randomize host order
nmap --randomize-hosts 192.168.1.0/24

# Slow down between probes
nmap --scan-delay 2s 192.168.1.10
nmap --max-rate 10 192.168.1.10   # max 10 packets per second

# Append random data to packets
nmap --data-length 25 192.168.1.10
```

**Banner Grabbing**

bash

```bash
# netcat
nc -nv 192.168.1.10 22
nc -nv 192.168.1.10 80

# HTTP banner via netcat
echo -e "GET / HTTP/1.0\r\n\r\n" | nc 192.168.1.10 80 | head -20

# telnet for old-school services
telnet 192.168.1.10 25

# curl for web services
curl -I http://192.168.1.10    # headers only
curl -v http://192.168.1.10   # verbose, shows request and response

# openssl for HTTPS/TLS
openssl s_client -connect 192.168.1.10:443
```

***

#### 1.13 Traffic Analysis

**Wireshark**

```
Capture filters (BPF syntax — applied at capture time):
  host 192.168.1.10           traffic to or from this host
  net 192.168.1.0/24          traffic within this subnet
  port 80                     traffic on this port
  tcp port 443                TCP on 443
  not arp and not icmp        exclude noise
  src host 10.0.0.5           traffic from this source only

Display filters (Wireshark syntax — applied after capture):
  ip.addr == 192.168.1.10
  tcp.port == 445
  http.request.method == "POST"
  dns.qry.name contains "target"
  frame contains "password"
  tcp.flags.syn == 1 && tcp.flags.ack == 0
  smb2
  ntlmssp
  kerberos
```

**Follow TCP stream** — right-click any packet → Follow → TCP Stream. Reconstructs the full conversation. Invaluable for reading HTTP, FTP, Telnet, SMTP in plaintext.

**Export objects** — File → Export Objects → HTTP. Extracts files transferred over HTTP from a capture.

**tcpdump**

bash

```bash
# Capture all traffic
tcpdump -i eth0

# Write to file
tcpdump -i eth0 -w capture.pcap

# Read file
tcpdump -r capture.pcap

# Don't resolve hostnames or ports (faster, cleaner)
tcpdump -i eth0 -nn

# Specific host
tcpdump -i eth0 host 192.168.1.10

# Specific port
tcpdump -i eth0 port 445

# Show ASCII payload
tcpdump -i eth0 -A port 80 | grep -E "(password|pass|user|login|auth)"

# Capture ICMP only
tcpdump -i eth0 icmp

# Complex filter: SMB traffic not from our host
tcpdump -i eth0 'port 445 and not src host 192.168.1.5'

# Multiple hosts
tcpdump -i eth0 'host 192.168.1.10 or host 192.168.1.20'
```

**What to Look For**

| Observation                         | Interpretation                                     |
| ----------------------------------- | -------------------------------------------------- |
| Credentials in cleartext            | FTP, Telnet, HTTP Basic, LDAP, old SQL connections |
| NTLM auth hashes                    | Relay or crack offline with hashcat                |
| Kerberos TGS tickets                | Extract for offline cracking (Kerberoasting)       |
| SMB auth to unknown hosts           | Lateral movement                                   |
| Large or high-frequency DNS queries | Possible tunneling or exfil                        |
| ICMP with large/non-zero payloads   | Possible tunneling                                 |
| HTTP to RFC 1918 addresses          | Internal service discovery                         |
| Certificates with internal CN/SAN   | Map internal infrastructure                        |
| Regular outbound intervals          | Beaconing C2                                       |
