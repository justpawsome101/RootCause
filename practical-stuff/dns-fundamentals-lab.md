---
description: >-
  DNS gets treated as boring infrastructure, but it sits at every stage of an
  attack: reconnaissance, defense evasion, and even command and control. This
  lab builds a DNS environment from scratch.
---

# DNS Fundamentals Lab

#### Setup

Ubuntu Server as the authoritative DNS server, base machine as the attacker box. Installed BIND9 on Ubuntu and confirmed it was running via `systemctl status bind9`. Walked through the default zone files in `/etc/bind/` to get a feel for the boilerplate (`db.local`, `db.127`, etc.) before building anything custom.

<figure><img src="../.gitbook/assets/image (41).png" alt=""><figcaption><p>Installing BIND9 on Ubuntu VM.</p></figcaption></figure>

<figure><img src="../.gitbook/assets/image (42).png" alt=""><figcaption><p>File contents of db.local for reference.</p></figcaption></figure>

#### 1. Authoritative DNS Server

Created a zone file for `amogus.sus`, defining the SOA record (serial, refresh, retry, expire, negative cache TTL), an NS record pointing to `ns1.amogus.sus`, and a handful of A records (`www`, `mail`, `dev`, `vpn`, `internal`, `db-prod`) all pointing at the same host, plus a CNAME (`remote` → `vpn`) and an MX record for `mail`.&#x20;

<figure><img src="../.gitbook/assets/image (43).png" alt=""><figcaption><p>Created amogus.sus zone file.</p></figcaption></figure>

Confirmed DNS resolution worked end-to-end with `dig` from my host machine, both for real records and for a nonexistent one (`fake.amogus.sus`, which correctly returned NXDOMAIN with the SOA in the authority section).

<figure><img src="../.gitbook/assets/image (45).png" alt=""><figcaption><p>Checked name resolution works using dig and an exising domain.</p></figcaption></figure>

<figure><img src="../.gitbook/assets/image (46).png" alt=""><figcaption><p>Confirmed name resolution works using dig and a non-exising domain.</p></figcaption></figure>

**Takeaway:** DNS is just flat text files mapping names to data, and the SOA record is what declares who's authoritative. Seeing `aa` (authoritative answer) in a `dig` response means you're talking to the source of truth, not a caching resolver.

#### 2. Zone Transfers (AXFR)

Set `allow-transfer { any; }` on the zone and ran `dig axfr` .

<figure><img src="../.gitbook/assets/image (44).png" alt=""><figcaption><p>Set allow-transfer to "any".</p></figcaption></figure>

&#x20;Got the entire zone dumped in one query: every hostname, every IP, the mail server, the "internal" and "db-prod" records, all of it, with zero authentication.&#x20;

<figure><img src="../.gitbook/assets/image (47).png" alt=""><figcaption><p>Zone dumped.</p></figcaption></figure>

Then locked it down with `allow-transfer { none; }`, restarted BIND, and confirmed the transfer now failed.

<figure><img src="../.gitbook/assets/image (48).png" alt=""><figcaption><p>Set allow transfer to "none".</p></figcaption></figure>

<figure><img src="../.gitbook/assets/image (49).png" alt=""><figcaption><p>Zone transfer failed.</p></figcaption></figure>

**Takeaway:** AXFR exists for primary/secondary replication, but misconfigured it just hands over a full infrastructure map. It's still one of the first things worth checking on an external assessment (`dig axfr @ns1.target.com target.com`) before anything else because a single hit here can define the rest of the engagement. Proper fix beyond `allow-transfer` restrictions is TSIG (a shared key that signs transfer requests, so IP-based restrictions can't be spoofed around).

<figure><img src="../.gitbook/assets/image (50).png" alt=""><figcaption><p>Variations and allowed users.</p></figcaption></figure>

#### 3. Subdomain Enumeration

Tried `dnsrecon` first but hit a Python/dependency issue, so switched to `fierce`. With transfer set back to `any`, fierce basically mirrored what a full AXFR gives you.&#x20;

<figure><img src="../.gitbook/assets/image (51).png" alt=""><figcaption><p>Zone file dumped with fierce.</p></figcaption></figure>

Set transfer to `none` and re-ran it, this time it fell back to wordlist-based brute forcing and only found the more "guessable" names (`dev`, `internal`, `mail`, `ns1`), missing `db-prod`.

<figure><img src="../.gitbook/assets/image (52).png" alt=""><figcaption><p>Subdomains brute forced by fierce.</p></figcaption></figure>

**Takeaway:** Zone transfer (when it works) is complete and instant; brute force is limited by the choice of wordlist. In the real world, where AXFR is almost always locked down the next steps are to brute force with a good wordlist and check certificate transparency logs (crt.sh) for historical subdomain data. Each method catches different things.

#### 4. PTR Records / Reverse DNS

Built a reverse zone (`10.168.192.in-addr.arpa`) with PTR records mapping the host IP back to `ns1`, `www`, and `mail`. Queried it directly with `dig -x` and got all three hostnames back for one IP.

<figure><img src="../.gitbook/assets/image (53).png" alt=""><figcaption><p>Reverse zone.</p></figcaption></figure>

<figure><img src="../.gitbook/assets/image (54).png" alt=""><figcaption><p>Added reverse zone to conf.local file.</p></figcaption></figure>

<figure><img src="../.gitbook/assets/image (55).png" alt=""><figcaption><p>Querying with dig shows three hostnames for one IP.</p></figcaption></figure>

**Takeaway:** Forward and reverse DNS are separate zones, configured independently and reverse often gets neglected. On a real engagement an attacker would often start from a known IP range (from a WHOIS lookup) rather than a domain name, and sweep it with a command such as: `for i in $(seq 1 254); do dig @ns -x 192.168.10.$i +short; done`.&#x20;

#### 5. DNS Sinkhole

Built a `blocklist` zone for a fake domain (`imposter.com`) with a wildcard `* IN A 0.0.0.0` and a short (60s) TTL, registered it in `named.conf.local`.&#x20;

<figure><img src="../.gitbook/assets/image (56).png" alt=""><figcaption><p>Blocklst file contents.</p></figcaption></figure>

<figure><img src="../.gitbook/assets/image (57).png" alt=""><figcaption><p>Added imposter.com to blocklist zone.</p></figcaption></figure>

Queried both the base domain and an arbitrary subdomain (`vent.imposter.com`) and both resolved to `0.0.0.0` thanks to the wildcard, showing that any subdomain under a sinkholed domain gets caught automatically.

<figure><img src="../.gitbook/assets/image (58).png" alt=""><figcaption><p>Resolved to 0.0.0.0.</p></figcaption></figure>

<figure><img src="../.gitbook/assets/image (59).png" alt=""><figcaption><p>Resolved to 0.0.0.0.</p></figcaption></figure>

**Takeaway:** A sinkhole works by intercepting the query at the resolver and returning a fake answer, so the client never reaches the real destination. Short TTL means the blocklist updates fast. This is basic incident response tradecraft  (a compromised host beaconing to `c2.evilhacker.com` gets its C2 channel killed at the DNS layer without ever touching the infected machine). Pi-hole is this same mechanism with a UI on top.

From the offensive side: if a domain controlled by the attacker is queried mid engagement and returns  `0.0.0.0` back instead of the real IP signals that the attacker is behind a sinkhole and their C2 is being intercepted.

#### 6. DoH Bypass

Queried `imposter.com` again, this time straight against Cloudflare's `1.1.1.1` instead of the local sinkholed resolver and got the _real_ IP (`64.190.63.222`) back instead of `0.0.0.0`. Same domain, completely different answer depending on which resolver was asked.

<figure><img src="../.gitbook/assets/image (60).png" alt=""><figcaption><p>Query using Cloudflares IP.</p></figcaption></figure>

**Takeaway:** DNS-over-HTTPS/TLS sends queries to a hardcoded resolver over 443/853, sidestepping any network-level DNS control entirely, including sinkholes. This is why modern malware hardcodes DoH: it bypasses network level defenses that assume they're the only path DNS traffic can take. Blocking DoT (port 853) at the firewall is easy, but DoH works on 443 alongside normal HTTPS traffic, so catching it needs SSL inspection or endpoint agents rather than just DNS-layer controls.

#### 7. DNS Tunneling (dnscat2)

Compiled and ran the dnscat2 server on Ubuntu, connected a client from Kali with a pre-shared secret, got an encrypted, verified session. From the server side, spawned a `shell` session inside that tunnel and ran commands (`whoami`, `pwd`) purely over DNS query/response traffic and confirmed with a working reverse shell riding entirely on TXT/CNAME/MX-style queries.

<figure><img src="../.gitbook/assets/image (61).png" alt=""><figcaption><p>Running dnscat2.</p></figcaption></figure>

<figure><img src="../.gitbook/assets/image (62).png" alt=""><figcaption><p>Successful connection.</p></figcaption></figure>

**Takeaway:** DNS is almost never blocked outbound, even on tightly firewalled networks. HTTP might be blocked, direct TCP might be blocked, but DNS resolution usually has to work or nothing else does either. Encoding data into subdomain labels and using an attacker-controlled authoritative server gets you a full bidirectional channel, including a shell, that most perimeter firewalls don't even see as suspicious.&#x20;

Detection wise, this isn't something a basic firewall catches, it needs deep DNS inspection. Tools like Zeek with DNS logging, or an NDR platform, are what actually catch this in practice. But many companies do not use these.

#### Conclusion

DNS isn't a side channel, it spans the whole kill chain: recon through zone misconfigurations, evasion through sinkholes and DoH, and even C2 through tunneling when nothing else gets out. Treating DNS with the same scrutiny as any other trust boundary, not an afterthought, is the main takeaway here.

A next step would be building out the defensive side: query logging, entropy analysis, and proper detection with something like Zeek to gain a better understanding of the defensive side.
