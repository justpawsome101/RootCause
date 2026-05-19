---
hidden: true
---

# A Complete Network Recon Workflow

When you land on a new network, work through this systematically.

**Step 1 — Understand your position**

bash

```bash
# What interfaces do I have?
ip addr
ipconfig /all          # Windows

# What's my routing table? What subnets are reachable?
ip route
route print            # Windows

# What DNS am I using?
cat /etc/resolv.conf
ipconfig /displaydns   # Windows cached DNS

# What's in my ARP cache already?
arp -a

# What hosts/services are listening on this machine?
ss -tlnp
netstat -tlnp
```

**Step 2 — Identify live hosts**

bash

```bash
arp-scan -l
nmap -sn 192.168.1.0/24
netdiscover -r 192.168.1.0/24
```

**Step 3 — Quick port scan of all live hosts**

bash

```bash
nmap -T4 --open -F 192.168.1.0/24 -oN quick_scan.txt
```

**Step 4 — Full scan of interesting hosts**

bash

```bash
nmap -sV -sC -p- -T4 192.168.1.10 -oN full_192.168.1.10.txt
```

**Step 5 — DNS enumeration**

bash

```bash
dig NS target.local
dig axfr @192.168.1.53 target.local
for ip in $(seq 1 254); do dig -x 192.168.1.$ip +short; done
```

**Step 6 — Service-specific enumeration**

bash

```bash
# SMB
crackmapexec smb 192.168.1.0/24
enum4linux -a 192.168.1.10

# SNMP
onesixtyone -c /usr/share/seclists/Discovery/SNMP/snmp.txt 192.168.1.0/24
snmpwalk -c public -v1 192.168.1.1

# LDAP (Active Directory)
ldapsearch -x -H ldap://192.168.1.10 -b "DC=target,DC=local"
crackmapexec ldap 192.168.1.10 -u '' -p ''   # anonymous bind check

# Web services
whatweb http://192.168.1.10
nikto -h http://192.168.1.10
gobuster dir -u http://192.168.1.10 -w /usr/share/seclists/Discovery/Web-Content/common.txt

# NFS
showmount -e 192.168.1.10

# FTP
ftp 192.168.1.21    # try anonymous:anonymous
```

**Step 7 — Document everything**

Every IP, open port, service version, hostname, and credential you find. Screenshots and command output. You'll be building on this for every subsequent phase of the engagement.
