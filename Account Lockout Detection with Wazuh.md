# Account Lockout Detection with Wazuh

## Objective

Simulate a brute force attack against an Active Directory user account, trigger the domain lockout policy, and detect the incident using Wazuh SIEM. Document the full forensic chain from failed logon attempts through account lockout to SIEM alerting.

---

## Environment

| Component | Details |
|-----------|---------|
| Host | Windows 11 — VirtualBox |
| Domain Controller | Windows Server 2022  DC01.lab.local |
| SIEM | Ubuntu 22.04 LTS  Wazuh 4.7.5 (192.168.20.10) |
| Network | VirtualBox Internal Network  labnet (192.168.20.0/24) |
| AD Domain | lab.local |
| Target Account | jsmith (John Smith)  OU=IT |

> This lab is part of a Home SOC environment built from scratch:
> - Active Directory domain with Windows Server 2022 Domain Controller
> - Wazuh SIEM integrated with Windows Security Event logging
> - Network connectivity validated using tcpdump, PowerShell, and Linux networking tools

---

## Attack Simulation

**Lockout policy configured on lab.local:**

| Policy | Value |
|--------|-------|
| LockoutThreshold | 5 failed attempts |
| LockoutDuration | 30 minutes |
| LockoutObservationWindow | 30 minutes |

<img width="958" height="773" alt="PasswordPolicy" src="https://github.com/user-attachments/assets/ee295e86-ae72-45ba-914f-81144148c9fe" />


The attack was simulated using `runas`  a legitimate Windows binary , to generate repeated failed authentication attempts against `jsmith@lab.local`. This mirrors real-world credential stuffing or password spraying behavior where an attacker attempts known or common passwords against a domain account.

**Command used:**
```powershell
runas /user:jsmith@lab.local "cmd.exe"
```

Each execution prompted for a password. An incorrect password was entered 6 times in succession.

**Result after 6th attempt:**
```
RUNAS ERROR: Unable to run - cmd.exe
1909: The referenced account is currently locked out and may not be logged on to.
```

Account lockout confirmed via PowerShell:
```powershell
Get-ADUser -Identity jsmith -Properties LockedOut | Select-Object Name, LockedOut
```
```
Name         LockedOut
----         ---------
John Smith   True
```

<img width="1037" height="767" alt="locked password" src="https://github.com/user-attachments/assets/5dd3d43d-7459-4160-9090-21864b81a27a" />


---

## Wazuh Alerts

Wazuh Agent was installed and running on DC01 prior to the simulation, forwarding Windows Security Events to the Ubuntu Manager in real time.

<img width="1023" height="772" alt="Wazuh running" src="https://github.com/user-attachments/assets/1ccd94f6-5695-481c-ab47-3443d7d2f4c1" />


Wazuh detected the full attack chain:

| Timestamp | Rule ID | Description | Level |
|-----------|---------|-------------|-------|
| 6/23/2026 20:23:48 | 60122 | Logon failure - Unknown user or bad password | 5 |
| 6/23/2026 20:24:27 | 60122 | Logon failure - Unknown user or bad password | 5 |
| 6/23/2026 20:24:32 | 60122 | Logon failure - Unknown user or bad password | 5 |
| 6/23/2026 20:24:36 | 60122 | Logon failure - Unknown user or bad password | 5 |
| 6/23/2026 20:24:42 | 60115 | User account locked out (multiple login errors) | **9** |

<img width="1296" height="835" alt="alerta de locked nivel 09" src="https://github.com/user-attachments/assets/35a1e97a-422c-4793-86aa-11bb6d88a4ed" />


**Dashboard summary:**
- Total alerts: 765
- Authentication failures: 6
- Agent source: DC01

---

## Forensic Evidence

### Attack Timeline

| Time | Event ID | Description |
|------|----------|-------------|
| 20:23:48 | 4625 | Logon failure — bad password (attempt 1) |
| 20:24:27 | 4625 | Logon failure — bad password (attempt 2) |
| 20:24:32 | 4625 | Logon failure — bad password (attempt 3) |
| 20:24:36 | 4625 | Logon failure — bad password (attempt 4) |
| 20:24:42 | 4740 | Account locked out — threshold reached |
| 20:24:42 | — | Wazuh Rule 60115 fired (Level 9) |

### Windows Security Event Log — Event ID 4740

The domain controller logged a native Windows security event confirming the lockout:

| Field | Value |
|-------|-------|
| Event ID | 4740 |
| Description | A user account was locked out |
| Account That Was Locked Out | jsmith |
| Security ID | LAB\jsmith |
| Caller Computer Name | DC01 |
| Timestamp | 6/23/2026 6:25:19 PM |
| Computer | DC01.lab.local |
| Task Category | User Account Management |
| Keywords | Audit Success |

<img width="1031" height="773" alt="Event ID 4740" src="https://github.com/user-attachments/assets/42bac0fd-05b4-4fce-8bca-3bc01f66b4ac" />


**Key forensic fields:**

`Account Name: jsmith` , identifies the targeted account

`Caller Computer Name: DC01` , confirms the lockout was triggered from the domain controller itself, consistent with local authentication attempts via `runas`. In a real attack, this field would show the attacker's machine — an immediate pivot point for investigation.

`Security ID: LAB\jsmith` , full domain context of the locked account

### Wazuh Rule Analysis

| Rule | Maps To | Significance |
|------|---------|--------------|
| 60122 | Event ID 4625 | Individual logon failure , bad password |
| 60115 | Event ID 4740 | Account lockout , threshold reached |

Rule 60115 firing at **Level 9** requires immediate triage in a real SOC environment. A single account lockout can indicate password spraying, credential stuffing, or a targeted attack against a specific user.

---

## Incident Summary

At 20:23 on 23 June 2026, Wazuh began detecting repeated logon failures against domain account `jsmith@lab.local` originating from DC01. Within 90 seconds, 6 failed authentication attempts were recorded — exceeding the domain lockout threshold of 5.

At 20:24:42, Wazuh fired Rule 60115 (Level 9): **User account locked out (multiple login errors)**. Simultaneously, Windows Security Event ID 4740 was generated on DC01, confirming the lockout of `LAB\jsmith`.

The attack pattern is consistent with **MITRE ATT&CK T1110.001 — Brute Force: Password Guessing**, where an attacker systematically attempts passwords against a known account. The use of `runas` — a built-in Windows binary — mirrors living-off-the-land techniques that avoid triggering endpoint detection tools focused on malicious software.

The account remained locked for 30 minutes per policy, preventing further authentication attempts during that window.

---

## MITRE ATT&CK Mapping

| Tactic | Technique | ID | Observable |
|--------|-----------|-----|------------|
| Credential Access | Brute Force: Password Guessing | T1110.001 | Multiple Event ID 4625 before lockout , 4 failures within 90 seconds |

---

## Recommendations

**1. Build correlation rules for spray detection**
A single lockout is noise. Create detection logic for multiple accounts locked within a 10-minute window , that pattern indicates an active spray attack requiring immediate response.

**2. Investigate source hosts generating repeated Event ID 4625**
The `Caller Computer Name` field in Event ID 4740 identifies the machine generating failed authentications. In a real attack, that host should be immediately isolated and reviewed for credential dumping tools or lateral movement activity.

**3. Review authentication logs for post-lockout activity**
After an account is locked, attackers may pivot to other accounts. Correlate lockout events with subsequent successful logons from the same source — that sequence is a strong indicator of compromise.

**4. Reduce lockout threshold in high-security environments**
5 attempts is standard. Consider 3 in environments with privileged accounts — fewer attempts means less opportunity for guessing before detection.

**5. Alert on first lockout immediately**
Configure Wazuh Rule 60115 to trigger a ticket automatically. Every lockout deserves analyst review — the question is always whether it was the user or an attacker.

---

## Lessons Learned

**Technical:**
- Windows Security Event ID 4740 is the ground truth for account lockouts, Wazuh Rule 60115 maps directly to it with no custom rules required
- Wazuh Agent on Windows Server requires both network connectivity (labnet) and correct Manager IP in `ossec.conf` to forward events
- Network troubleshooting with `tcpdump` revealed that routing asymmetry (receiving on `enp0s8`, responding on `enp0s3`) was causing silent packet drops — not firewall rules as initially suspected
- ARP table confirmed Layer 2 connectivity before Layer 3 issues were identified, which narrowed the problem significantly
- `Test-NetConnection -Port 1514` is more useful than `ping` for validating Wazuh agent connectivity specifically

**Detection:**
- `runas` with wrong credentials generates Event ID 4625 per attempt and 4740 on lockout, both captured by Wazuh out of the box
- Level 9 alerts in Wazuh are above the standard noise threshold and require immediate attention in a real SOC

**SOC relevance:**
- Account lockout detection is a Day 1 skill for SOC analysts , it appears in almost every Windows environment and is a common indicator of credential attacks
- The investigation chain — failed logins → lockout alert → Event Viewer confirmation → SIEM correlation — mirrors real incident response workflow
