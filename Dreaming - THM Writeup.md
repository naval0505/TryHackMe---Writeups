# Dreaming - TryHackMe Walkthrough

> Difficulty: Easy
> Platform: TryHackMe
> Operating System: Linux
> Target IP: `10.48.186.105`

---

# Overview

Dreaming is a Linux-based machine that focuses on web enumeration, CMS assessment, credential discovery, database abuse, and privilege escalation through insecure permissions and scheduled task execution.

The objective was to gain access to the machine, enumerate available services, identify weaknesses within the web application, pivot between multiple user accounts, and ultimately obtain root privileges.

---

# Initial Enumeration

## Nmap Scan

A full TCP port scan was performed against the target.

### Open Ports

| Port | Service |
| ---- | ------- |
| 22   | SSH     |
| 80   | HTTP    |

### Service Detection Results

```text
22/tcp open  ssh
OpenSSH 8.2p1 Ubuntu

80/tcp open  http
Apache httpd 2.4.41 Ubuntu
```

### Observations

* SSH service was available.
* Apache web server was running.
* Default Apache landing page was presented.
* No additional services were exposed.

---

# Web Enumeration

Since HTTP was the primary attack surface, directory enumeration was performed.

## Gobuster Results

```text
/app
/server-status
```

Interesting findings:

```text
/app
```

The `/app` directory appeared to host the actual web application.

---

# CMS Identification

Navigating to:

```text
http://TARGET/app
```

revealed a login portal.

Further inspection identified the application as:

```text
Pluck CMS 4.7.13
```

---

# Vulnerability Research

Version identification allowed targeted vulnerability research.

The discovered CMS version was associated with:

```text
CVE-2020-29607
```

### Description

The vulnerability allows an authenticated administrator to upload executable files through the file management functionality, resulting in remote code execution.

### References

* Exploit-DB ID: 49909
* CVE-2020-29607
* Public proof-of-concept exploit available online

---

# Initial Access

After authenticating to the CMS administration panel using valid credentials discovered during the challenge, the vulnerable upload functionality was abused.

The attack resulted in remote code execution through an uploaded server-side payload.

### Result

Access obtained as:

```text
www-data
```

---

# Post-Exploitation Enumeration

Once access was established, local enumeration began.

Important directories and files were inspected.

A Python script located within `/opt` contained useful information.

## Interesting Discovery

The script contained hardcoded credentials associated with another account.

### Findings

* Internal application URL
* Reused credentials
* Potential user pivot opportunity

---

# User Pivot - Lucien

Using the recovered credentials, access was obtained as:

```text
lucien
```

---

# Privilege Escalation Enumeration

Checking sudo permissions revealed:

```text
lucien may run:

(death) NOPASSWD:
/usr/bin/python3 /home/death/getDreams.py
```

This represented a potential privilege escalation path.

---

# Database Discovery

Reviewing shell history exposed database credentials.

### Database Access

The recovered credentials provided access to a local MySQL instance.

Database contents revealed:

```text
dreams
```

Example entries:

| Dreamer | Dream                              |
| ------- | ---------------------------------- |
| Alice   | Flying in the sky                  |
| Bob     | Exploring ancient ruins            |
| Carol   | Becoming a successful entrepreneur |
| Dave    | Becoming a professional musician   |

---

# Pivot to Death

Analysis of the privileged script showed that it processed entries from the database.

By understanding the script's behavior and how user-controlled database content was handled, code execution was achieved in the context of:

```text
death
```

### Result

User access obtained:

```text
death
```

---

# Further Enumeration

While operating as `death`, additional file-system enumeration was performed.

Several scheduled and automated processes were identified.

Particular attention was given to Python-related files and imported modules.

---

# Privilege Escalation to Morpheus

A Python import abuse opportunity was discovered.

### Root Cause

* Insecure file permissions
* Writable Python module
* Trusted script execution under a higher privileged account

When the scheduled task executed, the imported module was loaded automatically.

This resulted in code execution as:

```text
morpheus
```

---

# Sudo Rights Review

Running:

```bash
sudo -l
```

revealed:

```text
(ALL) NOPASSWD: ALL
```

This configuration granted unrestricted sudo access.

---

# Root Access

Administrative access was obtained immediately through sudo.

### Verification

```text
whoami

root
```

---

# Flags

## Morpheus Flag

```text
THM{DR34MS_5H4P3_TH3_W0RLD}
```

---

# Attack Path Summary

```text
Web Enumeration
        ↓
Pluck CMS Discovery
        ↓
Authenticated CMS Abuse
        ↓
www-data
        ↓
Credential Discovery
        ↓
lucien
        ↓
Database Enumeration
        ↓
death
        ↓
Python Module Abuse
        ↓
morpheus
        ↓
Sudo Misconfiguration
        ↓
root
```

---

# Key Lessons Learned

### Web Enumeration

* Always enumerate directories and hidden content.
* Default pages often conceal vulnerable applications.

### Version Identification

* Accurate software identification enables focused vulnerability research.
* Public CVEs frequently provide viable attack paths.

### Credential Security

* Hardcoded credentials remain a common weakness.
* Bash history can expose sensitive information.

### Database Security

* User-controlled data processed by privileged scripts can create dangerous trust boundaries.

### Python Security

* Writable libraries and import paths can become privilege escalation vectors.
* Scheduled tasks should execute with minimal privileges.

### Sudo Configuration

* `NOPASSWD: ALL` effectively grants full system compromise.
* Least privilege principles should always be enforced.

---

# Conclusion

Dreaming is an excellent beginner-friendly Linux machine that demonstrates a complete attack chain from web application compromise to full system takeover.

The challenge highlights several real-world security issues:

* Weak credential management
* Vulnerable CMS deployments
* Insecure database interactions
* Python module hijacking
* Dangerous sudo configurations

Overall, it provides valuable experience in enumeration, lateral movement, and privilege escalation while reinforcing the importance of secure operational practices.
