# SSH Brute Force Detection with Wazuh

## Objective
Simulate a brute force attack against SSH and detect it using Wazuh SIEM, 
investigating the evidence as a SOC analyst would.

---

## Environment
| Component | Details |
|-----------|---------|
| Attacker | Kali Linux 2026.1 — 192.168.20.11 |
| Target/SIEM | Ubuntu 22.04 LTS — 192.168.20.10 |
| SIEM | Wazuh 4.7.5 |
| Network | Isolated internal network — labnet |

---

## Attack Simulation
Tool: Hydra — password brute force against SSH service.

```bash
hydra -l vboxuser -P /usr/share/wordlists/rockyou.txt ssh://192.168.20.10 -t 4 -V
```

Duration: ~59 minutes. Wordlist: rockyou.txt (14 million passwords).

---

## Wazuh Alerts

| Rule ID | Description | Level |
|---------|-------------|-------|
| 5760 | sshd: authentication failed | 5 |
| 2501 | syslog: User authentication failure | 5 |
| 5758 | Maximum authentication attempts exceeded | 8 |
| 2502 | User missed the password more than one time | 10 — Highest Severity |

**Total alerts: 3765 | Authentication failures: 3672 | Authentication success: 9**

<img width="1267" height="803" alt="3765 hits" src="https://github.com/user-attachments/assets/a776d361-f03a-47e0-8b27-673d0ff9737b" />


---

## Forensic Evidence — Rule 2502

| Field | Value |
|-------|-------|
| data.srcip | 192.168.20.11 |
| data.dstuser | vboxuser |
| rule.id | 2502 |
| rule.level | 10 |
| rule.firedtimes | 30 |
| decoder.name | sshd |
| location | /var/log/auth.log |
| full_log | PAM 2 more authentication failures; rhost=192.168.20.11 user=vboxuser |

<img width="1288" height="875" alt="Screenshot 2026-06-09 140638" src="https://github.com/user-attachments/assets/d1107997-e509-4a05-92c2-77501d0fe6ea" />


---

## Incident Summary
A brute force attack was detected against the SSH service on Ubuntu 22.04.

Source IP 192.168.20.11 (Kali) generated 3,672 authentication failures 
targeting the account "vboxuser" over approximately 59 minutes.

Multiple Wazuh rules were triggered, escalating from individual failures 
(Rule 5760, Level 5) to repeated pattern detection (Rule 2502, Level 10).

**Authentication success: 9** — investigated and confirmed as legitimate 
logins performed manually during the lab session (sudo commands, terminal 
access). Hydra did not successfully authenticate — "vboxuser" password 
was not present in the rockyou.txt wordlist.

No successful compromise was confirmed.

---

## MITRE ATT&CK Mapping

| Field | Value |
|-------|-------|
| Technique ID | T1110 |
| Technique | Brute Force |
| Tactic | Credential Access |

<img width="1282" height="862" alt="Screenshot 2026-06-09 141034" src="https://github.com/user-attachments/assets/5b99eda1-1b11-494f-aaa5-34c025b5c9d0" />


--------

## Recommendations
1. Disable password authentication — enforce SSH key-based login only
2. Install and configure Fail2Ban to automatically block repeated failures
3. Restrict SSH access via firewall — whitelist trusted IPs only
4. Change default SSH port from 22 to reduce automated scanning
5. Monitor Rule 2502 and 5758 alerts as high-priority indicators

---

## Lessons Learned
1. **Brute force leaves a clear signature.** 3,672 failures from a single 
   source IP in under an hour is unmistakable in any SIEM.
2. **Alert escalation matters.** Rule 5760 (single failure) escalates to 
   Rule 2502 (repeated pattern) — understanding the chain is key.
3. **Every anomaly needs an explanation.** The 9 authentication successes 
   required investigation to confirm they were legitimate, not attacker access.
4. **Tools need context.** Hydra ran for 59 minutes without success because 
   the target password wasn't in the wordlist — a real attacker would pivot 
   to a different strategy.
