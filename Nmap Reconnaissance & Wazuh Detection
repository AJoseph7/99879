# Nmap Reconnaissance & Wazuh Detection

**Date:** May 2026  
**Lab Environment:** VirtualBox , Kali Linux (attacker) → Ubuntu 22.04 (defender/SIEM)

## Objective
Perform network reconnaissance from Kali Linux against the Ubuntu target and observe 
how Wazuh SIEM detects and logs the activity.

## Tools Used
- Nmap 7.x (Kali Linux)
- Wazuh 4.7.5 (Ubuntu — Manager + Dashboard)

## Network Setup
| Machine | IP | Role |
|---------|-----|------|
| Kali Linux | 192.168.20.11 | Attacker |
| Ubuntu 22.04 | 192.168.20.10 | Defender / SIEM |

---

## Scan 1 // Service Version Detection

**Command:**
```bash
nmap -sV 192.168.20.10
```

**What it does:**
- Scans the 1000 most common TCP ports
- `-sV` attempts to identify the service and version running on each open port

**Results:**
- Host: UP (0.0002s latency)
- 999 ports: closed (reset)
- 1 port open: `443/tcp ssl/https` — Wazuh Dashboard
- MAC Address: 08:00:27:42:9F:73 (VirtualBox NIC)
- Scan completed in 18.16 seconds

**Key observation:** Nmap identified the Wazuh dashboard running on port 443 
but could not fully identify the service — returned raw HTTP fingerprint data 
instead of a known service signature.

---

## Scan 2  Aggressive OS Detection

**Command:**
```bash
nmap -sS -A -O 192.168.20.10
```

**What it does:**
- `-sS` SYN scan (sends SYN packets, more detectable)
- `-A` aggressive mode, enables OS detection, version detection, script scanning
- `-O` OS fingerprinting

**Results:**
- OS detected: Linux 4.15 - 5.19
- Device type: general purpose/router
- TRACEROUTE: 1 hop to 192.168.20.10 (1.03ms)
- Scan completed in 28.08 seconds

---

## Wazuh Detection

After both scans, the Wazuh dashboard registered:

| Metric | Value |
|--------|-------|
| Total alerts generated | 130 |
| Level 12+ alerts | 0 |
| Authentication failures | 0 |
| Authentication successes | 8 |

**Top MITRE ATT&CK techniques detected:**
- Valid Accounts
- Sudo and Sudo Caching
- Disable or Modify Tools

**Notable alerts from Kali agent:**
- `SCA summary: System audit score less than 30%`   Level 9
- Multiple SSH hardening failures  Level 7
- Password policy violations   Level 3

**Alert spike visible** at 12:00-15:00 timestamp in the dashboard  
directly correlating with when the Nmap scans were executed.

---

## Key Takeaways

- Nmap reconnaissance generates significant noise in a SIEM even without 
  an active attack
- Wazuh SCA (Security Configuration Assessment) automatically audited 
  the Kali agent and flagged it as poorly secured (score < 30%) — 
  expected for an offensive security distro
- Port 443 being the only open port confirms the Ubuntu VM has a minimal 
  attack surface — only the Wazuh dashboard is exposed
- Even basic reconnaissance triggers dozens of alerts, highlighting the 
  importance of baselining normal network behavior in SOC environments

## Next Steps
- Install Wireshark on Kali to capture and analyze scan traffic
- Configure iptables logging on Ubuntu for network-level detection
- Simulate brute force attack and observe Wazuh response
