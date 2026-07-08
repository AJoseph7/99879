# Phishing Email Analysis  Microsoft Account Impersonation

## Executive Summary

A phishing email impersonating Microsoft Security was statically analysed using email header inspection, DNS record verification, IP and domain threat intelligence, and IOC extraction. Multiple independent indicators confirmed a credential harvesting campaign targeting Microsoft account holders. SPF, DKIM, and DMARC all failed; the originating IP was flagged by 14 threat intelligence vendors as a known Tor exit node with active malicious history; and the sender domain was registered 67 days prior to delivery and has since been suspended for abuse. No malicious content was executed during the investigation.

---

## Skills Demonstrated

- Email header analysis (Received chain, Return-Path, Reply-To, Authentication-Results)
- SPF / DKIM / DMARC interpretation and comparison against legitimate domain configuration
- IP reputation investigation via threat intelligence
- Domain WHOIS analysis
- Base64 decoding
- IOC extraction and documentation
- Detection opportunity mapping
- Threat verdict and MITRE ATT&CK mapping

## Tools Used

Kali Linux · `cat` · `whois` · `dig` · CyberChef · VirusTotal

---

## Objective

Analyse a suspicious email sample to determine whether it represents a genuine phishing attempt or a false positive, using static header analysis, authentication record verification, IP and domain reputation lookups, and body decoding to extract indicators of compromise and produce a structured SOC investigation report.

---

## Environment

| Component | Details |
|---|---|
| Analysis Machine | Kali Linux 2026.1 (192.168.20.11) |
| Email Sample | phishing-sample.eml |
| Tools | cat, whois, dig, CyberChef (gchq.github.io), VirusTotal |

> **Safety Notice:** This investigation was performed entirely through static analysis. No URLs were visited. No attachments were executed. No active interaction with attacker infrastructure occurred at any point during the investigation.

---

## Investigation Workflow

```
1. Review email metadata (From, To, Subject, Date, X-headers)
2. Analyse SMTP Received chain (hop-by-hop routing)
3. Validate SPF / DKIM / DMARC authentication results
4. Investigate IP reputation (VirusTotal, threat intel feeds)
5. Investigate domain reputation (WHOIS, VirusTotal)
6. Compare against legitimate domain DNS configuration
7. Decode message body (Base64 → CyberChef)
8. Extract IOCs
9. Map detection opportunities
10. Produce final verdict and MITRE mapping
```

---

## Attack Flow

```
Attacker (185.220.101.47 — Tor Exit Node)
         │
         │  PHPMailer via suspicious-relay.xyz
         ▼
Victim Inbox (john.smith@acmecorp.com.au)
         │
         │  Email received — SPF/DKIM/DMARC all fail
         ▼
SOC Investigation
         │
    ┌────┴────┐
    │         │
Headers    Body (Base64)
    │         │
    ▼         ▼
SPF fail   Credential
DKIM fail  harvesting URL
DMARC fail /verify?session=...
    │
    ▼
WHOIS → 67-day-old domain, suspended
    │
    ▼
VirusTotal → 14/92 malicious (IP), 12/92 (domain)
    │
    ▼
Verdict: TRUE POSITIVE — Confirmed Phishing
```

---

## Email Overview

| Field | Value |
|---|---|
| From | "Microsoft Security Team" \<security@micros0ft-login.com\> |
| To | john.smith@acmecorp.com.au |
| Reply-To | support.recovery@protonmail.com |
| Return-Path | bounce@suspicious-relay.xyz |
| Subject | [ACTION REQUIRED] Unusual Sign-In Activity Detected on Your Microsoft Account |
| Date | Tue, 14 May 2024 09:14:29 -0700 |
| X-Originating-IP | 185.220.101.47 |
| X-Mailer | PHPMailer 6.1.5 |
| X-Spam-Score | 8.4 |
| Importance | High |

---

## Header Analysis

### Received Chain

The email passed through two servers before delivery, both resolving to the same IP address:

```
Received: from mail-out.suspicious-relay.xyz [185.220.101.47]
Received: from webmail.micros0ft-login.com [185.220.101.47]
```

A legitimate Microsoft email would originate from Microsoft-owned infrastructure (e.g., `protection.outlook.com`). Both hops here resolve to `185.220.101.47` — the relay and the alleged Microsoft mail server are the same host, indicating a fabricated mail chain.

### From / Reply-To / Return-Path Discrepancy

```
From:         security@micros0ft-login.com   ← lookalike domain (0 instead of o)
Reply-To:     support.recovery@protonmail.com ← attacker-controlled inbox
Return-Path:  bounce@suspicious-relay.xyz     ← attacker-controlled relay
```

Three different addresses, none affiliated with Microsoft. Any victim clicking Reply would send their response directly to the attacker.

### X-Mailer

```
X-Mailer: PHPMailer 6.1.5
```

PHPMailer is an open-source PHP library commonly used to send bulk phishing emails programmatically. Microsoft's mail infrastructure does not use PHPMailer.

---

## Authentication Analysis

```
dkim=fail   (signature verification failed)
spf=fail    (185.220.101.47 not designated as permitted sender)
dmarc=fail  (p=NONE sp=NONE dis=NONE)
```

| Mechanism | Result | Meaning |
|---|---|---|
| SPF | FAIL | Sending IP not authorised by domain DNS |
| DKIM | FAIL | No valid cryptographic signature present |
| DMARC | FAIL (p=NONE) | No enforcement policy — email delivered regardless |

**Comparison with legitimate Microsoft domain:**

```
dig txt _dmarc.microsoft.com
→ "v=DMARC1; p=reject; pct=100; rua=mailto:itex-rua@microsoft.com"
```

The real `microsoft.com` uses `p=reject` with `pct=100` — any failing email is rejected outright. The phishing domain uses `p=NONE`, meaning it was configured purely to allow email sending without any authentication infrastructure. This contrast alone is sufficient to exclude a legitimate misconfiguration scenario.

---

## IP Reputation — 185.220.101.47

VirusTotal: **14/92 security vendors flagged as malicious**

| Vendor | Verdict |
|---|---|
| Abusix | Malicious |
| ADMINUSLabs | Malicious |
| BitDefender | Phishing |
| G-Data | Phishing |
| SOCRadar | Phishing |
| Sophos | Malware |
| VIPRE | Malware |
| Webroot | Malicious |

**Guardpot threat intelligence:**
- 65 malicious events across 5 attack vectors
- First seen: 2024-10-16 — Last active: 2026-05-31
- SQL injection (36), SSH brute force (12), TCP scanning (6), web app brute force (1)

This IP is a known **Tor exit node**. Threat actors route traffic through Tor to anonymise their infrastructure. The combination of Tor exit node status and documented multi-vector attack history makes this a high-confidence malicious indicator.

---

## Domain Analysis — micros0ft-login.com

| Field | Value |
|---|---|
| Creation Date | 2026-04-29 (**67 days old**) |
| Registrar | DIAMATRIX C.C. (domains.co.za) |
| Domain Status | clientHold — **suspended by registrar** |
| DNSSEC | Unsigned |
| Registrant | Hidden via WHOIS privacy service |

VirusTotal: **12/92 vendors flagged as malicious or phishing** (BitDefender, Fortinet, G-Data, CyRadar, Forcepoint ThreatSeeker)

The domain was registered 67 days before delivery — newly registered domains under 90 days are a standard phishing indicator. The `clientHold` status confirms it was suspended for abuse after the campaign was reported. The domain name itself uses a homoglyph substitution: zero (`0`) replacing the letter `O` in `microsoft`.

---

## Body Analysis

The email body was Base64-encoded. Decoded via CyberChef (From Base64):

```
Dear Microsoft User,

We detected unusual sign-in activity on your Microsoft account.

Sign-in Details:
  Date: May 14, 2024  Time: 16:12 UTC
  Location: Khakov, Ukraine
  IP Address: 185.220.101.47
  Device: Unknown Windows Device

Verify My Account Now:
http://micros0ft-login.com/verify?session=884f7c3bc954b4fa4949f3f536a33d2a

If you do not verify within 24 hours, your account will be suspended.
```

The message uses urgency manipulation — unrecognised login from Ukraine, 24-hour deadline, account suspension threat — designed to bypass critical thinking.

The phishing URL contains a **per-victim session token**:
```
http://micros0ft-login.com/verify?session=884f7c3bc954b4fa4949f3f536a33d2a
```

Unique tokens allow the attacker to track which targets clicked, serve personalised credential harvesting pages, and identify captured accounts. The URL uses HTTP (not HTTPS).

---

## Indicators of Compromise (IOCs)

| Type | Value | Verdict |
|---|---|---|
| IP Address | 185.220.101.47 | Malicious — Tor exit node, 14/92 VT detections |
| Domain | micros0ft-login.com | Malicious — 12/92 VT, 67 days old, suspended |
| URL | http://micros0ft-login.com/verify?session=884f7c3bc954b4fa4949f3f536a33d2a | Phishing — credential harvesting |
| Email | security@micros0ft-login.com | Spoofed Microsoft identity |
| Return-Path | bounce@suspicious-relay.xyz | Attacker-controlled relay |
| Reply-To | support.recovery@protonmail.com | Attacker-controlled inbox |

### Detection Opportunities

| IOC | Detection Method |
|---|---|
| Tor Exit Node (185.220.101.47) | Firewall / threat intel feed block |
| Lookalike domain (micros0ft-login.com) | Email gateway homoglyph detection |
| PHPMailer X-Mailer header | SIEM rule — alert on PHPMailer in enterprise email |
| Reply-To ≠ From domain | Secure Email Gateway policy |
| Newly registered domain (< 90 days) | Threat intel feed / DNS reputation filter |
| SPF + DKIM + DMARC all failing | Email gateway quarantine rule |

---

## Incident Classification

| Field | Value |
|---|---|
| Severity | Medium |
| Confidence | High |
| Attack Type | Credential Harvesting |
| Impact | Account Compromise (if user clicks and submits credentials) |
| Likelihood | High |
| Status | **Confirmed Phishing — True Positive** |

---

## MITRE ATT&CK Mapping

| Field | Value |
|---|---|
| Tactic | Initial Access |
| Technique | T1566 — Phishing |
| Sub-technique | **T1566.002  Spearphishing Link** |

T1566.002 covers phishing emails containing a malicious link designed to harvest credentials. The email delivers a direct link to a credential harvesting page rather than a malicious attachment, distinguishing it from T1566.001 (Spearphishing Attachment).

---

## Recommendations

- Deploy DMARC `p=reject; pct=100` on all corporate email domains — consistent with the real `microsoft.com` configuration. This would block this email at the gateway before delivery.
- Block identified IOCs — IP `185.220.101.47` and domain `micros0ft-login.com` — in email gateway, proxy, and firewall.
- Configure email gateway rules to flag emails where `From` and `Reply-To` domains differ, and where `Return-Path` is on a third unrelated domain.
- Create SIEM detection rule for `PHPMailer` in the `X-Mailer` header — rarely used by legitimate enterprise senders.
- Implement URL filtering to block newly registered domains (under 90 days) at the proxy level.
- Run user awareness training focused on inspecting actual email addresses (not display names) and recognising homoglyph domains.

---

## Lessons Learned

### Technical

- All three authentication mechanisms (SPF, DKIM, DMARC) failing simultaneously on an email purportedly from a major corporation is a near-certain phishing indicator. A single SPF failure can sometimes reflect a legitimate misconfiguration; all three failing together does not.
- The `auid` equivalent in email forensics is the `Received` chain — reading it from bottom to top reveals the true originating server, regardless of what the `From` field displays.
- Per-victim session tokens in phishing URLs are a tracking mechanism. In an enterprise incident, these tokens can help identify all affected users if the attacker's server logs are obtained.
- DMARC `p=NONE` on a domain claiming to be a major corporation is an immediate red flag. Cross-referencing the legitimate domain's DMARC policy is a quick and definitive check.

### Operational

- Static analysis is sufficient to confirm phishing in Tier 1 triage. This entire investigation required no sandbox, no URL visit, and no code execution — only header reading, WHOIS, dig, VirusTotal, and CyberChef.
- Newly registered domains (under 90 days) warrant automatic escalation. A 67-day-old domain impersonating Microsoft is not a false positive candidate.
- Display names are trivially spoofed and should never be used as the basis for trusting an email. The actual email address and domain must always be inspected.

### SOC Takeaways

- In a real SOC environment, this email would be triaged as a True Positive within minutes using the same workflow documented here. The combination of authentication failures, threat intel hits, and a suspicious phishing URL provides high-confidence attribution without requiring deeper analysis.
- The detection opportunities table above maps directly to tuning recommendations for the email gateway and SIEM — moving from investigation to prevention is the expected output of a phishing triage in Tier 1.
- Documenting the full investigation chain (not just the verdict) is essential for escalation, legal hold, and informing the threat intelligence team about new IOCs.
