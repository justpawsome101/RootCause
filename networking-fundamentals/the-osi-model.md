---
description: >-
  I love this section in networking because of one thing: PLEASE DO NOT THROW
  SAUASAGE PIZZA AWAY. (Easy way to remember the model from bottom to top💅).
---

# The OSI Model

The OSI (Open Systems Interconnection) model is a conceptual framework that standardizes how different network functions are layered. It has seven layers, each with a specific job.

```
Layer 7 — Application    HTTP, DNS, FTP, SMTP, SSH
Layer 6 — Presentation   Encryption (TLS), encoding, compression
Layer 5 — Session        Session management, NetBIOS, RPC
Layer 4 — Transport      TCP, UDP; ports, flow control, reliability
Layer 3 — Network        IP, ICMP; logical addressing, routing
Layer 2 — Data Link      Ethernet, ARP; MAC addressing, local delivery
Layer 1 — Physical       Cables, radio, voltages, bits on wire
```

**What Each Layer Does**

_**Note for terminology used (because I didn't know there were even different terms 💀)**_

* _**Frame** = Layer 2 unit of data (Ethernet, uses MACs)_
* _**Packet** = Layer 3 unit of data (IP, uses IPs)_
* _**Segment** = Layer 4 unit of data (TCP/UDP)_

**Layer 1 Physical:** Deals with the actual transmission of raw bits over a physical medium. Cables, fiber optics, radio frequencies, voltage levels. A network interface card (NIC) and the cable plugged into it operate here.

**Layer 2 Data Link:** Handles communication between devices on the same local network segment. Uses **MAC addresses** (Media Access Control) to identify devices. Ethernet is the dominant Layer 2 protocol. Switches operate here, they forward frames based on MAC address tables.

**Layer 3 Network:** Handles logical addressing and routing between different networks. Uses **IP addresses**. Routers operate here, they forward packets based on routing tables. ICMP (ping, traceroute) also lives here.

**Layer 4 Transport:** Handles end-to-end communication between applications. **TCP** provides reliable, ordered, connection-oriented delivery. **UDP** provides fast, connectionless, best-effort delivery. Introduces the concept of **ports,** allowing multiple services to run on the same host.

**Layer 5  Session:** Manages sessions (conversations) between applications. In practice, this layer is often handled within the application itself.&#x20;

**Layer 6 Presentation:** Handles data formatting, translation, and encryption. TLS operates here, it encrypts the payload before handing it to the transport layer. Character encoding (ASCII, UTF-8) and compression also belong here.

**Layer 7  Application:** The layer closest to the end user. The actual protocols applications use to communicate; HTTP, DNS, SMTP, FTP, SSH, etc. This is where most modern attacks live.

**Why It Matters Offensively**

Every attack class maps to a layer. Understanding which layer your attack operates at tells you what defensive controls are and aren't in the way. For example:

| Layer | Attack Examples                                    | Common Defenses                         |
| ----- | -------------------------------------------------- | --------------------------------------- |
| 7     | SQL injection, XSS, command injection, auth bypass | input validation, application hardening |
| 6     | SSL stripping, certificate spoofing                | HSTS, certificate pinning               |
| 5     | Session hijacking                                  | Token expiration, re-authentication     |
| 4     | Port scanning, SYN floods, session spoofing        | Stateful firewall, rate limiting        |
| 3     | IP spoofing, ICMP tunneling, routing manipulation  | ACLs, BCP38, uRPF                       |
| 2     | ARP spoofing, VLAN hopping, MAC flooding           | DAI, port security, 802.1X              |
| 1     | Physical access, hardware implants, cable taps     | Physical security controls              |
