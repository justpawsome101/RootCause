# The basics

#### Ah yes of course, the infamous starter: What is a network??

Its obviously all of your LinkedIn connections. (Fun fact, I have like 1000+ connections because I was unfortunately a LinkedIn warrior in my early university days.😭)

Anyways, a network in this case is **two or more devices connected in a way that allows for them to communicate and share resources**.

Networks exist at different scales (as we said "**two or more devices"**).

The most important two are:

**LAN (Local Area Network)** : a network confined to a small geographic area, like an office or building. Devices on a LAN are typically connected via Ethernet cable or Wi-Fi. Communication is fast and cheap. This is the environment you'll most often be operating inside during an engagement.

**WAN (Wide Area Network)** : a network spanning large geographic distances. The internet is the largest WAN. Organizations connect their offices together over WANs.

<figure><img src="../../.gitbook/assets/image.png" alt=""><figcaption></figcaption></figure>

**WLAN (Wireless LAN)** — a LAN using wireless radio communication instead of cables. Same logical behavior as a wired LAN.

#### In real environments, there is rarely “one internal network”.

Companies typically have many separate local networks across headquarters, branches, and data centers that need to be joined together so they can communicate as one. These connections form a **WAN (Wide Area Network)**, and they're built using one of the following technologies:

* **Leased Lines**: A permanently reserved, private cable between two locations. Fast and reliable, but expensive and inflexible.
* **MPLS (Multiprotocol Label Switching)**: A private virtual connection rented from a telecom provider. Traffic is routed efficiently using labels rather than IP addresses. More flexible than leased lines, but still costly.
* **VPN (Virtual Private Network)**: An encrypted tunnel built on top of the public internet. Cheaper than leased lines or MPLS, but performance depends on internet quality.
* **SD-WAN (Software-Defined Wide Area Network):**  Software that manages multiple connections (including VPNs and leased lines) simultaneously, automatically choosing the best path for each type of traffic. More flexible and cost-effective than traditional options.
* **SASE (Secure Access Service Edge):** A cloud-based framework that bundles SD-WAN with security services like firewalls and Zero Trust. Designed for modern companies with remote workers — users connect securely to company apps directly via the cloud, without all traffic being funneled through a central office first.

#### Most site-to-site connections today are **VPNs using IPsec tunnels**.

A VPN is an encrypted tunnel over the internet that connects two distant networks so they behave as if they are directly plugged into each other.

An IPsec tunnel works by:

1. Taking a normal internal packet (e.g sending data from `10.1.0.5` to `10.2.0.8`)
2. The firewall intercepts it, encrypts it and wraps it in a new outer packet
3. That outer packet travels across the internet between the two firewalls
4. The other firewall unwraps and decrypts it, then delivers it locally

To hosts on each LAN, it appears as a direct connection.

To the internet, it is unreadable encrypted traffic between two firewalls.

<figure><img src="../../.gitbook/assets/image (13).png" alt=""><figcaption></figcaption></figure>

A different way to think about this is the analogy that  you and your friend live far apart, but you want to pass secret notes. Instead of meeting in person, you put your note inside a locked box, mail it through the regular postal system, and your friend unlocks it with a key only they'd have on the other side. To anyone handling the mail, it's just a sealed box, they can't read what's inside.

That's basically a VPN. Two office networks passing private traffic through the public internet, locked so nobody in between can read it.

IPsec is just the specific locking mechanism most companies use.

#### Many organizations now use **SD-WAN** devices at each site.

These devices build VPN tunnels over multiple links (eg. internet, MPLS) and automatically choose the best path for traffic based on application and link quality.

For a tester, this means:

* VPN tunnels may exist over several different internet connections
* Traffic paths can change dynamically
* Branch offices may have direct internet access instead of routing through HQ
* The “shape” of the network is less obvious than the routing table suggests

The “internal network” is usually several separate office networks connected together with encrypted tunnels.

#### Networks are also described by their **topology**  which refers to the physical arrangement of devices:

* **Star** : all devices connect to a central switch or router. Most common in modern networks.
* **Bus** : all devices on a single shared cable. Legacy, rarely seen now.
* **Ring** : devices connected in a loop. Also largely legacy.
* **Mesh** : devices interconnect with each other directly. Common in wireless and WAN redundancy designs.
* **Hybrid** : combinations of the above. Most real networks are hybrid.

<figure><img src="../../.gitbook/assets/image (15).png" alt=""><figcaption></figcaption></figure>

Topology matters because it affects how traffic flows, where bottlenecks are, and where you can position yourself to intercept communications.

