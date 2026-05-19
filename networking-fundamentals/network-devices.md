# Network Devices

Understanding the hardware that makes up a network tells you where traffic flows, where segmentation exists, and where you can intercept or inject.

**Hub**

A hub is a Layer 1 device that repeats any signal it receives out of all its other ports. Every device connected to a hub sees every other device's traffic. Hubs are essentially extinct in modern networks (replaced by switches shem) but you'll occasionally find them in old legacy environments or embedded systems. From an attacker's perspective, a hub is paradise: passive sniffing captures everything.

**Switch**

A switch is a Layer 2 device that forwards frames based on MAC addresses. It maintains a **CAM table** (Content Addressable Memory) that maps MAC addresses to physical ports. When a frame arrives, the switch looks up the destination MAC, finds the port it's on, and forwards the frame only to that port. Unlike hubs, switches isolate traffic, you only see what's addressed to you.

This is why passive sniffing doesn't work out of the box on switched networks — you need to actively poison ARP to redirect traffic through yourself (see section "Core Protocols").

**Managed vs. unmanaged switches:** Managed switches support VLANs, port security, SNMP management, and logging. Unmanaged switches just forward frames with no configuration. Managed switches are more common in enterprise environments and present more attack surface (default credentials, SNMP community strings, telnet management).

**Router**

A router is a Layer 3 device that forwards packets between different networks based on IP addresses. It maintains a **routing table** of known networks and the next hop to reach them. When a packet arrives, the router looks up the destination IP, finds the best route, and forwards the packet out the appropriate interface.

Routers are the boundaries between network segments. Compromising a router gives you visibility into all traffic flowing through it and the ability to manipulate routing for traffic redirection or interception.

**Firewall**

A firewall filters traffic based on rules. Rules typically specify source/destination IP, port, protocol, and direction. Firewalls range from simple packet filters to deep-packet inspection next-generation firewalls. Covered in detail in section 1.8.

**Access Point (AP)**

A wireless access point bridges the wireless (802.11) and wired (Ethernet) networks. Clients connect wirelessly to the AP; the AP forwards their traffic onto the wired network. APs are the target in wireless attacks (section 1.12).

**Load Balancer**

Distributes incoming traffic across multiple backend servers. Relevant in web application engagements — understanding that you're hitting a load balancer explains why sessions may behave inconsistently, and the real backend servers may have a different security posture than the frontend.

**Proxy**

An intermediary that makes requests on behalf of clients. **Forward proxies** sit between internal users and the internet (corporate web proxy). **Reverse proxies** sit between external users and internal servers (WAF, CDN, API gateway). Understanding which proxies exist affects how traffic flows and how your activity is logged.

_NOTE: Burpsuite sits between your browser (client) and the internet (server) and makes web requests on your behalf, like a forward proxy. You can call it an intercepting forward proxy._

_What makes it special is that it lets you see, stop, and modify those requests before they reach the server, which is why it’s used for web security testing_
