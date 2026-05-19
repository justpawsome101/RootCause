# MAC Addresses

A MAC (Media Access Control) address is a 48-bit hardware identifier burned into a network interface card. Written in hexadecimal, separated by colons or hyphens: `AA:BB:CC:DD:EE:FF`.

The first three bytes (24 bits) are the **OUI (Organizationally Unique Identifier)** , assigned by IEEE to identify the manufacturer. The last three bytes are assigned by the manufacturer and (in theory) unique to the device.

bash

```bash
# Look up OUI
curl https://api.macvendors.com/AA:BB:CC:DD:EE:FF
```

OUI lookups during enumeration can reveal what hardware vendors are present in a network — useful for identifying cameras, printers, IoT devices, or virtual machine interfaces (VMware, VirtualBox have distinctive OUIs).

**MAC spoofing** — MAC addresses are software-configurable on modern interfaces. Any host can claim any MAC address. This breaks MAC-based access controls and can be used to bypass MAC filtering on wireless networks or wired 802.1X implementations that fall back to MAC authentication bypass (MAB).

bash

```bash
# Linux — spoof MAC address
ip link set dev eth0 down
ip link set dev eth0 address AA:BB:CC:DD:EE:FF
ip link set dev eth0 up

# Or using macchanger
macchanger -r eth0      # random MAC
macchanger -m AA:BB:CC:DD:EE:FF eth0   # specific MAC
```
