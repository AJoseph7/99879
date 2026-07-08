# Networking Fundamentals
**Platform:** TryHackMe  Cyber Security 101 path
**Rooms:** Networking Concepts, Networking Essentials, Networking Core Protocols, Networking Secure Protocols
**Completed as:** Single study block

---

## What I learned

Networking is the foundation of everything in cybersecurity. Every attack, every defence, every tool — Nmap, Wireshark, Metasploit, VPNs — only makes sense if you understand how data travels across a network. Without this base, you are just running commands you do not understand.

The key insight is that networks are built on **layers and protocols**. Each layer has a specific job, and each protocol has rules that define how devices communicate. When those rules are abused or misconfigured, that is where attacks happen.

---

## The OSI Model — 7 layers, one framework

The OSI (Open Systems Interconnection) model is a conceptual framework that describes how data travels from one device to another. Every network tool and attack maps to one or more of these layers.

| Layer | Name | What it does | Security relevance |
|---|---|---|---|
| 7 | Application | User-facing protocols — HTTP, FTP, SMTP, DNS | Most attacks target this layer — phishing, SQL injection, XSS |
| 6 | Presentation | Encryption, encoding, compression | TLS/SSL operates here — where HTTPS encryption happens |
| 5 | Session | Opens, manages, and closes communication sessions | Session hijacking attacks target this layer |
| 4 | Transport | Breaks data into segments — TCP (reliable) vs UDP (fast) | Port scanning targets this layer — Nmap operates here |
| 3 | Network | Routes packets using IP addresses | IP spoofing, routing attacks |
| 2 | Data Link | Transfers frames between devices on the same network using MAC addresses | ARP spoofing, MAC flooding |
| 1 | Physical | The actual cables, Wi-Fi signals, hardware | Physical access attacks, signal interception |

---

## Encapsulation — how data travels through the layers

Encapsulation is the process of each layer wrapping the data from the layer above it with its own header before sending it down.

```
Application data
      ↓ + TCP header  →  Segment       (Transport layer)
      ↓ + IP header   →  Packet        (Network layer)
      ↓ + MAC header  →  Frame         (Data Link layer)
      ↓ converted     →  Bits          (Physical layer)
```

On the receiving end, each layer strips its own header and passes the data up — this is called **de-encapsulation**.

**Why this matters in security:** Tools like Wireshark capture packets at the frame level and let you inspect every layer. If data is not encrypted at the Presentation layer (no HTTPS), you can read the actual content of the communication — usernames, passwords, everything.

---

## TCP/IP Model — the real-world version

The TCP/IP model is what the internet actually uses. It simplifies the OSI model into 4 layers:

| TCP/IP Layer | Equivalent OSI Layers | Key protocols |
|---|---|---|
| Application | 5, 6, 7 | HTTP, HTTPS, FTP, SMTP, DNS, SSH |
| Transport | 4 | TCP, UDP |
| Internet | 3 | IP, ICMP |
| Network Access | 1, 2 | Ethernet, Wi-Fi, ARP |

---

## TCP vs UDP — reliability vs speed

| Feature | TCP | UDP |
|---|---|---|
| Connection | Requires handshake (SYN → SYN-ACK → ACK) | No connection — just sends |
| Reliability | Guarantees delivery and order | No guarantee |
| Speed | Slower | Faster |
| Used for | HTTP, HTTPS, FTP, SSH, SMTP | DNS, VoIP, video streaming |
| Security relevance | SYN flood attacks abuse the TCP handshake | DNS amplification attacks use UDP |

---

## IP Addresses, Subnets and DHCP

### Reading an ifconfig output

During the TryHackMe labs, I worked with the following network configuration:

```
wlo1: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
      inet 192.168.66.89  netmask 255.255.255.0  broadcast 192.168.66.255
      inet6 fe80::73e1:ca5e:3f93:b1b3  prefixlen 64
      ether cc:5e:f8:02:21:a7
```

Breaking this down:

| Field | Value | What it means |
|---|---|---|
| inet | 192.168.66.89 | Private IPv4 address of this machine |
| netmask | 255.255.255.0 | Defines the network range — /24 means 254 usable hosts |
| broadcast | 192.168.66.255 | Address used to send a message to all devices on the network |
| inet6 | fe80::73e1... | IPv6 link-local address |
| ether | cc:5e:f8:02:21:a7 | MAC address — unique hardware identifier |

### IPv4 vs IPv6

| Feature | IPv4 | IPv6 |
|---|---|---|
| Format | 192.168.1.1 | fe80::73e1:ca5e:3f93:b1b3 |
| Address space | ~4.3 billion addresses | 340 undecillion addresses |
| Why it matters | Running out of addresses  NAT was created to extend it | The long-term solution to address exhaustion |

### DHCP — automatic IP assignment

DHCP (Dynamic Host Configuration Protocol) automatically assigns IP addresses to devices when they join a network. Without DHCP, every device would need a manually configured static IP.

**Security relevance:** A rogue DHCP server on the network can assign itself as the default gateway, intercepting all traffic — a man-in-the-middle attack. This is why DHCP snooping is enabled on enterprise switches.

### Public IP and NAT

Most home and office networks use private IP addresses (192.168.x.x, 10.x.x.x). NAT (Network Address Translation) allows all these private devices to share a single public IP when accessing the internet.

**Security relevance:** NAT hides internal network structure from the outside world — an attacker scanning from the internet cannot see individual internal IPs. However, once inside the network, NAT provides no protection.

---

## Key protocols and port numbers

Every protocol uses a specific port number. Knowing these is essential for reading firewall rules, Nmap results, and network logs.

| Protocol | Transport | Port | Purpose | Security relevance |
|---|---|---|---|---|
| HTTP | TCP | 80 | Unencrypted web traffic | All data visible in plaintext — use HTTPS instead |
| HTTPS | TCP | 443 | Encrypted web traffic | TLS encrypts the session — standard for all web traffic |
| FTP | TCP | 21 | File transfer | Credentials sent in plaintext — replaced by SFTP/FTPS |
| SSH | TCP | 22 | Encrypted remote access | Secure replacement for Telnet |
| TELNET | TCP | 23 | Unencrypted remote access | Never use — sends everything including passwords in plaintext |
| SMTP | TCP | 25 | Sending email | Used in phishing infrastructure — monitor outbound on port 25 |
| DNS | UDP/TCP | 53 | Resolves domain names to IPs | DNS poisoning redirects users to malicious sites |
| POP3 | TCP | 110 | Receiving email (downloads) | Unencrypted — use POP3S (port 995) |
| IMAP | TCP | 143 | Receiving email (syncs) | Unencrypted — use IMAPS (port 993) |

> **Note:** I do not have all ports memorised but what matters is knowing the most common ones (80, 443, 22, 21, 53) and knowing how to look up the rest. In real work, you reference documentation you do not memorise everything.

---

## Application layer protocols in detail

### DNS — Domain Name System
DNS translates human-readable domain names into IP addresses. When you type `google.com`, DNS resolves it to `142.250.x.x`.

**Security relevance:** DNS queries are often unencrypted  an attacker on the same network can see every domain you visit. DNS poisoning can redirect users to fake sites. DNS is also used for data exfiltration  malware can encode stolen data inside DNS queries.

### HTTP vs HTTPS
HTTP sends all data in plaintext. HTTPS wraps HTTP inside TLS encryption.

**Security relevance:** On a network captured with Wireshark, HTTP traffic shows usernames and passwords in cleartext. HTTPS makes this impossible to read without breaking the encryption.

### FTP commands
| Command | Purpose |
|---|---|
| USER | Send username |
| PASS | Send password (in plaintext on FTP) |
| RETR | Download a file from the server |
| STOR | Upload a file to the server |

**Security relevance:** FTP sends credentials in plaintext — anyone capturing traffic with Wireshark or Tcpdump can read them. Always use SFTP or FTPS instead.

### SMTP, POP3 — email protocols
- **SMTP** (port 25): Used to send email. Attackers abuse open SMTP relays to send spam and phishing emails.
- **POP3** (port 110): Downloads email to the client and typically deletes it from the server.
- **IMAP** (port 143): Syncs email across multiple devices  more common today.

### WHOIS
WHOIS is a query protocol that returns registration information about a domain — who owns it, when it was registered, name servers, and sometimes contact details.

**Security relevance:** Used in OSINT (Open Source Intelligence) to research threat actors and phishing domains. If you receive a suspicious email, WHOIS the sending domain.

### Telnet
Telnet allows remote command-line access to a device  but sends everything in plaintext including passwords.

**Security relevance:** Telnet should never be used in production. Finding an open Telnet port (23) on a network scan is a critical finding — it means credentials can be intercepted. SSH (port 22) is the secure replacement.

### HTTP methods
| Method | Purpose | Security relevance |
|---|---|---|
| GET | Retrieve data from a server | Parameters visible in URL  never use GET for sensitive data |
| POST | Send data to a server | Data in request body  used for login forms, file uploads |
| PUT | Upload or replace a resource | Can be abused to upload malicious files if not restricted |

---

## VPN — encrypted tunnels

A VPN (Virtual Private Network) creates an encrypted tunnel between a device and a VPN server. All traffic travels through this tunnel, hiding it from anyone monitoring the local network.

**Security relevance:**
- Protects traffic on untrusted networks (public Wi-Fi)
- Used by companies to allow remote workers to access internal resources securely
- Attackers also use VPNs to hide their origin  a reason why VPN traffic appears in threat intelligence reports

> **Connection to home lab:** The internal network configuration in my VirtualBox lab (Kali → Ubuntu on an isolated network) applies the same principle as a VPN — traffic is isolated and cannot reach or be seen from the outside network.

---

## Real-world application

**Evidence:** An organisation running unencrypted protocols (HTTP, FTP, Telnet) on its internal network, combined with no DHCP snooping and public DNS over port 53 without DNS-over-HTTPS, exposes all internal communications to interception.

**Risk level:** High

**Impact:**
- An attacker on the same network segment can use Wireshark or Tcpdump to capture FTP credentials, HTTP session cookies, and email content in plaintext
- A rogue DHCP server can redirect all traffic through an attacker-controlled gateway
- DNS poisoning can redirect employees to fake login pages

**How I would mitigate it:**
- Replace HTTP with HTTPS everywhere  enforce with HSTS headers
- Replace FTP with SFTP  block port 21 at the firewall
- Replace Telnet with SSH  block port 23 at the firewall
- Enable DHCP snooping on managed switches
- Use DNS over HTTPS (DoH) or DNS over TLS (DoT) to encrypt DNS queries
- Monitor unusual traffic on non-standard ports using Wireshark or Tcpdump (covered in next report)

---

## Quality checklist
- [x] Can explain the OSI model and map real attacks to specific layers
- [x] Understands encapsulation and why it matters for tools like Wireshark
- [x] Can read an ifconfig output and identify IP, MAC, subnet, and broadcast
- [x] Knows the difference between TCP and UDP and when each is used
- [x] Knows the most critical port numbers and what runs on them
- [x] Understands why protocols like FTP, Telnet, and HTTP are security risks
- [x] Can explain DNS, DHCP, NAT, and VPN in a corporate security context
