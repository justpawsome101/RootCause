# Firewalls, NAT, and Filtering

**Firewall Types**

**Packet filter (stateless):** Evaluates each packet independently based on source/destination IP, port, protocol, and direction. No memory of previous packets. Bypassed by fragmentation, using allowed ports, and source port manipulation.

**Stateful inspection:** Tracks connection state. It knows an inbound ACK is a legitimate reply to an outbound SYN rather than an unsolicited probe. Significantly better than stateless but still operates at Layer 3–4.

**Application layer / Next-Gen Firewall (NGFW):** Inspects up to Layer 7. Can identify protocols by behavior regardless of port (detecting HTTP on port 4444), detect tunneling, and apply policy per application. JA3 fingerprinting, SSL inspection, and user identity integration. Harder to bypass but misconfiguration is common.

**Web Application Firewall (WAF):** Specifically inspects HTTP/HTTPS. Blocks SQLi, XSS, path traversal, and other web attack patterns. Bypass techniques: encoding variations (`%27` instead of `'`), case manipulation, whitespace injection (`UN/**/ION`), HTTP parameter pollution, and using uncommon HTTP methods.

**Firewall Rule Logic**

Rules are evaluated top to bottom. First match wins. Most firewalls end with an implicit **deny all**.

```
Rule 1: Allow TCP from any to 10.0.0.10 on port 443
Rule 2: Allow TCP from 10.0.0.0/24 to any on port 80
Rule 3: Allow established/related sessions
Rule 4: Deny all
```

Common misconfigurations: overly broad rules (`any` source or destination), rules allowing entire RFC 1918 space when only specific hosts are needed, forgotten test rules left open, no egress filtering.

**NAT(Network Address Translation)**

NAT maps internal private addresses to one or more public addresses. The most common form is PAT (Port Address Translation) or "NAT overload"which is when thousands of internal hosts share a single public IP, differentiated by their source port mappings.

```
Internal: 192.168.1.10:54231 → Internet: 1.2.3.4:54231
Internal: 192.168.1.11:62045 → Internet: 1.2.3.4:62045
```

NAT maintains a translation table to match return traffic.

**Implications:**

* Hosts behind NAT can't be directly reached from outside without port forwarding
* Once inside a network, NAT is largely irrelevant, you're on the trusted side
* For C2, reverse shells and beaconing work fine (outbound connections from victim to attacker)
* Bind shells (attacker connects to victim) require port forwarding or other techniques
* Double-NAT complicates reverse shell callbacks — if your attacker machine is also NATted, you need a VPS or external server to catch callbacks

**Egress Filtering**

Many organizations filter outbound traffic, blocking non-standard ports, restricting DNS to internal resolvers, preventing direct internet access without a proxy. This is what you're working around when building C2 infrastructure.

**Common restrictions and bypasses:**

| Restriction                         | Bypass                                                         |
| ----------------------------------- | -------------------------------------------------------------- |
| Block all outbound except 80/443    | HTTP/HTTPS C2, protocol tunneling                              |
| DNS restricted to internal resolver | DNS tunneling if internal resolver allows external queries     |
| ICMP blocked                        | HTTP/HTTPS C2                                                  |
| HTTP proxy required                 | Use CONNECT method for tunneling, configure tools to use proxy |
| SSL inspection                      | Domain fronting, JA3 mimicry, certificate pinning bypass       |
