# Windows Fundamentals  + Active Directory Basics
**Platform:** TryHackMe  Cyber Security 101 path
**Category:** Operating Systems  Windows + Identity Management
**Completed as:** Single study block

---

## What I learned

Windows is a closed-source operating system created by Microsoft in 1985. Unlike Linux, users cannot access or modify the source code. It is the most widely used OS in corporate environments — which means that as a security professional, understanding Windows is not optional.

The key insight is that Windows is built around **users, permissions, and policies**. Most corporate attacks do not break through firewalls — they abuse misconfigured user accounts, weak passwords, or excessive permissions. Active Directory is the system that manages all of this at scale, which is why compromising it means compromising the entire organisation.

---

## Commands & tools — what they do and WHY they matter

| Command | What it does | Why a security professional uses it |
|---|---|---|
| `msinfo32` | Opens System Information | First step when auditing a machine — reveals OS version, installed software, and running services that may be outdated or vulnerable |
| `compmgmt.msc` | Opens Computer Management | Central console for disk management, local users, groups, and event logs — one place to see a lot |
| `taskmgr` | Opens Task Manager | Monitor running processes — an unexpected process consuming CPU from an unusual path (e.g. System32) is a red flag |
| `Win + R` | Opens Run Dialog Box | Fastest way to launch admin tools without navigating menus — used constantly in incident response |
| `ipconfig` | Shows network config | Identify the machine's IP, subnet, and gateway — essential first step when landing on a new machine |
| `netstat -a -b -e` | Shows active connections | `-a` shows all open ports, `-b` shows which executable owns each connection, `-e` shows traffic stats — powerful for detecting suspicious outbound connections |
| `whoami` | Shows current user | Know your privilege level immediately — are you a standard user or an admin? |
| `hostname` | Shows machine name | Identify the machine on the network — useful when pivoting across multiple hosts |
| `net user` | Lists all local accounts | Audit who has access to the machine — unexpected accounts are a sign of compromise |
| `net help` | Lists available net commands | Reference for user and group management commands |
| `control /name Microsoft.WindowsUpdate` | Opens Windows Update | Unpatched systems are the most common entry point — this is how you check and enforce patching |

> **The reasoning:** In Linux we use commands like `whoami`, `ip a`, and `cat /etc/passwd` to understand a machine. In Windows the equivalents are `whoami`, `ipconfig`, and `net user`. The logic is the same — always start by understanding where you are, who you are, and what is running.

---

## Windows Security — the security dashboard

Windows Security is the central panel to monitor and manage the health of a device. During this study session I opened my own dashboard (VM) and found **3 active warnings**:

| Feature | Status on my machine | What it means |
|---|---|---|
| Virus & Threat Protection | ⚠️ Warning | OneDrive backup for ransomware recovery not configured |
| Firewall & Network Protection | ✅ OK | No action needed |
| App & Browser Control | ⚠️ Warning | Blocking of potentially unwanted apps was OFF — turned it ON |
| Account Protection (UAC) | ⚠️ Warning | Microsoft account not linked for enhanced protection |

> **Real-world relevance:** Finding and fixing these misconfigurations on my own machine is exactly what an endpoint security audit looks like in a corporate environment. App & Browser Control being OFF means any downloaded file could execute without a warning — a common ransomware entry point.

**Why UAC matters:** UAC requires explicit user approval before any program can make system-level changes. Without it, malware downloaded accidentally would have immediate administrative access to the machine — no prompt, no warning. In corporate environments, UAC must never be disabled for convenience.

---

## Important system locations

| Path | Purpose | Security relevance |
|---|---|---|
| `C:\Windows\System32` | Critical Windows executables and DLL files | Malware hides here to appear legitimate — always investigate unexpected processes running from this path |
| `C:\Users` | Home directories for all local users | May contain documents, browser-saved credentials, SSH keys |
| `C:\Program Files` | Installed applications | Unauthorised software installed here may indicate a compromised machine |
| Event Viewer (via compmgmt.msc) | System and security logs | Key for incident response — shows login attempts, service failures, and policy changes |
| Disk Management (via compmgmt.msc) | Storage and partition management | Also covers Windows Server Backup — important for recovery planning after ransomware |

---

## Active Directory — the backbone of corporate Windows

Active Directory (AD) centralises the management of every user, computer, and permission across an organisation. Understanding AD is essential because it is the most common target in enterprise attacks — compromising AD means compromising everything.

| Concept | What it is | Why it matters in security |
|---|---|---|
| Windows Domain | A network where all users and computers are managed centrally | Without a domain, each machine would need separate administration — impossible at scale |
| Domain Controller (DC) | The server running AD and authenticating all users | The highest-value target in any corporate network — if the DC falls, everything falls |
| Security Groups — Builtin | Default groups available on any Windows host (Administrators, Users, Guests) | Misconfigured group membership is one of the most common privilege escalation paths |
| Machine accounts | When a computer joins a domain it gets its own account | Format: `MACHINENAME$` — example: `claude01$` — the `$` identifies it as a machine, not a human user |
| Managed Service Accounts (MSAs) | Special accounts used by services and applications to run automatically | Often have elevated privileges — a common attacker target if permissions are too broad |

> **To review:** Group Policy Objects (GPOs) — how they enforce security settings (password complexity, screen lock, software restrictions) across all machines in a domain simultaneously. Also: how MSAs differ from regular service accounts in terms of password management and attack surface.

---

## Linux vs Windows — security comparison

| Topic | Linux | Windows |
|---|---|---|
| Source code | Open source — fully inspectable | Closed source — trust the vendor |
| User accounts | `/etc/passwd` and `/etc/shadow` | Local Security Authority (LSA) + Active Directory |
| Privilege escalation | `sudo`, SUID binaries, misconfigured permissions | UAC bypass, misconfigured AD groups, token impersonation |
| Remote access | SSH (`ssh user@ip`) | RDP (Remote Desktop Protocol), WinRM, PsExec |
| Network info | `ip a`, `ifconfig` | `ipconfig`, `netstat` |
| Process inspection | `ps aux`, `top` | Task Manager, `tasklist` |
| Log location | `/var/log/auth.log` | Event Viewer — Security logs |
| Common attack path | Weak SSH, SUID abuse, cron jobs | AD misconfig, unpatched SMB, phishing |

> **Key insight:** Most corporate environments run both — Linux servers in the cloud and Windows desktops for employees. A security professional needs to be comfortable in both.

---

## Real-world application

**Evidence:** A corporate Windows environment with misconfigured Active Directory — users in the wrong security groups, UAC disabled, or unpatched systems — is one of the most common and most damaging attack surfaces in enterprise security.

**Risk level:** High

**Impact:** An attacker who gains access to a standard user account can exploit misconfigured group memberships or Managed Service Accounts to escalate privileges, pivot to the Domain Controller, and take control of every machine in the organisation. This is exactly the attack chain practiced in the Metasploit and Blue rooms on TryHackMe.

**How I would mitigate it:**
- Audit group memberships regularly — `net user [username]` shows what groups a user belongs to
- Never disable UAC — not even temporarily
- Enable App & Browser Control in Windows Security on every endpoint
- Enforce Windows Update — unpatched systems (especially SMB vulnerabilities) are the most exploited entry point
- Restrict who can authenticate directly to the Domain Controller
- Monitor Event Viewer security logs for failed logins, unusual times, and lateral movement indicators

---

## Quality checklist
- [x] Can explain what Active Directory is and why companies depend on it
- [x] Understands why the Domain Controller is the highest-value target in a corporate network
- [x] Can explain UAC and articulate the risk of disabling it
- [x] Knows the key CMD commands, what they reveal, and why each one matters
- [x] Opened Windows Security on own machine, identified 3 real misconfigurations, and fixed them
- [x] Can compare Windows and Linux from a security perspective
- [ ] To review: Group Policy Objects (GPOs) and Managed Service Accounts (MSAs) in depth

---

## Portfolio connections
- **Google Cybersecurity Certificate** — endpoint security and identity management concepts align directly
- **AZ-900** — Azure Active Directory (Microsoft Entra ID) uses the same AD principles in the cloud
- **VirtualBox Lab** — the Windows CMD commands (`ipconfig`, `netstat`, `whoami`) are used during post-exploitation after pivoting from Kali to a Windows target
- **Linux Fundamentals** — direct comparison between the two OS environments strengthens understanding of both
- **Metasploit rooms** — the AD misconfigurations studied here are the exact attack surface exploited in the Blue room (EternalBlue / MS17-010)
