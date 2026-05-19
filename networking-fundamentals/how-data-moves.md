# How Data Moves

**Packets and Frames**

Data isn't sent across a network as one continuous stream, it's broken into small chunks. To reiterate from the previous section:&#x20;

At Layer 3 (network), these chunks are called **packets**. At Layer 2 (data link), they're called **frames**. At Layer 4 (transport), they're called **segments** (TCP) or **datagrams** (UDP).

Each chunk contains:

* **Header**: control information (source, destination, protocol type, sequence number, etc.)
* **Payload**: the actual data being carried
* **Trailer** (sometimes): error-checking information, mainly at Layer 2

**Encapsulation**

As data travels down the network stack to be sent, each layer wraps the data from the layer above with its own header. This is called **encapsulation**.

```
Application data: "GET / HTTP/1.1"
    ↓ Transport adds TCP header (ports, sequence numbers)
    ↓ Network adds IP header (source/dest IP)
    ↓ Data Link adds Ethernet header (source/dest MAC) + trailer
    ↓ Physical converts to bits and transmits
```

On the receiving end, each layer strips its header and passes the payload up, this is called **decapsulation**.😭😭

Understanding encapsulation is important because it means each layer only sees what it needs to see. A firewall inspecting Layer 3 doesn't automatically see Layer 7 content. A switch operating at Layer 2 doesn't look at IP addresses. This "layered ignorance" creates opportunities for evasion and tunneling.

**MTU and Fragmentation**

The **Maximum Transmission Unit (MTU)** is the largest packet size that can be sent in a single frame on a given network. Ethernet's standard MTU is **1500 bytes**. If a packet is larger, it gets fragmented into smaller pieces and reassembled at the destination.

Fragmentation is an attack surface:

* Some IDS/firewall systems struggle with fragmented packets
* Overlapping fragments can cause different reassembly behavior on different operating systems (fragment overlap attacks)
* nmap's `-f` flag deliberately sends fragmented probes to evade inspection

