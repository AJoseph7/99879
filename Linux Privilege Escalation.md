# Detecting Linux Privilege Escalation via Misconfigured sudoers using auditd and Wazuh

## Skills Demonstrated

- Linux privilege model (UID, EUID, AUID)
- `sudoers` misconfiguration identification
- Linux Audit (`auditd`) rule configuration and forensic analysis
- PAM log analysis
- SIEM alerting with Wazuh
- Syscall-level forensic tracing via `ausearch`
- MITRE ATT&CK technique mapping
- Threat detection and incident investigation

## Tools Used

Ubuntu 22.04 · Kali Linux · auditd · ausearch · sudo · SSH · Wazuh 4.7.5 · `/var/log/auth.log`

---

## Objective

Simulate a privilege escalation caused by a misconfigured `sudoers` entry and investigate the activity using `auditd` and Wazuh to correlate host-level forensic evidence with SIEM alerts.

## Environment

| Host | Role | IP | OS |
|---|---|---|---|
| Kali Linux 2026.1 | Attacker | 192.168.20.11 | Kali |
| Ubuntu 22.04 LTS | Target / Wazuh Manager | 192.168.20.10 (hostname: Ubuntu2) | Wazuh 4.7.5 |

All hosts reside on an isolated internal lab network (`labnet`), with no external connectivity.

## Vulnerability / Misconfiguration

A misconfiguration was introduced into `/etc/sudoers` (via `visudo`) for the local user `hacker` (uid 1001):

```
hacker ALL=(ALL) NOPASSWD: /bin/bash
```

This entry grants `hacker` the ability to obtain a full, interactive root shell without re-authenticating — a broad `NOPASSWD` rule granting unrestricted access to `/bin/bash`. This type of misconfiguration is commonly introduced when administrators grant wide sudo access instead of scoping rules to specific, necessary commands.

## Detection Setup (auditd)

An audit rule was configured on the Ubuntu host to track privilege escalation activity:

```
sudo /sbin/auditctl -a always,exit -F arch=b64 -S execve -F euid=0 -F auid!=0 -k privilege_escalation
```

| Flag | Meaning |
|---|---|
| `arch=b64` | Monitor 64-bit syscalls |
| `-S execve` | Track all program execution events |
| `-F euid=0` | Only events where the *effective* UID is 0 (root) |
| `-F auid!=0` | Exclude actions performed natively under root's own login session |
| `-k privilege_escalation` | Tag matching events for retrieval via `ausearch` |

This rule does not prevent escalation — it records a forensic trail that survives privilege changes, because the `auid` (audit user ID / login UID) field is set at login time and remains immutable for the entire session, even after `sudo` changes the effective and real UID.

**What this rule detects:**

| Scenario | Detected? |
|---|---|
| `sudo /bin/bash` | ✓ Yes |
| `sudo su` | ✓ Yes |
| `sudo vim` / `sudo nano` | ✓ Yes |
| Any command executed as euid=0 by a non-root login | ✓ Yes |
| Commands run by a user without sudo | ✗ No (euid stays non-root) |
| Direct root login | ✗ No (auid=0 is excluded) |

## Attack Simulation

```
[Kali — 192.168.20.11]
        │
        │  SSH (hacker / password123)
        ▼
[Ubuntu — 192.168.20.10]
        │
        │  sudo -l  →  NOPASSWD: /bin/bash confirmed
        │
        │  sudo /bin/bash
        ▼
   Root Shell (uid=0)
        │
        ▼
   auditd logs syscall → Wazuh ingests → Rule 5402 fired
```

Steps executed:

1. SSH from Kali into the Ubuntu host as `hacker`:  
   `ssh hacker@192.168.20.10`
2. Confirmed the sudoers misconfiguration: `sudo -l`
3. Escalated to a root shell: `sudo /bin/bash`
4. Verified privilege change: `whoami` / `id` → `uid=0(root)` (was `uid=1001(hacker)`)

<img width="953" height="816" alt="kali sudo binbash" src="https://github.com/user-attachments/assets/a401e927-b7b2-48fd-8a8c-fbbffb22b473" /> 

## Wazuh Alerts

Wazuh's sudo/PAM log decoder (reading `/var/log/auth.log`) generated the following alerts on each successful escalation:

| Rule | Description | Level |
|---|---|---|
| **5402** | Successful sudo to ROOT executed | 3 |
| 5501 | PAM: Login session opened | 3 |
| 5502 | PAM: Login session closed | 3 |

Rule 5402 fired consistently across the testing window (17:37, 17:39, 17:43, 17:53 on Jun 29, and again on Jun 30 after the host was restarted), confirming reliable detection across sessions.

<img width="1277" height="805" alt="wazuh-5402-events" src="https://github.com/user-attachments/assets/b73a4cec-9365-4a29-a8e4-f22fc749fd92" />

<img width="1290" height="821" alt="wazuh-5402-rule-detail png" src="https://github.com/user-attachments/assets/891731d8-b36f-4fdc-9441-c42f0a7a5446" />

**Note on Rule 5403:** Wazuh also ships Rule 5403 (*"First time user executed sudo"*), which fires exactly once per user account — on the very first successful sudo invocation ever recorded. Because `hacker`'s sudo access was already exercised while validating the misconfiguration (`sudo -l`) earlier in the lab setup, Rule 5403 had already fired before this documented incident window. This is expected behavior given how the rule is designed, not a detection gap.

## Forensic Evidence (auditd / ausearch)

On the Ubuntu host, the auditd trail was queried directly:

```
sudo ausearch -k privilege_escalation --interpret | grep -E "time|exe|auid|euid|comm"
```

Representative output:

```
type=SYSCALL ... auid=hacker uid=root gid=root euid=root suid=root ... comm=sh exe=/usr/bin/dash key=privilege_escalation
```

---

**Why `auid` matters:**

```
auid  = hacker   ← the human who authenticated via SSH
uid   = root     ← what sudo changed it to
euid  = root     ← effective UID at time of syscall
```

Even though the process is running as root, `auid` continues to read `hacker` for the entire session. This allows an investigator to definitively trace any privileged action — even commands executed after `sudo /bin/bash` — back to the original account. This is the core forensic value of `auditd` in this scenario: `auth.log` alone confirms *that* a sudo escalation occurred, but only `auditd` confirms *who* initiated it at syscall level.

---

<img width="1282" height="805" alt="ausearch-top" src="https://github.com/user-attachments/assets/0accc5e7-6e1f-4284-903b-be28d86768ff" />

<img width="1282" height="800" alt="ausearch-bottom" src="https://github.com/user-attachments/assets/6c674a2e-684e-4e95-aa30-a189da90f495" />

## Timeline

| Time | Event | Source |
|---|---|---|
| Jun 29 @ 17:37 | SSH login as `hacker` from 192.168.20.11 | Wazuh Rule 5501 |
| Jun 29 @ 17:37 | `sudo -l` — misconfiguration confirmed | Kali terminal |
| Jun 29 @ 17:37 | `sudo /bin/bash` — root shell obtained | Wazuh Rule 5402 / ausearch |
| Jun 29 @ 17:37 | Session closed | Wazuh Rule 5502 |
| Jun 29 @ 17:39–17:53 | Attack repeated across multiple sessions | Wazuh Rule 5402 (×3) |
| Jun 30 @ 21:14 | ausearch query confirming auid=hacker trail | Ubuntu2 terminal |
| Jun 30 @ 21:18 | Wazuh dashboard evidence captured | Ubuntu2 Firefox |

## Incident Summary

A user with low-privilege SSH access (`hacker`) was able to obtain a full root shell on the Ubuntu host due to an overly permissive `sudoers` entry granting `NOPASSWD` access to `/bin/bash`. The escalation was detected through two independent layers: Wazuh's sudo log decoder (Rule 5402, real-time SIEM alert) and Linux Audit's syscall-level tracking (host-level forensic trail via `ausearch`). Together, these confirm both *that* an escalation occurred and *who* the originating user was — even after the UID changed to root.

## MITRE ATT&CK Mapping

| Field | Value |
|---|---|
| Tactic | Privilege Escalation |
| Technique | T1548 — Abuse Elevation Control Mechanism |
| Sub-technique | **T1548.003 — Sudo and Sudo Caching** |

This sub-technique covers adversaries abusing the `sudoers` configuration to execute commands as another user (typically root) without re-authentication. It is intentionally distinct from **T1068 (Exploitation for Privilege Escalation)**, which requires exploitation of a software vulnerability or kernel/binary flaw. This scenario involves no such vulnerability — the system granted elevated access exactly as configured. The misconfiguration made that grant overly broad; there was no code being exploited.

## Recommendations

- Remove `NOPASSWD` entries from `/etc/sudoers` and `/etc/sudoers.d/`; require authentication for all sudo invocations.
- Scope sudo grants to specific, necessary commands only — never to `/bin/bash`, `/bin/sh`, or other shell binaries.
- Periodically audit `/etc/sudoers` and `/etc/sudoers.d/` for unauthorized or overly permissive entries.
- Consider `sudo` I/O logging (`Defaults log_input,log_output`) for full session-level visibility into privileged commands executed after escalation.
- Persist `auditd` rules under `/etc/audit/rules.d/` (loaded via `augenrules --load`) rather than relying on `auditctl` alone, which does not survive a reboot.

## Lessons Learned

**`auid` is the most reliable attribution field.** It does not change even after `sudo` modifies the effective and real UID. An investigator can always trace root-level activity back to the originating account as long as `auditd` is running and the session was initiated interactively.

**Broad audit filters generate noise.** The filter `-F auid!=0` also matches the "unset" sentinel value (`auid=-1`) used for non-interactive, daemon-spawned processes — not only real interactive users. In this lab, Wazuh's own internal subprocess execution (`df`, `sed`, `sort`, `last`) was tagged with the same `privilege_escalation` key as the actual attacker activity. A more precise rule targeting only real human logins would use:

```
-F auid>=1000 -F auid!=-1
