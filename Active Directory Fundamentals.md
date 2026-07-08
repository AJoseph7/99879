# Active Directory Fundamentals   Building a Domain from Scratch

**Note:** This is reference documentation for the AD environment used in subsequent detection labs (Account Lockout Detection, etc.), not a standalone investigation. It exists so the setup is reproducible and verifiable  the actual SOC-relevant work starts in the labs that build on top of this.

## Objective

Deploy a functional Active Directory environment from zero using Windows Server 2022 and PowerShell. This lab covers domain controller promotion, organizational unit structure, user and group management, and Group Policy configuration  foundational skills for any SOC or sysadmin role operating in enterprise Windows environments.

**Environment:**
- Host: Windows 11  Dell Vostro 5490, i7-10510U, 16GB RAM
- Hypervisor: VirtualBox
- DC01: Windows Server 2022 Standard Evaluation  4GB RAM, 2 CPUs, 50GB disk
- Domain: `lab.local`
- Existing lab network: Kali Linux 2026.1 (192.168.20.11) + Ubuntu 22.04 (192.168.20.10) + Wazuh 4.7.5

---

## Why This Matters

Active Directory is the backbone of identity management in virtually every enterprise Windows environment. Understanding how AD works  how users are created, how policies are applied, how objects are organized  is essential for a SOC analyst. Most security incidents in corporate environments involve AD in some way: credential attacks, lateral movement, privilege escalation, and persistence mechanisms all interact with AD objects and policies. You cannot detect what you do not understand.

---

## Lab Setup   Windows Server 2022

Downloaded the Windows Server 2022 Standard Evaluation ISO directly from Microsoft (180-day free evaluation). Created the VM in VirtualBox with the following specs:

| Setting | Value |
|---|---|
| RAM | 4GB |
| CPUs | 2 |
| Disk | 50GB |
| Network | NAT (installation) |
| OS | Windows Server 2022 Standard Evaluation |

After installation, renamed the server to **DC01** using SConfig (option 2) and rebooted.

---

## Phase 1   Installing Active Directory Domain Services

### Step 1: Install the AD-DS Role

```powershell
Install-WindowsFeature -Name AD-Domain-Services -IncludeManagementTools
```

**Why:** Windows Server out of the box is just an OS  it has no domain functionality. This command installs the Active Directory Domain Services role, which gives the server the capability to become a Domain Controller. The `-IncludeManagementTools` flag installs the AD PowerShell module and RSAT tools needed to manage the domain via command line.

### Step 2: Promote the Server to Domain Controller

```powershell
Install-ADDSForest -DomainName "lab.local" -DomainNetbiosName "LAB" -InstallDns
```

**Why:** This is the command that actually creates the domain and promotes DC01 to a Domain Controller. `Install-ADDSForest` creates a new Active Directory forest , the top-level container for everything. `-DomainName "lab.local"` sets the fully qualified domain name. `-DomainNetbiosName "LAB"` sets the legacy short name used internally. `-InstallDns` deploys DNS alongside AD , this is mandatory because Active Directory relies entirely on DNS to locate domain controllers, authenticate users, and replicate data between DCs. After this command runs, the server reboots automatically.

---

## Phase 2   Creating Users

### Step 3: Create User jsmith (John Smith)

First attempt hit a syntax error  `-Enabled $true` was being parsed incorrectly due to line wrapping in the terminal. Corrected and rerun:

```powershell
New-ADUser -Name "John Smith" -GivenName "John" -Surname "Smith" `
  -SamAccountName "jsmith" `
  -UserPrincipalName "jsmith@lab.local" `
  -AccountPassword (ConvertTo-SecureString "<LabPassword>" -AsPlainText -Force) `
  -Enabled $true
```

**Why:** `New-ADUser` creates the user object in AD. `-SamAccountName` is the legacy login name used for authentication (this is what users type at the login screen). `-UserPrincipalName` is the modern UPN format , `user@domain`. `-AccountPassword` uses `ConvertTo-SecureString` to avoid passing a plain text password directly to the cmdlet. `-Enabled $true` activates the account immediately  by default, new accounts are created disabled.

**Verification:**

```powershell
Get-ADUser -Identity "jsmith"
```

```
DistinguishedName : CN=John Smith,CN=Users,DC=lab,DC=local
Enabled           : True
GivenName         : John
Name              : John Smith
ObjectClass       : user
ObjectGUID        : 6a7d1b5c-2e65-48d3-8c41-9f8c62ee7db7
SamAccountName    : jsmith
SID               : S-1-5-21-2075389030-3758795886-1683061984-1103
Surname           : Smith
UserPrincipalName : jsmith@lab.local
```

<img width="1042" height="802" alt="Primeiro usuário AD criado 002" src="https://github.com/user-attachments/assets/55e56bb3-ec2f-4063-9661-3584e080a103" />


### Step 4: Create User jdoe (Jane Doe)

```powershell
New-ADUser -Name "Jane Doe" -GivenName "Jane" -Surname "Doe" `
  -SamAccountName "jdoe" `
  -UserPrincipalName "jdoe@lab.local" `
  -AccountPassword (ConvertTo-SecureString "<LabPassword>" -AsPlainText -Force) `
  -Enabled $true
Enable-ADAccount -Identity "jdoe"
```

**Note on Enable-ADAccount:** Used as an extra safeguard after the earlier error on jsmith to ensure the account was active.

---

## Phase 3  Security Group

### Step 5: Create IT-Team Security Group

```powershell
New-ADGroup -Name "IT-Team" -GroupScope Global -GroupCategory Security
```

**Why:** Groups in AD are used to manage access at scale  instead of assigning permissions to individual users, you assign them to a group and add users to it. `-GroupScope Global` means the group can be referenced across any domain in the forest. `-GroupCategory Security` means it's used for access control and permissions, as opposed to a Distribution group which is only for email.

### Step 6: Add Members to IT-Team

```powershell
Add-ADGroupMember -Identity "IT-Team" -Members "jsmith","jdoe"
```

**Verification:**

```powershell
Get-ADGroupMember -Identity "IT-Team"
```

```
distinguishedName : CN=John Smith,CN=Users,DC=lab,DC=local
name              : John Smith
objectGUID        : 6a7d1b5c-2e65-48d3-8c41-9f8c62ee7db7
SamAccountName    : jsmith
SID               : S-1-5-21-2075389030-3758795886-1683061984-1103

distinguishedName : CN=Jane Doe,CN=Users,DC=lab,DC=local
name              : Jane Doe
objectGUID        : 408c33be-3588-477f-9269-5004beb8b2e4
SamAccountName    : jdoe
SID               : S-1-5-21-2075389030-3758795886-1683061984-1104
```

<img width="1026" height="770" alt="IT Team members" src="https://github.com/user-attachments/assets/d9be8fec-05a3-4e51-b876-71ccd28ca3e9" />


---

## Phase 4   Organizational Unit

### Step 7: Create OU=IT

```powershell
New-ADOrganizationalUnit -Name "IT" -Path "DC=lab,DC=local"
```

**Why:** An Organizational Unit is a container within AD used to logically organize objects  users, computers, groups  by department, location, or function. OUs are important for two reasons: structure (easier to find and manage objects) and policy (Group Policy Objects can be applied to an OU, affecting everything inside it). Without OUs, you apply policies at the domain level, which affects everyone.

**First attempt to move users failed**  tried `Move-ADObject` before the OU was created, which produced: `The operation could not be performed because the object's parent is either uninstantiated or deleted`. Created the OU first, then moved:

### Step 8: Move Users into OU=IT

```powershell
Move-ADObject -Identity "CN=John Smith,CN=Users,DC=lab,DC=local" -TargetPath "OU=IT,DC=lab,DC=local"
Move-ADObject -Identity "CN=Jane Doe,CN=Users,DC=lab,DC=local" -TargetPath "OU=IT,DC=lab,DC=local"
```

**Why the DN syntax:** Active Directory uses Distinguished Names (DN) to identify every object by its full path in the directory tree. `CN=` is Common Name, `OU=` is Organizational Unit, `DC=` is Domain Component. So `CN=John Smith,CN=Users,DC=lab,DC=local` means: the object named "John Smith" inside the "Users" container inside the "lab.local" domain.

**Verification:**

```powershell
Get-ADUser -Filter * -SearchBase "OU=IT,DC=lab,DC=local"
```

```
DistinguishedName : CN=John Smith,OU=IT,DC=lab,DC=local
Enabled           : True
SamAccountName    : jsmith
UserPrincipalName : jsmith@lab.local

DistinguishedName : CN=Jane Doe,OU=IT,DC=lab,DC=local
Enabled           : True
SamAccountName    : jdoe
UserPrincipalName : jdoe@lab.local
```

Both users now show `OU=IT` in their Distinguished Name  confirming they were moved successfully.

<img width="1033" height="781" alt="OU IT  O AD 003" src="https://github.com/user-attachments/assets/d407de38-774b-474d-8a17-47eca6048536" />


---

## Phase 5 — Group Policy Object

### Step 9: Create and Link GPO

```powershell
New-GPO -Name "Password-Policy" | New-GPLink -Target "OU=IT,DC=lab,DC=local"
```

**Why:** A Group Policy Object is a set of rules that gets applied to users and computers within a defined scope. `New-GPO` creates the policy object. Piping it directly into `New-GPLink` links it to `OU=IT` in one command — this means every user and computer inside OU=IT will have this policy applied at login/refresh. Without the link, the GPO exists but does nothing.

Output confirmed:

```
GpoId       : f681df6a-8745-4eea-8d19-dd3bfd17aaca
DisplayName : Password-Policy
Enabled     : True
Target      : OU=IT,DC=lab,DC=local
Order       : 1
```

<img width="1007" height="783" alt="GPO criada e aplicada na OU=IT 004" src="https://github.com/user-attachments/assets/efc4c573-ef46-4c10-9918-b5ce26cba154" />


### Step 10: Configure Password Policy via Registry

```powershell
Set-GPRegistryValue -Name "Password-Policy" `
  -Key "HKLM\System\CurrentControlSet\Services\Netlogon\Parameters" `
  -ValueName "MaximumPasswordAge" `
  -Type DWord `
  -Value 30
```

**Why:** This pushes a registry value through the GPO to define that passwords expire after 30 days. `HKLM` is `HKEY_LOCAL_MACHINE` — the system-wide registry hive. `DWord` is a 32-bit integer data type. Value 30 = 30 days. The Netlogon Parameters path controls domain authentication settings on domain-joined machines.

**Final verification:**

```powershell
Get-GPO -Name "Password-Policy"
```

```
DisplayName     : Password-Policy
DomainName      : lab.local
Owner           : LAB\Domain Admins
GpoStatus       : AllSettingsEnabled
ModificationTime: 6/9/2026 7:17:28 PM
ComputerVersion : AD Version: 1, SysVol Version: 1
```

`ComputerVersion: AD Version: 1` confirms the GPO has been modified — a version of 0 would mean untouched.

<img width="958" height="762" alt="final conf GPO" src="https://github.com/user-attachments/assets/94c16bbe-4a6e-4eac-a9d0-a3490d4c92d1" />


---

## Errors Encountered and Resolved

| Error | Cause | Resolution |
|---|---|---|
| `New-ADUser: parameter 'Enabled$true' not found` | Missing space between `-Enabled` and `$true` due to terminal line wrap | Re-typed command with correct spacing |
| `Move-ADObject: object's parent is uninstantiated` | Tried to move users before OU=IT was created | Created OU first with `New-ADOrganizationalUnit`, then moved |
| `Get-GPReport: term not recognized` | `Get-GPReport` requires GPMC module not installed on Server Core | Used `Get-GPO` instead — confirmed GPO state via `ComputerVersion` field |

---

## Final Structure

```
lab.local
└── DC01.lab.local (Domain Controller)
    └── OU=IT
        ├── jsmith (John Smith) — IT-Team member
        ├── jdoe (Jane Doe)     — IT-Team member
        └── GPO: Password-Policy (MaximumPasswordAge: 30 days)
```

## Results Summary

| Component | Status | Details |
|---|---|---|
| Domain | ✅ Created | lab.local |
| Domain Controller | ✅ Promoted | DC01.lab.local |
| Users | ✅ Created | jsmith (John Smith), jdoe (Jane Doe) |
| Security Group | ✅ Created | IT-Team — Global Security |
| Organizational Unit | ✅ Created | OU=IT |
| Users in OU | ✅ Confirmed | Both users moved to OU=IT |
| GPO | ✅ Applied | Password-Policy → OU=IT, MaxPasswordAge: 30 days |

---

## Key Concepts Learned

**Distinguished Names (DN):** Every AD object has a unique path expressed as a DN. Understanding `CN=`, `OU=`, `DC=` notation is essential for scripting and troubleshooting — you'll see this format constantly in security logs, LDAP queries, and SIEM alerts.

**SID vs GUID:** Each user gets a Security Identifier (SID) — this is what Windows actually uses for access control decisions, not the username. The SID persists even if the account is renamed. This matters in forensics — logs will often show SIDs, not display names.

**GPO Scope and Inheritance:** GPOs apply based on where they're linked. Linking to OU=IT affects only that OU. If linked at the domain level, it affects everyone. Understanding GPO scope is critical for both hardening (applying CIS benchmarks) and attack detection (malicious GPO modifications are a known persistence technique — MITRE T1484).

**Default Containers vs OUs:** Users created without specifying a path land in `CN=Users` — the default container. Unlike OUs, default containers cannot have GPOs applied directly to them. Moving users to OUs is standard practice in any real environment.

---

## Next Lab

**Account Lockout Detection with Wazuh** — configure a lockout policy on DC01, trigger repeated failed logins, and detect Event ID 4740 (account lockout) in Wazuh. Maps to MITRE ATT&CK T1110 — Brute Force.
