# TryHackMe – Grep Writeup

## Scenario

Today we are solving another **TryHackMe Easy** Linux machine named **Grep**.

The machine initially appears to expose only a standard Apache web server, but deeper enumeration reveals additional virtual hosts, a custom CMS, source code disclosure, file upload functionality, and finally a leaked credential service that leads to full system compromise.

---

# Initial Enumeration

## Nmap Scan

We begin by performing a full TCP port scan.

```bash
nmap -p- 10.48.136.106
```

Open ports discovered:

```
22/tcp     SSH
80/tcp     HTTP
443/tcp    HTTPS
51337/tcp  HTTP
```

After identifying the open ports, perform service and version detection.

```bash
nmap -sC -sV 10.48.136.106
```

Service detection reveals:

- OpenSSH 8.2p1
- Apache 2.4.41
- HTTPS using a certificate for **grep.thm**
- HTTP service running on port **51337**

The SSL certificate immediately provides an important clue.

```
Common Name : grep.thm
```

This suggests that the application expects requests using a virtual hostname rather than the raw IP address.

---

# Website Enumeration

Visiting port 80 only displays the default Apache page.

No useful functionality is available.

Directory fuzzing is performed using Gobuster.

```bash
gobuster dir \
-u http://10.48.136.106 \
-w /usr/share/wordlists/SecLists/Discovery/Web-Content/raft-large-files.txt
```

Interesting discoveries include:

```
/index.php
/javascript/
/phpmyadmin (403)
/server-status (403)
```

Additional wordlists provide similar results.

No obvious attack surface is exposed on port 80.

---

# HTTPS Enumeration

Since the SSL certificate references **grep.thm**, we update our hosts file.

```text
10.48.136.106 grep.thm
```

Browsing:

```
https://grep.thm
```

now loads an entirely different application.

Instead of the Apache default page, a CMS named **SearchME CMS** becomes available.

This confirms that the primary web application is hosted using virtual hosts.

---

# OSINT Investigation

Searching publicly for SearchME CMS quickly identifies its GitHub repository.

While reviewing commit history, one commit exposes sensitive developer code.

The vulnerable commit reveals:

```php
if (isset($headers['X-THM-API-Key']) &&
$headers['X-THM-API-Key'] ===
'ffe60ecaa8bba2f12b43d1a4b15b8f39')
```

The application trusts a hardcoded API key.

Possessing this key allows interaction with protected functionality.

This demonstrates why sensitive credentials should never be committed to public repositories.

---

# User Registration

Using the exposed API functionality, registration succeeds.

A normal user account is created and authenticated successfully.

After logging into the CMS, several features become accessible.

One of the available functions is a profile image upload page.

This immediately becomes the primary attack vector.

---

# File Upload Vulnerability

A PHP reverse shell is prepared.

```php
<?php
// PentestMonkey PHP Reverse Shell
?>
```

Since the upload mechanism performs only basic validation, the payload is disguised.

The file is modified into a polyglot image.

```
sh2.jpg.php
```

Using a hex editor, JPEG magic bytes are inserted before the PHP code.

This allows the upload filter to accept the file while Apache still executes the embedded PHP code.

After uploading the payload, the uploaded file is accessed through the web server.

A reverse shell connects back successfully.

---

# Initial Shell

Listener:

```bash
nc -lvnp 4444
```

The uploaded payload executes.

```
www-data
```

The shell is upgraded immediately.

```bash
python3 -c 'import pty;pty.spawn("/bin/bash")'
```

Then stabilize the terminal.

```bash
CTRL+Z
stty raw -echo
fg
export TERM=xterm
```

A fully interactive shell is now available.

---

# Local Enumeration

Searching the web directories reveals backup files.

Inside:

```
/var/www/backups
```

interesting application data is discovered.

One database dump contains user information.

Example:

```sql
INSERT INTO users
```

Among the recovered accounts:

```
admin

test
```

The administrator email stands out.

```
admin@searchme2023cms.grep.thm
```

This suggests another internal service may exist.

---

# Discovering Leak Checker

Further enumeration discovers another application.

```
/var/www/leakchecker
```

Files present:

```
check_email.php

index.php
```

This strongly suggests another virtual host.

A new hosts entry is added.

```text
leakchecker.grep.thm
```

Port **51337** serves this application.

Browsing:

```
http://leakchecker.grep.thm:51337
```

reveals an email lookup service.

The previously discovered administrator email is entered into the search form.

```
admin@searchme2023cms.grep.thm
```

The application returns the administrator password.

This represents an information disclosure vulnerability where internal credential data has become publicly accessible.

At this point we have valid administrator credentials that can be used for further access.

---

# Attack Chain

```
Port Scan
      │
      ▼
SSL Certificate
      │
      ▼
grep.thm
      │
      ▼
SearchME CMS
      │
      ▼
GitHub OSINT
      │
      ▼
Hardcoded API Key
      │
      ▼
User Registration
      │
      ▼
Authenticated Upload
      │
      ▼
Polyglot PHP Reverse Shell
      │
      ▼
www-data Shell
      │
      ▼
Backup Enumeration
      │
      ▼
Admin Email Discovery
      │
      ▼
LeakChecker Service
      │
      ▼
Administrator Password Disclosure
```

---

# Enumeration Techniques Used

- Nmap
- Gobuster
- Virtual Host Discovery
- SSL Certificate Enumeration
- GitHub OSINT
- Source Code Review
- File Upload Testing
- Polyglot PHP Upload
- Reverse Shell
- Linux Enumeration
- Backup Discovery
- Virtual Host Enumeration
- Credential Leakage Investigation

---

# Vulnerabilities Identified

- Virtual host exposure
- Public GitHub source disclosure
- Hardcoded API key
- Weak upload validation
- Executable PHP upload
- Public backup files
- Information disclosure
- Credential leakage
- Multiple internal virtual hosts

---

# Tools Used

- Nmap
- Gobuster
- Burp Suite
- GitHub
- HexEdit
- Netcat
- Python PTY
- Linux CLI

---

# Lessons Learned

This machine demonstrates how several individually minor weaknesses can combine into a complete compromise. A public GitHub repository exposed a hardcoded API key, allowing unauthorized account creation. Weak file upload validation permitted a disguised PHP reverse shell to bypass restrictions and execute on the server. Local enumeration uncovered backup files containing sensitive application data, which eventually led to another internal service exposing administrator credentials. The challenge highlights the importance of secure source code management, proper upload validation, protecting backup files, and preventing sensitive information disclosure through internal applications.

---

# Conclusion

The Grep machine focused on realistic web application penetration testing and privilege escalation through information disclosure rather than exploiting complex software vulnerabilities. The attack progressed from reconnaissance and virtual host discovery to OSINT, source code analysis, authenticated file upload abuse, shell access, and internal enumeration. Finally, a leaked administrator credential from a secondary application completed the compromise, demonstrating how poor operational security across multiple services can expose an entire environment. :contentReference[oaicite:0]{index=0}
