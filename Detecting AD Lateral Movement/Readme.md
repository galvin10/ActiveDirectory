# Detecting AD Lateral Movement

**Date:** June 14, 2026

## Overview

Completed the **Detecting AD Lateral Movement** room on TryHackMe. This room focused on identifying how attackers move through an Active Directory environment after gaining initial access and how to detect those activities using Windows Event Logs, Sysmon, and Splunk.

## Key Concepts Learned

### What is Lateral Movement?

Lateral movement occurs when an attacker moves from one compromised system to other systems within the network to expand access, escalate privileges, and reach high-value targets such as Domain Controllers.

### Common Lateral Movement Techniques

#### Remote Desktop Protocol (RDP)

* Uses TCP port 3389.
* Allows interactive remote access to systems.
* Detection events:

  * Event ID 4624 (Successful Logon)
  * Logon Type 10 (Remote Interactive)

#### SMB & PsExec

* Uses SMB (TCP 445) for remote administration.
* Commonly leveraged by attackers for command execution.
* Detection indicators:

  * Event ID 4624 (Logon Type 3)
  * Service creation (Event ID 7045)
  * Process creation events (Sysmon Event ID 1)

#### Windows Management Instrumentation (WMI)

* Enables remote execution and administration.
* Frequently abused for stealthy movement.
* Detection:

  * WmiPrvSE.exe spawning suspicious child processes.
  * Sysmon Process Creation events.

#### PowerShell Remoting

* Uses WinRM (TCP 5985/5986).
* Allows remote command execution.
* Detection:

  * PowerShell operational logs.
  * Event ID 4688 for process creation.
  * WinRM activity logs.

#### Pass-the-Hash (PtH)

* Uses NTLM hashes instead of plaintext passwords.
* Allows authentication without knowing the password.
* Detection:

  * Unusual NTLM authentication activity.
  * Event ID 4624 Logon Type 3.
  * Authentication from unexpected hosts.

#### Pass-the-Ticket (PtT)

* Uses stolen Kerberos tickets.
* Bypasses password requirements entirely.
* Detection:

  * Abnormal Kerberos activity.
  * Event IDs 4768, 4769, and 4624.

---

## Important Windows Event IDs

| Event ID | Description                 |
| -------- | --------------------------- |
| 4624     | Successful Logon            |
| 4625     | Failed Logon                |
| 4648     | Explicit Credential Usage   |
| 4672     | Special Privileges Assigned |
| 4688     | Process Creation            |
| 4768     | Kerberos TGT Request        |
| 4769     | Kerberos TGS Request        |
| 7045     | Service Installed           |
| Sysmon 1 | Process Creation            |
| Sysmon 3 | Network Connection          |

---

## Investigation Workflow

### 1. Identify Initial Access

* Review authentication logs.
* Identify compromised accounts.
* Determine source systems.

### 2. Track Authentication Activity

* Monitor 4624 logons.
* Review Logon Types.
* Correlate timestamps and hosts.

### 3. Look for Remote Execution

* Service creation events.
* PsExec indicators.
* WMI and PowerShell execution.

### 4. Trace Privileged Accounts

* Identify Domain Admin activity.
* Review privileged logons.
* Monitor credential usage.

### 5. Correlate Events

* Authentication → Remote Execution → Privilege Escalation.
* Build the attack timeline.

---

## Key Takeaways

* Attackers rarely stop after initial compromise.
* Lateral movement often relies on legitimate administrative tools.
* Authentication logs are critical for investigations.
* Correlating multiple log sources provides visibility into attacker behavior.
* Monitoring RDP, SMB, WMI, WinRM, and Kerberos activity helps detect movement early.
* Event IDs 4624, 4688, 4768, 4769, and 7045 are especially valuable during investigations.

## Room Completed

**TryHackMe — Detecting AD Lateral Movement**
