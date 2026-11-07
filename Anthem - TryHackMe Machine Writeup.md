# TryHackMe: Anthem Writeup

![Difficulty](https://img.shields.io/badge/Difficulty-Easy-brightgreen)
![Platform](https://img.shields.io/badge/Platform-TryHackMe-red)
![OS](https://img.shields.io/badge/OS-Windows-blue)
![Category](https://img.shields.io/badge/Category-Penetration%20Testing-green)

A complete walkthrough of the **Anthem** machine on TryHackMe. This writeup documents the reconnaissance, web enumeration, credential discovery, initial access over RDP, and privilege escalation on a Windows host running an Umbraco CMS.

---

## Table of Contents

- [Overview](#overview)
- [Environment](#environment)
- [Attack Summary](#attack-summary)
- [1. Reconnaissance](#1-reconnaissance)
- [2. Web Enumeration](#2-web-enumeration)
- [3. Attempted Exploitation](#3-attempted-exploitation)
- [4. Credential Discovery](#4-credential-discovery)
- [5. Initial Access](#5-initial-access)
- [6. Privilege Escalation](#6-privilege-escalation)
- [Flags](#flags)
- [Lessons Learned](#lessons-learned)
- [Tools Used](#tools-used)
- [References](#references)

---

## Overview

| Field        | Value                             |
|--------------|-----------------------------------|
| Machine      | Anthem                            |
| Platform     | TryHackMe                         |
| Difficulty   | Easy                              |
| Target IP    | `10.49.153.112`                   |
| OS           | Windows 10 (Build 10.0.17763)     |
| CMS          | Umbraco on IIS 10.0               |
| Objective    | Obtain `user.txt` and `root.txt`  |

---

## Environment

| Component         | Details                       |
|-------------------|-------------------------------|
| Attacker Machine  | Kali Linux                    |
| Target            | `10.49.153.112`               |
| Virtual Host      | `anthem.com`                  |

---

## Attack Summary

The attack chain progresses through the following stages:

1. **Port scanning** showed only HTTP (80) and RDP (3389) open. ICMP was assumed to be filtered, so host discovery was skipped with `-Pn`.
2. **Web enumeration** of `robots.txt` leaked a string that was later confirmed to be a valid credential.
3. **Archive review** on the website exposed author names and an email address pattern.
4. **Public exploit research** against Umbraco did not succeed.
5. **Username construction** from the author archive produced a valid login for the Umbraco backend.
6. **Credential reuse** allowed Remote Desktop access as the user `sg`.
7. **Hidden file discovery** in `C:\backup` revealed the Administrator password after changing file permissions.
8. **Privilege escalation** was completed by logging in as `Administrator`.

---

## 1. Reconnaissance

### 1.1 Full TCP Port Scan

ICMP echo requests are commonly blocked on Windows hosts, so host discovery was disabled with `-Pn` and all ports were scanned.

```bash
nmap -Pn -p- -v 10.49.153.112
```

**Results:**

| Port  | State | Service       | Notes                        |
|-------|-------|---------------|------------------------------|
| 80    | open  | http          | Microsoft HTTPAPI httpd 2.0  |
| 3389  | open  | ms-wbt-server | Microsoft Terminal Services  |

All other 65,533 ports were filtered.

### 1.2 Service and Version Detection

```bash
nmap -Pn -sC -sV -p 80,3389 10.49.153.112
```

Key findings:

- **HTTP (80):** Microsoft HTTPAPI httpd, serving the public website.
- **RDP (3389):** Microsoft Terminal Services with a self-signed certificate. The NTLM info disclosed the hostname `WIN-LU09299160F`.
- **Operating system:** Windows 10 (build 10.0.17763).

Because RDP is exposed, any credentials discovered through the web application become a direct path to an interactive session.

---

## 2. Web Enumeration

### 2.1 Virtual Host

The site was accessed under the domain name `anthem.com`, which was added to the local hosts file for name resolution.

```bash
echo "10.49.153.112 anthem.com" | sudo tee -a /etc/hosts
```

### 2.2 Robots.txt Disclosure

Checking `robots.txt` revealed both a disallow list and a string placed at the top of the file:

```
UmbracoIsTheBest!

# Use for all search robots
User-agent: *

# Define the directories not to crawl
Disallow: /bin/
Disallow: /config/
Disallow: /umbraco/
Disallow: /umbraco_client/
```

The disallowed paths confirmed that the site is powered by Umbraco. Developers sometimes leave notes or credentials in files like this, so the string was retained for later testing.

### 2.3 Platform Fingerprinting

The backend was identified as Umbraco CMS running on IIS 10.0 on a Windows Server host.

### 2.4 Directory and File Enumeration

Directory brute-forcing was performed against the web application to identify additional content. Brute-forcing the Umbraco login was avoided to reduce the chance of the source IP being blocked.

```bash
gobuster dir -u http://anthem.com/ -w /usr/share/wordlists/dirb/common.txt -x txt,html,aspx
```

Enumeration of the site's content revealed two articles and their authors.

---

## 3. Attempted Exploitation

### 3.1 Metasploit Module Search

A public exploit for Umbraco was searched for within Metasploit:

```bash
msf > search umbraco
```

```
Matching Modules
================

   #  Name                                      Disclosure Date  Rank       Check  Description
   -  ----                                      ---------------  ----       -----  -----------
   0  exploit/windows/http/umbraco_upload_aspx  2012-06-28       excellent  No     Umbraco CMS Remote Command Execution
```

The module `exploit/windows/http/umbraco_upload_aspx` was tested, but it did not succeed against this instance. Attention was then directed toward credential-based access.

---

## 4. Credential Discovery

### 4.1 Author Enumeration

The two articles on the site listed authors whose naming pattern suggested a username scheme. One author was **James Orchard Halliwell**, which was used to infer an email format similar to `JD@anthem.com`.

### 4.2 Identifying the Author Pattern

The article content referenced "Solomon Grundy," an English nursery rhyme first collected and published in 1842 by James Orchard Halliwell in *The Nursery Rhymes of England*. The rhyme was used as a hint to construct a valid username from the author's naming convention.

Using this pattern, the following email address was crafted for the Umbraco backend:

```
sg@anthem.com
```

### 4.3 Umbraco Backend Login

The Umbraco login page was accessed and authenticated with:

```
Username: sg@anthem.com
Password: UmbracoIsTheBest!
```

Authentication succeeded, and the account was identified as **Solomon Grundy**. The backend was then reviewed for further content.

### 4.4 Flag Recovery from the Website

Within the CMS and the public site, the first web flag was found in the author articles. Two additional flags were located in the source code of the hiring page.

> Flag values are intentionally omitted from this public writeup in accordance with TryHackMe guidelines.

---

## 5. Initial Access

### 5.1 Password Reuse over RDP

The same credential that worked for the Umbraco backend was tested against the Remote Desktop service. Password reuse across web and system accounts is a common weakness, and this proved to be the case here.

```bash
xfreerdp /v:10.49.153.112 /u:sg /p:'UmbracoIsTheBest!' /cert:ignore
```

The session opened successfully as `sg`.

### 5.2 User Flag

The user flag was located on the desktop of the `sg` account.

```
C:\Users\sg\Desktop\user.txt
```

---

## 6. Privilege Escalation

### 6.1 Discovering Hidden Content

A hint in the environment indicated that something was hidden. Hidden items were made visible in File Explorer:

```
File Explorer -> View -> Show -> Hidden items (checked)
```

With hidden items visible, the directory `C:\backup` contained a file of interest.

### 6.2 Gaining Read Access to the Backup File

The file's permissions did not allow the `sg` account to read it directly. The permissions were modified through the graphical interface:

```
Right-Click -> Properties -> Security -> Edit -> Add
```

The `sg` user was added to the permissions list with read access and the change was applied. The file then became readable.

### 6.3 Recovering the Administrator Password

The file contained the password for the Administrator account:

```
Administrator : ChangeMeBaby1MoreTime
```

### 6.4 Gaining Administrator Access

The credential was used to log in as `Administrator`:

```
Username: Administrator
Password: ChangeMeBaby1MoreTime
```

With administrative access, the root flag was retrieved from the Administrator's desktop.

```
C:\Users\Administrator\Desktop\root.txt
```

---

## Flags

| Flag        | Location                                  | Value         |
|-------------|-------------------------------------------|---------------|
| Web flags   | Author articles and hiring page source    | `<web-flags>` |
| `user.txt`  | `C:\Users\sg\Desktop\user.txt`            | `<user-flag>` |
| `root.txt`  | `C:\Users\Administrator\Desktop\root.txt` | `<root-flag>` |

> Flag values are intentionally omitted from this public writeup in accordance with TryHackMe guidelines.

---

## Lessons Learned

- **Do not leave credentials in publicly accessible files.** A password placed at the top of `robots.txt` was enough to begin the attack chain.
- **Robots.txt is not an access control mechanism.** Disallowed paths are visible to anyone and often reveal the application's structure.
- **Avoid password reuse across services.** The same credential worked for both the CMS and Remote Desktop, turning a web compromise into a system shell.
- **Hidden files are not protected files.** Storing sensitive data in a hidden directory provides no security if permissions allow it to be read.
- **Overly permissive file ACLs are a serious risk.** Read access to a backup containing an Administrator password effectively grants full system control.
- **Public exploits are not guaranteed to work.** Version-matched modules may still fail due to configuration differences, so manual enumeration remains essential.

---

## Tools Used

| Tool       | Purpose                                         |
|------------|-------------------------------------------------|
| Nmap       | Port scanning and service fingerprinting        |
| Gobuster   | Web directory and file enumeration              |
| Metasploit | Public exploit search and testing               |
| xfreerdp   | Remote Desktop access with recovered credentials|
| Browser    | Umbraco backend login and content review        |

---

## References

- [TryHackMe: Anthem](https://tryhackme.com/)
- [Umbraco CMS](https://umbraco.com/)
- [Nmap Documentation](https://nmap.org/book/man.html)
- [OWASP Testing Guide](https://owasp.org/www-project-web-security-testing-guide/)

---

## Disclaimer

This writeup is intended solely for educational purposes and was performed in a legal, authorized lab environment provided by TryHackMe. The techniques described should only be used against systems where you have explicit written permission to test.


# Jai Shri Ram
