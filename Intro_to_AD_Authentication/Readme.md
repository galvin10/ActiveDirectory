# Intro to AD Authentication – Documentation

**Date:** June 20, 2026

## Overview

Completed the **Intro to AD Authentication** room, which covered the fundamentals of authentication in Active Directory environments, including the differences between Authentication and Authorization, NTLM authentication, Kerberos authentication, common AD authentication attacks, and defensive monitoring techniques.

---

# Learning Objectives

* Understand Authentication Materials used in Active Directory
* Differentiate Authentication vs Authorization
* Learn how NTLM authentication works
* Learn how Kerberos authentication works
* Understand Kerberos tickets and credential caches
* Explore common Active Directory authentication attacks
* Learn detection techniques using Windows Event Logs
* Understand mitigation strategies for authentication attacks

---

# Authentication Materials

Authentication materials are credentials used to prove identity within Active Directory.

### Common Authentication Materials

* Username and Password
* Certificates
* Password Hashes

The objective remains the same:

> Prove your identity to the domain before access is granted.

---

# Authentication vs Authorization

## Authentication

Authentication verifies identity.

Example:

```text
You are John
```

## Authorization

Authorization determines permissions.

Example:

```text
John can access the Finance Share
```

### Workflow

```text
Authentication → Authorization → Resource Access
```

Identity must be verified before permissions can be evaluated.

---

# NTLM Authentication

## What is NTLM?

NTLM (New Technology LAN Manager) is Microsoft's challenge-response authentication protocol.

Versions:

* NTLMv1 (Insecure)
* NTLMv2 (Improved but still vulnerable)

---

## NTLM Authentication Process

### Step 1

Client sends username to the target service.

### Step 2

Server generates a random challenge (nonce).

### Step 3

Client encrypts the challenge using the NT hash of the password.

### Step 4

Server forwards the challenge and response to the Domain Controller.

### Step 5

Domain Controller validates the response.

### Step 6

Access is granted if validation succeeds.

---

## NTLM Advantages

* Simple implementation
* No time synchronization required
* Works in workgroup environments
* Kerberos fallback option

---

## NTLM Weaknesses

* No mutual authentication
* Susceptible to relay attacks
* Vulnerable to Pass-the-Hash
* Weak cryptographic design
* Slower than Kerberos

---

## NTLM Authentication with Impacket

Example:

```bash
smbclient.py thm.loc/claire:'Password123!'@192.168.11.51
```

This authenticates using NTLM and provides SMB access if credentials are valid.

---

# Kerberos Authentication

## What is Kerberos?

Kerberos is the default authentication protocol used in modern Active Directory environments.

It uses a ticket-based authentication system.

---

# Kerberos Components

| Component | Purpose                    |
| --------- | -------------------------- |
| KDC       | Handles ticket requests    |
| AS        | Authentication Service     |
| TGS       | Ticket Granting Service    |
| TGT       | Ticket Granting Ticket     |
| ST        | Service Ticket             |
| SPN       | Service Principal Name     |
| KRBTGT    | Signs all Kerberos tickets |

---

# Kerberos Authentication Workflow

## Step 1 – AS-REQ

Client sends authentication request to KDC.

## Step 2 – AS-REP

KDC verifies identity and issues:

* Session Key
* TGT

## Step 3 – TGS-REQ

Client requests access to a service.

## Step 4 – TGS-REP

KDC issues a Service Ticket.

## Step 5 – AP-REQ

Client presents Service Ticket to target service.

Access is granted if validation succeeds.

---

# Kerberos Advantages

* Mutual authentication
* Single Sign-On (SSO)
* Better performance
* No password transmission
* Supports delegation
* Time-limited tickets

---

# Kerberos Weaknesses

* Requires clock synchronization
* Complex configuration
* Vulnerable to ticket theft
* Vulnerable to Kerberoasting
* KDC becomes a critical dependency

---

# Kerberos Credential Cache (ccache)

Kerberos tickets are stored in credential cache files.

Common location:

```bash
/tmp/krb5cc_<uid>
```

Useful commands:

```bash
klist
```

Environment variable:

```bash
export KRB5CCNAME=ticket.ccache
```

---

# Kerberos Authentication with Impacket

## Obtain TGT

```bash
getTGT.py thm.loc/mary:'Password123!' -dc-ip 192.168.11.100
```

## Export Ticket

```bash
export KRB5CCNAME=mary.ccache
```

## Authenticate Using Kerberos

```bash
smbclient.py thm.loc/mary@SERVER1.thm.loc -k -no-pass -dc-ip 192.168.11.100
```

---

# Active Directory Authentication Attacks

## 1. Weak Password Hashing

NTLM hashes can be cracked offline if passwords are weak.

### Example

```bash
hashcat -m 1000 hash.txt rockyou.txt
```

---

## 2. Pass-the-Hash (PtH)

Authenticate using NTLM hashes without knowing the password.

### Example

```bash
smbclient.py user@target -hashes LMHASH:NTHASH
```

---

## 3. Kerberoasting

Request service tickets and crack service account passwords offline.

### Enumeration

```bash
GetUserSPNs.py domain/user:password -request
```

### Crack Ticket

```bash
hashcat -m 13100 ticket.txt rockyou.txt
```

---

## 4. Golden Ticket Attack

Forge Kerberos TGTs using the KRBTGT hash.

### Example

```bash
ticketer.py -nthash KRBTGT_HASH -domain-sid DOMAIN_SID -domain domain.local Administrator
```

### Load Ticket

```bash
export KRB5CCNAME=Administrator.ccache
```

---

# Windows Event IDs for Detection

## Authentication Events

| Event ID | Description                         |
| -------- | ----------------------------------- |
| 4624     | Successful Logon                    |
| 4625     | Failed Logon                        |
| 4768     | Kerberos TGT Request                |
| 4769     | Kerberos Service Ticket Request     |
| 4771     | Kerberos Pre-Authentication Failure |

---

# Detecting Pass-the-Hash

Indicators:

* Event ID 4624
* Authentication Package = NTLM
* Logon Type = 3
* Missing Source Address

---

# Detecting Kerberoasting

Indicators:

* Large number of Event ID 4769
* RC4 Encryption (0x17)
* Multiple SPN requests in a short period

---

# Detecting AS-REP Roasting

Indicators:

* Multiple Event ID 4771 events
* Numerous authentication failures
* Activity across many accounts

---

# Mitigation Strategies

| Attack            | Mitigation                                                |
| ----------------- | --------------------------------------------------------- |
| Pass-the-Hash     | Use Protected Users group and disable NTLM where possible |
| NTLM Relay        | Enable SMB Signing and EPA                                |
| Kerberoasting     | Strong service account passwords or gMSA                  |
| Golden Ticket     | Reset KRBTGT password twice after compromise              |
| Password Spraying | Account lockout policies and monitoring                   |

---

# Key Takeaways

* NTLM uses challenge-response authentication but has significant security weaknesses.
* Kerberos is the preferred AD authentication protocol due to stronger security and ticket-based authentication.
* Authentication attacks such as Pass-the-Hash, Kerberoasting, and Golden Tickets remain common in enterprise environments.
* Monitoring Windows Event IDs provides valuable visibility into authentication-based attacks.
* Proper credential management and strong security controls significantly reduce attack opportunities.

---

## Skills Gained

* Active Directory Authentication Fundamentals
* NTLM Authentication Analysis
* Kerberos Ticket Workflow
* Impacket Usage
* Hashcat Usage
* Pass-the-Hash Understanding
* Kerberoasting Concepts
* Golden Ticket Concepts
* Windows Event Log Monitoring
* AD Authentication Attack Detection

---

**Room Completed:** Intro to AD Authentication
**Platform:** TryHackMe
**Focus Area:** Active Directory Authentication & Security
