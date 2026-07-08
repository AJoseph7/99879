# Linux Fundamentals 

**Platform:** TryHackMe — Cyber Security 101 path  
**Category:** Operating Systems — Linux  
**Completed as:** Single study block (Parts 1, 2 & 3)

---

## What I Learned

Linux is an open-source operating system widely used in servers, cloud infrastructure, 
and cybersecurity tools. Unlike Windows, its open nature allows deep customization and 
full transparency — which is why most security tools, web servers, and CTF environments 
run on Linux.

The key insight is that in Linux, **everything is a file** — configurations, logs, 
devices, and processes. Understanding how to navigate and manipulate those files is 
the foundation of both attacking and defending systems.

---

## Commands Used & Why They Matter

| Command | Purpose |
|---------|---------|
| pwd | Print working directory — always know where you are |
| whoami | Shows current user — know your privilege level |
| ls | List files and folders — first recon step on any machine |
| cd | Navigate between directories |
| cat | Read file contents — inspect configs, logs, text files |
| grep | Filter specific strings — find keywords in large outputs |
| nano | Terminal text editor — create or modify files without a GUI |
| mkdir | Create directories |
| echo | Print text or write into a file |
| find | Search files by name, type, or permission — locate sensitive files |
| man | Open the manual for any command (ex: man ls) |
| wget [url] | Download files from a remote server directly into the machine |
| python3 -m http.server | Spin up a simple HTTP server to serve files to a target |
| fg | Bring a backgrounded process back to the foreground |
| ip a / ifconfig | Check network configuration and IP address |
| ssh user@ip | Connect remotely to another machine |
| rm | Delete files or directories — irreversible, no recycle bin ⚠️ |
| sudo | Run a command as root — grants elevated privileges ⚠️ |
| sed | Edit files non-interactively — substitute, delete, or insert lines |
| systemctl | Start, stop, restart, and check status of services |
| journalctl | Read systemd logs — used when auth.log does not exist |

---

## File Permissions & Numeric Format

Every file in Linux has permissions controlling who can read, write, or execute it.

| Symbol | Meaning | Value |
|--------|---------|-------|
| r | Read | 4 |
| w | Write | 2 |
| x | Execute | 1 |
| - | No permission | 0 |

| Symbolic | Numeric | Meaning |
|----------|---------|---------|
| rwxrwxrwx | 777 | Everyone can do everything — dangerous |
| rwxr-xr-x | 755 | Owner full access; others read and execute |
| rw-r--r-- | 644 | Owner read/write; others read only |
| rwx------ | 700 | Only owner has any access |

> Permissions are changed with `chmod` and ownership with `chown`.  
> A file with 777 permissions in a corporate environment is a red flag.

---

## Important System Directories

| Path | Purpose | Security Relevance |
|------|---------|-------------------|
| /etc/passwd | All user accounts | Readable by everyone — recon target |
| /etc/shadow | Hashed passwords | Root-only — cracking = account access |
| /root | Root user home directory | Full compromise if accessible |
| /home | Regular user directories | May contain SSH keys and scripts |
| /var/log/auth.log | Authentication log | Key for incident response |
| /var/ossec/etc/ossec.conf | Wazuh agent config | Defines which logs the agent collects |
| /etc/systemd/journald.conf | Journald config | Controls log forwarding behaviour |

---

## Evidence Collected — Lab Application

These commands were applied directly in the home lab during real investigations, 
not just studied theoretically.

**1. Investigating missing auth.log on Kali:**
```bash
sudo grep "useradd" /var/log/auth.log
# Result: /var/log/auth.log: No such file or directory
```
**Finding:** Kali Linux uses `journald` instead of traditional syslog files. 
The Wazuh agent was configured to read a file that did not exist — which is why 
no alerts were generated. Log source must always be verified before trusting SIEM data.

---

**2. Locating the event in journald:**
```bash
sudo journalctl | grep useradd
# Result: Jun 8 13:15:39 kali useradd[11339]: new user: name=hacker, UID=1001
```
**Finding:** The event occurred and was recorded — but Wazuh never saw it because 
it was reading the wrong log source. This is the difference between "nothing happened" 
and "something happened but wasn't collected."

---

**3. Editing the Wazuh agent config with sed:**
```bash
sudo sed -i '208d' /var/ossec/etc/ossec.conf
sudo sed -i 's/#ForwardToSyslog=no/ForwardToSyslog=yes/' /etc/systemd/journald.conf
```
**Finding:** `sed` edits files non-interactively without opening an editor. 
The `-i` flag modifies the file in-place. The `s/old/new/` syntax substitutes text. 
Critical for automation and scripting in real environments.

---

**4. Verifying and restarting services:**
```bash
sudo systemctl status wazuh-agent
sudo systemctl restart wazuh-agent
sudo systemctl status ssh
```
**Finding:** `systemctl` is the standard way to manage services in modern Linux. 
Knowing how to check status, restart, and troubleshoot failed services is essential 
for both attack and defence scenarios.

---

## Risk Assessment

**Scenario:** A misconfigured Linux server with weak permissions and poor log collection.

**Risk:** High

**Impact:** An attacker with shell access can read `/etc/passwd`, attempt to crack 
`/etc/shadow` hashes, escalate privileges via sudo misconfigurations, and move 
laterally across the network — all while going undetected if log collection is broken.

**Mitigation:**
- Audit permissions with `find / -perm -777` and fix with `chmod`
- Restrict sudo — only grant it to users who genuinely need it
- Disable root SSH login — enforce key-based authentication
- Verify log sources are correctly configured in the SIEM agent
- Monitor `/var/log/auth.log` for suspicious authentication activity

---

## Portfolio Connections

- Google Cybersecurity Certificate
- Home Lab: VirtualBox — Kali Linux + Ubuntu + Wazuh 4.7.5
- Wazuh Detection Lab — User Account Creation (MITRE T1136)
- SSH Brute Force Detection (MITRE T1110)

Commands covered in this document were applied directly in the lab — 
`grep`, `sed`, `nano`, `systemctl`, and `journalctl` were all used during 
real troubleshooting sessions, not just practiced in isolation.
