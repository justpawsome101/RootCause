---
hidden: true
---

# IP Addressing and Subnetting

**IPv4 Structure**

An IPv4 address is a 32-bit number written as four decimal octets separated by dots: `192.168.1.10`. Each octet ranges from 0–255 (`00000000` to `11111111` in binary).

Every IP address on a network has two parts:

* **Network portion** — identifies which network the address belongs to
* **Host portion** — identifies the specific device within that network

The **subnet mask** defines where the network portion ends and the host portion begins. Written as `255.255.255.0` (dotted decimal) or `/24` (CIDR prefix length).

```
IP:   192.168.1.10     → 11000000.10101000.00000001.00001010
Mask: 255.255.255.0    → 11111111.11111111.11111111.00000000
                                                    ^^^^^^^^
                                              Host portion (8 bits)
```

**CIDR Notation**

CIDR (Classless Inter-Domain Routing) expresses the subnet mask as a prefix length appended to the IP: `192.168.1.0/24`. The `/24` means the first 24 bits are the network — the same as a `255.255.255.0` mask.

**Key formula:** `2^(32 - prefix) - 2 = usable hosts`

| CIDR | Subnet Mask     | Total IPs  | Usable Hosts   |
| ---- | --------------- | ---------- | -------------- |
| /8   | 255.0.0.0       | 16,777,216 | 16,777,214     |
| /16  | 255.255.0.0     | 65,536     | 65,534         |
| /24  | 255.255.255.0   | 256        | 254            |
| /25  | 255.255.255.128 | 128        | 126            |
| /26  | 255.255.255.192 | 64         | 62             |
| /27  | 255.255.255.224 | 32         | 30             |
| /28  | 255.255.255.240 | 16         | 14             |
| /30  | 255.255.255.252 | 4          | 2              |
| /32  | 255.255.255.255 | 1          | 1 (host route) |

The two addresses you subtract are the **network address** (all host bits zero) and the **broadcast address** (all host bits one).

**Special and Private Ranges**

**Private (RFC 1918)** — not routable on the public internet. Used inside organizations. Almost always what you encounter inside a target network:

| Range                         | CIDR           | Common Use                     |
| ----------------------------- | -------------- | ------------------------------ |
| 10.0.0.0 – 10.255.255.255     | 10.0.0.0/8     | Large enterprise, data centers |
| 172.16.0.0 – 172.31.255.255   | 172.16.0.0/12  | Medium enterprise              |
| 192.168.0.0 – 192.168.255.255 | 192.168.0.0/16 | SOHO, home networks            |

**Other important ranges:**

| Address/Range           | Purpose                        | Red team notes                                                  |
| ----------------------- | ------------------------------ | --------------------------------------------------------------- |
| 127.0.0.1 / 127.0.0.0/8 | Loopback                       | Services binding only to localhost — port forward to reach them |
| 169.254.0.0/16          | APIPA / Link-local             | DHCP failure indicator; may reveal dual-homed hosts             |
| 0.0.0.0                 | All interfaces / default route | Services binding to 0.0.0.0 accept connections on any interface |
| 255.255.255.255         | Limited broadcast              | DHCP discovery; doesn't cross routers                           |
| 100.64.0.0/10           | Shared address space           | ISP carrier-grade NAT; rarely seen internally                   |

**Loopback** is especially relevant — services configured to bind only to `127.0.0.1` are "protected" by not being externally accessible. But if you're on that machine, they're fully reachable. And if you can establish a port forward, you expose them remotely.

**Network and Broadcast Addresses**

For any subnet:

* **Network address** — first address (all host bits zero). Not assignable to a host.
* **Broadcast address** — last address (all host bits one). Packets sent here go to every host on the subnet.

For `192.168.1.0/24`:

* Network: `192.168.1.0`
* Usable range: `192.168.1.1` – `192.168.1.254`
* Broadcast: `192.168.1.255`

**IPv6**

IPv6 uses 128-bit addresses written in eight groups of four hexadecimal digits, separated by colons: `2001:0db8:85a3:0000:0000:8a2e:0370:7334`

Consecutive groups of zeros can be collapsed with `::` (once per address): `2001:db8:85a3::8a2e:370:7334`

**Key IPv6 address types:**

| Type           | Range       | Equivalent to          |
| -------------- | ----------- | ---------------------- |
| Loopback       | `::1`       | 127.0.0.1              |
| Link-local     | `fe80::/10` | APIPA (169.254.x.x)    |
| Unique local   | `fc00::/7`  | RFC 1918 private space |
| Global unicast | `2000::/3`  | Public IPv4            |
| Multicast      | `ff00::/8`  | 224.0.0.0/4            |

**Red team relevance:** IPv6 is often enabled by default and poorly monitored. Many firewall rules, IDS signatures, and monitoring tools are IPv4-only. IPv6 traffic may flow through completely unmonitored paths. Always check for IPv6 when you land on a host:

bash

```bash
ip -6 addr
ip -6 route
ping6 fe80::1%eth0    # ping link-local gateway
nmap -6 -sn fe80::/10  # IPv6 host discovery (local segment)
```
