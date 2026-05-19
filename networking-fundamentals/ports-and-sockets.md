---
hidden: true
---

# Ports and Sockets

**What Ports Are**

An IP address identifies a host. A **port** identifies a specific service or process on that host. Ports are 16-bit numbers ranging from 0 to 65,535.

A **socket** is the combination of an IP address and a port: `192.168.1.10:80`. A **connection** is defined by four values: source IP, source port, destination IP, destination port. This four-tuple is what allows a host to maintain thousands of simultaneous connections — each has a unique combination.

**Port Ranges**

| Range       | Name              | Notes                                        |
| ----------- | ----------------- | -------------------------------------------- |
| 0–1023      | Well-known ports  | Assigned by IANA; require root/admin to bind |
| 1024–49151  | Registered ports  | Common application ports                     |
| 49152–65535 | Dynamic/ephemeral | Assigned to clients for outgoing connections |

When you connect to a web server on port 80, your OS assigns you an ephemeral source port (e.g., `54231`) for the outgoing connection. The server sees `your_ip:54231 → server_ip:80`.

**Key Ports to Know**

| Port      | Protocol | Service             |
| --------- | -------- | ------------------- |
| 21        | TCP      | FTP (control)       |
| 22        | TCP      | SSH                 |
| 23        | TCP      | Telnet              |
| 25        | TCP      | SMTP                |
| 53        | UDP/TCP  | DNS                 |
| 67/68     | UDP      | DHCP                |
| 80        | TCP      | HTTP                |
| 88        | TCP      | Kerberos            |
| 110       | TCP      | POP3                |
| 135       | TCP      | RPC endpoint mapper |
| 137-139   | UDP/TCP  | NetBIOS             |
| 143       | TCP      | IMAP                |
| 161/162   | UDP      | SNMP                |
| 389       | TCP      | LDAP                |
| 443       | TCP      | HTTPS               |
| 445       | TCP      | SMB                 |
| 636       | TCP      | LDAPS               |
| 873       | TCP      | Rsync               |
| 1433      | TCP      | MSSQL               |
| 1521      | TCP      | Oracle DB           |
| 2049      | TCP      | NFS                 |
| 3306      | TCP      | MySQL               |
| 3389      | TCP      | RDP                 |
| 5985/5986 | TCP      | WinRM               |
| 6379      | TCP      | Redis               |
| 8080/8443 | TCP      | HTTP alternative    |
| 27017     | TCP      | MongoDB             |
