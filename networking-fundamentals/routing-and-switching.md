---
hidden: true
---

# Routing and Switching

**How Switches Forward Traffic**

When a frame arrives at a switch port, the switch reads the destination MAC address and looks it up in its CAM table. If found, the frame is sent only to that port. If not found (**unknown unicast**), the frame is **flooded** out all ports except the one it came in on — this is normal switch behavior for unknown destinations.

The switch also **learns** — it associates the source MAC of every incoming frame with the port it arrived on, building up the CAM table over time.

**MAC flooding attack:**

bash

```bash
# Flood the CAM table with fake MAC addresses
# If the table fills up, some switches fail open and flood everything
macof -i eth0
```

Modern enterprise switches mitigate this with port security — limiting the number of MAC addresses learned per port.

**Spanning Tree Protocol (STP)**

Networks often have redundant switch links for fault tolerance — but redundant Layer 2 paths cause **broadcast storms** (frames looping forever). STP prevents this by automatically blocking redundant paths and only enabling them if the primary fails.

**STP attacks:**

_Root bridge takeover_ — STP elects a "root bridge" based on **Bridge ID** (priority + MAC address). The switch with the lowest Bridge ID wins. By injecting STP BPDUs (Bridge Protocol Data Units) claiming a lower priority, you can become the root bridge and force all traffic through you.

bash

```bash
# Using Yersinia
yersinia stp -attack 4   # claim root role
```

_STP DoS_ — repeatedly send topology change notifications, forcing the network into constant recalculation.

**VLANs and VLAN Hopping**

A VLAN (Virtual LAN) logically segments a physical network. Ports are assigned to VLANs — a port in VLAN 10 can only communicate with other ports in VLAN 10 without going through a router or Layer 3 switch.

**802.1Q tagging** — VLAN membership is encoded in a 4-byte tag inserted into Ethernet frames. Tags are added on trunk ports (which carry multiple VLANs) and stripped on access ports (which carry one).

**Native VLAN** — the VLAN that carries untagged traffic on a trunk port. Defaults to VLAN 1 on Cisco. Critical to configure correctly.

**Switch spoofing attack:**

Some switches have ports configured in `dynamic auto` or `dynamic desirable` mode, meaning they'll negotiate trunk links via DTP (Dynamic Trunking Protocol) with anyone asking. An attacker can send DTP frames to negotiate a trunk:

bash

```bash
# Using Yersinia
yersinia dtp -attack 1   # enable trunk
```

Once trunked, you receive traffic for all VLANs.

**Double-tagging attack:**

Craft a frame with two 802.1Q headers. The outer tag matches the native VLAN (which gets stripped and not re-tagged by the first switch). The inner tag contains the target VLAN. The second switch delivers the frame to the target VLAN.

Limitation: one-way only (no return traffic), and only works when attacker is on the native VLAN. Still useful for injecting packets into an isolated VLAN.

**Routing Fundamentals**

A routing table is a list of known networks and instructions for reaching them. Each entry has: destination network, subnet mask, next hop (or exit interface), metric (cost), and administrative distance (trustworthiness of the source).

bash

```bash
# View routing table
ip route          # Linux
route -n          # Linux, numeric
netstat -r        # Linux/Windows
route print       # Windows
```

**Key routing concepts for red teamers:**

_Default route_ (`0.0.0.0/0`) — "if no more specific route exists, send traffic here." This is the default gateway. Controlling the default gateway means intercepting all off-subnet traffic.

_Longest prefix match_ — when multiple routes could match a destination, the most specific (longest) one wins. `192.168.1.0/24` beats `192.168.0.0/16` for a destination of `192.168.1.50`.

_Dual-homed hosts_ — a host with interfaces on two different subnets is a natural pivot point. It can route traffic between segments and may have weaker controls than dedicated network infrastructure.
